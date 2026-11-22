# CKS Practice Questions

Original performance-based practice questions for the Certified Kubernetes Security Specialist (CKS) exam, loosely inspired by the domain breakdown in [CKS-prep](https://github.com/GuillermoW93/CKS-prep). Answers are collapsed — expand to check your work.

---

## Domain: Cluster Setup (10%)

**Q1.** A CIS benchmark scan flagged that the kube-apiserver is running with `--insecure-port=8080`. Fix the static pod manifest on the control plane node to disable the insecure port, then verify the apiserver restarts healthy.

<details><summary>Solution</summary>

```bash
# Edit the manifest (kubelet watches this path and restarts automatically)
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Remove --insecure-port=8080 (or set to 0 on older versions)
# Save and wait ~30s
watch crictl ps   # confirm apiserver container restarts
kubectl get --raw='/healthz'
```
</details>

**Q2.** Using `kubeadm`, a new worker node `worker3` needs to join the cluster. Generate a fresh join command with a short-lived token (TTL 10m) on the control plane, and join the node.

<details><summary>Solution</summary>

```bash
# On control plane
kubeadm token create --ttl 10m --print-join-command

# On worker3, run the printed command, e.g.:
kubeadm join 10.0.0.1:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<hash>
```
</details>

**Q3.** Run `kube-bench` against the control plane node and remediate any FAIL related to `kube-controller-manager` file permissions.

<details><summary>Solution</summary>

```bash
kube-bench run --targets master
# Typical fix:
chmod 600 /etc/kubernetes/manifests/kube-controller-manager.yaml
chown root:root /etc/kubernetes/manifests/kube-controller-manager.yaml
```
</details>

---

## Domain: Cluster Hardening (15%)

**Q4.** In namespace `restricted`, a ServiceAccount `deployer` currently has `cluster-admin` via a ClusterRoleBinding. Replace this with a least-privilege Role that only allows `get`, `list`, `create` on `deployments` and `pods` within that namespace.

<details><summary>Solution</summary>

```bash
kubectl delete clusterrolebinding deployer-admin-binding

kubectl create role deployer-role -n restricted \
  --verb=get,list,create --resource=deployments,pods

kubectl create rolebinding deployer-binding -n restricted \
  --role=deployer-role --serviceaccount=restricted:deployer
```
</details>

**Q5.** Disable anonymous authentication on the kubelet for all nodes, and ensure the kubelet only allows authorized API calls via webhook authorization.

<details><summary>Solution</summary>

```bash
# On each node, edit kubelet config (commonly /var/lib/kubelet/config.yaml)
authentication:
  anonymous:
    enabled: false
authorization:
  mode: Webhook

systemctl restart kubelet
```
</details>

**Q6.** Create a NetworkPolicy in namespace `payments` that denies all ingress and egress traffic by default, then a second policy that allows ingress only from pods labeled `app=frontend` on port 8443.

<details><summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: payments
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8443
```
</details>

**Q7.** A secret `db-creds` was created with `kubectl create secret generic`. Verify whether etcd encryption-at-rest is enabled; if not, configure an `EncryptionConfiguration` using `aescbc` and re-encrypt existing secrets.

<details><summary>Solution</summary>

```bash
# Check apiserver flags
grep encryption-provider-config /etc/kubernetes/manifests/kube-apiserver.yaml

# Create /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64 32-byte key>
      - identity: {}

# Add to kube-apiserver.yaml:
# --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
# mount the /etc/kubernetes/enc volume

# Re-encrypt existing secrets
kubectl get secrets -A -o json | kubectl replace -f -
```
</details>

---

## Domain: System Hardening (15%)

**Q8.** Write a Seccomp profile named `audit.json` that logs (but doesn't block) all syscalls, place it correctly for the kubelet, and apply it to a pod `seccomp-demo`.

<details><summary>Solution</summary>

```bash
mkdir -p /var/lib/kubelet/seccomp/profiles
cat <<EOF > /var/lib/kubelet/seccomp/profiles/audit.json
{"defaultAction": "SCMP_ACT_LOG"}
EOF
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-demo
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: demo
    image: busybox
    command: ["sleep", "3600"]
```
</details>

**Q9.** Harden a pod spec so it runs as a non-root UID 1000, drops all Linux capabilities, and has a read-only root filesystem.

<details><summary>Solution</summary>

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```
</details>

**Q10.** Use `AppArmor` to restrict a container to a profile named `k8s-restricted` that is already loaded on the node. Apply it via annotation (pre-1.30 style) and confirm enforcement.

<details><summary>Solution</summary>

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-restricted
spec:
  containers:
  - name: app
    image: myapp:1.0
```
```bash
kubectl exec <pod> -- cat /proc/1/attr/current
```
</details>

---

## Domain: Minimize Microservice Vulnerabilities (20%)

**Q11.** Enforce Pod Security Standards at the `restricted` level on namespace `secure-apps` using labels (built-in admission controller, no third-party tool).

<details><summary>Solution</summary>

```bash
kubectl label ns secure-apps \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
```
</details>

**Q12.** A deployment mounts a `Secret` as an environment variable. Refactor it to mount the secret as a volume instead (more secure against process-list/env leakage), at path `/etc/secret-vol`, read-only.

<details><summary>Solution</summary>

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secret-vol
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-creds
```
</details>

**Q13.** Write an OPA Gatekeeper `ConstraintTemplate` + `Constraint` that rejects any Pod without `resources.limits.memory` defined.

<details><summary>Solution</summary>

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requirememorylimit
spec:
  crd:
    spec:
      names:
        kind: RequireMemoryLimit
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package requirememorylimit
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits.memory
        msg := sprintf("container %v has no memory limit", [container.name])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireMemoryLimit
metadata:
  name: require-memory-limit
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
```
</details>

**Q14.** Scan the image `myregistry.io/app:1.4` for critical CVEs using Trivy, and fail the pipeline (exit nonzero) only on CRITICAL severity.

<details><summary>Solution</summary>

```bash
trivy image --severity CRITICAL --exit-code 1 myregistry.io/app:1.4
```
</details>

---

## Domain: Supply Chain Security (20%)

**Q15.** Configure an `ImagePolicyWebhook` admission controller so only images from `registry.company.com/` are allowed cluster-wide; all others must be rejected.

<details><summary>Solution</summary>

```yaml
# /etc/kubernetes/admission/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/admission/webhook-kubeconfig.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false

# kube-apiserver flags:
# --enable-admission-plugins=...,ImagePolicyWebhook
# --admission-control-config-file=/etc/kubernetes/admission/admission-config.yaml
```
Webhook backend logic rejects any image not matching `registry.company.com/*`.
</details>

**Q16.** Use `kubesec` or `docker scout`/`trivy config` to statically analyze a Dockerfile that runs as root and uses `ADD` from a URL. Identify and fix the two issues.

<details><summary>Solution</summary>

Issues: running as root, `ADD <url>` (supply-chain risk, no integrity check — prefer `COPY` + checksum verification).

```dockerfile
FROM alpine:3.19
RUN addgroup -S app && adduser -S app -G app
COPY --chown=app:app ./app /app
USER app
ENTRYPOINT ["/app/start.sh"]
```
</details>

**Q17.** Verify the integrity of a container image using cosign (image was signed during CI). Show the command to verify signature against a known public key.

<details><summary>Solution</summary>

```bash
cosign verify --key cosign.pub myregistry.io/app:1.4
```
</details>

---

## Domain: Monitoring, Logging & Runtime Security (20%)

**Q18.** Enable audit logging on the apiserver: log all `delete` requests at `RequestResponse` level, everything else at `Metadata` level, writing to `/var/log/kubernetes/audit.log` with 7-day retention.

<details><summary>Solution</summary>

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  verbs: ["delete"]
- level: Metadata
```
```bash
# kube-apiserver.yaml flags:
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=7
```
</details>

**Q19.** Using Falco, write a custom rule that alerts whenever a shell is spawned inside a container in namespace `production`.

<details><summary>Solution</summary>

```yaml
- rule: Shell Spawned in Production
  desc: Detect shell spawned in production namespace
  condition: >
    spawned_process and container and
    k8s.ns.name = "production" and
    proc.name in (bash, sh, zsh)
  output: >
    Shell spawned in production (user=%user.name container=%container.name
    command=%proc.cmdline)
  priority: WARNING
```
</details>

**Q20.** A pod was compromised and is making suspicious outbound connections. Immediately isolate it at the network layer without deleting the pod (preserve forensic state), then capture a snapshot of running processes inside it.

<details><summary>Solution</summary>

```bash
# Isolate: apply a deny-all NetworkPolicy targeting just this pod's labels
kubectl label pod <pod> forensics=isolate
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-compromised
  namespace: <ns>
spec:
  podSelector:
    matchLabels:
      forensics: isolate
  policyTypes: ["Ingress", "Egress"]
EOF

# Capture process snapshot
kubectl exec <pod> -- ps aux > forensics-ps-snapshot.txt
```
</details>
