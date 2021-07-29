# Kubernetes-Minikube-Documentation for Milestone #2 (Ocean DAO Round 7)
Provide examples and code documentation in order to help other developer teams build access policies in building and deploying their own kubernetes minikube. We have completed Milestone #2 and the instructions are as follows. :sparkles: Enjoy coding and developing!

Here is a video that shows you the steps:

[![mPowered](https://mpowered.io/assets/images/logo.png)](https://player.vimeo.com/video/580934725)

# Set Up a Compute-to-Data Environment with remote Minikube

Reference: https://docs.oceanprotocol.com/tutorials/compute-to-data/

## Requirements

- functioning internet-accessable provider (ours was https://provider.mpowered.io)
- machine capable of running compute (we used a machine with 8 CPUs, 16 GB Ram, and 100GB SSD and fast internet connection)
- Ubuntu 20.04
- SSH bridge (explained later)

## Install Docker and Git

```bash
sudo apt update
sudo apt install git docker.io
sudo usermod -aG docker $USER && newgrp docker
```

## Install Minikube

```bash
wget -q --show-progress https://github.com/kubernetes/minikube/releases/download/v1.22.0/minikube_1.22.0-0_amd64.deb
sudo dpkg -i minikube_1.22.0-0_amd64.deb
```

## Download and Configure Operator Service

```bash
git clone https://github.com/oceanprotocol/operator-service.git
```

Change POSTGRES_PASSWORD to nice random password.

```bash
vi operator-service/kubernetes/postgres-configmap.yaml
```

Change the ALGO_POD_TIMEOUT to an hour, add bridge code, and use latest docker images
```bash
vi operator-service/kubernetes/deployment.yaml
```

The code below is added to deployment.yaml between `terminationMessagePolicy: File` and `dnsPolicy: ClusterFirst`.

It creates an SSH tunnel or bridge between the remote server which will run the C2D algorithm and the provider. Doing it this way solves any networking issues that may arise.

Alternatively, you could expose the operator service to the internet by runing a port forward or create your ingress service.

```yaml
      - name: provider-bridge
        image: alpine:3.11
        env:
        - name: SSH_USERNAME
          valueFrom:
            configMapKeyRef:
              key: SSH_USERNAME
              name: bridge-config
        volumeMounts:
        - name: ssh-private-key
          mountPath: /tmp/id_rsa
          subPath: id_rsa
        - name: ssh-known-hosts
          mountPath: /root/.ssh/known_hosts
          subPath: known_hosts
        command: ["/bin/sh", "-c", "--"]
        args:
          - apk add --no-cache openssh-client ca-certificates bash;
            cp /tmp/id_rsa /root/.ssh/;
            chmod 700 /root/.ssh;
            chmod 600 /root/.ssh/id_rsa;
            ssh -2nNT -o ServerAliveInterval=60 -R 8050:localhost:8050 $SSH_USERNAME@<hostname>
      volumes:
      - name: ssh-private-key
        configMap:
          name: bridge-config
      - name: ssh-known-hosts
        configMap:
          name: bridge-config
```

where `<hostname>` is the IP address or hostname of the provider server.

You will also need to create a new file called `bridge-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bridge-config
  namespace: ocean-operator
data:
  SSH_USERNAME: <username>
  id_rsa: |
    <id_rsa>
  known_hosts: |
    <known_hosts>
```

where `<username>` is a SSH username on the provider server, `<id_rsa>` and `<known_hosts>` are the SSH credential files.

## Download and Configure Operator Engine

```bash
git clone https://github.com/oceanprotocol/operator-engine.git
```

Add your IPFS URLs, remove any AWS references, add notification URLs, and use latest version of images.

```bash
vi operator-engine/kubernetes/operator.yml
```

You may optionally prevent outgoing traffic from the algorithm pod by creating the file `deny-algorithm-egress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-algorithm-egress
  namespace: ocean-compute
spec:
  podSelector:
    matchExpressions:
      - {key: component, operator: In, values: [algorithm]}
      - {key: allowNetworkAccess, operator: DoesNotExist}
    matchLabels:
      component: algorithm
  policyTypes:
  - Egress
  egress: []
```

## Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(<kubectl.sha256) kubectl" | sha256sum --check

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## Start Minikube

First command is imporant, and solves a [PersistentVolumeClaims problem](https://github.com/kubernetes/minikube/issues/7828). 

```bash
minikube config set kubernetes-version v1.16.0
minikube start --cni=calico --driver=docker --container-runtime=docker
```

Wait untill all the defaults are working (1/1).

```bash
watch kubectl get pods --all-namespaces
```

## Create namespaces

```bash
kubectl create ns ocean-operator
kubectl create ns ocean-compute
```

## Deploy Operator Service

```bash
kubectl config set-context --current --namespace ocean-operator
kubectl create -f operator-service/kubernetes/postgres-configmap.yaml
kubectl create -f operator-service/kubernetes/postgres-storage.yaml
kubectl create -f operator-service/kubernetes/postgres-deployment.yaml
kubectl create -f operator-service/kubernetes/postgresql-service.yaml
kubectl create -f bridge-configmap.yaml
kubectl apply  -f operator-service/kubernetes/deployment.yaml
```

Wait until the pods are created and running, and the SSH connection to the provider server has been made.

Then, on the provider server, initialize the database:

```bash
curl -v -X POST "http://127.0.0.1:8050/api/v1/operator/pgsqlinit" -H "accept: application/json"
```

## Deploy Operator Engine

```bash
kubectl config set-context --current --namespace ocean-compute
kubectl apply  -f operator-engine/kubernetes/sa.yml
kubectl apply  -f operator-engine/kubernetes/binding.yml
kubectl apply  -f operator-engine/kubernetes/operator.yml
kubectl apply  -f deny-algorithm-egress.yaml
kubectl create -f operator-service/kubernetes/postgres-configmap.yaml
```

Wait until the compute operator pod is running.

Now your remote C2D minikube can accept and process jobs. If you are an SME and would like to have bespoke implementations of C2D, please send us an email at hello[at]mpowered[dot]io
