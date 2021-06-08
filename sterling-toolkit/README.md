# Deploy Sterling Toolkit on OpenShift

Toolkit helps to configure files and directories for Sterling Solutions

## Configure Storage for Tookit

1. Create a new project on OpenShift for Tookit

```shell
oc new-project sterling-toolkit
```

2. Define Permissions

```shell
oc adm policy add-scc-to-user anyuid -z default -n sterling-toolkit
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

## Deploy Toolkit Container

1. Deploy toolkit

```shell
oc create -f toolkit-deploy.yaml
```

# Setup Sterling Solutions in Toolkit Container

## Copy files to Toolkit Container

1. Get pod information

```shell
oc project sterling-toolkit

oc get pods

NAME                         READY   STATUS    RESTARTS   AGE
sterling-toolkit-59..   1/1     Running   0          73m
```

Export Toolkit Pod 

```shell
export  MY_TOOLKIT_POD=sterling-toolkit-59..
```

2. Connect to Pod and setup directories

```shell
oc rsh pod/$MY_TOOLKIT_POD
```

## Setup toolkit for Sterling B2Bi

1. Create directories and b2biuser

```shell
cd /var/nfs-data/

mkdir resources logs documents

useradd -u 1010 b2biuser
chown -R b2biuser:b2biuser logs 
chown -R b2biuser:b2biuser resources 
chown -R b2biuser:b2biuser documents

exit
```

2. Copy files to pod

```shell
oc cp db2jcc4.jar $MY_TOOLKIT_POD:/var/nfs-data/resources/
oc cp db2jcc_license_cu.jar $MY_TOOLKIT_POD:/var/nfs-data/resources/
oc cp local_policy.jar $MY_TOOLKIT_POD:/var/nfs-data/resources/
```

3. Validate

```shell
oc rsh $MY_TOOLKIT_POD ls -l /var/nfs-data/resources/
```

## Setup toolkit for Sterling Connect:Direct

1. Create directories and set permissions

```shell
cd /var/nfs-data/

mkdir -p connectdirect/CDFILES

chmod -R 777 connectdirect

exit
```

1. Create TLS certificates and copy the same to CDFILES folder 

Certificate files are required when Secure Plus is being configured. The user must provide the key certificate and trusted certificate files in PEM format.



```shell
openssl req -x509 -sha256 -days 3650 -newkey rsa:2048 -new -nodes -keyout sspkey.pem -out sspcert.crt -subj '/C=US/ST=California/L=/O=Company/OU=IT/CN=cdnode1.company.com/emailAddress=cdadmin@company.com'
```
provided

```
Generating a 2048 bit RSA private key
........................+++
...............................................+++
writing new private key to 'cdkey.pem'
-----
No value provided for Subject Attribute L, skipped
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

## Setup toolkit for Sterling Secure Proxy - Configuration Manager

1. Create directories and user with id 1000

```shell
cd /var/nfs-data/

mkdir -p sspcm/CM_RESOURCES

chmod -R a+rwx sspcm

chown -R 1000:1000 sspcm 

exit
```



## Setup toolkit for Sterling Secure Proxy - Engine

1. Create directories and user with id 1000

```shell
cd /var/nfs-data/

mkdir -p sspengine/ENG_RESOURCES

chmod -R a+rwx sspengine

chown -R 1000:1000 sspengine

exit
```


## Setup toolkit for Sterling Secure Proxy - Perimeter Server

1. Create directories and user with id 1000

```shell
cd /var/nfs-data/

mkdir sspps

chmod -R a+rwx sspps

chown -R 1000:1000 sspps

exit
```

Return to Sterling Solutions