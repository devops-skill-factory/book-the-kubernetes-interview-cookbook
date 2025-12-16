# Kubernetes Scheduler

The [Kubernetes scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) is a core control plane component that **assigns newly created pods (which have no node specified) to suitable nodes in the cluster, ensuring efficient workload distribution**.

It continuously watches for unscheduled pods via the API server and evaluates nodes based on various constraints, such as resource requirements, hardware/software policies, affinity/anti-affinity rules, taints/tolerations, and data locality.

The scheduling process involves two main phases: **filtering** (eliminating nodes that cannot run the pod) and **scoring** (ranking feasible nodes to select the best one).

Once the optimal node is chosen, the scheduler binds the pod to that node by updating the pod specification, allowing the kubelet on the node to run it.

## Practice

In these practice exercises by manipulating node labels and Pod specifications, you can directly observe the `kube-scheduler`'s decision-making process.

Since Minikube often runs as a single-node cluster, we'll focus on how to simulate a multi-node environment's scheduling dynamics on that one node, or explicitly force decisions.

Start Minikube:
```bash
minikube start
```

### Exercise 1: Resource Requirements and Unschedulable Pods

This exercise focuses on the **Filtering (Predicates)** phase, where the scheduler determines if a node has the necessary resources.

### The Goal

To create a Pod that is *deliberately* too large to fit on the Minikube node and observe the scheduler's behavior.

### Steps

1.  **Check Current Node Capacity:**
    First, find your Minikube node's available resources.

    ```bash
    kubectl describe node minikube | grep Allocatable: -A 5
    ```

    You will see something like this (values will vary):
    ```bash
    Allocatable:

     cpu:                8  # CPU 8 cores
     ephemeral-storage:  479492784Ki # Approximately 457.28 Gi
     hugepages-1Gi:      0
     hugepages-2Mi:      0
     memory:             14244940Ki # Approximately 13.58 Gi (Gibibytes)
    ```

2.  **Define an Over-Sized Pod (`too-big-pod.yaml`):**
    Create a Pod definition that requests more memory or CPU than your Minikube node has available (based on your check in step 1). For example, if your node has 1800Mi allocatable, request 3000Mi.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: too-big-pod
    spec:
      containers:
      - name: large-container
        image: busybox
        command: ["sleep", "3600"]
        resources:
          requests:
            memory: "15000000Ki" # Requesting an impossible amount
            cpu: "2"
    ```

3.  **Apply and Observe:**

    ```bash
    kubectl apply -f too-big-pod.yaml
    kubectl get pods
    ```

    The Pod will remain in the **Pending** state.
    ```bash
    NAME          READY   STATUS    RESTARTS   AGE
    too-big-pod   0/1     Pending   0          71s
    ```

4.  **Analyze the Scheduler's Decision:**
    Check the Pod's events to see why the scheduler could not place it.

    ```bash
    kubectl describe pod too-big-pod
    ```

    Look for an event with the reason **`FailedScheduling`** and a message indicating **"0/1 nodes are available: 1 Insufficient memory."** This clearly demonstrates the scheduler's filtering mechanism.
    
    ```bash
    Events:
    Type     Reason            Age   From               Message
    ----     ------            ----  ----               -------
    Warning  FailedScheduling  9s    default-scheduler  0/1 nodes are available: 1 Insufficient memory. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
    ```

5.  **Clean Up:**

    ```bash
    kubectl delete pod too-big-pod
    ```

### Exercise 2: Taints and Tolerations for Node Dedication

This exercise focuses on using **Taints and Tolerations** to simulate dedicating a node for specialized workloads, demonstrating a crucial **Filtering** step.

### The Goal

To prevent general Pods from running on the node and only allow a "special" Pod with the correct toleration to be scheduled.

### Steps

1.  **Taint the Minikube Node (Repel General Pods):**
    Taint the node with a key-value pair and the `NoSchedule` effect. This immediately prevents any new Pods without a matching toleration from being scheduled.

    ```bash
    kubectl taint nodes minikube dedicated=special-workload:NoSchedule
    ```

    *Result: node/minikube tainted*

2.  **Test with a General Pod (Verify Filtering):**
    Deploy a simple Pod *without* any tolerations (`general-pod-taint.yaml`).

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: general-app
    spec:
      containers:
      - name: nginx-container
        image: nginx
    ```

    Apply the YAML, then check its status and events. It should be **Pending** and the `kubectl describe pod` command will show that the node had an **"unmet constraint"** (the taint).

    You should see something like this:
    ```bash
    Events:
    Type     Reason            Age   From               Message
    ----     ------            ----  ----               -------
    Warning  FailedScheduling  22s   default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {dedicated: special-workload}. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling
    ```

