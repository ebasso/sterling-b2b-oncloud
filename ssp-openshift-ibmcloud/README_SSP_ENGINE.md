# Deploy Secure Proxy Engine on OpenShift

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

mkdir -p sspengine/ENG_RESOURCES

chmod -R a+rwx sspengine

chown -R 1000:1000 sspengine 

exit
```


## Define Security

1. Define our project 

```shell
oc project sterling-ssp
```

2. Change directory and setup permissions on OpenShift

```shell
cd ibm-ssp-engine/ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration

sh createSecurityClusterPrereqs.sh

cd ../../../../..
```

4. Change Rolebinding

```shell
cd ibm-ssp-engine/ibm_cloud_pak/pak_extensions/pre-install/namespaceAdministration

sh createSecurityNamespacePrereqs.sh sterling-ssp

cd ../../../../..

```


## Configure Storage for Secure Proxy Engine

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

5. Create file my-ssp-engine-pv.yaml, change from previous command:
  
```
cp ssp-engine-pv-nfs.yaml my-ssp-engine-pv-nfs.yaml
```

6. Allocate PV/PVC

```shell
oc create -f my-ssp-engine-pv-nfs.yaml
```

## Configuring Secrets

1. Change file **my-ssp-engine-secrets.yaml** for passphrase

```shell
echo -n "Passw0rd@123" | base64
```

2. Change file **my-ssp-engine-secrets.yaml** for password
 
```shell
echo -n "passw0rd" | base64
```

3. Create secrets

```shell
cp ssp-engine-secrets.yaml my-ssp-engine-secrets.yaml

oc create -f my-ssp-engine-secrets.yaml
```

## Deploy with Helm

1. Create file my-ssp-engine-override.yaml, and change 

```
cp ssp-engine-override.yaml my-ssp-engine-override.yaml
```

2. Deploy with Helm

```shell
cd ibm-ssp-engine

helm install ssp-engine --namespace sterling-ssp --timeout 120m0s -f ../my-ssp-engine-override.yaml .
```

You can check install using this commands:

```shell
$ oc get pods
NAME              READY   STATUS    RESTARTS   AGE
ssp-engineXXXXX   1/1     Running   0          24s

$ oc logs -f ssp-engineXXXXX
```

```shell
$ oc get svc
NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)           AGE
ssp-engine-ibm-ssp-engine   LoadBalancer   172.21.81.136   52.xxx.xxx.12   63366:31012/TCP   5m46s
```

## Copy defkeyCert.txt to local

```shell
$ oc cp ssp-engineXXXXX:/spinstall/IBM/SP/defkeyCert.txt defkeyCert.txt
```
