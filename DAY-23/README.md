
# Kubernetes Certificate Signing Request (CSR) Creation and Approval

This guide will walk you through the steps to generate a private key, a Certificate Signing Request (CSR), create a Kubernetes CertificateSigningRequest, approve it, retrieve the issued certificate, and export it.

## Steps

### 1. Generate Private Key and CSR

First, generate a private key and CSR using OpenSSL.

```bash
# Generate the private key
openssl genrsa -out learner.key 2048

# Generate the CSR with Common Name (CN) as 'learner'
openssl req -new -key learner.key -out learner.csr -subj "/CN=learner"
```

### 2. Base64 Encode the CSR

To create a Kubernetes CSR object, you need to base64 encode the CSR.

```bash
# Encode the CSR to base64
cat learner.csr | base64 | tr -d '\n'
```

### 3. Create Kubernetes CertificateSigningRequest (CSR)

Create a YAML file named `learner-csr.yaml` with the base64-encoded CSR content in the `request` field.

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: learner
spec:
  request: <BASE64_ENCODED_CSR>  # Replace this with the actual base64 encoded CSR
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
  expirationSeconds: 604800  # 1 week (604800 seconds)
```

Apply the CSR:

```bash
kubectl apply -f learner-csr.yaml
```

### 4. Approve the CSR

Approve the CertificateSigningRequest (CSR) using the following command:

```bash
kubectl certificate approve learner
```

### 5. Retrieve the Issued Certificate

Once the CSR is approved, retrieve the issued certificate and decode it:

```bash
# Decode and save the issued certificate to learner.crt
kubectl get csr learner -o jsonpath='{.status.certificate}' | base64 --decode > learner.crt
```

### 6. Export the Issued Certificate in YAML

If you want to export the full CertificateSigningRequest resource, including the issued certificate, to a YAML file:

```bash
# Export the CSR with the certificate
kubectl get csr learner -o yaml > learner-csr.yaml
```

### Summary

- **Private Key**: `learner.key`
- **Certificate Signing Request**: `learner.csr`
- **Issued Certificate**: `learner.crt`

This process creates, approves, and retrieves a signed certificate from Kubernetes, with an expiration of one week.

---

# Kubernetes Role and RoleBinding for Pod Reader Access

This section guides you on how to grant the user `learner` read-only access to Pods in the `default` namespace.

## Steps

### Step 1: Set up `learner` User Credentials

First, set up credentials and context for the `learner` user.

```bash
# Create credentials for the learner user
kubectl config set-credentials learner --client-certificate=/path/to/learner.crt --client-key=/path/to/learner.key

# Create a context for the learner user
kubectl config set-context learner-context --cluster=<your-cluster-name> --namespace=default --user=learner

# Switch to the learner context
kubectl config use-context learner-context
```

### Step 2: Create a Role for Pod Reader Permissions

We'll create a `Role` named `pod-reader` that grants the ability to list, get, and watch pods in the `default` namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Apply the role:

```bash
kubectl apply -f pod-reader-role.yaml
```

### Step 3: Bind `learner` User to the Role

Create a `RoleBinding` to assign the `pod-reader` role to the `learner` user.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: learner  # Name of the user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader  # The Role we are binding
  apiGroup: rbac.authorization.k8s.io
```

Apply the RoleBinding:

```bash
kubectl apply -f pod-reader-rolebinding.yaml
```

### Step 4: Verify the `Role` and `RoleBinding` Creation

Make sure the `Role` and `RoleBinding` have been properly created.

```bash
kubectl get role pod-reader -n default
kubectl get rolebinding read-pods -n default
```

### Step 5: Test Permissions

Now, let's test what the `learner` user can and cannot do.

#### 5.1 Switch to the `learner` Context

Switch to the `learner-context`:

```bash
kubectl config use-context learner-context
```

#### 5.2 Try to Create a Pod

Attempt to create a new pod as the `learner` user:

```bash
kubectl run nginx --image=nginx -n default
```

**Expected Outcome**: You will **not** be able to create a pod due to insufficient permissions.

#### 5.3 List Pods in the Namespace

Now, list the pods in the `default` namespace:

```bash
kubectl get pods -n default
```

**Expected Outcome**: You will **be able to list** the pods, as this is allowed by the `pod-reader` role.

#### 5.4 Try to Create a Deployment

Try to create a deployment as the `learner` user:

```bash
kubectl create deployment nginx-deploy --image=nginx -n default
```

**Expected Outcome**: You will **not** be able to create a deployment due to insufficient permissions.

## Conclusion

The `learner` user has been granted read-only access to Pods in the `default` namespace via the `pod-reader` Role. This user can list and watch pods but cannot create, modify, or delete resources like pods or deployments.

