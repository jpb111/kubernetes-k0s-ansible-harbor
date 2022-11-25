# Installing Harbor on Kubernetes

## Introduction 

Local Helm repositories offer privacy and security that helps organizations share Helm charts throughout the organization with fine access control. 

## Advantages of Helm chart

By making use of Helm, business will immediately benefit from:

* Greatly improved productivity
* Reduced complexity of deployments
* Implementation of cloud-native applications
* More reproducible deployments and results
* Ability to leverage Kubernetes with a single CLI command
* Better scalability
* Ability to re-use Helm charts across multiple environments
* More streamlined CI/CD pipeline
* Easier rolling back to previous versions of an app


In this guide,  Harbor is deployed to Kubernetes as a local helm repository. A hello-kube python application is deployed to kubernetes using the harbor image registry. A helm chart is created for the hello-kube app and is uploaded to the harbor helm charts for the application deployment. k0s kubernetes clusters are deployed using Ansible on Ubuntu instances created using multipass. 

### [Harbor](https://goharbor.io/)

Harbor is a open source registry for containers and Helm charts. 

### [Metallb](https://metallb.universe.tf/)

MetalLB is a load-balancer implementation for bare metal Kubernetes clusters

### [Longhorn](https://longhorn.io/) 

Longhorn is a lightweight, reliable, and powerful distributed open source block storage system for Kubernetes.

### [Helm](https://helm.sh/) 

Helm Charts help you define, install, and upgrade even the most complex Kubernetes application.

### [Docker](https://www.docker.com/)

Docker is a set of platform as a service product that use OS-level virtualization to deliver software in packages called containers.

### [Ansible](https://www.ansible.com/)

Ansible is an open sourceIT automation tool that automates provisioning, configuration management, application deployment, orchestration, and many other manual IT processes.

