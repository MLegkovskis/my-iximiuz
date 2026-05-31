# Troubleshooting a Sleepy Init Sequence: Complete Journey

## A Step-by-Step Kubernetes Debugging Adventure

---

## Part 1: The Problem Statement

We had a Pod named `sleepy` in the `challenge` namespace that refused to become healthy. The symptoms:
- Sidecar container (sleepy-sidecar) would start
- App container would crash immediately
- Pod never became ready
- The sidecar was configured as a "native sidecar" with `restartPolicy: Always`

---

## Part 2: The Debugging Process - Commands That Told the Story

### Step 1: Initial Investigation

**First command to run when a Pod won't start:**
```bash
kubectl get pod sleepy -n challenge -o yaml
```

**What this revealed:**
```yaml
spec:
  restartPolicy: Never           # First red flag - explicitly set!
  initContainers:
  - image: ghcr.io/iximiuz/labs/kubernetes-native-sidecars/sleepy-sidecar:v1.0.0
    name: sleepy-sidecar
    restartPolicy: Always        # Native sidecar
  containers:
  - image: ghcr.io/iximiuz/labs/kubernetes-native-sidecars/app:v1.0.0
    name: app
```

**Why this mattered:** The pod had `restartPolicy: Never` but contained a native sidecar that needed to keep running. This mismatch was a key clue.

**Critical detail:** The `restartPolicy: Never` was **explicitly set** in the original Pod spec, not a default. This is important because:
- Default would be `Always` if not specified
- The challenge **intentionally set it to `Never`** to create the problem
- Native sidecars with `restartPolicy: Always` **don't work properly** when pod-level policy is `Never`

---

### Step 2: Understanding Container States

**Next critical command:**
```bash
kubectl describe pod sleepy -n challenge
```

**The telling output:**
```
Init Containers:
  sleepy-sidecar:
    State: Running
    Ready: False                 # Sidecar running but not ready
Containers:
  app:
    State: Terminated
      Reason: Error
      Exit Code: 1              # App crashed immediately
Events:
  Killing: Stopping container sleepy-sidecar  # Sidecar killed!
```

**What this taught us:** The sidecar was running but Kubernetes didn't consider it "ready". When the app crashed, the entire pod was being torn down because of `restartPolicy: Never`.

---

### Step 3: Examining Container Logs

**Checking the app's failure reason:**
```bash
kubectl logs sleepy -n challenge -c app
```
**Output:**
```
Starting app...
Requesting identity...
An error occurred: 503 Server Error: SERVICE UNAVAILABLE for url: http://localhost:4567/identity
```

**Checking the sidecar's perspective:**
```bash
kubectl logs sleepy -n challenge -c sleepy-sidecar
```
**Output:**
```
Starting sleepy sidecar...
 * Running on http://127.0.0.1:4567
127.0.0.1 - - [31/May/2026 19:17:01] "GET /identity HTTP/1.1" 503 -
127.0.0.1 - - [31/May/2026 19:17:02] "GET /identity HTTP/1.1" 503 -
... (16 seconds of 503s) ...
127.0.0.1 - - [31/May/2026 19:17:16] "GET /identity HTTP/1.1" 200 -  # Finally healthy!
```

**The breakthrough:** The sidecar took **16 seconds** to become healthy, but the app tried once, got a 503, and crashed immediately. No retry logic!

---

## Part 3: The Iterative Solutions

### Iteration 1: Adding a Readiness Probe (Failed)

**What we tried:**
```yaml
readinessProbe:
  httpGet:
    path: /identity
    port: 4567
    host: 127.0.0.1
  initialDelaySeconds: 2
  periodSeconds: 1
  failureThreshold: 20
```

**Why it failed:** The `httpGet` probe with `host: 127.0.0.1` doesn't work reliably in all Kubernetes versions due to network namespace isolation.

**The clue from describe:**
```
Readiness probe failed: dial tcp 127.0.0.1:4567: connect: connection refused
```
Even though the container was running and curl worked, the httpGet probe couldn't connect.

---

### Iteration 2: Changing Pod RestartPolicy

**What we learned:** Native sidecars need proper pod-level restart policy.

**Before:**
```yaml
spec:
  restartPolicy: Never  # Wrong!
```

**After:**
```yaml
spec:
  restartPolicy: OnFailure  # Correct for sidecars
```

**Why this matters:** With `restartPolicy: Never`, when the app container fails, the entire pod is marked as failed and **all containers are killed**, including the sidecar. This defeats the purpose of a sidecar that should keep running.

---

### Iteration 3: Adding Retry Logic to the App

**Since we couldn't change the app's code, we wrapped it:**
```yaml
command:
- sh
- -c
- |
  for i in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20; do
    if curl -sf http://127.0.0.1:4567/identity > /dev/null 2>&1; then
      echo "Sidecar is ready! Starting app..."
      python /app/app.py
      exit 0
    fi
    echo "Attempt $i: Sidecar not ready, waiting 2 seconds..."
    sleep 2
  done
  echo "Sidecar never became ready"
  exit 1
```

