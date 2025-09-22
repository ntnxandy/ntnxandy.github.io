# ***NAI-Installations Workshop*** 

# Infos:

## Links:
- auch eine sehr gute Erklärung:
https://nai.howntnx.win/iep/iep_test/#sample-chat-application

- Demo Katalog im VPN: (husain) 
https://demo.lab.ntnx.pro/

- Installations Repo:
https://github.com/ntnxandy/nai-install.git


## Tipps:

- geht nicht mit Starter - Pro oder Ultimate ist erforderlich
- Sicherstellen, das genug Worker Nodes (min 1) depoyed werden (autoscale aktivieren - 3)
- Worker Nodes mindestens 16-18 vCPU Cores und min 32 GB RAM (mehr ist besser)

***Benötigte Apps:***
- Prometheus-node-exporter
- envoy-Gateway (<font color="#00b050">neu für NAI</font>)
- Kserve


***Optional:***

Kubens und Kubectx wird in der Anleitung für den Wechsel der namespaces verwendet:
kubens NAME_SPACE wechselt in den namespace. Ist deutlich einfacher als ständig den Namespace mit anzugeben.


## Kubens und Kubectx install

### Download:

```
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
```

### completion script for bash:

```
git clone https://github.com/ahmetb/kubectx.git ~/.kubectx
COMPDIR=$(pkg-config --variable=completionsdir bash-completion)
ln -sf ~/.kubectx/completion/kubens.bash $COMPDIR/kubens
ln -sf ~/.kubectx/completion/kubectx.bash $COMPDIR/kubectx
cat << EOF >> ~/.bashrc


#kubectx and kubens
export PATH=~/.kubectx:\$PATH
EOF
```

---


# NAI Pre-requisites:

- Laufenden NTNX-Fileserver
- NFS Share erzeugen (model_share)
	- Compression Enabled
	- Authentication - System
	- Default Access (for all clients) - Read-Write
	- Squash - Root Squash