The Ansible-KOs deployment playbook used here to deploy k0s kubernetes clusters are largely based on the extensive and outstanding work of the contributors of [k3s-ansible](https://github.com/k3s-io/k3s-ansible), [kubespray](https://github.com/kubernetes-sigs/kubespray), [movd](https://github.com/movd/k0s-ansible). 


## Prerequisite Mac

- Install Multipass
- Install Kubectl 
- Install Ansible
- Install Helm
- Install Docker 

Install [Multipass](https://multipass.run/), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/#install-with-homebrew-on-macos), [ansible](https://docs.ansible.com/ansible/latest/installation_guide/index.html)  in mac os using [brew](https://brew.sh/) and [Docker](https://docs.docker.com/) using this [link](https://docs.docker.com/desktop/install/mac-install/). 

```ShellSession

brew install --cask multipass
brew install kubectl
brew install ansible
brew install helm

```
 
## Step by step guide

### 1. Create multipass ubuntu instances

Clone the github repo 

```ShellSession

git clone https://github.com/jpb111/kubernetes-k0s-ansible-harbor.git

cd k0s-ansible-local-docker-registery

```

Create minimum 2 ubuntu instance with multipass by running the script  multipass_create_instances.sh. This script will also create a cloud-init.yml named multipass-cloud-init.yml file with the public ssh key of the host system. Create a public key in host system if no pubic key exist with the below command. 

```ShellSession

ssh-keygen -t rsa

```
To support harbor which we will install for the local docker registery. Meet the minimum system [requirements](https://goharbor.io/docs/1.10/install-config/installation-prereqs/).  


```ShellSession

./tools/multipass_create_instances.sh 2

```

```ShellSession
Create cloud-init to import ssh key...
[2/2] Creating instance k0s-2 with multipass...
Launched: k0s-2                                                                 
Name                    State             IPv4             Image                
k0s-1                   Running           192.168.64.6     Ubuntu 22.04 LTS
k0s-2                   Running           192.168.64.7     Ubuntu 22.04 LTS
```


### 2. Create ansible inventory

python script to create an ansible inventory 

```ShellSession

./tools/multipass_generate_inventory.py

```

This will create a file inventory.yml in tools folder.
Copy this file into the inventory folder

The file will look like this tools/inventory.yml

```yml
---
---
all:
  children:
    controller:
      hosts: {}
    initial_controller:
      hosts:
        k0s-1:
    worker:
      hosts:
        k0s-2:
      
  hosts:
    k0s-1:
      ansible_host: 192.168.64.6
    k0s-2:
      ansible_host: 192.168.64.7

  vars:
    ansible_user: k0s

  
```  
### Test the connection to the virtual machines

```ShellSession

ansible all -m ping
```

```ShellSession


k0s-1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
k0s-2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

### 3. Create the kubernetes cluster

```ShellSession

ansible-playbook site.yml -i inventory/inventory.yml

```

```ShellSession

PLAY RECAP ***********************************************************************************************************************
k0s-1                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
k0s-2                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

### Use the cluster with kubectl

When the playbook ran, a kubeconfig got copied to the local machine at inventory/artifacts
folder.  


To use Cluster navigate to inventory/artifacts and copy the path of the file k0s-Kubeconfig.yml and assign it to KUBECONFIG. 

```ShellSession

export KUBECONFIG=/Users/johnpaulbabu/Documents/DevOps/kubernetes-k0s-ansible-harbor/inventory/artifacts/k0s-kubeconfig.yml

```


Check the kubernetes worker node 

```ShellSession

kubectl get nodes -o wide

```

This command will only show the worker nodes. 


```ShellSession

NAME    STATUS     ROLES    AGE     VERSION       INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k0s-2   Ready      <none>   3m22s   v1.22.4+k0s   192.168.64.7   <none>        Ubuntu 22.04.1 LTS   5.15.0-52-generic   containerd://1.5.8

 
```



### check the pods

```ShellSession

kubectl get pods -A

```

```ShellSession

NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-77b4ff5f78-zbw4c          1/1     Running   0          2m12s
kube-system   konnectivity-agent-8zj7c          1/1     Running   0          58s
kube-system   kube-proxy-5s5pm                  1/1     Running   0          119s
kube-system   kube-router-7s8pv                 1/1     Running   0          119s
kube-system   metrics-server-5b898fd875-rplbk   1/1     Running   0          2m12s

```

### 4. Install [MealLB](https://metallb.universe.tf/) load balancer to access the pods from the host machine, install by [manifest](https://metallb.universe.tf/installation/) 

```ShellSession

kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.3.2/deploy/longhorn.yaml

```

check metallb

```ShellSession

kubectl get ns

kubectl get pods -n metallb-system

```


```ShellSession

NAME              STATUS   AGE
default           Active   31m
kube-node-lease   Active   31m
kube-public       Active   31m
kube-system       Active   31m
metallb-system    Active   24s

NAME                          READY   STATUS              RESTARTS   AGE
controller-6b78bff7d9-2b4gt   0/1     ContainerCreating   0          29s
speaker-2zwq5                 0/1     ContainerCreating   0          29s
speaker-r2frj                 0/1     ContainerCreating   0          29s

```


###  Create ConfigMap for MetalLB

Creating [ConfigMap](https://metallb.universe.tf/configuration/), metallb-configmap.yml which includes an IP address range for the load balancer. These Ip address range for metallb should be based on the ubuntu instances IPs example 192.168.64.7. 

```yaml

cat << EOF > metallb-configmap.yml 

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.64.15-198.168.64.40

---

apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
  
EOF
```

Deploy the yaml file 

```ShellSession

 kubectl apply -f metallb-configmap.yml

```

### 5. [Install](https://longhorn.io/docs/1.3.2/deploy/install/install-with-kubectl/) [longhorn](https://staging--longhornio.netlify.app/) block storage. 

Longhorn has some [Prerequisites] (https://longhorn.io/docs/1.3.2/deploy/install/#installation-requirements).  Install open-iscsi and NFSv4 client. These are already installed in step 3 as ansible a role prereq_longhorn.


```ShellSession

kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.3.2/deploy/longhorn.yaml

```
One way to monitor the progress of the installation is to watch pods being created in the longhorn-system namespace:



```ShellSession

kubectl get pods \
--namespace longhorn-system \
--watch

```


```ShellSession

instance-manager-e-1f4571b1                    0/1     ContainerCreating   0          17m
instance-manager-e-2f9f0089                    0/1     ContainerCreating   0          27m
instance-manager-r-02a01d9f                    0/1     ContainerCreating   0          17m
instance-manager-r-a26b1f18                    0/1     ContainerCreating   0          27m
longhorn-admission-webhook-58457dfc88-6cw95    1/1     Running             0          41m
longhorn-admission-webhook-58457dfc88-jxfw7    1/1     Running             0          41m
longhorn-conversion-webhook-5d6d76dffb-4rqdh   1/1     Running             0          41m
longhorn-conversion-webhook-5d6d76dffb-6698d   1/1     Running             0          41m
longhorn-csi-plugin-lpz2x                      0/2     ContainerCreating   0          17m
longhorn-csi-plugin-z9s5n                      0/2     ContainerCreating   0          17m
longhorn-driver-deployer-65c878b8bc-zxlpt      1/1     Running             0          41m
longhorn-manager-4fvxn                         1/1     Running             0          41m
longhorn-manager-9xxsx                         1/1     Running             0          41m
longhorn-ui-84897c5d5b-7qb2h                   1/1     Running             0          41m
engine-image-ei-a5371358-zn8vn                 0/1     Running             0          29m
engine-image-ei-a5371358-zn8vn                 1/1     Running             0          29m
instance-manager-r-a26b1f18                    1/1     Running             0          29m
instance-manager-e-2f9f0089                    1/1     Running             0          29m
csi-attacher-846b795bf5-njncg                  1/1     Running             0          20m
csi-attacher-846b795bf5-75zbz                  1/1     Running             0          20m



```



```ShellSession

kubectl get all -n longhorn-system

NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/csi-attacher                  ClusterIP   10.101.92.11     <none>        12345/TCP   13h
service/csi-provisioner               ClusterIP   10.103.237.191   <none>        12345/TCP   13h
service/csi-resizer                   ClusterIP   10.110.240.14    <none>        12345/TCP   13h
service/csi-snapshotter               ClusterIP   10.103.4.194     <none>        12345/TCP   13h
service/longhorn-admission-webhook    ClusterIP   10.100.246.157   <none>        9443/TCP    14h
service/longhorn-backend              ClusterIP   10.100.45.110    <none>        9500/TCP    14h
service/longhorn-conversion-webhook   ClusterIP   10.107.186.55    <none>        9443/TCP    14h
service/longhorn-engine-manager       ClusterIP   None             <none>        <none>      14h
service/longhorn-frontend             ClusterIP   10.98.39.12      <none>        80/TCP      14h
service/longhorn-replica-manager      ClusterIP   None             <none>        <none>      14h

```

Incase if we need to access the longhorn front end we can assign a loadbalancer to the service/longhorn-frontend  to access the front end longhorn-ui. Edit the longhorn front end file and change type from ClusterIP to LoadBalancer 



```ShellSession

kubectl edit service/longhorn-frontend -n longhorn-system 

```
```ShellSession

nodePort: 31587
    port: 80
    protocol: TCP
    targetPort: http
  selector:
    app: longhorn-ui
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
```

```ShellSession

kubectl get all -n longhorn-system

```

Metallb will assign an IP address to Longhorn and we can access the longhorn front end UI. I have skipped this step. 

```ShellSession

kubectl get svc -n longhorn-system

```

The external IP address assigned by the loadbalancer to view the UI can be see with the name longhorn-frontend. 


### 9. Install [harbor](https://goharbor.io/) private docker registery. 


Before installing harbor make sure all the pods especially the one with longhorn-system namespace are up and running. 


```ShellSession

kubectl get pods -A

```


Install harbor through bitnami 


```ShellSession

helm repo add bitnami https://charts.bitnami.com/bitnami

helm install harbor bitnami/harbor -n harbor --create-namespace

```

```ShellSession

kubectl get pods -n harbor 

```

```ShellSession
NAME                                    READY   STATUS    RESTARTS        AGE
AME                                        READY   STATUS    RESTARTS      AGE
pod/harbor-chartmuseum-77c468f58b-bqt45     1/1     Running   0             29m
pod/harbor-core-74bf4f59cd-q6n9m            1/1     Running   3 (25m ago)   29m
pod/harbor-jobservice-789dd48bff-f8lr5      1/1     Running   2 (24m ago)   29m
pod/harbor-nginx-5d57ff5bf4-8x74t           1/1     Running   0             29m
pod/harbor-notary-server-86c9645dd5-lsgt2   1/1     Running   2 (24m ago)   29m
pod/harbor-notary-signer-76dd9b8dd6-fz6jr   1/1     Running   1 (26m ago)   29m
pod/harbor-portal-b58948757-srsjl           1/1     Running   0             29m
pod/harbor-postgresql-0                     1/1     Running   0             29m
pod/harbor-redis-master-0                   1/1     Running   0             29m
pod/harbor-registry-5c96d45d5f-9c5jv        2/2     Running   0             29m
pod/harbor-trivy-0                          1/1     Running   0             29m
```

```ShellSession

kubectl get svc -n harbor

```
```ShellSession

** Please be patient while the chart is being deployed **

1. Get the Harbor URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace harbor -w harbor'
    export SERVICE_IP=$(kubectl get svc --namespace harbor harbor --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
    echo "Harbor URL: http://$SERVICE_IP/"

2. Login with the following credentials to see your Harbor application

  echo Username: "admin"
  echo Password: $(kubectl get secret --namespace harbor harbor-core-envvars -o jsonpath="{.data.HARBOR_ADMIN_PASSWORD}" | base64 -d)

```
```ShellSession
export SERVICE_IP=$(kubectl get svc --namespace harbor harbor --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
echo "Harbor URL: http://$SERVICE_IP/"

```



The harbor url metallb assigned is http://192.168.64.16/ , click on advanced and proceed safe and will have the habor login, we can login with the username as admin and the generated 
password 

![harbor proceed safe](images/harbor-connection.png)

```ShellSession

 echo Password: $(kubectl get secret --namespace harbor harbor-core-envvars -o jsonpath="{.data.HARBOR_ADMIN_PASSWORD}" | base64 -d)

```
to access the local docker registery. 


![harbor login](images/harbor-login.jpg)
![harbor dashboard](images/harbor-dashboard.jpg)


Npw open the file 

```ShellSession

sudo vi /etc/hosts

```
and add 

```ShellSession

192.168.64.16 core.harbor.domain

```

to the file to redirect  192.168.64.16 to core.harbor.domain

```ShellSession

192.168.64.16 core.harbor.domain
::1             localhost

```

Now we can access harbor at https://core.harbor.domain/

This is the default harbor domain name, whcih can be changed by configuring the values.yml(https://goharbor.io/docs/1.10/install-config/configure-yml-file/) file of harbor. 


Login and change the password by clicking the admin icon. 
![passwordchange](images/harbor-password-change.png)


Since a secure ssl certificate is not used, in order for docker to allow to push an image to the local docker harbor registry, core.harbor.domain needed to be added as an insecure-registries in daemon.json file. Its is not a good practise. But for testing this should work.
 
To do that follow the link [Test an insecure registry](https://docs.docker.com/registry/insecure/)
to add 

```ShellSession
{
  "insecure-registries" : ["core.harbor.domain"]
}
```
to daemon.json file, save and restart docker. 

![docker daemon.json](images/docker-daemon.png)

Login and click to create a new project and check in public. 

![harbor new project](images/harbor-new-project.png)
![Harbor project](images/harbor-project.png)

On the new project page click push command to see how to push and image to the local docker registery. 

![Docker push command](images/docker-push-command.png)


### 10. Push an image to the harbor local docker registery  

Clone the gihub repo https://github.com/jpb111/python-docker-hello-kube.git to download a python flask hello-kube web application. 

```ShellSession

git clone  https://github.com/jpb111/python-docker-hello-kube.git
cd python-docker-hello-kube

```
Build a docker image with REPOSITORY name as app. 

```ShellSession

docker build . -t core.harbor.domain/python/hello

```
Check if the image is build 

```ShellSession

docker images

REPOSITORY                        TAG       IMAGE ID       CREATED        SIZE
core.harbor.domain/python/hello   latest    15258c49a6de   10 seconds ago   70.6MB

```

Login to docker 

```ShellSession

docker login core.harbor.domain


```


Enter username as admin and the password 

```ShellSession
Username: admin
Password: <Enter the password of harbor login>

```


Push the image into the local docker registery 

```ShellSession

docker push core.harbor.domain/python/hello

```

```ShellSession

The push refers to repository [core.harbor.domain/python/hello]
75147629c5ea: Pushed 
547d4fd8c06e: Pushed 
20233e24749e: Pushed 
ea7888c753d5: Pushed 
98a9e33fdbd9: Pushed 
2a46a238813c: Pushed 
e5fa9461c9a9: Pushed 
6666686122fd: Pushed 
994393dc58e7: Pushed 
latest: digest: sha256:c4f012a5aa2acacbb80d5480f42cffc2a5de681846c3e498acf29153bf54a4c7 size: 2202

```

In the https://core.harbor.domain/ the image can be seen in hello-kube project in the app repository. 

![docker image](images/docker-image.png)


### 11. Deploy the app 

The app can be deployed using the kubernetes deployment script. But before the image can be pulled from the harbor  registry by worker nodes each worker node needed to have the certificate. The certificate can be downloaded from the harbor docker registry. 
![certificate download](images/registry-certificate.png). 

Copy the downloaded registry certificate ca.crt to the k0s-ansible-local-docker-registery folder and run the playbook 

```yaml

ansible-playbook certificate.yml -i inventory/inventory.yml

```

```yaml

- hosts: worker
  become: yes
  tasks:
    - name: copying harbor docker registry certificate file ca.crt to worker nodes
      copy:
        src: ca.crt
        dest: /etc/ssl/certs/
      tags:
         - certificates

    - name: update ca certificates
      command: update-ca-certificates
      tags:
         - updateCertificate

    - name: Add 192.168.64.15 core.harbor.domain to worker nodes /etc/hosts
      tags: hostsFile
      lineinfile:
        path: /etc/hosts
        line: 192.168.64.15 core.harbor.domain
        state: present
        backup: yes
      register: hostsFile
      
```

to copy the ca.crt file to all the multipass worker nodes and also < 192.168.64.16 core.harbor.domain > to worker node's. 


 ```ShellSession
    
    /etc/hosts

```
file. 


After running this playbook, if needed try to ssh into one of the worker nodes and check if the files and text are < 192.168.64.16 core.harbor.domain > added if not added it will cause an image pull error when we try to deploy the app. 



### Create a secret docker-registry 

```ShellSession

kubectl get pods -A

```

create secret docker-registry which contains the login information for the harbor. Email is not used but we need to add it. 
Also replace the Password with the password of harbor registery created. 


```ShellSession

kubectl create secret docker-registry docker-registry \
 --docker-server="core.harbor.domain" \
 --docker-email=example@gmail.com \
 --docker-username=admin \
 --docker-password=Password 

``` 

Now lets create a  deployment.yml file we need to add the docker-registry-creds as imagePullSecrets also image path of the harbor registery. 

```yaml

cat << EOF > deployment.yml

apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
  - protocol: "TCP"
    port: 5000
    targetPort: 5000
  type: LoadBalancer

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  selector:
    matchLabels:
      app: hello
  replicas: 2
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image:  core.harbor.domain/python/hello
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
      imagePullSecrets:
        - name: docker-registry-creds
EOF

```

Now deploy the deployment.yml file 

```ShellSession

kubectl apply -f deployment.yml

```


```ShellSession

service/hello-service created
deployment.apps/hello-deployment created

```

```ShellSession

kubectl get pods -A

```

```ShellSession

NAMESPACE         NAME                                           READY   STATUS    RESTARTS       AGE
default           hello-deployment-57489fb685-jmrqf              1/1     Running   0              46s
default           hello-deployment-57489fb685-vzfpw              1/1     Running   0              46s
harbor            harbor-chartmuseum-65c4fb56df-klwq9            1/1     Running   0              74m
harbor            harbor-core-59cb8dc5c8-fbtvc                   1/1     Running   17 (73m ago)   18h
harbor            harbor-jobservice-77cdf889fb-n8nrm             1/1     Running   1 (71m ago)    74m
harbor            harbor-nginx-58867f76cc-lwplj                  1/1     Running   8 (76m ago)    18h
harbor            harbor-notary-server-b69699bb5-72kfj           1/1     Running   11 (71m ago)   18h
harbor            harbor-notary-signer-599877cc4d-jmxzl          1/1     Running   10 (71m ago)   18h
harbor            harbor-portal-b58948757-72vjz                  1/1     Running   3 (78m ago)    18h
harbor            harbor-postgresql-0                            1/1     Running   0              73m
harbor            harbor-redis-master-0                          1/1     Running   0              73m
harbor            harbor-registry-5b5b46d99-r77zl                2/2     Running   0              74m
harbor            harbor-trivy-0                                 1/1     Running   0              73m
kube-system       coredns-77b4ff5f78-th279                       1/1     Running   2 (78m ago)    20h
kube-system       konnectivity-agent-hjkvn                       1/1     Running   4 (78m ago)    20h
kube-system       kube-proxy-9gnmp                               1/1     Running   2 (78m ago)    20h
kube-system       kube-router-ktprq                              1/1     Running   8 (78m ago)    20h
kube-system       metrics-server-5b898fd875-b85nf                1/1     Running   6 (76m ago)    20h
longhorn-system   csi-attacher-846b795bf5-jk7d8                  1/1     Running   6 (78m ago)    20h
longhorn-system   csi-attacher-846b795bf5-nphwn                  1/1     Running   4 (78m ago)    20h
longhorn-system   csi-attacher-846b795bf5-tp5bv                  1/1     Running   6 (78m ago)    20h
longhorn-system   csi-provisioner-74f88ccdfd-5tvxr               1/1     Running   6 (76m ago)    20h
longhorn-system   csi-provisioner-74f88ccdfd-h44sq               1/1     Running   5 (76m ago)    20h
longhorn-system   csi-provisioner-74f88ccdfd-jrbhp               1/1     Running   5 (69m ago)    20h
longhorn-system   csi-resizer-75cf7df868-9kf6q                   1/1     Running   5 (78m ago)    20h
longhorn-system   csi-resizer-75cf7df868-lslrk                   1/1     Running   3 (78m ago)    20h
longhorn-system   csi-resizer-75cf7df868-s77qr                   1/1     Running   3 (78m ago)    20h
longhorn-system   csi-snapshotter-65d98cc868-2xmvr               1/1     Running   3 (78m ago)    20h
longhorn-system   csi-snapshotter-65d98cc868-7ztzj               1/1     Running   2 (78m ago)    20h
longhorn-system   csi-snapshotter-65d98cc868-fh64t               1/1     Running   6 (78m ago)    20h
longhorn-system   engine-image-ei-a5371358-qp6gj                 1/1     Running   2 (78m ago)    20h
longhorn-system   instance-manager-e-61ded4fe                    1/1     Running   0              75m
longhorn-system   instance-manager-r-a357e4d5                    1/1     Running   0              75m
longhorn-system   longhorn-admission-webhook-58457dfc88-sf6gw    1/1     Running   2 (78m ago)    20h
longhorn-system   longhorn-admission-webhook-58457dfc88-wztnc    1/1     Running   2 (78m ago)    20h
longhorn-system   longhorn-conversion-webhook-5d6d76dffb-g55hq   1/1     Running   2 (78m ago)    20h
longhorn-system   longhorn-conversion-webhook-5d6d76dffb-rvvhn   1/1     Running   2 (78m ago)    20h
longhorn-system   longhorn-csi-plugin-sjpmz                      2/2     Running   9 (75m ago)    20h
longhorn-system   longhorn-driver-deployer-65c878b8bc-q2pz9      1/1     Running   2 (78m ago)    20h
longhorn-system   longhorn-manager-zpkcx                         1/1     Running   2 (78m ago)    20h
longhorn-system   longhorn-ui-84897c5d5b-sgwmq                   1/1     Running   6              20h
metallb-system    controller-6b78bff7d9-ltrn2                    1/1     Running   4 (76m ago)    20h
metallb-system    speaker-n7jqk                                  1/1     Running   2 (78m ago)    20h

```

The pod hello-deployment is running with default namespace. 



```ShellSession

 kubectl get svc

```

```ShellSession

NAME            TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
hello-service   LoadBalancer   10.101.209.174   192.168.64.17   5000:32245/TCP   13h
kubernetes      ClusterIP      10.96.0.1        <none>          443/TCP          20h

```


The metallb has assigned an IP, 192.168.64.17 to the  python application and if we go to 

the IP http://192.168.64.17:5000/. We can view the application 

![hello](images/hello-app1.png). 

To delete deployment 

```ShellSession

kubectl delete deployment hello-deployment

```


### 12. Creating a helm chart for the  hello-kube application

In this section we will create a basic helm chart for the hello-kube python application and deploy it in the kubernetes cluster. 



```ShellSession

helm create hello-kube

```

This commnad will give the structure for the helm chart and create folder hello-kube.
Since we are using a basic helm chart, We clean up the files and folderse in the hello-kube and 
make it in the below structure. 

![hello-kube dir](/images/hello-kube-helm.png)

Now we will edit the deployment.yaml file in the templates folder inside hello-kube

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kube
spec:
  selector:
    matchLabels:
      app:  hello-kube
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      labels:
        app:  hello-kube
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 5000
              protocol: TCP

```


Later we will edit the service.yaml file 

```yaml

apiVersion: v1
kind: Service
metadata:
  name: hello-kube
spec:
  selector:
    app: hello-kube
  ports:
  - protocol: "TCP"
    port: {{ .Values.service.port }}
    targetPort: http
  type: {{ .Values.service.type }}

```

Now we will edit the values.yaml file and add our configurations for deployment which will be passed as variable to deployment.yaml and service.yaml files. 

```yaml

# Default values for hello-kube.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: core.harbor.domain/python/hello
  pullPolicy: Always
  # Overrides the image tag whose default is the chart appVersion.
  tag: "latest"

imagePullSecrets: 
  - name: docker-registry-creds
service:
  type: LoadBalancer
  port: 5000


```

now install the helm chart 

```ShellSession

helm install hello-kube hello-kube

```


```ShellSession

NAME: hello-kube
LAST DEPLOYED: Fri Nov 18 16:40:26 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

```

```ShellSession

kubectl get pods -A 


NAMESPACE         NAME                                           READY   STATUS    RESTARTS       AGE
default           hello-kube-77bd555dc9-m4hq2                    1/1     Running   0              9s
harbor            harbor-chartmuseum-5c5dcd74c-vp7gh             1/1     Running   0              14m
harbor            harbor-core-547b7999db-mrkcv                   1/1     Running   35 (12m ago)   2d1h

```
We can see the application running as hello-kube in default name space. 

```ShellSession

kubectl get svc

NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
hello-kube   LoadBalancer   10.99.220.11   192.168.64.17   5000:32675/TCP   14s

```
```ShellSession

curl 192.168.64.17:5000
Hello, Kube!%       

```

To delete the deployment 
```ShellSession

helm delete hello-kube

```

Package the application as a helm chart and upload it to a harbor helm chart registry. 

```ShellSession

helm package hello-kube

```

This will create file hello-kube.0.1.0.tgz in the current folder. 

Upload this file  the harbor local registery in the project folder under helm-char

![file-upload](/images/helm-chart-upload.png)


Adding the helm repo 

![helm repo](/images/helm-repo.png)



```ShellSession


helm repo add --ca-file ca.crt  stable https://core.harbor.domain/chartrepo/python

helm search repo stable

```

```ShellSession

NAME                    CHART VERSION   APP VERSION     DESCRIPTION                       
stable/hello-kube       0.1.0           1.16.0          A Helm chart for python-hello-kube

```


```ShellSession

helm install hello stable/hello-kube  

```

```ShellSession

NAME: hello
LAST DEPLOYED: Thu Nov 24 11:47:19 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

```

```ShellSession

 kubectl get pods -A

```

Hello is deployed in default name space. 


```ShellSession

NAMESPACE         NAME                                           READY   STATUS    RESTARTS         AGE
default           hello-kube-58cfd8cb66-fn54s                    1/1     Running   0                13s
harbor            harbor-chartmuseum-5c5dcd74c-8w8zd             1/1     Running   0                3h10m
harbor            harbor-core-547b7999db-mrkcv                   1/1     Running   57 (31m ago)     7d21h
harbor            harbor-jobservice-766f5d4ff9-zvdgb             1/1     Running   5 (29m ago)      22h

```

Let get the external IP address assigned by metallb. 

```ShellSession

kubectl get svc   

NAME         TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
hello-kube   LoadBalancer   10.103.253.160   192.168.64.17   5000:32119/TCP   79s

```

Check if the app is running in the given IP. 

```ShellSession

curl 192.168.64.17:5000

```

```ShellSession

Hello, Kube!%   

```



Hello, Kube is running at  192.168.64.17:5000

![hello-kube-app](/images/hello-kube-app.png) 



To unistall the deployment and delete the repo


```ShellSession

helm delete hello 

helm repo remove stable

```




----Finished------

--------------------------------------------------------------------------------------------------------






## Troubleshooting 

Kubernetes [cheetseet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)


Error 

```ShellSession

The connection to the server localhost:8080 was refused - did you specify the right host or port?

```

Run 


```ShellSession

export KUBECONFIG=/Users/johnpaulbabu/Documents/DevOps/kubernetes-k0s-ansible-harbor/inventory/artifacts/k0s-kubeconfig.yml


```
Error while trying to pull the image from harbor 

```ShellSession
  Warning  Failed     21s (x2 over 41s)  kubelet            Error: ErrImagePull
  Normal   BackOff    10s (x2 over 41s)  kubelet            Back-off pulling image "core.harbor.domain/python/hello"
  Warning  Failed     10s (x2 over 41s)  kubelet            Error: ImagePullBackOff
Get more details of the pods 

```

Check docker login and also ssh into worker nodes and navigate to  /etc/hosts to check file if (harbor) Ip_address core.harbor.domain is added, refer step 10. 


To get the details of a pod 

```ShellSession

kubectl describe pod <pod name>

```

To ssh into ubuntu instances  


```ShellSession

ssh k0s@<ip address>

```

To reset 



ansible-playbook reset.yml -i inventory/inventory.yml

```

To delete a deployment 

```ShellSession

kubectl delete deployments  <deployment name>

```
To delete everything 

```ShellSession
multipass delete --all
multipass purge

```

