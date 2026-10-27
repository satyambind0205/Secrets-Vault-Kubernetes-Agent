# Fetch Vault Secrets in Kubernetes Pods (PASM)

This guide shows how to let a Kubernetes pod fetch a secret from the Vault, either as an **environment variable** or as a **file**. Follow the steps in order. Every step has a command, the expected result, and what to do if it fails.

## How it works

1. You label a pod and add annotations that say which secret it needs.
2. A mutating webhook adds an init container (`secret-v`) and a sidecar (`pasm-sidecar`) to the pod.
3. The init container logs in to the Vault with the pod's service account token. The Vault checks the token with the cluster (TokenReview), then checks the policy.
4. If the policy allows it, the init container checks out the secret.
   - **File mode:** the secret is written to a shared folder inside the pod. The sidecar refreshes the file (every 15 minutes by default).
   - **Environment mode:** a temporary Kubernetes Secret is created and exposed to the container as an environment variable. The sidecar deletes it afterwards. The value is set once when the pod starts and is not refreshed.

## Placeholders

Replace these values with your own everywhere they appear.

| Placeholder | Meaning |
|---|---|
| `<VAULT_IP>` | IP or host name of the Vault |
| `<VAULT_UUID>` | UUID of the Vault (used in the login URL) |
| `<NODE_IP>` | IP address of the Kubernetes control-plane node, as registered in the Vault |
| `<NAMESPACE>` | Kubernetes namespace for your workload |
| `<SERVICE_ACCOUNT>` | Service account the pod runs as |
| `<POLICY_NAME>` | Name of the new Vault policy |
| `<BOX>` / `<SECRET>` | Box name and secret name in the Vault (no dots, see Step 2.5) |
| `<ENV_NAME>` | Environment variable name the container will see |
| `<INIT_IMAGE>` | PASM init image available to the cluster |

## Where to run the commands

Run everything on the Kubernetes control-plane node (needs `kubectl`, `curl`, `openssl`, `python3`):

```bash
ssh root@<NODE_IP>
export KUBECONFIG=/etc/kubernetes/admin.conf
```

The shell variables set in Step 2.1 only live in that terminal. If you open a new session, repeat Step 2.1.

## Prerequisites

- A working Kubernetes cluster, node `Ready`, with the PASM mutating webhook deployed and running.
- The PASM webhook and init images available to the cluster runtime (on a multi-node cluster, push them to a registry).
- The Vault is registered with the cluster (Step 2.3, or already done by an admin). The address registered in the Vault must be listed in the API server certificate SANs, and the Vault must be allowed to reach the API server on TCP 6443.
- A box and a secret already exist in the Vault.
- A Vault admin account.

> Do not open TCP 6443 to all sources. Allow only the Vault addresses.

## Step 1. Check the cluster

```bash
kubectl get nodes
kubectl get pods -A
kubectl get pods,svc -n mutatingwebhook
kubectl get mutatingwebhookconfiguration entrust-pasm.mutatingwebhook.com
```

Expected: node `Ready`, webhook pod `Running`, webhook configuration present, no pod in `Error` or `CrashLoopBackOff`.

Check that the API server is healthy and its certificate contains `<NODE_IP>`:

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -ext subjectAltName
curl --fail --silent --cacert /etc/kubernetes/pki/ca.crt https://<NODE_IP>:6443/readyz; echo
```

Expected: `<NODE_IP>` in the SAN list, and `ok`.

Check that the node can reach the Vault:

```bash
curl -sk -o /dev/null -w "Vault HTTPS status: %{http_code}\n" --max-time 10 https://<VAULT_IP>/
```

Expected: an HTTP status such as `200`. `000` means no connection (check routing and firewall).

## Step 2. Vault setup

### 2.1 Log in as admin

The password is typed at a prompt and is not stored.

```bash
VAULT_IP="<VAULT_IP>"; VAULT_UUID="<VAULT_UUID>"
read -p "Vault admin user: " VAULT_USER
read -s -p "Vault admin password: " VAULT_PASS; echo
VAULT_TOKEN=$(curl -sk -X POST "https://$VAULT_IP/vault/1.0/Login/$VAULT_UUID/" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$VAULT_USER\",\"password\":\"$VAULT_PASS\"}" \
  | python3 -c 'import sys,json; print(json.load(sys.stdin).get("access_token",""))')
