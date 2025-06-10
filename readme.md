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

Sample output:
```
aws-cli/2.13.25 Python/3.11.5 Darwin/22.6.0 exe/x86_64 prompt/off
```

#### Install kubectl
Install kubectl following the AWS guide: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

Verify installation:
```bash
kubectl version --client
```

Sample output:
```
Client Version: v1.28.2
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
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

Sample output:
```
version.BuildInfo{Version:"v3.12.3", GitCommit:"3a31588ad33fe3b89af5a2a54ee1d25bfe6eaa5e", GitTreeState:"clean", GoVersion:"go1.20.7"}
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

Sample output:
```
Added new context arn:aws:eks:us-east-1:123456789012:cluster/itzeks-695000unr5-aaebx7z8 to /Users/username/.kube/config
```

Verify connection:
```bash
kubectl get nodes
```

Sample output:
```
NAME                             STATUS   ROLES    AGE   VERSION
ip-10-0-1-123.ec2.internal      Ready    <none>   1d    v1.28.1-eks-43840fb
ip-10-0-2-456.ec2.internal      Ready    <none>   1d    v1.28.1-eks-43840fb
```

## Installation Methods

### Method 1: Online Installation

#### Step 1: Add IBM Helm Repository
```bash
helm repo add ibm-helm https://raw.githubusercontent.com/IBM/charts/master/repo/ibm-helm
helm repo update
```

Sample output:
```
"ibm-helm" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ibm-helm" chart repository
Update Complete. ⎈Happy Helming!⎈
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

Sample output:
```
namespace/eks-event-streams created
secret/ibm-entitlement-key created
```

#### Step 3: Install Event Streams Operator
```bash
helm install eventstreams-op ibm-helm/ibm-eventstreams-operator -n eks-event-streams
```

Sample output:
```
NAME: eventstreams-op
LAST DEPLOYED: Mon Jun 10 10:30:45 2025
NAMESPACE: eks-event-streams
STATUS: deployed
REVISION: 1
```

#### Step 4: Verify Installation
```bash
kubectl get deploy eventstreams-cluster-operator -n eks-event-streams -w
```

Sample output:
```
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
eventstreams-cluster-operator   1/1     1            1           2m30s
```

### Method 2: Offline Installation

#### Step 1: Download Event Streams CASE
```bash
oc-ibm_pak get ibm-eventstreams
```

Sample output:
```
Downloading and extracting the CASE ...
- Success
Retrieving CASE version ...
- Success
Validating the CASE ...
- Success
Creating inventory ...
- Success
Finding inventory items
- Success
Resolving inventory items ...
Parsing inventory items
- Success

CASE inventory summary:
- Total items: 5
- Operators: 1
- Images: 4
```

#### Step 2: Generate Image Manifests
```bash
oc-ibm_pak generate mirror-manifests ibm-eventstreams <target-registry>
```

Example:
```bash
oc-ibm_pak generate mirror-manifests ibm-eventstreams trialibfsn6.jfrog.io/my-event-streams
```

Sample output:
```
Generating mirror manifests ...
- Success
Generated mirror manifests are saved at ~/.ibm-pak/data/mirror/ibm-eventstreams/3.7.0/

To mirror images to your target registry, run:
skopeo copy --all docker://SOURCE docker://TARGET
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
  --set public.path="cpopen/" \
  --set private.repo=<your-registry> \
  --set private.path="cp/" \
  --set imagePullSecrets=artifactory-secret \
  --set watchAnyNamespace=false
```

Verify deployment:
```bash
kubectl get deploy eventstreams-cluster-operator -n <namespace>
```

Sample output:
```
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
eventstreams-cluster-operator   1/1     1            1           3m15s
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
  --set "controller.extraArgs.enable-ssl-passthrough="
```

### Step 3: Get External IP
```bash
kubectl get svc nginx-ingress-ingress-nginx-controller -n ingress-basic
kubectl get ingressclass
```

Sample output:
```
NAME                                     TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller   LoadBalancer   10.100.123.45   a1b2c3d4-ingress-basic-123456-1234567890.us-east-1.elb.amazonaws.com   80:31234/TCP,443:32345/TCP   5m30s

NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       5m30s
```

Get the IP address of the load balancer:
```bash
nslookup a1b2c3d4-ingress-basic-123456-1234567890.us-east-1.elb.amazonaws.com
```

Sample output:
```
Server:    8.8.8.8
Address:   8.8.8.8#53

