# Kubernetes YAML Structure — CKAD Companion

> **Purpose:** Learn to read and build Kubernetes YAML by understanding the **tree**, not by memorizing indentation.

---

## 1. The One Mental Model

Every Kubernetes YAML document can be understood as a tree.

```text
Object
├── metadata
└── spec
    ├── simple values
    ├── dictionaries / maps
    └── lists
        └── list items
```

When you need to add a field:

1. Identify the Kubernetes object.
2. Find the field's parent/owner.
3. Decide whether the field is a **value**, **map**, or **list**.
4. If it is a list, determine what each `-` represents.
5. If uncertain, check the schema with `kubectl explain`.
6. Apply and verify.

> **Don't memorize indentation. Understand the tree.**

> **YAML warning:** YAML uses spaces, not tab characters, for indentation. Configure your editor to insert spaces.


---

## 2. YAML Has Three Shapes You Must Recognize

### Value / Scalar

```yaml
replicas: 3
image: nginx
```

Some YAML scalars can be interpreted as booleans, numbers, or other native types. Quote a value when the Kubernetes field expects a string and the unquoted form could be ambiguous (for example, a ConfigMap value such as `"NO"` or a version-like string). Do not quote a field merely to change its YAML type when the Kubernetes schema expects an integer or boolean.

One key has one value.

### Dictionary / Map

```yaml
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 500m
```

A key contains named child keys. **No `-` is used.**

### List

```yaml
containers:
  - name: app
    image: nginx
  - name: sidecar
    image: busybox
```

A key contains multiple list items. **Each `-` starts one item.**

---

## 3. The Most Important Distinction: List vs Map

| Shape | Pattern | Typical examples |
|---|---|---|
| Value | `key: value` | `replicas`, `image` |
| Map | `key:` + named keys | `metadata`, `resources`, `selector` |
| List | `key:` + `-` items | `containers`, `env`, `ports` |
| List of maps | each `-` starts an object | `containers`, `env`, `ports`, `volumes` |

### Quick test

Ask:

> **What does each `-` represent?**

For:

```yaml
containers:
  - name: api
  - name: worker
```

each `-` represents **one container object**.

For:

```yaml
resources:
  requests:
    cpu: 100m
```

there is no `-` because `resources` is a **map**.

---

## 4. Common Kubernetes Field Shapes

| Field | Shape | Dash? | Typical location |
|---|---|---:|---|
| `metadata` | Map | No | Object |
| `spec` | Map | No | Object |
| `containers` | List of maps | Yes | Pod spec |
| `env` | List of maps | Yes | Container |
| `ports` | List of maps | Yes | Container |
| `volumeMounts` | List of maps | Yes | Container |
| `volumes` | List of maps | Yes | Pod spec |
| `resources` | Map | No | Container |
| `requests` | Map | No | Resources |
| `limits` | Map | No | Resources |
| `securityContext` | Map | No | Pod/container |
| `selector` | Map | No | Resource spec |

---

## 5. Where Does the Field Belong?

Think about **who owns the setting**.

```text
Object identity
└── metadata

Pod-level configuration
└── spec
    └── template.spec

Container configuration
└── spec.containers[].<field>

Pod volume definition
└── spec.volumes[]

Container's use of a volume
└── spec.containers[].volumeMounts[]
```

This distinction is especially important for:

### `volumes` vs `volumeMounts`

```yaml
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data

  volumes:
    - name: data
      emptyDir: {}
```

- `volumes` defines the volume for the **Pod**.
- `volumeMounts` tells a **container** where to mount it.

---


## 5A. Multi-Document YAML

Multiple Kubernetes objects can share one file by separating documents with `---`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

This is useful when a task requires several related resources in one manifest file.

## 6. The Kubernetes Object Envelope

Most manifests begin with:

```yaml
apiVersion: ...
kind: ...
metadata:
  ...
spec:
  ...
```

Think of the top-level keys as the first branches of the tree.

```text
Object
├── apiVersion
├── kind
├── metadata
└── spec
```

`metadata` describes the object; `spec` contains the desired configuration.

> **YAML syntax trap:** Use spaces for indentation, never tab characters. YAML parsers reject tabs used for indentation.

### Multi-document YAML

Multiple Kubernetes objects can live in one file separated by `---`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
---
apiVersion: v1
kind: Service
metadata:
  name: web
```

This is useful when a task requires a Deployment and its Service in one manifest.

---

## 7. Pod Structure

A Pod commonly has this nesting:

```text
Pod
└── spec
    ├── containers[]
    │   ├── name
    │   ├── image
    │   ├── env[]
    │   ├── ports[]
    │   ├── resources
    │   └── volumeMounts[]
    └── volumes[]
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: nginx
      env:
        - name: APP_MODE
          value: production
      resources:
        requests:
          cpu: 100m
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: {}
```

Notice the repeated pattern:

- `containers` → list
- each container → map
- `env` → list
- `resources` → map
- `volumeMounts` → list
- `volumes` → list

---

## 8. Deployment Nesting

A Deployment adds another level before the Pod specification:

```text
Deployment
└── spec
    └── template
        └── spec
            └── containers[]
```

Typical structure:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: nginx
```

### CKAD trap

Pod-level fields do **not** automatically belong directly under the Deployment's `spec`.

For `apps/v1` Deployments, every key in `spec.selector.matchLabels` must be present in the Pod template's labels. The template may contain additional labels.

For a Deployment, the Pod configuration generally lives under:

```text
spec.template.spec
```

The Deployment selector must match the labels on the Pod template (for `apps/v1`, the selector is required and cannot be changed after creation).

---

## 9. Job and CronJob Nesting

### Job

```text
Job
└── spec
    └── template
        └── spec
            └── containers[]
```

