# Deploying Sterling External Authentication Server on Openshift using the IBM Cloud

Steps required to install Sterling External Authentication Server (SEAS) on OpenShift on the IBM Cloud

Download the following file from Passport Advantage:

* IBM Sterling External Authentication Server V 6.0.2 for RHOCP English



## Clone this repository


1. Login on OpenShift

```shell
git clone https://github.com/ebasso/sterling-b2b-oncloud.git

cd sterling-b2b-oncloud/seas-openshift-ibmcloud
```

## Create a new project on OpenShift for SEAS

1. Login on OpenShift

```shell
oc login --token=sha256~... --server=https://...containers.cloud.ibm.com:31234
```

2. Create a new project on OpenShift for SEAS

```shell
oc new-project sterling-seas
```

## Setup Container Images on Registry

[Setup RH OpenShift Image Registry](../third-party/setup-roks-image-registry.md)

### Upload images to Registry

1. Get and export variable

```shell
oc get route image-registry -n openshift-image-registry

export MY_IMG_REGISTRY=image-registry-openshift-image-registry....us-south.containers.appdomain.cloud
```

2. Change to SEAS Project and export project

```shell
oc project sterling-seas

export MY_SEAS_PROJECT=sterling-seas
```

3. Browse to the location where you have downloaded the SEAS container image.

```shell
unzip ../SEAS.V6020.Docker.tar.zip

tar -xvf SEAS.V6020.Docker.tar

tar -xvf seas_6020_docker.tar

tar -xvf seas_6020_helmchart.tar

rm SEAS.V6020.Docker.tar
rm seas_6020_docker.tar
rm seas_6020_helmchart.tar
```
   
4. Login to Registry. Load/tag/push and check.

```shell
docker login -u $(oc whoami) -p $(oc whoami -t) $MY_IMG_REGISTRY

docker load -i seas_docker_image_6020.tar

docker tag seas-docker-image:6.0.2.0     $MY_IMG_REGISTRY/$MY_SEAS_PROJECT/seas-docker-image:6.0.2.0

docker push $MY_IMG_REGISTRY/$MY_SEAS_PROJECT/seas-docker-image:6.0.2.0

```

5. Check result

```shell
oc get imagestream 
```

6. Returno do previous directory and extract the helm charts

```shell
cd <CHANGE HERE>/seas-openshift-ibmcloud

tar -xzvf <Downloads_Directory>/ibm-seas-1.1.0.tgz
```


# Deploy Sterling Toolkit on OpenShift

Follow the article [Deploy Sterling Toolkit on OpenShift](../sterling-toolkit/) to setup toolkit


# Deploy SEAS on OpenShift


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

mkdir -p seas

chmod -R a+rwx seas

chown -R 1000:1000 seas 

exit
```


## Define Security

1. Define our project 

```shell
oc project sterling-seas
```

2. Change directory and setup permissions on OpenShift

```shell
cd ibm-seas/ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration

sh createSecurityClusterPrereqs.sh

cd ../../../../..
```

4. Change Rolebinding

```shell
cd ibm-seas/ibm_cloud_pak/pak_extensions/pre-install/namespaceAdministration

sh createSecurityNamespacePrereqs.sh sterling-seas

cd ../../../../..

```


## Configure Storage for SEAS

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

5. Create file my-seas-pv.yaml, change from previous command:
  
```
cp seas-pv-nfs.yaml my-seas-pv-nfs.yaml
```

6. Allocate PV/PVC

```shell
oc create -f my-seas-pv-nfs.yaml
```

## Configuring Secrets

1. Change file **my-seas-secrets.yaml** for passphrase

```shell
echo -n "Passw0rd@123" | base64
```

2. Change file **my-seas-secrets.yaml** for password
 
```shell
echo -n "passw0rd" | base64
```

3. Create secrets

```shell
cp seas-secrets.yaml my-seas-secrets.yaml

oc create -f my-seas-secrets.yaml
```

## Deploy with Helm

1. Create file my-seas-override.yaml, and change 

```
cp seas-override.yaml my-seas-override.yaml
```

2. Deploy with Helm

```shell
cd ibm-seas

helm install seas --namespace sterling-seas --timeout 120m0s -f ../my-seas-override.yaml .
```

You can check install using this commands:

```shell
$ oc get pods
NAME              READY   STATUS    RESTARTS   AGE
seasXXXXX   1/1     Running   0          24s

$ oc logs -f seasXXXXX
```

```shell
$ oc get svc
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)           AGE
seasXXXXX       LoadBalancer   172.21.81.131   52.xxx.xxx.16   63366:31012/TCP   5m46s
```
