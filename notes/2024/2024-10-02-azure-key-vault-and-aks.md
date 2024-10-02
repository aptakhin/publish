# Azure Key Vault and Azure Kubernetes Service

It's a bit tricky. I tried a few approaches. That one is working in AKS `1.24.*-1.29.*`.

```bash
# Subscription where do we work
export SUBSCRIPTION_ID=5b57c6fd-a955-...
# Resource group where located main resources for the service
export RESOURCE_GROUP=rg-...
# User which will be created in Azure system
export UAMI=finioo-cj-dev-user
export KEYVAULT_NAME=kv-finioo-cj-dev
# K8s cluster name
export CLUSTER_NAME=k8s-dev
# Where the cluster is located
export CLUSTER_RESOURCE_GROUP=rg-openapi-dev-k8s
# Namespace where will be located the secrets for the service
export SERVICE_NAMESPACE=finioo-cj-dev
# Another federated identity. Don't ask :D
export FEDERATED_IDENTITY_NAME="finioo-cj-dev-aks-federated-identity"

az account set --subscription $SUBSCRIPTION_ID

az keyvault create -n $KEYVAULT_NAME -g $RESOURCE_GROUP -l westeurope --enable-rbac-authorization

# Create for test and don't drop the key until the end. We use it in the end of process
az keyvault secret set --vault-name $KEYVAULT_NAME -n example --value test

# Setup updating of secrets every 2 minutes
az aks addon update --name $CLUSTER_NAME --resource-group $CLUSTER_RESOURCE_GROUP -a azure-keyvault-secrets-provider --enable-secret-rotation --rotation-poll-interval 2m

# Enable workload identity for AKS
az aks update --name $CLUSTER_NAME --resource-group $CLUSTER_RESOURCE_GROUP --enable-oidc-issuer
az aks update --name $CLUSTER_NAME --resource-group $CLUSTER_RESOURCE_GROUP --enable-workload-identity

# And create the workload identity
az identity create --name $UAMI --resource-group $RESOURCE_GROUP
export USER_ASSIGNED_CLIENT_ID="$(az identity show -g $RESOURCE_GROUP --name $UAMI --query 'clientId' -o tsv)"
export IDENTITY_TENANT=$(az aks show --name $CLUSTER_NAME --resource-group $CLUSTER_RESOURCE_GROUP --query identity.tenantId -o tsv)

# Add access-permissions
az role assignment create --role "Key Vault Secrets User" --assignee $USER_ASSIGNED_CLIENT_ID --scope "/subscriptions/$SUBSCRIPTION_ID/resourcegroups/$RESOURCE_GROUP/providers/microsoft.keyvault/vaults/$KEYVAULT_NAME"

# Make namespace service account
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_NAMESPACE=$SERVICE_NAMESPACE

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  labels:
    azure.workload.identity/use: "true"
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
EOF

# Create identity
export AKS_OIDC_ISSUER="$(az aks show --resource-group $CLUSTER_RESOURCE_GROUP --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)"

az identity federated-credential create --name $FEDERATED_IDENTITY_NAME --identity-name $UAMI --resource-group $RESOURCE_GROUP --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}

# Add secret provider
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kv-workload-identity
  namespace: ${SERVICE_NAMESPACE}
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEYVAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: example2
          objectType: secret              # object types: secret, key, or cert
          objectVersion: ""               # [OPTIONAL] object versions, default to latest if empty
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
EOF


# Add test pod
cat <<EOF | kubectl apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: busybox-secrets-store-inline-user-msi
  namespace: ${SERVICE_NAMESPACE}
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}
  containers:
    - name: busybox
      image: registry.k8s.io/e2e-test-images/busybox:1.29-1
      command:
        - "/bin/sleep"
        - "10000"
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kv-workload-identity"
EOF

# We can check secret mounted
kubectl describe pod busybox-secrets-store-inline-user-msi -n ${SERVICE_NAMESPACE}

kubectl exec busybox-secrets-store-inline-user-msi -n ${SERVICE_NAMESPACE} -- sh -c 'ls /mnt/secrets-store/'

kubectl exec busybox-secrets-store-inline-user-msi -n ${SERVICE_NAMESPACE} -- sh -c 'cat /mnt/secrets-store/example'

kubectl delete pod busybox-secrets-store-inline-user-msi -n ${SERVICE_NAMESPACE}
```