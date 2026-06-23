# PASM-Vault-Kubernetes-Agent

## Pulling Secrets to Kubernetes/OpenShift containers from Entrust Secrets Vault

This document focuses on the way one can pull secrets within Kubernetes/OpenShift pods/containers using Entrust’s Secrets vault. For other details on the vault, please refer to the Entrust Keycontrol (KC)
documentation.

The documentation mentions sample kubectl commands. While working with OpenShift clusters, the kubectl needs to be replaced with `oc` .

### Prerequisites
1. Secrets Vault created in Keycontrol.
2. Desired secrets are added in Secrets Vault.
3. Secrets Vault Authentication token for Vault Admin user. (Required for executing APIs on Secrets Vault. Details about obtaining vault login token given below. Also available in apidoc accessible at
`https://<vault-ip>/apidoc under category ‘Vault Auth’)`

### Steps to pull secrets:
Following steps need to be followed to pull secrets in the containers.

#### 1. Logging in to Secrets vault as admin using API and getting Vault authentication token
URL: Usually of the form `https://<vault-ip>/vault/1.0/Login/<vault-uuid>`. This URL is visible on vault management UI of the KeyControl. Also note the UUID from the URL. This will be required in further steps.

#### Request Body:

```json
{
 "username":"admin",
 "password":"password"
}
```

#### Response Body:

```json
{
 "is_user": true,
 "box_admin": false,
 "appliance_id": "ae98ee17-f1df-4ff1-be52-52c74be703c8",
 "access_token": "M2ZkMjkxYjctNTcxYS00Mjk5LThiNzQtZjZhNDM5YjZhNTE2.eyJkYXRhIjoiU0ZSWFVBRUEvRzBXUzZNZlhjY0R5UFRubXFoYjFqUWtRQk1MbUY0MmhIV0N6L3gxTkduM2haMlNZK1JEMlRp
 "expires_at": "2023-03-22T16:34:34.021762Z",
 "admin": true,
 "user": "admin"
}
```

Copy the token received in the response body in the field access_token . This is required while executing APIs described in further steps and is referred as Vault Authentication Token.


#### 2. Configuring Kubernetes cluster within Secrets vault

To configure Kubernetes/OpenShift cluster in vault following API needs to be executed.

URL: `https://<vault-ip>/vault/1.0/SetK8sConfiguration`

#### Essential Headers:
1. `X-Vault-Auth: <Vault-Authentication-Token>`

#### Request Body:

```json
{
 // URL of Kubernetes API server (or Loadbalancer if one is setup in front of
 // control plane) along with port. This URL must be accessible by Secrets vault.
 // If Cluster is behind a service mesh, a proper ingress should be configured 
 // so that API server is accessible.
 "k8s_url": "https://<KC accessible IP/FQDN of K8s API server>:6443",
 
 // Base64 encoded string of the certificate presented by API server
 // during SSL handshake
 "k8s_ca_string": "<base64-encoded-ca-string>"
}

```

The certificate string can be obtained from kubeconfig file or using following kubectl command as follow.
`kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'`
The number 0 must be replaced by appropriate cluster number if kubectl is configured with multiple clusters. Note: The certificate string retrieved as above is already base64 encoded. No need to encode it again.

To obtain certificate details for OpenShift containers describe the secret associated with the service account. The secret name is usually of the form 
<service-account-name>-<random-string>. Following
command can be used to get the secret details:
`oc get secrets/vault-injector-token-2v2l7 -n testmutatingwebhook -o yaml`

Following is the sample output of the command.

```yaml
apiVersion: v1
data:
 ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkekNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdGMyVnkKZG1WeUxXTmhRREUyT0RFNE9UazBNRE1
 namespace: dGVzdG11dGF0aW5nd2ViaG9vaw==
 token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklsaFljRFpsUW5wbWJuTlNYMXB1VFMxcFExQndWR040WTBVeWVYRXRNbVZJVkZWRmMwOVdXbDltWVhNaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwz
kind: Secret
metadata:
 annotations:
 kubernetes.io/service-account.name: vault-injector
 kubernetes.io/service-account.uid: ec873c5f-d3ca-430e-9c18-d736ab41b8b0
 creationTimestamp: "2023-09-08T06:18:32Z"
 name: vault-injector-token-2v2l7
 namespace: testmutatingwebhook
 resourceVersion: "11801690"
 uid: f04c01e5-40c6-48a4-8a75-d77f16996be2
type: kubernetes.io/service-account-token
```

The value of the field ca.crt should be used as `k8s_ca_string`.

#### 3. Deploying and configuring Mutating Webhook

Currently the webhook code is in the form of image and needs to be deployed as container.
First load the provided image mutating-webhook.tar into a local docker using
`docker load --input mutating-webhook.tar`
Then push the image into the registry that is used within Kubernetes with following command.

`docker image push <registry-url>/mutating-webhook:latest`

After pushing the image into the registry deploy the container and service using following webhook YAML specification

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mutatingwebhook
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mutatingwebhook
  namespace: mutatingwebhook
  labels:
    app: mutatingwebhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mutatingwebhook
  template:
    metadata:
      labels:
        app: mutatingwebhook
    spec:
      containers:
      - name: mutatingwebhook
        image: 1.2.3.4:5000/mutating-webhook:latest
        ports:
        - containerPort: 5000
        env:
        # Specify these variables either as a mutatingwebhook container environment variables or 
        # add appropriate annotations in the pod specification. Refer documentation further to get 
        # the details about annotations. The annotations in pod specifications, if specified, take 
        # precedance over these environment variables.
        - name: ENTRUST_VAULT_IPS  
          value: <comma-separated-list-of-vault-ips>
        - name: ENTRUST_VAULT_UUID
          value: <uuid-of-vault-from-which-secrets-to-be-fetched>
        - name: ENTRUST_VAULT_INIT_CONTAINER_URL
          value: <complete-url-of-init-container-image-from-image-registry>
        securityContext:                           #Required, only if deploying on OpenShift
            allowPrivilegeEscalation: false        #Required, only if deploying on OpenShift
            capabilities:                          #Required, only if deploying on OpenShift
              drop:                                #Required, only if deploying on OpenShift
                - ALL                              #Required, only if deploying on OpenShift
      securityContext:                             #Required, only if deploying on OpenShift
        runAsNonRoot: true                         #Required, only if deploying on OpenShift
        runAsUser: 1000700000                      #Required, only if deploying on OpenShift
        seccompProfile:                            #Required, only if deploying on OpenShift
          type: RuntimeDefault                     #Required, only if deploying on OpenShift
---
apiVersion: v1
kind: Service
metadata:
  name: entrust-pasm # DO NOT CHANGE THIS
  namespace: mutatingwebhook # DO NOT CHANGE THIS
spec:
  selector:
    app: mutatingwebhook
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000

```


#### NOTE: If image registry requires authentication, then make sure to add the credentials as secret in Kubernetes and add ‘imagePullSecrets’ tag in the containers section.

After the container is deployed, obtain the webhook server certificate using following command. This certificate will be needed when we configure the webhook in Kubernetes.

`kubectl exec -it -n mutatingwebhook $(kubectl get pods --no-headers -o custom-columns=":metadata.name" -n mutatingwebhook) -- wget -q -O- localhost:8080/ca.pem`

#### NOTE: If you have changed container name while deploying, make sure you replace it at appropriate places in above command.

This will give base64 encoded certificate. With this value, we can now configure webhook in Kubernetes. In following sample YAML spec replace the `caBundle` value with the value obtained from the command and configure the webhook with `kubectl apply`.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: "entrust-pasm.mutatingwebhook.com"
webhooks:
- name: "entrust-pasm.mutatingwebhook.com"
  objectSelector:
    matchLabels:
      entrust.vault.inject.secret: enabled
  rules:
  - apiGroups:   ["*"]
    apiVersions: ["*"]
    operations:  ["CREATE"]
    resources:   ["deployments", "jobs", "pods", "statefulsets"]
  clientConfig:
    service:
      namespace: "mutatingwebhook"
      name: "entrust-pasm"
      path: "/mutate"
      port: 5000
      # Replace value below with the value obtained from command
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVEakNDQXZhZ0F3SUJBZ0lVYUpXeGRoNVFNYlg5ZG5Eb1dLaHNILzBPSHdvd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1RURUxNQWtHQTFVRUJoTUNhSGt4RURBT0JnTlZCQWdUQjBWNFlXMXdiR1V4RFRBTEJnTlZCQWNUQkhCMQpibVV4RURBT0JnTlZCQW9UQjBWNFlXMXdiR1V4Q3pBSkJnTlZCQXNUQWtOQk1CNFhEVEl6TURFeU56RTJORGd3Ck1Gb1hEVFF6TURFeU1qRTJORGd3TUZvd1RURUxNQWtHQTFVRUJoTUNhSGt4RURBT0JnTlZCQWdUQjBWNFlXMXcKYkdVeERUQUxCZ05WQkFjVEJIQjFibVV4RURBT0JnTlZCQW9UQjBWNFlXMXdiR1V4Q3pBSkJnTlZCQXNUQWtOQgpNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQTk1UStEcTJwVDlTb2VGQmYvRU0wCkpFRDNKcFpRV3FzU2hmSGxpaFQ0QUZyVHF6RkRMME5qUGJQS3VHL0tWaUJhMkhTTzJCdHNDam0yZSs5bDlZNlUKbDNKcTFDUWhXOVNDUmx0UFZ1d2dVWnE2TjZXWG1lM29Ld3liT1p1aUJOWTFtYXF2NlBxTG54YUxFVnR0STN5aQpwQjVsOHU1dzlLeDB0YVpSRjNubnNKeHNhcGs5STZFc1U2KzIyQ2FFNXdBMDlvRFo4T3p3Mjl4UXBMeHcvaUxKCllZVHlidGRXdnArbCtFcldwOWF0K1Azc0ZGeFIvUWUreXhYb09YU21HN1pteUVQR0pEM28wbi9uZDlMcTBtem8KZ1RXcEtxeTdNYXMrbURHQTlUbFg4a3NnTjRTSnpMekFGUUJBdlRhR2ZhZVZPWnNPaDZXOE1HZFErQTdMaWVHOQpGUUlEQVFBQm80SGxNSUhpTUE0R0ExVWREd0VCL3dRRUF3SUZvREFkQmdOVkhTVUVGakFVQmdnckJnRUZCUWNECkFRWUlLd1lCQlFVSEF3SXdEQVlEVlIwVEFRSC9CQUl3QURBZEJnTlZIUTRFRmdRVTZFSmRreHJxbVhoOVVlK3IKOElWUVByWXRTbzR3Z1lNR0ExVWRFUVI4TUhxQ0QyVjRZVzF3YkdVdGQyVmlhRzl2YTRJeGJYVjBZWFJwYm1kMwpaV0pvYjI5ckxtMTFkR0YwYVc1bmQyVmlhRzl2YXk1emRtTXVZMngxYzNSbGNpNXNiMk5oYklJamJYVjBZWFJwCmJtZDNaV0pvYjI5ckxtMTFkR0YwYVc1bmQyVmlhRzl2YXk1emRtT0NDV3h2WTJGc2FHOXpkSWNFZndBQUFUQU4KQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBY1ZsaE5xVjNoRmhJS0k3aHZGNXBWWGVOYTV0SUUvSmNDeTJQWWhlTgpmSjlHa2dvWUdzcm1xRkdNZjlBZm9rVkVsL1FjRklyL2NJVDc0ekc1OXBzK2pyQzZhVUlKL2s4OFgrTjVvZjhtCjFqUmZZK2Nld3dIY0FuYnl5K1FEZ3owTXZCODZOc1JFVm9NdmJHeVpzOG4vcmJwWTd0Tk9SUkZwU0N4OGZZMjAKWXllNDdGek8vclZjenVDVVBBNE1taHFmcHA4dlFLditpNXIvbzRtTGsyaWFtR29QTVNXcVppVUtGNjExSktiYgpwQjJkUW9JcHB4WFdodHZnN3NrcDVWdG9wYXc1S3FwanN4aXk2YUdFQVNHV2RhcGlhRzdaVXVPeFg2RzZZd09KCmVFa0JqK0J4VjE2emphcU1ERGYvTWNma3FoLzVRMzBOMk9kOTF6b1I5RkNaQWc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0t
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  timeoutSeconds: 5
```

#### 4. Configuring Policy in Secrets Vault
Next thing is to create a policy in Secrets Vault that will enable Kubernetes service accounts to read secrets from the vault. To create such policy following API needs to be executed.

URL: `https://<vault-ip>/vault/1.0/CreatePolicy/`

#### Essential Headers:

X-Vault-Auth: <Vault-Authentication-Token>

#### Request Body:

```json
{
  "name": "vault_user_policy", // Name of the policy to be created
  "role": "Vault User Role", // Role of user. DO NOT CHANGE THIS
  "principals": [
    {
      "k8s_user": {
        "k8s_namespace": "default", // Namespace in which the service account resides
        "k8s_service_account": "default" // Name of the service account
      }
    }
  ],
  "resources": [
    {
      "box_id": "box1", // Name of the box in which the secret(s) that need access reside. Can be '*' to indicate all boxes
      "secret_id": [ // List of secrets to which access needs to be granted. Can be '*' to indicate all secrets.
        "secret1",
        "secret2"
      ]
    }
  ]
}
```

Couple of points to note here.

The deployment of pod (in which the secrets need to be fetched) must happen by the service accounts mentioned in the policy.

The service accounts added in policy must have system:auth-delegator ClusterRole at the Kubernetes side. This can be done by creating clusterrolebinding with following sample command.

```
kubectl create clusterrolebinding <binding-name> \
 --clusterrole=system:auth-delegator \
 --group=<group-name> \
 --serviceaccount=<namespace>:<serviceaccount>
```

After creating the role binding, the output of following command should be positive.

`kubectl auth can-i create tokenreviews --as=system:serviceaccount:<namespace>:<serviceaccount>`

####5. Pushing Init container image to the registry
This container will serve as init container during the pod deployment that requires the secrets to be fetched.

Push the supplied image file init-container.tar to the image registry using the steps mentioned in section 
#### Configure mutating webhook in Kubernetes.

#### 6. Deploying Pod with Secrets
Secrets can be added to the pod either as volume mounts or as environment variables.

#### 6.1 Adding secrets as volume mounts to the pod
Following is the sample pod specification along with labels and annotations required for successfully pulling secrets.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: testnamespace
  labels:
    app: test
    entrust.vault.inject.secret: enabled # Must have this label
  annotations:
    entrust.vault.ips: <comma-separated-list-of-vault-ips>
    entrust.vault.uuid: <uuid-of-vault-from-which-secrets-to-be-fetched>
    entrust.vault.init.container.url: <complete-url-of-init-container-image-from-image-registry>
    entrust.vault.secret.file.k8s-box.ca-cert: output/cert.pem # box and secret name from vault and value denoting location of secret on the container. The location within container is relative to root (/).   
spec:
  serviceAccountName: k8suser # Service account name configured in Secrets Vault Policy
  containers:
  - name: ubuntu
    image: 10.254.154.247:5000/ubuntu:latest
    command: ['cat','/output/ans.txt']
    imagePullPolicy: IfNotPresent
    securityContext:                           #Required, only if deploying on OpenShift
            allowPrivilegeEscalation: false    #Required, only if deploying on OpenShift
            capabilities:                      #Required, only if deploying on OpenShift
              drop:                            #Required, only if deploying on OpenShift
                - ALL                          #Required, only if deploying on OpenShift
  securityContext:                             #Required, only if deploying on OpenShift
    runAsNonRoot: true                         #Required, only if deploying on OpenShift
    runAsUser: 1000700000                      #Required, only if deploying on OpenShift
    seccompProfile:                            #Required, only if deploying on OpenShift
      type: RuntimeDefault                     #Required, only if deploying on OpenShift
```

In a nutshell, pod specification must have:

1. `entrust.vault.inject.secret`: enabled label to indicate it needs secret.

2. `entrust.vault.ips` annotation that denotes a comma separated list of IPs of vault which are in cluster. This annotation, if specified, will override ENTRUST_VAULT_IPS environment variable from mutating webhook configuration.

3. `entrust.vault.uuid` annotation that denotes UUID of the vault from which we need to fetch the secrets. This annotation, if specified, will override ENTRUST_VAULT_UUID environment variable from mutating webhook configuration.

4. `entrust.vault.init.container.url` annotation that denotes the complete URL of the init container image pushed into image registry. This annotation, if specified, will override ENTRUST_VAULT_INIT_CONTAINER_URL environment variable from mutating webhook configuration.

5. Annotations in the form of `entrust.vault.secret.file.{box-name}.{secret-name}`. The `box-name` , `secret-name` placeholders within the annotation should denote the name of the box and secret name within that box which needs to be pulled. So, for example, if the secret named db-secret from the box k8s-box the name of the annotation would be `entrust.vault.secret.file.k8s-box.db-secret`. The value of the annotation should be the path to the file where the secret needs to be stored within the container. The path should be relative to the / directory. i.e., if the secret needs to be present in /output/cert.pem the value of annotation should be output/cert.pem 

6. In `spec` section, the `serviceAccountName` should have the name of the service account that was added in policy in the Secrets vault.


#### 6.2 Pulling secret as environment variable within Pod
Kubernetes supports initiating environment variables for a container either directly or from Kubernetes secrets. For security purposes, Secrets vault takes the later approach. Also, if secrets are injected using environment variables, a sidecar container will be added which will delete the Kubernetes secrets after successful injection of Secrets Vault secrets as environment variables in main application container.

Since we need to create & delete Kubernetes secrets for this, the service account must also have the required permissions to create & delete the Kubernetes secrets. Following sample command can be used to grant a service account with required permissions.

```
kubectl create rolebinding secretrole --namespace testmutatingwebhook \
--clusterrole=edit --serviceaccount=testmutatingwebhook:test
```

replace the namespace & service account values appropriately.

To check if proper permissions are set up for the service account, following commands can be used.

```
kubectl auth can-i create secrets -n testmutatingwebhook \
--as=system:serviceaccount:testmutatingwebhook:test
```

```
kubectl auth can-i delete secrets -n testmutatingwebhook \
--as=system:serviceaccount:testmutatingwebhook:test
```

The output of both the above commands should be `yes`.

Following is the sample pod specification for deploying a pod with secrets as environment variables.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: testnamespace
  labels:
    app: test
    entrust.vault.inject.secret: enabled # Must have this label
  annotations:
    entrust.vault.ips: <comma-separated-list-of-vault-ips>
    entrust.vault.uuid: <uuid-of-vault-from-which-secrets-to-be-fetched>
    entrust.vault.init.container.url: <complete-url-of-init-container-image-from-image-registry>
    entrust.vault.secret.env.k8s-box.ca-cert: CERT_DATA # box and secret name from vault and value denoting the name of environment variable.   
spec:
  serviceAccountName: k8suser # Service account name configured in Secrets Vault Policy
  containers:
  - name: ubuntu
    image: 10.254.154.247:5000/ubuntu:latest
    command: ["printenv", "CERT_DATA"]
    imagePullPolicy: IfNotPresent
    securityContext:                           #Required, only if deploying on OpenShift
            allowPrivilegeEscalation: false    #Required, only if deploying on OpenShift
            capabilities:                      #Required, only if deploying on OpenShift
              drop:                            #Required, only if deploying on OpenShift
                - ALL                          #Required, only if deploying on OpenShift
  securityContext:                             #Required, only if deploying on OpenShift
    runAsNonRoot: true                         #Required, only if deploying on OpenShift
    runAsUser: 1000700000                      #Required, only if deploying on OpenShift
    seccompProfile:                            #Required, only if deploying on OpenShift
      type: RuntimeDefault                     #Required, only if deploying on OpenShift

```

To pull the secret as environment variable, we must add annotation of the form `entrust.vault.secret.env.{box-name}.{secret-name}`. The value of the annotation should indicate the name of the environment variable in which the secret data is expected to be present. The rest of the annotations are similar as documented in previous section.
