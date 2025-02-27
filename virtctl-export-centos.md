## instructions to create container and qcow2 disk image for OracleDB-23c VM templating on centos8-streams


### prepare server
```bash
#install packages
#install aria2
sudo dnf install -y wget
wget https://rpmfind.net/linux/epel/8/Everything/x86_64/Packages/a/aria2-1.35.0-2.el8.x86_64.rpm
sudo yum localinstall aria2-1.35.0-2.el8.x86_64.rpm

#install oc and virtctl
wget https://downloads-openshift-console.apps.ocx.sandbox1719.opentlc.com/amd64/linux/oc.tar --no-check-certificate

wget https://hyperconverged-cluster-cli-download-openshift-cnv.apps.ocx.sandbox1719.opentlc.com/amd64/linux/virtctl.tar.gz --no-check-certificate

tar -xvf oc.tar
tar -zxvf virtctl.tar.gz 
sudo mv oc virtctl /usr/local/bin

#install podman and libguestfs-tools
sudo yum install podman libguestfs-tools -y


#install web-server
sudo dnf install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd

sudo dnf install firewalld -y
sudo systemctl enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

chmod 777 -R /var/www/html
setenforce 0
```

### Download disk image
```bash
#get kubeconfig and oc login
export KUBECONFIG=~/kubeconfig
oc whoami

#create vmexport resource for existing VM
oc project demo-cnv
oc get vms
virtctl vmexport create oracledb23c-golden-export --vm oracledb23c-golden


#get external URL
oc get VirtualMachineExport/oracledb23c-golden-export -o yaml
#get token tokenSecretRef
- secret-oracledb23c-export

#download disk image
aria2c --check-certificate=false --header="x-kubevirt-export-token:C3CcuB2RJWOSWM15QAs1"  --out="oracledb23c-exported-disk.img" "https://virt-exportproxy-openshift-cnv.apps.ocx.sandbox1719.opentlc.com/api/export.kubevirt.io/v1alpha1/namespaces/demo-cnv/virtualmachineexports/oracledb23c-golden-export/volumes/oracledb23c-golden/disk.img" 

```

### create qcow2 disk image
```bash
#convert image
qemu-img convert -q -f raw -O qcow2 oracledb23c-exported-disk.img oracledb23c.qcow2

#optional to save space
virt-sparsify oracledb23c.qcow2 oracledb23c-sparsed.qcow2

#optional to create qcow2.gz file
pigz -k oracledb23c-sparsed.qcow2

#copy disk image
sudo cp oracledb23c-sparsed.qcow2 /var/www/html/.
```
```bash
virtctl expose virtualmachineinstance demo-utils-2 --name http --port 27017 --target-port 80

oc get service

curl 172.30.161.152:27017
```

### create container disk image

```bash
#create Dockerfile 
cat << EOF > Dockerfile
FROM scratch
ADD oracledb23c.qcow2 /disk/
EOF

#optional if free-space needed
```bash
export TMPDIR=/home/admin/tmpdir
echo $TMPDIR

#create container image
podman build -t oracledb23c:1.0 --file Dockerfile 

#push ccontainer image to quay registry
podman login 172.30.134.8:80 --tls-verify=false
podman tag localhost/oracledb23c:1.0 172.30.134.8:80/tamer/oracledb23c:1.0
podman push 172.30.134.8:80/tamer/oracledb23c:1.0 --tls-verify=false
```

### trust registry certificate
```bash
# download-certificate of quay registry from browser

