# Deploying Sterling B2Bi on Openshift using the IBM Cloud

Steps required to install Sterling B2Bi on OpenShift on the IBM Cloud

Download the following file from Advantage Passport:

* IBM Sterling B2B Integrator Certified Container V6.1 (PartNumber: CC7W7ML) **or**
* IBM Sterling File Gateway V6.1.0.0 Certified Container (PartNumber: CC7W8ML)

Perform the correct Sizing of your environment, I had problems installing it due to lack of resources. It worked when my works had:

* vCPUs: 8
* Memory: 32G

## Create a new project on OpenShift for B2Bi

1. Create a new project on OpenShift for B2Bi

```shell
oc new-project sterling-b2bi-app
```

## Deploy DB2 on OpenShift

### Deploy Container

1. Create a new project on OpenShift for DB2 

```shell
oc new-project sterling-b2bi-db2
```

2. define service and access accounts.

```shell
oc create serviceaccount sterling-b2bi-db2-sa

oc adm policy add-scc-to-user privileged -n sterling-b2bi-db2 -z sterling-b2bi-db2-sa
```
3. Perform the deploy

```shell
oc create -f db2-deploy.yaml
```

4. Check for Running status .

```shell
oc get pods

  NAME    READY   STATUS    RESTARTS   AGE
  db2-0   1/1     Running   0          31m
```

Wait until you see this line as the last entry in the log. Ctrl + C to stop the logs.

```shell
oc logs -f db2-0
  
  ...
  /database/config/db2inst1/sqllib/ctrl/db2strst.lck
```

### Create Database

1. Connect to the pod.

```shell
oc rsh pod/db2-0
```

2. Creating the database

```shell
su - db2inst1

cat <<EOF >> create_b2bi_db.sql
CREATE DATABASE B2BIDB AUTOMATIC STORAGE YES USING CODESET UTF-8 TERRITORY DEFAULT COLLATE USING SYSTEM PAGESIZE 32768;
CONNECT TO B2BIDB;
UPDATE DATABASE CONFIG FOR B2BIDB USING LOGFILSIZ 65536;
UPDATE DATABASE CONFIG FOR B2BIDB USING LOGPRIMARY 40;
UPDATE DATABASE CONFIG FOR B2BIDB USING NUM_LOG_SPAN 32;
UPDATE DATABASE CONFIG FOR B2BIDB USING AUTO_MAINT ON;
UPDATE DATABASE CONFIG FOR B2BIDB USING AUTO_TBL_MAINT ON;
UPDATE DATABASE CONFIG FOR B2BIDB USING AUTO_RUNSTATS ON;
UPDATE DATABASE CONFIG FOR B2BIDB USING AUTO_REORG ON;
UPDATE DATABASE CONFIG FOR B2BIDB USING AUTO_DB_BACKUP ON;
CREATE USER TEMPORARY TABLESPACE B2BUSERTEMP PAGESIZE 32K BUFFERPOOL IBMDEFAULTBP;
CREATE BUFFERPOOL B2BIDB_04KBP IMMEDIATE SIZE 1000 PAGESIZE 4K;
CREATE REGULAR TABLESPACE TS_REG04_B2BIDB PAGESIZE 4K BUFFERPOOL B2BIDB_04KBP;
CREATE BUFFERPOOL B2BIDB_08KBP IMMEDIATE SIZE 1000 PAGESIZE 8K;
CREATE REGULAR TABLESPACE TS_REG08_B2BIDB PAGESIZE 8K BUFFERPOOL B2BIDB_08KBP;
CREATE BUFFERPOOL B2BIDB_16KBP IMMEDIATE SIZE 1000 PAGESIZE 16K;
CREATE REGULAR TABLESPACE TS_REG16_B2BIDB PAGESIZE 16K BUFFERPOOL B2BIDB_16KBP;
CONNECT RESET;
EOF

db2 -stvf create_b2bi_db.sql

db2 list database directory

exit

exit
```

