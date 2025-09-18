
# NKP Installation

# NKP Infos:
## Tips:
Iso sollte nicht umbenannt werden!!                                                    => Der Name wird später für die nkp-env benötigt
Rocky image aktuellste aus den Downloads                                       => passend zu der NKP version die installiert werden soll
NKP für Linux muss auch noch geladen werden                                => passend zu der NKP version die installiert werden soll
Ubuntu-image für GPU muss erstellt werden                                     => mit GPU infos erweitern


***Wichtig:***
Updates: Der reihen nach,,nicht versionen Überspringen!!!
Der Cluster Namen muss einzigartig sein! Keine Doppelten Namen! => Lizensierung geht sonst nicht!


## Links:

https://github.com/vEDW/nkp-quickstart
 
[https://github.com/nutanixdev/nkp-quickstart](https://github.com/nutanixdev/nkp-quickstart)  
[https://cluster-api.sigs.k8s.io/introduction](https://cluster-api.sigs.k8s.io/introduction)  
[https://opendocs.nutanix.com/](https://opendocs.nutanix.com/)

Bootcamp:
https://bootcamps.nutanix.com/cloudnative/

---


# Kubernetes Befehl Übersicht

## Befehle


| **Befehl**                                                          | **Funktion**                                  |
| ------------------------------------------------------------------- | --------------------------------------------- |
| kubectl get nodes                                                   | Zeigt die Nodes an                            |
| cp CLUSTERNAME.conf ~/.kube/config                                  | definiert die Kubeconfig                      |
| kubectl get pods                                                    | Zeigt die Pods an                             |
| kubectl get services                                                | Zeigt die Services an                         |
| kubectl get deployments                                             | Zeigt die deployments an                      |
| kubectl get namespaces                                              | Zeigt die namespaces an                       |
| kubectl get sc                                                      | Zeigt den Storage an                          |
| nkp get dashboard                                                   | Zeigt die NKP admin und login an              |
| kubectl get po -A -o wide                                           | Zeigt details der Pods an                     |
| kubectl describe deployments traefik-forward-auth-mgmt -n kommander | Zeigt die config eines deployments an         |
| kubectl logs -n capi-system deployments/capi-controller-manager -f  | Zeigt die Logs fürs deployment                |
| kubectl get nodes -o wide                                           | Zeigt details der Nodes                       |
| kubectl config view                                                 | Zeigt die Kubectl configuration an            |
| MAC: Tilde ~                                                        | Mit Alt-N (Leertaste)                         |
| kubectl get service NAME -n NAMESPACE                               | IP Übersicht mit Ingress-IP                   |
| kubectl get pv -A                                                   | Zeigt das Storage an im K8s Cluster           |
| kubectl get po - NAMESPACE                                          | Zeigt die Applikationen in einem Namespace an |
| kubens NAMESPACE                                                    | wechselt in den Namespace                     |
| kubectx NAMESPACE                                                   | Wechselt auch den Namespace                   |
| kubectl get jobs                                                    | zeigt die laufenden Jobs an                   |
| kubectl logs -f _pod_associated_with_job                            | Zeigt die logs für den Job an                 |
| kubectl get po,pvc                                                  | Zeigt die pods und den PVC an                 |
| alias k='kubectl'                                                   | Alias auf k setzten                           |




---


# Vorbereitung:

### NKP Node OS Image Download

Das Benötigte NKP Node OS Image runterladen und ins Image Repository von PC laden, oder eine Custom image über den Image Builder bauen. 
Links: https://portal.nutanix.com/page/downloads?product=nkp
<img width="634" height="74" alt="Image" src="https://github.com/user-attachments/assets/1d32ccfa-c20d-4c01-a0b5-19f372e4cbf0" />


***Alternative über den Imagebuilder***

```
source nkp-env
nkp create image nutanix ubuntu-22.04 --endpoint 10.8.23.7 --cluster RNO-POC012 --subnet primary-RNO-POC012 --insecure
```

Parameter:
nkp create image nutanix <OS_Name> \
      --cluster <Prism_Element_cluster_name> \
      --endpoint <Prism_Central_endpoint_URL> \              
      --subnet <subnet_name_or_UUID_associated_with_Prism_Element> \               
      --source-image <base_image_name_or_UUID_or_URL>

Link:
https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Kubernetes-Platform-v2_12:top-nib-building-customized-image-t.html

Mit dem NKP Node OS Image wird der Jumhost erstellt. 

### Container anlegen

Storage-Container erstellen, in den später NKP installiert werden soll. 
Ist übersichtlicher, da die pvs VG alle in dem entsprechenden Container liegen. 

### IP-Adressen:

- Summe: min 10-12 IP-Adressen zum Start
- einige statische und aus der DHCP-Range
- Loadbalancer: min2 oder mehr Statische IPs
- CL_Endpoint statisch (VIP-NKP Konsole)

***Management:***
- DHCP für Conrolplane (3)
- DHCP Worker (4) 
- DHCP für 
***Workload Nodes***
- DHCP für 3 Worker

---

### Jumphost VM Create:

VM Deployment über PrismCentral:

NKP Node OS Image (Rocky) runterladen. (https://portal.nutanix.com/page/downloads?product=nkp)

- UEFI
- 1 vCPU / 4 Cores / 8 GB RAM
- 128 GB Disk (anpassen)
- Runtergeladenes NKP Rocky Image nutzen
- Storage-Container und Netzwerk auswählen
- mit Cloud Init script installieren

### Cloud Init script:

```
#cloud-config
fqdn: nkp-quickstart
ssh_pwauth: true
users:
- name: nutanix
  primary_group: nutanix
  groups: nutanix, docker
  lock_passwd: false
  plain_text_passwd: nutanix/4u
bootcmd:
- mkdir -p /etc/docker
write_files:
- content: |
    {
        "insecure-registries": ["registry.nutanixdemo.com"]
    }
  path: /etc/docker/daemon.json
runcmd:
- mv /etc/yum.repos.d/nutanix_rocky9.repo /etc/yum.repos.d/nutanix_rocky9.repo.disabled
- dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
- dnf -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin git tmux
- systemctl --now enable docker
- usermod -aG docker nutanix
- 'curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl'
- chmod +x ./kubectl
- mv ./kubectl /usr/local/bin/kubectl
- 'curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash'
- eject
- 'wall "If you are seeing this message, please reconnect your SSH session. Otherwise, the NKP CLI installation process may fail."'
final_message: "The machine is ready after $UPTIME seconds. Go ahead and install the NKP CLI using: /home/nutanix/nkp-quickstart/scripts/get-nkp-cli.sh"
```


Wenn die VM deployed ist, mit ssh und den Credentials im Cloud-init einloggen. 


---

# Installation

Für die installation einfach die folgenden schritte ausführen:

1. erst den Jumphost installieren (mit dem Cloud init script installieren)
2. auf den Host per SSH verbinden (nutanix/nutanix/4u oder wie im script angegeben)
3. git clone https://github.com/vEDW/nkp-quickstart.git ausführen
4. in den nkp-quickstart ordner wechseln

Unter https://portal.nutanix.com/page/downloads?product=nkp den NKP Link für den Download kopieren

5.  ./get-nkp-cli ausführen              (hier wird nach dem link gefragt)
6.  auf der Shell: nkp version           (version prüfen)
7.  vi nkp-env                           (Konfig file anpassen)
8. . nkp-create-mgmt-cluster.sh          (erstellt den Cluster auf dem Nutanix Cluster)
9.  nkp get dashboard                    (Zugangsdaten anzeigen lassen)
```
nkp get dashboard --kubeconfig="/KONFIG.conf"
```
   
11. fertig !



#### Env-file anpassen:

Anpassungen des files nkp-env im nkp-quickstart Ordner.

### Anpassungen der nkp-env
```
export CLUSTER_NAME=nkp-starter                                           # NKP cluster name. When using NKP Pro/Ultimate, this name is used to generate the license key
export CLUSTER_HOSTNAME=nkp-starter.localhost                             # Hostname for the cluster - this needs to be configured in your DNS server based on first ip in LB_IP_RANGE
export NUTANIX_USER=nkp-admin                                             # Prism Central username for nkp clusterapi integration - avoid admin if possible
export NUTANIX_PASSWORD='supersecretpassword'                             # Keep the password enclosed between single quotes - Ex: 'password'
export NUTANIX_ENDPOINT=10.38.207.7                                       # Prism Central IP address or FQDN without https://
export NUTANIX_PORT=9440                                                  # Prism Central port (default: 9440)
export LB_IP_RANGE=10.38.141.51-10.38.141.52                              # Load balancer IP pool - Ex: 10.42.236.204-10.42.236.204
export CONTROL_PLANE_ENDPOINT_IP=10.38.207.50                             # Kubernetes VIP. Must be in the same subnet as the VMs - Ex: 10.42.236.203
export CONTROL_PLANE_REPLICAS=3                                           # Number of control plane VMs - Ex: 3
export WORKER_NODES_REPLICAS=4                                            # Number of worker VMs - Ex: 4
export NUTANIX_MACHINE_TEMPLATE_IMAGE_NAME=nkp-rocky-9.5-release-cis-1.32.3-20250430150550.qcow2           # Update with the NKP Rocky image name
export NUTANIX_PRISM_ELEMENT_CLUSTER_NAME=PHX-POC207                      # Prism Element cluster name - Ex: PHX-POC207
export NUTANIX_SUBNET_NAME=secondary                                      # Prism Element Subnet on which to deploy the kubernetes cluster
export NUTANIX_STORAGE_CONTAINER_NAME=SelfServiceContainer                # Change to your preferred Prism storage container
export REGISTRY_MIRROR_URL=registry-1.docker.io                           # Required on Nutanix HPOC or internet connected deployments to avoid docker rate limiting
export REGISTRY_MIRROR_USERNAME=                                          # Required on internet connected deployments to avoid docker rate limiting
export REGISTRY_MIRROR_PASSWORD=                                          # Required on internet connected deployments to avoid docker rate limiting
export REGISTRY_URL=registry-1.docker.io                                  # Optional, used to pull images from a private registry
export REGISTRY_USERNAME=                                                 # Optional, used to pull images from a private registry
export REGISTRY_PASSWORD=                                                 # Optional, used to pull images from a private registry
export CSI_HYPERVISOR_ATTACHED=true                                       # Optional, used to enable hypervisor attached volumes for the cluster
export SSH_PUBLIC_KEY_FILE=~/.ssh/id_rsa.pub                              # Optional, used to add your public key to the cluster VMs for SSH access
export CP_CATEGORIES=                                                     # Optional, used to add categories to the control plane resources in Prism Central "key1=value1,key2=value2"
export WORKER_CATEGORIES=                                                 # Optional, used to add categories to the worker resources in Prism Central "key1=value1,key2=value2"
```
Beispiel:
<img width="1306" height="462" alt="Image" src="https://github.com/user-attachments/assets/ce5644f1-4ac5-4f27-836c-b5a6c6ffafe2" />


TIPPS:

für HPOC 
```
export REGISTRY_MIRROR_URL=registry.nutanixdemo.com/docker.io
```

Cluster Host Name rausnehmen, wenn kein DNS da ist oder für Test Zwecke
Docker-USER mit angeben 
Docker-Password anpassen

export REGISTRY_MIRROR_URL=registry.nutanixdemo.com/docker.io
<img width="576" height="404" alt="Image" src="https://github.com/user-attachments/assets/ac739d52-40c0-4988-bdfb-a791319904d1" />

<img width="949" height="539" alt="Image" src="https://github.com/user-attachments/assets/82437d34-b19b-47f4-8c85-95c3bad598de" />

dauert ca. 20 Minuten..



# Troubleshooting

#### Löschen des Bootstrap:
```
nkp delete bootstrap   (kann man die cluster löschen)
```

#### Cluster löschen
```
nkp delte cluster --cluster-name NAME --kubeconfig= --self-managed
```

#### Kubeconfig definieren :
```
cp CLUSTERNAME.conf ~/.kube/config (entspricht dem Namen, der gewählt wurde)
```

Kubectl alias setzten:

```bash
alias k=kubectl
```

#### UUID ausgeben
```
kubectl get namespace kube-system --output jsonpath={.metadata.uid}
```

## Optionale Tools:

***Optional:***

Kubens und Kubectx wird in der Anleitung für den Wechsel der namespaces verwendet:
kubens NAME_SPACE wechselt in den namespace. Ist deutlich einfacher als ständig den Namespace mit anzugeben.

### Download:

```
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
```

### completion script for bash

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