#create-cm
cat << EOF > cm-registry-certificate.yml
apiVersion: v1
data:
  ca-bundle.crt: |
    -----BEGIN CERTIFICATE-----
    MIIDezCCAmOgAwIBAgIIKv4GxIPCpdswDQYJKoZIhvcNAQELBQAwJjEkMCIGA1UE
    AwwbaW5ncmVzcy1vcGVyYXRvckAxNzA2MDI4MTY1MB4XDTI0MDEyMzE2NDI0NFoX
    DTI2MDEyMjE2NDI0NVowLTErMCkGA1UEAwwiKi5hcHBzLm9jeC5zYW5kYm94MTcx
    OS5vcGVudGxjLmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANNl
    jAA1axsdEvOE+1kb1iVP827P0dc11i/EOhCr096eVVGZNDQUVf1zmh5C9OYkdCjp
    chT2WoYVkRTRb5TmSLBirTPJI6lKFYf9zlzQws+0/geuzdguE2JHc4t4Zej8tOX9
    w6mUbKg7qXcN6isee9ZP6TEYmuKpcZK4dh1VJHCJw1iJi7AcemMa9fcZZvJ/yqkm
    0HzlX+2g6R6CmMTVWc0loetjIu2q6defK/8XtjZpqeoJk8teUjLCyMmu7wxBHmzc
    tQgsAbmLWCTLIsNAOTxuC3h/3knzsOBQ0xg4AqTFNc/k69+hGGp+oxiujT80jZxs
    gNTpdy2CXejufZI4Iu0CAwEAAaOBpTCBojAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0l
    BAwwCgYIKwYBBQUHAwEwDAYDVR0TAQH/BAIwADAdBgNVHQ4EFgQU+u4hl4deiRb1
    NFEExUeU79RF8xAwHwYDVR0jBBgwFoAUqB2vw+OxgN1vJv407f5sfYuQ5bcwLQYD
    VR0RBCYwJIIiKi5hcHBzLm9jeC5zYW5kYm94MTcxOS5vcGVudGxjLmNvbTANBgkq
    hkiG9w0BAQsFAAOCAQEAfF7wXat1V3FjdCv4YGh7cRbw38zDyE0rUdK1PuxCtCtL
    6131H8XyyzivWCSNR9gvIy2GSocDg1l5kU71iJuDVgsWqFIGi0+aye62YbkQI1ZP
    t3uAPtPHeUWReiEeP6QvrDcUregBuwgUekWZ9TXgO8iMYjZpT4YqwVh2+4YyRdiz
    FNalcZDjNw135wMAWU/mgE65xLir9kx/vXABa2T5ViZNp7shBN3kZj0xMmTVnfaV
    zrF+wF7tIg7twBMAn9a0fdsBKf60pDe9rm4ZFJKXR5GRUPjC1kOCHY7FAYPCpyf1
    b+t3kU7j7tdQvlb3aWDkck2YLodz5CfwRf75dux6TQ==
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MIIDDDCCAfSgAwIBAgIBATANBgkqhkiG9w0BAQsFADAmMSQwIgYDVQQDDBtpbmdy
    ZXNzLW9wZXJhdG9yQDE3MDYwMjgxNjUwHhcNMjQwMTIzMTY0MjQ0WhcNMjYwMTIy
    MTY0MjQ1WjAmMSQwIgYDVQQDDBtpbmdyZXNzLW9wZXJhdG9yQDE3MDYwMjgxNjUw
    ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDn3NGWJfDhksdLlViF+V1+
    gcN1cLsOFCFdvUy7x5IdLAK+VHoWCruy7CXNEzwckvnnRWJrHVU/frPv3Jhat1X2
    SuT0iU9BirfFG8KupyEs3ny0J8OeOzYlRWZSkeqKXL1qg4dj+TxiHojnQ0c5+ioz
    ZnZU4lNNSB2HMBJNQ7hsYAlX5sG0VawVFtSv36IiAZRELyZj9YBoxySsw7KTKAfG
    o6EwFUKCiF702yo5e+9Ti0K0LB5uwKfMmPtD59gVq7mNKiF43K4sj+n1CgKDUhOQ
    qzQ3xQxRLUULFnXn0awXM7ro57ks2celAhD7QgFG3ql5KmUlPCBCoNYLQA8bz1kP
    AgMBAAGjRTBDMA4GA1UdDwEB/wQEAwICpDASBgNVHRMBAf8ECDAGAQH/AgEAMB0G
    A1UdDgQWBBSoHa/D47GA3W8m/jTt/mx9i5DltzANBgkqhkiG9w0BAQsFAAOCAQEA
    tYXW6e06fExWIDVtKsqTU3Av6MAFNxkFH2I+D3vGTyU4iDdNrzQj9WzL9YwF4D+g
    EZvMRBbRWATcsBNai+vAygKeaFjwtf1sK2r4rH1vB8+EzJ6N/GIUz6uI/N7mXSek
    QldthhfNZ9+ub6Lhv7RawxJ4pcA132Kgd9dwPSY4r3rcvxpj/UCbi2aTHizQrpuZ
    2xjJswkwZ/Usq+d0EpK333nL9F05FdaPd5jhU9aYiBfL3PjH9hrF8KdWLqn0MZCT
    wO9C0ghZr5St2x82jbFy6rUu3JWjWmJ8HiutvvshDoDG3iJq+ACh1voC8J3TMz0b
    fSx1PWmnKA3m5mBNsRHqWQ==
    -----END CERTIFICATE-----
kind: ConfigMap
metadata:
  name: cm-registry-certificate 
  namespace: openshift-config
EOF

oc apply -f cm-registry-certificate.yml

oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"registry-certificate"}}}'

# optional to add insecure registries
oc patch cdi cdi-kubevirt-hyperconverged cdi --patch '{"spec": {"config": {"insecureRegistries": ["my-private-registry-host:5000"]}}}' --type merge
```
