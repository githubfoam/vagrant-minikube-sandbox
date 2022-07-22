# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.

# Minikube
KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || "1.16.3" # OK
# X Exiting due to GUEST_MISSING_CONNTRACK: Sorry, Kubernetes 1.24.3 requires conntrack to be installed in root's path
# KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || "1.24.3" # NOT OK conntrack missing

$ubuntu_docker_script = <<-SCRIPT

# vg-minikube01: Package 'docker.io' is not installed, so not removed
# vg-minikube01: E: Unable to locate package docker
# vg-minikube01: E: Unable to locate package docker-engine
# The SSH command responded with a non-zero exit status. Vagrant
# assumes that this means the command failed. The output for this command
# should be in the log above. Please read the output to determine what
# went wrong.
# Uninstall old versions
# apt-get remove docker docker-engine docker.io containerd runc -y

# Set up the repository
apt-get update -y
apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release -y

# Add Docker’s official GPG key    
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# set up the stable repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


# Install Docker Engine
apt-get update -y
apt-get install \
    docker-ce \
    docker-ce-cli \
    containerd.io -y


docker --version

# Verify that Docker Engine is installed correctly     
docker run hello-world

# Post-installation steps for Linux
# Manage Docker as a non-root user

# Create the docker group
groupadd docker
# Add your user to the docker group
# usermod -aG docker $USER # by default run by root
usermod -aG docker vagrant

SCRIPT


$installer = <<SCRIPT
#!/bin/bash
sudo apt-get -y update
sudo apt-get install -y zip unzip curl wget socat ebtables 
SCRIPT

$docker = <<SCRIPT
#!/bin/bash
#curl -fsSL https://apt.dockerproject.org/gpg | sudo apt-key add -
#sudo apt-add-repository "deb https://apt.dockerproject.org/repo ubuntu-xenial main"
#sudo apt-get install -y docker-engine=17.03.1~ce-0~ubuntu-xenial
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get -y update
sudo apt-get install -y docker-ce
sudo systemctl start docker
sudo usermod -a -G docker vagrant
SCRIPT


$minikubescript = <<SCRIPT
#!/bin/bash

echo "current user is $(whoami)"
echo "current directory is $(pwd)"

#Install minikube
echo "Downloading Minikube"
curl -q -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 2>/dev/null
chmod +x minikube
sudo mv minikube /usr/local/bin/
stat /usr/local/bin/minikube # verify

#https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
#Install kubectl
echo "Downloading Kubectl"
curl -q -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/v${KUBERNETES_VERSION}/bin/linux/amd64/kubectl 2>/dev/null
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
stat /usr/local/bin/kubectl # verify

# Install crictl
curl -qL https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.16.1/crictl-v1.16.1-linux-amd64.tar.gz 2>/dev/null | tar xzvf -
chmod +x crictl
sudo mv crictl /usr/local/bin/
stat /usr/local/bin/crictl  # verify

#Install stern
# TODO: Check sha256sum
echo "Downloading Stern"
curl -q -Lo stern https://github.com/wercker/stern/releases/download/1.10.0/stern_linux_amd64 2>/dev/null
chmod +x stern
sudo mv stern /usr/local/bin/
stat /usr/local/bin/stern  # verify

#Install kubecfg
# TODO: Check sha256sum
echo "Downloading Kubecfg"
curl -q -Lo kubecfg https://github.com/ksonnet/kubecfg/releases/download/v0.9.0/kubecfg-linux-amd64 2>/dev/null
chmod +x kubecfg
sudo mv kubecfg /usr/local/bin/
stat /usr/local/bin/kubecfg  # verify

#Setup minikube
echo "127.0.0.1 minikube minikube." | sudo tee -a /etc/hosts
mkdir -p $HOME/.minikube
mkdir -p $HOME/.kube
touch $HOME/.kube/config

export KUBECONFIG=$HOME/.kube/config

# Permissions
echo "USER...:$USER"
echo "HOME...:$HOME"
sudo chown -R $USER:$USER $HOME/.kube
sudo chown -R $USER:$USER $HOME/.minikube