3.  **Define a Special Pod with Toleration (Bypass Filtering):**
    Create a new Pod that includes a `toleration` matching the taint you applied in step 1.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: special-app
    spec:
      containers:
      - name: special-container
        image: busybox
        command: ["sleep", "3600"]
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "special-workload"
        effect: "NoSchedule"
    ```

4.  **Apply and Observe (Verify Successful Scheduling):**
    Apply the special Pod. It should be scheduled and move to the **Running** state, proving that the scheduler allowed it to bypass the taint filter.

    ```bash
    kubectl apply -f special-pod-taint.yaml
    kubectl get pods -o wide
    ```

    You should see:
    ```bash
    NAME          READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
    general-app   0/1     Pending   0          4m34s   <none>       <none>     <none>           <none>
    special-app   1/1     Running   0          32s     10.244.0.5   minikube   <none>           <none>
    ```

5.  **Clean Up:**

    ```bash
    kubectl delete pod general-app special-app
    # **Crucially, remove the taint from the node**
    kubectl taint nodes minikube dedicated:NoSchedule-
    ```

## Exercise 3: Pod Anti-Affinity for High Availability

This exercise focuses on **Inter-Pod Spreading**, using **Pod Anti-Affinity** to force the scheduler to think about *where* other related Pods are running.

### The Goal

To simulate an environment with two "nodes" (by using two Minikube VMs or by using a multi-node Minikube setup) and ensure that replicas of an application are always placed on different nodes for high availability.

### Steps (Using Multi-Node Minikube)

*For a single-node Minikube, this will result in the Pod remaining Pending, which also demonstrates the principle.*

1.  **Start Minikube with Multiple Nodes:**
    For the full anti-affinity experience, you need multiple nodes.

    ```bash
    minikube start --nodes 2
    kubectl get nodes
    ```

2.  **Define the Deployment with Pod Anti-Affinity (`anti-affinity-deploy.yaml`):**
    This deployment has 2 replicas and a **required** anti-affinity rule. It says: "Do not schedule me on any node that already has a Pod with the label `app: my-web-app`." The `topologyKey: "kubernetes.io/hostname"` means "different nodes."

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web-app-deployment
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: my-web-app
      template:
        metadata:
          labels:
            app: my-web-app # This is the label the anti-affinity targets
        spec:
          containers:
          - name: web-app-container
            image: nginx
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    app: my-web-app
                topologyKey: "kubernetes.io/hostname"
    ```

3.  **Apply and Observe:**

    ```bash
    kubectl apply -f anti-affinity-deploy.yaml
    kubectl get pods -o wide
    ```

4.  **Analyze the Result (Proof of Spreading):**
    Check the `NODE` column. You should see:

      * **Pod 1** on `minikube`
      * **Pod 2** on `minikube-m02`
        This proves the scheduler correctly ran the Pods on different nodes to satisfy the high-availability anti-affinity rule.

    Output:
    ```bash
    NAME                                 READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
    web-app-deployment-9cb6fdf4d-hhcw7   1/1     Running   0          39s   10.244.1.2   minikube-m02   <none>           <none>
    web-app-deployment-9cb6fdf4d-kmpxb   1/1     Running   0          39s   10.244.0.3   minikube       <none>           <none>
    ```

5.  **Clean Up:**

    ```bash
    kubectl delete deployment web-app-deployment
    minikube delete # If you created a multi-node cluster
    ```
