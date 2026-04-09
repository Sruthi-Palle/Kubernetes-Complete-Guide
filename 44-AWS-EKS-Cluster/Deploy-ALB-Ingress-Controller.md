# Deploying Ingress controller (ALB)

## OIDC Explanation

### The Challenge of Cross-Environment Resource Management

A fundamental architectural challenge exists when a Kubernetes pod needs to interact with the broader AWS ecosystem.

- **Internal vs. External Resources:** The ALB Controller is deployed as a pod within the EKS cluster. However, the resources it manages, such as Elastic Load Balancers (ELB), exist outside the Kubernetes cluster within the AWS environment.
- **Authorization Requirements:** In AWS, resource creation is strictly governed by IAM. Any entity attempting to create a resource must be an IAM user or role associated with specific policies granting the necessary permissions.
- **The Identity Gap:** By default, pods utilize Kubernetes Service Accounts, which lack inherent authority to interact with AWS APIs. Without a method to bridge the Service Account to an IAM role, the ALB Controller cannot execute its primary function of creating load balancers.

### The IAM OIDC Provider: The Binding Mechanism

The IAM OIDC provider serves as the critical "connector" that resolves the identity gap between Kubernetes and AWS.

### Core Functionality

The OIDC provider allows for the binding of a Kubernetes Service Account to an AWS IAM role. This binding ensures that when a pod (the ALB Controller) performs an action, it does so with the permissions granted to the linked IAM role.

**Key Components of the Connectivity Model**

<table style="min-width: 50px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Component</strong></p></td><td colspan="1" rowspan="1"><p><strong>Description</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>ALB Controller Pod</strong></p></td><td colspan="1" rowspan="1"><p>The functional unit running within the EKS cluster responsible for ELB management.</p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>Service Account</strong></p></td><td colspan="1" rowspan="1"><p>The Kubernetes-native identity assigned to the ALB Controller pod.</p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>IAM Role</strong></p></td><td colspan="1" rowspan="1"><p>The AWS-native identity that holds the specific permissions (policies) required to create ELB resources.</p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>IAM OIDC Provider</strong></p></td><td colspan="1" rowspan="1"><p>The bridge that connects the Service Account to the IAM Role, enabling cross-platform authorization.</p></td></tr></tbody></table>

### Implementation Workflow

To successfully enable the ALB Controller to create AWS resources, a specific sequence of prerequisite steps must be followed:

1.  **Install the IAM OIDC Provider:** This is the foundational step required before the ALB Controller can be installed.
2.  **Create an IAM Role:** Define a specific role within AWS that will represent the ALB Controller's permissions.
3.  **Assign Policies:** Attach IAM policies to the role that explicitly grant permissions to create and manage Elastic Load Balancers and other required AWS resources.
4.  **Bind the Service Account:** Connect the Kubernetes Service Account to the IAM role using the IAM OIDC provider.

**Interview Significance**

- **Key Interview Takeaway:** If asked how a pod within an EKS cluster can perform activities on resources located outside the cluster, explain that we avoid hardcoded access keys. Instead, we use **IRSA (IAM Roles for Service Accounts)**. By linking a **Kubernetes Service Account** to an **IAM Role** via an **OIDC Provider**, we grant pods temporary, least-privilege credentials to manage resources like ALBs securely.

---

---

# Configuring OIDC and Deployment steps:

### Prerequisites

- An active **EKS Cluster**.
- **Helm, eksctl** and **kubectl** installed locally.
- Make sure you are connected to correct cluster by running `kubectl config current-context`
- The **IAM OIDC Provider** must be enabled for your cluster.

  **Check If there is an IAM OIDC Provider configured already:**
  - **Retrieve your Cluster's OIDC ID:** Every EKS cluster has a unique OIDC issuer URL. You need the ID (the string at the end of the URL).

    Bash

    ```plaintext
    aws eks describe-cluster --name <your-cluster-name> --query "cluster.identity.oidc.issuer" --output text
    ```

    _Output example:_ [`https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED532145ADBA9028C359FDA458`](https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED532145ADBA9028C359FDA458)

  - **Check if that ID exists in IAM:** Copy the ID (the part after `/id/`) and run:

    Bash

    ```plaintext
    aws iam list-open-id-connect-providers | grep <YOUR_OIDC_ID>
    ```

  - **If the grep returns a match:** You are good to go. Skip the `associate` command and move straight to creating the IAM policy.
  - **If the grep returns nothing:** You must run the `eksctl utils associate-iam-oidc-provider` command.

  **If IAM OIDC provider not configured, you can run below command :**

  ```bash theme={null}
  eksctl utils associate-iam-oidc-provider --cluster <your-cluster-name> --approve
  ```

---

### Step 1: Create the IAM Policy

The controller needs permissions to make API calls to AWS (like creating an ALB).

1.  **Download the policy:**

    ```bash
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
    ```

2.  **Create the IAM policy:**

    ```bash
    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam-policy.json
    ```

### Step 2: Create an IAM Role and Service Account

