# Proof of Concept Kubernetes Cluster

This is intended for use provisioning Ubuntu 16.04 nodes via ssh.

From my understanding, automatic provisioning of nodes for Kubernetes is very cloud-platform dependent. 

In this case I am using DigitalOcean's cloud-init user-data doctl (CLI for DO), but I assume any provider supporting the [cloud-init user-data](https://cloudinit.readthedocs.io/en/latest/topics/format.html#example) would work fine.

## Step 1 (Create a master node with Kubeadm)

Basically a Kubernetes cluster has (at least one) Master node which ties the worker nodes together and manages services which expose API endpoints which manage Deployments, Dashboards, health checks, etc.

So, in order to control our cluster, we need a master node. This node can either be your laptop or a remote VPS or whatever. 

We need to run the provisioning script cnf_master to install the Docker runtime, various dependencies for installation, and the Kubernetes services Kubectl and Kubelet. We are using Kubeadm to init our cluster, so we need that too.

In order to quickly create a master node which will use the Calico Container Networking Interface [CNI](https://docs.projectcalico.org/v2.0/getting-started/kubernetes/) to connect with worker nodes. 

Let's create a new master node on DigitalOcean with [doctl](https://github.com/digitalocean/doctl/blob/master/README.md)

1 VCPU, 1 GB RAM, Ubuntu 16.04 x64
note: I am using the --ssh-keys flag to enable communication with my droplet without password

```bash
$ doctl compute droplet create k --image ubuntu-16-04-x64 --ssh-keys 23238460 --region nyc1 --size s-1vcpu-1gb --user-data-file ./cnf/cnf-master

```

In the background, the droplet will be provisioned with the user-data file we specify at ./cnf/cnf-master, which runs 

	kubeadm init --pod-network-cidr=192.168.0.0/16

note the pod network cidr flag is necessary for calico.

This brings up our basic services for our kubernetes master node and runs the API so that other nodes can join the cluster. This init script creates an access token which I am too lazy to pipe anywhere, so we can SSH to the droplet and create a new one in the kubeadm join syntax like so:

	$ sudo kubeadm token create --print-join-command 

this will output something similar to:

	$ kubeadm join --discovery-token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:1234..cdef 1.2.3.4:6443

which we then run on all worker nodes we provision by adding the line to the cnf worker file at the end to replace 

	# - KUBEADM JOIN COMMAND HERE

Then, we provision n worker nodes by running doctl again from the command line 

```bash
doctl compute droplet create k1 --image ubuntu-16-04-x64 --ssh-keys 23238460 --region nyc1 --size s-1vcpu-1gb --user-data-file ./cnf/cnf-worker
doctl compute droplet create k2 --image ubuntu-16-04-x64 --ssh-keys 23238460 --region nyc1 --size s-1vcpu-1gb --user-data-file ./cnf/cnf-worker
doctl compute droplet create k3 --image ubuntu-16-04-x64 --ssh-keys 23238460 --region nyc1 --size s-1vcpu-1gb --user-data-file ./cnf/cnf-worker
etc...
```

If all goes well, we now have worker nodes connected to the master node and networking via Calico.

One bummer of using Kubeadm is that currently the Kubernetes WebUI only supports manual user auth by username and password, which is less secure and somewhat hacky. 

So, I will have to figure out what to do about that. 