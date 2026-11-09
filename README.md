# Kubernetes Learning Lab

A hands-on journey to learning Kubernetes through practical experiments and small projects.

## Goals

- Understand Kubernetes architecture
- Deploy and manage applications
- Work with Pods, Deployments, and Services
- Configure networking and storage
- Implement scaling and self-healing
- Practice troubleshooting

## Progress

- [ ] Cluster setup
- [ ] Pods
- [ ] Deployments
- [ ] Services
- [ ] ConfigMaps and Secrets
- [ ] Health Probes
- [ ] Resource Management
- [ ] Persistent Volumes
- [ ] Ingress
- [ ] Autoscaling
- [ ] Rolling Updates and Rollbacks


## Completed Milestones

### Milestone 1: First Kubernetes Workload

- Created a dedicated Namespace.
- Deployed Nginx using a Deployment.
- Configured two replicas.
- Inspected the automatically created ReplicaSet.
- Exposed Pods through a ClusterIP Service.
- Configured CPU and memory requests and limits.
- Added readiness and liveness probes.
- Tested self-healing by deleting a Pod.
- Verified automatic Pod replacement.
- Accessed Nginx using kubectl port-forward.

### Milestone 2: Scaling and Rolling Updates

- Configured the Deployment RollingUpdate strategy.
- Practiced maxSurge and maxUnavailable.
- Updated the Nginx container image.
- Observed old and new ReplicaSets during rollout.
- Inspected Deployment revision history.
- Practiced server-side dry-run validation.
- Practiced Deployment rollback.

### Milestone 3: Troubleshooting Workloads

- Practiced diagnosing failed container image pulls.
- Investigated ErrImagePull and ImagePullBackOff.
- Used kubectl describe and Kubernetes Events to identify failures.
- Inspected Deployment conditions and ReplicaSet status.
- Observed how maxSurge and maxUnavailable preserve availability during a failed rollout.
- Recovered the Deployment by restoring a valid container image.