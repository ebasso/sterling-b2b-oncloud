# Deploying Sterling Secure Proxy on Openshift using the IBM Cloud

Steps required to install Sterling Secure Proxy (SSP) on OpenShift on the IBM Cloud

Download the following file from Passport Advantage:

* IBM Sterling Secure Proxy V 6.0.2 for RHOCP English (CC8IAEN )



## Clone this repository


1. Login on OpenShift

```shell
git clone https://github.com/ebasso/sterling-b2b-oncloud.git

cd ssp-openshift-ibmcloud
```

## Create a new project on OpenShift for SSP

1. Login on OpenShift

```shell
oc login --token=sha256~... --server=https://...containers.cloud.ibm.com:31234
```

2. Create a new project on OpenShift for SSP

```shell
oc new-project sterling-ssp
```

## Setup Container Images on Registry

[Setup RH OpenShift Image Registry](../third-party/setup-roks-image-registry.md)

### Upload images to Registry

1. Get and export variable

```shell
oc get route image-registry -n openshift-image-registry

export MY_IMG_REGISTRY=image-registry-openshift-image-registry....us-south.containers.appdomain.cloud
```

2. Change to SSP Project and export project

```shell
oc project sterling-ssp

export MY_SSP_PROJECT=sterling-ssp
```

3. Browse to the location where you have downloaded the SSP container image.

```shell
unzip SSP.V6020.Docker.tar.zip

tar -xvf SSP.V6020.Docker.tar

tar -xvf ssp_6020_docker.tar

tar -xvf ssp_6020_helmchart.tar

rm SSP.V6020.Docker.tar

rm ssp_6020_docker.tar

rm ssp_6020_helmchart.tar
```
   
4. Login to Registry. Load/tag/push and check.

```shell
docker login -u $(oc whoami) -p $(oc whoami -t) $MY_IMG_REGISTRY

docker load -i ssp_cm_docker_image_6020.tar
docker load -i ssp_engine_docker_image_6020.tar
docker load -i ssp_ps_docker_image_6020.tar

docker tag ssp-cm-docker-image:6.0.2.0     $MY_IMG_REGISTRY/$MY_SSP_PROJECT/ssp-cm-docker-image:6.0.2.0
docker tag ssp-engine-docker-image:6.0.2.0 $MY_IMG_REGISTRY/$MY_SSP_PROJECT/ssp-engine-docker-image:6.0.2.0
docker tag ssp-ps-docker-image:6.0.2.0     $MY_IMG_REGISTRY/$MY_SSP_PROJECT/ssp-ps-docker-image:6.0.2.0

docker push $MY_IMG_REGISTRY/$MY_SSP_PROJECT/ssp-cm-docker-image:6.0.2.0
docker push $MY_IMG_REGISTRY/$MY_SSP_PROJECT/ssp-engine-docker-image:6.0.2.0
docker push $MY_IMG_REGISTRY/$MY_SSP_PROJECT/ssp-ps-docker-image:6.0.2.0
```

5. Check result

```shell
oc get imagestream 
```

6. Returno do previous directory and extract the helm charts

```shell
cd <CHANGE HERE>/ssp-openshift-ibmcloud

tar -xzvf <Downloads_Directory>/ibm-ssp-cm-1.1.0.tgz
tar -xzvf <Downloads_Directory>/ibm-ssp-engine-1.1.0.tgz
tar -xzvf <Downloads_Directory>/ibm-ssp-ps-1.1.0.tgz
```


## Deploy Sterling Toolkit on OpenShift

Follow the article [Deploy Sterling Toolkit on OpenShift](../sterling-toolkit/) to setup toolkit


## Deploy Secure Proxy Engine on OpenShift

Follow the article [Deploy Secure Proxy Engine on OpenShift](README_SSP_ENGINE.md)


## Deploy Secure Proxy CM on OpenShift

Follow the article [Deploy Secure Proxy Configuration Manager on OpenShift](README_SSP_CM.md)

## Checking

$ oc get svc
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                          AGE
ssp-cm-ibm-ssp-cm           LoadBalancer   172.xx.xxx.204   52.xxx.xxx.12   8443:32103/TCP,62366:30939/TCP   5m32s
ssp-engine-ibm-ssp-engine   LoadBalancer   172.xx.xx.244    52.xxx.xxx.11   63366:31072/TCP                  22m