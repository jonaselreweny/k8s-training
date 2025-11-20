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
3. Load the `.env` file to load the environment variables:
    ```bash
    if [ -f "neo4j-setup/.env" ]; then
        export $(cat neo4j-setup/.env | grep -v '^#' | xargs)
        echo "Environment variables loaded from .env file"
        
        # Also set KUBECONFIG if participant number is set
        if [ ! -z "$PARTICIPANT_NUMBER" ]; then
            export KUBECONFIG="$PWD/neo4j-setup/user-configs/user-$PARTICIPANT_NUMBER-kubeconfig.yaml"
            echo "KUBECONFIG set for participant $PARTICIPANT_NUMBER"
        fi
    else
        echo ".env file not found!"
    fi
    ```
4. The instructor should at this point provide you with a kubeconfig files to connect to the kubernetes cluster. Save these files locally in the `user-configs` directory.
     ```
    neo4j-setup/
     └── user-configs
         ├── user-1-kubeconfig.yaml
         ├── user-2-kubeconfig.yaml
         └── ...
    ```
5. Set the `KUBECONFIG` environment variable to point to your kubeconfig file:
    ```bash
    mkdir -p ~/.kube
    cp neo4j-setup/user-configs/user-$PARTICIPANT_NUMBER-kubeconfig.yaml ~/.kube/config
    ```
6. Verify the connection to the cluster by checking the nodes:
    ```bash
    kubectl get pods
    ```
    This should return a list of pods running in the cluster which at the moment are none.

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

## Steps to Deploy Neo4j with Load Balancer

1. Create a load balancer service with persistent external IP for Neo4j:
    ```bash
    kubectl apply -f ./templates/cluster-lb.yaml
    ```
2. Verify that the service has been created and has an external IP assigned:
    ```bash
    kubectl get services
    ```