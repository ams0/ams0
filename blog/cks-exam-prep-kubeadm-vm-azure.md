copy to clipboard history

## CKS Exam Prep: Kubeadm VM

## in Azure

*If you want to prepare for the CKS exam, you’ll need a VM to learn on! Here’s I show you how to create a VM in Azure easily and ready to go. This blog is part of a series, check [here](https://medium.com/cooking-with-azure/certified-kubernetes-security-specialist-cks-exam-guide-a8fc2b4c47ea) the index.*

![](https://cdn-images-1.medium.com/max/3256/1*Ga31eAKWEplbJzZrUaxPeg.png)

The Certified Kubernetes Security Specialist (CKS) exam is a recently released certification from the Linux Foundation meant to test your skills to create and secure an hardened Kubernetes cluster. The cluster you’ll have to deal with during the exam are deployed with the standard, official Kubernetes community tool *kubeadm*; hence, you’ll be better off studying for the exam on the real deal than other distributions (although there are really good ones out there, like [KIND](https://kind.sigs.k8s.io/)).

Since I ❤ Azure, I want to share with you an effortless way to quickly obtain a VM running in Azure with Kubeadm, using *containerd* as container runtime (that’s what you’ll get during the exam) and *canal *(calico+flannel) as CNI (so you can use NetworkPolicies). Ready? Let’s start

First download the script:

    wget [http://bit.ly/kubeadm-containerd](http://bit.ly/kubeadm-containerd)

Then create a resource group in Azure and deploy the Virtual Machine pointing to the script as *custom-data*:

    az group create -n cks

    az vm create -g cks -n cks '\
    --image  UbuntuLTS \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --admin-username cks \
    --size Standard_B4ms \
    --custom-data kubeadm-containerd.sh

*Voila’!* In a few minutes you’ll be able to SSH into the VM and enjoy a fully functional 1-node kubeadm cluster with containerd and canal. Here’s how you do that:

    > ssh csk@<VM IP>
    # sudo su
    $ export KUBECONFIG=/etc/kubernetes/admin.conf
    $ alias k=kubectl

    $ k get no -o wide
    NAME   STATUS   ROLES                  AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    cks    Ready    control-plane,master   120m   v1.20.1   10.0.0.4      <none>        Ubuntu 18.04.5 LTS   5.4.0-1031-azure   containerd://1.3.3

## *Update!*

I created a Terraform template to deploy a multinode (1 controller + N nodes in VM Scale Set) using Kubeadm+containerd in Azure [here](https://github.com/ams0/CKS/tree/main/kubeadm-containerd-multinode).

