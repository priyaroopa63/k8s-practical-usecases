# Karpenter v0.37.0 on EKS — Complete Working Guide

> Tested on EKS v1.35 · ap-south-1 · Git Bash (MINGW64) · Single-node bootstrap cluster (t2.small)

Karpenter is a node autoscaler that watches for unschedulable pods and provisions the right EC2 instance to fit them — then removes it when it's no longer needed. This guide documents every step including the pitfalls we hit and how to fix them.

---

## Prerequisites

- EKS cluster running and accessible via `kubectl`
- AWS CLI installed and configured
- `eksctl` installed
- `helm` installed
- `envsubst` available — on Windows Git Bash this comes with Git by default. If missing, you'll need to manually replace `${AWS_ACCOUNT_ID}`, `${AWS_REGION}` and `${CLUSTER_NAME}` in any JSON files before applying them.

---

## Step 1 — Configure AWS CLI & kubeconfig

```bash
aws configure
aws eks update-kubeconfig --region ap-south-1 --name <your-cluster-name>
```

---

## Step 2 — Export Environment Variables

Pin the Karpenter version explicitly. Never use `latest` — version mismatches between the Helm chart and CRDs will silently break things.

```bash
export CLUSTER_NAME="karpenter"
export AWS_REGION="$(aws configure get region)"
export AWS_PARTITION="aws"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query 'Account' --output text)"
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name $CLUSTER_NAME \
  --query "cluster.endpoint" --output text)"
export OIDC_ENDPOINT="$(aws eks describe-cluster --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" --output text)"
export KARPENTER_VERSION="0.37.0"
export KARPENTER_NAMESPACE="kube-system"
```

Verify everything resolved correctly before continuing:

```bash
echo "Cluster:   $CLUSTER_NAME"
echo "Region:    $AWS_REGION"
echo "Account:   $AWS_ACCOUNT_ID"
echo "Endpoint:  $CLUSTER_ENDPOINT"
echo "OIDC:      $OIDC_ENDPOINT"
echo "Version:   $KARPENTER_VERSION"
```

If any value is blank, do not continue — the IAM trust policies will be created with empty ARNs and you'll spend time debugging auth failures later.

---

## Step 3 — Create IAM OIDC Provider

Required for IRSA (IAM Roles for Service Accounts). Without this, the Karpenter controller pod cannot assume its IAM role and will fail to call EC2 APIs.

Check if it already exists first:

```bash
aws iam list-open-id-connect-providers | grep $(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" \
  --output text | cut -d '/' -f 5)
```

If nothing is returned, create it:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --approve
```

---

## Step 4 — Create KarpenterNodeRole

This role is assumed by the EC2 instances that Karpenter launches. It is different from the controller role — the node role goes on the EC2 instance itself.

```bash
cat > node-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --assume-role-policy-document file://node-trust-policy.json
```

---

## Step 5 — Attach Required Node Policies

> ⚠️ **Critical — do not skip any of these four.** Missing `AmazonEKSWorkerNodePolicy` is the most common reason Karpenter-provisioned nodes boot successfully but never register with the cluster. The node calls `ec2:DescribeInstances` during bootstrap to discover itself — without this policy it gets a 403 and the bootstrap loop silently fails.

Attach each policy individually to make sure none get missed:

```bash
aws iam attach-role-policy \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKSWorkerNodePolicy"

aws iam attach-role-policy \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKS_CNI_Policy"

aws iam attach-role-policy \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"

aws iam attach-role-policy \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonSSMManagedInstanceCore"
```

Verify all four are attached before moving on:

```bash
aws iam list-attached-role-policies \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --query 'AttachedPolicies[].PolicyName' \
  --output table
```

You should see all four listed. If any are missing, attach them again — the command is idempotent.

---

## Step 6 — Create Instance Profile

The instance profile is the container that wraps the node role and gets attached to EC2 instances.

```bash
aws iam create-instance-profile \
  --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}"

aws iam add-role-to-instance-profile \
  --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}" \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}"
```

---

## Step 7 — Create KarpenterControllerRole

The controller role is assumed by the Karpenter pod itself via IRSA. It gives Karpenter permission to call EC2 APIs to launch and terminate instances.

> ⚠️ The OIDC issuer URL must have the `https://` prefix stripped in the trust policy. The `${OIDC_ENDPOINT#*//}` shell substitution handles this automatically — do not hardcode the URL manually or you'll get a malformed ARN.

```bash
cat > controller-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT#*//}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
        "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:${KARPENTER_NAMESPACE}:karpenter"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --assume-role-policy-document file://controller-trust-policy.json
```

---

## Step 8 — Create and Attach Controller Policy

Create the policy document first. The `${...}` variables will be substituted by `envsubst` in the next command.

