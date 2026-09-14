# Kubernetes kubectl Complete Command Reference

A practical, group-wise command handbook for Kubernetes administration, application operations, troubleshooting, security, networking, storage, scheduling, automation, and production workflows.

This reference is centered on `kubectl`, the primary CLI for communicating with the Kubernetes API server. Kubernetes documents `kubectl` as the main interface for creating, inspecting, updating, deleting, debugging, and managing Kubernetes resources. For production, declarative management with `kubectl apply` and version-controlled manifests is generally preferred over one-off imperative commands.

Reference basis: current Kubernetes kubectl documentation and generated command reference. Kubernetes documentation currently publishes the kubectl reference for Kubernetes v1.37. Command availability can vary by Kubernetes version, enabled APIs, server configuration, and installed plugins.

Official references: https://kubernetes.io/docs/reference/kubectl/  |  https://kubernetes.io/docs/reference/kubectl/generated/  |  https://kubernetes.io/docs/reference/kubectl/quick-reference/

## 1. Command Map

| Group | Commands / subcommands | Primary purpose |

| --- | --- | --- |

| Discovery | api-resources, api-versions, explain, version | Discover API objects, versions, schemas, and client/server versions |

| Configuration | config | Manage kubeconfig, contexts, clusters, users, and namespaces |

| Read / inspect | get, describe, events | Inspect current cluster state and events |

| Declarative management | apply, diff, replace | Manage resources from manifests |

| Imperative creation | create, run, expose | Create resources directly from CLI |

| Mutation | edit, patch, label, annotate, taint, set | Modify resources |

| Deletion | delete | Remove resources |

| Workloads | rollout, scale, autoscale, wait | Manage Deployments, StatefulSets, DaemonSets, Jobs, and readiness |

| Containers | logs, exec, attach, cp, port-forward, debug | Inspect and interact with running containers |

| Nodes | cordon, uncordon, drain | Prepare nodes for maintenance |

| Security | auth, certificate | Check authorization and work with certificate requests |

| Networking | expose, port-forward, proxy | Expose workloads and access services |

| Kustomize | kustomize, apply -k, get -k | Build and apply Kustomize overlays |

| Observability | top, events, logs | CPU/memory metrics, events, and application logs |

| Automation | completion, plugin, options | Shell completion, extensions, global options |



## 2. kubectl Syntax Fundamentals

General form:

```bash
kubectl [global flags] COMMAND [SUBCOMMAND] [TYPE] [NAME] [flags]
```

Common resource forms:

```bash
kubectl get pods
kubectl get pod/nginx
kubectl get pods nginx
kubectl get deployments.apps
kubectl get deployment.apps/nginx
kubectl get pods -n production
kubectl get pods -A
kubectl get pods -l app=web
```

Common output modes:

```bash
kubectl get pod nginx -o wide
kubectl get pod nginx -o yaml
kubectl get pod nginx -o json
kubectl get pod nginx -o name
kubectl get pod nginx -o jsonpath='{.status.phase}'
kubectl get pods -o custom-columns='NAME:.metadata.name,IP:.status.podIP'
```

## 3. Global Flags and Common Options

| Flag | Meaning | Example |

| --- | --- | --- |

| --namespace / -n | Use a namespace | kubectl get pods -n production |

| --all-namespaces / -A | Query namespaced resources in all namespaces | kubectl get pods -A |

| --context | Use a specific kubeconfig context | kubectl --context=prod get nodes |

| --kubeconfig | Use a particular kubeconfig file | kubectl --kubeconfig=/tmp/prod.conf get pods |

| --server | Override API server endpoint | kubectl --server=https://api.example:6443 get ns |

| --certificate-authority | CA certificate for TLS verification | kubectl --certificate-authority=ca.crt get nodes |

| --token | Bearer token authentication | kubectl --token="$TOKEN" get pods |

| --as | Impersonate a user | kubectl --as=alice get pods |

| --as-group | Impersonate a group | kubectl --as=alice --as-group=devs get pods |

| --request-timeout | Set request timeout | kubectl --request-timeout=10s get pods |

| --v | Set client verbosity | kubectl -v=6 get pods |

| --dry-run=client | Render without contacting server | kubectl create deployment web --image=nginx --dry-run=client -o yaml |

| --dry-run=server | Validate against API server without persisting | kubectl apply --dry-run=server -f app.yaml |



# 4. DISCOVERY AND API INTROSPECTION

## 4.1 kubectl api-resources

Lists resource types supported by the API server, including short names, API groups, whether the resource is namespaced, and its kind.

```bash
kubectl api-resources
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
kubectl api-resources --verbs=list,get
kubectl api-resources --api-group=apps
kubectl api-resources -o wide
```

Useful when you do not know the exact resource name or whether a resource is namespaced.

## 4.2 kubectl api-versions

Lists API group/version combinations served by the API server.

```bash
kubectl api-versions
kubectl api-versions | grep apps
kubectl api-versions | grep batch
```

## 4.3 kubectl explain

Reads the resource schema from the server's OpenAPI information and explains fields.

```bash
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
kubectl explain service.spec.ports
kubectl explain pod --recursive
kubectl explain deployment --recursive --api-version=apps/v1
kubectl explain pod.spec.containers --recursive --max-depth=2
```

Use `explain` before writing unfamiliar YAML; it is especially useful for API-version and field-name mistakes.

## 4.4 kubectl version

```bash
kubectl version
kubectl version --client
kubectl version --output=yaml
kubectl version --output=json
kubectl version --short
```

Use this to verify client/server versions and investigate version skew.

# 5. KUBECONFIG AND CONTEXT MANAGEMENT

## 5.1 kubectl config current-context

```bash
kubectl config current-context
```

Shows the active context.

## 5.2 kubectl config get-contexts

```bash
kubectl config get-contexts
kubectl config get-contexts my-context
kubectl config get-contexts -o name
```

## 5.3 kubectl config use-context

```bash
kubectl config use-context dev
kubectl config use-context prod
```

Switches the active context. Always verify the active context before destructive production operations.

## 5.4 kubectl config get-clusters

```bash
kubectl config get-clusters
```

## 5.5 kubectl config get-users

```bash
kubectl config get-users
```

## 5.6 kubectl config view

```bash
kubectl config view
kubectl config view --minify
kubectl config view --raw
kubectl config view -o json
kubectl config view --flatten
kubectl config view --minify -o jsonpath='{.contexts[0].context.namespace}'
```

## 5.7 kubectl config set-context

```bash
kubectl config set-context dev --cluster=mycluster --user=dev-user --namespace=development
kubectl config set-context --current --namespace=production
kubectl config set-context prod --namespace=production
```

## 5.8 kubectl config set-cluster

```bash
kubectl config set-cluster mycluster --server=https://api.example.com:6443
kubectl config set-cluster mycluster --certificate-authority=ca.crt
kubectl config set-cluster mycluster --embed-certs=true
```

## 5.9 kubectl config set-credentials

```bash
kubectl config set-credentials alice --token="$TOKEN"
kubectl config set-credentials alice --client-certificate=alice.crt --client-key=alice.key
kubectl config set-credentials oidc-user --auth-provider=oidc
```

## 5.10 kubectl config rename-context

```bash
kubectl config rename-context old-context prod
```

## 5.11 kubectl config delete-context

