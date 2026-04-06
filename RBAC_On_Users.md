# RBAC on Users

## 1. Create the AKS Cluster
- enable OIDC
- enable workloadID
- enable secretProviderCSI driver(if Azure Key vault is used)

## 2. Create an Admin Group and add the admin User to the admin group(if admin group not exists)
- Assign the object ID of the Admin Group in the step-3

## 3. Enable AAD to the AKS cluster
- This forbids the Local Accounts.
- This enables easy Revocation: If an employee leaves the company, their access to the cluster is instantly killed the moment their Azure AD account is disabled. With local certificates, you'd have to rotate the entire cluster's security keys to kick them out.
```bash
az aks update \

--resource-group sandeepRG \

--name myAKSCluster \

--enable-aad \

--aad-admin-group-object-ids <your-admin-group-id-created-before> \

--aad-tenant-id <your-tenant-id>
```
## 4. Add admin with Cluster Admin role and other user with User Role
### 4.a Create Access to admin
- This is the mandatory step else even admin cannot do any of the operations.
- ```Azure Kubernetes Service RBAC Cluster Admin```
- Update your IAM of the AKS cluster
### 4.b Create Access to the Users
- This is the mandatory step to provide access to develops
- ```Azure Kubernetes Service Cluster User Role```
- Update IAM of the Cluster.

## 5. Create roles
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: deployment-manager
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  # Only allow create and get (plus list/watch for better UX)
  verbs: ["get", "list", "watch", "create", "update", "patch"]
```

# 6. Create role bindings
```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jonsnow@organization.domain.name" to read pods in the "default" namespace.
# You need to already have a Role named "deployment-manager" in that namespace.
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: jonsnow@codeversechroniclestechblog.onmicrosoft.com # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: deployment-manager # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```
## 7. Deploy Role and RoleBinding
```bash
kubectl apply -f roles.yaml
kubectl apply -f rolebinding.yaml
```