Non-authoritative answer:
Name:   a1b2c3d4-ingress-basic-123456-1234567890.us-east-1.elb.amazonaws.com
Address: 52.123.45.67
```

## Deploy Event Streams Instance

### Step 1: Download and Configure Event Streams CR
```bash
curl -O https://raw.githubusercontent.com/IBM/ibm-event-automation/main/event-streams/cr-examples/eventstreams/kubernetes/development.yaml
```
More configuration for different environemnts can be found here - https://github.com/IBM/ibm-event-automation/tree/main/event-streams/cr-examples/eventstreams/kubernetes

Edit the `development.yaml` file and update the host with your external IP using nip.io:
```yaml
apiVersion: eventstreams.ibm.com/v1beta2
kind: EventStreams
metadata:
  name: development
  namespace: <namespace>
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
kubectl apply -f development.yaml -n <namespace>
```

Sample output:
```
eventstreams.eventstreams.ibm.com/development created
```

### Step 3: Monitor Installation
```bash
kubectl get eventstreams -n <namespace>
kubectl get ingress -n <namespace>
kubectl get pods -n eks-event-streams-offline # check individual pods 
```

Sample output:
```
NAME          READY   STATUS    PHASE
development   1/1     Ready     Ready

NAME                        CLASS   HOSTS                                    ADDRESS         PORTS     AGE
development-ibm-es-admui    nginx   eventstreams-ui.52.123.45.67.nip.io    52.123.45.67    80, 443   5m
development-ibm-es-apicurio nginx   eventstreams-apicurio.52.123.45.67.nip.io 52.123.45.67 80, 443   5m
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

Sample output:
```
Abc123DefGhi456JklMno789
```

## Testing and Verification

### Test Connection
```bash
openssl s_client -connect kafka-bootstrap.eventstreams.<external-ip>.nip.io:443 -servername kafka-bootstrap.eventstreams.<external-ip>.nip.io
```

### View the Admin Dashboard 
The hostname can be retrieved from the event streams configuration yaml file
Navigate to `https://adminui.<external-ip>.nip.io/`

Enter username `es-admin` and password from the Create Admin section.

Sample admin dashboard view shows:
- Cluster overview with broker status
- Topic management interface
- User and access control settings
- Monitoring metrics and logs

### Generate Credentials and Topics
From the admin UI:
1. Navigate to "Topics" → "Create topic"
2. Go to "Connect to this cluster" → "Generate credentials"
3. Download the properties file for client applications

### Test with Demo Application
1. Download the demo JAR: https://github.com/ibm-messaging/kafka-java-vertx-starter/releases
2. Download properties file from the Event Streams admin UI
3. Check Java version (must be 8 or 11):
```bash
java -version
```

Sample output:
```
openjdk version "11.0.19" 2023-04-18
OpenJDK Runtime Environment (build 11.0.19+7)
OpenJDK 64-Bit Server VM (build 11.0.19+7, mixed mode)
```

4. Run the demo:
```bash
java -Dproperties_path=<path-to-properties> -jar demo-all.jar
```

Sample output:
```
Starting Kafka producer/consumer demo...
Connected to Event Streams cluster
Producer sent: Message 1
Consumer received: Message 1
Producer sent: Message 2
Consumer received: Message 2
```

View the application at `http://localhost:8080`

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

Sample output:
```
NAME                                         READY   STATUS    RESTARTS   AGE
development-entity-operator-12345-abcde     3/3     Running   0          10m
development-kafka-0                          1/1     Running   0          12m
development-kafka-1                          1/1     Running   0          12m
development-kafka-2                          1/1     Running   0          12m
development-zookeeper-0                      1/1     Running   0          15m
```

### Get User Information
```bash
kubectl get kafkauser -n eks-event-streams
kubectl get secret <username> -n eks-event-streams -o jsonpath="{.data.password}" | base64 -d
```

Sample output:
```
NAME       CLUSTER      AUTHENTICATION   AUTHORIZATION   READY
es-admin   development  scram-sha-512    simple          True
test-user  development  scram-sha-512    simple          True

Password123456
```

## Additional Resources

- [IBM Event Automation Documentation](https://www.ibm.com/products/event-automation)
- [Event Streams CR Examples](https://github.com/IBM/ibm-event-automation/tree/main/event-streams/cr-examples/eventstreams/kubernetes)
- [Offline Installation Guide](https://ibm.github.io/event-automation/es/installing/offline/)