```bash
kubectl config delete-context old-context
```

## 5.12 kubectl config delete-cluster

```bash
kubectl config delete-cluster old-cluster
```

## 5.13 kubectl config delete-user

```bash
kubectl config delete-user old-user
```

## 5.14 kubectl config set

Sets an arbitrary kubeconfig property.

```bash
kubectl config set clusters.mycluster.server https://api.example.com:6443
kubectl config set contexts.prod.namespace production
```

## 5.15 Multiple kubeconfig files

```bash
export KUBECONFIG="$HOME/.kube/config:$HOME/.kube/config-prod"
kubectl config view
kubectl config get-contexts
```

Use separate files when practical and be careful when merging credentials or modifying shared kubeconfig files.

# 6. LISTING AND INSPECTING RESOURCES

## 6.1 kubectl get

The most frequently used inspection command. It lists one or more resources and supports selectors, field selectors, namespaces, watch mode, and machine-readable output.

```bash
kubectl get pods
kubectl get pods -A
kubectl get pods -n production
kubectl get pod nginx
kubectl get pods -o wide
kubectl get deployment nginx -o yaml
kubectl get svc,deploy,pods
kubectl get all -n production
kubectl get pods -l app=nginx
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --sort-by=.status.startTime
kubectl get pods -w
kubectl get pods --watch-only
kubectl get pod nginx --show-managed-fields
kubectl get pod nginx --subresource=status
```

## 6.2 Output formats

| Format | Use |

| --- | --- |

| -o wide | Additional human-readable columns |

| -o yaml | Full YAML representation |

| -o json | Full JSON representation |

| -o name | Machine-friendly resource names |

| -o jsonpath='...' | Extract selected fields |

| -o jsonpath-as-json='...' | JSON-formatted JSONPath output |

| -o custom-columns='...' | Custom tabular output |

| -o go-template='...' | Go template output |

| -o go-template-file=... | Go template from file |



### JSONPath examples

```bash
kubectl get pod nginx -o jsonpath='{.status.podIP}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 -d
kubectl get pod nginx -o jsonpath='{.spec.containers[*].image}'
```

## 6.3 kubectl describe

Prints a detailed human-readable description, including related state and events.

```bash
kubectl describe pod nginx
kubectl describe deployment nginx
kubectl describe service nginx
kubectl describe node worker-1
kubectl describe namespace production
kubectl describe pod -l app=nginx
kubectl describe pods -n production
```

For troubleshooting, `describe` is often more useful than `get` because it exposes scheduling decisions, conditions, mounts, probes, and recent events.

## 6.4 kubectl events

```bash
kubectl events
kubectl events -A
kubectl events -n production
kubectl events --for pod/nginx
kubectl events --for deployment/nginx
kubectl events --types=Warning
kubectl events --watch
```

# 7. DECLARATIVE RESOURCE MANAGEMENT

## 7.1 kubectl apply

