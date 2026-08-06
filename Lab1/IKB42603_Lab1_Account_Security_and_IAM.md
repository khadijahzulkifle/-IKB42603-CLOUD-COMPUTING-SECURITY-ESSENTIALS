# Lab 1: Cloud Account Security, Identity and Access Management

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 1  
**Topic:** Identity Governance, Least Privilege, LocalStack IAM and Kubernetes RBAC  
**Environment:** LocalStack (localhost:4566) and Kind Kubernetes Cluster (ccse-lab1)  
**Name:** Khadijah

---

# Objective

The objective of this lab is to understand cloud identity and access management using LocalStack IAM and Kubernetes RBAC. This lab demonstrates how to implement the principle of least privilege, manage cloud identities securely, and enforce authorization using Kubernetes Role-Based Access Control (RBAC).

---

# Learning Outcomes

At the end of this lab, students will be able to:

1. Stand up a local cloud lab using Docker and LocalStack (an AWS-compatible simulator).
2. Apply the principle of least privilege by replacing root usage with scoped IAM users, groups, and policies.
3. Create and test fine-grained permissions, distinguishing what an identity is allowed versus denied to do.
4. Implement and verify Role-Based Access Control (RBAC) in Kubernetes.
5. Audit identities and reason about MFA, access keys, and credential hygiene.

---

# Environment

- Operating System: Kali Linux
- Docker
- LocalStack
- AWS CLI
- kind (Kubernetes in Docker)
- kubectl
- Kali Linux Terminal

---

# Environment Setup

The AWS CLI was configured to communicate with the LocalStack endpoint.

Command

```bash
EP='--endpoint-url=http://localhost:4566'
```

Verification

```bash
aws --endpoint-url=http://localhost:4566 sts get-caller-identity
```

The returned account ID **000000000000** confirmed that all AWS CLI commands were executed against LocalStack instead of the real AWS cloud.

---

# Step-by-Step Implementation

## Task 1: Map the Cloud Identity Landscape

Before creating IAM resources, the basic cloud identity concepts were reviewed.

| Concept | AWS Term | Purpose |
|---|---|---|
| All-powerful owner | Root User | The original account owner with full control over all resources and billing. It should be protected and not used for daily administration. |
| Human/app identity | IAM User | A named identity for a person, application or service that needs credentials to access cloud resources. |
| Permission bundle | IAM Policy | A JSON permission document that defines which actions are allowed or denied on specific resources. |
| Collection of users | IAM Group | A way to manage permissions for multiple users together by attaching policies to the group. |
| Temporary identity | IAM Role | An identity that can be assumed temporarily to grant short-lived permissions without long-term user credentials. |

---

## Task 2: Create a Least-Privilege Admin

### Step 1: Create Admins Group

**Command**

```bash
aws $EP iam create-group --group-name Admins
```

**Result**

The group **Admins** was created successfully.

**Figure 1:** Create Admins Group

![](<STEP 2.1.png>)

---

### Step 2: Attach Administrator Policy

**Command**

```bash
aws $EP iam attach-group-policy --group-name Admins \
--policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

**Verification**

```bash
aws $EP iam list-attached-group-policies --group-name Admins
```

The **AdministratorAccess** policy was successfully attached to the **Admins** group.

**Figure 2:** Administrator Policy Attached

![](<STEP 2.2.png>)

---

### Step 3: Create Personal Admin User

**Command**

```bash
aws $EP iam create-user --user-name CloudAdmin_khadijah
```

**Result**

The user **CloudAdmin_khadijah** was created successfully.

**Figure 3:** Create Administrator User

![](<STEP 2.3.png>)

---

### Step 4: Add User to Admins Group

**Command**

```bash
aws $EP iam add-user-to-group --group-name Admins \
--user-name CloudAdmin_khadijah
```

**Verification**

```bash
aws $EP iam get-group --group-name Admins
```

The output confirmed that **CloudAdmin_khadijah** is a member of the **Admins** group. The administrator permissions are inherited through the group instead of being attached directly to the user.

**Figure 4:** Verify Admin Group Membership

![](<STEP 2.4.png>)

---

## Task 3: Enforce Least Privilege

### Step 5: Create Read-Only Analyst User

**Command**

```bash
aws $EP iam create-user --user-name Analyst_khadijah
```

**Result**

The user **Analyst_khadijah** was created successfully.

**Figure 5:** Create Analyst User

![](<STEP 3.1.png>)

---

### Step 6: Attach S3 Read-Only Policy

**Command**

```bash
aws $EP iam attach-user-policy \
--user-name Analyst_khadijah \
--policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

