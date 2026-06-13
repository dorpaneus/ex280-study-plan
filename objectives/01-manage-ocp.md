# Objective 1 - Manage OpenShift Container Platform

> **Exam study points (verbatim from Red Hat):**
> - Use the web console to manage and configure an OpenShift cluster
> - Use the command-line interface to manage and configure an OpenShift cluster
> - Create and delete projects
> - Locate and examine container images
> - Identify images using tags and digests
> - Query, format, and filter attributes of Kubernetes resources
> - Import, export, and configure Kubernetes resources
> - Examine resources and cluster status
> - Monitor cluster events and alerts
> - View logs
> - Troubleshoot common container, pod, and cluster events and alerts
> - Use product documentation
> - Be prepared to perform all tasks of a Red Hat Certified Technologist in OpenShift

Plus, in 4.18: **be comfortable with cluster updates** (channel selection, EUS-to-EUS, MCP pausing).

---

## §1 - `oc` and the web console

Both are first-class tools. You'll be faster with `oc` on the exam, but the web console is unbeatable for browsing operators, events, and YAML.

### Logging in

```bash
# Username/password (HTPasswd, kube:admin, etc.)
oc login -u kubeadmin -p $PASSWORD https://api.crc.testing:6443

# Token (web console → top-right user menu → "Copy login command")
oc login --token=sha256~xxxxxxx --server=https://api.example.com:6443

# Where am I, who am I, what are my permissions
oc whoami
oc whoami --show-server
oc whoami --show-console      # URL of the web console
oc whoami --show-token

# Switch users on the same cluster (separate kubeconfig contexts)
oc login -u alice -p alicepw
oc config get-contexts
oc config use-context default/api-crc-testing:6443/alice
```

### Console URL & credentials

```bash
# CRC
crc console
crc console --credentials

# Any cluster: parse from routes
oc get route -n openshift-console console -o jsonpath='{.spec.host}'
```

## §2 - Projects (namespaces with extra annotations)

A **Project** is a Kubernetes `Namespace` plus OpenShift annotations (display name, description, requester). Always use `oc new-project` / `oc delete project` so the annotations come along.

```bash
# Create
oc new-project myapp --display-name="My App" --description="EX280 practice"

# Switch
oc project myapp
oc project                       # show current

# List
oc projects                      # human-friendly
oc get projects                  # raw

# Delete (asynchronous; namespace stays "Terminating" for a few seconds)
oc delete project myapp

# Wait until fully gone
oc wait --for=delete project/myapp --timeout=60s
```

> ⚠️ On the exam, **read the prompt for the exact project name** - case-sensitive, hyphens matter. Use `oc project <name>` immediately so all subsequent commands land in the right place.

## §3 - Container images: tags & digests

OpenShift pulls images by **tag** (`:1.2.0`, `:latest`) or **digest** (`@sha256:abc…`). Digests are immutable; tags can move.

```bash
# Inspect an image without pulling it
oc image info quay.io/openshifttest/hello-openshift:1.2.0

# Get the digest from a tag
oc image info quay.io/openshifttest/hello-openshift:1.2.0 --output=json | jq -r .digest

# Pull and re-tag into the internal registry (image stream)
oc import-image hello:1.2.0 --from=quay.io/openshifttest/hello-openshift:1.2.0 \
  --confirm -n myapp
oc get is hello -n myapp
oc get istag hello:1.2.0 -n myapp
```

### ImageStreams: OpenShift's image abstraction

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: hello
spec:
  lookupPolicy:
    local: true                # let pods reference this IS directly
  tags:
    - name: "1.2.0"
      from:
        kind: DockerImage
        name: quay.io/openshifttest/hello-openshift:1.2.0
      importPolicy:
        scheduled: true        # periodically check for new digest
```

Apply with `oc apply -f imagestream.yaml`.

## §4 - Querying resources: format & filter

This is exam gold. Master these.

```bash
# Default + wide
oc get pods
oc get pods -o wide

# YAML / JSON
oc get pod mypod -o yaml
oc get pod mypod -o json | jq '.status.phase'