Applies a desired-state manifest to the cluster. This is the standard approach for reproducible, version-controlled resource management.

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f .
kubectl apply -R -f manifests/
kubectl apply -f deployment.yaml --namespace=production
kubectl apply -f app.yaml --server-side
kubectl apply -f app.yaml --dry-run=server
kubectl apply -f app.yaml --dry-run=client -o yaml
kubectl apply -k overlays/production
```

### Apply from stdin

```bash
cat deployment.yaml | kubectl apply -f -
kubectl create deployment web --image=nginx --dry-run=client -o yaml | kubectl apply -f -
```

### Apply subcommands

```bash
kubectl apply edit-last-applied deployment/nginx
kubectl apply view-last-applied deployment/nginx
kubectl apply set-last-applied -f deployment.yaml
kubectl apply set-last-applied -f deployment.yaml --create-annotation=false
```

## 7.2 kubectl diff

Shows the differences between the live object and the object that would be applied.

```bash
kubectl diff -f deployment.yaml
kubectl diff -R -f manifests/
kubectl diff -k overlays/production
kubectl diff -f app.yaml --server-side
```

## 7.3 kubectl replace

Replaces an existing resource with the supplied object. Use carefully because replacement is less forgiving than declarative patch/apply workflows.

```bash
kubectl replace -f deployment.yaml
kubectl replace --force -f pod.yaml
kubectl replace -f app.yaml --dry-run=server
```

# 8. CREATION COMMANDS

## 8.1 kubectl create

Imperatively creates resources. Useful for quick operations, experiments, and generating YAML; for production, consider committing manifests and using `apply`.

```bash
kubectl create namespace production
kubectl create deployment nginx --image=nginx:1.29
kubectl create service clusterip nginx --tcp=80:80
kubectl create configmap app-config --from-literal=ENV=prod
kubectl create secret generic app-secret --from-literal=password='change-me'
kubectl create serviceaccount app-sa
kubectl create job batch-job --image=busybox -- echo hello
kubectl create cronjob nightly --image=busybox --schedule='0 2 * * *' -- echo backup
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=production:app-sa
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
kubectl create clusterrolebinding node-readers --clusterrole=node-reader --group=ops
kubectl create quota team-quota --hard=pods=20,cpu=10,memory=20Gi
kubectl create priorityclass high-priority --value=100000
kubectl create serviceaccount app -n production
```

### Generate YAML without creating

```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip web --tcp=80:8080 --dry-run=client -o yaml
kubectl create configmap app-config --from-literal=MODE=prod --dry-run=client -o yaml
kubectl create secret generic app-secret --from-literal=token=example --dry-run=client -o yaml
```

# 9. WORKLOAD EXECUTION

## 9.1 kubectl run

Creates a Pod from the command line. Primarily useful for debugging, temporary workloads, or quick tests.

```bash
kubectl run nginx --image=nginx
kubectl run busybox --image=busybox --restart=Never -- sleep 3600
kubectl run curl --image=curlimages/curl:latest --rm -it -- sh
kubectl run dns-test --image=busybox:1.36 --restart=Never -- nslookup kubernetes.default
kubectl run test --image=busybox --dry-run=client -o yaml -- echo hello
```

## 9.2 kubectl exec

Runs a command inside a running container.

```bash
kubectl exec -it pod/nginx -- sh
kubectl exec -it pod/nginx -- bash
kubectl exec pod/nginx -- env
kubectl exec pod/nginx -- ls -la /etc
kubectl exec pod/nginx -c app -- sh
kubectl exec pod/nginx -- cat /etc/resolv.conf
kubectl exec deployment/nginx -- printenv
```

For multi-container Pods, use `-c CONTAINER`. Do not assume an image contains Bash; `sh` is more portable.

## 9.3 kubectl attach

```bash
kubectl attach pod/mypod
kubectl attach pod/mypod -c app
kubectl attach pod/mypod -c app -it
```

Attaches to an already-running process. It is different from `exec`, which starts a new process.

## 9.4 kubectl logs

```bash
kubectl logs pod/nginx
kubectl logs pod/nginx -c app
kubectl logs pod/nginx --previous
kubectl logs pod/nginx -f
kubectl logs pod/nginx --tail=100
kubectl logs pod/nginx --since=1h
kubectl logs pod/nginx --since-time='2026-09-14T00:00:00Z'
kubectl logs deployment/nginx
kubectl logs -l app=nginx --all-containers=true
kubectl logs pod/nginx --timestamps
kubectl logs pod/nginx --prefix
kubectl logs pod/nginx --ignore-errors
```

Use `--previous` for the previous terminated container instance, especially for CrashLoopBackOff investigation.

## 9.5 kubectl cp

```bash
kubectl cp production/nginx:/tmp/app.log ./app.log
kubectl cp ./config.json production/nginx:/tmp/config.json
kubectl cp ./config.json production/nginx:/tmp/config.json -c app
```

`kubectl cp` relies on `tar` being available in the container in common usage. For minimal images, consider alternatives such as `kubectl exec` with streams.

## 9.6 kubectl port-forward

```bash
kubectl port-forward pod/nginx 8080:80
kubectl port-forward service/nginx 8080:80
kubectl port-forward deployment/nginx 8080:80
kubectl port-forward pod/nginx 127.0.0.1:8080:80
kubectl port-forward service/nginx 8080:80 --address=0.0.0.0
```

Port-forwarding is mainly for local debugging and administrative access, not a replacement for production ingress/load-balancing.

## 9.7 kubectl proxy

```bash
kubectl proxy
kubectl proxy --port=8001
kubectl proxy --api-prefix=/api
kubectl proxy --address=127.0.0.1 --port=8001
```

Runs a local HTTP proxy to the Kubernetes API server.

# 10. RESOURCE MUTATION

## 10.1 kubectl edit

```bash
kubectl edit deployment nginx
kubectl edit service nginx
kubectl edit configmap app-config
kubectl edit deployment nginx -n production
```

Opens the live object in your configured editor and updates it after a successful save. Prefer Git-managed manifests for durable production changes.

## 10.2 kubectl patch

Updates selected fields without replacing the entire object.

```bash
kubectl patch deployment nginx -p '{"spec":{"replicas":3}}'
kubectl patch service nginx -p '{"spec":{"type":"LoadBalancer"}}'
kubectl patch deployment nginx --type=merge -p '{"spec":{"replicas":5}}'
kubectl patch deployment nginx --type=json -p='[{"op":"replace","path":"/spec/replicas","value":5}]'
kubectl patch deployment nginx --type=strategic -p '{"spec":{"template":{"metadata":{"labels":{"version":"v2"}}}}}'
```

Patch types include strategic merge, JSON merge, and JSON Patch. JSON Patch is useful for precise array/index operations.

## 10.3 kubectl label

```bash
kubectl label pod nginx app=web
kubectl label pod nginx environment=prod --overwrite
kubectl label pods -l app=nginx tier=frontend
kubectl label node worker-1 disktype=ssd
kubectl label namespace production environment=prod
kubectl label pod nginx app-
```

A trailing `-` removes a label.

## 10.4 kubectl annotate

```bash
kubectl annotate pod nginx description='frontend pod'
kubectl annotate deployment nginx owner=platform --overwrite
kubectl annotate pod nginx description-
kubectl annotate pods -l app=nginx team=platform
```

A trailing `-` removes an annotation.

## 10.5 kubectl taint

```bash
kubectl taint nodes worker-1 dedicated=payments:NoSchedule
kubectl taint nodes worker-1 dedicated=payments:PreferNoSchedule
kubectl taint nodes worker-1 dedicated=payments:NoExecute
kubectl taint nodes worker-1 dedicated=payments:NoSchedule-
kubectl taint nodes worker-1 key=value:NoSchedule --overwrite
```

## 10.6 kubectl set

Changes common workload fields without manually editing YAML.

```bash
kubectl set image deployment/nginx nginx=nginx:1.29
kubectl set image deployment/nginx nginx=nginx:1.29 --record
kubectl set env deployment/nginx ENV=production
kubectl set env deployment/nginx --from=configmap/app-config
kubectl set env deployment/nginx --from=secret/app-secret
kubectl set resources deployment/nginx -c=nginx --requests=cpu=100m,memory=128Mi --limits=cpu=500m,memory=512Mi
kubectl set service deployment/nginx --name=web --port=80
kubectl set subject rolebinding/read-pods --serviceaccount=production:app
kubectl set selector service/web app=frontend
```

# 11. DELETION

## 11.1 kubectl delete

```bash
kubectl delete pod nginx
kubectl delete pod nginx -n production
kubectl delete pods -l app=nginx
kubectl delete deployment nginx
kubectl delete service nginx
kubectl delete -f deployment.yaml
kubectl delete -k overlays/production
kubectl delete namespace test
kubectl delete pod nginx --grace-period=30
kubectl delete pod nginx --grace-period=0 --force
kubectl delete pod nginx --ignore-not-found
kubectl delete pods --all -n test
```

Force deletion should be exceptional. A force-deleted Pod may remain running briefly if the kubelet cannot immediately observe the deletion.

# 12. DEPLOYMENT AND ROLLOUT MANAGEMENT

## 12.1 kubectl rollout status

```bash
kubectl rollout status deployment/nginx
kubectl rollout status deployment/nginx --timeout=5m
kubectl rollout status statefulset/database
kubectl rollout status daemonset/node-agent
```

## 12.2 kubectl rollout history

```bash
kubectl rollout history deployment/nginx
kubectl rollout history deployment/nginx --revision=3
```

## 12.3 kubectl rollout undo

```bash
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=3
kubectl rollout history deployment/nginx
```

## 12.4 kubectl rollout pause

```bash
kubectl rollout pause deployment/nginx
```

## 12.5 kubectl rollout resume

```bash
kubectl rollout resume deployment/nginx
```

## 12.6 kubectl rollout restart

```bash
kubectl rollout restart deployment/nginx
kubectl rollout restart deployment -l app=nginx
kubectl rollout restart daemonset/node-agent
kubectl rollout restart statefulset/database
```

## 12.7 kubectl scale

```bash
kubectl scale deployment/nginx --replicas=5
kubectl scale deployment nginx --replicas=0
kubectl scale statefulset/database --replicas=3
kubectl scale deployment nginx --current-replicas=3 --replicas=5
kubectl scale --replicas=3 deployment/nginx
```

## 12.8 kubectl autoscale

```bash
kubectl autoscale deployment nginx --min=2 --max=10
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=70
kubectl autoscale statefulset api --min=2 --max=8 --cpu-percent=60
```

## 12.9 kubectl wait

```bash
kubectl wait --for=condition=Ready pod/nginx
kubectl wait --for=condition=Available deployment/nginx --timeout=120s
kubectl wait --for=condition=Ready pod -l app=nginx --timeout=180s
kubectl wait --for=jsonpath='{.status.phase}'=Running pod/nginx
kubectl wait --for=delete pod/nginx --timeout=60s
```

# 13. NODE ADMINISTRATION

## 13.1 kubectl get nodes

```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get nodes -l node-role.kubernetes.io/worker
kubectl get nodes --sort-by=.status.capacity.cpu
```

## 13.2 kubectl cordon

Marks a node unschedulable. Existing Pods continue running.

```bash
kubectl cordon worker-1
kubectl get nodes
```

## 13.3 kubectl uncordon

```bash
kubectl uncordon worker-1
```

## 13.4 kubectl drain

Evicts workload Pods from a node so it can be maintained. Always understand PodDisruptionBudgets, daemonsets, local storage, and unmanaged Pods before draining.

```bash
kubectl drain worker-1
kubectl drain worker-1 --ignore-daemonsets
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
kubectl drain worker-1 --ignore-daemonsets --force
kubectl drain worker-1 --ignore-daemonsets --grace-period=60
kubectl drain worker-1 --ignore-daemonsets --timeout=10m
kubectl drain worker-1 --dry-run=server
```

# 14. SECURITY AND AUTHORIZATION

## 14.1 kubectl auth can-i

```bash
kubectl auth can-i get pods
kubectl auth can-i create deployments
kubectl auth can-i delete pods -n production
kubectl auth can-i list pods --all-namespaces
kubectl auth can-i '*' '*'
kubectl auth can-i --list
kubectl auth can-i --list -n production
kubectl auth can-i get pods --subresource=log
kubectl auth can-i get jobs.batch/myjob -n production
```

## 14.2 Impersonation

```bash
kubectl auth can-i get pods --as=alice
kubectl auth can-i get pods --as=alice --as-group=developers
kubectl get pods --as=alice --as-group=developers
```

## 14.3 kubectl certificate

Manages CertificateSigningRequest resources.

```bash
kubectl certificate approve my-csr
kubectl certificate deny my-csr
kubectl certificate approve csr/my-csr
kubectl get csr
kubectl describe csr my-csr
```

# 15. APPLICATION EXPOSURE AND SERVICES

## 15.1 kubectl expose

```bash
kubectl expose deployment nginx --port=80 --target-port=80
kubectl expose deployment nginx --type=ClusterIP --port=80
kubectl expose deployment nginx --type=LoadBalancer --port=80
kubectl expose pod nginx --name=nginx-svc --port=80 --target-port=8080
kubectl expose deployment nginx --type=NodePort --port=80
```

## 15.2 Service inspection

```bash
kubectl get svc
kubectl get svc nginx -o yaml
kubectl describe svc nginx
kubectl get endpoints
kubectl get endpointslices
kubectl get endpointslices -l kubernetes.io/service-name=nginx
```

# 16. KUSTOMIZE

## 16.1 kubectl kustomize

```bash
kubectl kustomize .
kubectl kustomize overlays/production
kubectl kustomize overlays/staging
```

## 16.2 Apply Kustomize

```bash
kubectl apply -k overlays/production
kubectl diff -k overlays/production
kubectl delete -k overlays/production
kubectl get -k overlays/production
```

## 16.3 Build then inspect

```bash
kubectl kustomize overlays/production | less
```

# 17. DEBUGGING

## 17.1 kubectl debug

Creates debugging containers or Pods, depending on the selected mode and resource.

```bash
kubectl debug pod/nginx -it --image=busybox
kubectl debug pod/nginx -it --image=busybox --target=nginx
kubectl debug node/worker-1 -it --image=ubuntu
kubectl debug deployment/nginx -it --image=busybox
kubectl debug pod/nginx --copy-to=nginx-debug --share-processes
kubectl debug pod/nginx --copy-to=nginx-debug --set-image='*=busybox'
```

Use `kubectl debug` for cases where the original container is too minimal, crashed, lacks diagnostic tools, or requires process/network inspection.

## 17.2 Common Pod investigation sequence

```bash
kubectl get pod nginx -o wide
kubectl describe pod nginx
kubectl get events --field-selector=involvedObject.name=nginx
kubectl logs nginx
kubectl logs nginx --previous
kubectl get pod nginx -o yaml
kubectl exec -it nginx -- sh
```

## 17.3 Common CrashLoopBackOff sequence

```bash
kubectl get pod nginx
kubectl describe pod nginx
kubectl logs nginx --previous
kubectl get pod nginx -o jsonpath='{.status.containerStatuses[*].state}'
kubectl get pod nginx -o jsonpath='{.status.containerStatuses[*].lastState}'
```

## 17.4 ImagePullBackOff sequence

```bash
kubectl describe pod nginx
kubectl get events --sort-by=.lastTimestamp
kubectl get pod nginx -o jsonpath='{.spec.containers[*].image}'
```

## 17.5 Pending Pod sequence

```bash
kubectl describe pod nginx
kubectl get nodes
kubectl get nodes --show-labels
kubectl get pod nginx -o yaml
kubectl get pdb -A
```

# 18. RESOURCE METRICS

## 18.1 kubectl top nodes

```bash
kubectl top nodes
kubectl top node worker-1
kubectl top node --sort-by=cpu
kubectl top node --sort-by=memory
kubectl top node --show-capacity
```

## 18.2 kubectl top pods

```bash
kubectl top pods
kubectl top pod nginx
kubectl top pods -A
kubectl top pods -n production --sort-by=cpu
kubectl top pods -n production --sort-by=memory
kubectl top pod nginx --containers
kubectl top pods -l app=nginx
```

The `top` commands depend on a working Metrics API, commonly provided by Metrics Server.

# 19. LABELS, SELECTORS, AND FIELD SELECTORS

## 19.1 Label selectors

```bash
kubectl get pods -l app=nginx
kubectl get pods -l 'environment in (prod,staging)'
kubectl get pods -l 'tier!=backend'
kubectl get nodes -l disktype=ssd
kubectl delete pods -l app=temporary
kubectl logs -l app=nginx --all-containers=true
```

## 19.2 Field selectors

```bash
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=status.phase!=Succeeded
kubectl get events --field-selector=type=Warning
kubectl get pods --field-selector=spec.nodeName=worker-1
```

Field selectors are resource-specific; the API server supports only certain fields for each resource.

# 20. NAMESPACES

```bash
kubectl get namespaces
kubectl get ns
kubectl create namespace production
kubectl delete namespace test
kubectl describe namespace production
kubectl label namespace production environment=prod
kubectl annotate namespace production owner=platform
kubectl config set-context --current --namespace=production
kubectl get pods -n production
kubectl get pods -A
```

# 21. CONFIGMAPS

## 21.1 Inspect

```bash
kubectl get configmap
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

