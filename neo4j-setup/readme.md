# Neo4j on k8s Deployment Guide

To deploy Neo4j DBMS on Kubernetes, you have to configure the Helm chart repository. That has been prepared for you in this training in accordance with the [documentation](https://neo4j.com/docs/operations-manual/current/kubernetes/helm-charts-setup/).

## Prerequisites

You need to have an Azure account and the following tools installed on your local machine:

1. Install Azure CLI
   - Follow the instructions at [Install Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) to install the Azure CLI on your local machine.

2. Install kubectl
   - Follow the [instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to install kubectl. 
   ```bash
   sudo apt-get update
   sudo apt-get install -y kubectl
   ```


## Steps to Deploy Neo4j with Load Balancer

1. Set the current namespace to `neo4j-<your-number>`:
    ```bash
    kubectl config set-context --current --namespace=neo4j-<your-number>
    ```
2. Create a load balancer service with persistent external IP for Neo4j:
    ```bash
    kubectl apply -f ./templates/cluster-lb.yaml
    ```
3. Verify that the service has been created and has an external IP assigned:
    ```bash
    kubectl get services
    ```