The read-only policy was successfully attached.

**Figure 6:** Attach Read-Only Policy

![](<STEP 3.2-1.png>)

---

### Step 7: Verify Analyst Permissions

**Verification**

```bash
aws $EP iam list-attached-user-policies --user-name Analyst_khadijah
```

The output showed that the Analyst account has only the **AmazonS3ReadOnlyAccess** policy.

**Figure 7:** Verify Analyst Permissions

![](<STEP 3.3.png>)

### Least Privilege Explanation

The Analyst account demonstrates the principle of least privilege because it has only read-only access to Amazon S3 resources. If the account is compromised, the attacker cannot create users, delete resources, modify IAM policies, or perform administrative tasks. This minimizes the potential impact of unauthorized access.

---

## Task 4: Credential Hygiene and Access Keys

### Step 8: Create Access Key

**Command**

```bash
aws $EP iam create-access-key --user-name Analyst_khadijah
```

**Result**

An access key was created successfully for **Analyst_khadijah**.

**Figure 8:** Create Access Key

![](<STEP 4.1.png>)

**Security Note**

The secret access key is not included in this report. In real cloud environments, access keys should never be shared publicly or stored in plain text.

---

### Step 9: List Access Keys

**Command**

```bash
aws $EP iam list-access-keys --user-name Analyst_khadijah
```

The command lists the access keys created for the Analyst account.

**Figure 9:** List Access Keys

![](<STEP 4.2.png>)

---

### Step 10: Rotate and Deactivate Access Key

**Command**

```bash
aws $EP iam update-access-key \
--user-name Analyst_khadijah \
--access-key-id xxxxxxxxxxxx \
--status Inactive
```

**Result**

The access key status changed to **Inactive**, demonstrating access key rotation.

**Figure 10:** Deactivate Access Key

![](<STEP 4.3.png>)

---

# Session B: Kubernetes RBAC

## Task 5: Create Local Kubernetes Cluster

**Command**

```bash
kind create cluster --name ccse-lab1

kubectl cluster-info --context kind-ccse-lab1

kubectl get nodes
```

**Result**

The Kubernetes cluster **ccse-lab1** was created successfully and the node status was verified.

**Figure 11:** Create Kubernetes Cluster

![](<STEP CLUSTER.png>)

---

## Task 6: Separate Environments with Namespaces

**Command**

```bash
kubectl create namespace dev

kubectl create namespace prod

kubectl get namespaces
```

**Result**

The **dev** and **prod** namespaces were created successfully.

**Figure 12:** Create Namespaces

![](<STEP 5.png>)

---

## Task 7: Define a Role and Bind It

### Step 11: Create Service Account

**Command**

```bash
kubectl create serviceaccount dev-user -n dev
```

**Result**

The service account **dev-user** was created successfully in the **dev** namespace.

---

### Step 12: Create Pod Reader Role

**Command**

```bash
kubectl create role pod-reader -n dev \
--verb=get,list,watch --resource=pods
```

**Result**

The **pod-reader** Role was created successfully. It allows only **get**, **list**, and **watch** permissions on Pods.

---

### Step 13: Create RoleBinding

**Command**

```bash
kubectl create rolebinding dev-user-binding -n dev \
--role=pod-reader \
--serviceaccount=dev:dev-user
```

**Result**

The RoleBinding **dev-user-binding** successfully assigned the **pod-reader** Role to the **dev-user** service account.

**Figure 13:** Create RoleBinding

![](<STEP 6.png>)

---

## Task 8: Test Access Control

The Kubernetes service account identity was stored in a variable.

```bash
SA=system:serviceaccount:dev:dev-user
```

---

### Test 1: List Pods in Dev

**Command**

```bash
kubectl auth can-i list pods -n dev --as=$SA
```

**Result**

```text
yes
```

**Explanation**

The service account can list Pods in the **dev** namespace because the assigned Role allows this action.

---

### Test 2: Delete Pods in Dev

**Command**

```bash
kubectl auth can-i delete pods -n dev --as=$SA
```

**Result**

```text
no
```

**Explanation**

The service account cannot delete Pods because delete permission was not granted.

---

### Test 3: List Pods in Prod

**Command**

