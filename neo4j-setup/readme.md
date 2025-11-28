# Neo4j on Kubernetes Training
The following instructions will guide you through the steps to connect to a shared Kubernetes cluster and deploy a Neo4j using Helm charts. It assumes that a Kubernetes cluster is already set up and prepared for use and focuses on the steps required to deploy Neo4j.

## Prerequisites

We will use `kubectl` to interact with the kubernetes cluster.

1. Install kubectl
   - Follow the [instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to install kubectl. For example, on Debian-based distributions, you can use:
   ```bash
   sudo apt-get update
   sudo apt-get install -y kubectl
   ```

## Connect to the AKS Cluster and Set Default Context
1. Make a copy of the `example.env` file and name it `.env`:
    ```bash
    cp neo4j-setup/example.env neo4j-setup/.env
    ```
2. Open the `.env` file and set the `PARTICIPANT_NUMBER` variable to your assigned number (e.g., `1`, `2`, etc.)
3. The instructor should at this point provide you with a kubeconfig files to connect to the kubernetes cluster. Save these files locally in the `user-configs` directory.
     ```
    neo4j-setup/
     └── user-configs
         ├── user-1-kubeconfig.yaml
         ├── user-2-kubeconfig.yaml
         └── ...
    ```
4. Load environment variables and set up Kubernetes access:
    ```bash
    if [ -f "neo4j-setup/.env" ]; then
        export $(cat neo4j-setup/.env | grep -v '^#' | xargs)
        echo "Environment variables loaded from .env file"
        
        # Set up Kubernetes access for participant
        if [ ! -z "$PARTICIPANT_NUMBER" ]; then
            # Backup existing kubeconfig if it exists
            if [ -f ~/.kube/config ]; then
                cp ~/.kube/config ~/.kube/config.backup.$(date +%Y%m%d-%H%M%S)
                echo "Backed up existing kubeconfig"
            fi
            
            export KUBECONFIG="$PWD/neo4j-setup/user-configs/user-$PARTICIPANT_NUMBER-kubeconfig.yaml"
            echo "KUBECONFIG set for participant $PARTICIPANT_NUMBER"
        fi
    else
        echo ".env file not found!"
    fi
    ```
    
    **Note**: Your original kubeconfig (if any) is backed up to `~/.kube/config.backup.TIMESTAMP`
5. Verify the connection to the cluster by checking the nodes:
    ```bash
    kubectl get pods
    ```
    This should return a list of pods running in the cluster which at the moment are none.
6. Create secret for the initial Neo4j password (just for demo purposes, use a strong password in production and preferably use SSO):
    ```bash
    kubectl create secret generic neo4jpwd --from-literal=NEO4J_AUTH=$NEO4J_AUTH
    kubectl get secret neo4jpwd -o yaml
    ```
## Configure Neo4j Helm Chart Repository
To deploy Neo4j DBMS on Kubernetes you have to configure the Neo4j Helm chart repository. More information can be found in the [documentation](https://neo4j.com/docs/operations-manual/current/kubernetes/helm-charts-setup/).
1. Configure the Neo4j Helm chart repository:
    ```bash
    helm repo add neo4j https://helm.neo4j.com/neo4j
    helm repo update
    ```
2. Check for available Neo4j Helm chart versions:
    ```bash
    helm search repo neo4j
    ```

## Steps to Deploy Load Balancer with Persistent External IP
To avoid changing IP addresses after the load balancer pod restarts, we will create a load balancer service with a persistent external IP for Neo4j.

1. Create a load balancer service with persistent external IP for Neo4j:
    ```bash
    kubectl apply -f neo4j-setup/templates/cluster-lb.yaml
    ```
2. Verify that the service has been created and has an external IP assigned:
    ```bash
    kubectl get services
    ```
## Create the cluster

1. Deploy the Neo4j core cluster using Helm:
    ```bash
    helm install server1 neo4j/neo4j -f neo4j-setup/templates/neo4j-core-cluster.yaml
    helm install server2 neo4j/neo4j -f neo4j-setup/templates/neo4j-core-cluster.yaml
    helm install server3 neo4j/neo4j -f neo4j-setup/templates/neo4j-core-cluster.yaml
    ```
2. Verify that the pods are running:
    ```bash
    kubectl get pods
    ```
3. Access the Neo4j Browser using the external IP of the load balancer service: