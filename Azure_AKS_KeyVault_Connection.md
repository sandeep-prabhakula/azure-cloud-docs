# Creating AKS and KeyVault Resources.
## 1. Creating AKS
- Create an AKS Cluster enabling the Secret Provider CSI Driver, Enabling Workload ID and Enabling OIDC.
## 2. Create an AMD64 Agent pool
- Creation of AMD64 architecture 
```bash
az aks nodepool add \
    --resource-group $RESOURCE_GROUP \
    --cluster-name $CLUSTER_NAME \
    --name amd64public \
    --node-count 1 \
    --enable-node-public-ip \
    --node-vm-size standard_d2as_v6
```
## 3. Create KeyVault
- Create a KeyVault enabling RBAC.


# Connecting AKS with KeyVault using Workload ID
## 1. Environment Variable Setup
- Required variables for setting up AKS and KeyVault connection.
```bash
export SUBSCRIPTION_ID=4b52ac84-45dd-411d-9310-5d864d88ceb9
export RESOURCE_GROUP=azkeyVaultTestRG
export UAMI=azurekeyvaultsecretsprovider-azkeyvaulttestaks
export KEYVAULT_NAME=azKeyVaultTestKeyVault
export CLUSTER_NAME=azKeyVaultTestAKS
az account set --subscription $SUBSCRIPTION_ID
```
## 2. Identity and Permissions
- Creates authorization for the AKS Via Managed Identity to KeyVault
```bash
az identity create --name $UAMI --resource-group $RESOURCE_GROUP

export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group $RESOURCE_GROUP --name $UAMI --query 'clientId' -o tsv)"
export IDENTITY_TENANT=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query identity.tenantId -o tsv)

export KEYVAULT_SCOPE=$(az keyvault show --name $KEYVAULT_NAME --query id -o tsv)
az role assignment create --role "Key Vault Administrator" --assignee $USER_ASSIGNED_CLIENT_ID --scope $KEYVAULT_SCOPE
```
- Fetches the AKS's OIDC_ISSUER_URL for verifying that a request is raised from the AKS Cluster.
```bash
export AKS_OIDC_ISSUER="$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)"

echo $AKS_OIDC_ISSUER
```
## 3. Creating the Kubernetes Service Account
- Identity of the application in K8s Cluster.
- Create different service account for different applications or microservices.
```bash
export SERVICE_ACCOUNT_NAME="workload-identity-sa" # Change the service account name for different applications in same K8s Cluster
export SERVICE_ACCOUNT_NAMESPACE="default"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE
EOF
```
## 4. Establishing the "Trust" (Federated Identity) 
- Create a Trust between AKS Cluster's Pod and EntraID(Azure Login System) using Federation Identity.
- For Every application the following commands need to be executed. 
Example: 
- Federation tells Azure's Login System to trust the Kubernetes Service Account.

- Once the Login System is convinced, it gives the Pod a "Ticket."

- The Pod then takes that Ticket to the Key Vault to prove it has the Managed Identity's permissions.
```bash
export FEDERATED_IDENTITY_NAME="aksfederatedidentity" # change the federation identity name for every new service account creation

az identity federated-credential create --name $FEDERATED_IDENTITY_NAME \ 
--identity-name $UAMI \
--resource-group $RESOURCE_GROUP \
--issuer ${AKS_OIDC_ISSUER} \
--subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}
```

## 5. SecretProviderClass Configuration

- Configure separate "SecretProviderClass" for each of the different application's pod.
- Once you fetch the value from Azure, don't just leave it in a file—take that value and paste it into a native Kubernetes Secret so the rest of the cluster can use it easily.
```bash
cat <<EOF | kubectl apply -f -
# This is a SecretProviderClass example using workload identity to access your key vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-wi # needs to be unique per namespace
spec:
  provider: azure
  secretObjects:                              # THIS SECTION IS REQUIRED
    - secretName: mongo-uri-sync              # The name of the K8s Secret
      type: Opaque
      data:
        - objectName: dburi                   # Matches Azure Key Vault objectName
          key: mongodb_uri                    # The key inside the K8s secret
    - secretName: email-creds-sync
      type: Opaque
      data:
        - objectName: emailid
          key: email_user
        - objectName: password
          key: email_pass
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEYVAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: secret1             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
        - |
          objectName: key1                # Set to the name of your key
          objectType: key
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
EOF
```

## 6. The Deployment (The Pod)

- Chose the Correct SecretProviderClass for the Pod in case of multiple SecretProviderClass.
- Create the Service
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
  namespace: default
spec:
  type: NodePort
  selector:
    app: my-app        # This MUST match the 'app' label in your Deployment's template
  ports:
    - protocol: TCP
      nodePort: 30001  # Port opened on the Node's Public IP
      port: 80         # Internal Cluster port (Internal Service IP)
      targetPort: 8080 # The ACTUAL port your backend code is listening on
EOF
```
- Create a Deployment for the application
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blogbackend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: workload-identity-sa
      containers:
        - name: blog-backend
          ports:
            - containerPort: 8080
          image: sandeepprabhakula/blogging:latest
          env:
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: mongo-uri-sync       # Matches secretName from SPC
                  key: mongodb_uri           # Matches key from SPC
            - name: EMAIL_USERNAME
              valueFrom:
                secretKeyRef:
                  name: email-creds-sync     # Matches secretName from SPC
                  key: email_user            # Matches key from SPC
            - name: EMAIL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: email-creds-sync     # Matches secretName from SPC
                  key: email_pass            # Matches key from SPC
          volumeMounts:
            - name: secrets-store-inline
              mountPath: "/mnt/secrets-store"
              readOnly: true
      volumes:
        - name: secrets-store-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "azure-kvname-wi-backend"
EOF
```

## 7. Verification
- List and View the Secrets, Certificates and Keys
```bash
kubectl exec busybox-secrets-store-inline-wi -- ls /mnt/secrets-store/

kubectl exec busybox-secrets-store-inline-wi -- cat /mnt/secrets-store/ExampleSecret
```

# Documentation URLs
- https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver
- https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-identity-access?tabs=azure-portal&pivots=access-with-a-microsoft-entra-workload-identity