**What this does:** Polls the sidecar every 2 seconds for up to 40 seconds, only starting the actual app when the sidecar returns 200 OK.

---

### Iteration 4: The Final Fix - Using `exec` Probe

**The winning solution:**
```yaml
readinessProbe:
  exec:
    command:
    - sh
    - -c
    - curl -sf http://127.0.0.1:4567/identity > /dev/null 2>&1
  initialDelaySeconds: 2
  periodSeconds: 1
  failureThreshold: 30
```

**Why this works:** The `exec` probe runs `curl` **inside the container's network namespace**, so `127.0.0.1` resolves correctly. The `httpGet` probe runs from the kubelet's network namespace and may not reach localhost.

---

## Part 4: The Final Working Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sleepy
  namespace: challenge
spec:
  restartPolicy: OnFailure  # Critical for sidecars
  initContainers:
  - name: sleepy-sidecar
    image: ghcr.io/iximiuz/labs/kubernetes-native-sidecars/sleepy-sidecar:v1.0.0
    restartPolicy: Always    # Makes it a native sidecar
    readinessProbe:
      exec:                  # Use exec, not httpGet!
        command:
        - sh
        - -c
        - curl -sf http://127.0.0.1:4567/identity > /dev/null 2>&1
      initialDelaySeconds: 2
      periodSeconds: 1
      failureThreshold: 30   # Give it 30 seconds to become ready
  containers:
  - name: app
    image: ghcr.io/iximiuz/labs/kubernetes-native-sidecars/app:v1.0.0
    command:
    - sh
    - -c
    - |
      for i in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20; do
        if curl -sf http://127.0.0.1:4567/identity > /dev/null 2>&1; then
          python /app/app.py
          exit 0
        fi
        sleep 2
      done
      exit 1
```

---

## Part 5: Key Technical Insights

### Why `curl` Works But `httpGet` Fails

| Probe Type | Runs From | Can Reach `127.0.0.1` |
|------------|-----------|----------------------|
| `httpGet` | kubelet (host network) | ❌ No - different network namespace |
| `exec` | Inside container | ✅ Yes - same network namespace |

### Native Sidecar Requirements Summary

1. **Pod-level restartPolicy** must be `Always` or `OnFailure` (not `Never`)
2. **Init container** must have `restartPolicy: Always`
3. **Readiness probe** required if sidecar needs time to initialize
4. **Use `exec` probe** for services listening on localhost
5. **App may need retry logic** if it starts before sidecar is ready

### The 16-Second Mystery

The sidecar returned 503 for exactly 16 seconds before returning 200. Without proper probes, the app would crash in <1 second. The solution had to wait at least 16 seconds.

**Failure thresholds calculation:**
- `periodSeconds: 1` × `failureThreshold: 30` = 30 seconds max wait
- Enough time for the 16-second initialization window

---

## Part 6: Real-World Debugging Workflow

When a Pod won't start, follow this sequence:

```bash
# 1. Quick status check
kubectl get pod <name> -n <namespace>

# 2. Detailed view (exit codes, events, probe failures)
kubectl describe pod <name> -n <namespace>

# 3. Check logs for each container
kubectl logs <name> -n <namespace> -c <container>

# 4. Check previous instance logs (if restarted)
kubectl logs <name> -n <namespace> -c <container> --previous

# 5. Examine the full spec
kubectl get pod <name> -n <namespace> -o yaml

# 6. Watch real-time status
kubectl get pod <name> -n <namespace> -w
```

---

## Part 7: The "Aha!" Moments

### Moment 1: Seeing the 503→200 Transition
When we saw the sidecar logs showing 16 seconds of 503s followed by 200s, we realized the sidecar wasn't broken - it was just **slow to start**.

### Moment 2: The `httpGet` vs `exec` Discovery
The `describe` output showed `connection refused` even though the container was running. This told us the probe itself was the problem, not the container.

### Moment 3: App Crash Timing
The app crashed in <1 second while the sidecar needed 16 seconds. This revealed the need for **retry logic** and proper **readiness gates**.

---

## Part 8: Commands Summary Table

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `kubectl get pod -o yaml` | View full spec | Start of debugging |
| `kubectl describe pod` | See events and status | After getting basic info |
| `kubectl logs -c <name>` | View container output | When container crashes |
| `kubectl logs --previous` | Check previous run | After restart |
| `kubectl get pod -w` | Watch changes | While testing fixes |
| `kubectl exec -c <name> -- <cmd>` | Test inside container | To verify network/service |

---

## Conclusion

The sleepy sidecar challenge taught us:
1. **Native sidecars need `restartPolicy: Always` in `initContainers`**
2. **Pod-level `restartPolicy` must allow continuation** (not `Never`)
3. **Readiness probes are essential** for slow-starting sidecars
4. **`exec` probes with `curl` are more reliable** than `httpGet` for localhost
5. **Apps may need retry wrappers** when depending on slow sidecars
6. **Always check logs** - they tell the real story

The journey from "pod crashes immediately" to "pod runs successfully" involved understanding Kubernetes probe mechanics, network namespaces, sidecar lifecycles, and the importance of proper retry logic.
