## Kubernetes Cluster Management at Scale with Gardener

- [1. The Scenario](#1-the-scenario)
  - [1.1 The Daily Conversation](#11-the-daily-conversation)
- [2. The Problem](#2-the-problem)
- [3. What We Actually Need?](#3-what-we-actually-need)
- [4. Gardener - Kubernetes as a Service - Across Any Cloud](#4-gardener---kubernetes-as-a-service---across-any-cloud)
- [5. Gardener Architecture: Garden, Seed, and Shoot](#5-gardener-architecture-garden-seed-and-shoot)
- [6. Hands-on: Running Gardener on AWS EC2](#6-hands-on-running-gardener-on-aws-ec2)


### 1. The Scenario

![Platform Bottleneck](platform-bottleneck.png)

Let's assume we have a large software company, more than 5,000+ employees, multiple products, customers all over the world.

This company has 50+ development teams. Each team owns different projects, different microservices, and has different needs:

- Team A builds customer-facing APIs that handle millions of requests per day

- Team B works on internal data pipelines that process terabytes of information

- Team C maintains the company's e-commerce platform

- Team D is building a new AI/ML service

And so on, and so on...

Every single team needs one thing in common: **a place to run their containers.**

#### 1.1. The Daily Conversation


Now, here's a conversation that happens every single day, multiple times a day, in this company:


**Developer:** "Hey Platform Team, we're spinning up a new service and we need a Kubernetes cluster."

**Platform Team:** "Sure, what do you need?"

**Developer:** "Nothing fancy:

- 3 worker nodes

- 8GB RAM and 4 vCPU per node

- Needs to auto-scale

- We don't care if it's AWS, Azure, or GCP, just give us something that works

- Oh, and when can we have it?"

**Platform Team:** (sighs internally) "We'll get back to you."


This exact conversation plays out multiple times a day, every single day of the year. And it's not just about creating clusters, it's about keeping them healthy, updating them, scaling them, and making sure they don't break at 2 AM.

### 2. The Problem

![Problem](problem.png)

Every team just wants one thing: a place to run their containers. So they ask the platform team for a Kubernetes cluster. Simple, but here's where it gets hard.

1. **Provisioning is the easy part**: Terraform or eksctl can spin up a cluster in minutes. That's day one. The real work is day two: upgrades, patches, backups, cert rotation, and 2 AM fixes, all of it stays with the platform team forever.

2. **Consistency is hard**: Build 50 clusters over time and they slowly drift — different versions, different configs. Keeping a whole fleet identical and healthy is a constant fight.
Every cloud is different. AWS, Azure, and GCP each have their own tools and rules. The platform team has to know them all.
3. **It doesn't scale**: Five clusters is fine. Fifty across three clouds is not. The platform team becomes the bottleneck.

In the end, teams wait, the platform team drowns, and clusters quietly turn into a risk instead of a tool.

### 3. What We Actually Need?

If we want to fix the problem mentioed above, we need a way of managing clusters that does four things:

![Needs](needs.png)

### 4. Gardener - Kubernetes as a Service - Across Any Cloud    

![Gardener](gardener01.png)

- **Gardener** is an **open-source project** (originally built by SAP) that does exactly what we described: it manages Kubernetes clusters as a service, at scale, across any cloud.

- The core idea is simple: use Kubernetes to manage Kubernetes.

- You don't click buttons or run scripts to build a cluster. Instead, you write a **Shoot** manifest that describes the cluster you want: the provider, the region, the version, the worker pools. You apply it, and Gardener does the rest. 

- It provisions the machines, sets up networking, builds the control plane, and from then on keeps the cluster healthy: upgrades, patches, certificate rotation, self-healing, all automatic.

- If you want AWS instead of GCP, you change one field. The manifest stays the same; Gardener's provider extension handles the cloud-specific details behind the scenes. The platform team only speaks one language: **the Shoot API**.

So the four problems from before disappear:

1. Provisioning + Day-2: handled by Gardener, automatically, the same way every time.
2. Consistency: every cluster is built and maintained by the same controllers, so no drift.
3. Every cloud is different: one API, provider details hidden behind extensions.
4. It doesn't scale: Gardener is designed to run thousands of clusters, not five.

- The platform team stops being a bottleneck. They build the platform; teams help themselves.

But to understand how Gardener pulls this off, we need to look at its architecture and that starts with three words: **Garden**, **Seed**, and **Shoot**.

### 5. Gardener Architecture: Garden, Seed, and Shoot

![Gardener Architecture](gardener02.png)


- Gardener's design comes straight from its name. There are three kinds of clusters, and each plays a different role.

  - **Garden** - the control center.

      - This is the central cluster where Gardener itself runs. It's where you, the platform team, submit your Shoot manifests. The Garden cluster doesn't run anyone's workloads; it's the brain that knows about every cluster in the system and decides what needs to happen. Think of it as the place where you plant everything.

  - **Seed** — where the control planes live.

      - Here's Gardener's clever trick. Normally, every Kubernetes cluster needs its own control plane (API server, etcd, scheduler, etc.), usually running on dedicated master nodes. That's expensive and a pain to manage at scale. Instead, Gardener runs each cluster's control plane as pods inside a Seed cluster. So one Seed can host the control planes of many clusters at once — cheap, dense, and easy to operate. Seeds are the soil where clusters take root.

  - **Shoot** — the cluster teams actually use.

      - A Shoot is the end-user cluster — the one Team A runs their containers on. Its control plane lives as pods on a Seed, and its worker nodes run in the team's own cloud account (AWS, Azure, GCP...). To the team, it looks and behaves like a completely normal Kubernetes cluster. They never see the machinery underneath.

- So the flow is:
  - You apply a Shoot manifest to the Garden cluster
  - Gardener places that cluster's control plane on a Seed
  - the Shoot's worker nodes spin up in the target cloud
  - Gardener keeps it all healthy, forever.

This is why Gardener scales to thousands of clusters: control planes are just containers, managed like any other Kubernetes workload. No special master VMs, no snowflakes — Kubernetes managing Kubernetes, all the way down.

### 6. Hands-on: Running Gardener on AWS EC2

Gardener's local setup runs a single KinD cluster that acts as **both** Garden and Seed, then lets you create a **Shoot** on top of it. The setup is designed for **native Linux**.

> **Why a cloud Linux box instead of a laptop?**
> Running this on macOS (Docker Desktop) is slow and fragile: the nested virtualization layers (gVisor networking + virtiofs) make container sandbox creation and image extraction so slow. The same setup on a native Linux host with an SSD finishes in minutes. So we run it on a fresh AWS EC2 instance and tear it down when done.

#### 6.1. Create the EC2 instance (AWS CLI)

Verify your AWS CLI is configured, then set a region:

```
aws sts get-caller-identity
export AWS_REGION=eu-north-1
```

Set variables:

```
export KEY_NAME=gardener-key
export SG_NAME=gardener-sg
export INSTANCE_TYPE=m6i.2xlarge
```

Create an SSH key pair:

```
mkdir -p ~/.ssh && chmod 700 ~/.ssh
aws ec2 create-key-pair --region $AWS_REGION --key-name $KEY_NAME \
  --query 'KeyMaterial' --output text > ~/.ssh/$KEY_NAME.pem
chmod 400 ~/.ssh/$KEY_NAME.pem
```

Create a security group that allows SSH only from your IP:

```
export MY_IP=$(curl -s https://checkip.amazonaws.com)
export VPC_ID=$(aws ec2 describe-vpcs --region $AWS_REGION \
  --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)
export SG_ID=$(aws ec2 create-security-group --region $AWS_REGION \
  --group-name $SG_NAME --description "Gardener dev box SSH" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --region $AWS_REGION \
  --group-id $SG_ID --protocol tcp --port 22 --cidr ${MY_IP}/32
```

Find the latest Ubuntu 22.04 AMI:

```
export AMI_ID=$(aws ec2 describe-images --region $AWS_REGION --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
            "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)
```

Launch the instance with a 120 GiB gp3 SSD root volume:

```
export INSTANCE_ID=$(aws ec2 run-instances --region $AWS_REGION \
  --image-id $AMI_ID --instance-type $INSTANCE_TYPE --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":120,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=gardener-dev}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --region $AWS_REGION --instance-ids $INSTANCE_ID
```

Get the public IP and connect:

```
export EC2_IP=$(aws ec2 describe-instances --region $AWS_REGION \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

ssh -i ~/.ssh/$KEY_NAME.pem ubuntu@$EC2_IP
```

#### 6.2. Prepare the machine (run inside EC2)

Install Docker:

```
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg make git jq
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
newgrp docker
```

Mark the local registry as insecure (it serves HTTP, not HTTPS):

```
echo '{ "insecure-registries": ["registry.local.gardener.cloud:5001"] }' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
```

Raise inotify limits (Gardener's KinD setup runs many containers):

```
sudo sysctl fs.inotify.max_user_instances=8192
sudo sysctl fs.inotify.max_user_watches=524288
```

Install the required tools:

```
sudo snap install go --classic
sudo snap install kubectl --classic
sudo snap install helm --classic
sudo snap install yq
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
chmod +x skaffold && sudo mv skaffold /usr/local/bin/skaffold
```

Confirm the host uses cgroup v2:

```
stat -fc %T /sys/fs/cgroup/
```

#### 6.3. Deploy Gardener (run inside EC2)

```
git clone https://github.com/gardener/gardener.git
cd gardener
make kind-up
make gardener-up
```

`make gardener-up` builds and deploys all components via Skaffold, then waits for the Seed. When it finishes you should see:

```
Last operation state is 'Succeeded' and all conditions passed for seed/local.
```

#### 6.4. Create a Shoot and test it (run inside EC2)

Switch to the virtual garden cluster and confirm the Seed is ready:

```
export KUBECONFIG=$PWD/dev-setup/kubeconfigs/virtual-garden/kubeconfig
kubectl get seed local
```

```
NAME    STATUS   LAST OPERATION               PROVIDER   REGION   VERSION        K8S VERSION
local   Ready    Reconcile Succeeded (100%)   local      local    v1.146.0-dev   v1.35.1
```

Apply the example Shoot and watch it build:

```
kubectl apply -f example/provider-local/shoot.yaml
kubectl -n garden-local get shoot local -w
```

On a native Linux box this climbs smoothly to:

```
NAME    PROVIDER   K8S VERSION   LAST OPERATION             STATUS
local   local      1.36.0        Create Succeeded (100%)    progressing
```

Access the Shoot and verify the worker node and system pods:

```
./hack/usage/generate-kubeconfig.sh > /tmp/shoot-admin.yaml
kubectl --kubeconfig /tmp/shoot-admin.yaml get nodes
kubectl --kubeconfig /tmp/shoot-admin.yaml get pods -A
```

```
NAME                                               STATUS   ROLES    VERSION
machine-shoot--local--local-local-z1-...           Ready    worker   v1.36.0
```

All `kube-system` pods (coredns, calico, metrics-server, vpn-shoot, ...) should be `Running`. That's the whole point of Gardener in one flow: you declared *what* you wanted in a Shoot manifest, and it built the cluster — control plane on the Seed, worker node provisioned automatically.

#### 6.5. Problems encountered and fixes

| Problem | Cause | Fix |
|---|---|---|
| Worker stuck at ~84% with `context deadline exceeded` (on macOS) | Docker Desktop's nested gVisor/virtiofs I/O is too slow for sandbox/image work | Run on a native Linux host (this EC2 setup) |
| `make kind-up` fails: `kubelet not healthy / required cgroups disabled` | Host on cgroup v1 (e.g. Ubuntu 20.04) | Use Ubuntu 22.04 (cgroup v2 by default), confirm with `stat -fc %T /sys/fs/cgroup/` |
| `jq: command not found` | `jq` not installed | `sudo apt-get install -y jq` |
| `http: server gave HTTP response to HTTPS client` | Local registry serves HTTP, Docker expects HTTPS | Add it to `insecure-registries` in `/etc/docker/daemon.json`, restart Docker |

#### 6.5. (Optional) Automated e2e test

​```
make test-e2e-local-simple
​```

#### 6.6. Clean up (stop paying)

Inside EC2, tear the cluster down (optional):

```
make kind-down
exit
```

Back on your local machine, delete all AWS resources:

```
aws ec2 terminate-instances --region $AWS_REGION --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $INSTANCE_ID
aws ec2 delete-security-group --region $AWS_REGION --group-id $SG_ID
aws ec2 delete-key-pair --region $AWS_REGION --key-name $KEY_NAME
rm -f ~/.ssh/$KEY_NAME.pem
```

If you lost the shell variables, find the instance by its tag first:

```
export AWS_REGION=eu-north-1
export INSTANCE_ID=$(aws ec2 describe-instances --region $AWS_REGION \
  --filters "Name=tag:Name,Values=gardener-dev" "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)
```

The root EBS volume has `DeleteOnTermination=true`, so it is removed with the instance. Always **terminate** (not stop) the instance, otherwise the EBS volume keeps incurring cost.

### References

1. https://gardener.cloud/

2. https://gardener.cloud/docs/gardener/deployment/getting_started_locally/

3. https://github.com/gardener/gardener/blob/master/docs/development/local_setup.md