unset VAULT_PASS
echo "Token length: ${#VAULT_TOKEN}"
```

Expected: a token length of several hundred characters. `0` means the login failed.

### 2.2 Check whether the cluster is already registered

```bash
curl -k -sS "https://$VAULT_IP/vault/1.0/GetK8sConfiguration/" -H "X-Vault-Auth: $VAULT_TOKEN"; echo
```

If `k8s_url` is `https://<NODE_IP>:6443` and `k8s_status` is `enabled`, the cluster is already registered. Skip to Step 2.4.

If the URL is empty or different, continue with Steps 2.3, 2.3a and 2.3b. Registering again replaces the existing registration, so only do it if you are sure.

### 2.3 Get the cluster CA certificate

The Vault needs the cluster CA certificate to trust the Kubernetes API server (to validate service account tokens). It is the CA that signed the API server certificate.

Confirm the CA file is valid:

```bash
ls -l /etc/kubernetes/pki/ca.crt
openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -subject -issuer -dates
```

Expected: the file exists and the dates show it has not expired.

Convert it to a single-line Base64 string. The Vault accepts either the PEM text or this Base64 form:

```bash
CA_B64=$(base64 -w0 /etc/kubernetes/pki/ca.crt)
echo "CA length: ${#CA_B64}"
```

Expected: a length greater than 0.

If you cannot read `/etc/kubernetes/pki/ca.crt` (for example on a managed cluster), get the same CA from the kubeconfig. This value is already Base64:

```bash
CA_B64=$(kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
echo "CA length: ${#CA_B64}"
```

> Never copy the CA private key (`ca.key`) anywhere.

### 2.3a Allow the Vault to reach the API server

The Vault connects to `https://<NODE_IP>:6443` when you register the cluster and when it validates tokens. On the node, allow only the Vault address (not all sources):

```bash
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="<VAULT_IP>/32" port port="6443" protocol="tcp" accept'
firewall-cmd --reload
firewall-cmd --zone=public --list-rich-rules
```

Skip this if the rule is already listed. Also make sure `<NODE_IP>` is in the API server certificate SANs (Step 1).

### 2.3b Register the cluster in the Vault

Run in the same terminal as Step 2.3 (it uses `CA_B64`) and Step 2.1 (it uses `VAULT_TOKEN`). The `k8s_status` field is required.

```bash
curl -k -sS -w "\nHTTP status: %{http_code}\n" -X POST "https://$VAULT_IP/vault/1.0/SetK8sConfiguration/" \
  -H "Content-Type: application/json" -H "X-Vault-Auth: $VAULT_TOKEN" \
  -d "{\"k8s_url\":\"https://<NODE_IP>:6443\",\"k8s_ca_string\":\"$CA_B64\",\"k8s_status\":\"enabled\"}"
```

Expected: HTTP `200` and `Kubernetes configuration updated successfully`.

The Vault checks the URL and CA by connecting to the API server. A `400` or `500` here usually means the Vault cannot reach `<NODE_IP>:6443` (Step 2.3a), the address is not in the certificate SANs, or the CA is wrong.

### 2.4 Verify the registration

```bash
curl -k -sS "https://$VAULT_IP/vault/1.0/GetK8sConfiguration/" -H "X-Vault-Auth: $VAULT_TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('k8s_url   :', d.get('k8s_url')); print('k8s_status:', d.get('k8s_status')); print('CA length  :', len(d.get('k8s_ca_string','')))"
```

