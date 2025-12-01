# Neo4j on Kubernetes Training
The following instructions will guide you through the steps to connect to a shared Kubernetes cluster and deploy a Neo4j using Helm charts. It assumes that a Kubernetes cluster is already set up and prepared for use and focuses on the steps required to deploy Neo4j.

## Prerequisites

We will use `kubectl` to interact with the kubernetes cluster and VS Code with the Neo4j extension to interact with the Neo4j database.

1. Install kubectl
   - Follow the [instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to install kubectl. For example, on Debian-based distributions, you can use:
   ```bash
   sudo apt-get update
   sudo apt-get install -y kubectl
   ```
2. Install the Neo4j for VS Code extension:
   - Open VS Code and go to the Extensions view by clicking on the Extensions icon in the Activity Bar on the side of the window or by pressing `Ctrl+Shift+X`.
   - Search for `Neo4j` in the Extensions view search bar.
   - Find the `Neo4j for VS Code` extension by Neo4j and click the `Install` button.
   - After installation, you may need to reload VS Code to activate the extension.

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
    **Note**: It may take a few moments for the external IP to be assigned. If the `EXTERNAL-IP` column shows `<pending>`, wait a bit and run the command again. Make note of the external IP address assigned to the `neo4j-loadbalancer` service. That IP address will be used to access Neo4j later.

## Create the Cluster

1. Deploy the Neo4j core cluster using Helm:
    ```bash
    helm install server1 neo4j/neo4j -f neo4j-setup/templates/neo4j-core-cluster.yaml --version 2025.9.0
    helm install server2 neo4j/neo4j -f neo4j-setup/templates/neo4j-core-cluster.yaml --version 2025.9.0
    helm install server3 neo4j/neo4j -f neo4j-setup/templates/neo4j-core-cluster.yaml --version 2025.9.0
    ```
2. Verify that the pods are running:
    ```bash
    kubectl get pods
    ```
    **Note**: It may take a few moments for the pods to be in the `Running` state. If any pod shows `ContainerCreating` or `Pending`, wait a bit and run the command again. Once all pods are in the `Running` state, and the `READY` column shows `1/1`, you can proceed to the next step.
3. Access the Neo4j logs on the pods to verify that Neo4j has started:
    ```bash
    kubectl exec server1-0 -- tail /logs/neo4j.log
    ```
    **Note**: You can replace `server1-0` with the name of any of the other Neo4j pods to check their logs as well.
4. Check that the services are running:
    ```bash
    kubectl get services
    ```
    Should look something like this:
    ```
    NAME                TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
    cluster-lb          LoadBalancer   10.0.9.206     51.105.128.238   7687 ...
    server1             ClusterIP      10.0.227.138   <none>           7687 ...
    server1-admin       ClusterIP      10.0.160.71    <none>           6362 ...
    server1-internals   ClusterIP      10.0.97.189    <none>           6362 ...
    server2             ClusterIP      10.0.122.228   <none>           7687 ...
    server2-admin       ClusterIP      10.0.238.177   <none>           6362 ...
    server2-internals   ClusterIP      10.0.195.30    <none>           6362 ...
    server3             ClusterIP      10.0.149.125   <none>           7687 ...
    server3-admin       ClusterIP      10.0.227.22    <none>           6362 ...
    server3-internals   ClusterIP      10.0.147.8     <none>           6362 ...
    ```
5. Access Neo4j using cypher-shell (replace `<password>` with the password you set in the NEO4J_AUTH environment variable):
    ```bash
    kubectl exec -it server1-0 -- bin/cypher-shell -u neo4j -p <password>
    ```
    You should see the cypher prompt:
    ```
    neo4j@neo4j>
    ```
    You can now run cypher commands against your Neo4j cluster.
6. List databases to verify everything is working:
    ```cypher
    SHOW DATABASES YIELD name, type, address, role, writer, currentStatus;
    ```
    You should see output similar to this:
    ```
    +-------------------------------------------------------------------------------------------------------+
    | name     | type       | address                                  | role      | writer | currentStatus |
    +-------------------------------------------------------------------------------------------------------+
    | "neo4j"  | "standard" | "server1.neo4j-1.svc.cluster.local:7687" | "primary" | FALSE  | "online"      |
    | "neo4j"  | "standard" | "server3.neo4j-1.svc.cluster.local:7687" | "primary" | TRUE   | "online"      |
    | "neo4j"  | "standard" | "server2.neo4j-1.svc.cluster.local:7687" | "primary" | FALSE  | "online"      |
    | "system" | "system"   | "server3.neo4j-1.svc.cluster.local:7687" | "primary" | FALSE  | "online"      |
    | "system" | "system"   | "server2.neo4j-1.svc.cluster.local:7687" | "primary" | FALSE  | "online"      |
    | "system" | "system"   | "server1.neo4j-1.svc.cluster.local:7687" | "primary" | TRUE   | "online"      |
    +-------------------------------------------------------------------------------------------------------+
    ```
7. Create a database with the topology of having 3 primaries:
    ```cypher
    CREATE DATABASE mydb IF NOT EXISTS TOPOLOGY 3 PRIMARIES;
    ```
8. Verify the new database has been created:
    ```cypher
    SHOW DATABASES YIELD name, type, address, role, writer, currentStatus;
    ```
9. Exit cypher-shell by running:
    ```cypher
    :exit
    ```
## Access Neo4j from VS Code
1. Open the Neo4j extension in VS Code.
2. Click on the `Add Connection` button.
3. Fill in the connection details:
   - **Name**: A name for your connection (e.g., `Neo4j Cluster`).
   - **URI**: `neo4j://<EXTERNAL-IP>:7687` (replace `<EXTERNAL-IP>` with the external IP address of the load balancer service you noted earlier).
   - **Username**: `neo4j`
   - **Password**: The password you set in the NEO4J_AUTH environment variable.
4. Click `Connect` to establish the connection.
5. You should now be connected to your Neo4j cluster and can run queries directly from VS Code.
6. Select the `mydb` database from the database dropdown in the Neo4j extension to run queries against it.
7. Open the `movies.cypher` file from the `neo4j-setup/cypher/` directory and run the script to load sample data into the `mydb` database.
8. **Optional**: Create a new Cypher file in VS Code and run some sample queries against the `mydb` database, for example:
   ```cypher
   MATCH path = (:Person)-[:ACTED_IN*1..3]-(:Movie)
   RETURN path
   LIMIT 10;
   ```
## Backup Databases in a Cluster
Refer to the [Neo4j Backup and Restore documentation](https://neo4j.com/docs/operations-manual/current/kubernetes/operations/backup-restore/) for detailed instructions on how to perform backups and restores in a Neo4j cluster deployed on Kubernetes. There are several methods available and the choice depends on your specific requirements and setup. The step-by-step guide below is one of the methods available. To facilitate the backup process smoothly in this training, we are using a storage class that supports ReadWriteMany access mode, allowing multiple pods to read and write to the same persistent volume simultaneously.
1. Check out the backup job template in `neo4j-setup/templates/backup-job.yaml` to understand how the backup job is configured. The template is scheduled to run every minute for demonstration purposes; you may want to adjust the schedule according to your needs in a production environment (see [CronJob documentation](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) for more details).
2. Deploy the backup job using Helm:
   ``` bash
   helm install neo4j-backup neo4j/neo4j-admin -f ./neo4j-setup/templates/backup-job.yaml --set backup.databaseNamespace=neo4j-$PARTICIPANT_NUMBER
   ```
3. Verify that the backup job has been scheduled:
   ``` bash
   kubectl get cronjobs
   ```
4. Once the job runs, you can check the status of the backup jobs:
   ``` bash
   kubectl get jobs
   ```
5. Monitor the backup job logs to ensure backups are being created successfully, replacing `<backup-job-name>` with the actual name of the backup job:
   ``` bash
    kubectl logs job/<backup-job-name>
   ```
## Restore a Database from Backup
Refer to the [Neo4j Backup and Restore documentation](https://neo4j.com/docs/operations-manual/current/kubernetes/operations/backup-restore/) for detailed instructions on how to perform restores in a Neo4j cluster deployed on Kubernetes. The step-by-step guide below outlines the process to restore a database, in this case `mydb` from a backup created by the backup job.
1. The backup pod stores the backup files in the PVC named `backup-pvc` under the `/backups` directory. The cluster pods are mounted to the same PVC but unfortunately the Neo4j Helm Chart adds a `/backups` subdirectory by default. Therefore, we need to copy the backup files to a subdirectory named `backups` within the `/backups` mount point in the cluster pods. This issue has been reported to the Neo4j team for resolution in future releases. For now, we can use a temporary pod to organize the backup files:
   ``` bash
   kubectl run backup-organizer --rm -it --restart=Never --image=busybox \
   --overrides='{"spec":{"containers":[{"name":"organizer","image":"busybox","command":["sh","-c","mkdir -p /backups/backups && cp /backups/mydb*.backup /backups/backups/ 2>/dev/null && echo Done && ls -la /backups/backups/"],"volumeMounts":[{"name":"backup-pvc","mountPath":"/backups"}]}],"volumes":[{"name":"backup-pvc","persistentVolumeClaim":{"claimName":"backup-pvc"}}]}}'
   ```
2. Verify that the backup files have been copied to the correct location:
   ``` bash
   kubectl exec -it server1-0 -- ls -la /backups
   ```
3. Before restoring, ensure that the `mydb` database is dropped to avoid conflicts. Either use cypher-shell or the Neo4j VS Code extension to connect to the `system` database and run:
   ``` cypher
   DROP DATABASE mydb IF EXISTS;
   ```
4. Bash into one of the Neo4j pods to perform the restore operation:
   ```bash
   kubectl exec -it server1-0 -- bash
   ```
5. Restore the `mydb` database from the backup files:
   ``` bash
   neo4j-admin database restore mydb --from-path=/backups/ --expand-commands
   ```
6. Run cypher-shell to connect to the `system` database, replacing `<password>` with your Neo4j password:
   ``` bash
   cypher-shell -u neo4j -p <password> --database=system
   ```
7. Create the `mydb` database again using cypher-shell or the Neo4j VS Code extension:
   ``` cypher
   CREATE DATABASE mydb IF NOT EXISTS OPTIONS { existingData: 'use', seedUri: 'file:///backups/'};
   ```
   **Note**: The `seedUri` option points to the directory where the backup files are located within all the cluster pods.
8. Verify that the `mydb` database has been restored successfully:
   ``` cypher
   SHOW DATABASES YIELD name, type, address, role, writer, currentStatus;
   ```
   and test some queries against the `mydb` database to ensure data integrity.
   ```cypher
    MATCH (n) RETURN n LIMIT 10;
   ```
9. Exit cypher-shell and the pod bash shell:
   ``` bash
   :exit
   exit
   ```  
