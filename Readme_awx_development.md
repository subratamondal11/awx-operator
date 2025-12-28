## AWX Development

Link: [AWX Development](https://github.com/ansible/awx-operator/blob/devel/docs/development.md)

Redhat is providing [Ansible Operator](https://quay.io/repository/operator-framework/ansible-operator?tab=tags) to create AWX-operator.

[Ansible Operator Image](https://quay.io/repository/operator-framework/ansible-operator?tab=tags) is the base image of AWX-operator.

[Ansible Operator Image](https://quay.io/repository/operator-framework/ansible-operator?tab=tags) uses RedHat Universal Base Image ([UBI](https://catalog.redhat.com/en/search?searchType=All&q=ubi&p=1)) as the backend of its own base image.

[Ansible Operator Documentation](https://sdk.operatorframework.io/docs/building-operators/ansible/reference/ansible-base-images/)

[My own AWX operator](https://quay.io/repository/subratamondal11/awx-operator?tab=tags)

```text
Flow: RedHat Universal Base Image -> Ansible Operator (Dockerfile base OS) -> AWX Operator (Exporting as docker image)
```
## Create own awx-operator by using below method:

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







## Development Guide

```bash
git clone https://github.com/ansible/awx-operator.git
```

There are development scripts and yaml exaples in the [`dev/`](../dev) directory that, along with the up.sh and down.sh scripts in the root of the repo, can be used to build, deploy and test changes made to the awx-operator.


## Prerequisites

You will need to have the following tools installed:

* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [podman](https://podman.io/docs/installation) or [docker](https://docs.docker.com/get-docker/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [oc](https://docs.openshift.com/container-platform/4.11/cli_reference/openshift_cli/getting-started-cli.html) (if using Openshift)

You will also need to have a container registry account. This guide uses quay.io, but any container registry will work. You will need to create a robot account and login at the CLI with `podman login` or `docker login`.

## Quay.io Setup for Development

Before using the development scripts, you'll need to set up a Quay.io repository and pull secret:



### 1. Create a Private Quay.io Repository
- Go to [quay.io](https://quay.io) and create a private repository named `awx-operator` under your username
- The repository URL should be `quay.io/username/awx-operator`

### 2. Create a Bot Account
- In your Quay.io repository, go to Settings → Robot Accounts
- Create a new robot account with write permissions to your repository
- Click on the robot account name to view its credentials

### 3. Generate Kubernetes Pull Secret
- In the robot account details, click "Kubernetes Secret"
- Copy the generated YAML content from the pop-up

### 4. Create Local Pull Secret File
- Create a file at `hacking/pull-secret.yml` in your awx-operator checkout
- Paste the Kubernetes secret YAML content into this file
- **Important**: Change the `name` field in the secret from the default to `redhat-operators-pull-secret`
- The `hacking/` directory is in `.gitignore`, so this file won't be committed to git

Example `hacking/pull-secret.yml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redhat-operators-pull-secret  # Change this name
  namespace: awx
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-credentials>
```

## Build and Deploy

If you clone the repo, and make sure you are logged in at the CLI with kubectl or oc and your cluster, you can run:

```bash
# Login to quay.io to upload your custome image there before starting the application
docker login quay.io
# or
podman login quay.io
# Username: use robot account credentials of quay.io (make sure the robot account has write permission on the repository)

export QUAY_USER=subratamondal11
export NAMESPACE=awx
export TAG=test

# make below changes
vi up.sh

# ENGINE=${ENGINE:-podman}
ENGINE=${ENGINE:-docker}

# -- Create CR
# uncomment the CR you want to use
$KUBE_APPLY awx-demo.yml
# $KUBE_APPLY dev/awx-cr/awx-openshift-cr.yml
:wq

./up.sh

# for cleanup
./down.sh
```

- Check if the test tag is used to run the AWX-Operator
```bash
kubectl -n awx get pods
kubectl -n awx describe pod/awx-operator-controller-manager-xxxxxx
```

```bash
kubectl config set-context --current --namespace=awx
```

```dockerfile
# If you want to update the base OS

# vi Dockerfile

FROM quay.io/operator-framework/ansible-operator:main
# FROM quay.io/operator-framework/ansible-operator:v1.36.1

USER root
# RUN dnf update --security --bugfix -y && \
#     dnf install -y openssl

# RUN microdnf update --security --bugfix -y
RUN microdnf update -y && \
    microdnf install -y openssl

# RUN microdnf install -y openssl vim

USER 1001

ARG DEFAULT_AWX_VERSION
ARG OPERATOR_VERSION
ENV DEFAULT_AWX_VERSION=${DEFAULT_AWX_VERSION}
ENV OPERATOR_VERSION=${OPERATOR_VERSION}

COPY requirements.yml ${HOME}/requirements.yml
RUN ansible-galaxy collection install -r ${HOME}/requirements.yml \
 && chmod -R ug+rwx ${HOME}/.ansible

COPY watches.yaml ${HOME}/watches.yaml
COPY roles/ ${HOME}/roles/
COPY playbooks/ ${HOME}/playbooks/

ENTRYPOINT ["/tini", "--", "/usr/local/bin/ansible-operator", "run", \
    "--watches-file=./watches.yaml", \
    "--reconcile-period=0s" \
    ]

:wq

./up.sh
```

### Get the details about the ansible-operator image
```bash
kubectl exec -it pod/awx-operator-controller-manager-xxxxxx -- bash

bash$ rpm -qa
bash$ env
bash$ set

```
### Now run the awx pod and start development

```yaml
# vi awx-demo.yml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
spec:
  service_type: nodeport
  nodeport_port: 32000
:wq

# kubectl apply -f awx-demo.yml

# kubectl get all
# kubectl get awx
# kubectl describe awx awx-demo

# kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
```
```bash
kubectl get secret awx-demo-admin-password -o jsonpath="{.data.password}" | base64 --decode ; echo
kubectl port-forward svc/awx-demo-service 8080:80 --address 0.0.0.0
```

### Prepare for development

A) AWX UI / Backend
- Clone the [AWX repo](https://github.com/ansible/awx) locally. 
- Modify the code.
- Build a dev container image:

B) Operator itself
- Clone the AWX Operator repo
- Make modifications.
- Deploy:

```bash
export QUAY_USER=subratamondal11
export NAMESPACE=awx
export TAG=test
./up
```
- Use CRs to test how changes affect AWX deployment.