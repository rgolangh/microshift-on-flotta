# Microshift on a Flotta edge device

[](demo.gif)/![](demo.gif)

This effort is to prove microshift can be deployed as a flotta workload.
Flotta workload is essentially a managed, updatable, podman container, controlled
by an k8s operator and follows the podSpec definition.

# The deployment environment
The flotta 'device' is an x86 kubevirt VM running standard Fedora 35 cloud image,
and the flotta-operator deployed on a kind cluster with latest kubevirt upstream installed.

I stored the various yamls I used in a git repo so this all effort can be reproduced.
The VM manifest covers the device requirements for both flotta and microshift parts.

The edge workload declares the microshift workload with the needed volumes and flags.

## Steps to reproduce this POC
- deploy the flotta operator on any cluster, I used kind
- record the flotta service endpoint ip
 ```bash
 IP=$(oc get svc -n flotta flotta-operator-controller-manager -o jsonpath='{.spec.clusterIP}')
 ```
- deploy kubevirt https://kubevirt.io/quickstart_kind/
- create a device VM
  - replace your ssh key in `fedora-kubevirt-vm.yaml`
  - deploy the VM:
    ```bash
    oc create -f fedora-kubevirt-vm.yaml
    ```
-  expose the VM ssh
  ```bash
  oc expose $(oc get pods -l kubevirt.io/vm=flotta-device -o name) --port 22 --name device-ssh
  ```
- ssh into the device
  ```bash
  ssh fedora@$(oc get svc/device-ssh -o jsonpath='{.spec.clusterIP}')
  ```
  - build yggdrasil and flotta device worker
   ```bash
   sudo su -
   cd /root/flotta-device-worker
   make install
   cd /root/yggdrasil
   make install PREFIX=/var/local
   ```
   
   - copy the devices key and cert files from the operator into /etc/pki/consumer/
     ```bash
        # on a machine with access to both the cluster and the vm
        export VM_IP=$(oc get svc device-ssh -o jsonpath={.spec.cluserIP})
        oc get secret -n flotta -l reg-client-ca=true -o jsonpath={.items[1].data.client"\."crt} | base64 -d | ssh fedora@$VM_IP sudo tee /etc/pki/consumer/cert.pem
        oc get secret -n flotta -l reg-client-ca=true -o jsonpath={.items[1].data.client"\."key} | base64 -d | ssh fedora@$VM_IP sudo tee /etc/pki/consumer/key.pem
        oc get secret -n flotta flotta-ca -o jsonpath={.data.ca"\."crt} | base64 -d | ssh fedora@$VM_IP sudo tee /etc/pki/ca-trust/source/anchors/flotta-ca.pem
        ssh fedora@$VM_IP sudo update-ca-trust
        ssh fedora@$VM_IP "echo $(oc get svc -n flotta  flotta-operator-controller-manager -o jsonpath={.spec.clusterIP}) project-flotta.io | sudo tee -a /etc/hosts"
    
     ```
     - log in the device and start the device daemon (yggd)
       ```bash
          sudo su -
          cd /root/yggdrasil
          ./yggd \
              --log-level info \
              --protocol http \
              --path-prefix api/flotta-management/v1 \
              --client-id $(cat /etc/machine-id) \
              --cert-file /etc/pki/consumer/cert.pem \
              --key-file /etc/pki/consumer/key.pem \
              --server project-flotta.io:8043
       ```
       
       - you should see the device registering to the cluser
       ```
       oc get edgedevice
       oc get edsr
       ```
       
       - deploy microshift
       ```
       oc create -f edgeworkload.yaml
       ```
       

## Things that didn't work or deviates from flotta
- sharing directories with selinux policies from non /var/local location fails various components.
  - use /var/lib/kubelet and not /var/local/kubelet
  - use /var/logs and not /var/local/logs otherwise crio fails to create a container log
- used firewalld + nftables. According to the flotta device blueprint it should use nftables directly, no firewalld, so need to check the nft tables isn't messed up.