3. Test db2 connectivity from the outside. 

Your can test connectivity from the outside using DBEaver or another SQL Tool. Identify the LoadBalancer IP address (port 50000). EXTERNAL-IP parameter

```shell
oc get svc

  NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP        PORT(S)           AGE
  sterling-b2bi-db2-svc-lb  LoadBalancer   172.xx.xx.250   169.xx.xx.83   50000:30707/TCP   43m
```

4. Copy JDBC Files from Container

```shell
oc cp db2-0:/opt/ibm/db2/V11.5/java/db2jcc4.jar db2jcc4.jar
oc cp db2-0:/opt/ibm/db2/V11.5/java/db2jcc_license_cu.jar db2jcc_license_cu.jar
oc cp db2-0:/opt/ibm/db2/V11.5/java/jdk64/jre/lib/security/policy/unlimited/local_policy.jar local_policy.jar

```

## Deploy MQ on OpenShift

### Deploy Container

1. Create a new project on OpenShift for MQ

```shell
oc new-project sterling-b2bi-mq
```

2. Create Secrets

```shell
oc create -f mq-secret.yaml
```

3. Install IBM MQ from the IBM Github repository.

Clone the IBM MQ charts from the Github repository using the command below.

```shell
git clone https://github.com/IBM/charts.git
```

This will create a directory called "charts". Go to the "charts/stable/ibm-mqadvanced-server-dev" folder and copy mq-override.yaml

```shell
cd charts/stable/ibm-mqadvanced-server-dev

cp ../../../mq-override.yaml override.yaml
```

4. Install the helm chart with the following command , assuming that the file "override.yaml" is placed in the same folder.

```shell
helm install sterling-b2bi-mq --namespace sterling-b2bi-mq --timeout 90m0s -f override.yaml .
```

5. Test connectivity from the outside. 

```shell
oc get svc

  NAME                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
  sterling-b2bi-mq-ibm-mq           ClusterIP   172.xx.6.39     <none>        9443/TCP,1414/TCP   92s
  sterling-b2bi-mq-ibm-mq-metrics   ClusterIP   172.xx.60.239   <none>        9157/TCP            92s
```

and return to default directory

```shell
cd ../../..
```

## Setup Container Images on Registry

[Setup RH OpenShift Image Registry](../third-party/setup-roks-image-registry.md)
## Upload images to Registry

1. Get and export variable

```shell
oc get route image-registry -n openshift-image-registry

export MY_IMG_REGISTRY=image-registry-openshift-image-registry....us-south.containers.appdomain.cloud
```

2. Change to B2Bi Project and export project

```shell
oc project sterling-b2bi-app

export MY_SB2BI_PROJECT=sterling-b2bi-app
```

3. Browse to the location where you have downloaded the B2Bi container image.
   
4. Login to Registry. Load/tag/push and check.

```shell
docker login -u $(oc whoami) -p $(oc whoami -t) $MY_IMG_REGISTRY

docker load -i b2bi-6.1.0.0.tar
docker load -i ps-6.1.0.0.tar
docker load -i purge-6.1.0.0.tar

docker tag b2bi:6.1.0.0 $MY_IMG_REGISTRY/$MY_SB2BI_PROJECT/b2bi:6.1.0.0
docker tag ps:6.1.0.0 $MY_IMG_REGISTRY/$MY_SB2BI_PROJECT/ps:6.1.0.0
docker tag purge:6.1.0.0 $MY_IMG_REGISTRY/$MY_SB2BI_PROJECT/purge:6.1.0.0

docker push $MY_IMG_REGISTRY/$MY_SB2BI_PROJECT/b2bi:6.1.0.0
docker push $MY_IMG_REGISTRY/$MY_SB2BI_PROJECT/ps:6.1.0.0
docker push $MY_IMG_REGISTRY/$MY_SB2BI_PROJECT/purge:6.1.0.0
```

5. Check result

```shell
oc get imagestream 
```