```bash
cat > controller-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Karpenter",
      "Effect": "Allow",
      "Resource": "*",
      "Action": [
        "ssm:GetParameter",
        "ec2:DescribeImages",
        "ec2:RunInstances",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeLaunchTemplates",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeInstanceTypeOfferings",
        "ec2:DescribeAvailabilityZones",
        "ec2:DeleteLaunchTemplate",
        "ec2:CreateTags",
        "ec2:CreateLaunchTemplate",
        "ec2:CreateFleet",
        "ec2:DescribeSpotPriceHistory",
        "pricing:GetProducts"
      ]
    },
    {
      "Sid": "ConditionalEC2Termination",
      "Effect": "Allow",
      "Action": "ec2:TerminateInstances",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/karpenter.sh/nodepool": "*"
        }
      }
    },
    {
      "Sid": "PassNodeIAMRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
    },
    {
      "Sid": "EKSClusterEndpointLookup",
      "Effect": "Allow",
      "Action": "eks:DescribeCluster",
      "Resource": "arn:aws:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${CLUSTER_NAME}"
    },
    {
      "Sid": "AllowInstanceProfileReadActions",
      "Effect": "Allow",
      "Resource": "*",
      "Action": "iam:GetInstanceProfile"
    },
    {
      "Sid": "CreateServiceLinkedRoleForEC2Spot",
      "Effect": "Allow",
      "Action": "iam:CreateServiceLinkedRole",
      "Resource": "arn:aws:iam::*:role/aws-service-role/spot.amazonaws.com/AWSServiceRoleForEC2Spot"
    }
  ]
}
EOF
```

Substitute variables and attach:

```bash
envsubst < controller-policy.json > controller-policy-final.json

aws iam put-role-policy \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --policy-name "KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --policy-document file://controller-policy-final.json
```

Verify it was attached correctly:

```bash
aws iam get-role-policy \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --policy-name "KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --query 'PolicyDocument.Statement[].Sid' \
  --output table
```

You should see all 6 Sids listed.

---

## Step 9 — Tag Subnets & Security Group

> ⚠️ **Do not skip this.** The `EC2NodeClass` uses these tags to discover which subnets and security groups to use when launching nodes. Missing tags = Karpenter cannot find any subnets = `NodeClassNotReady` and no nodes will ever launch.

```bash
# Tag subnets across all nodegroups
for NG in $(aws eks list-nodegroups --cluster-name $CLUSTER_NAME \
  --query 'nodegroups' --output text); do
  aws ec2 create-tags \
    --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
    --resources $(aws eks describe-nodegroup \
      --cluster-name $CLUSTER_NAME \
      --nodegroup-name $NG \
      --query 'nodegroup.subnets' --output text)
done

# Tag cluster security group
SG=$(aws eks describe-cluster --name $CLUSTER_NAME \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

aws ec2 create-tags \
  --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
  --resources $SG
```

Verify the tags landed:

```bash
aws ec2 describe-subnets \
  --filters "Name=tag:karpenter.sh/discovery,Values=${CLUSTER_NAME}" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone}' \
  --output table

aws ec2 describe-security-groups \
  --filters "Name=tag:karpenter.sh/discovery,Values=${CLUSTER_NAME}" \
  --query 'SecurityGroups[].{ID:GroupId,Name:GroupName}' \
  --output table
```

You should see subnets across multiple AZs and the cluster SG. If either table is empty, the tags did not apply correctly — re-run the tagging commands.

---

## Step 10 — Map Node Role to aws-auth

Allows Karpenter-provisioned nodes to authenticate with the Kubernetes API and join the cluster.

```bash
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster $CLUSTER_NAME \
  --arn "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}" \
  --group system:bootstrappers \
  --group system:nodes
```

Verify it was added:

```bash
kubectl get configmap aws-auth -n kube-system -o yaml | grep -A 4 "KarpenterNodeRole"
```

You should see your role ARN with `system:bootstrappers` and `system:nodes` groups.

---

## Step 11 — Install Karpenter via Helm

Install CRDs first as a separate Helm release. This is important — the CRD version must exactly match the Helm chart version. Mismatched versions cause `no matches for kind` errors that are confusing to debug.

```bash
helm upgrade --install karpenter-crd \
  oci://public.ecr.aws/karpenter/karpenter-crd \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --create-namespace
```

Then install the controller. A few things to note about the flags below:

- `replicas=1` — prevents a pod anti-affinity deadlock on single-node clusters. With 2 replicas, the second pod refuses to schedule on the same node as the first and stays `Pending` forever.
- `cpu=100m` — the default of `1` full CPU causes `Insufficient cpu` errors on small bootstrap nodes like `t2.small`. 100m is plenty for dev/test.
- No `--wait` flag — avoids Helm timing out and leaving a stuck lock that blocks future upgrades.