Using `eksctl` is the fastest way to link a Kubernetes Service Account to the IAM policy created above.

```bash
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### Step 3: Install the Controller via Helm

Now, deploy the actual controller pods into the `kube-system` namespace.

1.  **Add the EKS chart repo:**

    ```bash
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update
    ```

2.  **Install the chart:**

    > **Note:** Replace `<cluster-name>`, `<region>`, and `<vpc-id>` with your actual values.

    ```bash
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
      -n kube-system \
      --set clusterName=<your-cluster-name> \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller \
      --set region=<your-region> \
      --set vpcId=<your-vpc-id>
    ```

---

### Step 4: Verify the Installation

Check that the controller is running and healthy:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Common Troubleshooting Tips

- **Subnet Tagging:** For the controller to auto-discover your subnets, ensure they are tagged correctly:
  - **Public Subnets:** `kubernetes.io/role/elb` = `1`
  - **Private Subnets:** `kubernetes.io/role/internal-elb` = `1`

- **VPC ID:** If you are using a custom VPC (not the default one created by eksctl), double-check that the `vpcId` passed in the Helm command is correct.
- **Logs:** If the controller isn't provisioning ALBs, check the logs for permission errors:

  ```bash
  kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
  ```

---

---

# Command Explanation

```bash
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

This command is essentially the "glue" that connects your Kubernetes cluster to your AWS account. It sets up **IAM Roles for Service Accounts (IRSA)**, which allows pods to interact with AWS services securely without needing global credentials.

Here is a breakdown of what each part of that command is doing:

---

### 1\. Line-by-Line Breakdown

| Parameter                     | What it does                                                                                                                                            |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `create iamserviceaccount`    | Tells `eksctl` to create two things: a Kubernetes **ServiceAccount** and an AWS **IAM Role**.                                                           |
| `--namespace=kube-system`     | Places the ServiceAccount in the `kube-system` namespace, where cluster-level add-ons live.                                                             |
| `--name=aws-load-balancer...` | The name of the Kubernetes ServiceAccount object that the Controller pod will use.                                                                      |
| `--role-name ...`             | The name of the IAM Role that will be created in your **AWS Console**.                                                                                  |
| `--attach-policy-arn`         | Links the specific permissions (created in Step 1) to this IAM Role. This dictates _exactly_ what the controller can do (e.g., "Create Load Balancer"). |
| `--approve`                   | Without this, `eksctl` only shows you the plan. This flag tells it to actually execute the changes in CloudFormation.                                   |

---

### 2\. What happens "Under the Hood"?

When you run this, `eksctl` performs three major actions:

1.  **Creates an IAM Role** in AWS with a **Trust Policy**. This policy says: _"I trust the OIDC provider of this specific EKS cluster to assume this role."_
2.  **Attaches your Policy** to that role so it has the permissions to manage ALBs.
3.  **Creates a Kubernetes ServiceAccount** and adds an **Annotation** to it: `eks.amazonaws.com/role-arn: arn:aws:iam::1234567890:role/AmazonEKSLoadBalancerControllerRole`

### The Result

When the Load Balancer Controller pod starts up, it looks at that ServiceAccount, sees the annotation, and automatically "assumes" the IAM role. It can now reach out to AWS and spin up a Load Balancer on your behalf!

---

### Verification:

To verify that the "link" was created successfully and that your pods can actually use those permissions, you can perform two quick checks.

### 1\. Verify the Service Account Annotation

First, check if the Kubernetes Service Account was correctly annotated with the IAM Role ARN. This annotation is the signal to EKS that any pod using this account should be granted the IAM role's permissions.

Run this command:

```bash
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```

**What to look for:** Under the `metadata` section, you should see an `annotations` field like this:

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/AmazonEKSLoadBalancerControllerRole
```

---

### 2\. Check the Trust Relationship in AWS

The IAM Role in your AWS Console must "trust" your cluster's OIDC provider. Without this, the pod can't "assume" the role.

1.  Go to the **IAM Console** > **Roles**.
2.  Search for `AmazonEKSLoadBalancerControllerRole`.
3.  Click the **Trust relationships** tab.

It should look similar to this JSON structure:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.region.amazonaws.com/id/EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.region.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
```

---

### Troubleshooting Tip

If the controller pods are running but you see errors in the logs like `WebIdentityErr: failed to retrieve credentials`, it usually means:

- The **OIDC Provider URL** in the Trust Relationship doesn't match your cluster exactly.
- The **Service Account name** or **Namespace** in the "Condition" block has a typo.

Once this link is solid, your controller will have the "green light" to start creating ALBs whenever you deploy an Ingress resource!

### When to use --overwrite-existing-serviceaccounts flag?

- Recovery: If a previous installation failed halfway through.

- Updates: If you are changing the IAM Role or Policy associated with the controller.

- Migrations: If you manually created the ServiceAccount with same name earlier and now want eksctl to take over the management of its IAM permissions.

```bash
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --overwrite-existing-serviceaccounts
```
