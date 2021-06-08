# Setup RH OpenShift Image Registry

1. The following procedure is a one-time setup to activate the internal OpenShift registry route.

```shell
oc project openshift-image-registry
oc get svc
oc create route reencrypt --service=image-registry
oc get route image-registry
```

2. Now that we have our registry url, let's export a variable

```
export MY_IMG_REGISTRY=image-registry-openshift-image-registry....us-south.containers.appdomain.cloud
```

3. Edit Image Registry Route and insert the following `haproxy.router...` line under annotation key.

```
oc edit route image-registry
```

4. Example: 

```yaml
Annotation to add:

apiVersion: route.openshift.io/v1
kind: Route
metadata:
annotations:
    openshift.io/host.generated: "true"
    haproxy.router.openshift.io/balance: source
```
Save and close the file (syntax like Vim -> `: wq!`). 
