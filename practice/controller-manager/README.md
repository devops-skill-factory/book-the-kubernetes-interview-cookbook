# Controller Manager

The [Kubernetes Controller Manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) is a core control plane component that **runs as a daemon, embedding multiple built-in controllers in a single process**.  

Each **controller implements a continuous reconciliation loop** that watches the cluster's current state via the API server and performs actions to align it with the desired state defined in resource specifications.  

Common examples of these controllers include the *ReplicaSet controller* (for maintaining pod replicas), *Node controller* (for monitoring node health), *Endpoints controller* (for managing service endpoints), and *Namespace controller* (for lifecycle management).  

By handling these control loops efficiently, the Controller Manager ensures the overall reliability and declarative nature of the Kubernetes cluster, with cloud-specific logic often separated into the [Cloud Controller Manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/).

## Exercise 1: ReplicaSet Controller in Action

This exercise proves that the **ReplicaSet Controller** (running inside the Controller Manager) is constantly watching and reconciles the current state with desired state.

1.  **Create a Deployment:**
    `kubectl create deployment nginx-test --image=nginx --replicas=3`
2.  **Watch the Pods:** In a separate terminal, run: `kubectl get pods -w`
    You should see the output like this:
    ```bash
    NAME                          READY   STATUS    RESTARTS   AGE
    nginx-test-586bbf5c4c-527qw   1/1     Running   0          8s
    nginx-test-586bbf5c4c-rmkx9   1/1     Running   0          8s
    nginx-test-586bbf5c4c-ssfss   1/1     Running   0          8s
    ```
3.  **The Sabotage:** Manually delete one of the pods:
    `kubectl delete pod <name-of-one-pod>`
4.  **Observe:** Notice how fast a new pod is created.
    * **The Logic:** The Controller Manager saw: *Desired (3) != Actual (2)*. Its loop immediately triggered a `Create` call.
    
    ```bash
    NAME                          READY   STATUS    RESTARTS   AGE
    nginx-test-586bbf5c4c-527qw   1/1     Running   0          8s
    nginx-test-586bbf5c4c-rmkx9   1/1     Running   0          8s
    nginx-test-586bbf5c4c-ssfss   1/1     Running   0          8s
    nginx-test-586bbf5c4c-ssfss   1/1     Terminating   0          92s
    nginx-test-586bbf5c4c-7fspv   0/1     Pending       0          1s
    nginx-test-586bbf5c4c-ssfss   1/1     Terminating   0          93s
    nginx-test-586bbf5c4c-7fspv   0/1     Pending       0          1s
    nginx-test-586bbf5c4c-7fspv   0/1     ContainerCreating   0          1s
    nginx-test-586bbf5c4c-ssfss   0/1     Completed           0          93s
    nginx-test-586bbf5c4c-ssfss   0/1     Completed           0          94s
    nginx-test-586bbf5c4c-ssfss   0/1     Completed           0          94s
    nginx-test-586bbf5c4c-7fspv   1/1     Running             0          5s
    ```

## Exercise 2: "Stopping the Brain"
In this exercise, you will temporarily "kill" the Controller Manager to see what happens to the cluster when the reconciliation loops stop running.

> **NOTE:** This requires a local cluster like **Minikube** where you have access to the control plane components.

1. Start Minikube with 1 node (control plane):
```bash
minikube start
```
2.  **Start a Deployment:** Ensure you have 3 replicas of Nginx running.  
    `kubectl create deployment nginx-test --image=nginx --replicas=3`
    ```bash
    NAME                          READY   STATUS    RESTARTS   AGE
    nginx-test-586bbf5c4c-gk2z6   1/1     Running   0          4m37s
    nginx-test-586bbf5c4c-nqjjx   1/1     Running   0          4m37s
    nginx-test-586bbf5c4c-x7bmw   1/1     Running   0          4m37s
    ```
3. SSH into the Minikube Node:
```bash
minikube ssh
```
4.  **Pause the Controller Manager:** * On Minikube/Kind, the controller manager runs as a static pod in the `kube-system` namespace.
    * Find the manifest file (usually at `/etc/kubernetes/manifests/kube-controller-manager.yaml` on the node) and move it out of that folder temporarily. This will stop the pod.
    ```bash
    cd /etc/kubernetes/manifests
    ls
    ```
    ```bash
    # Move it to your home directory so Kubelet stops the pod
    sudo mv kube-controller-manager.yaml ~/
    ```
5. To verify it's stopped: Exit the SSH session (`exit`) and run:
```bash
kubectl get pods -n kube-system
```
You will notice the `kube-controller-manager-minikube` pod is either gone or in a "*Terminating*" state.

6.  **Perform Sabotage:** Delete a pod while the Manager is down:  
    `kubectl delete pod <name-of-pod>`  
    `kubectl delete pod nginx-test-586bbf5c4c-gk2z`

7.  **Observe the "Ghost":** Notice that the pod stays deleted. Kubernetes does **not** replace it. The "Self-healing" is gone because the loop is not running.
```bash
NAME                          READY   STATUS    RESTARTS   AGE
nginx-test-586bbf5c4c-nqjjx   1/1     Running   0          5m11s
nginx-test-586bbf5c4c-x7bmw   1/1     Running   0          5m11s
```

8.  **Restore:** Move the manifest file back. Watch as the Controller Manager wakes up, realizes a pod is missing, and instantly reconciles it.