## Deploy Sterling Toolkit on OpenShift

### Configure Storage for Tookit

1. Create a new project on OpenShift for Tookit

```shell
oc new-project sterling-b2bi-toolkit
```

2. Define Permissions

```shell
oc adm policy add-scc-to-user anyuid -z default -n sterling-b2bi-toolkit
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
oc project sterling-b2bi-toolkit

oc get pods

NAME                         READY   STATUS    RESTARTS   AGE
sterling-b2bi-toolkit-59..   1/1     Running   0          73m
```

Export Toolkit Pod 

```shell
export  MY_TOOLKIT_POD=sterling-b2bi-toolkit-59..
```

2. Connect to Pod and setup directories

```shell
oc rsh pod/$MY_TOOLKIT_POD
```

```shell
cd /var/nfs-data/

mkdir resources logs documents

useradd -u 1010 b2biuser
chown b2biuser:b2biuser logs 
chown b2biuser:b2biuser resources 
chown b2biuser:b2biuser documents

exit
```

3. Copy files to pod

```shell
oc cp db2jcc4.jar $MY_TOOLKIT_POD:/var/nfs-data/resources/
oc cp db2jcc_license_cu.jar $MY_TOOLKIT_POD:/var/nfs-data/resources/
oc cp local_policy.jar $MY_TOOLKIT_POD:/var/nfs-data/resources/
```

4. Validate

```shell
oc rsh $MY_TOOLKIT_POD ls -l /var/nfs-data/resources/
```

## Deploy Sterling B2Bi on OpenShift

### Pre-install

1. Create a new project on OpenShift for Tookit

```shell
oc project sterling-b2bi-app
```
2. Extract files from ibm-b2bi-prod-2.0.0.tgz

```shell
tar -xzvf ibm-b2bi-prod-2.0.0.tgz
```

3. Change directory and setup permissions on OpenShift

```shell
cd ibm-b2bi-prod/ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration

oc apply -f ibm-b2bi-scc.yaml --validate=false
oc apply -f ibm-b2bi-cr-scc.yaml --validate=false
oc apply -f ibm-b2bi-psp.yaml
oc apply -f ibm-b2bi-cr.yaml

cd ../../../../..
```

4. Change Rolebinding

```shell
cd ibm-b2bi-prod/ibm_cloud_pak/pak_extensions/pre-install/namespaceAdministration

sed 's/{{ NAMESPACE }}/'$MY_SB2BI_PROJECT'/g' ibm-b2bi-rb-scc.yaml > my-ibm-b2bi-rb-scc.yaml
sed 's/{{ NAMESPACE }}/'$MY_SB2BI_PROJECT'/g' ibm-b2bi-rb.yaml > my-ibm-b2bi-rb.yaml

oc create -f my-ibm-b2bi-rb-scc.yaml
oc create -f my-ibm-b2bi-rb.yaml

cd ../../../../..
```

### Configure Storage for B2Bi

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
cp b2bi-pv.yaml my-b2bi-pv.yaml
```

6. Allocate PV/PVC

```shell
oc create -f my-b2bi-pv.yaml
```

### Configuring passphrase for B2Bi, DB secret and MQ secret

1. Change file **b2i-secrets.yaml**

2. Create secrets

```shell
oc create -f b2bi-secrets.yaml
```


### Deploy with Helm

1. Create file my-b2bi-override.yaml, and change 

```
cp b2bi-override.yaml my-b2bi-override.yaml
```

2. Deploy with Helm

```shell
cd ibm-b2bi-prod

helm install sterling-b2bi-app --namespace sterling-b2bi-app --timeout 120m0s -f ../my-b2bi-override.yaml .
```

You can check install using this commands:

```shell
$ oc get pods
NAME                                    READY   STATUS    RESTARTS   AGE
sterling-b2bi-app-b2bi-db-setup-bltg5   1/1     Running   0          24s

$ oc logs -f sterling-b2bi-app-b2bi-db-setup-bltg5
```