# Milestone 4 — Configuration and Application Health

## Goals

- Separate application configuration from the container image.
- Use ConfigMaps for non-sensitive configuration.
- Use Secrets for sensitive values, while understanding their security limitations.
- Inject configuration as environment variables.
- Mount ConfigMap content as files inside a container.
- Understand Pod-level `volumes` versus container-level `volumeMounts`.

## 1. Why externalize configuration?

The same application image can run in development, testing, and production with different configuration. This avoids building a different image for each environment.

- **ConfigMap:** non-sensitive configuration, such as environment names, log levels, and feature flags.
- **Secret:** sensitive configuration, such as passwords, tokens, and credentials.

Both can be consumed through environment variables or mounted files.

## 2. ConfigMap manifest

File: `workloads/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: learning-lab
data:
  APP_ENV: development
  LOG_LEVEL: debug
  index.html: |
    <!DOCTYPE html>
    <html>
      <head>
        <title>Kubernetes Learning Lab</title>
      </head>
      <body>
        <h1>Milestone 4</h1>
        <p>This page is served using a Kubernetes ConfigMap.</p>
      </body>
    </html>
```

Apply and inspect:

```bash
kubectl apply -f workloads/configmap.yaml
kubectl get configmap nginx-config -n learning-lab
kubectl describe configmap nginx-config -n learning-lab
```

The `|` YAML block scalar preserves the multiline HTML value.

## 3. Create a demo Secret

Create a fake token for this lab. Do not use a real credential.

```bash
kubectl create secret generic nginx-demo-secret \
  --namespace learning-lab \
  --from-literal=DEMO_API_TOKEN='local-demo-token-123'
```

Inspect metadata without printing the value:

```bash
kubectl describe secret nginx-demo-secret -n learning-lab
```

Important security points:

- Kubernetes Secret values are base64-encoded in their API representation; base64 is not encryption.
- Restrict access with RBAC.
- Configure encryption at rest for cluster data.
- Avoid committing real credentials to Git.
- A Secret is not automatically safe merely because it uses the `Secret` resource kind.

## 4. Complete Deployment manifest

File: `workloads/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx
  namespace: learning-lab

spec:
  replicas: 2

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.28.0-alpine

          ports:
            - name: http
              containerPort: 80

          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: nginx-config
                  key: APP_ENV

            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: nginx-config
                  key: LOG_LEVEL

            - name: DEMO_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: nginx-demo-secret
                  key: DEMO_API_TOKEN

          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi

          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10

          volumeMounts:
            - name: nginx-page
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
              readOnly: true

      volumes:
        - name: nginx-page
          configMap:
            name: nginx-config
            items:
              - key: index.html
                path: index.html
```

### Understand the YAML hierarchy

```text
Deployment
└── spec
    └── template
        └── spec
            ├── containers
            │   └── nginx
            │       ├── image
            │       ├── ports
            │       ├── env
            │       ├── resources
            │       ├── readinessProbe
            │       ├── livenessProbe
            │       └── volumeMounts
            └── volumes
```

- `volumes` belongs to the **Pod specification**. It declares volumes available to the Pod.
- `volumeMounts` belongs to an individual **container**. It chooses where that container sees a volume.
- `configMapKeyRef` reads one ConfigMap key into an environment variable.
- `secretKeyRef` reads one Secret key into an environment variable.
- `subPath` mounts a specific file from the volume at the requested path.

## 5. Apply in dependency order

The ConfigMap and Secret must exist before the Pods can consume them.

```bash
kubectl apply -f workloads/configmap.yaml

kubectl create secret generic nginx-demo-secret \
  --namespace learning-lab \
  --from-literal=DEMO_API_TOKEN='local-demo-token-123'
```

If the Secret already exists, the create command will report `AlreadyExists`. For this lab, leave it in place.

Validate and apply the Deployment:

```bash
kubectl apply --dry-run=server -f workloads/deployment.yaml
kubectl apply -f workloads/deployment.yaml
kubectl rollout status deployment/nginx -n learning-lab
```

## 6. Verify the configuration

```bash
kubectl get pods -n learning-lab

kubectl exec deployment/nginx -n learning-lab -- printenv APP_ENV
kubectl exec deployment/nginx -n learning-lab -- printenv LOG_LEVEL

kubectl exec deployment/nginx -n learning-lab -- \
  sh -c 'test -n "$DEMO_API_TOKEN" && echo "Secret successfully injected"'
```

Port-forward the Service:

```bash
kubectl port-forward service/nginx-service 8080:80 -n learning-lab
```

In another terminal:

```bash
curl http://localhost:8080
```

Expected HTML includes:

```html
<h1>Milestone 4</h1>
<p>This page is served using a Kubernetes ConfigMap.</p>
```

## 7. Important behavior and caveats

- Environment variables are established when a container starts. Updating the ConfigMap or Secret does not change environment variables inside an already-running container. A Pod restart/rollout is needed to load the new values.
- ConfigMap volumes can update eventually when their source changes, but a ConfigMap mounted using `subPath` does **not** receive those updates in an existing container.
- The ConfigMap and Secret must exist in the same namespace as the Pod that references them.
- A successful server-side dry run validates the Kubernetes object schema; it does not prove every referenced dependency or image will work at runtime.
- This Nginx example demonstrates configuration injection. Nginx itself does not use `APP_ENV`, `LOG_LEVEL`, or `DEMO_API_TOKEN`; the values are present in its container environment for demonstration.

## 8. Troubleshooting commands

```bash
kubectl describe deployment nginx -n learning-lab
kubectl describe pod <pod-name> -n learning-lab
kubectl get events -n learning-lab --sort-by=.metadata.creationTimestamp
kubectl get configmap nginx-config -n learning-lab -o yaml
kubectl describe secret nginx-demo-secret -n learning-lab
kubectl rollout status deployment/nginx -n learning-lab
kubectl rollout history deployment/nginx -n learning-lab
```

If a Pod cannot start because a referenced ConfigMap or Secret is missing, inspect the Pod's Events with `kubectl describe pod`.

## 9. Review questions

1. Why use a ConfigMap instead of hardcoding `APP_ENV` into the image?
2. What is the difference between `volumes` and `volumeMounts`?
3. Why is base64 encoding not sufficient protection for a Secret?
4. If `LOG_LEVEL` changes in the ConfigMap, does an existing container's environment variable change automatically?
5. Why does the Secret need to exist in the Pod's namespace?
6. What is the limitation of mounting a ConfigMap file using `subPath`?

## 10. Milestone checklist

- [ ] Created `nginx-config` ConfigMap.
- [ ] Created the demo `nginx-demo-secret`.
- [ ] Added ConfigMap and Secret environment variable references.
- [ ] Mounted the ConfigMap HTML file into Nginx.
- [ ] Applied the Deployment and waited for rollout completion.
- [ ] Verified environment variables without exposing the token.
- [ ] Verified the custom page through the Service.
- [ ] Updated the repository README and committed the milestone.
