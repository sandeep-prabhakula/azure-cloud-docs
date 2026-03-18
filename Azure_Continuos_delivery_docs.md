# Note
1. Keep the Self Host Linux Agent pool VM running for successful CI Job because Azure does not support Azure Hosted Agents for Pipelines.
2. Make sure VM and VMSS instances have same architecture.
# Connection between Git Repo and Argo CD
## 1. Create an AKS Service
- Choose Regions which are not prone to resource issue.
### 1.1 create Agent Pool 
- Check the box to enable public IP for each node(Mandatory setting).
## 2. K8s cluster login
```bash
        az aks get-credentials --name <AKS_name> --overwrite-existing --resource-group <RG_name>
```
## 3. Install ArgoCD
- Search for installation docs or follow below commands
```bash
    kubectl create namespace argocd
    kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
## 4. Configure ArgoCD
### 4.1 Get ArgoCD Credentials
- Get password from the secrets and decode
```bash
    kubectl get secrets -n argocd
    kubectl edit secret argocd-initial-admin-secret -n argocd
    # decode password
    echo dDFadjBtOUpQSVVydDNPOA==| base64 --decode
    # decoded password = t1Zv0m9JPIUrt3O8
```
### 4.2 Access ArgoCD GUI
- Fetch all argoCD services
```bash
    kubectl get svc -n argocd
```
- update the 'argocd-server' with changing the type from "ClusterIP" to "NodePort"
```bash
    kubectl edit svc argocd-server -n argocd
```
- Fetch the external IP of the VMSS instance copy the external IP and the port the node port is open at.
```bash
    kubectl get nodes -o wide
```
- Add the inbound rule in VMSS instance for the port number of NodePort.
- Azure Portal Path - VMSS -> get into the VMSS -> Instances -> select the Instance you want to open the port for -> Network Settings -> Create Port Rule -> Inbound port rule
- Setup "Destination Port" as same as the NodePort's PortNumber
- Keep Priority as 100.
- Access the ArgoCD GUI on Public IP of VMSS's Instance and port number of the NodePort of ArgoServer.

## 5. Login to ArgoCD GUI
- Choose HTTP PortNumber to access ArgoCD GUI 
- username: admin
- password: < which is fetched previously from the Secrets.> bliEsk2WSPQ4P8uV

## 6. Connect ArgoCD K8s Cluster with Azure Devops Repo
- create a personal access token and connect ArgoCD and Repo via access Token.
- Access Token = <YOUR_PERSONAL_ACCESS_TOKEN>
- Use the access token to replace the organization name with access token in the repo URL
From this URL -> "https://codeversechroniclestechblogs@dev.azure.com/codeversechroniclestechblogs/cicdTestingProject/_git/cicdTestingProject"->
To this url -> "https://<YOUR_PERSONAL_ACCESS_TOKEN>@dev.azure.com/codeversechroniclestechblogs/cicdTestingProject/_git/cicdTestingProject"
- Click on Connect button and make sure the connection is successful.

# Connection Between ArgoCD and AKS
## 1. Create the App in ArgoCD GUI
- choose automatic sync option.
- Choose a path where all k8s yaml files are stored. 

## 2. Add a Shell Script Stage to update the K8s Specifications
- make sure the repo url matches the url in the shell script
## 3. Create an Access key to access the container registry by ArgoCD
- in this case:
- secret-name='my-secret'
- namespace='default'
- container-registry-name='azcicdtesting'
- The below 2 parameters are found in container registry access keys when admin is checked.
- username='azCICDtesting'
- password='5aEI8ugdB44L0zibzh3XkBVjNbJ9Mk5t6YokCalTdWHHuRNzjzOwJQQJ99CCAC8vTInEqg7NAAACAZCRGcqS'
```bash
    kubectl create secret docker-registry my-secret \
    --namespace default \
    --docker-server=azcicdtesting.azurecr.io \
    --docker-username=azCICDtesting \
    --docker-password=5aEI8ugdB44L0zibzh3XkBVjNbJ9Mk5t6YokCalTdWHHuRNzjzOwJQQJ99CCAC8vTInEqg7NAAACAZCRGcqS
```
- Add below code in vote-deployment.yaml file. Below code should be in same level of 'Container' key
```yaml
    imagePullSecrets:
    - name: my-secret
```
## 4. Open the Ports for the application
- Goto the VMSS instance's inbound port rule and add port for application service 
```bash
    kubectl get svc 
```
- choose the port of the NodePort Type and add to inbound rule of VMSS instance.


# RCA: Compatibility issue between the VM and VMSS instances.
- sometimes VM which we use for Pipelines may be AMD64 and VMSS instances may be ARM64 in that case there's a compatibility issue. 
- Both the VM and VMSS instance should have same architecture either ARM64 or AMD 64 
- if there's no possibility of matching the architecture run the following command to make the VM used for CI Jobs compatible with VMSS instances.
```bash
    docker run --privileged --rm tonistiigi/binfmt --install arm64
```
