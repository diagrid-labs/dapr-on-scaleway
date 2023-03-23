# Dapr on Scaleway Kubernetes

This how-to guide combines [two Dapr docs articles](#resources) and explains how to run Dapr apps on Scaleway Kubernetes. You don't need to follow the three original articles, as this guide contains all the steps necessary.

In this guide:

- An Scaleway Kubernetes cluster will be created.
- Dapr will be installed on the cluster.
- Redis will be installed as the state store.
- Two Dapr applications will be deployed to the cluster.

The NodeJS application has a `neworder` POST endpoint that persists order IDs, and an `order` GET endpoint to retrieve the latest order ID.

The Python application creates the order IDs and calls the `neworder` endpoint of the NodeJS service.


## Prerequisites  

- [Scaleway account](https://console.scaleway.com/register/)
  - [Scaleway API keys for the CLI](https://www.scaleway.com/en/docs/identity-and-access-management/iam/how-to/create-api-keys/)
- [Scaleway CLI](https://www.scaleway.com/en/cli/)
- [Docker](https://docs.docker.com/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://github.com/helm/helm#install)
- [Dapr CLI](https://docs.dapr.io/getting-started/install-dapr-cli/)
- Optional: [VSCode](https://code.visualstudio.com/) with [REST client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)

## 1. Create a Scaleway Kubernetes cluster

1. Create a Kubernetes cluser using the Scaleway CLI:

    ```bash
    scw k8s cluster create name=dapr-scaleway
    ```

    Expected response:

    ```bash
    ID                <CLUSTER_ID>
    Type              kapsule
    Name              dapr-scaleway
    Status            creating
    ...
    ```

2. Install the KubeConfig using the Scaleway CLI:

    ```bash
    scw k8s kubeconfig install <CLUSTER_ID>
    ```

    Expected response:

    ```bash
    Kubeconfig for cluster <CLUSTER_ID> successfully written at / Users/<YOURUSERNAME>/.kube/config
    ```

3. Add a pool with two nodes:

    ```bash
    scw k8s pool create cluster-id=<CLUSTER_ID> name=daprpool node-type=PLAY2-NANO size=2
    ```

    Expected response:

    ```bash
    ID                            <POOL_ID>
    ClusterID                     <CLUSTER_ID>
    CreatedAt                     1 second from now
    UpdatedAt                     1 second from now
    Name                          daprpool
    Status                        scaling
    Version                       1.26.2
    NodeType                      play2_nano
    ...
    ```

4. Verify the connection to the cluster:

    ```bash
    kubectl cluster-info
    ```

    Expected response:

    ```bash
    
    ```

5. Verify the two nodes are deployed:

    ```bash
    kubectl get nodes
    ```

     Expected response:

    ```bash
    NAME                                STATUS   ROLES   AGE   VERSION
    <node1>   Ready    agent   20h   v1.24.9
    <node2>   Ready    agent   20h   v1.24.9
    ```

## 2. Setup Dapr on Scaleway Kubernetes

1. Initialize Dapr on the provisioned Scaleway Kubernetes cluster:

    ```bash
    dapr init --kubernetes --wait
    ```

    Expected response:

    ```bash
    Making the jump to hyperspace...
    Container images will be pulled from Docker Hub
    Deploying the Dapr control plane to your cluster...
    Success! Dapr has been installed to namespace dapr-system. To verify, run `dapr status -k' in your terminal.
    ```

2. Verify the Dapr services are running:

    ```bash
    dapr status -k
    ```

    Expected response:

    ```bash
    NAME                   NAMESPACE    HEALTHY  STATUS   REPLICAS  VERSION  AGE  CREATED
    dapr-sidecar-injector  dapr-system  True     Running  1         1.10.4   1m   2023-03-21 11:04.03
    dapr-sentry            dapr-system  True     Running  1         1.10.4   1m   2023-03-21 11:04.03
    dapr-operator          dapr-system  True     Running  1         1.10.4   1m   2023-03-21 11:04.03
    dapr-placement-server  dapr-system  True     Running  1         1.10.4   1m   2023-03-21 11:04.03
    dapr-dashboard         dapr-system  True     Running  1         0.12.0   1m   2023-03-21 11:04.03
    ```

    Ensure that all services are healthy and running before continuing.

## 3. Add a Redis State Store

1. Add a reference to the bitnami Redis Helm chart:

    ```bash
    helm repo add bitnami https://charts.bitnami.com/bitnami
    ```

    Expected response:

    ```bash
    "bitnami" has been added to your repositories
    ```

2. Update Helm:

    ```bash
    helm repo update
    ```

    Expected response:

    ```bash
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "bitnami" chart repository
    Update Complete. ⎈Happy Helming!⎈
    ```

3. Install Redis:

    ```bash
    helm install redis bitnami/redis --set image.tag=6.2
    ```

    Expected response:

    ```bash
    NAME: redis
    LAST DEPLOYED: Tue Mar 21 11:40:28 2023
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    CHART NAME: redis
    CHART VERSION: 17.8.7
    APP VERSION: 7.0.10

    ** Please be patient while the chart is being deployed **

    Redis can be accessed on the following DNS names from within your cluster:

    redis-master.default.svc.cluster.local for read/write operations (port 6379)
    redis-replicas.default.svc.cluster.local for read-only operations (port 6379)
    ...
    ```

4. Verify the Redis pods are running:

    ```bash
    kubectl get pods
    ```

    Expected response:

    ```bash
    NAME               READY   STATUS    RESTARTS   AGE
    redis-master-0     1/1     Running   0          13m
    redis-replicas-0   1/1     Running   0          13m
    redis-replicas-1   1/1     Running   0          12m
    redis-replicas-2   1/1     Running   0          11m
    ```

5. Ensure you are in the root directory of this repo and run:

    ```bash
    kubectl apply -f ./resources/redis.yaml
    ```

    Expected response:

    ```bash
    component.dapr.io/statestore created
    ```

## 4. Deploy the Node app

1. Ensure you're in the root of the repository and apply the Node app configuration to deploy the Node app:

    ```bash
    kubectl apply -f ./resources/node.yaml
    ```

    Expected response:

    ```bash
    service/nodeapp created
    deployment.apps/nodeapp created
    ```

2. Watch the status of the deployment:

    ```bash
    kubectl rollout status deploy/nodeapp
    ```

    This should eventually result in:

    ```bash
    deployment "nodeapp" successfully rolled out
    ```

3. Get the external IP of the service:

    ```bash
    kubectl get svc nodeapp
    ```

    Expected response:

    ```bash
    NAME      TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
    nodeapp   LoadBalancer   <INTERNAL_IP>   <EXTERNAL_IP>   80:31198/TCP   49m
    ```

4. Verify the service is running using curl or a REST client:

    ```bash
    curl http://<EXTERNAL_IP>/ports
    ```

    Expected response:

    ```json
    {"DAPR_HTTP_PORT":"3500","DAPR_GRPC_PORT":"50001"}
    ```

5. Submit an order to the app using curl or a REST client:

    ```bash
    curl --request POST --data "@sample.json" --header Content-Type:application/json http://<EXTERNAL_IP>/neworder
    ```

6. Confirm that the order ID is persisted using curl or a REST client:

    ```bash
    curl http://<EXTERNAL_IP>/order
    ```

    Expected response:

    ```json
    { "orderId": "42" }
    ```

## 5. Deploy the Python app

1. Ensure you're in the root of the repository and apply the Node app configuration to deploy the Python app:

    ```bash
    kubectl apply -f ./resources/python.yaml
    ```

    Expected response:

    ```bash
    deployment.apps/pythonapp created
    ```

2. Watch the status of the deployment:

    ```bash
    kubectl rollout status deploy/pythonapp
    ```

    This should eventually result in:

    ```bash
    deployment "pythonapp" successfully rolled out
    ```

3. Follow the logs of the Node.js app:

    ```bash
    kubectl logs --selector=app=node -c node -f
    ```

    Expected response:

    ```bash
    Got a new order! Order ID: 44
    Successfully persisted state for Order ID: 44
    Got a new order! Order ID: 45
    Successfully persisted state for Order ID: 45
    Got a new order! Order ID: 46
    Successfully persisted state for Order ID: 46
    ...
    ```

4. Confirm that the latest order ID is persisted using curl or a REST client:

    ```bash
    curl http://<EXTERNAL_IP>/order
    ```

## 6. Clean-up

Follow these steps to remove all the apps, components and cloud resources created in this how-to guide.

1. Navigate to the resources folder and run this command to delete the services and state resources:

    ```bash
    kubectl delete -f .
    ```

    Expected response:

    ```bash
    service "nodeapp" deleted
    deployment.apps "nodeapp" deleted
    service "pythonapp" deleted
    deployment.apps "pythonapp" deleted
    component.dapr.io "statestore" deleted
    ```

2. Delete the Scaleway cluster:

## Resources

1. [Creating and managing a Kubernetes Kapsule with CLI](https://www.scaleway.com/en/docs/containers/kubernetes/api-cli/creating-managing-kubernetes-lifecycle-cliv2/)
2. [Tutorial: Configure state store and pub/sub message broker](https://docs.dapr.io/getting-started/tutorials/configure-state-pubsub/)
2. [Hello Kubernetes](https://github.com/dapr/quickstarts/tree/master/tutorials/hello-kubernetes)

## More information

Read more about hosting [Dapr on Kubernetes](https://docs.dapr.io/operations/hosting/kubernetes/) on the Dapr docs site.

Any questions or comments about this sample? Join the [Dapr discord](https://bit.ly/dapr-discord) and post a message the `#kubernetes` channel.
Have you made something with Dapr? Post a message in the `#show-and-tell` channel, we love to see your creations!