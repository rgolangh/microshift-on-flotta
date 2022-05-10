Microshift on a Flotta edge device

This effort is to prove microshift can be deployed as a flotta 'workload'.

The 'device' is an x86 kubevirt VM running standard Fedora 35 cloud image,
flotta-operator deployed on kind cluster with latest kubevirt upstream installed.

I stored the various yamls I used in a git repo so this all effort can be reproduced.
The VM manifest covers the device requirements for both flotta and microshift parts.
<VM.yaml>

The edge workload declares the microshift workload with the needed volumes and flags.
<EDGEWORLOAD.yaml>

Things that didn't work or deviates from flotta
sharing directories with selinux policies from non /var/local location fails various components.
use /var/lib/kubelet and not /var/local/kubelet
use /var/logs and not /var/local/logs otherwise crio fails to create a container log
used firewalld + nftables. According to the flotta device blueprint it should use nftables directly, no firewalld, so need to check the nft tables isn't messed up.

