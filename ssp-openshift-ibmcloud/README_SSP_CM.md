# Deploy Secure Proxy Configuration Manager on OpenShift

## Setup directories and files on Toolkit Container

1. Get pod information

```shell
oc project sterling-tookit

oc get pods

NAME                         READY   STATUS    RESTARTS   AGE
sterling-tookit-59..         1/1     Running   0          73m
```

Export Toolkit Pod 

```shell
export  MY_TOOLKIT_POD=sterling-tookit-59..
```

2. Connect to Pod and setup directories

```shell
oc rsh pod/$MY_TOOLKIT_POD
```

```shell
cd /var/nfs-data/

mkdir -p sspcm/CM_RESOURCES

chmod -R a+rwx sspcm

chown -R 1000:1000 sspcm 

exit
```

## Copy defkeyCert.txt to pod

```shell
$ oc cp defkeyCert.txt $MY_TOOLKIT_POD:/var/nfs-data/sspcm/CM_RESOURCES
```



## Define Security

1. Define our project 

```shell
oc project sterling-ssp
```

2. Change directory and setup permissions on OpenShift

```shell
cd ibm-ssp-cm/ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration

sh createSecurityClusterPrereqs.sh

cd ../../../../..
```

4. Change Rolebinding

```shell
cd ibm-ssp-cm/ibm_cloud_pak/pak_extensions/pre-install/namespaceAdministration

sh createSecurityNamespacePrereqs.sh sterling-ssp

cd ../../../../..

```


## Configure Storage for Secure Proxy Configuration Manager

1. Locate the required information on the default storage volume

```
oc get pv -n openshift-image-registry

NAME       CAPACITY ACCESS MOD  RECLAIM POLICY  STATUS  CLAIM                                              STORAGECLASS    
...                           
pvc-99...  100Gi    RWX           Delete          Bound   openshift-image-registry/image-registry-storage  ibmc-file-gold      
...
```

1. Get the details of the PV

```shell
oc describe pv pvc-99...

...
failure-domain.beta.kubernetes.io/region=us-south
failure-domain.beta.kubernetes.io/zone=dal10
...
Type:   NFS (an NFS mount that lasts the lifetime of a pod)
Server: fsf-xxxxxxx-xx.adn.networklayer.com
Path:   /IBMxxSEVxxxxxxx_xx/data01
...
```

5. Create file my-ssp-cm-pv.yaml, change from previous command:
  
```
cp ssp-cm-pv-nfs.yaml my-ssp-cm-pv-nfs.yaml
```

6. Allocate PV/PVC

```shell
oc create -f my-ssp-cm-pv-nfs.yaml
```

## Configuring Secrets

1. Change file **my-ssp-cm-secrets.yaml** for passphrase

```shell
echo -n "Passw0rd@123" | base64
```

2. Change file **my-ssp-cm-secrets.yaml** for password
 
```shell
echo -n "passw0rd" | base64
```

3. Create secrets

```shell
cp ssp-cm-secrets.yaml my-ssp-cm-secrets.yaml

oc create -f ssp-cm-secrets.yaml
```

### Deploy with Helm

1. Create file my-ssp-cm-override.yaml, and change 

```
cp ssp-cm-override.yaml my-ssp-cm-override.yaml
```

2. Deploy with Helm

```shell
cd ibm-ssp-cm

helm install ssp-cm --namespace sterling-ssp --timeout 120m0s -f ../my-ssp-cm-override.yaml .
```

You can check install using this commands:

```shell
$ oc get pods
NAME              READY   STATUS    RESTARTS   AGE
ssp-cmXXXX   1/1     Running   0          24s

$ oc logs -f ssp-cmXXXXX
```