## 21.2 Create

```bash
kubectl create configmap app-config --from-literal=ENV=prod --from-literal=LOG_LEVEL=info
kubectl create configmap app-config --from-file=config.properties
kubectl create configmap app-config --from-file=./config-dir/
```

## 21.3 Update

```bash
kubectl edit configmap app-config
kubectl patch configmap app-config -p '{"data":{"ENV":"prod"}}'
```

# 22. SECRETS

## 22.1 Inspect

```bash
kubectl get secrets
kubectl get secret app-secret
kubectl get secret app-secret -o yaml
kubectl describe secret app-secret
```

Be careful with `-o yaml` and `-o json`: Secret values are normally base64-encoded, not encrypted in the API representation.

## 22.2 Create

```bash
kubectl create secret generic app-secret --from-literal=username=admin --from-literal=password='change-me'
kubectl create secret generic tls-secret --from-file=tls.crt=server.crt --from-file=tls.key=server.key
kubectl create secret docker-registry regcred --docker-server=registry.example.com --docker-username=user --docker-password='password' --docker-email=user@example.com
```

## 22.3 Decode a secret field

```bash
kubectl get secret app-secret -o jsonpath='{.data.password}' | base64 -d
```

# 23. RESOURCE QUOTAS AND LIMIT RANGES

## 23.1 ResourceQuota

