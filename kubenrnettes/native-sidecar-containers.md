# Native Sidecar Containers in Kubernetes - Simplified

## The Big Picture

**Key confusion**: There's NO `sidecarContainers` field in Kubernetes. Native sidecars are just **init containers** with `restartPolicy: Always`.

---

## The Three Container Types at a Glance

### 1. Regular Containers
```yaml
spec:
  containers:
  - name: app
    image: nginx
    # Start at the same time
    # Run forever
    # Can restart independently
```

### 2. Traditional Init Containers
```yaml
spec:
  initContainers:
  - name: setup
    image: busybox
    command: ['sh', '-c', 'echo "Setting up..."']
    # Run to completion (ONCE)
    # Block all subsequent containers
    # No probes allowed
```

### 3. Native Sidecar Containers (NEW)
```yaml
spec:
  initContainers:
  - name: proxy
    image: envoy
    restartPolicy: Always  # THIS makes it a sidecar!
    # Start before regular containers
    # Run continuously alongside them
    # Can have probes!
```

---

## Realistic Example: Web App with Auth Sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-auth
spec:
  restartPolicy: OnFailure
  
  initContainers:
  # 1. Traditional init - runs once, blocks until done
  - name: db-migration
    image: migrate/migrate
    command: ['migrate', 'up', 'database']
    # No restartPolicy means traditional init
    
  # 2. Native sidecar - starts now, runs forever
  - name: auth-sidecar
    image: envoy:latest
    restartPolicy: Always  # Native sidecar!
    ports:
    - containerPort: 8080
    startupProbe:      # Sidecars can have probes
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
  
  # Regular containers - start AFTER sidecar is ready
  containers:
  - name: web-app
    image: nginx
    ports:
    - containerPort: 80
```

---

## How to Spot a Native Sidecar

**Look for these clues**:

1. **Location**: Inside `spec.initContainers` (NOT a separate section)
2. **Restart Policy**: `restartPolicy: Always` (must be explicitly set)
3. **Purpose**: Long-running helper, not one-off setup

```yaml
initContainers:
- name: log-collector
  restartPolicy: Always  # THE MAGIC LINE
  image: fluentd
```

---

## The Nuance: Probes on Init Containers

| Probe Type | Traditional Init | Native Sidecar |
|-----------|----------------|----------------|
| startupProbe | ❌ Not allowed | ✅ Allowed |
| readinessProbe | ❌ Not allowed | ✅ Allowed |
| livenessProbe | ❌ Not allowed | ✅ Allowed |

**Why?** Traditional init containers run once and exit - probes make no sense. Sidecars run continuously, so probes work normally.

---

## Restart Policy Explained

### The Confusing Part:
- **Pod-level**: `spec.restartPolicy` applies to ALL regular containers
- **Container-level**: Only `initContainers` with `restartPolicy: Always` are sidecars

### Three Scenarios:

```yaml
# Scenario 1: Traditional init (runs once, then stops)
initContainers:
- name: setup
  image: busybox
  # No restartPolicy = default OnFailure behavior

# Scenario 2: Native sidecar (runs forever, restarts on crash)
initContainers:
- name: sidecar
  image: envoy
  restartPolicy: Always  # Must be explicitly set!

# Scenario 3: Regular container (uses pod's restartPolicy)
containers:
- name: app
  image: nginx
  # restartPolicy inherited from pod level
```

---

## Execution Flow Example

```yaml
spec:
  initContainers:
  - name: step1           # Runs first, must exit 0
    image: busybox
    command: ['sh', '-c', 'echo "Setup"']
    
  - name: sidecar         # Runs second, but keeps running
    image: envoy
    restartPolicy: Always  # Doesn't block step3!
    
  - name: step3           # Runs third (sidecar already running)
    image: busybox
    command: ['sh', '-c', 'echo "Final init"']
    
  containers:
  - name: main            # Runs last, after ALL init containers
    image: nginx          # (including sidecar) are ready
```

**Flow**:
1. `step1` runs → completes
2. `sidecar` starts → runs continuously
3. `step3` runs (while sidecar is still running!)
4. `main` starts → runs alongside sidecar

---

## Why This Design?

**The clever trick**: Reusing `initContainers` ensures START ORDER:
- All init containers (including sidecars) start BEFORE regular containers
- But sidecars don't BLOCK the next init container
- And they don't BLOCK pod readiness (unless you configure probes)

**No new field needed** - elegant backward compatibility!

---

## Quick Reference: When to Use What

| You need to... | Use |
|----------------|-----|
| Run setup once before app starts | Traditional init |
| Run a proxy alongside your app | Native sidecar |
| Run a log collector that starts first | Native sidecar |
| Just run multiple apps together | Regular containers |
| Wait for a database to be ready | Traditional init |

---

## TL;DR

```yaml
# Traditional init - runs once, blocks everything
initContainers:
- name: wait-for-db
  image: busybox
  
# Native sidecar - starts early, runs forever  
initContainers:
- name: proxy
  restartPolicy: Always  # THIS is the ONLY difference!
  image: envoy
  
# Regular container - normal app
containers:
- name: app
  image: nginx
```

**One line summary**: Native sidecars = init containers with `restartPolicy: Always` that run continuously alongside your main containers instead of exiting.
