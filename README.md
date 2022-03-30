# Securing Secrets using Secret Store CSI driver
Secrets Store CSI Driver for Kubernetes secrets - Integrates secrets stores with Kubernetes via a Container Storage Interface (CSI) volume. 
The Secrets Store CSI Driver **secrets-store.csi.k8s.io** allows Kubernetes to mount multiple secrets, keys, and certs stored in external secrets stores (for example from Azure Key Vault, AWS Secrets Manager etc..) into the Pods as a *volume*. 
Once the Volume is attached, the data in it is mounted into the container's file system. [1]

The below implementation is focused on integrating the Secret stored in Azure Key-Vault with Kubernetes Pods/container using **Azure Key Vault Provider for Secrets Store CSI Driver**

### Azure Key Vault Provider for Secrets Store CSI Driver

Azure Key Vault provider for Secrets Store CSI Driver allows us to get secret contents stored in an Azure Key Vault instance and use the Secrets Store CSI driver interface to mount them into Kubernetes Pods [2] [3]

#### Pre-requisite
* Azure CLI and Kubectl installed 
* Kubernetes cluster with requried access/permissions.

**Note**: the installation steps provided below is for Linux nodes. For Windows nodes, please refer to official documentation.

## Install Secrets Store CSI Driver
We have two ways to deploy the SS CSI Driver in Kubernetes: using Helm and using the Deployment manifest files (YAML). To make is simpler, we would go with deployment via manifest files (YAML) 

1. Get the manifest file from official Kubernetes SIGs Github repository for Secrets Store CSI Driver
git clone https://github.com/kubernetes-sigs/secrets-store-csi-driver.git

2. Run the below kubectl commands to deploy manifest files, from the root directory. For this implementation, I've copied the required manifest files under *deploy* directory. However, I'd recommend to get the latest manifest files from official Github repository:
```
kubectl apply -f deploy/rbac-secretproviderclass.yaml
kubectl apply -f deploy/csidriver.yaml
kubectl apply -f deploy/secrets-store.csi.x-k8s.io_secretproviderclasses.yaml
kubectl apply -f deploy/secrets-store.csi.x-k8s.io_secretproviderclasspodstatuses.yaml
kubectl apply -f deploy/secrets-store-csi-driver.yaml
```
#### If using the driver to sync secrets-store content as Kubernetes Secrets, deploy the additional RBAC permissions required to enable this feature
```
kubectl apply -f deploy/rbac-secretprovidersyncing.yaml
```
#### If using the secret rotation feature, deploy the additional RBAC permissions required to enable this feature
```
kubectl apply -f deploy/rbac-secretproviderrotation.yaml
```

3. Validate the installer is running as expected
```
kubectl get pods -l app=csi-secrets-store -n kube-system
```

We should see the Secrets Store CSI driver pods running on each agent node:
```
csi-secrets-store-rdr2v         3/3     Running   0          1m
csi-secrets-store-mk4rd         3/3     Running   0          1m
```

Also, we should see the following CRDs deployed:
```
kubectl get crd

Output: 
NAME  
secretproviderclasses.secrets-store.csi.x-k8s.io  
secretproviderclasspodstatuses.secrets-store.csi.x-k8s.io
```

## Install Azure Key Vault Provider for Secrets Store CSI Driver

We will install the resources using deployment manifest files (YAML) similar to previous steps.

1. Get the deployment manifest file for Azure KV Provider. Again, for this implementation, I've copied the required manifest files under *azure* directory. However, I'd recommend to get the latest manifest files from official Github repository [4]:
```
kubectl apply -f azure/provider-azure-installer.yaml
```

2. Validate the provider's installer is running as expected
```
kubectl get pods -l app=csi-secrets-store-provider-azure

Output:

NAME                                     READY   STATUS    RESTARTS   AGE
csi-secrets-store-provider-azure-8pvrb   1/1     Running   0          1m
csi-secrets-store-provider-azure-t7re4   1/1     Running   0          1m
```

If using Secrets Store CSI Driver and the Azure Keyvault Provider in a cluster with **Pod Security Policy (PSP)** enabled, review and create the following policy that enables the spec required for Secrets Store CSI Driver and the Azure Keyvault Provider to work:
```
kubectl apply -f azure/pod-security-policy.yaml
```

## How to use with Kubernetes resources
Refer to [Usage.md](./Usage.md)

## Uninstall
We can delete all the components with the following commands:
```
kubectl delete -f deploy/*
kubectl delete -f azure/*
```

### References:
[1] https://github.com/kubernetes-sigs/secrets-store-csi-driver  
[2] https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/  
[3] https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#use-azure-key-vault-with-secrets-store-csi-driver  
[4] https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/deployment/provider-azure-installer.yaml