Expected: `k8s_url` is `https://<NODE_IP>:6443`, status `enabled`, CA length greater than 0.

### 2.5 Check the box and secret exist

```bash
curl -k -sS -X POST "https://$VAULT_IP/vault/1.0/ListBoxes/" -H "Content-Type: application/json" \
  -H "X-Vault-Auth: $VAULT_TOKEN" -d '{}' | python3 -m json.tool | grep -E '"(name|box_id)"'
curl -k -sS -X POST "https://$VAULT_IP/vault/1.0/ListSecretIds/" -H "Content-Type: application/json" \
  -H "X-Vault-Auth: $VAULT_TOKEN" -d '{"box_id":"<BOX>"}' | python3 -m json.tool | grep -E '"(name|secret_id)"'
```

Expected: your box and secret are listed.

> The Vault stores an internal ID with a suffix (for example `mybox-dca240`). Use the plain names (`name`) in the policy and in pod annotations.

> **Naming rules.** Pod annotation keys are split on `.`, so a box or secret name that contains a dot does not work. Kubernetes also limits the key name part of an annotation to 63 characters (letters, digits, `-`, `_` and `.`).

### 2.6 Create the policy

This allows one namespace and service account to read one secret.

```bash
curl -k -sS -w "\nHTTP status: %{http_code}\n" -X POST "https://$VAULT_IP/vault/1.0/CreatePolicy/" \
  -H "Content-Type: application/json" -H "X-Vault-Auth: $VAULT_TOKEN" \
  -d '{
    "name": "<POLICY_NAME>",
    "role": "Vault User Role",
    "principals": [ { "k8s_user": { "k8s_namespace": "<NAMESPACE>", "k8s_service_account": "<SERVICE_ACCOUNT>" } } ],
    "resources": [ { "box_id": "<BOX>", "secret_id": ["<SECRET>"] } ]
  }'
```

Expected: HTTP `200` and a `policy_id`.

## Step 3. Kubernetes objects

### 3.1 Namespace and service account

The names must match the Vault policy exactly.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl create namespace <NAMESPACE>
kubectl create serviceaccount <SERVICE_ACCOUNT> -n <NAMESPACE>
```

### 3.2 TokenReview permission

Lets the Vault ask the cluster whether a token is valid.

```bash
kubectl create clusterrolebinding <NAMESPACE>-auth-delegator \
  --clusterrole=system:auth-delegator --serviceaccount=<NAMESPACE>:<SERVICE_ACCOUNT>
kubectl auth can-i create tokenreviews --as=system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>
```

Expected: `yes`. A warning that `tokenreviews` is not namespace scoped is harmless.

### 3.3 Secret permission (environment mode only)

Environment mode creates and deletes one temporary Kubernetes Secret. This Role grants only that. File mode does not need it.

```bash
kubectl apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: secret-mgmt, namespace: <NAMESPACE>}
rules:
  - {apiGroups: [""], resources: ["secrets"], verbs: ["create", "delete"]}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: secret-mgmt, namespace: <NAMESPACE>}
