# Kubegrade NetworkPolicy Troubleshooting Script

The following scenario provides a realistic demonstration of **Pod-to-pod communication failure caused by NetworkPolicy**. It simulates a situation where an overly restrictive NetworkPolicy has been applied intentionally to the `backend`, resulting in connection timeouts for the `frontend`.

## Scenario Outline

**The Setup**:
- Namespace: `netpol-demo`
- `backend` Deployment and Service: Serves "Hello from the secured Backend!" on port 80 (target 8080).
- `frontend` Deployment: A shell client running a continuous `curl` loop attempting to connect to the `backend` Service.

**The Problem**:
A NetworkPolicy named `backend-deny-all` was deployed. This policy selects the `backend` pods and designates `policyTypes: [Ingress]`, but it fails to specify any actual `ingress` rules. Due to Kubernetes defaults, this means **all incoming traffic to the backend is denied**.

The `frontend` curl requests will hang and eventually yield "Connection Failed / Timeout". 

**The Fix**:
Kubegrade should analyze the cluster state and observe that:
1. The `frontend` pod and `backend` pod exist.
2. The `backend` pod is healthy and running smoothly.
3. The `backend-deny-all` NetworkPolicy blocks all incoming traffic to `backend`.
4. The correct fix is to apply an explicit Allow rule (like checking label selectors for `app: frontend`).

---

## Step-by-step Demonstration

### 1. Show the Initial Broken State
First, demonstrate that the frontend cannot reach the backend.

```bash
# View the failing logs of the frontend pod to see the timeout cascade
kubectl logs -l app=frontend -n netpol-demo --tail=10 -f

# Explain what should be happening to the audience:
# "Our frontend is supposed to be talking to the backend service, but as you can see, the curl requests are dropping with exit codes / timeouts. Let's see if Kubegrade can find the needle in the haystack."
```

### 2. Invoke Kubegrade Diagnosis
Switch to the Kubegrade interface. Point it at the `netpol-demo` namespace or the `frontend` deployment giving the errors.

Kubegrade should automatically:
- Identify the timeouts from the `frontend`.
- Trace the traffic from `frontend` down to the `backend` service.
- Detect the `backend-deny-all` NetworkPolicy intercepting the connection.

### 3. Review the Proposed Solution
Kubegrade should generate an insight regarding the restrictive `NetworkPolicy`. It will propose either deleting it or fixing it by adding an ingress rule. 

**The fix to apply:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### 4. Verify Resolution
After accepting/applying the fix through Kubegrade or standard `kubectl`, return to the logs to confirm the communication succeeds instantly.

```bash
# Apply the fix (if Kubegrade provides it)
kubectl apply -f /home/ubntu/k8s/network-policy-demo/netpol-fixed.yaml

# Check the logs again
kubectl logs -l app=frontend -n netpol-demo --tail=10 -f

# Expected Output:
# "Trying to reach backend: Hello from the secured Backend!"
```

## Behind the Scenes Setup
(Note: EKS clusters require the Amazon VPC CNI `ENABLE_NETWORK_POLICY=true` flag to actively drop packets. This has already been set up in your current cluster environment).
