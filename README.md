# Install Kubernetes on Raspberry Pis running Ubuntu 20.04 for your own private cloud container service

Kubernetes is an enterprise-grade container orchestration system designed from the beginning to be cloud-native. It has grown to be the de-facto cloud container platform and expanded as it has embraced new technologies, like container-native virtualizaton and serverless computing. Kubernetes manages containers and more from micro-scale at the edge to massive scale in both public and private cloud environments. It is a perfect choice for a "private cloud at home" project, providing both robust container orchestration and the opportunity to learn about a technology in such demand and so thoroughly integrated into the cloud that its name is practically synonymous with "cloud computing".

Nothing says "cloud" quite like Kubernetes, and nothing screams "cluster me!" quite like Raspberry Pis. Running a local Kubernetes cluster on cheap Raspberry Pi hardware is a great way to gain experience managing and developing on a true cloud technology giant.

## Installing your own Kubernetes cluster on Raspberry Pis

For today's exercise, we will install a Kubernetes 1.18.2 cluster on three or more Raspberry Pi 4s running Ubuntu 20.04. Ubuntu 20.04 (Focal Fossa) offers a Raspberry Pi-focused 64-bit ARM (arm64) image with both 64-bit kernel and userspace.  Since we want to use these Raspberry Pis for running a Kubernetes cluster, the ability to run aarch64 container images is important: it can be difficult to find 32-bit images for common software or even standard base images. With its arm64 image, Ubuntu 20.04 allows us use 64-bit container images with Kubernetes.

### aarch64 vs arm64, 32-bit vs 64-bit, ARM vs x86

Note that aarch64 and arm64 are effectively the same thing. The different names arise from usage within different communities. Many container images will be labeled aarch64, and will run fine on systems labeled arm64.  Systems with aarch64/arm64 architecture are capable of running 32-bit ARM images, as well, but the opposite is not true - a 32-bit ARM system cannot run 64-bit container images. This is why the Ubuntu 20.04 arm64 image is so useful.

Without getting too deep in the woods explaining different architecture types, it is worth noting that arm64/aarch64 and x86_64 architectures differ, and Kubernetes nodes running on 64-bit ARM architecture cannot run container images built for x86_64.  In practice, you will find some images that are not built for both architectures, and may not be usable in your cluster. You will also need to build your own images on an aarch64-based system, or jump through some hoops to allow your regular x86_64 systems to build aarch64 images. We will cover how to build aarch64 images on your regular system in a future article in the "private cloud at home" project.

For the best of both worlds, x86_64 nodes can be added later to the Kubernetes cluster we are setting up today. Images of a given architecture can be scheduled to run on the appropriate nodes by Kubernetes' scheduler through the use of [Kubernetes Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).

Enough about architectures and images.  It's time to install Kubernetes. let's get to it!

### Requirements

The requirements for the exercise today are minimal. You will need:

