# How to automate deep learning training with Kubernetes GPU-cluster

This guide should help fellow researchers and hobbyists to easily automate and accelerate there deep leaning training with their own Kubernetes GPU cluster.</br>
Therefore I will explain how to easily setup a GPU cluster on multiple Ubuntu 16.04 bare metal servers and provide some useful scripts and .yaml files that do the entire setup for you.

By the way: If you need a Kubernetes GPU-cluster for other reasons, this guide might be helpful to you as well.

**Why did i write this guide?**</br>
I have worked as in intern for the Startup [understand.ai](https://understand.ai) and noticed the hassle of firstly designing a machine learning algorithm locally and than bringing it to the cloud for training with different parameters and datasets.</br>
The second part, bringing it to the cloud for extensive training, takes always longer than thought, is frustrating and involves usually a lot of pitfalls.

For this reason i decided to work on this problem and make the second part effortless, easy and quick.</br>
The result of this work is this handy guide, that describes how everyone can setup their own Kubernetes GPU cluster to accelerate their work.

**The new process for the deep learning researchers:**</br>
The automated deep learning training with a Kubernetes GPU-cluster improves the process of brining your algorithm for training in the cloud significantly.

This illustration visualizes the new workflow, that involves only two simple steps:</br>
![My inspiration for the project, designed by Langhalsdino.](resources/description.jpg?raw=true "My inspiration for the project")

**Disclaimer**</br>
Be aware, that the following sections might be opinionated. Kubernetes is an evolving, fast paced environment, which means this guide will probably be outdated at times, depending on the authors spare time and individual contributions. Due to this fact contributions are highly appreciated.

## Table of Contents

  * [Quick Kubernetes revive]()
  * [Rough overview on the structure of the cluster]()
  * [Initiate nodes]()
    - [Constraints of my setup]()
    - [Setup instructions]()
        - [Use fast setup script]()
        - [Manually step by step instructions]()
  * [How to build your GPU container]()
  * [Some helpful commands]()
  * [Acknowledgements]()
  * [License]()

## Quick Kubernetes revive

**These articles might be helpful, if you need to refresh your Kubernetes knowledge:**

  * [Introduction to Kubernetes by DigitalOcean](https://www.digitalocean.com/community/tutorials/an-introduction-to-kubernetes)
  * [Kubernetes concepts](https://kubernetes.io/docs/concepts/)
  * [Kubernetes by example](http://kubernetesbyexample.com/)
  * [Kubernetes basics - interactive tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

## Rough overview on the structure of the cluster
The main idea is, to have a small CPU only master node, that controls a cluster of GPU-worker nodes.
![Rough overview on the structure of the cluster, designed by Langhalsdino](resources/System-overview.jpg?raw=true "Rough overview")

## Initiate nodes
Before we can use the cluster, it is important to firstly initiate the cluster. </br>
Therefore each node has to be manually initiated and joined to the cluster.

### Constraints of my setup
This are the constraints for my setup, I have been in some places tighter than necessary, but this is my setup and it worked for me 😒  

**Master**

+ Ubuntu 16.04
+ SSH access with sudo user
+ Internet access
+ ufw deactivated (not recommended, but for ease of use)
+ Enabled Ports (udp and tcp)
    - 6443, 443, 8080
    - 30000-32767 (only if your apps need them)
    - These will be used to access services from outside of the cluster

**Worker**

+ Ubuntu 16.04
+ SSH access with sudo user
+ Internet access
+ ufw deactivated (not recommended, but for ease of use)
+ Enabled Ports (udp and tcp)
    - 6443, 443

### Setup instructions
These instruction cover my experience on Ubuntu 16.04 and may or may not be suited to transfer to other OS’s.

I have created two scripts that fully initiate the master and worker node as described bellow. If you want to take the fast track, just use them. Otherwise, i recommended to read the step by step instructions.

<h4>Fast Track - Setup script</h4>
Ok, lets take the fast track. Copy the corresponding scripts on your master and workers.</br>
Furthermore make sure that your setup fits into my constraints.

**MASTER NODE**

Execute the initialization script and remember the token 😉 <br/>
The token will look like this: ```—token f38242.e7f3XXXXXXXXe231e```.

```
chmod +x init-master.sh
sudo ./init-master.sh <IP-of-master>
```

**WORKER NODE**

Execute the initialization script with the correct token and IP of your master.<br/>
The port is usually ```6443```.

```
chmod +x init-worker.sh
sudo ./init-worker.sh <Token-of-Master> <IP-of-master>:<Port>
```

<h4>Detailed step by step instructions</h4>

**MASTER NODE**

**1.** Add Kubernetes Repository to the packagemanager
```
sudo su -
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
exit
```

**2.** Install docker-engine, kubeadm, kubectl, kubernetes-cni

```
sudo apt-get install -y docker-engine
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
sudo groupadd docker
sudo usermod -aG docker $USER
echo 'You might need to reboot / relogin to make docker work correctly'
```

**3.** Since we want to build a cluster that uses GPUs we need to enable GPU acceleration in the master node.
Keep in mind, that this instruction may become obsolete or change completely in a later version of Kubernetes!

**3.I**
Add GPU support to the Kubeadm configuration, while cluster is not initialized.
```
sudo vim /etc/systemd/system/kubelet.service.d/<<Number>>-kubeadm.conf
```
append ExecStart with the flag ```—feature-gates="Accelerators=true"```, so it will look like this:
```
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS [...] --feature-gates="Accelerators=true"
```

**3.II** Restart kubelet
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**4.** Now we will initialize the master node.<br/>
Therefore you will need the IP of your master node.
Furthermore this step will provide you with the credentials to add further worker nodes, so remember your token 😉 </br>
The token will look like this: ``` —token f38242.e7f3XXXXXXXXe231e 130.211.XXX.XXX:6443```
```
sudo kubeadm init --apiserver-advertise-address=<ip-address>
```
**5.** Since Kubernetes 1.6 changed from ABAC roll-management to RBAC we need to advertise the credentials of the user.
You will need to perform this step for each time you will log into the machine!!
```
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```

**6.** Install network add-on that your pods can communicate with each other. Kubernetes 1.6 has some requirements for the network add-on, some of them are:

 + CNI-based networks
 + RBAC support

This GoogleSheet contains a selection of suitable network add- on GoogleSheet-Network-Add-on-vergleich .
I will use wave-works, just because of my personal preference ;)
```
kubectl apply -f https://git.io/weave-kube-1.6
```
**5.II** You are ready to go, maybe check your pods to confirm that everything is working ;)
```
kubectl get pods —all-namespaces
```
**N.** If you want to tear down your master, you will need to reset the master node
```
sudo kubeadm reset
```

**WORKER NODE**

The beginning should be familiar to you and make this process a lot faster ;)

**1.** Add Kubernetes Repository to the packagemanager
```
sudo su -
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
exit
```

**2.** Install docker-engine, kubeadm, kubectl, kubernetes-cni

```
sudo apt-get install -y docker-engine
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
sudo groupadd docker
sudo usermod -aG docker $USER
echo 'You might need to reboot / relogin to make docker work correctly'
```

**3.** Since we want to build a cluster that uses GPUs we need to enable GPU acceleration in the worker nodes that have a GPU installed.
Keep in mind, that this instruction may become obsolete or change completely in a later version of Kubernetes!

**3.I**
Add GPU support to the Kubeadm configuration, while cluster is not initialized.
```
sudo vim /etc/systemd/system/kubelet.service.d/<<Number>>-kubeadm.conf
```
append ExecStart with the flag ```—feature-gates="Accelerators=true"```, so it will look like this:
```
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS [...] --feature-gates="Accelerators=true"
```

**3.II** Restart kubelet
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**4.** Now we will add the worker to the cluster.<br/>
Therefore you will need to remember the token from your master node, so take a deep dive into your notes xD
```
sudo kubeadm join --token f38242.e7f3XXXXXXe231e 130.211.XXX.XXX:6443
```
**5.** Finished, check your nodes on your master and see if everything worked.
```
kubectl get nodes
```
**N.** If you want to tear down your worker node, you will need to remove the node from the cluster and reset the worker node.
Furthermore it will be beneficial to remove the worker node from the cluster  
***On master:***   
```
kubectl delete node <worker node name>
```
***On worker node***
```     
sudo kubeadm reset
```

**Client**

In order to control your cluster e.g. your master from your client, you will need to authenticate your client with the right user.
This guid won’t cover creating a separate user for client, we will just copy the user from the master node.<br/>
This will be easier, trust me 🤓 </br>
[Instruction to add custom user, will be added in the future]

**1.** Install kubectl on your client. I have only tested it on may mac, but linux should work as well.
I don’t know about windows, but who cares about windows anyway :D</br>
**On Mac**
```
brew install kubectl
```
**2.** Copy the admin authentication from the master to your client
```
scp uai@130.211.XXX.64:~/admin.conf ~/.kube/
```
**3.** Add the admin.conf configuration and credentials to Kubernetes configuration. You will need to do this for every agent
```
export KUBECONFIG=~/.kube/admin.conf
```
You are ready to use kubectl on you local client.

**3.II** You can test by listing all your pods
```
kubectl get pods —all-namespaces
```


**Install Kubernetes dashboard**

The kubernetes dashboard is pretty beautiful and gives script kiddies like me access to a lot of functionality.
In order to use the dashboard you will need to get your client running, RBAC will ensure it 👮

**You can perform this steps directly on the master or from your client**

**1.** Check if the dashboard is already installed
kubectl get pods --all-namespaces | grep dashboard

**2.** If the dashboard isn’t installed, install it ;)
```
kubectl create -f https://git.io/kube-dashboard
```
If this did not work check if the container defined in the .yaml [git.io/kube-dashboard](https/git.io/kube-dashboard) exist. (This bug cost me a lot of time)

In order to have access to your dashboard you will need to be authenticated with you client.

**3.** Proxy the dashboard to your client
```
kubectl proxy
```

**4.** Access the dashboard within your browser by visiting
[127.0.0.1:8001/ui](127.0.0.1:8001/ui)

## How to build your GPU container
This guide should help you to get a Docker container running, that needs GPU access.

For this guide i have chosen to build an example Docker container, that uses TensorFlow GPU binaries and can run TensorFlow programs in a Jupyter notebook.

Keep in mind, that this guide has been written for Kubernetes 1.6, therefore further changes can compromise this guide.

### Essential parts of .yml
In order to get your Nvidia GPU with CUDA running you have to pass the Nvidia driver and CUDA libraries to your container.
So we will use hostPath to make them available to the Kubernetes pod.
The actual path differ from machine to machine, since they are set by your Nvidia driver and CUDA installation.
```
volumes:
    - hostPath:
        path: /usr/lib/nvidia-375/bin name: bin
    - hostPath:
        path: /usr/lib/nvidia-375
        name: lib
```
Mount the volumes with the driver and CUDA in the right directory for your container. These might differ, due to specific requirements of your container.
```
volumeMounts:
    - mountPath: /usr/local/nvidia/bin
        name: bin
    - mountPath: /usr/local/nvidia/lib
        name: lib
```
Since you want to tell Kubernetes that you need n GPUs , you can define your requirements here.
```
resources:
    limits:
        alpha.kubernetes.io/nvidia-gpu: 1
```
Thats it, it is everything you need to build your Kuberntes 1.6 container 😏

Some note at the end, that describes my overall experience:<br/>
**Kubernetes + Docker + Machine Learning + GPUs = Pure awesomeness**

### Example GPU deployment
My example-gpu-deployment.yaml file describes two parts, a deployment and a service, since i want to make jupyter notebook available form the outside.

Run kubectl apply to make it available to the outside
```
kubectl create -f deployment.yaml
```

The deployment.yaml file looks like this:
```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: tf-jupyter
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: tf-jupyter
    spec:
      volumes:
      - hostPath:
          path: /usr/lib/nvidia-375/bin
        name: bin
      - hostPath:
          path: /usr/lib/nvidia-375
        name: lib
      containers:
      - name: tensorflow
        image: tensorflow/tensorflow:0.11.0rc0-gpu
        ports:
        - containerPort: 8888
        resources:
          limits:
            alpha.kubernetes.io/nvidia-gpu: 1
        volumeMounts:
        - mountPath: /usr/local/nvidia/bin
          name: bin
        - mountPath: /usr/local/nvidia/lib
          name: lib
---
apiVersion: v1
kind: Service
metadata:
  name: tf-jupyter-service
  labels:
    app: tf-jupyter
spec:
  selector:
    app: tf-jupyter
  ports:
  - port: 8888
    protocol: TCP
    nodePort: 30061
  type: LoadBalancer
---
```

## Some helpful commands

**Get commands** with basic output
```
kubectl get services                 # List all services in the namespace
kubectl get pods --all-namespaces    # List all pods in all namespaces
kubectl get pods -o wide             # List all pods in the namespace, with more details
kubectl get deployment my-dep        # List a particular deployment
```

**Describe commands** with verbose output
```
kubectl describe nodes <node-name>
kubectl describe pods <pod-name>
```

**Deleting Resources**
```
kubectl delete -f ./pod.yaml                   # Delete a pod using the type and name specified in pod.yaml
kubectl delete pod,service baz foo             # Delete pods and services with same names "baz" and "foo"
kubectl delete pods,services -l name=<Label>   # Delete pods and services with label name=myLabel
kubectl -n <namespace> delete po,svc --all     # Delete all pods and services in namespace my-ns
```

**Get into the bash console** of one of your pods:

```
kubectl exec -it <pod-name> — /bin/bash
```

## Common issues

Some people contacted me with some issues on their CUDA deployment related to the forwarding of drivers.<br>
If the example-gpu-deployment.yaml is not working for you, i would recommended you to try to install CUDA as described by this guide [Installing Tenserflow on Ubuntu](http://simonboehm.com/tech/2017/06/23/installingTensorFlow.html) in more detail and try the example-gpu-deployment-nvidia-375-82.yaml.
It might be necessary to adjust the version number in the yaml file.

If you encountered another issue, feel free to open an issue on github.

## Acknowledgements
There are a lot of guides, github repositories, issues and people out there who helped me a lot. </br>
So I want to thank everybody for their help.</br>
Specially the Startup [understand.ai](http://understand.ai) for their support.

### Authors

* **Frederic Tausch** - *Initial work* - [Langhalsdino](https://github.com/Langhalsdino)

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details
