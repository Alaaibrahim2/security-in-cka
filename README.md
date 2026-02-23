# Kubernetes Authentication: From User Accounts to Service Accounts

This guide explains step by step how authentication works in Kubernetes, starting from creating a user using certificates and ending with ServiceAccount authentication. It also includes commands and expected outputs.

---

## Part 1: Create a User (Certificate-Based Authentication)

### Step 1: Generate a private key

```bash
openssl genrsa -out adam.key 2048
```

Output:

```
Generating RSA private key, 2048 bit long modulus
...
```

---

### Step 2: Create a Certificate Signing Request (CSR)

```bash
openssl req -new -key adam.key -out adam.csr -subj "/CN=adam/O=devops"
```

Explanation:

* CN → username
* O → group

Output:

```
(no output if successful)
```

---

### Step 3: Sign the certificate using the cluster CA

```bash
openssl x509 -req -in adam.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out adam.crt -days 365
```

Output:

```
Signature ok
subject=/CN=adam/O=devops
```

Now we have:

* adam.key
* adam.crt

---

## Part 2: Configure kubeconfig for the User

### Step 4: Add credentials

```bash
kubectl config set-credentials adam \
  --client-key=adam.key \
  --client-certificate=adam.crt \
  --embed-certs=true
```

Output:

```
User "adam" set.
```

---

### Step 5: Create a context

```bash
kubectl config set-context adam \
  --cluster=kind-cluster \
  --user=adam
```

Output:

```
Context "adam" created.
```

---

### Step 6: Use the context

```bash
kubectl config use-context adam
```

Output:

```
Switched to context "adam".
```

---

### Step 7: Test access

```bash
kubectl get pods
```

Output example (if RBAC not configured):

```
Error from server (Forbidden): pods is forbidden
```

This means authentication succeeded but authorization failed.

---

## Part 3: Create RBAC Authorization

### Step 8: Create a role

```bash
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods
```

Output:

```
role.rbac.authorization.k8s.io/pod-reader created
```

---

### Step 9: Bind the role

```bash
kubectl create rolebinding adam-binding \
  --role=pod-reader \
  --user=adam
```

Output:

```
rolebinding.rbac.authorization.k8s.io/adam-binding created
```

---

### Step 10: Test again

```bash
kubectl get pods
```

Output:

```
No resources found
```

Authentication and authorization now work.

---

# Part 4: Service Accounts

ServiceAccounts are used by applications inside the cluster.

---

## Step 11: Create a ServiceAccount

```bash
kubectl create serviceaccount myapp
```

Output:

```
serviceaccount/myapp created
```

---

## Step 12: Verify

```bash
kubectl get sa
```

Output:

```
NAME      SECRETS   AGE
myapp     0         5s
```

---

## Step 13: Create a Pod using this ServiceAccount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  serviceAccountName: myapp
  containers:
  - name: nginx
    image: nginx
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Output:

```
pod/test-pod created
```

---

## Step 14: Verify token inside the Pod

```bash
kubectl exec -it test-pod -- sh
```

Inside the container:

```bash
ls /var/run/secrets/kubernetes.io/serviceaccount
```

Output:

```
ca.crt
namespace
token
```

---

## Step 15: View the token

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

Output:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IjE2In0...
```

This is a JWT token.

---

# Part 5: Authentication Flow for ServiceAccount

### Step 16: Application sends request

```http
Authorization: Bearer <token>
```

---

### Step 17: API Server validation

The API Server:

1. Extracts the token.
2. Decodes header and payload.
3. Checks issuer.
4. Checks expiration.
5. Verifies signature using:

```
/etc/kubernetes/pki/sa.pub
```

---

### Step 18: Identity extracted

Example:

```
system:serviceaccount:default:myapp
```

---

### Step 19: Authorization using RBAC

Example role:

```bash
kubectl create role pod-reader \
  --verb=get,list \
  --resource=pods

kubectl create rolebinding sa-binding \
  --role=pod-reader \
  --serviceaccount=default:myapp
```

---

## Step 20: Test from inside the Pod

```bash
curl https://kubernetes.default.svc
```

If allowed, you will receive a response.

---

# Conclusion

This repository covered:

* User authentication using certificates
* kubeconfig setup
* RBAC
* ServiceAccount authentication
* JWT validation

This is the complete authentication flow in Kubernetes.

---

End of guide.