export MINIKUBE_WANTUPDATENOTIFICATION=false
export MINIKUBE_WANTREPORTERRORPROMPT=false
export MINIKUBE_HOME=$HOME
export CHANGE_MINIKUBE_NONE_USER=true
export KUBECONFIG=$HOME/.kube/config

# v1.24.3
apt-get update && apt-get -qq -y install conntrack #http://conntrack-tools.netfilter.org/

# Disable SWAP since is not supported on a kubernetes cluster
sudo swapoff -a

## Start minikube
sudo -E minikube start \
  -v 4 --vm-driver none \
  --kubernetes-version v${KUBERNETES_VERSION} \
  --bootstrapper kubeadm

minikube version --short
minikube version --components
minikube status

#post-install checks 
kubectl cluster-info
kubectl get nodes
kubectl get pod

## Addons
sudo -E minikube addons  enable ingress
minikube addons list #verify

# Permissions
sudo chown -R $USER:$USER $HOME/.kube
sudo chown -R $USER:$USER $HOME/.minikube

# Enforce sysctl
sudo sysctl -w vm.max_map_count=262144
sudo echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.d/90-vm_max_map_count.conf

kubectl get pod -n default -o wide  --all-namespaces
kubectl -n default get services

#https://kubernetes.io/docs/tutorials/hello-minikube/
echo "================================"
echo "Running Hello Minikube tutorial"
echo "================================"

#Create a Deployment 
#A Kubernetes Pod is a group of one or more Containers, tied together for the purposes of administration and networking
#The Pod in this tutorial has only one Container.
#A Kubernetes Deployment checks on the health of your Pod and restarts the Pod's Container if it terminates.
#Deployments are the recommended way to manage the creation and scaling of Pods.

#create a Deployment that manages a Pod. The Pod runs a Container based on the provided Docker image
kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4

#View the Deployment
kubectl get deployments 

#View the Pod
kubectl get pods

#View cluster events
kubectl get events

#View the kubectl configuration
kubectl config view

#Create a Service
#By default, the Pod is only accessible by its internal IP address within the Kubernetes cluster. 
#To make the hello-node Container accessible from outside the Kubernetes virtual network
#expose the Pod as a Kubernetes Service.

#Expose the Pod to the public internet
#The --type=LoadBalancer flag indicates to expose Service outside of the cluster
#The application code inside the image k8s.gcr.io/echoserver only listens on TCP port 8080
#On minikube, the LoadBalancer type makes the Service accessible through the minikube service
kubectl expose deployment hello-node --type=LoadBalancer --port=8080

#View the Service
kubectl get services

minikube service hello-node

#Verify
kubectl get services

#Enable addons
#List the currently supported addons
minikube addons list

#Enable metrics-server addon 
minikube addons enable metrics-server

#View the Pod and Service created
kubectl get pod,svc -n kube-system

#Disable metrics-server
# minikube addons disable metrics-server

#Clean up the resources
# kubectl delete service hello-node
# kubectl delete deployment hello-node

#stop the Minikube virtual machine (VM)
# echo "stopping minikube.."
# minikube stop


#delete the Minikube VM
# minikube delete

SCRIPT

$prometheus = <<SCRIPT
#!/bin/bash

#create seperate namespace
kubectl apply -f monitoring-namespace.yaml

# Create configMap for prometheus.
kubectl apply -f prometheus-config.yaml

# Create prometheus deployment
kubectl apply -f prometheus-deployment.yaml

# Create prometheus service.
kubectl apply -f prometheus-service.yaml

#verify
kubectl get services --namespace=monitoring prometheus -o yaml

#browser window accessing the service.
minikube service --namespace=monitoring prometheus

# Create grafana deployment
kubectl apply -f grafana-deployment.yaml

# Create grafana service.
kubectl apply -f grafana-service.yaml

# open a browser window accessing the service
#Username\password is admin\admin
minikube service --namespace=monitoring grafana


#Prometheus Node Exporter Daemonset
# run an instance of this on every node
kubectl apply -f node-exporter-daemonset.yaml

SCRIPT

