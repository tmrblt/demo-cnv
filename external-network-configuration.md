### 1. label-nodes
```bash
oc label node worker01 external-network=true
oc label node worker02 external-network=true
```

### 2. backup current NodeNetworkState
```bash
oc get nns worker01 -o yaml > worker01-nns-backup.yaml
oc get nns worker02 -o yaml > worker02-nns-backup.yaml
```

### 3. create node network configuration policy
Check following resources for nncp samples that fits your environment (bonding,static-ip,dhcp,vlan etc)
- https://infohub.delltechnologies.com/en-US/l/red-hat-openshift-4-12-deployment-on-dell-powerflex-4-5/openshift-ingress-api-network-configuration-2/1/
- https://infohub.delltechnologies.com/en-US/l/red-hat-openshift-4-12-deployment-on-dell-powerflex-4-5/steps-to-deploy-the-openshift-cluster-1/
- https://access.redhat.com/documentation/en-us/assisted_installer_for_openshift_container_platform/2024/html-single/installing_openshift_container_platform_with_the_assisted_installer/index#example_of_nmstate_configuration

**create nncp file**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
 name: br1-ens4-policy
spec:
 nodeSelector:
   external-network: "true"
 desiredState:
   interfaces:
     - name: br1
       description: Linux bridge with ens4 as a port
       type: linux-bridge
       state: up
       ipv4:
         dhcp: true
         enabled: true
       bridge:
         options:
           stp:
             enabled: false
         port:
           - name: ens4
```
**create nncp**
```bash
oc create -f br1-ens4-policy.yaml
```
**check node network configuration enactment and other resources**
```bash
oc get nnce
oc get nnce worker01.br1-ens4-policy -o yaml
oc get nncp
oc get nnce
oc get nns worker01 -o yaml
```

### 4. Create Network Attachment Definition
GUI: Networking → NetworkAttachmentDefinitions → Create network attachment definition

Name: br1-network

Description:	Linux bridge on nodes "Worker01" and "Worker02" with ens4 as a port

Network Type:	CNV Linux bridge

Bridge Name:	br1

### 5. Add additional NIC to VM then restart VM
Virtualization → VirtualMachines

### 6. ssh into vm and check network

