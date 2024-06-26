# How to add additional network to pod using NMstate Operator and Multus

Tested on:
* Red Hat Openshift version 4.15.15
* Kubernetes NMState Operator 4.15.0-202406052127

VMware requarements (if your OCP cluster was installed on top of VMware vSphere):
* Promiscuous mode - Override
* MAC address changes - Override
* Forged transmits - Override

You can find these settings in the path - Virtual Switches -> VMs Network -> Edit Settings -> Security.

[Plugin's description link](https://docs.openshift.com/container-platform/4.15/networking/multiple_networks/understanding-multiple-networks.html#additional-networks-provided)

## Bridge CNI plugin and localnet topology

Bridge: Configure a bridge-based additional network to allow pods on the same host to communicate with each other and the host.

[Topology description (localnet or layer2)](https://docs.openshift.com/container-platform/4.15/networking/multiple_networks/configuring-additional-network.html#configuration-ovnk-additional-networks_configuring-additional-network)

1) Create NodeNetworkConfigurationPolicy (NNCP)
```   
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ovs-br224 # <- put your nncp name
spec:
  desiredState:
    interfaces:
      - bridge:
          allow-extra-patch-ports: true
          options:
            stp:
              enabled: false
          port:
            - name: enp6s21 # <- put your if name
        name: ovs-br224 # <- put your bridge name
        state: up
        type: ovs-bridge
    nodeSelector:
      node-role.kubernetes.io/control-plane: '' # <- select nodes    
```

2) Create local-mapping configuration
```
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: localnet-mapping
spec:
  desiredState:
    ovn:
      bridge-mappings:
        - bridge: ovs-br224 # <- put your bridge name
          localnet: localnet-192.168.0 # <- put your network name (will be used on the next step)
          state: present
```

3) Create Network Attachment Definitions (NAD)
```
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: localnet-192 # <- put your NAD name
  namespace: igor # <- put your namespace
spec:
  config: '{
    "name": "localnet-192.168.0",
    "type": "ovn-k8s-cni-overlay",
    "cniVersion": "0.3.1",
    "topology": "localnet",
    "netAttachDefName": "igor/localnet-192" }'
```

`"name": "localnet-192.168.0"` - localnet from the previous step

`"type": "ovn-k8s-cni-overlay"` - the name of the CNI plug-in to be configured

`"topology": "localnet"` - the topological configuration for the network.

`"netAttachDefName": "igor/localnet-192"` - the value of the namespace and name fields 

More detailes you can find on documentation:

https://docs.openshift.com/container-platform/4.15/virt/vm_networking/virt-connecting-vm-to-ovn-secondary-network.html#creating-ovn-nad_virt-connecting-vm-to-ovn-secondary-network

4) Connect your pod (or Virtual Machine) to NAD from the previous step

Edit Deployment:
```
template:
    metadata:
      name: nginx-example
      creationTimestamp: null
      labels:
        name: nginx-example
      annotations:
        k8s.v1.cni.cncf.io/networks: '[ { "name": "localnet-192", "ips": [ "192.168.1.125/24" ] } ]' # <- add your NAD under the 'spec' section
...
```

5) Check you pod status
```
 annotations:
    k8s.ovn.org/pod-networks: '{"default":{"ip_addresses":["10.128.1.165/23"],"mac_address":"0a:58:0a:80:01:a5","gateway_ips":["10.128.0.1"],"routes":[{"dest":"10.128.0.0/14","nextHop":"10.128.0.1"},{"dest":"172.30.0.0/16","nextHop":"10.128.0.1"},{"dest":"100.64.0.0/16","nextHop":"10.128.0.1"}],"ip_address":"10.128.1.165/23","gateway_ip":"10.128.0.1"},"igor/localnet-192":{"ip_addresses":["192.168.1.125/24"],"mac_address":"0a:58:c0:a8:01:7d","ip_address":"192.168.1.125/24"}}'
    k8s.v1.cni.cncf.io/network-status: |-
      [{
          "name": "ovn-kubernetes",
          "interface": "eth0",
          "ips": [
              "10.128.1.165"
          ],
          "mac": "0a:58:0a:80:01:a5",
          "default": true,
          "dns": {}
      },{
          "name": "igor/localnet-192",
          "interface": "net1",
          "ips": [
              "192.168.1.125"
          ],
          "mac": "0a:58:c0:a8:01:7d",
          "dns": {}
      }]
```

## MACVLAN CNI plugin and localnet topology

1) Create Network Attachment Definitions (NAD)
   
```
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
name: macvlan-ens224
namespace: namespace
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "macvlan-net224",
    "type": "macvlan",
    "master": "ens224",
    "mode": "bridge",
    "ipam": {
        "type": "static",
        "routes": [ {  "dst": "192.168.32.0/24", "gateway": "192.168.32.1" } ],
        "gateway": "192.168.32.1",
        "capabilities": { "mac": true } } }'
```

2) Connect your pod (or Virtual Machine) to NAD as described above.