# Custom columns
oc get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# JSONPath
oc get pods -o jsonpath='{.items[*].metadata.name}'
oc get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Label selectors
oc get pods -l app=hello
oc get pods -l 'app in (hello,nodejs)'
oc get pods -l 'environment!=prod'

# All resources in a namespace
oc get all -n myapp

# All resource types ever (incl. CRDs)
oc api-resources
oc api-resources --namespaced=true
oc api-resources --api-group=apps

# Watch a resource live
oc get pods -w
```

### Field selectors

```bash
oc get pods --field-selector=status.phase=Running
oc get events --field-selector=involvedObject.kind=Pod,type=Warning
```

### Sorting

```bash
oc get pods --sort-by=.status.startTime
oc get events --sort-by=.lastTimestamp
```

## §5 - Importing, exporting, editing resources

```bash
# Export an existing resource as YAML (no cluster-side fields)
oc get deployment hello -o yaml > hello.yaml      # has status, resourceVersion etc.
oc get deployment hello -o yaml --show-managed-fields=false > hello.yaml

# Apply (declarative — creates or updates)
oc apply -f hello.yaml

# Create (errors if it already exists)
oc create -f hello.yaml

# Replace (full overwrite)
oc replace -f hello.yaml

# Edit live (opens $EDITOR; the change is applied on save)
oc edit deployment hello

# Patch (great for one-liners)
oc patch deployment hello -p '{"spec":{"replicas":3}}'
oc patch deployment hello --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"nginx:1.27"}]'

# Generate YAML without applying (the exam-day cheat code)
oc create deployment hello --image=nginx --replicas=3 --dry-run=client -o yaml > hello.yaml
oc create configmap mycfg --from-literal=key=value --dry-run=client -o yaml
oc create secret generic mysec --from-literal=password=hunter2 --dry-run=client -o yaml
```

## §6 - Cluster & resource status

```bash
# Are all platform operators healthy?
oc get clusteroperators
oc get co     # short form
oc describe co kube-apiserver

# Cluster version & channel
oc get clusterversion
oc adm upgrade

# Nodes & their roles
oc get nodes
oc get nodes --show-labels
oc get nodes -L node-role.kubernetes.io/worker
oc describe node worker-0 | less

# Resources used per node
oc adm top nodes
oc adm top pods -A
```

## §7 - Events, logs, troubleshooting

```bash
# Events for a namespace (most recent last)
oc get events --sort-by=.lastTimestamp
oc get events -A --field-selector=type=Warning

# A specific pod's logs
oc logs mypod
oc logs mypod -c container2          # multi-container
oc logs deployment/hello             # latest replica
oc logs mypod --previous             # crashed pod's last logs
oc logs -f mypod                     # follow

# Describe = events + spec + status of the object
oc describe pod mypod

# Exec into a container
oc rsh mypod                         # default container
oc rsh -c sidecar mypod
oc exec mypod -- ls /etc

# Open a debug pod (no exec into a broken container — start a new one!)
oc debug pod/mypod                   # same image, no entrypoint
oc debug node/worker-0               # node-level debug pod with `chroot /host`
oc debug deployment/hello --as-root

# Node-level logs (kubelet, crio)
oc adm node-logs worker-0
oc adm node-logs worker-0 -u kubelet
oc adm node-logs worker-0 -u crio --tail=50

# Cluster-wide must-gather (for support tickets / postmortems)
oc adm must-gather --dest-dir=./must-gather-$(date +%s)
```

### Classic troubleshooting flow

```
Pod stuck Pending          → oc describe pod → look at Events → fix scheduling (resources, taints, nodeSelector)
Pod CrashLoopBackOff       → oc logs --previous → fix command/image/env
ImagePullBackOff           → oc describe → wrong image/tag/pull-secret? oc get events for auth errors
Service has no endpoints   → oc get endpoints svc/x → labels mismatch between Service.selector and Pod.labels
Route 503 / 404            → oc get route → wrong service / wrong port / TLS termination wrong
RBAC error                 → oc auth can-i <verb> <resource> -n <ns> --as=<user>
```

## §8 - Monitoring & alerts

OCP 4.18 ships an integrated monitoring stack (Prometheus + Alertmanager + Thanos). You don't manage it deeply on EX280, but you should be able to **find** alerts.

```bash
# Open the console → Observe → Alerting → Alerts
oc whoami --show-console

