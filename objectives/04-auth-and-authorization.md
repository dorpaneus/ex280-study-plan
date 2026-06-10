# Objective 4 — Manage Authentication and Authorization

> **Exam study points:**
> - Configure the HTPasswd identity provider for authentication
> - Create and delete users
> - Modify user passwords
> - Create and manage groups
> - Modify user and group permissions

This is the single most common exam-task area. Be **fast** at it.

---

## §1 — How OCP authenticates

OCP runs an OAuth server that wraps **Identity Providers (IdPs)**: HTPasswd, LDAP, GitHub, Keystone, OIDC, request-header. EX280 cares about **HTPasswd**.

```
[user] → oauth-openshift route → IdP (HTPasswd) → User + Identity objects → token → API
```

### Object model

| Object | Purpose |
|--------|---------|
| `User` | Cluster-wide user identity |
| `Identity` | Link between IdP + provider username |
| `UserIdentityMapping` | Glues Identity ↔ User |
| `Group` | A named collection of usernames |
| `OAuth/cluster` | Singleton cluster-wide OAuth config |
| `Secret` (htpasswd) | Stores the password hash file, in `openshift-config` |

## §2 — Configure HTPasswd IdP

### Step-by-step

```bash
# 1. Create the htpasswd file locally
htpasswd -c -B -b users.htpasswd alice 'alicepw'
htpasswd -B -b users.htpasswd bob 'bobpw'           # note: no -c (don't recreate)
htpasswd -B -b users.htpasswd carol 'carolpw'

# 2. Create a Secret from it in openshift-config
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config

# 3. Patch the cluster's OAuth resource to add the HTPasswd IdP
oc edit oauth cluster
```

OAuth resource (full example):

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: htpasswd_provider           # any string; what users will see at login
      mappingMethod: claim              # default: claim. Other options: lookup, generate, add
      type: HTPasswd
      htpasswd:
        fileData:
          name: htpasswd-secret
```

Apply (or finish `oc edit`). The OAuth pods restart automatically; give them ~30 s.

```bash
oc get pods -n openshift-authentication
oc get co authentication
oc login -u alice -p alicepw
```

### Updating the htpasswd file later

```bash
# 1. Extract the existing Secret (overwrites users.htpasswd locally)
oc extract secret/htpasswd-secret -n openshift-config --to=. --confirm

# 2. Add / change / remove users
htpasswd -B -b users.htpasswd dave 'davepw'        # add
htpasswd -B -b users.htpasswd alice 'newpassword'  # change password
htpasswd -D users.htpasswd bob                     # delete user

# 3. Replace the Secret in-place
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  --dry-run=client -o yaml \
  -n openshift-config \
  | oc replace -f -
```

OAuth pods reload automatically.

## §3 — Manage users (and the `kubeadmin` story)

```bash
# Users are auto-created on first login when mappingMethod=claim. You can also create them explicitly:
oc create user alice

# Delete (also removes their tokens & mappings — but NOT the htpasswd entry; remove that too with htpasswd -D)
oc delete user alice
oc delete identity htpasswd_provider:alice

# Identities
oc get identity
oc describe identity htpasswd_provider:alice
```

### Removing kubeadmin (do this after you've made yourself cluster-admin!)

```bash
oc adm policy add-cluster-role-to-user cluster-admin alice
oc login -u alice -p alicepw
oc whoami                       # should print alice
oc delete secret kubeadmin -n kube-system
```

On the exam you usually **don't** delete kubeadmin (the proctor expects it).

## §4 — Groups

```bash
# Create
oc adm groups new developers
oc adm groups new ops alice bob

# Membership
oc adm groups add-users developers alice bob
oc adm groups remove-users developers bob

# Inspect
oc get groups
oc describe group developers
oc get group developers -o yaml
```

YAML form (rarely needed on the exam, but good to know):

```yaml
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: developers
users:
  - alice
  - bob
```

## §5 — RBAC: roles & bindings

Two scopes:

| Scope | Role kind | Binding kind |
|-------|-----------|--------------|
| **Cluster-wide** | `ClusterRole` | `ClusterRoleBinding` |
| **Namespace** | `Role` (in that ns) **or** `ClusterRole` | `RoleBinding` (in that ns) |

A `ClusterRole` can be referenced by a `RoleBinding` to grant it only inside one namespace.

### Built-in roles you must know

| Role | Permissions |
|------|-------------|
| `cluster-admin` | Everything |
| `admin` (project admin) | All resources in a project including RBAC |
| `edit` | Read/write most resources, no RBAC, no project itself |
| `view` | Read-only |
| `basic-user` | Get info about own user/groups |
| `self-provisioner` | Can create projects (granted to `system:authenticated:oauth` by default) |
| `cluster-status` | Get cluster status only |

### `oc adm policy` — the easy way

```bash
# Cluster-wide
oc adm policy add-cluster-role-to-user cluster-admin alice
oc adm policy remove-cluster-role-from-user cluster-admin alice
oc adm policy add-cluster-role-to-group view ops