```bash
kubectl auth can-i list pods -n prod --as=$SA
```

**Result**

```text
no
```

**Explanation**

The service account cannot access the **prod** namespace because the Role and RoleBinding apply only to the **dev** namespace.

**Figure 14:** RBAC Permission Testing

![](<STEP 7.png>)

---

## Authentication vs Authorization

The service account successfully passed **authentication** because Kubernetes recognized its identity.

The authorization process then checked the assigned permissions. The account was allowed to list Pods in the **dev** namespace but denied permission to delete Pods and access the **prod** namespace because those permissions were not granted.

---

## RBAC Verification

**Command**

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

The output confirmed that the **dev-user-binding** RoleBinding correctly connects the **dev-user** service account to the **pod-reader** Role.

**Figure 15:** RBAC Verification

![](<step 8.png>)

---

# Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Attaching policies to groups makes permission management easier and more efficient. When multiple users require the same permissions, the policy only needs to be attached to the group once. Any changes made to the group's permissions are automatically applied to all members, reducing administrative effort and minimizing configuration errors.

---

### Q2. What is the difference between an IAM User and an IAM Role?

An **IAM User** is a permanent identity created for a person or application and uses long-term credentials such as passwords or access keys. An **IAM Role** is a temporary identity that provides short-term permissions and can be assumed when needed without requiring permanent credentials.

---

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

The **Analyst_khadijah** account demonstrates the principle of least privilege because it is assigned only the **AmazonS3ReadOnlyAccess** policy. If the account is compromised, the attacker can only read Amazon S3 data and cannot modify resources, create users, or change IAM policies. This limits the potential damage and reduces the blast radius of the attack.

---

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?

A **Role** defines the actions that are allowed within a namespace, such as **get**, **list**, and **watch** Pods. A **RoleBinding** assigns those permissions to a specific user, group, or service account. In this lab, the **pod-reader** Role grants read permissions, while the **dev-user-binding** RoleBinding assigns those permissions to the **dev-user** service account.

---

### Q5. Why did the developer service account fail to access the `prod` namespace, and which security principle does that demonstrate?

The developer service account could not access the **prod** namespace because its Role and RoleBinding were created only in the **dev** namespace. This demonstrates the **Principle of Least Privilege**, where users are granted only the minimum permissions required to perform their tasks.

---

# Security Best-Practices Checklist

- [x] Root user is not used for daily tasks because a dedicated administrator account was created.
- [x] Permissions are assigned through IAM Groups instead of directly to users.
- [x] A least-privilege read-only user (**Analyst_khadijah**) was created.
- [x] Access keys were created, listed, and deactivated to demonstrate key rotation.
- [x] Kubernetes RBAC successfully blocked unauthorized actions in the **dev** and **prod** namespaces.

---

# Challenges Encountered

1. Configuring LocalStack to work correctly with the AWS CLI.
2. Creating the Kubernetes cluster using Kind.
3. Understanding Kubernetes RBAC permissions and access control.

---

# Lessons Learned

- Learned how to configure a local cloud environment using Docker and LocalStack.
- Understood the importance of the Principle of Least Privilege.
- Learned how IAM Users, Groups, Policies, and Roles work together.
- Gained hands-on experience configuring Kubernetes RBAC.
- Understood the difference between authentication and authorization.

---

# References

1. LocalStack. (n.d.). *LocalStack Documentation*. https://docs.localstack.cloud

2. Kubernetes. (n.d.). *Role-Based Access Control (RBAC)*. https://kubernetes.io/docs/reference/access-authn-authz/rbac/

3. Kali Linux. (n.d.). *Kali Linux Documentation*. https://www.kali.org/docs/

4. Amazon Web Services. (n.d.). *AWS Identity and Access Management (IAM) Documentation*. https://docs.aws.amazon.com/iam/

5. Prof. Dr. Shahrulniza Musa. (2026). *IKB42603 Cloud Computing Security Essentials Lab Manual: Lab 1 – Cloud Account Security, Identity & Access Management*. Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT).

---

# Conclusion

This lab successfully demonstrated cloud identity management and access control using LocalStack IAM and Kubernetes RBAC. IAM Groups and Policies were used to implement the Principle of Least Privilege, while secure credential management was demonstrated through access key creation and rotation. Kubernetes RBAC effectively enforced authorization by allowing only permitted actions within the assigned namespace. Overall, the lab provided practical experience in implementing secure cloud identity and access management in a local cloud environment.