# CLI: prometheus / alertmanager are operands of the cluster-monitoring-operator
oc -n openshift-monitoring get pods
oc -n openshift-monitoring get prometheuses,alertmanagers

# Query the alerting API (read-only) via the thanos route
oc -n openshift-monitoring get route thanos-querier
```

User-workload monitoring (for your own apps) is opt-in:

```yaml
# cluster-monitoring-config in openshift-monitoring
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

## §9 - OpenShift cluster updates (new emphasis in 4.18 / DO280 v4.18)

```bash
# Available updates on the current channel
oc adm upgrade

# Switch channel (stable-4.18, fast-4.18, candidate-4.18, eus-4.18)
oc adm upgrade channel stable-4.18

# Start an update
oc adm upgrade --to-latest=true
oc adm upgrade --to=4.18.7

# Watch progress
oc get clusterversion -w
oc get clusteroperators

# Pause / resume a MachineConfigPool to stagger node updates
oc patch machineconfigpool worker --type=merge -p '{"spec":{"paused":true}}'
oc patch machineconfigpool worker --type=merge -p '{"spec":{"paused":false}}'
```

> EUS-to-EUS upgrades (Extended Update Support: 4.14 → 4.16 → 4.18) pause workers across two minor versions, then unpause once you're on the target EUS minor. Practice the pause/unpause pattern in CRC.

---

## 🧪 Labs

### Lab 1.1 — Project lifecycle (10 min)

1. Create a project `lab11`.
2. Show its annotations: `oc get project lab11 -o yaml` — note `openshift.io/requester`.
3. Switch to it and create a pod from `quay.io/openshifttest/hello-openshift:1.2.0`.
4. Delete the project; wait for `Terminating` to finish.

Lab 1.1 — Project lifecycle (10 min)

This lab walks through the four moments in a project's life: create, inspect, populate, delete. Each step has a collapsible Solution.

Prerequisites:


Logged in as cluster-admin (or any user with self-provisioner).
oc CLI version close to 4.18.



Step 1 — Create a project named lab11

<details>
<summary>💡 Solution</summary>
bashoc new-project lab11
# Now using project "lab11" on server "https://api.<cluster>:6443".

oc new-project is the user-facing command — it creates the project AND switches your context to it.

Alternative (cluster-admin only):

bashoc create namespace lab11
oc project lab11

The two are not equivalent in OpenShift:


oc new-project goes through the project request API, applies the project template (if one is configured), and sets openshift.io/requester to your username.
oc create namespace bypasses that — no requester annotation, no template processing.


</details>

Step 2 — Show the project's annotations and find openshift.io/requester

<details>
<summary>💡 Solution</summary>
bashoc get project lab11 -o yaml

Look in the metadata.annotations block. Key entries to recognize:

yamlmetadata:
  annotations:
    openshift.io/description: ""
    openshift.io/display-name: ""
    openshift.io/requester: kubeadmin        # ← who asked for this project
    openshift.io/sa.scc.mcs: s0:c26,c10      # SCC MCS labels
    openshift.io/sa.scc.supplemental-groups: 1000680000/10000
    openshift.io/sa.scc.uid-range: 1000680000/10000

Direct extract via jsonpath:

bashoc get project lab11 -o jsonpath='{.metadata.annotations.openshift\.io/requester}{"\n"}'
# kubeadmin

(Note the \. to escape the dot inside the annotation key.)

Why this matters: openshift.io/requester drives the annotationSelector mode of ClusterResourceQuota — you can build "per-developer quota" rules that auto-attach to every project a given user creates.

</details>

Step 3 — Switch into lab11 and run a pod from hello-openshift

<details>
<summary>💡 Solution</summary>
You're already inside lab11 after Step 1. If not:

bashoc project lab11

Create the pod (simplest form):

bashoc run hello --image=quay.io/openshifttest/hello-openshift:1.2.0
# pod/hello created

Verify:

bashoc get pod hello -w
# hello   0/1   ContainerCreating   0   3s
# hello   1/1   Running             0   12s

oc logs hello
# serving on 8080
# serving on 8888

Gotcha — oc run creates a bare Pod, not a Deployment. A bare Pod isn't restarted by a controller if the node fails. On the exam, prefer:

