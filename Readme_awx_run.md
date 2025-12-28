# AWX on minikube



## Create user
```bash
useradd subrata
su - subrata
```
- add this user to sudoers list



## Install docker
[Install Docker](https://docs.docker.com/engine/install/rhel/)

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run hello-world
```



## Install minikube
[Download Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download)

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```



## Start minikube
```bash
minikube start

# or

minikube start --driver=docker -p minikube-docker

minikube profile list

minikube delete -p minikube-docker

```



## Install kubectl

[Kubectl Download](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```



## Kubectl bash completion:

```bash
# Save the completion script to a file:
kubectl completion bash > ~/.kube/completion.bash.inc

# Then, add the following line to your ~/.bashrc:
vi ~/.bashrc
source ~/.kube/completion.bash.inc

# Reload the configuration:
source ~/.bashrc

# If you're using an alias for kubectl (e.g., alias k=kubectl), you need to set up completion for the alias as well.
# Add the following lines to your ~/.bashrc:
alias k=kubectl
complete -F __start_kubectl k

# Reload the configuration:
source ~/.bashrc
```



## Install AWX
Follow the docs [Install AWX-Operator](https://docs.ansible.com/projects/awx-operator/en/latest/installation/basic-install.html)

[Ansible AWX Operator Latest Version](https://quay.io/repository/ansible/awx-operator?tab=tags)

```bash
git clone git@github.com:ansible/awx-operator.git
or
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git checkout tags/2.7.2
export VERSION=2.7.2
make deploy
kubectl config set-context --current --namespace=awx
```



### Next, create a file named awx.yml in the same folder with the suggested content below. The metadata.name you provide will be the name of the resulting AWX deployment.

#### Create persistent storage to share the projects directory between both task and web pods.

```yaml
# vi awx.yml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  projects_persistence: true
  projects_storage_class: standard
  projects_storage_size: 10Gi
  service_type: nodeport
  nodeport_port: 32000
:wq

# kubectl apply -f awx.yml

# kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
```

### or

#### Create a pvc and attach the pvc with the awx deployment

```yaml
# vi pvc.yml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: awx-projects-pvc
  namespace: awx
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

# kubectl apply -f pvc.yml


# vi awx.yml

apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  projects_existing_claim: awx-projects-pvc
  service_type: nodeport
  nodeport_port: 32000
:wq

# kubectl apply -f awx.yml
```




### Check the sync status (Both the pod should share same data)
```bash
kubectl get pods

kubectl exec -it awx-web-xxxxxx -n awx -- bash
kubectl exec -it awx-task-xxxxxx -n awx -- bash

kubectl exec -it awx-web-xxxxxx -n awx -- ls -l /var/lib/awx/projects
kubectl exec -it awx-task-xxxxxx -n awx -- ls -l /var/lib/awx/projects
```
### Actual location of data on host  (upload the playbooks here)
```bash
minikube ssh
ls -l /var/hostpath-provisioner/awx/awx-projects-pvc/
# or
ls -l /var/hostpath-provisioner/awx/awx-projects-claim/
```




## Expose AWX port

```bash
# You should see the NodePort service like below
k get all
service/awx-service NodePort    10.100.151.160   <none>        80:31627/TCP   9h

# The webpage should be available on
minikube ip
192.168.49.2

minikube service -n awx awx-service --url

curl 192.168.49.2:31627
```




## If you wish to access it from external network then use below options
### Option 1
```bash
# Port forward
kubectl port-forward svc/awx-service 8080:80 --address 0.0.0.0

curl localhost:8080
```




### Option 2
```bash
# Minikube inbuilt method (Will get random IP from 10.96.0.0/12 pool)
# open a new console and run below command, this will run continiously
minikube tunnel

# open another terminal
k expose deployment awx-web --type=LoadBalancer --target-port 8052 --port 8080
# or
k expose deployment awx-web --type=LoadBalancer --port 8052

k get all

curl localhost:8080
# or
curl localhost:8052
```




## Option 3
```bash
### Add metallb for externel IP (Will get IP from specific range)
minikube addons enable metallb

minikube addon list

kubectl get crd | grep metallb
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
kubectl get crd | grep metallb

kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: minikube-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.49.100-192.168.49.110
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2adv
  namespace: metallb-system
EOF

kubectl get pods -n metallb-system

k expose deployment awx-web --type=LoadBalancer --target-port 8052 --port 8080
k expose deployment awx-web --type=LoadBalancer --port 8052

k get all
service/awx-web LoadBalancer   10.104.218.170   192.168.49.100   8081:32523/TCP   177m

curl 192.168.49.100:8081
```



### Fetch credetials
```bash
admin
foM4qwGDU1qj45RjaRAwgh1XDcDNdjLa

kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode ; echo
```




### Forget password ? (Change the admin password)
```bash
kubectl get pods
kubectl exec -it awx-demo-web-5c475d9cdb-fmfrx -- /bin/bash
awx-manage changepassword admin
```