### CronJob

```text
CronJob
└── spec
    └── jobTemplate
        └── spec
            └── template
                └── spec
                    └── containers[]
```

The extra nesting is why blindly copying a Pod manifest into a CronJob often produces misplaced fields.

---

## 10. High-Value Kubernetes Structures

### Environment Variables

```yaml
env:
  - name: APP_MODE
    value: production
```

`env` is a **list of dictionaries**.

### Resources

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

`resources`, `requests`, and `limits` are maps in this structure. No dash is used.

### Probes

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
```

A probe is a **map**, with nested configuration below it.

### Security Context

```yaml
securityContext:
  runAsNonRoot: true
```

A security context is a **map**.

---

## 11. Service Structure

A Service has its own `spec` structure:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: app
  ports:
    - port: 80
      targetPort: 8080
```

Important shapes:

```text
spec
├── selector     → map
└── ports[]      → list of maps
```

Remember:

- `port` = Service port
- `targetPort` = port targeted on the selected Pods; it may be an integer or a named port such as `http-web`

Example named-port mapping:

```yaml
# Pod
ports:
- name: http-web
  containerPort: 8080

# Service
ports:
- port: 80
  targetPort: http-web
```

---

## 12. ConfigMap and Secret

The important structural idea is that configuration data is usually represented as a **map of keys to values**.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: production
  LOG_LEVEL: info
```

The same tree-thinking approach applies to Secrets:

```text
Object
├── metadata
└── data
    ├── key
    └── key
```

The exact field and value format should be confirmed from the resource schema when needed.

---

## 13. PersistentVolumeClaim

A PVC follows the familiar object envelope:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Here:

```text
spec
├── accessModes[]       → list
└── resources           → map
    └── requests        → map
```

The same rule keeps working: identify the type of each node before worrying about indentation.

---

## 14. Common YAML Mistakes

### Mistake 1 — Adding `-` to a map

Wrong:

```yaml
resources:
  - requests:
      cpu: 100m
```

Correct:

```yaml
resources:
  requests:
    cpu: 100m
```

### Mistake 2 — Putting a Pod field at the wrong level

Wrong:

```yaml
spec:
  containers:
    - name: app
      image: nginx
      volumes:
        - name: data
```

Correct:

```yaml
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: {}
```

### Mistake 3 — Forgetting that a list item is itself a map

```yaml
containers:
  - name: app
    image: nginx
```

The `-` does not mean "indent this line." It means:

> **Start one list item.**

Everything belonging to that item is nested beneath it.

---

## 15. Use `kubectl explain` Instead of Guessing

When you know the resource but are unsure about a field:

```bash
kubectl explain deployment.spec
```

For deeper structure:

```bash
kubectl explain deployment.spec.template.spec --recursive
```

Examples:

```bash
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.env
kubectl explain deployment.spec.template.spec.containers
```

### Exam habit

Don't spend several minutes guessing where a field belongs.

**Check the schema, then write the YAML.**

---

## 16. Generate → Modify → Verify

For CKAD, YAML is often faster when you generate a valid starting structure and modify it.

Typical workflow:

```text
Generate / inspect
      ↓
Modify the required field
      ↓
Validate / apply
      ↓
Inspect the resulting object
      ↓
Verify the required behavior
```

Useful inspection commands include:

```bash
kubectl get <resource> <name> -o yaml
kubectl explain <resource>.<field>
kubectl describe <resource> <name>
```

The goal is not to memorize every manifest. The goal is to be able to **construct, modify, inspect, and correct** YAML quickly.

---

## 17. 🧪 Quick Structure Practice

### Exercise 1

Is this valid?

```yaml
resources:
  - limits:
      memory: 256Mi
```

<details>
<summary>💡 Hint</summary>

Is `resources` a list or a map?
</details>

<details>
<summary>✅ Solution</summary>

No. `resources` is a map:

```yaml
resources:
  limits:
    memory: 256Mi
```
</details>

### Exercise 2

Where should the following go?

```yaml
volumeMounts:
  - name: data
    mountPath: /data
```

<details>
<summary>💡 Hint</summary>

Who actually performs the mount?
</details>

<details>
<summary>✅ Solution</summary>

`volumeMounts` belongs to a **container**:

```yaml
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data
```

The Pod-level `volumes` definition is separate.
</details>

### Exercise 3

How many containers are represented?

```yaml
containers:
  - name: api
    image: nginx
  - name: sidecar
    image: busybox
```

<details>
<summary>✅ Solution</summary>

Two. Each `-` starts one container object.
</details>

---

## 18. Final CKAD Decision Tree

```text
Need to add/modify a field
          │
          ▼
What Kubernetes object?
          │
          ▼
Who owns the setting?
          │
          ├── Object → metadata
          ├── Pod → spec / template.spec
          └── Container → containers[].<field>
          │
          ▼
What is the field's type?
          │
          ├── Value → key: value
          ├── Map → key: + named children
          └── List → key: + "-" items
          │
          ▼
What does each "-" represent?
          │
          ▼
Still unsure?
          │
          └── kubectl explain ... --recursive
          │
          ▼
Apply → Inspect → Verify
```

---

## 19. One-Page Memory Sheet

### Remember

```text
VALUE
key: value

MAP
key:
  child: value

LIST
key:
  - item

LIST OF MAPS
key:
  - name: first
    value: ...
  - name: second
    value: ...
```

### Kubernetes

```text
metadata        → object information
spec            → desired configuration
containers[]    → container list
env[]            → environment-variable list
ports[]          → port list
resources       → map
volumeMounts[]   → container mounts
volumes[]        → Pod volumes
selector         → map
```

### Most Important Habit

> **Find the owner → identify the type → build the tree → verify with `kubectl explain`.**
