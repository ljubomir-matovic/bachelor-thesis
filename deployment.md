# Deployment of Dask cluster on Kubernetes cluster

 Dask cluster is designed to handle distributed computing tasks efficiently by dividing the workload among multiple nodes. The cluster is composed of two main components: the **scheduler** and the **workers**. Here’s a detailed look at their roles and interactions:

Certainly! Here’s an explanation of the components of a Dask cluster, focusing on the role of the scheduler and workers:

## Understanding the Components of a Dask Cluster: Scheduler and Workers

A Dask cluster is designed to handle distributed computing tasks efficiently by dividing the workload among multiple nodes. The cluster is composed of two main components: the **scheduler** and the **workers**. Here’s a detailed look at their roles and interactions:

1. **Scheduler**

**Role:** The Dask scheduler is the central coordinator of the cluster. Its primary function is to manage and schedule the execution of computational tasks.

**Responsibilities:**

- **Task Scheduling:** The scheduler receives tasks from the client application and determines how to distribute these tasks across the available worker nodes.
- **Task Coordination:** It maintains a record of the task dependencies and ensures that tasks are executed in the correct order. The scheduler optimizes task allocation based on the workers' availability and current load.
- **Monitoring:** The scheduler continuously monitors the status of tasks and workers. It manages task retries and handles failures to ensure that computations are completed successfully.

2. **Workers**

**Role:** Dask workers are the nodes that perform the actual computations. They execute the tasks assigned to them by the scheduler.

**Responsibilities:**

- **Task Execution:** Each worker executes tasks in parallel based on the instructions it receives from the scheduler. Workers perform the computations and return results to the scheduler.
- **Resource Management:** Workers manage their own computational resources, such as CPU and memory. They process data chunks and handle intermediate results as required by the tasks.
- **Communication:** Workers communicate with the scheduler and other workers to exchange data and synchronize their operations.

### How They Work Together

1. **Task Submission:** When a client submits a computation to the Dask cluster, the request is sent to the scheduler. This submission includes information about the tasks to be performed and their dependencies.

2. **Task Scheduling:** The scheduler analyzes the tasks and their dependencies. It allocates tasks to workers based on their current availability and computational capacity.

3. **Task Execution:** Workers receive tasks from the scheduler and execute them. They perform the necessary computations and handle any intermediate data processing.

4. **Results Collection:** Once the workers complete the tasks, they send the results back to the scheduler. The scheduler aggregates these results and ensures that all dependencies are resolved.

5. **Completion:** The scheduler informs the client application of the completed computation once all tasks are finished and results are collected.

## Dask operator installation

The package `dask-kubernetes` provides a Dask operator for Kubernetes. `dask-kubernetes` is one of many options for deploying Dask clusters, see [Deploying Dask](https://docs.dask.org/en/stable/deploying.html#distributed-computing) in the Dask documentation for an overview of additional options.

I installed Dask operator via Helm.

```sh
helm install --repo https://helm.dask.org --create-namespace -n <your_namespace> --generate-name dask-kubernetes-operator
```

The Dask Operator is a set of custom resources and a controller that runs on your Kubernetes cluster and allows you to create and manage your Dask clusters as Kubernetes resources. Creating clusters can either be done via the [Kubernetes API with kubectl](https://kubernetes.dask.org/en/latest/operator_resources.html) or the [Python API with KubeCluster](https://kubernetes.dask.org/en/latest/operator_kubecluster.html).

I used Kubernetes API with kubectl approach.

## YAML configuration

```yaml
# dask-cluster.yaml
apiVersion: kubernetes.dask.org/v1
kind: DaskCluster
metadata:
  name: <your_cluster_name>
  namespace: <your_namespace>
spec:
  worker:
    replicas: 2 # specifying number of workers
    spec:
      volumes:
        - name: <volume-name> # you can use volumes for storing .csv files
          hostPath:
            path: <your-path>
            type: DirectoryOrCreate
      containers:
        - name: worker
        image: "ghcr.io/dask/dask:latest"
        imagePullPolicy: "IfNotPresent"
        env:
          - name: EXTRA_CONDA_PACKAGES # adding extra packages
            value: "optuna optuna-integration scikit-learn"
        args:
          - dask-worker
          - --name
          - $(DASK_WORKER_NAME)
          - --dashboard
          - --dashboard-address
          - "8788"
        ports:
          - name: http-dashboard
            containerPort: 8788
            protocol: TCP
  scheduler:
    spec:
      containers:
      - name: scheduler
        image: "ghcr.io/dask/dask:latest"
        imagePullPolicy: "IfNotPresent"
        env:
          - name: EXTRA_CONDA_PACKAGES # adding extra packages
            value: "optuna optuna-integration scikit-learn"
        args:
          - dask-scheduler
        ports:
          - name: tcp-comm
            containerPort: 8786 # you can change port
            protocol: TCP
          - name: http-dashboard
            containerPort: 8787 # you can change port
            protocol: TCP
        readinessProbe:
          httpGet:
            port: http-dashboard
            path: /health
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            port: http-dashboard
            path: /health
          initialDelaySeconds: 15
          periodSeconds: 20
    service:
      type: NodePort
      selector:
        dask.org/cluster-name: <your_cluster_name>
        dask.org/component: scheduler
      ports:
      - name: dask-tcp-comm
        protocol: TCP
        port: 8786 # you can change port
        targetPort: "tcp-comm"
      - name: dask-http-dashboard
        protocol: TCP
        port: 8787 # you can change port
        targetPort: "http-dashboard"
```

## Creating Dask cluster

You can create Dask cluster applying next command:

```sh
kubectl apply -f dask-cluster.yaml
```

If you change number of workers, use previous command to apply that.

## List Dask cluster, pods and services

```sh
kubectl get daskclusters -n <namespace>

kubectl get pods -n <namespace>

kubectl get svc -n <namespace>
```

## Deleting Dask cluster

```sh
kubectl delete daskcluster <cluster_name> -n <namespace>
```

## Changing Dask scheduler NodePort ports

```sh
kubectl patch service <sceduler-name> -n <namespace> -p '{"spec":{"ports":[{"name":"dask-tcp-comm","protocol":"TCP","port":8786,"targetPort":"tcp-comm","nodePort":<new-node-port>},{"name":"dask-http-dashboard","protocol":"TCP","port":8787,"targetPort":"http-dashboard","nodePort":<new-node-port>}]}}'
```
