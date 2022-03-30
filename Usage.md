# How it works ?
To use the Azure Key-Vault secret in Kubernetes Resources like Pod, we will first setup the Key-Vault with required secret (in case it doesn't exist) 

#### Prerequisite
1. Secrets Store CSI Driver is deployed on Kubernetes Cluster
2. Azure Key Vault Provider for Secrets Store CSI Driver is deployed on Kubernetes Cluster

## Create Keyvault and set Secrets

Login as a user and set the appropriate subscription ID
```
az login
az account set -s "${SUBSCRIPTION_ID}"
```
Set the Environment variables, used by commands later
```
export SUBSCRIPTION_ID="<subscription-id>"
export TENANT_ID="<tenant-id>"

export KEYVAULT_RESOURCE_GROUP=<keyvault-resource-group>
export KEYVAULT_LOCATION=<keyvault-location>
export KEYVAULT_NAME=secret-store-$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 10 | head -n 1)
```

Create an Azure Keyvault instance:
```
az group create -n ${KEYVAULT_RESOURCE_GROUP} --location ${KEYVAULT_LOCATION}
az keyvault create -n ${KEYVAULT_NAME} -g ${KEYVAULT_RESOURCE_GROUP} --location ${KEYVAULT_LOCATION}
```

Add a secret to your Keyvault:
```
az keyvault secret set --vault-name ${KEYVAULT_NAME} --name testsecret --value "value@123"
```

## Create an identity on Azure and set access policies
In this implementation, we will be using the **Service Principal** auth mode for accessing the Key Vault instance we just created.
```
# Create a service principal to access keyvault
export SERVICE_PRINCIPAL_CLIENT_SECRET="$(az ad sp create-for-rbac --skip-assignment --name http://secrets-store-test --query 'password' -otsv)"
export SERVICE_PRINCIPAL_CLIENT_ID="$(az ad sp show --id http://secrets-store-test --query 'appId' -otsv)"
```
Set the access policy for keyvault objects:
```
az keyvault set-policy -n ${KEYVAULT_NAME} --secret-permissions get --spn ${SERVICE_PRINCIPAL_CLIENT_ID}
```

## Create the Kubernetes Secret with credentials
Create the Kubernetes secret with the service principal credentials:
```
kubectl create secret generic secrets-store-creds --from-literal clientid=${SERVICE_PRINCIPAL_CLIENT_ID} --from-literal clientsecret=${SERVICE_PRINCIPAL_CLIENT_SECRET}
kubectl label secret secrets-store-creds secrets-store.csi.k8s.io/used=true
```
**Note**: There is a known down-side with this approach, as Service Principal credentials(client id & client secret) need to be created as a kubernetes Secret which is stored as plaintext in etcd. However, this is the only supported way to connect to Azure Key Vault from a non Azure environment.

## Deploy SecretProviderClass 

Create **SecretProviderClass** in the cluster that contains all the required parameters, please replace values for "keyvaultName" and "tenantId":
```
kubectl apply -f azure/secret-provider-class.yaml
```
## Use the secret in Kubernetes Pod
Create the Pod with Volume referencing the secrets-store.csi.k8s.io driver:
```
kubectl apply -f azure/secrets-store-pod.yaml
```

Once the Pod is started, we can validate the new mounted content at the volume path specified in the deployment yaml.
```
kubectl exec busybox-secrets-store -- ls /mnt/secrets-store/

Output: 
testsecret
```
```
kubectl exec busybox-secrets-store -- cat /mnt/secrets-store/testsecret

Output: 
value@123
```