```bash
kubectl get resourcequota
kubectl describe resourcequota team-quota
kubectl create quota team-quota --hard=pods=20,cpu=10,memory=20Gi
kubectl create quota compute-quota --hard=requests.cpu=4,requests.memory=8Gi,limits.cpu=8,limits.memory=16Gi
```

## 23.2 LimitRange

```bash
kubectl get limitrange
kubectl describe limitrange
kubectl apply -f limitrange.yaml
```

# 24. RBAC

## 24.1 Inspect RBAC

```bash
kubectl get roles -A
kubectl get rolebindings -A
kubectl get clusterroles
kubectl get clusterrolebindings
kubectl describe role pod-reader -n production
kubectl describe rolebinding read-pods -n production
kubectl describe clusterrole node-reader
kubectl describe clusterrolebinding node-readers
```

## 24.2 Create Role and RoleBinding

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n production
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=production:app -n production
```

## 24.3 Create ClusterRole and ClusterRoleBinding

```bash
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
kubectl create clusterrolebinding node-readers --clusterrole=node-reader --group=ops
```

## 24.4 Test RBAC

```bash
kubectl auth can-i get pods -n production
kubectl auth can-i get nodes
kubectl auth can-i list secrets -n production
kubectl auth can-i --list -n production
```

# 25. SERVICE ACCOUNTS

```bash
kubectl get serviceaccounts -A
kubectl get serviceaccount app -n production
kubectl describe serviceaccount app -n production
kubectl create serviceaccount app -n production
kubectl create token app -n production
kubectl create token app -n production --duration=1h
```

Modern Kubernetes commonly uses projected/service-account tokens and `kubectl create token` rather than manually creating long-lived token Secrets.

# 26. STORAGE

## 26.1 PersistentVolumeClaims

```bash
kubectl get pvc -A
kubectl get pvc -n production
kubectl describe pvc data-db-0 -n production
kubectl get pvc data-db-0 -o yaml
```

## 26.2 PersistentVolumes

```bash
kubectl get pv
kubectl describe pv pv-name
kubectl get pv -o wide
kubectl get pv -o yaml
```

## 26.3 StorageClasses

```bash
kubectl get storageclass
kubectl get sc
kubectl describe storageclass standard
kubectl get storageclass -o yaml
```

## 26.4 VolumeSnapshots

If the VolumeSnapshot CRDs and CSI driver support them:

```bash
kubectl get volumesnapshots -A
kubectl describe volumesnapshot my-snapshot -n production
kubectl get volumesnapshotclasses
```

# 27. DEPLOYMENTS

```bash
kubectl get deployments
kubectl describe deployment nginx
kubectl get deployment nginx -o yaml
kubectl scale deployment nginx --replicas=5
kubectl set image deployment/nginx nginx=nginx:1.29
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
kubectl rollout restart deployment/nginx
kubectl pause deployment/nginx
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx
```

# 28. STATEFULSETS

```bash
kubectl get statefulsets
kubectl describe statefulset database
kubectl scale statefulset database --replicas=3
kubectl rollout status statefulset/database
kubectl rollout restart statefulset/database
kubectl delete statefulset database
kubectl delete statefulset database --cascade=orphan
```

# 29. DAEMONSETS

```bash
kubectl get daemonsets -A
kubectl describe daemonset node-agent
kubectl rollout status daemonset/node-agent
kubectl rollout restart daemonset/node-agent
kubectl rollout undo daemonset/node-agent
kubectl get pods -l app=node-agent -o wide
```

# 30. JOBS AND CRONJOBS

## 30.1 Jobs

```bash
kubectl get jobs
kubectl describe job batch-job
kubectl logs job/batch-job
kubectl create job batch-job --image=busybox -- echo hello
kubectl delete job batch-job
```

## 30.2 CronJobs

```bash
kubectl get cronjobs
kubectl describe cronjob nightly
kubectl create cronjob nightly --image=busybox --schedule='0 2 * * *' -- echo backup
kubectl patch cronjob nightly -p '{"spec":{"suspend":true}}'
kubectl patch cronjob nightly -p '{"spec":{"suspend":false}}'
```

# 31. REPLICASETS AND REPLICATIONCONTROLLERS

```bash
kubectl get replicasets
kubectl describe replicaset nginx
kubectl get replicationcontrollers
kubectl describe replicationcontroller frontend
kubectl scale replicaset nginx --replicas=3
```

# 32. PODS

## 32.1 Pod lifecycle inspection

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl describe pod nginx
kubectl get pod nginx -o yaml
kubectl get pod nginx -o json
kubectl delete pod nginx
kubectl delete pod nginx --grace-period=0 --force
```

## 32.2 Find Pods on a node

```bash
kubectl get pods -A --field-selector=spec.nodeName=worker-1 -o wide
```

## 32.3 Find non-running Pods

```bash
kubectl get pods -A --field-selector=status.phase!=Running
```

# 33. NETWORK POLICY AND NETWORKING INSPECTION

```bash
kubectl get networkpolicies -A
kubectl get netpol -A
kubectl describe networkpolicy default-deny -n production
kubectl get services -A
kubectl get endpointslices -A
kubectl get ingress -A
kubectl get gateway -A
kubectl get httproute -A
```

Gateway API resources are available only when the relevant CRDs/API are installed and served.

## 33.1 DNS debugging

```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -it --rm -- nslookup kubernetes.default
kubectl run curl-test --image=curlimages/curl:latest --rm -it -- sh
kubectl exec -it dns-test -- cat /etc/resolv.conf
```

# 34. INGRESS

```bash
kubectl get ingress -A
kubectl describe ingress app-ingress -n production
kubectl get ingress app-ingress -o yaml
kubectl get ingressclass
kubectl describe ingressclass nginx
```

# 35. CRDs AND CUSTOM RESOURCES

## 35.1 CRDs

```bash
kubectl get crds
kubectl get crd
kubectl describe crd widgets.example.com
kubectl get crd widgets.example.com -o yaml
```

## 35.2 Custom resources

```bash
kubectl api-resources | grep -i widget
kubectl get widgets -A
kubectl describe widget example -n production
kubectl get widget example -o yaml
```

# 36. API SERVER ACCESS AND RAW API

## 36.1 kubectl get --raw

```bash
kubectl get --raw /version
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /livez
kubectl get --raw /api
kubectl get --raw /apis
kubectl get --raw /apis/apps/v1
```

## 36.2 Proxy and API exploration

```bash
kubectl proxy --port=8001
curl http://127.0.0.1:8001/version
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods
```

# 37. PLUGINS

## 37.1 kubectl plugin list

```bash
kubectl plugin list
kubectl plugin list --name-only
```

Plugins are external executables named `kubectl-<plugin>`. Kubernetes supports extending kubectl through this mechanism.

## 37.2 Invoke a plugin

```bash
kubectl myplugin
kubectl plugin-name args
```

# 38. SHELL COMPLETION

## Bash

```bash
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
alias k=kubectl
complete -o default -F __start_kubectl k
```

## Zsh

```bash
source <(kubectl completion zsh)
echo '[[ $commands[kubectl] ]] && source <(kubectl completion zsh)' >> ~/.zshrc
```

## Fish

```bash
kubectl completion fish | source
kubectl completion fish > ~/.config/fish/completions/kubectl.fish
```

## PowerShell

```bash
kubectl completion powershell | Out-String | Invoke-Expression
```

# 39. kubectl options

