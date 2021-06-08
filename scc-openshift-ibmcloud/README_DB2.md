
## Deploy DB2 on OpenShift

### Deploy Container

1. Create a new project on OpenShift for DB2 

```shell
oc new-project sterling-scc-db2
```

2. define service and access accounts.

```shell
oc create serviceaccount sterling-scc-db2-sa

oc adm policy add-scc-to-user privileged -n sterling-scc-db2 -z sterling-scc-db2-sa
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

cat <<EOF >> create_iccm.sql
CREATE DATABASE ICCM AUTOMATIC STORAGE YES USING CODESET UTF-8 TERRITORY DEFAULT COLLATE USING SYSTEM PAGESIZE 32768@

CONNECT TO ICCM@
CREATE BUFFERPOOL ICCM_04KBP IMMEDIATE SIZE 1000 PAGESIZE 4K@
CREATE BUFFERPOOL ICCM_08KBP IMMEDIATE SIZE 1000 PAGESIZE 8K@
CREATE BUFFERPOOL ICCM_16KBP IMMEDIATE SIZE 1000 PAGESIZE 16K@
CONNECT RESET@

CONNECT TO ICCM@
CREATE USER TEMPORARY TABLESPACE SCCUSERTMP PAGESIZE 32K BUFFERPOOL IBMDEFAULTBP@
CREATE REGULAR TABLESPACE TS_REG04_ICCM PAGESIZE 4K BUFFERPOOL ICCM_04KBP@
CREATE REGULAR TABLESPACE TS_REG08_ICCM PAGESIZE 8K BUFFERPOOL ICCM_08KBP@
CREATE REGULAR TABLESPACE TS_REG16_ICCM PAGESIZE 16K BUFFERPOOL ICCM_16KBP@
CONNECT RESET@
EOF

db2 -td@ -vf ./create_iccm.sql

db2 list database directory

exit

exit
```

3. Test db2 connectivity from the outside. 

Your can test connectivity from the outside using DBEaver or another SQL Tool. Identify the LoadBalancer IP address (port 50000). EXTERNAL-IP parameter

```shell
oc get svc

  NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP        PORT(S)           AGE
  sterling-scc-db2-svc-lb  LoadBalancer   172.xx.xx.250   169.xx.xx.83   50000:30707/TCP   43m
```

4. Copy JDBC Files from Container

```shell
oc cp db2-0:/opt/ibm/db2/V11.5/java/db2jcc4.jar db2jcc4.jar
oc cp db2-0:/opt/ibm/db2/V11.5/java/db2jcc_license_cu.jar db2jcc_license_cu.jar
```