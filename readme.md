# IBM Event Streams Installation on EKS

This guide provides step-by-step instructions for installing IBM Event Streams Kafka on Amazon EKS (Elastic Kubernetes Service).

## Prerequisites

### 1. EKS Environment
- Reserve an EKS environment from IBM TechZone: https://techzone.ibm.com/my/reservations/create/651c2ff545a5d900171d94d2

### 2. IBM entitlement
- Retrieve the ibm entitlement key from: https://www.ibm.com/docs/en/cloud-paks/cp-integration/16.1.1?topic=fayekoi-finding-applying-your-entitlement-key-by-using-ui-online-installation

### 3. Required Tools Installation

#### Install AWS CLI
Install the AWS CLI following the official guide: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

Verify installation:
```bash
aws --version
```

#### Install kubectl
Install kubectl following the AWS guide: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

Verify installation:
```bash
kubectl version --client
```

#### Install Helm
Install Helm chart manager: https://helm.sh/docs/intro/install/

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Verify installation:
```bash
helm version
```

#### Install IBM Pak (For Offline Installation)
Download IBM Pak from: https://github.com/IBM/ibm-pak/releases

```bash
tar -xvf oc-ibm_pak-darwin-amd64.tar.gz
cp oc-ibm_pak-darwin-amd64 /usr/local/bin/oc-ibm_pak
```

#### Install Skopeo (For Offline Installation)
```bash
brew install skopeo
```

## Configuration

### 1. Configure AWS CLI
```bash
aws configure
# Or export environment variables:
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-key>
export AWS_DEFAULT_REGION=us-east-1
```

### 2. Configure kubectl for EKS
```bash
aws eks update-kubeconfig --region <your-aws-region> --name <your-cluster-name>
# Example:
aws eks update-kubeconfig --region us-east-1 --name itzeks-695000unr5-aaebx7z8
```

Verify connection:
```bash
kubectl get nodes
```

## Installation Methods

### Method 1: Online Installation

#### Step 1: Add IBM Helm Repository
```bash
helm repo add ibm-helm https://raw.githubusercontent.com/IBM/charts/master/repo/ibm-helm
helm repo update
```

#### Step 2: Create Namespace and Secrets
```bash
kubectl create namespace eks-event-streams

kubectl create secret docker-registry ibm-entitlement-key \
  --docker-username=cp \
  --docker-password=<your-ibm-entitlement-key> \
  --docker-server="cp.icr.io" \
  -n eks-event-streams
```

#### Step 3: Install Event Streams Operator
```bash
helm install eventstreams-op ibm-helm/ibm-eventstreams-operator -n eks-event-streams
```

#### Step 4: Verify Installation
```bash
kubectl get deploy eventstreams-cluster-operator -n eks-event-streams -w
```

### Method 2: Offline Installation

#### Step 1: Download Event Streams CASE
```bash
oc-ibm_pak get ibm-eventstreams
```

#### Step 2: Generate Image Manifests
```bash
oc-ibm_pak generate mirror-manifests ibm-eventstreams <target-registry>
```

#### Step 3: Copy Images to Private Registry
Authenticate to both source and target registries:
```bash
skopeo login cp.icr.io -u cp -p <ibm-entitlement-key>
skopeo login <target-registry> -u <username> -p <password>
```

Copy images:
```bash
cat ~/.ibm-pak/data/mirror/ibm-eventstreams/<case-version>/images-mapping.txt | \
  awk -F'=' '{ print "skopeo copy --all docker://"$1" docker://"$2 }' | \
  xargs -S1024 -I {} sh -c 'echo {}; {}'
```

#### Step 4: Create Namespace and Registry Secret
```bash
kubectl create namespace eks-event-streams-offline

kubectl create secret docker-registry artifactory-secret \
  --docker-username=<registry-username> \
  --docker-password=<registry-password> \
  --docker-server="<your-registry>" \
  -n eks-event-streams-offline
```

#### Step 5: Install Event Streams Operator from Local Charts
```bash
helm install eventstreams-op ~/.ibm-pak/data/cases/ibm-eventstreams/3.7.0/charts/ibm-eventstreams-operator-3.7.0.tgz \
  -n eks-event-streams-offline \
  --set imagePullPolicy="Always" \
  --set public.repo=<your-registry> \
  --set public.path="" \
  --set private.repo=<your-registry> \
  --set private.path="" \
  --set imagePullSecrets=artifactory-secret \
  --set watchAnyNamespace=false
```

## Install NGINX Ingress Controller

### Step 1: Add NGINX Helm Repository
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### Step 2: Create Namespace and Install NGINX
```bash
kubectl create namespace ingress-basic

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-basic \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb"
```

### Step 3: Get External IP
```bash
kubectl get svc nginx-ingress-ingress-nginx-controller -n ingress-basic
kubectl get ingressclass
```

## Deploy Event Streams Instance

### Step 1: Download and Configure Event Streams CR
```bash
curl -O https://raw.githubusercontent.com/IBM/ibm-event-automation/main/event-streams/cr-examples/eventstreams/kubernetes/development.yaml
```

Edit the `development.yaml` file and update the host with your external IP using nip.io:
```yaml
spec:
  adminUI:
    endpoints:
    - name: ui
      host: eventstreams-ui.<external-ip>.nip.io
  apicurioRegistry:
    endpoints:
    - name: ui
      host: eventstreams-apicurio.<external-ip>.nip.io
```

### Step 2: Apply Event Streams Configuration
```bash
kubectl apply -f development.yaml -n eks-event-streams
```

### Step 3: Monitor Installation
```bash
kubectl get eventstreams -n eks-event-streams
kubectl get ingress -n eks-event-streams
```

## Create Admin User

### Step 1: Download and Apply Admin User Configuration
```bash
curl -O https://raw.githubusercontent.com/IBM/ibm-event-automation/main/event-streams/cr-examples/eventstreams/kubernetes/es-admin-kafkauser.yaml

kubectl apply -f es-admin-kafkauser.yaml -n eks-event-streams
```

### Step 2: Restart Operator
```bash
kubectl rollout restart deployment/eventstreams-cluster-operator -n eks-event-streams
```

### Step 3: Get Admin Password
```bash
kubectl get secret es-admin -n eks-event-streams -o jsonpath='{.data.password}' | base64 --decode
```

## Testing and Verification

### Test Connection
```bash
openssl s_client -connect kafka-bootstrap.eventstreams.<external-ip>.nip.io:443 -servername kafka-bootstrap.eventstreams.<external-ip>.nip.io
```

### Test with Demo Application
1. Download the demo JAR: https://github.com/ibm-messaging/kafka-java-vertx-starter/releases
2. Download properties file from the Event Streams admin UI
3. Run the demo:
```bash
java -Dproperties_path=<path-to-properties> -jar demo-all.jar
```

## Troubleshooting

### View Ingress Logs
```bash
kubectl logs -f -l app.kubernetes.io/name=ingress-nginx -n ingress-basic
```

### Check Pod Status
```bash
kubectl get pods -n eks-event-streams
kubectl get pods -n ingress-basic
```

### Get User Information
```bash
kubectl get kafkauser -n eks-event-streams
kubectl get secret <username> -n eks-event-streams -o jsonpath="{.data.password}" | base64 -d
```

## Additional Resources

- [IBM Event Automation Documentation](https://www.ibm.com/products/event-automation)
- [Event Streams CR Examples](https://github.com/IBM/ibm-event-automation/tree/main/event-streams/cr-examples/eventstreams/kubernetes)
- [Offline Installation Guide](https://ibm.github.io/event-automation/es/installing/offline/)