```bash
helm upgrade --install karpenter \
  oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.clusterEndpoint=${CLUSTER_ENDPOINT}" \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=100m \
  --set controller.resources.requests.memory=256Mi \
  --set controller.resources.limits.cpu=200m \
  --set controller.resources.limits.memory=256Mi \
  --set replicas=1
```

Verify the pod is running before continuing:

```bash
kubectl get pods -n ${KARPENTER_NAMESPACE} -l app.kubernetes.io/name=karpenter
```

Wait until you see `1/1 Running`. If the pod is stuck in `Pending`, check events:

```bash
kubectl describe pod -n ${KARPENTER_NAMESPACE} -l app.kubernetes.io/name=karpenter | grep -A 10 "Events"
```

---

## Step 12 — Verify CRDs

```bash
kubectl get crds | grep karpenter
```

You should see exactly these three:

```
ec2nodeclasses.karpenter.k8s.aws
nodeclaims.karpenter.sh
nodepools.karpenter.sh
```

Also confirm the API versions — v0.37.0 uses `v1beta1`, not `v1`:

```bash
kubectl get crd ec2nodeclasses.karpenter.k8s.aws \
  -o jsonpath='{.spec.versions[*].name}'
```

Expected output: `v1beta1`. If it says `v1`, your CRD version doesn't match the chart version.

---

based on this only yml files needs to create.

## Step 13 — Create EC2NodeClass

The EC2NodeClass defines AWS-specific settings: which subnets to use, which security groups, which AMI family, and which instance profile.

> ⚠️ Use `instanceProfile` here, **not** `role`. In `v1beta1`, the `role` field triggers Karpenter to try to create a new instance profile dynamically — which requires `iam:CreateInstanceProfile` on the controller role. Using `instanceProfile` directly points to the one we already created in Step 6 and avoids that permission issue entirely. Also note that switching between `role` and `instanceProfile` on an existing EC2NodeClass is not supported — you must delete and recreate it.

```bash
cat > ec2nodeclass.yaml << 'EOF'
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  instanceProfile: "KarpenterNodeInstanceProfile-karpenter"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "karpenter"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "karpenter"
  tags:
    managed-by: karpenter
    cluster: "karpenter"
EOF

kubectl apply -f ec2nodeclass.yaml
```

Verify it becomes `Ready`:

```bash
kubectl describe ec2nodeclass default | grep -A 5 "Conditions"
```

You should see `Status: True` and `Type: Ready`. If it shows `NodeClassNotReady` with a subnet or security group error, your tags from Step 9 didn't apply correctly.

---

## Step 14 — Create NodePool

The NodePool defines what kinds of nodes Karpenter is allowed to create.

> ⚠️ **Always set `limits`.** Without them, a misconfigured deployment or a runaway HPA could trigger Karpenter to scale to hundreds of nodes before anyone notices. The limits below cap the total CPU and memory Karpenter can provision across all nodes — treat this as your safety net.

> ⚠️ **v1beta1 naming differences from v1** — `consolidationPolicy` only supports `WhenUnderutilized` or `WhenEmpty` in this version. The `WhenEmptyOrUnderutilized` value and the `consolidateAfter` field are v1-only features and will cause a `strict decoding error` if used here.

```bash
cat > nodepool.yaml << 'EOF'
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        managed-by: karpenter
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
  limits:
    cpu: "100"
    memory: 400Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
EOF

kubectl apply -f nodepool.yaml
```

Verify both resources are ready:

```bash
kubectl get nodepool
kubectl get ec2nodeclass
```

---

## Step 15 — Test Autoscaling

Deploy a test workload using the `pause` image — it requests CPU but uses almost no actual resources, making it ideal for testing node provisioning without burning money.

```bash
cat > inflate.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: "1"
EOF

kubectl apply -f inflate.yaml
```

Scale up to force Karpenter to provision a new node:

```bash
kubectl scale deployment inflate --replicas 5
```

Watch Karpenter react in real time — run these in separate terminals:

```bash
# Terminal 1 — watch pods schedule
kubectl get pods -w

# Terminal 2 — watch new nodes appear
kubectl get nodes -w

# Terminal 3 — watch Karpenter controller logs
kubectl logs -n ${KARPENTER_NAMESPACE} \
  -l app.kubernetes.io/name=karpenter -f
```

What to look for in the logs:

```
launched nodeclaim   ← Karpenter sent the EC2 launch request
registered node      ← new node joined the cluster
```

A new node should appear within 60-90 seconds. Pods will go `Pending → Running` once the node is `Ready`.

**Test scale-down:**

```bash
kubectl scale deployment inflate --replicas 0
```

Karpenter should deprovision the node within 1-2 minutes via the `WhenUnderutilized` consolidation policy.

---
