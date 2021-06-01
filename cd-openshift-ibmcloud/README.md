# Deploying Sterling Connect:Direct on Openshift using the IBM Cloud

Steps required to install Sterling Connect:Direct on OpenShift on the IBM Cloud

Download the following file from Advantage Passport:

* IBM Sterling B2B Integrator Certified Container V6.1 (PartNumber: CC7W7ML) 


## Clone this repository


1. Login on OpenShift

```shell
git clone https://github.com/ebasso/sterling-b2b-oncloud.git

cd cd-openshift-ibmcloud
```

2. Create a new project on OpenShift for B2Bi

```shell
oc new-project sterling-cd-node1

## Create a new project on OpenShift for Connect:Direct


1. Login on OpenShift

```shell
oc login --token=sha256~... --server=https://...containers.cloud.ibm.com:31234
```

2. Create a new project on OpenShift for B2Bi

```shell
oc new-project sterling-cd-node1
```

## Setup Container Images on Registry


### Upload images to Registry

1. Get and export variable

```shell
oc get route image-registry -n openshift-image-registry

export MY_IMG_REGISTRY=image-registry-openshift-image-registry....us-south.containers.appdomain.cloud
```

2. Change to B2Bi Project and export project

```shell
oc project sterling-cd-node1

export MY_CD_PROJECT=sterling-cd-node1
```

3. Browse to the location where you have downloaded the B2Bi container image.

```shell
unzip IBM_CD_V6.1_UNIX_RHOS.tar.Z.zip

tar -xzvf IBM_CD_V6.1_UNIX_RHOS.tar.Z

tar -xvf 6.1.0.0-IBMConnectDirectforUNIX-Certified-Container-Linux-x86-iFix000.tar
 
```
4. Returno do previous directory and extract ibm-connect-direct-x.x.x.tgz

```shell
cd <CHANGE HERE>/cd-openshift-ibmcloud

tar -xzvf <Downloads_Directory>/ibm-connect-direct-1.1.0.tgz
```

   
5. Login to Registry. Load/tag/push and check.

```shell
docker login -u $(oc whoami) -p $(oc whoami -t) $MY_IMG_REGISTRY

docker load -i cdu6.1_certified_container_6.1.0.0_ifix000.tar

docker tag cdu6.1_certified_container_6.1.0.0:6.1.0.0_ifix000 $MY_IMG_REGISTRY/$MY_CD_PROJECT/cdu6.1_certified_container_6.1.0.0:6.1.0.0_ifix000

docker push $MY_IMG_REGISTRY/$MY_CD_PROJECT/cdu6.1_certified_container_6.1.0.0:6.1.0.0_ifix000
```

5. Check result

```shell
oc get imagestream 
```


## Deploy Sterling Toolkit on OpenShift

### Configure Storage for Tookit

1. Create a new project on OpenShift for Tookit

```shell
oc new-project sterling-cd-toolkit
```

2. Define Permissions

```shell
oc adm policy add-scc-to-user anyuid -z default -n sterling-cd-toolkit
```
3. Locate the required information on the default storage volume

```shell
oc get pv -n openshift-image-registry

NAME       CAPACITY ACCESS MOD  RECLAIM POLICY  STATUS  CLAIM                                              STORAGECLASS    
...                               
pvc-99...  100Gi    RWX           Delete          Bound   openshift-image-registry/image-registry-storage  ibmc-file-gold      
...
```
4. Get the details of the PV

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

1. Create file my-toolkit-pv-pvc.yaml, change from previous command.
  
```
cp toolkit-pv-pvc.yaml my-toolkit-pv-pvc.yaml
```

6. Allocate PV/PVC

```shell
oc create -f my-toolkit-pv-pvc.yaml
```

### Deploy Toolkit Container

1. Deploy toolkit

```shell
oc create -f toolkit-deploy.yaml
```

### Copy files to Toolkit Container

1. Get pod information

```shell
oc project sterling-cd-toolkit

oc get pods

NAME                         READY   STATUS    RESTARTS   AGE
sterling-cd-toolkit-59..   1/1     Running   0          73m
```

Export Toolkit Pod 

```shell
export  MY_TOOLKIT_POD=sterling-cd-toolkit-59..
```

2. Connect to Pod and setup directories

```shell
oc rsh pod/$MY_TOOLKIT_POD
```

```shell
cd /var/nfs-data/

mkdir -p CDFILES

chmod -R 777 CDFILES

exit
```

3. Create TLS certificates and copy the same to CDFILES folder 

Certificate files are required when Secure Plus is being configured. The user must provide the key certificate and trusted certificate files in PEM format.



```shell
openssl req -x509 -sha256 -days 3650 -newkey rsa:2048 -new -nodes -keyout cdkey.pem -out cdcert.crt
```
provided

```
Generating a 2048 bit RSA private key
................................+++
writing new private key to 'cdkey.pem'
...
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:US
State or Province Name (full name) []:California
Locality Name (eg, city) []:
Organization Name (eg, company) []:Company
Organizational Unit Name (eg, section) []:IT
Common Name (eg, fully qualified host name) []:cdnode1.company.com
Email Address []:cdadmin@company.com
```


1. Copy files to pod

```shell
cat cdkey.pem cdcert.crt > cdcert.pem

oc cp cdcert.pem $MY_TOOLKIT_POD:/var/nfs-data/connectdirect/CDFILES
```

5. Validate

```shell
oc rsh $MY_TOOLKIT_POD ls -l /var/nfs-data/connectdirect/CDFILES
```


## Deploy Sterling Connect:Direct on OpenShift

### Pre-install

1. Define our project 

```shell
oc project sterling-cd-node1
```

2. Change directory and setup permissions on OpenShift

```shell
cd ibm-connect-direct/ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration

./createSecurityClusterPrereqs.sh

cd ../../../../..
```

4. Change Rolebinding

```shell
cd ibm-b2bi-prod/ibm_cloud_pak/pak_extensions/pre-install/namespaceAdministration

./createSecurityNamespacePrereqs.sh sterling-cd-node1

cd ../../../../..
```

### Configure Storage for Connect:Direct

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

5. Create file my-b2bi-pv.yaml, change from previous command:
  
```
cp cd-pv-nfs.yaml my-cd-pv-nfs.yaml
```

6. Allocate PV/PVC

```shell
oc create -f my-cd-pv-nfs.yaml
```

### Configuring passphrase for Connect:Direct

1. Change file **cd-secrets.yaml**

```shell
echo -n "passw0rd" | base64
```

2. Create secrets

```shell
oc create -f cd-secrets.yaml
```


### Deploy with Helm

1. Create file my-cd-override.yaml, and change 

```
cp cd-override.yaml my-cd-override.yaml
```

2. Deploy with Helm

```shell
cd ibm-connect-direct

helm install sterling-cd-node1 --namespace sterling-cd-node1 --timeout 120m0s -f ../my-cd-override.yaml .
```

You can check install using this commands:

```shell
$ oc get pods
NAME                                    READY   STATUS    RESTARTS   AGE
sterling-cd-node-b2bi-db-setup-bltg5   1/1     Running   0          24s

$ oc logs -f sterling-b2bi-app-b2bi-db-setup-bltg5
```