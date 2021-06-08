# Deploying IBM Control Center on Openshift using the IBM Cloud

Steps required to install IBM Control Center on OpenShift on the IBM Cloud

Download the following file from Fix Central:

* 6.2.0.0-IBMControlCenterforLinux64Container-iFix04.zip



## Clone this repository


1. Login on OpenShift

```shell
git clone https://github.com/ebasso/sterling-b2b-oncloud.git

cd sterling-b2b-oncloud/scc-openshift-ibmcloud
```

## Create a new project on OpenShift for scc

1. Login on OpenShift

```shell
oc login --token=sha256~... --server=https://...containers.cloud.ibm.com:31234
```

2. Create a new project on OpenShift for scc

```shell
oc new-project sterling-scc
```

## Setup Container Images on Registry

[Setup RH OpenShift Image Registry](../third-party/setup-roks-image-registry.md)

### Upload images to Registry

1. Get and export variable

```shell
oc get route image-registry -n openshift-image-registry

export MY_IMG_REGISTRY=image-registry-openshift-image-registry....us-south.containers.appdomain.cloud
```

2. Change to scc Project and export project

```shell
oc project sterling-scc

export MY_SCC_PROJECT=sterling-scc
```

3. Browse to the location where you have downloaded the scc container image.

```shell
unzip 6.2.0.0-IBMControlCenterforLinux64Container-iFix04.zip
```
   
4. Login to Registry. Load/tag/push and check.

```shell
docker login -u $(oc whoami) -p $(oc whoami -t) $MY_IMG_REGISTRY

docker load -i ibmscc_6.2.tar

docker tag scc-docker-image:6.0.2.0     $MY_IMG_REGISTRY/$MY_SCC_PROJECT/scc-docker-image:6.0.2.0

docker push $MY_IMG_REGISTRY/$MY_SCC_PROJECT/scc-docker-image:6.0.2.0

```

5. Check result

```shell
oc get imagestream 
```

6. Returno do previous directory and extract the helm charts

```shell
cd <CHANGE HERE>/scc-openshift-ibmcloud

tar -xzvf <Downloads_Directory>/ibm-scc-helmchart-1.0.0.tgz
```

# Deploy DB2 for SCC on OpenShift

Follow the article [Deploy DB2 for SCC on OpenShift](README_DB2.md) to setup toolkit


# Deploy Sterling Toolkit on OpenShift

Follow the article [Deploy Sterling Toolkit on OpenShift](../sterling-toolkit/) to setup toolkit


# Deploy scc on OpenShift


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

mkdir -p scc

chmod -R a+rwx scc

chown -R 1001:1001 scc 

exit
```


## Define Security

1. Define our project 

```shell
oc project sterling-scc
```

2. Change directory and setup permissions on OpenShift

```shell
cd ibm-scc/ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration

sh createSecurityClusterPrereqs.sh

cd ../../../../..
```

4. Change Rolebinding

```shell
cd ibm-scc/ibm_cloud_pak/pak_extensions/pre-install/namespaceAdministration

sh createSecurityNamespacePrereqs.sh sterling-scc

cd ../../../../..

```


## Configure Storage for scc

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

5. Create file my-scc-pv.yaml, change from previous command:
  
```
cp scc-pv-nfs.yaml my-scc-pv-nfs.yaml
```

6. Allocate PV/PVC

```shell
oc create -f my-scc-pv-nfs.yaml
```

## Configuring Secrets

1. Change file **my-scc-secrets.yaml** for ccDBPassword and cognosDBPassword

```shell
echo -n "db2pw123" | base64
```

2. Change file **my-scc-secrets.yaml** for adminPassword
 
```shell
echo -n "passw0rd" | base64
```

3. Create secrets

```shell
cp scc-secrets.yaml my-scc-secrets.yaml

oc create -f my-scc-secrets.yaml
```

## Deploy with Helm

1. Create file my-scc-override.yaml, and change 

```
cp scc-override.yaml my-scc-override.yaml
```

2. Deploy with Helm

```shell
cd ibm-scc

helm install scc --namespace sterling-scc --timeout 120m0s -f ../my-scc-override.yaml .
```

You can check install using this commands:

```shell
$ oc get pods
NAME              READY   STATUS    RESTARTS   AGE
sccXXXXX   1/1     Running   0          24s

$ oc logs -f sccXXXXX
```

```shell
$ oc get svc
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)           AGE
sccXXXXX       LoadBalancer   172.21.81.131   52.xxx.xxx.16   63366:31012/TCP   5m46s
```