bashoc create deployment hello --image=quay.io/openshifttest/hello-openshift:1.2.0

unless the task explicitly asks for a Pod.

</details>

Step 4 — Delete the project and wait for Terminating to finish

<details>
<summary>💡 Solution</summary>
bashoc delete project lab11
# project.project.openshift.io "lab11" deleted

# Watch the status until the project disappears entirely:
oc get project lab11 -w
# NAME    DISPLAY NAME   STATUS
# lab11                  Terminating
# (... typically 5–20 seconds ...)
# Error from server (NotFound): namespaces "lab11" not found

Alternative — block until deleted (cleaner in scripts):

bashoc delete project lab11 --wait=true

If the project hangs in Terminating forever: there's a stuck finalizer, usually on a custom resource managed by an operator that's no longer running. To diagnose:

bashoc get project lab11 -o json | jq '.spec.finalizers, .metadata.finalizers'
oc get all,pvc,configmap,secret,role,rolebinding -n lab11

Force-removal (use carefully — last resort):

bashoc get project lab11 -o json \
  | jq '.spec.finalizers = [] | .metadata.finalizers = []' \
  | oc replace --raw "/api/v1/namespaces/lab11/finalize" -f -

On the exam, projects normally terminate in seconds. If yours doesn't, suspect a misconfigured operator.

</details>
   
---
### Lab 1.2 — Querying & filtering (15 min)

In `openshift-monitoring`:

1. List all pods sorted by start time.
2. Show only their names and node names (custom columns).
3. List only pods with label `app.kubernetes.io/component=prometheus`.
4. Get the image of the first container of each prometheus pod (jsonpath).

### Lab 1.3 — Imperative → declarative (15 min)

1. `oc create deployment web --image=quay.io/openshifttest/hello-openshift:1.2.0 --replicas=2 --dry-run=client -o yaml > web.yaml`
2. Add a `resources` block (requests/limits).
3. `oc apply -f web.yaml`.
4. Patch `replicas` to 4 in two ways: `oc patch` and `oc edit`.

### Lab 1.4 — Break & fix (25 min)

In a fresh project:

1. Deploy `oc new-app --image=nginx:does-not-exist`.
2. Use `oc describe` / `oc get events` to identify the cause.
3. Fix by patching the image to `nginx:1.27`.
4. Now break it again by setting `replicas=2` and an impossible `resources.requests.cpu: "1000"`. Diagnose with events. Fix.

### Lab 1.5 — Cluster ops sweep (20 min)

1. `oc get clusteroperators` — any `AVAILABLE=False` or `DEGRADED=True`?
2. For one operator, `oc describe co <name>` and read its conditions.
3. `oc adm top nodes` and `oc adm top pods -A | sort -k4 -h | tail`.
4. Find the top 5 noisiest pods by log volume (`oc logs <pod> | wc -l` for a few candidates).

### Lab 1.6 — Cluster update dry-run (CRC) (15 min)

1. `oc adm upgrade` — note current channel & available.
2. `oc adm upgrade channel candidate-4.18` (don't actually upgrade in CRC unless you have time to spare).
3. Pause the `worker` MCP, change a MachineConfig (e.g. add a label), then unpause and watch reconciliation.

---

## ⚡ Drills (timed, no docs)

| Drill | Target time |
|-------|-------------|
| Log in to a cluster from a fresh shell using a token | 30 s |
| Create a project named exactly `team-alpha-prod` | 15 s |
| Get all pods cluster-wide in `Pending` state | 30 s |
| Get the image of the `installer` container in any pod that has one | 60 s |
| Show the last 5 Warning events across all namespaces | 45 s |
| List the `clusteroperators` whose `Available` condition is False | 30 s |
| Tail-follow logs of every container in `deployment/hello` | 30 s |
| Open a debug shell on a worker node and `chroot /host` | 60 s |

---

## 🔗 Docs links to bookmark

- CLI: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/cli_tools/openshift-cli-oc>
- Projects: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/building_applications/projects>
- Image streams: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/images/managing-image-streams>
- Updates: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/updating_clusters/index>
- Troubleshooting: <https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/support/troubleshooting>