Vagrant.configure("2") do |config|

  config.vm.provider "virtualbox" do |vb|
    # vb.gui = false
    vb.memory = "1024"
    vb.cpus = 2
    # vb.customize ["modifyvm", :id, "--groups", "/kali-sandbox"] # create vbox group
  end


    
    config.vm.define "vg-minikube-01" do |kalicluster|
      # https://app.vagrantup.com/ubuntu/boxes/hirsute64
      # kalicluster.vm.box = "ubuntu/hirsute64" #21.04
      # https://app.vagrantup.com/ubuntu/boxes/impish64
      # kalicluster.vm.box = "ubuntu/impish64" #21.10
      # https://app.vagrantup.com/ubuntu/boxes/focal64
      kalicluster.vm.box = "ubuntu/focal64" #Official Ubuntu 20.04 LTS (Focal Fossa) builds      
      # https://app.vagrantup.com/ubuntu/boxes/xenial64
      # kalicluster.vm.box = "ubuntu/xenial64" #16.04
      kalicluster.vm.hostname = "vg-minikube-01"
      #bridged network,DHCP disabled, manual IP assignment                  
      # kalicluster.vm.network "public_network", ip: "10.10.8.67"
      #bridged network,DHCP enabled,auto IP assignment
      # kalicluster.vm.network "public_network"
      kalicluster.vm.network "private_network", ip: "192.168.51.6"
      # kalicluster.vm.network "forwarded_port", guest: 80, host: 81
      #Disabling the default /vagrant share can be done as follows:
      # kalicluster.vm.synced_folder ".", "/vagrant", disabled: true
      kalicluster.vm.provider "virtualbox" do |vb|
          vb.name = "vbox-minikube-01"
          vb.cpus = 2
          vb.memory = 4096          
          vb.gui = false         
      end
      kalicluster.vm.provision "shell", inline: $installer, privileged: false
      kalicluster.vm.provision "shell", inline: $docker, privileged: false #OK
      # kalicluster.vm.provision "shell", inline: $ubuntu_docker_script
      kalicluster.vm.provision "shell", inline: $minikubescript, privileged: false, env: {"KUBERNETES_VERSION" => KUBERNETES_VERSION}
    end 

    config.vm.define "vg-minikube-02" do |kalicluster|
      # https://app.vagrantup.com/ubuntu/boxes/hirsute64
      # kalicluster.vm.box = "ubuntu/hirsute64" #21.04
      # https://app.vagrantup.com/ubuntu/boxes/impish64
      # kalicluster.vm.box = "ubuntu/impish64" #21.10
      # https://app.vagrantup.com/ubuntu/boxes/focal64
      # kalicluster.vm.box = "ubuntu/focal64" #Official Ubuntu 20.04 LTS (Focal Fossa) builds      
      # https://app.vagrantup.com/ubuntu/boxes/xenial64
      kalicluster.vm.box = "ubuntu/bionic64" #18.04      
      # https://app.vagrantup.com/ubuntu/boxes/xenial64
      # kalicluster.vm.box = "ubuntu/xenial64" #16.04
      kalicluster.vm.hostname = "vg-minikube-01"
      #bridged network,DHCP disabled, manual IP assignment                  
      # kalicluster.vm.network "public_network", ip: "10.10.8.67"
      #bridged network,DHCP enabled,auto IP assignment
      # kalicluster.vm.network "public_network"
      kalicluster.vm.network "private_network", ip: "192.168.51.7"
      # kalicluster.vm.network "forwarded_port", guest: 80, host: 81
      #Disabling the default /vagrant share can be done as follows:
      # kalicluster.vm.synced_folder ".", "/vagrant", disabled: true
      kalicluster.vm.provider "virtualbox" do |vb|
          vb.name = "vbox-minikube-02"
          vb.cpus = 2
          vb.memory = 4096          
          vb.gui = false     
          # vb.customize ["modifyvm", :id, "--cableconnected1", "on"]     
      end
      # kalicluster.vm.provision "shell",    inline: "hostnamectl set-hostname vg-kali-05"
      kalicluster.vm.provision "shell", inline: $installer, privileged: false
      # kalicluster.vm.provision "shell", inline: $docker, privileged: false #OK
      kalicluster.vm.provision "shell", inline: $ubuntu_docker_script
      kalicluster.vm.provision "shell", inline: $minikubescript, privileged: false, env: {"KUBERNETES_VERSION" => KUBERNETES_VERSION}
    end


end

