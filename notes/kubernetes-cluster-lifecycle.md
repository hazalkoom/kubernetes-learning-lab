# Kubernetes Learning Lab --- Starting, Stopping, and Shutting Down

This note explains what happens to the local Kubernetes learning cluster
when Fedora shuts down, how to stop and resume it manually, and how to
remove it intentionally.

## 1. How our local cluster is running

We created the cluster with **Kind** (Kubernetes IN Docker).

Kind runs Kubernetes nodes as Docker containers. In our setup, the
control-plane node is a Docker container named:

``` bash
learning-lab-control-plane
```

The structure looks like this:

``` mermaid
flowchart TD
    A[Fedora host] --> B[Docker Engine]
    B --> C[Kind node container]
    C --> D[Kubernetes control plane]
    C --> E[containerd]
    E --> F[Nginx Pod 1]
    E --> G[Nginx Pod 2]
```

Important distinction:

-   Docker runs the Kind **node container**.
-   Inside that node, Kubernetes components run and `containerd` runs
    the application containers.
-   The Nginx Pods are Kubernetes resources; they are not the same thing
    as the outer Kind Docker container.

## 2. What happens when Fedora shuts down?

During a normal Fedora shutdown:

1.  Fedora shuts down system services.
2.  Docker stops.
3.  The Kind node container stops.
4.  The Kubernetes API server and workloads stop running.
5.  The cluster's persisted data remains on disk, provided the Docker
    container and its data are not deleted.

You do **not** need to delete the cluster or manually delete its Pods
before shutting down the PC.

The Nginx application is only part of this local learning environment.
It does not need to stay running when you are not studying.

## 3. Will the cluster start automatically after reboot?

Docker has been configured to start at boot with:

``` bash
sudo systemctl enable docker
```

That enables the Docker service. Whether the Kind node container also
starts automatically depends on that container's Docker restart policy.

Inspect the policy:

``` bash
docker inspect learning-lab-control-plane \
  --format '{{.HostConfig.RestartPolicy.Name}}'
```

Common values include:

-   `no`: Docker does not automatically restart the container.
-   `always`: Docker attempts to restart it whenever it stops, including
    after daemon restarts.
-   `unless-stopped`: Docker restarts it unless it was explicitly
    stopped by the user.

Do not assume the container will automatically start until you inspect
its policy.

If the policy is `unless-stopped` or `always`, the node will normally
come back when Docker starts, subject to the policy's behavior and
whether the container still exists.

## 4. Stop and resume the cluster manually

To stop the Kind node without deleting it:

``` bash
docker stop learning-lab-control-plane
```

To start it again:

``` bash
sudo systemctl start docker
docker start learning-lab-control-plane
```

Then check Kubernetes:

``` bash
kubectl get nodes
```

Wait for the node's status to become `Ready`. It can take a short time
for the API server and other components to become available after the
container starts.

You can also inspect the container:

``` bash
docker ps -a
```

A stopped container should appear with a status such as `Exited`.

> Note: if the container has an automatic restart policy, an explicit
> `docker stop` suppresses automatic restarting until it is manually
> started or the daemon/container state changes according to Docker's
> restart-policy rules.

## 5. Delete the cluster completely

Only do this when you intentionally want to remove the learning cluster:

``` bash
kind delete cluster --name learning-lab
```

This deletes the Kind cluster, including its Kubernetes resources and
the node container. It is **not** the normal way to shut down your PC.

To recreate the cluster later:

``` bash
kind create cluster --name learning-lab
```

After recreating it, deploy the application again from the repository's
manifests:

``` bash
kubectl apply -f workloads/namespace.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Active \
  namespace/learning-lab --timeout=30s
kubectl apply -f workloads/deployment.yaml
kubectl apply -f workloads/service.yaml
```

The manifests in Git are the definitions we can use to recreate our
application. Deleting the cluster does not delete the files in the
repository.

## 6. Daily routine

  -------------------------------------------------------------------------------
  Situation                           Action
  ----------------------------------- -------------------------------------------
  Finished studying and shutting down Shut down Fedora normally
  Fedora                              

  Want to stop the cluster            `docker stop learning-lab-control-plane`
  temporarily                         

  Want to resume the cluster          `docker start learning-lab-control-plane`

  Want to check the node              `kubectl get nodes`

  Want to inspect container state     `docker ps -a`

  Intentionally want to delete the    `kind delete cluster --name learning-lab`
  cluster                             
  -------------------------------------------------------------------------------

**Recommended:** For an ordinary end-of-day shutdown, simply shut down
Fedora normally. Do not run `kind delete cluster`.

## 7. Useful mental model

``` text
Normal PC shutdown
    └── Docker stops
          └── Kind node stops
                └── Kubernetes workloads stop running

Next boot
    └── Docker starts
          └── Kind node starts (depending on restart policy)
                └── Kubernetes becomes available
```

The key difference:

-   **Stop** means pause the existing local cluster by stopping its node
    container.
-   **Delete** means remove the cluster itself.
-   **Recreate** means build a new cluster, then apply the YAML
    manifests to create the workloads again.