subjects: [{kind: ServiceAccount, name: <SERVICE_ACCOUNT>, namespace: <NAMESPACE>}]
roleRef: {apiGroup: rbac.authorization.k8s.io, kind: Role, name: secret-mgmt}
EOF
kubectl auth can-i create secrets -n <NAMESPACE> --as=system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>
kubectl auth can-i delete secrets -n <NAMESPACE> --as=system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>
```

Expected: `yes` and `yes`.

## Step 4. Check the webhook default Vault

```bash
kubectl get deploy mutatingwebhook -n mutatingwebhook -o jsonpath='{.spec.template.spec.containers[0].env}'; echo
```

The webhook has a **default** Vault (`ENTRUST_VAULT_IPS`, `ENTRUST_VAULT_UUID`). A pod that sets no Vault annotations is sent to that default. If your Vault is different, always set `entrust.vault.ips` and `entrust.vault.uuid` on the pod, as in the examples below.

> The webhook uses `failurePolicy: Ignore`. If the webhook is down, or it fails on a request, the pod starts **without** the secret, so confirm the webhook is `Running` before testing.

> The webhook only needs to act when a pod is created. Use `operations: ["CREATE"]` in the webhook configuration. Kubernetes does not allow adding init containers to an existing pod, so `UPDATE` cannot work.

## Step 5. Fetch the secret into a pod

Both examples only check that the secret is present. They never print its value.

### Option A. Environment variable

Annotation format: `entrust.vault.secret.env.<BOX>.<SECRET>: <ENV_NAME>`

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: env-test
  namespace: <NAMESPACE>
  labels: {entrust.vault.inject.secret: enabled}
  annotations:
    entrust.vault.ips: "<VAULT_IP>"
    entrust.vault.uuid: "<VAULT_UUID>"
    entrust.vault.init.container.url: "<INIT_IMAGE>"
    entrust.vault.secret.env.<BOX>.<SECRET>: <ENV_NAME>
spec:
  serviceAccountName: <SERVICE_ACCOUNT>
  restartPolicy: Never
  containers:
    - name: verify-secret
      image: <INIT_IMAGE>
      imagePullPolicy: IfNotPresent
      command: ["sh", "-c"]
      args: ['test -n "$<ENV_NAME>" && echo secret-present || exit 1']
EOF
```

Wait about 25 seconds, then check:

```bash
kubectl get pod env-test -n <NAMESPACE>
kubectl logs env-test -n <NAMESPACE> -c verify-secret
kubectl logs env-test -n <NAMESPACE> -c secret-v
kubectl logs env-test -n <NAMESPACE> -c pasm-sidecar
```

Expected:

- `verify-secret` prints `secret-present`.
- `secret-v` (init container) log shows the login, the checkout, and `response for <ENV_NAME> k8s secret creation : 201`.
- `pasm-sidecar` log shows `response for k8s <ENV_NAME> secret deletion : 200`.

### Option B. File

Annotation format: `entrust.vault.secret.file.<BOX>.<SECRET>: <folder>/<file name>`

The file is written under `/<folder>/<file name>` inside the container.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: file-test
  namespace: <NAMESPACE>
  labels: {entrust.vault.inject.secret: enabled}
  annotations:
    entrust.vault.ips: "<VAULT_IP>"
    entrust.vault.uuid: "<VAULT_UUID>"
    entrust.vault.init.container.url: "<INIT_IMAGE>"
    entrust.vault.secret.file.<BOX>.<SECRET>: output/secret.txt
spec:
  serviceAccountName: <SERVICE_ACCOUNT>
  restartPolicy: Never
  containers:
    - name: verify-file
      image: <INIT_IMAGE>
      imagePullPolicy: IfNotPresent
      command: ["sh", "-c", "test -s /output/secret.txt && echo secret-file-present || exit 1"]