```bash
kubectl options
```

Displays flags inherited by kubectl commands.

# 40. MACHINE-READABLE OUTPUT AND AUTOMATION

## 40.1 Prefer stable output

For scripts, prefer JSON/YAML/name/JSONPath/custom columns rather than parsing human-readable tables.

```bash
kubectl get pods -o name
kubectl get pods -o json
kubectl get pods -o yaml
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase'
```

## 40.2 Explicit namespace/context

```bash
kubectl --context=prod --namespace=payments get pods -o name
kubectl --context=prod get nodes -o name
```

## 40.3 Use fully qualified resources in scripts when ambiguity matters

```bash
kubectl get deployments.apps
kubectl get jobs.batch
kubectl get certificatesigningrequests.certificates.k8s.io
```

# 41. SERVER-SIDE APPLY AND FIELD OWNERSHIP

```bash
kubectl apply --server-side -f app.yaml
kubectl apply --server-side --force-conflicts -f app.yaml
kubectl get deployment nginx -o yaml --show-managed-fields
kubectl apply --server-side --field-manager=platform-controller -f app.yaml
```

Server-side apply stores field ownership information. `--force-conflicts` can take ownership from another manager and should be used intentionally.

# 42. DRY RUN AND VALIDATION

```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml
kubectl apply -f app.yaml --dry-run=client
kubectl apply -f app.yaml --dry-run=server
kubectl create namespace test --dry-run=client -o yaml
kubectl create configmap app --from-literal=ENV=prod --dry-run=client -o yaml
```

Client dry-run constructs locally. Server dry-run sends the request through API admission and validation but does not persist the object.

# 43. COMMON PRODUCTION WORKFLOWS

## 43.1 Verify cluster before changing production

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
kubectl auth can-i get pods -n production
```

## 43.2 Deploy an application

```bash
kubectl diff -f manifests/
kubectl apply -f manifests/
kubectl rollout status deployment/web -n production --timeout=5m
kubectl get pods -n production -l app=web -o wide
```

## 43.3 Roll out a new image

```bash
kubectl set image deployment/web web=registry.example.com/web:v2 -n production
kubectl rollout status deployment/web -n production
kubectl rollout history deployment/web -n production
```

## 43.4 Roll back

```bash
kubectl rollout history deployment/web -n production
kubectl rollout undo deployment/web -n production
kubectl rollout status deployment/web -n production
```

## 43.5 Investigate a failing Pod

```bash
kubectl get pods -n production -o wide
kubectl describe pod failing-pod -n production
kubectl logs failing-pod -n production --all-containers=true
kubectl logs failing-pod -n production --previous
kubectl get events -n production --sort-by=.lastTimestamp
```

## 43.6 Drain a node for maintenance

```bash
kubectl get nodes
kubectl cordon worker-1
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
# perform maintenance
kubectl uncordon worker-1
kubectl get nodes
```

## 43.7 Check application resource usage

```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

## 43.8 Test DNS and service connectivity

```bash
kubectl run net-debug --image=curlimages/curl:latest --rm -it -- sh
# inside:
curl -v http://my-service.production.svc.cluster.local:8080
# or:
kubectl run dns-debug --image=busybox:1.36 --rm -it --restart=Never -- nslookup my-service.production.svc.cluster.local
```

# 44. COMMANDS BY RESOURCE TYPE

## Pods

Typical kubectl operations: get, describe, logs, exec, attach, cp, port-forward, delete, label, annotate, patch, edit, debug, wait, run.

```bash
kubectl get pods ...
kubectl describe pods ...
kubectl logs pods ...
kubectl exec pods ...
kubectl attach pods ...
kubectl cp pods ...
```

## Deployments

Typical kubectl operations: get, describe, edit, patch, set image, set env, set resources, scale, autoscale, rollout status, rollout history, rollout undo, rollout restart, rollout pause, rollout resume, delete.

```bash
kubectl get deployments ...
kubectl describe deployments ...
kubectl edit deployments ...
kubectl patch deployments ...
kubectl set image deployments ...
kubectl set env deployments ...
```

## StatefulSets

Typical kubectl operations: get, describe, edit, patch, scale, rollout status, rollout restart, delete.

```bash
kubectl get statefulsets ...
kubectl describe statefulsets ...
kubectl edit statefulsets ...
kubectl patch statefulsets ...
kubectl scale statefulsets ...
kubectl rollout status statefulsets ...
```

## DaemonSets

Typical kubectl operations: get, describe, edit, patch, rollout status, rollout restart, rollout undo, delete.

```bash
kubectl get daemonsets ...
kubectl describe daemonsets ...
kubectl edit daemonsets ...
kubectl patch daemonsets ...
kubectl rollout status daemonsets ...
kubectl rollout restart daemonsets ...
```

## Jobs

Typical kubectl operations: get, describe, logs, create, delete, wait.

```bash
kubectl get jobs ...
kubectl describe jobs ...
kubectl logs jobs ...
kubectl create jobs ...
kubectl delete jobs ...
kubectl wait jobs ...
```

## CronJobs

Typical kubectl operations: get, describe, create, patch, edit, delete.

```bash
kubectl get cronjobs ...
kubectl describe cronjobs ...
kubectl create cronjobs ...
kubectl patch cronjobs ...
kubectl edit cronjobs ...
kubectl delete cronjobs ...
```

## Services

Typical kubectl operations: get, describe, edit, patch, expose, delete, port-forward.

```bash
kubectl get services ...
kubectl describe services ...
kubectl edit services ...
kubectl patch services ...
kubectl expose services ...
kubectl delete services ...
```

## Nodes

Typical kubectl operations: get, describe, label, annotate, taint, cordon, uncordon, drain, top.

```bash
kubectl get nodes ...
kubectl describe nodes ...
kubectl label nodes ...
kubectl annotate nodes ...
kubectl taint nodes ...
kubectl cordon nodes ...
```

## Namespaces

Typical kubectl operations: get, describe, create, label, annotate, delete.

```bash
kubectl get namespaces ...
kubectl describe namespaces ...
kubectl create namespaces ...
kubectl label namespaces ...
kubectl annotate namespaces ...
kubectl delete namespaces ...
```

## ConfigMaps

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get configmaps ...
kubectl describe configmaps ...
kubectl create configmaps ...
kubectl edit configmaps ...
kubectl patch configmaps ...
kubectl delete configmaps ...
```

## Secrets

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get secrets ...
kubectl describe secrets ...
kubectl create secrets ...
kubectl edit secrets ...
kubectl patch secrets ...
kubectl delete secrets ...
```

## ServiceAccounts

Typical kubectl operations: get, describe, create, delete, create token.

```bash
kubectl get serviceaccounts ...
kubectl describe serviceaccounts ...
kubectl create serviceaccounts ...
kubectl delete serviceaccounts ...
kubectl create token serviceaccounts ...
```

## Roles

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get roles ...
kubectl describe roles ...
kubectl create roles ...
kubectl edit roles ...
kubectl patch roles ...
kubectl delete roles ...
```

## RoleBindings

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get rolebindings ...
kubectl describe rolebindings ...
kubectl create rolebindings ...
kubectl edit rolebindings ...
kubectl patch rolebindings ...
kubectl delete rolebindings ...
```