* 3 (or more) Raspberry Pi 4s (preferably the 4GB RAM models)
* [Ubuntu 20.04 arm64](https://ubuntu.com/download/server/arm) installed on the Raspberry Pis

To make the initial setup more convenient, you may wish to read a previous article, [Modify a disk image to create a Raspberry Pi-based homelab](https://opensource.com/article/20/5/disk-image-raspberry-pi), to add a user and SSH authorized_keys to the Ubuntu image before writing it to an SD card and installing on the Raspberry Pi.

## Configure the hosts

Once Ubuntu is installed on the Raspberry Pis and they are accessible via SSH, a few changes need to be made before Kubernetes can be installed.

### Install and configure Docker

At the time of this writing, Ubuntu 20.04 ships the most recent version of Docker, v19.03, in the base repositories and can be installed directly using the `apt` command. Note that the package name is "docker.io". Install Docker on all of the Raspberry Pis:

```shell
# Install the docker.io package
$ sudo apt install -y docker.io
```

After the package is installed, we need to make some changes to enable [cgroups, or Control Groups](https://en.wikipedia.org/wiki/Cgroups). Cgroups allow the Linux kernel to limit and isolate resources. Practically speaking, this allows Kubernetes to better manage resources used by containers it runs, and increases security by isolating containers from one another.

Check the output of `docker info` before making the following changes, on all of the RPis:

```shell
# Check `docker info`
# Some output omitted
$ sudo docker info
(...)
 Cgroup Driver: cgroups
(...)
WARNING: No memory limit support
WARNING: No swap limit support
WARNING: No kernel memory limit support
WARNING: No kernel memory TCP limit support
WARNING: No oom kill disable support
```

The output above highlights the bits that need to be changed: the Cgroup driver, and limit support.

First, change the default cgroups driver used by Docker from `cgroups` to `systemd`, to allow systemd to act as the a cgroups manager and ensure there is only one cgroup manager in use. This helps with system stability and is recommended by Kubernetes. To do this, create or replace the "/etc/docker/daemon.json" file with the following:

```shell
# Create or replace the contents of /etc/docker/daemon.json to enable the systemd cgroup driver

$ sudo cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### Enable cgroups limit support

Next, we need to enable limit support, as shown by the warnings in the output from `docker info`, above. We need to modify the kernel command line to enable these options at boot. For the Raspberry Pi 4, the following changes are added to the "/boot/firmware/cmdline.txt" file.

* cgroup_enable=cpuset
* cgroup_enable=memory
* cgroup_memory=1
* swapaccount=1

Make sure they are added to the end of the line in the cmdline.txt file. This can be accomplished in one line using `sed`:

```shell
# Append the cgroups and swap options to the kernel command line
# Note the space before "cgroup_enable=cpuset", to add a space after the last existing item on the line
$ sudo sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt
```

The command above matches the end of the line (the first `$`) then replaces the end of the line with the options listed, effectively appending them at the end.

With those changes made, Docker and the kernel should be configured as needed for Kubernetes. Reboot the Raspberry Pis, and when they come back up, check the output of `docker info` again.  The `Cgroups driver` should now be `systemd`, and the warnings should be gone.

### Allow iptables to see bridged traffic

According to the documentation, Kubernetes needs iptables to be configured to see bridged network traffic. This can be done by changing the sysctl config:

```shell
# Enable net.bridge.bridge-nf-call-iptables and -iptables6
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sudo sysctl --system
```

### Install the Kubernetes packages for Ubuntu

Since we are using Ubuntu, we can install the Kubernetes packages from the Kubernetes.io Apt repository. There is not currently a repository for Ubuntu 20.04 (focal), but Kubernetes 1.18.2 is available in the last Ubuntu LTS repository: Ubuntu 18.04 (xenial). The latest Kubernetes packages can be installed from there.

Add the Kubernetes repo to Ubuntu's Sources:

```shell
# Add the packages.cloud.google.com atp key
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# Add the Kubernetes repo
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

When Kubernetes does add a Focal repository, make sure to switch to that - perhaps when the next Kubernetes version is released.

With the repository added to the Sources list, now install the three required Kubernetes packages: kubelet, kubeadm, and  kubectl.

```shell
# Update the apt cache and install kubelet, kubeadm, and kubectl
# (Output omitted)
$ sudo apt update && sudo apt install -y kubelet kubeadm kubectl
```

Finally, use the `apt-mark hold` command to disable regular updates for these three packages. Upgrades to Kubernetes need more hand-holding than is possible with the general update process, and will require manual attention.

```shell
# Disable (mark as held) updates for the Kubernetes packages
$ sudo apt-mark hold kubelet kubeadm kubectl
kubelet set on hold.
kubeadm set on hold.
kubectl set on hold.
```

That is it for the host configuration!  Now we can move on to setting up Kubernetes itself.

## Create a Kubernetes Cluster

With the Kubernetes packages installed, we can continue creating an actual cluster. Before getting started, some decisions must be made. First, one of the Raspberry Pis needs to be designated the Control Plane, or primary, node. The remaining nodes will be designated as compute nodes.

Also necessary will be picking a network CIDR to use for the pods in the Kubernetes cluster. Setting the `pod-network-cidr` during the cluster creation ensure that the `podCIDR` value is set and can be used by the Container Network Interface (CNI) add-on later. In this exercise, we will be using the [Flannel](https://github.com/coreos/flannel) CNI. The CIDR should not overlap any currently used within your home network, and not be one managed by your router or DHCP server. Make sure to use a subnet that is larger than you expect to need: there are ALWAYS more pods than initially planned for! We will use 10.244.0.0/16 for this exercise, but pick one tha works for you.

With those decisions out of the way, we can initialize the Control Plane node. SSH or otherwise login to the node you have designated for the Control Plane.

### Initialize the Control Plane

Kubernetes uses a bootstrap token to authenticate nodes being joined to the cluster. This token needs to be passed to the `kubeadm init` command when initializing the Control Plane node.  Generate a token to use with the `kubeadm token generate` command:

```shell
# Generate a bootstrap token to authenticate nodes joining the cluster
$ TOKEN=$(sudo kubeadm token generate)
$ echo $TOKEN
d584xg.xupvwv7wllcpmwjy
```

We are now ready to initialize the Control Plane, using the `kubeadm init` command.

```shell
# Initialize the Control Plane
# (output omitted)
$ sudo kubeadm init --token=${TOKEN} --kubernetes-version=v1.18.2 --pod-network-cidr=10.244.0.0/16
```

If everything is successful, at the end of the output you should see something similar to this:

```txt
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.2.114:6443 --token zqqoy7.9oi8dpkfmqkop2p5 \
    --discovery-token-ca-cert-hash sha256:71270ea137214422221319c1bdb9ba6d4b76abfa2506753703ed654a90c4982b
```

Make a note of two things - first, the Kubernetes `kubectl` connection information has been written to "/etc/kubernetes/admin.conf".  This kubeconfig file can be copied to "~/.kube/config", either for root or a normal user on the master node, or to a remote machine. This will allow you to control your cluster with the `kubectl` command.

Second, the last line of the output starting with `kubernetes join` is a command you can run to join more nodes to the cluster.

After copying the new kubeconfig to somewhere your user can use it, you can validate the Control Plane has been installed with the `kubectl get nodes` command:

```shell
# Show the nodes in the Kubernetes cluster
# Your node name will vary
$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
elderberry   Ready    master   7m32s   v1.18.2
```

### Install a Container Network Interface add-on

A Container Network Interface (CNI) add-on handles configuration and cleanup of the pod networks. As mentioned above, in this exercise we will be using the Flannel CNI add-on.  With the `podCIDR` value already set, we can just download the Flannel YAML and use `kubectl apply` to install it into the cluster. This can be done on one line using `kubectl apply -f -` to take the data from standard input.  This will create the ClusterRoles, ServiceAccounts and DaemonSets (etc.) necessary to manage the pod networking.

Download and apply the Flannel YAML data to the cluster:

```shell
# Download the Flannel YAML data and apply it
# (output omitted)
$ curl -sSL https://raw.githubusercontent.com/coreos/flannel/v0.12.0/Documentation/kube-flannel.yml | kubectl apply -f -
```

### Join the compute nodes to the cluster

With the CNI add-on in place, it is now time to add compute nodes to the cluster. Joining the compute nodes is just a matter of running the `kubeadm join` command provided at the end of the `kube init` command run to initialize the Control Plane node. For the remaining Raspberry Pis you wish to join to your cluster, login to the host, and run the command.

```shell
# Join a node to the cluster - your tokens and ca-cert-hash will vary
$ sudo kubeadm join 192.168.2.114:6443 --token zqqoy7.9oi8dpkfmqkop2p5 \
    --discovery-token-ca-cert-hash sha256:71270ea137214422221319c1bdb9ba6d4b76abfa2506753703ed654a90c4982b
```

Once the join process has completed on each node, you should be able to see the new nodes in the output of `kubectl get nodes`:

```shell
# Show the nodes in the Kubernetes cluster
# Your node name will vary
$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
elderberry   Ready    master   7m32s   v1.18.2
gooseberry    Ready    <none>   2m39s   v1.18.2
huckleberry   Ready    <none>   17s     v1.18.2
```

### Validate the cluster

At this point, you have a fully working Kubernetes cluster. Pods can be run, deployments and jobs created, etc. Applications running in the cluster can be accessed from any of the nodes in the cluster using [Services](https://kubernetes.io/docs/concepts/services-networking/service/). External access can be achieved with a NodePort service, or ingress controllers.

To validate the cluster is running, we will create a new namespace, deployment and service, and check that the pods running in the deployment respond as expected. This deployment uses the "quay.io/clcollins/kube-verify:01" image - an Nginx container listening for requests (actually, the same image used in the previous article [Create a simple Cloud-init service for your homelab](https://opensource.com/article/20/5/create-simple-cloud-init-service-your-homelab)). [The image Containerfile can be viewed here.](https://github.com/clcollins/homelabCloudInit/blob/master/simpleCloudInitService/data/Containerfile)

Create a namespace named `kube-verify` for the deployment:

```shell
# Create a new namespace
$ kubectl create namespace kube-verify
# List the namespaces
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   63m
kube-node-lease   Active   63m
kube-public       Active   63m
kube-system       Active   63m
kube-verify       Active   19s
```

Now, create a deployment in the new namespace:

```shell
# Create a new deployment
$ cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-verify
  namespace: kube-verify
  labels:
    app: kube-verify
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kube-verify
  template:
    metadata:
      labels:
        app: kube-verify
    spec:
      containers:
      - name: nginx
        image: quay.io/clcollins/kube-verify:01
        ports:
        - containerPort: 8080
EOF
deployment.apps/kube-verify created
```

Kubernetes will now start creating the deployment, consisting of three pods each running the "quay.io/clcollins/kube-verify:01" image. After a minute or so, the new pods should be running, and can be viewed with `kubectl get all -n kube-verify` to list all the resources created in the new namespace.

```shell
# Check the resources that were created by the deployment
$ kubectl get all -n kube-verify
NAME                               READY   STATUS              RESTARTS   AGE
pod/kube-verify-5f976b5474-25p5r   0/1     Running             0          46s
pod/kube-verify-5f976b5474-sc7zd   1/1     Running             0          46s
pod/kube-verify-5f976b5474-tvl7w   1/1     Running             0          46s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kube-verify   3/3     3            3           47s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/kube-verify-5f976b5474   3         3         3       47s
```

The new deployment can be seen, as well as a replicaset created by the deployment, and three pods created by the replicaset to fulfill the `replicas: 3` request in the deployment. We can see the internals of Kubernetes are working.

Now create a Service to expose the Nginx "application" (or, in this case, the Welcome page) running in the three pods. This will act as a single endpoint through which we can connect to the pods.

```shell
# Create a service for the deployment
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Service
metadata:
  name: kube-verify
  namespace: kube-verify
spec:
  selector:
    app: kube-verify
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
service/kube-verify created
```

With the service created, we can examine it and get the IP address for our new service:

```shell
# Examine the new service
$ kubectl get -n kube-verify service/kube-verify
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kube-verify   ClusterIP   10.98.188.200   <none>        80/TCP    30s
```

You can see in this example the "kube-verify" service has been assigned a ClusterIP (internal to the cluster only) of `10.98.188.200`. This IP is reachable from any of our nodes, but not from outside of the cluster.  We can verify the containers inside our deployment are working by connecting to them at this IP:

```shell
# Use curl to connect to the ClusterIP:
# (output truncated for brevity)
$ curl 10.98.188.200
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
    "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
<head>
```

Success! Our service is running and Nginx inside the containers is responding to our requests.

At this point, we have a running Kubernetes cluster on our Raspberry Pis, with a CNI add-on (Flannel) installed, and a test deployment and service running an Nginx webserver. In the large public clouds, Kubernetes has different ingress controllers to interact with the public cloud provider's different load balancer solutions, such as AWS's Elastic Load Balancers. Similarly, in private clouds there are ingress controllers for interacting with hardware load balancer appliances (like F5 Networks' load balancers) or Nginx and HAProxy controllers for handling traffic coming into the nodes. In a future article, we will tackle exposing services in the cluster to the outside world by installing our own ingress controller.

We will also look at dynamic storage provisioners and StorageClasses for allocating persistent storage for our applications in a future article, including making use of the [NFS server we setup in a previous article](https://opensource.com/article/20/5/nfs-raspberry-pi) to create on-demand storage for our pods.

## Go forth, and Kubernetes

If "Kubernetes" (κυβερνήτης) is Greek for pilot, an individual that steers a ship, does that also mean "Kubernetes" is greek for pilot, the action of guiding the ship?  Eh, no.  "Kubernan" (κυβερνάω) is apparently the Greek for "to pilot" or "to steer", so go forth and Kubernan, and if you see me out at a conference or something, give me a pass for trying to verb a noun. From another language. That I don't speak.

Disclaimer: As mentioned, I don't read or speak Greek, especially the ancient variety, so I'm choosing to believe something I read on the internet. You know how that goes. Take it with a grain of salt, and give me a little break since I didn't make a "It's all Greek to me" joke.  However, just mentioning it, I therefore was able to make the joke without actually making it, so I'm either sneaky, or clever, or both.  Or, neither. I didn't claim it was a _good_ joke.

Go fourth and pilot your containers like a pro with your own Kubernetes container service in  your Private Cloud at Home!  As you become more comfortable, you can modify your k8s cluster to try different options, like the aforementioned ingress controllers and dynamic StorageClasses for persistent volumes. This continuous learning is at the heart of [DevOps](https://opensource.com/tags/devops) and the continuous integration and delivery of new services mirrors the Agile methodology - both of which have been embraced as we all learned to deal with the massive scale enabled by the cloud and discovered our traditional practices were unable to keep pace.

Look at that!  Technology, policy, philosophy, a _tiny_ bit of Greek, and a terrible meta-joke, all in one article!
