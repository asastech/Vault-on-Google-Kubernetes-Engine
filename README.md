# Vault on Google Kubernetes Engine

This README walks you through provisioning a multi-node [HashiCorp Vault](https://www.vaultproject.io) cluster on [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine).

## Cluster Features

* High Availability - The Vault cluster will be provisioned in [multi-server mode](https://www.vaultproject.io/docs/concepts/ha.html) for high availability.
* Google Cloud Storage Storage Backend - Vault's data is persisted in [Google Cloud Storage](https://cloud.google.com/storage).
* Production Hardening - Vault is configured and deployed based on the guidance found in the [production hardening](https://www.vaultproject.io/guides/operations/production.html) guide.
* Auto Initialization and Unsealing - Vault is automatically initialized and unsealed at runtime. Keys are encrypted using [Cloud KMS](https://cloud.google.com/kms) and stored on [Google Cloud Storage](https://cloud.google.com/storage). The unsealing mechanism is done through a sidecar service that routinely checks Vault instances and applies the unseal keys if needed.

## Tutorial

### Enable required APIs

First set your Project ID as an environment variable:

```bash
PROJECT_ID="your-project-id"
```

Enable the following GCP APIs:

```bash
gcloud services enable \
  cloudapis.googleapis.com \
  cloudkms.googleapis.com \
  container.googleapis.com \
  containerregistry.googleapis.com \
  iam.googleapis.com \
  --project ${PROJECT_ID}
```

### Set Configuration

```bash
COMPUTE_ZONE="europe-west4-c"
```

```bash
COMPUTE_REGION="europe-west4"
```

```bash
GCS_BUCKET_NAME="${PROJECT_ID}-vault-storage"
```

```bash
KMS_KEY_ID="projects/${PROJECT_ID}/locations/global/keyRings/vault/cryptoKeys/vault-init"
```

### Create KMS Keyring and Crypto Key

In this section you will create a Cloud KMS [keyring](https://cloud.google.com/kms/docs/object-hierarchy#key_ring) and [cryptographic key](https://cloud.google.com/kms/docs/object-hierarchy#key) suitable for encrypting and decrypting Vault [master keys](https://www.vaultproject.io/docs/concepts/seal.html) and [root tokens](https://www.vaultproject.io/docs/concepts/tokens.html#root-tokens).

Create the `vault` kms keyring:

```bash
gcloud kms keyrings create vault \
  --location global \
  --project ${PROJECT_ID}
```

Create the `vault-init` encryption key:

```bash
gcloud kms keys create vault-init \
  --location global \
  --keyring vault \
  --purpose encryption \
  --project ${PROJECT_ID}
```

### Create a Google Cloud Storage Bucket

Google Cloud Storage is used to [persist Vault's data](https://www.vaultproject.io/docs/configuration/storage/google-cloud-storage.html) and hold encrypted Vault master keys and root tokens.

Create a GCS bucket:

```bash
gsutil mb -p ${PROJECT_ID} gs://${GCS_BUCKET_NAME}
```

### Create the Vault IAM Service Account

An [IAM service account](https://cloud.google.com/iam/docs/service-accounts) is used by Vault to access the GCS bucket and KMS encryption key created in the previous sections.

Create the `vault` service account:

```bash
gcloud iam service-accounts create vault-server \
  --display-name "vault service account" \
  --project ${PROJECT_ID}
```

Grant access to the vault storage bucket:

```bash
gsutil iam ch \
  serviceAccount:vault-server@${PROJECT_ID}.iam.gserviceaccount.com:objectAdmin \
  gs://${GCS_BUCKET_NAME}
```

```bash
gsutil iam ch \
  serviceAccount:vault-server@${PROJECT_ID}.iam.gserviceaccount.com:legacyBucketReader \
  gs://${GCS_BUCKET_NAME}
```

Grant access to the `vault-init` KMS encryption key:

```bash
gcloud kms keys add-iam-policy-binding \
  vault-init \
  --location global \
  --keyring vault \
  --member serviceAccount:vault-server@${PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/cloudkms.cryptoKeyEncrypterDecrypter \
  --project ${PROJECT_ID}
```

### Provision a Kubernetes Cluster

In this section you will provision a three node Kubernetes cluster using [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) with access to the `vault-server` service account across the entire cluster.

Create the `vault` Kubernetes cluster:

```bash
gcloud container clusters create vault \
  --enable-autorepair \
  --machine-type e2-standard-2 \
  --service-account vault-server@${PROJECT_ID}.iam.gserviceaccount.com \
  --num-nodes 3 \
  --zone ${COMPUTE_ZONE} \
  --project ${PROJECT_ID}
```

> Warning: Each node in the `vault` Kubernetes cluster has access to the `vault-server` service account. The `vault` cluster should only be used for running Vault. Other workloads should run on a different cluster and access Vault through an internal or external load balancer.

### Provision IP Address

In this section you will create a public IP address that will be used to expose the Vault server to external clients.

Create the `vault` compute address:

```bash
gcloud compute addresses create vault \
  --region ${COMPUTE_REGION} \
  --project ${PROJECT_ID}
```

Store the `vault` compute address in an environment variable:

```bash
VAULT_LOAD_BALANCER_IP=$(gcloud compute addresses describe vault \
  --region ${COMPUTE_REGION} \
  --project ${PROJECT_ID} \
  --format='value(address)')
```

### Generate TLS Certificates

In this section you will generate the self-signed TLS certificates used to secure communication between Vault clients and servers.

Create a Certificate Authority:

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Generate the Vault TLS certificates:

```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname="vault,vault.default.svc.cluster.local,localhost,127.0.0.1,${VAULT_LOAD_BALANCER_IP}" \
  -profile=default \
  vault-csr.json | cfssljson -bare vault
```

### Deploy Vault

In this section you will deploy the multi-node Vault cluster using a collection of Kubernetes and application configuration files.

Create the `vault` secret to hold the Vault TLS certificates:

```bash
cat vault.pem ca.pem > vault-combined.pem
```

```bash
kubectl create secret generic vault \
  --from-file=ca.pem \
  --from-file=vault.pem=vault-combined.pem \
  --from-file=vault-key.pem
```

The `vault` configmap holds the Google Cloud Platform settings required bootstrap the Vault cluster.

Create the `vault` configmap:

```bash
kubectl create configmap vault \
  --from-literal api-addr=https://${VAULT_LOAD_BALANCER_IP}:8200 \
  --from-literal gcs-bucket-name=${GCS_BUCKET_NAME} \
  --from-literal kms-key-id=${KMS_KEY_ID}
```

#### Create the Vault StatefulSet

In this section you will create the `vault` statefulset used to provision and manage two Vault server instances.

Create the `vault` statefulset:

```bash
kubectl apply -f vault.yaml
```

```bash
service "vault" created
statefulset "vault" created
```

At this point the multi-node cluster is up and running:

```bash
kubectl get pods
```

### Automatic Initialization and Unsealing

In a typical deployment Vault must be initialized and unsealed before it can be used. In our deployment we are using the [vault-init](https://github.com/kelseyhightower/vault-init) container to automate the initialization and unseal steps.

```bash
kubectl logs vault-0 -c vault-init
```

The `vault-init` container runs every 10 seconds and ensures each vault instance is automatically unsealed.

#### Health Checks

A [readiness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes) is used to ensure Vault instances are not routed traffic when they are [sealed](https://www.vaultproject.io/docs/concepts/seal.html).

> Sealed Vault instances do not forward or redirect clients even in HA setups.

### Expose the Vault Cluster

In this section you will expose the Vault cluster using an external network load balancer.

Generate the `vault` service configuration:

```bash
cat > vault-load-balancer.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: vault-load-balancer
spec:
  type: LoadBalancer
  loadBalancerIP: ${VAULT_LOAD_BALANCER_IP}
  ports:
    - name: http
      port: 8200
    - name: server
      port: 8201
  selector:
    app: vault
EOF
```

Create the `vault-load-balancer` service:

```bash
kubectl apply -f vault-load-balancer.yaml
```

Wait until the `EXTERNAL-IP` is populated:

```bash
kubectl get svc vault-load-balancer
```

```bash
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
vault-load-balancer   LoadBalancer   XX.XX.XXX.XXX   <pending>     8200:31805/TCP,8201:32754/TCP
```

### Smoke Tests

Source the `vault.env` script to configure the vault CLI to use the Vault cluster via the external load balancer:

```bash
source vault.env
```

Get the status of the Vault cluster:

```bash
vault status
```

#### Logging in

Download and decrypt the root token:

```bash
export VAULT_TOKEN=$(gsutil cat gs://${GCS_BUCKET_NAME}/root-token.enc | \
  base64 --decode | \
  gcloud kms decrypt \
    --project ${PROJECT_ID} \
    --location global \
    --keyring vault \
    --key vault-init \
    --ciphertext-file - \
    --plaintext-file - 
)
```

#### Working with Secrets

The following examples assume Vault 0.11 or later.

```bash
vault secrets enable -version=2 kv
```

```bash
vault kv enable-versioning secret/
```

```bash
vault kv put secret/my-secret my-value=s3cr3t
```

```bash
vault kv get secret/my-secret
```