EOF
```

Wait about 25 seconds, then check:

```bash
kubectl get pod file-test -n <NAMESPACE>
kubectl logs file-test -n <NAMESPACE> -c verify-file
kubectl logs file-test -n <NAMESPACE> -c secret-v
```

Expected: `verify-file` prints `secret-file-present`, and the init log says the secret was "successfully fetched and mounted".

> A pod showing `1/2 NotReady` after the main container finishes is normal. The sidecar keeps running.

## Using it in your own workload

1. Run the pod as `<SERVICE_ACCOUNT>` in `<NAMESPACE>`.
2. Add the label `entrust.vault.inject.secret: enabled`.
3. Add the Vault annotations (`entrust.vault.ips`, `entrust.vault.uuid`, `entrust.vault.init.container.url`).
4. Add one `entrust.vault.secret.env.*` or `entrust.vault.secret.file.*` annotation per secret.
5. Read the environment variable or file from your application container.

For a Deployment, StatefulSet or Job, put the label and the annotations on the **pod template** (`spec.template.metadata`), not on the controller object itself. The webhook reads the pod's `spec.containers`, which exists only on Pod objects.

## Troubleshooting

| Symptom | Likely cause | What to do |
|---|---|---|
| Login token length is `0` | Wrong credentials, UUID or Vault address | Check `<VAULT_IP>`, `<VAULT_UUID>` and the admin account |
| Vault HTTPS status `000` from the node | Routing or firewall | Check the route from the node to the Vault and firewall rules |
| `k8s_status` is not `enabled` or the URL is wrong | Cluster not registered correctly | Register the cluster again (Steps 2.3 to 2.3b) with the right address and CA |
| `SetK8sConfiguration` returns 400 or 500 | Vault cannot reach the API server, address not in certificate SANs, wrong CA, or `k8s_status` missing | Check Step 2.3a, the SANs in Step 1, that the CA matches the API server (Step 2.3), and that `k8s_status` is in the request |
| `CA length` is `0` or `base64: invalid input` | CA file path wrong or empty | Use the kubeconfig method in Step 2.3 |
| Checkout returns 403 or similar | Policy does not match | The namespace, service account, box and secret in the policy must match exactly |
| Pod starts but no secret and no `secret-v` container | Webhook down, label missing, `failurePolicy: Ignore` skipped injection, or the label is on a Deployment instead of the pod template | Check the webhook pod, and that the label `entrust.vault.inject.secret: enabled` is on the pod |
| Pod has its own init container and gets no secret and no `secret-v` | The webhook fails on pods that already have an init container; with `Ignore` the pod starts without secrets | Remove the pod's own init container. This is a known limit. |
| Secret fetched from the wrong Vault | Vault annotations missing, so the webhook default was used | Set `entrust.vault.ips` and `entrust.vault.uuid` on the pod |
| Annotation is ignored or the secret is not found | Box or secret name contains a dot | Use box and secret names without dots |
| Environment secret creation fails | Missing Role for Secrets | Re-check Step 3.3 |
| Environment variable does not change after the secret changes in the Vault | Environment secrets are set once at pod start | Restart the pod. Only file secrets are refreshed. |
| Pod pulls the wrong image or fails to pull | Image not on the node | Import the image into the node runtime or use a registry |

Useful logs:

```bash
kubectl logs <pod> -n <NAMESPACE> -c secret-v        # init container
kubectl logs <pod> -n <NAMESPACE> -c pasm-sidecar    # sidecar (environment mode cleanup, file refresh)
kubectl describe pod <pod> -n <NAMESPACE>
```

## Security notes

- The init container and the sidecar do not verify the Vault TLS certificate. The pod's service account token (sent to the Vault login) and the secrets travel over that connection. Use the Vault only on a trusted network.
- In environment mode, the pod's service account can create and delete Kubernetes Secrets in its namespace (Step 3.3). Keep that Role limited to those two verbs.
- Allow TCP 6443 only from the Vault addresses.

## Clean up

Removes only what you created for testing.

```bash
kubectl delete pod env-test file-test -n <NAMESPACE> --ignore-not-found
```

To remove the whole setup later, delete the namespace, the cluster role binding `<NAMESPACE>-auth-delegator`, and the Vault policy `<POLICY_NAME>`.

## Limits

- Tested with Password and key-value secrets.
- Tested on a single-node Kubernetes cluster. Multi-node clusters need the images in a registry.
- Pods that define their own init containers are not supported. The webhook fails on them and, because of `failurePolicy: Ignore`, the pod starts without secrets.
- Only file secrets are refreshed (every 15 minutes by default, set with `entrust.vault.secret.refresh.duration`). Environment-variable secrets are not refreshed.
- Box and secret names must not contain a dot.
- OpenShift is not covered.