# Project-scoped (current project unless -n given)
oc policy add-role-to-user edit alice -n myapp
oc policy add-role-to-group view ops -n myapp
oc policy remove-role-from-user edit alice -n myapp

# To a Service Account
oc adm policy add-cluster-role-to-user system:openshift:scc:anyuid -z mysa -n myapp

# Inspect what's bound
oc get rolebindings -n myapp
oc get clusterrolebindings | grep alice
oc describe clusterrolebinding cluster-admins
```

### Verifying access (super useful on the exam!)

```bash
oc auth can-i create pods -n myapp                 # as me
oc auth can-i delete deploy -n myapp --as=alice    # as alice
oc auth can-i '*' '*' --as=alice                   # is alice cluster-admin?
oc auth can-i --list -n myapp --as=alice
```

### Authoring a custom Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: myapp
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: myapp
subjects:
  - kind: User
    name: alice
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

## §6 — Disabling self-provisioning (classic exam task)

By default any authenticated user can `oc new-project`. To force admin-driven provisioning:

```bash
# Remove the cluster-wide binding
oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth

# Verify
oc login -u alice -p alicepw
oc new-project test     # → Forbidden
```

You can also tweak the message users see when refused — set `projectRequestMessage` on `Project.config.openshift.io/cluster`:

```yaml
apiVersion: config.openshift.io/v1
kind: Project
metadata:
  name: cluster
spec:
  projectRequestMessage: "Please open a ticket with #platform to request a project."
```

> ⚠️ When you **add** a new IdP and want users to self-provision, you must **re-add** the binding if you previously removed it.

---

## 🧪 Labs

### Lab 4.1 — Configure HTPasswd + first user (25 min)

1. Create users `alice` & `bob` in a local `users.htpasswd` (bcrypt, `-B`).
2. Create Secret `htpasswd-secret` in `openshift-config`.
3. Patch `oauth/cluster` to use the new IdP (call it `local`).
4. Wait for `clusteroperator/authentication` to be `Available=True`.
5. `oc login -u alice -p alicepw` from a separate terminal/profile.

### Lab 4.2 — Promote, demote, change password (15 min)

1. Make `alice` a `cluster-admin`.
2. Log in as alice and create project `team-alpha`.
3. From `kubeadmin`, remove `alice` from `cluster-admin`.
4. Verify she can no longer create projects cluster-wide.
5. Change alice's password (extract → htpasswd → replace). Confirm new password works.

### Lab 4.3 — Groups + project-scoped roles (25 min)

1. Create groups `devs` (alice, bob) and `viewers` (carol).
2. In project `team-alpha`: `devs` get `edit`, `viewers` get `view`.
3. As bob: create a Deployment. ✅
4. As carol: try to create a Deployment. ❌
5. Verify with `oc auth can-i --as=bob` and `--as=carol`.

### Lab 4.4 — Self-provisioning lockdown (15 min)

1. Remove `self-provisioner` from `system:authenticated:oauth`.
2. As alice, try `oc new-project nope` → expect Forbidden.
3. Set `projectRequestMessage` on the Project config to point users to your ticketing system.
4. As alice, confirm the new message appears in the error.
5. Restore `self-provisioner` to authenticated users.

---

## ⚡ Drills (timed, no docs)

| Drill | Target time |
|-------|-------------|
| Configure HTPasswd from scratch (file → secret → oauth patch) | 4 min |
| Add a new user to an existing htpasswd Secret | 90 s |
| Bind `cluster-admin` to a user | 15 s |
| Create a group + add 3 members + give it `edit` in a project | 60 s |
| Verify a user has `delete pods` permission in a namespace | 15 s |
| Disable self-provisioning cluster-wide | 30 s |

---

## ❗ Common exam mistakes

1. **Using MD5 / plaintext in htpasswd** — always use `-B` (bcrypt).
2. **Forgetting `-c`** on the first user creates the file; **using `-c`** on later users wipes the file!
3. **Wrong namespace for the Secret** — it MUST be `openshift-config`.
4. **Forgetting that OAuth pods take ~30 s to reload** — wait for `clusteroperator/authentication` to settle before testing.
5. **Removing self-provisioner but not restoring** when a test scenario expects developers to create their own projects.
6. **Confusing `User` deletion with `Identity` deletion** — both should be cleaned up to fully remove a user.

---

## 🔗 Docs to bookmark

- HTPasswd: `…/authentication_and_authorization/configuring-htpasswd-identity-provider`
- Users & groups: `…/authentication_and_authorization/understanding-authentication`
- RBAC: `…/authentication_and_authorization/using-rbac`
- Removing kubeadmin: `…/authentication_and_authorization/removing-kubeadmin`