## ClusterRoles

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get clusterroles ...
kubectl describe clusterroles ...
kubectl create clusterroles ...
kubectl edit clusterroles ...
kubectl patch clusterroles ...
kubectl delete clusterroles ...
```

## ClusterRoleBindings

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get clusterrolebindings ...
kubectl describe clusterrolebindings ...
kubectl create clusterrolebindings ...
kubectl edit clusterrolebindings ...
kubectl patch clusterrolebindings ...
kubectl delete clusterrolebindings ...
```

## PersistentVolumes

Typical kubectl operations: get, describe, patch, delete.

```bash
kubectl get persistentvolumes ...
kubectl describe persistentvolumes ...
kubectl patch persistentvolumes ...
kubectl delete persistentvolumes ...
```

## PersistentVolumeClaims

Typical kubectl operations: get, describe, edit, patch, delete.

```bash
kubectl get persistentvolumeclaims ...
kubectl describe persistentvolumeclaims ...
kubectl edit persistentvolumeclaims ...
kubectl patch persistentvolumeclaims ...
kubectl delete persistentvolumeclaims ...
```

## StorageClasses

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get storageclasses ...
kubectl describe storageclasses ...
kubectl create storageclasses ...
kubectl edit storageclasses ...
kubectl patch storageclasses ...
kubectl delete storageclasses ...
```

## ResourceQuotas

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get resourcequotas ...
kubectl describe resourcequotas ...
kubectl create resourcequotas ...
kubectl edit resourcequotas ...
kubectl patch resourcequotas ...
kubectl delete resourcequotas ...
```

## LimitRanges

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get limitranges ...
kubectl describe limitranges ...
kubectl create limitranges ...
kubectl edit limitranges ...
kubectl patch limitranges ...
kubectl delete limitranges ...
```

## NetworkPolicies

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get networkpolicies ...
kubectl describe networkpolicies ...
kubectl create networkpolicies ...
kubectl edit networkpolicies ...
kubectl patch networkpolicies ...
kubectl delete networkpolicies ...
```

## Ingress

Typical kubectl operations: get, describe, edit, patch, delete.

```bash
kubectl get ingress ...
kubectl describe ingress ...
kubectl edit ingress ...
kubectl patch ingress ...
kubectl delete ingress ...
```

## CRDs

Typical kubectl operations: get, describe, create, edit, patch, delete.

```bash
kubectl get crds ...
kubectl describe crds ...
kubectl create crds ...
kubectl edit crds ...
kubectl patch crds ...
kubectl delete crds ...
```

## CSRs

Typical kubectl operations: get, describe, approve, deny, delete.

```bash
kubectl get csrs ...
kubectl describe csrs ...
kubectl approve csrs ...
kubectl deny csrs ...
kubectl delete csrs ...
```

# 45. COMPLETE TOP-LEVEL kubectl COMMAND CHECKLIST

| Command | Purpose |

| --- | --- |



# 46. Important kubectl Subcommands Checklist

## `kubectl auth` subcommands

| Subcommand | Example |

| --- | --- |

| can-i | `kubectl auth can-i ... |



## `kubectl certificate` subcommands

| Subcommand | Example |

| --- | --- |

| approve | `kubectl certificate approve ... |

| deny | `kubectl certificate deny ... |



## `kubectl completion` subcommands

| Subcommand | Example |

| --- | --- |

| bash | `kubectl completion bash ... |

| zsh | `kubectl completion zsh ... |

| fish | `kubectl completion fish ... |

| powershell | `kubectl completion powershell ... |



## `kubectl config` subcommands

| Subcommand | Example |

| --- | --- |

| current-context | `kubectl config current-context ... |

| delete-cluster | `kubectl config delete-cluster ... |

| delete-context | `kubectl config delete-context ... |

| delete-user | `kubectl config delete-user ... |

| get-clusters | `kubectl config get-clusters ... |

| get-contexts | `kubectl config get-contexts ... |

| get-users | `kubectl config get-users ... |

| rename-context | `kubectl config rename-context ... |

| set | `kubectl config set ... |

| set-cluster | `kubectl config set-cluster ... |

| set-credentials | `kubectl config set-credentials ... |

| set-context | `kubectl config set-context ... |

| use-context | `kubectl config use-context ... |

| view | `kubectl config view ... |



## `kubectl rollout` subcommands

| Subcommand | Example |

| --- | --- |

| history | `kubectl rollout history ... |

| pause | `kubectl rollout pause ... |

| restart | `kubectl rollout restart ... |

| resume | `kubectl rollout resume ... |

| status | `kubectl rollout status ... |

| undo | `kubectl rollout undo ... |



## `kubectl top` subcommands

| Subcommand | Example |

| --- | --- |

| node | `kubectl top node ... |

| pod | `kubectl top pod ... |



## `kubectl plugin` subcommands

| Subcommand | Example |

| --- | --- |

| list | `kubectl plugin list ... |



## `kubectl apply` subcommands

| Subcommand | Example |

| --- | --- |

| edit-last-applied | `kubectl apply edit-last-applied ... |

| set-last-applied | `kubectl apply set-last-applied ... |

| view-last-applied | `kubectl apply view-last-applied ... |



## `kubectl create` subcommands

| Subcommand | Example |

| --- | --- |

| clusterrole | `kubectl create clusterrole ... |

| clusterrolebinding | `kubectl create clusterrolebinding ... |

| configmap | `kubectl create configmap ... |

| cronjob | `kubectl create cronjob ... |

| deployment | `kubectl create deployment ... |

| namespace | `kubectl create namespace ... |

| poddisruptionbudget | `kubectl create poddisruptionbudget ... |

| priorityclass | `kubectl create priorityclass ... |

| quota | `kubectl create quota ... |

| role | `kubectl create role ... |

| rolebinding | `kubectl create rolebinding ... |

| secret | `kubectl create secret ... |

| service | `kubectl create service ... |

| serviceaccount | `kubectl create serviceaccount ... |

| token | `kubectl create token ... |

| job | `kubectl create job ... |



## `kubectl set` subcommands

| Subcommand | Example |

| --- | --- |

| env | `kubectl set env ... |

| image | `kubectl set image ... |

| resources | `kubectl set resources ... |

| selector | `kubectl set selector ... |

| serviceaccount | `kubectl set serviceaccount ... |

