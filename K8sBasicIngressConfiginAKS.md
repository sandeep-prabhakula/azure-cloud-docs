# 1. Login to your azure account via CLI
``` bash
    az login
```
- Verify the credentials after logging in 
```bash
    az account show
```
# 2. Create the AKS Infra 
- Terraform (Recommended)
- Azure CLI
- Azure Portal

# 3. login to the AKS cluster
```bash
    az aks get-credentials --name my-aks-cluster-name --resource-group my-RG --overwrite-existing
```

# 4. Install the ingress controller into your VMSS
```bash
    
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace ingress-basic \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

- Breakdown of the Command
    1. helm install ingress-nginx: This tells Helm to install a new "release" and name it ingress-nginx.

    2. ingress-nginx/ingress-nginx: This specifies the chart to install. The first part is the repository name, and the second is the specific package (the NGINX controller).

    3. --create-namespace: A convenience flag. If the namespace you specify doesn't exist yet, Helm will create it for you.

    4. --namespace ingress-basic: This tells Helm to tuck all the resources (pods, services, config maps) into a specific logical folder in your cluster called ingress-basic. This keeps your ingress resources isolated from your actual applications.

- The Azure-Specific Configuration
    1. The --set flag is used to override default configurations in the Helm chart.

    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

    2. When you run this on Azure Kubernetes Service (AKS), the ingress controller creates an Azure Load Balancer in your Azure portal.

- The Problem: 
    By default, the Azure Load Balancer might try to "ping" the root path (/) to see if the controller is alive. If your app doesn't respond there, the load balancer thinks the service is down.

- The Solution: 
    This annotation tells the Azure Load Balancer to check the /healthz endpoint instead. This is a built-in health check path for NGINX that always returns a "healthy" signal if the controller is running properly.

- In Simple Terms
    You are telling Kubernetes: "Go get the NGINX software, put it in a room called 'ingress-basic', and tell the Azure Load Balancer that if it wants to check if we're awake, it should knock on the door labeled '/healthz'."