- Git Repo runterladen (git clone https://github.com/ntnxandy/nai-install.git)

***WORKERNODES für NAI im Nodepool erweitern bevor NAI installiert wird.***


### NAI Docker Download Credential:


https://portal.nutanix.com/page/downloads?product=nai

<img width="1084" height="116" alt="Image" src="https://github.com/user-attachments/assets/fbd144b2-d905-41a7-ba79-a2b6c79494b4" />

```
docker login --username ntnxXXX
dckr_pat_jOFNCXXX 

```

### LLM Model:

Wir verwenden zur Demo folgende Modelle:

- meta-llama/Meta-Llama-3.1-8B-Instruct
- google/gemma-2-2b-it (kleiner für reine CPU Umgebungen)

Bei Huggingface für die LLMs, die verwendet werden sollen wie zum Beispiel meta-llama/Meta-Llama-3.1-8B-Instruct den Zugang requestieren! (Dauert evtl ein paar Stunden)
evtl auch für google/gemma-2-2b-it


<font color="#92d050">From testing`google/gemma-2-2b-it` model is quicker to download and obtain download rights, than `meta-llama/Meta-Llama-3.1-8B-Instruct` model.</font>
<font color="#92d050">Feel free to use the [google/gemma-2-2b-it](https://hf.co/google/gemma-2-2b-it) model if necessary. The procedure to request access to the model is the same.`</font>


- Unter den Usereinstellungen einen Huggingface Token mit Lese-rechten erstellen.

https://huggingface.co/settings/tokens

<img width="1091" height="309" alt="Image" src="https://github.com/user-attachments/assets/85cb4a85-e7c1-4509-8b13-752ff8776927" />

Den Token solltet Ihr Euch merken, den brauchen wir später in der NAI-Gui



#### benötigte Apps:

oy Gateway
Kserve v.0.15.0 in raw deployment
(min Pro Lizenz wird benötigt)
Cert Manager




---

# Installation


1. Auf dem Fileserver den Model-Share erstellen
2. Git Repo runterladen (git clone https://github.com/ntnxandy/nai-install.git)
3. File Storage Class erstellen (manuell)
4. Storage Class erstellen (nai-nfs-storage.yaml ausführen)
```
kubectl get storageclass
```



### Create File Storage Class

Prüfen ob alles Services, Namespaces und Nodes laufen
```
kubectl get nodes
Kubectl get namespaces
kubectl get po -A -o wide
```

#### Git files laden 


```
git clone https://github.com/ntnxandy/nai-install.git
```
install files runterladen oder manuel erstellen.

### NAI-NFS StorageClasse definieren

Nutzen oder erstellen der **nai-nfs-storage.yaml** Datei. 
Anpassen des files mit dem richtigen Fileserver Namen und Share.

manuell:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nai-nfs-storage
parameters:
  nfsPath: /model_share.                    # Change this to your NFS share path
  nfsServer: labFS.ntnxlab.local            # Change this to your Fileserver
  storageType: NutanixFiles
provisioner: csi.nutanix.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```


#### Storage Class erstellen:

```
kubectl apply -f nai-nfs-storage.yaml
```

Prüfen ob die Storage Class erstellt wurde:

```
kubectl get storageclass
```




### Cert Manager installieren

Prüfen ob Cert Manager installiert ist:
```
kubectl get deploy -n cert-manager
```
Output:

<img width="464" height="109" alt="Image" src="https://github.com/user-attachments/assets/6792c9b6-cc63-4ab0-b776-17d51b111b6d" />


Falls nicht - nach installieren
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.4/cert-manager.yaml
```


### Envoy Gateway installieren

Install Envoy Gateway `v1.3.2`

```
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.3.2 -n envoy-gateway-system --create-namespace
```

Output:

<img width="977" height="549" alt="Image" src="https://github.com/user-attachments/assets/a7a59043-8f24-4503-9d24-12dba9be9168" />

Wenn die Installation durch ist, prüfen ob der service läuft:

```
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

Output:

`deployment.apps/envoy-gateway condition met`

### kserve installieren

Prüfen ob kserve installiert ist
```
kubectl get deploy -n kserve
```

```
export KSERVE_VERSION=v0.15.0
```


Install kserve:

```
helm upgrade --install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd --version ${KSERVE_VERSION} -n kserve --create-namespace
```

<img width="1189" height="304" alt="Image" src="https://github.com/user-attachments/assets/0e76f264-211b-4587-bb49-05d35f560115" />

```
helm upgrade --install kserve oci://ghcr.io/kserve/charts/kserve --version ${KSERVE_VERSION} --namespace kserve --create-namespace \
--set kserve.controller.deploymentMode=RawDeployment \
--set kserve.controller.gateway.disableIngressCreation=true
```

<img width="1237" height="247" alt="Image" src="https://github.com/user-attachments/assets/bdd333b3-3182-41c7-8487-86932a43a84d" />

=>> <font color="#92d050">dauert ca 2 minute</font>

Output:

<img width="705" height="442" alt="Image" src="https://github.com/user-attachments/assets/c2c1877a-097b-4c01-9b5c-4e68924b9a3c" />

Checken ob kserve installiert wurde:

```
kubectl get pods -n kserve
```

Wenn kubens installiert wurde könnt ihr das auch mit den beiden Befehlen prüfen:
```
kubens kserve
kubectl get pods
```

Output:

<img width="705" height="54" alt="Image" src="https://github.com/user-attachments/assets/6086ae46-08bc-4e51-95f2-5b017064dd86" />

Make sure both the containers are running for `kserve-controller-manager` pod

=> dauert einige Minuten!!

### NKP Values definieren

Docker Credentials:
(aus dem Nutanix Support Portal)

docker login --username ntnxXXX
dckr_pat_jOFNCJQjQXXX 

git clone https://github.com/ntnxandy/nai-install.git

./env file erstellen:
```
export DOCKER_USERNAME=ntnxsvXXX                                  
export DOCKER_PASSWORD=dckr_pat_jOFXXX         
export NAI_CORE_VERSION=v2.4.0                                      
export NAI_DEFAULT_RWO_STORAGECLASS=nutanix-volume
export NAI_API_RWX_STORAGECLASS=nai-nfs-storage

```

und das erstellte ./env file laden:
```
cp sample.env .env
.env anpassen (dann laden)
source .env
```


<font color="#ff0000">nkp-values.yaml</font> (ist in meinem Git Repo nai_install) - wird später über das install script verwendet
oder selber erstellen:
```
# nai-monitoring stack values for nai-monitoring stack deployment in NKE environment
naiMonitoring:

  ## Component scraping node exporter
  ##
  nodeExporter:
    serviceMonitor:
      enabled: true
      endpoint:
        port: http-metrics
        scheme: http
        targetPort: 9100
      namespaceSelector:
        matchNames:
        - kommander
      serviceSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus-node-exporter
          app.kubernetes.io/component: metrics

  ## Component scraping dcgm exporter
  ##
  dcgmExporter:
    podLevelMetrics: true
    serviceMonitor:
      enabled: true
      endpoint:
        targetPort: 9400
      namespaceSelector:
        matchNames:
        - kommander
      serviceSelector:
        matchLabels:
          app: nvidia-dcgm-exporter
```

alternative:
```
helm repo add ntnx-charts https://nutanix.github.io/helm-releases
helm repo update ntnx-charts
helm pull ntnx-charts/nai-core --version=nai-core-version --untar=true
```

---

# NAI Deloyment

### NAI-Core über Helm
 
<font color="#ff0000">nai-deploy.sh</font> ausführen (von git clone https://github.com/ntnxandy/nai-install.git)

oder selber erstellen:
```
#!/usr/bin/env bash

set -ex
set -o pipefail

helm repo add ntnx-charts https://nutanix.github.io/helm-releases
helm repo update ntnx-charts

#NAI-core
helm upgrade --install nai-core ntnx-charts/nai-core --version=$NAI_CORE_VERSION -n nai-system --create-namespace --wait \
--set imagePullSecret.credentials.username=$DOCKER_USERNAME \
--set imagePullSecret.credentials.password=$DOCKER_PASSWORD \
--insecure-skip-tls-verify \
--set naiApi.storageClassName=$NAI_API_RWX_STORAGECLASS \
--set defaultStorageClassName=$NAI_DEFAULT_RWO_STORAGECLASS \
-f nkp-values.yaml
```


```
./nai-deploy.sh
```

Output:

<img width="641" height="376" alt="Image" src="https://github.com/user-attachments/assets/97ef6f6a-9ed8-46e8-9579-710f4bf11c7c" />

Verify:
```
kubectl get pods -n nai-system
kubectl get po,deploy -n nai-system
```

mit kubens:
```
kubens nai-system
kubectl get po,deploy
```

Output:
<img width="711" height="322" alt="Image" src="https://github.com/user-attachments/assets/d7045085-ccdc-4e2e-b348-29ec9aec4111" />


### Install SSL Certificate and Gateway Elements

#### Public Certificate (CA)
<img width="772" height="208" alt="Image" src="https://github.com/user-attachments/assets/23f17b30-ba74-4b24-a8d0-8a72df762515" />

```
kubectl -n istio-system create secret tls nai-cert --cert=path/to/nai.crt --key=path/to/nai.key
```


#### Selfsigned Zertifikat nutzen:


Endpoint anzeigen lassen NAI_UI_Endpoint:
```
NAI_UI_ENDPOINT=$(kubectl get svc -n envoy-gateway-system -l "gateway.envoyproxy.io/owning-gateway-name=nai-ingress-gateway,gateway.envoyproxy.io/owning-gateway-namespace=nai-system" -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}' | grep -v '^$' || kubectl get svc -n envoy-gateway-system -l "gateway.envoyproxy.io/owning-gateway-name=nai-ingress-gateway,gateway.envoyproxy.io/owning-gateway-namespace=nai-system" -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
```


```
echo $NAI_UI_ENDPOINT
```

Output => IP-Adresse des Endpoint

Nutzen die IP Adresse für den NAI Service

```
nai.${NAI_UI_ENDPOINT}.nip.io
```

Output:

<img width="198" height="27" alt="Image" src="https://github.com/user-attachments/assets/3d8484ff-20a4-43ab-9a8b-8492dc92d1f4" />

#### Ingress Zertifikat erstellen:

deployment über ingress_cert.yaml oder manuell ausführen.

***Vorher den NAI_UI_Endpoint anpassen***
```
kubectl apply -f ingress_cert.yaml
```

oder über .bash
```
cat << EOF | k apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nai-cert
  namespace: nai-system
spec:
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  secretName: nai-cert
  commonName: nai.${NAI_UI_ENDPOINT}.nip.io         
  dnsNames:
  - nai.${NAI_UI_ENDPOINT}.nip.io                   
  ipAddresses:
  - ${NAI_UI_ENDPOINT}                              
EOF
```

	 nai.${NAI_UI_ENDPOINT}.nip.io           #IP-Adress von oben eintragen

<img width="707" height="291" alt="Image" src="https://github.com/user-attachments/assets/3f1fae32-64d0-48b2-93ed-cff876da98d6" />


***Envoy Gateway Patchen:***

```
kubectl patch gateway nai-ingress-gateway -n nai-system --type='json' -p='[{"op": "replace", "path": "/spec/listeners/1/tls/certificateRefs/0/name", "value": "nai-cert"}]'
```




#### Create EnvoyProxy

```
k apply -f -<<EOF
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: envoy-service-config
  namespace: nai-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: LoadBalancer
EOF
```

***oder das file envoy_proxy.yaml nutzen***

```
kubectl apply -f envoy_proxy.yaml
```

Patch nai-ingress-gateway: with EnvoyProxy

.bash
```
kubectl patch gateway nai-ingress-gateway -n nai-system --type=merge \
-p '{
    "spec": {
        "infrastructure": {
            "parametersRef": {
                "group": "gateway.envoyproxy.io",
                "kind": "EnvoyProxy",
                "name": "envoy-service-config"
            }
        }
    }
}'
```

Wenn alles gut gelaufen ist sollten ihr jetzt die NAI Plattform über die IP-Adresse erreichen:
```
https://nai.10.x.x.216.nip.io
```


---

# Nutanix NAI

## Accessing UI of NAI:

Wenn alles geklappt hat solltet ihr unter der IP-Adresse auf die GUI zugreifen können. 

https://nai.NAI_Endpoint_ip.nip.io

<img width="268" height="32" alt="Image" src="https://github.com/user-attachments/assets/1e45f16a-4934-4ce1-836d-98a61c3108c4" />

<img width="593" height="424" alt="Image" src="https://github.com/user-attachments/assets/a4ca1c2f-2d4b-41bb-a071-214844062464" />


***Login default:  admin and  Nutanix.123***

Change admin user Password
Login with admin and new password



## LLM runterladen

1. In the NAI GUI, go to **Models**
2. Click on Import Model from Hugging Face
3. Choose the `meta-llama/Meta-Llama-3.1-8B-Instruct` model
4. Input your Hugging Face token that was created in the previous [section](https://nai.howntnx.win/iep/iep_pre_reqs/#create-a-hugging-face-token-with-read-permissions) and click **Import**
5. Provide the Model Instance Name as `Meta-Llama-3.1-8B-Instruct` and click **Import**

in der Model Access control unter huggingface das gewüschte LLM Model aktivieren. (alternativ alle von NTNX supporteten)

#### Validierung:

**Get jobs in nai-admin namespace

```
kubens nai-admin 
kubectl get jobs
```

Output:
<img width="661" height="217" alt="Image" src="https://github.com/user-attachments/assets/ad91df59-cbb4-4785-96d1-812886d9d462" />

**Validate creation of pods and PVC

```
kubectl get po,pvc
```

Output:
<img width="705" height="198" alt="Image" src="https://github.com/user-attachments/assets/541a66b2-e171-4a99-af13-49d00c80a457" />

**Verify download of model using pod logs

```
kubectl logs -f _pod_associated_with_job
```

Beispiel: (den pod nehmen)
kubectl logs -f nai-113745ed-54b9-40f4-997c-fd-pvc-claim


<img width="924" height="189" alt="Image" src="https://github.com/user-attachments/assets/45ae2eaa-c5a2-45ab-8de6-7a570537d402" />

Output:
<img width="705" height="298" alt="Image" src="https://github.com/user-attachments/assets/e6c09003-01fb-47fd-abe7-ae5b650259d7" />

**Optional - verify the events in the namespace for the pvc creation

```
kubectl get events | awk '{print $1, $3}'
```

Output:
<img width="337" height="356" alt="Image" src="https://github.com/user-attachments/assets/81a54ad5-73e6-48b2-8d47-65ff33f5ce25" />


**NAI v2.3 can bis zu 7 mio paramenter modelle auf CPU only supporten

## Create and Test Inference Endpoint

Inference Endpoints Menü => Create Endpoint

**GPU Access:
- **Endpoint Name**: `llama-8b`
- **Model Instance Name**: `Meta-LLaMA-8B-Instruct`
- **Use GPUs for running the models** : `Checked`
- **No of GPUs (per instance)**:
- **GPU Card**: `NVIDIA-L40S` (or other available GPU)
- **No of Instances**: `1`
- **API Keys**: Create a new API key or use an existing one

**CPU Access:
- **Endpoint Name**: `llama-8b`
- **Model Instance Name**: `Meta-LLaMA-8B-Instruct`
- **Use GPUs for running the models** : `leave unchecked`
- **No of Instances**: `1`
- **API Keys**: Create a new API key or use an existing one

Create klicken

Monitor the nai-admin namespace if servcies are starting

```
kubens nai-admin
kubectl get po,deploy
```

Output:
<img width="691" height="151" alt="Image" src="https://github.com/user-attachments/assets/c8ef1795-a0e8-44be-b986-1e70bbcdf264" />

Check Envents:

```
kubectl get events -n nai-admin --sort-by='.lastTimestamp' | awk '{print $1, $3, $5}'
```

Output:
<img width="493" height="262" alt="Image" src="https://github.com/user-attachments/assets/6105c0e1-9950-40f5-86b5-c5eaab271ffa" />

Check status of inference servcie:

```
kubectl get isvc
```

<img width="695" height="87" alt="Image" src="https://github.com/user-attachments/assets/6c28daba-8387-478d-b784-8581c772b161" />

#### Prüfen

```
kubectl get po,deploy
```

<img width="648" height="118" alt="Image" src="https://github.com/user-attachments/assets/109f701c-470b-41ee-a0c1-6de23a2aec0a" />

```
kubectl logs -f 
```

<img width="963" height="159" alt="Image" src="https://github.com/user-attachments/assets/cfb55a04-a54a-4f71-a059-1a63025ff8fe" />


### Testing

Prepare API Key aus der GUI und auf der CLI den Export ausführen:

```
export API_KEY=_your_endpoint_api_key  
```

***API Key eintragen aus der GUI***

***curl Command zum testen der API: 
 IP-Adresse und endpoint noch anpassen***
 
```

curl -k -X 'POST' 'https://nai.10.x.x.216.nip.io/api/v1/chat/completions' \
-H "Authorization: Bearer $API_KEY" \
-H 'accept: application/json' \
-H 'Content-Type: application/json' \
-d '{
    "model": "ENDPOINT-NAME",
    "messages": [
        {
        "role": "user",
        "content": "What is the capital of France?"
        }
    ],
    "max_tokens": 256,
    "stream": false
}'
```

Output:
<img width="698" height="458" alt="Image" src="https://github.com/user-attachments/assets/2836db5a-6360-4e66-82c6-02c3a598b0ef" />


# Demo Apps

## Chatbot App:

## mit yaml files aus nai-istall/chatbot Ordner

*** Das file chatbot_install.yaml vorher mit der IP anpassen***

hostnames:
  "chat.nai.10.x.x.216.nip.io"    # Input Gateway IP address


Chatbot installation mit kubectl apply -f chatbot_install.yaml
=> Legt automatisch einen Namespace: chat an und erstellt die App

Chatbot löschen mit kubectl delete -f chatbot_delete.yaml
=> Löscht den Namespace mit allem drum und dran :-D



### Manuelle installation:

Create Namespace und ihn den neuen Namespace wechseln
```
kubectl create ns chat
kubens chat
```

Create the App:
```
kubectl apply -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nai-chatapp
  namespace: chat
  labels:
    app: nai-chatapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nai-chatapp
  template:
    metadata:
      labels:
        app: nai-chatapp
    spec:
      containers:
      - name: nai-chatapp
        image: johnugeorge/nai-chatapp:0.12
        ports:
        - containerPort: 8502
EOF
```

Create the service:
```
kubectl apply -f -<<EOF
apiVersion: v1
kind: Service
metadata:
  name: nai-chatapp
  namespace: chat
spec:
  selector:
    app: nai-chatapp
  ports:
    - protocol: TCP
      port: 8502
EOF
```

1. Change this line to point to the IP address of your NAI cluster for the `VirtualService` resource
2. Insert `chat` as the subdomain in the `nai.10.x.x.216.nip.io` main domain.

**chat.nai.10.x.x.216.nip.io

```
kubectl apply -f -<<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nai-chatapp-route
  namespace: chat                   # Same namespace as your chat app service
spec:
  parentRefs:
  - name: nai-ingress-gateway
    namespace: nai-system           # Namespace of the Gateway
  hostnames:
  - "chat.nai.10.x.x.216.nip.io"    # Input Gateway IP address
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: nai-chatapp
      kind: Service
      port: 8502
EOF
```

1. We should be able to see the chat application running on the NAI cluster.
    
2. Input the following:
    
    - Endpoint URL - e.g. `https://nai.10.x.x.216.nip.io/api/v1/chat/completions` (can be found in the Endpoints on NAI GUI)
    - Endpoint Name - e.g. `llama-8b`
    - API key - created during Endpoint creation