| subject | `kubectl set subject ... |



# 47. Troubleshooting Command Matrix

| Symptom | First commands | What to look for |

| --- | --- | --- |

| Pod Pending | get pod; describe pod; get nodes | Scheduling events, taints, affinity, resource shortage, PVC binding |

| CrashLoopBackOff | describe pod; logs; logs --previous | Application exit, probes, config, permissions |

| ImagePullBackOff | describe pod; get pod -o yaml | Registry auth, image name/tag, network, admission |

| Service unreachable | get svc; get endpointslices; describe svc; exec/curl | Selectors, endpoints, ports, NetworkPolicy, DNS |

| DNS failure | run/exec nslookup; cat /etc/resolv.conf | CoreDNS, search domains, service name, NetworkPolicy |

| Node NotReady | get nodes; describe node; get events -A | Kubelet, pressure conditions, networking, runtime |

| Deployment not progressing | rollout status; describe deployment; get rs; get pods | ReplicaSet, probes, image, scheduling, quota |

| PVC Pending | get pvc; describe pvc; get storageclass | StorageClass, provisioner, topology, capacity |

| RBAC denied | auth can-i; auth can-i --list | Role/ClusterRole and binding |

| High CPU | top pods; top nodes | CPU consumers and capacity |

| High memory | top pods --sort-by=memory; describe pod | Memory consumers, limits, OOMKilled |



# 48. Safety Rules for Production

- Verify `kubectl config current-context` before production changes.
- Prefer `kubectl diff` before `kubectl apply` for important changes.
- Prefer Git/version-controlled manifests over manual `kubectl edit` for persistent configuration.
- Use `--dry-run=server` when you need server-side validation without persistence.
- Use `kubectl auth can-i` before operating under a restricted identity.
- Be cautious with `kubectl delete namespace`, `delete --all`, `drain`, and forced Pod deletion.
- Do not expose Secret values casually through terminal history, CI logs, or chat.
- Use machine-readable output (`-o json`, `-o yaml`, `-o name`, JSONPath, custom columns) in automation rather than parsing human tables.
- Use compatible kubectl/client versions. Kubernetes documents kubectl version skew separately; avoid assuming arbitrary client/server combinations are supported.
- Treat alpha/beta APIs and plugin commands as version-dependent.

# 49. Useful Aliases

```bash
alias k=kubectl
alias kg='kubectl get'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kgs='kubectl get svc'
alias kga='kubectl get all'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'

```

# 50. Fast Daily Cheat Sheet

```bash
# Context
kubectl config current-context
kubectl config get-contexts
kubectl config use-context NAME

# Cluster
kubectl cluster-info
kubectl get nodes -o wide

# Workloads
kubectl get pods -A -o wide
kubectl get deploy -A
kubectl get sts -A
kubectl get ds -A
kubectl get jobs -A
kubectl get cronjobs -A

# Services
kubectl get svc -A
kubectl get ingress -A
kubectl get endpointslices -A

# Debug
kubectl describe pod POD -n NS
kubectl logs POD -n NS
kubectl logs POD -n NS --previous
kubectl exec -it POD -n NS -- sh
kubectl get events -n NS --sort-by=.lastTimestamp

# Resources
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Deploy
kubectl diff -f .
kubectl apply -f .
kubectl rollout status deployment/NAME -n NS

# Rollback
kubectl rollout history deployment/NAME -n NS
kubectl rollout undo deployment/NAME -n NS

# Node maintenance
kubectl cordon NODE
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data
kubectl uncordon NODE

# RBAC
kubectl auth can-i get pods -n NS
kubectl auth can-i --list -n NS

# Discovery
kubectl api-resources
kubectl api-versions
kubectl explain deployment.spec --recursive

```

# 51. Version and Availability Notes

This document intentionally separates core kubectl commands from Kubernetes API resources. A resource may exist only when its API/CRD is installed and served. Some commands and flags change between Kubernetes releases. The official generated reference should be treated as the authoritative command syntax for the exact Kubernetes version you operate.

For example, the current Kubernetes quick reference documents Kubernetes v1.37, while kubectl is subject to a supported client/server version-skew policy. Always verify `kubectl version` and consult the documentation matching your cluster version for production automation.

# 52. Official Documentation

- Kubernetes kubectl overview: https://kubernetes.io/docs/concepts/overview/kubectl/
- Kubernetes kubectl reference: https://kubernetes.io/docs/reference/kubectl/
- Generated kubectl commands: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands
- kubectl quick reference: https://kubernetes.io/docs/reference/kubectl/quick-reference/
- kubectl usage conventions: https://kubernetes.io/docs/reference/kubectl/conventions/
- kubectl JSONPath: https://kubernetes.io/docs/reference/kubectl/jsonpath/

# Appendix A. Reusable Command Templates

| Task | Template |

| --- | --- |

| List all resources in a namespace | kubectl get all -n NAMESPACE |

| List Pods with node/IP | kubectl get pods -n NAMESPACE -o wide |

| Find Pods by label | kubectl get pods -n NAMESPACE -l KEY=VALUE |

| Find Pods on a node | kubectl get pods -A --field-selector=spec.nodeName=NODE -o wide |

| Extract a field | kubectl get RESOURCE NAME -o jsonpath='{.FIELD.PATH}' |

| Watch a resource | kubectl get RESOURCE -w |

| Describe a resource | kubectl describe RESOURCE NAME -n NAMESPACE |

| Stream logs | kubectl logs -f POD -n NAMESPACE |

| Previous container logs | kubectl logs POD -n NAMESPACE --previous |

| Shell into container | kubectl exec -it POD -n NAMESPACE -c CONTAINER -- sh |

| Forward service port | kubectl port-forward svc/SERVICE LOCAL_PORT:REMOTE_PORT -n NAMESPACE |

| Apply manifest | kubectl apply -f FILE.yaml |

| Apply directory | kubectl apply -R -f DIRECTORY/ |

| Apply Kustomize | kubectl apply -k DIRECTORY/ |

| Preview changes | kubectl diff -f FILE.yaml |

| Server dry-run | kubectl apply --dry-run=server -f FILE.yaml |

| Create YAML | kubectl create deployment NAME --image=IMAGE --dry-run=client -o yaml |

| Scale | kubectl scale deployment/NAME --replicas=N |

| Update image | kubectl set image deployment/NAME CONTAINER=IMAGE |

| Restart | kubectl rollout restart deployment/NAME |

| Check rollout | kubectl rollout status deployment/NAME |

| Rollback | kubectl rollout undo deployment/NAME |

| Check authorization | kubectl auth can-i VERB RESOURCE -n NAMESPACE |

| Cordon node | kubectl cordon NODE |

| Drain node | kubectl drain NODE --ignore-daemonsets --delete-emptydir-data |

| Uncordon node | kubectl uncordon NODE |



# Appendix B. Practical Notes on Resource Names

Kubernetes resource types commonly have aliases. Examples include `po` for pods, `deploy` for deployments, `sts` for statefulsets, `ds` for daemonsets, `svc` for services, `ns` for namespaces, `cm` for configmaps, `sa` for serviceaccounts, `rs` for replicasets, `job`, `cj` for cronjobs, `pv`, `pvc`, `sc` for storageclasses, and `netpol` for networkpolicies. Verify aliases on the cluster with `kubectl api-resources` rather than hard-coding assumptions across unusual environments.

# Appendix C. Production Command Sequence: Safe Deployment

```bash
# 1. Verify context
kubectl config current-context

# 2. Inspect current state
kubectl get deployment web -n production
kubectl get pods -n production -l app=web -o wide

# 3. Preview
kubectl diff -f manifests/

# 4. Validate server-side without persisting
kubectl apply --dry-run=server -f manifests/

# 5. Apply
kubectl apply -f manifests/

# 6. Watch rollout
kubectl rollout status deployment/web -n production --timeout=5m

# 7. Verify
kubectl get pods -n production -l app=web -o wide
kubectl get svc web -n production
kubectl get endpointslices -n production -l kubernetes.io/service-name=web

# 8. If failure occurs
kubectl describe deployment web -n production
kubectl get events -n production --sort-by=.lastTimestamp
kubectl logs -l app=web -n production --all-containers=true --tail=100

# 9. Roll back if necessary
kubectl rollout undo deployment/web -n production
kubectl rollout status deployment/web -n production

```
