# AKS Deployment Guide

## Prerequisites

You need to have an Azure account and the following tools installed on your local machine:

1. Install Azure CLI
   - Follow the instructions at [Install Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) to install the Azure CLI on your local machine.

2. Install kubectl
   - Follow the [instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to install kubectl.

3. Copy the `example.env` file to `.env`:
   ```bash
   cd aks-setup
   cp example.env .env
   ```
   Modify the variables in the `.env` file as needed.
4. Load the environment variables from the `.env` file to your current shell session:
   ```bash
   source .env
   ```

## Deploying an AKS Cluster

1. Log in to your Azure account using the Azure CLI:
   ```bash
   az login
   ``` 
   Select your subscription & tenant

2. Set default region:
    ```bash
    az configure --defaults location=$AZURE_REGION
    ```
3. Create a resource group:
    ```bash
    az group create --name $RESOURCE_GROUP
    ```
4. Set default resource group and verify:
    ```bash
    az configure --defaults group=$RESOURCE_GROUP
    az configure --list-defaults
    ```
5. You might need to register resource providers in your Azure subscription. For example, Microsoft.ContainerService is required. Run the following command to verify it:
    ```bash
    az provider show --namespace Microsoft.ContainerService --query registrationState
    ```
    If the output is not "Registered", register the provider:
    ```bash
    az provider register --namespace Microsoft.ContainerService
    ```
6. Create the AKS cluster:
    ```bash
    az aks create \
        --resource-group $RESOURCE_GROUP \
        --name $CLUSTER_NAME \
        --node-count 1 \
        --os-sku Ubuntu \
        --load-balancer-sku standard \
        --nodepool-name sysnodepool \
        --node-osdisk-size 128 \
        --node-vm-size $SYS_NODE_POOL_VM_SIZE \
        --enable-addons azure-keyvault-secrets-provider \
        --enable-oidc-issuer \
        --enable-workload-identity \
        --generate-ssh-keys
    ```
    **Note:** You can adjust the parameters as needed for your specific requirements. A node count of 1 is used here for cost savings, but for production environments, consider using a higher node count for redundancy and load balancing.
7. Add a user node pool to the AKS cluster:
    ```bash
    az aks nodepool add \
    --cluster-name $CLUSTER_NAME \
    --name neo4jpool1 \
    --resource-group $RESOURCE_GROUP \
    --mode User \
    --node-count 3 \
    --node-vm-size $USER_NODE_POOL_VM_SIZE
    ```
    **Note:** Adjust the node count and VM size as per your requirements.
8. Verify the cluster and node pools:
    ```bash
    az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query "agentPoolProfiles[].{Name:name, Mode:mode, Count:count, VMSize:vmSize, provisioningState:provisioningState}" --output table
    ```
9. Get AKS credentials to configure kubectl:
    ```bash
    az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --overwrite-existing
    ```
10. Verify kubectl is configured correctly:
    ```bash
    kubectl get nodes
    ```
11. Create new storage classes for Neo4j with fast persistent (retain) storage for the dbs and standard file storage for backups:
    ```bash
    kubectl apply -f ./templates/data-tlog-azuredisk-storage.yaml
    kubectl apply -f ./templates/backup-azurefile-storage.yaml
    ```
12. Verify that the neo4j-ssd and neo4j-backup storage classes were created:
    ```bash
    kubectl get storageclass
    ```
13. Place your Bloom and GDS license files in the `licenses` directory:
    ```
    licenses/
    ├── bloom.license
    └── gds.license
    ```
14. Create and verify a license secret for Bloom and GDS licenses:
    ```bash
    kubectl create secret  generic --from-file=./licenses/bloom.license --from-file=./licenses/gds.license gds-bloom-license
    kubectl get secret  gds-bloom-license -o yaml
    ```
15. Create namespaces and service accounts for workshop participants:
    ```bash
    for i in {1..$PARTICIPANT_COUNT}; do 
        kubectl create namespace neo4j-$i
        kubectl create serviceaccount neo4j-user-$i -n neo4j-$i
    done
    ```
16. Verify the namespaces and service accounts are created:
    ```bash
    kubectl get namespaces
    for i in {1..$PARTICIPANT_COUNT}; do kubectl get serviceaccount -n neo4j-$i; done
    ```
17. Create backup persistent volume claims (PVCs) for each participant:
    ```bash
    for i in {1..$PARTICIPANT_COUNT}; do
      cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: backup-pvc
      namespace: neo4j-$i
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: neo4j-backup
      resources:
        requests:
          storage: 10Gi
    EOF
    done
    ```
18. Create role bindings to grant permissions to service accounts:
    ```bash
    for i in {1..$PARTICIPANT_COUNT}; do 
        kubectl create rolebinding neo4j-admin \
            --clusterrole=admin \
            --serviceaccount=neo4j-$i:neo4j-user-$i \
            -n neo4j-$i
    done
    ```
19. Copy license secret to all participant namespaces:
    ```bash
    for i in {1..$PARTICIPANT_COUNT}; do 
        kubectl get secret gds-bloom-license -o yaml | \
        sed "s/namespace: default/namespace: neo4j-$i/" | \
        kubectl apply -f -
    done
    ```
20. Verify license secrets are copied to each namespace:
    ```bash
    for i in {1..$PARTICIPANT_COUNT}; do 
        echo "Checking namespace neo4j-$i:"
        kubectl get secret gds-bloom-license -n neo4j-$i
    done
    ```
21. Generate kubeconfig files for each participant (24h duration):
    ```bash
    rm -f user-configs/user-*-kubeconfig.yaml
    
    CLUSTER_SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
    CLUSTER_CA=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
    
    for i in {1..$PARTICIPANT_COUNT}; do 
        TOKEN=$(kubectl create token neo4j-user-$i -n neo4j-$i --duration=24h)
        
        cat > user-configs/user-$i-kubeconfig.yaml << EOF
    apiVersion: v1
    kind: Config
    clusters:
    - name: $CLUSTER_NAME
      cluster:
        certificate-authority-data: $CLUSTER_CA
        server: $CLUSTER_SERVER
    contexts:
    - name: neo4j-context-$i
      context:
        cluster: $CLUSTER_NAME
        namespace: neo4j-$i
        user: neo4j-user-$i
    current-context: neo4j-context-$i
    users:
    - name: neo4j-user-$i
      user:
        token: $TOKEN
    EOF
            
    echo "Created user-configs/user-$i-kubeconfig.yaml"
    done
    ```
    
## Delete AKS Cluster and Resource Group
To delete the AKS cluster and the associated resource group, run the following command:
```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```