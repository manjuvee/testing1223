# Image Mirror and Pull Secret Setup

## Image Mirror Setup
All operator bundles are now built with a reference to their production image. This means that the CSV’s will reference images by the registry they will live in when we ship i.e. docker.io/ibmcom for all of the operator related images/bundles etc. and cp.icr.io/cp for the application images that the operators instantiate (Operands).

Images that are built and pushed into us.icr.io/abp-builds during development and similarly images that are pushed into cp.stg.icr.io/cp for end of sprint drivers now require a mirror to enable reference to these locations. 

```
apiVersion: operator.openshift.io/v1alpha1 
kind: ImageContentSourcePolicy
metadata:
  name: mirror-config
spec:
  repositoryDigestMirrors:
  - mirrors:
    - us.icr.io/abp-builds  # NOTE: Omit this line if you do not have access
    - cp.stg.icr.io/cp 
    source: cp.icr.io/cp 
  - mirrors: 
    - us.icr.io/abp-builds  # NOTE: Omit this line if you do not have access
    - cp.stg.icr.io/cp 
    source: docker.io/ibmcom
```

You can either save the above YAML snippet to a file and `oc apply -f the-file.yaml` or you can apply it using the OpenShift UI by clicking the :plus: icon in the top right which will take you to https://<YOUR-CLUSTER-URL>/k8s/all-namespaces/import where you can directly copy/paste and click Create.

**Note:** Applying this YAML will roll every node in your cluster in a controlled fashion. This can take some time to finish and the progress can be observed using the command oc get machineconfigpools  (Or in the OCP UI at `https://<YOUR-CLUSTER-URL>/k8s/cluster/machineconfiguration.openshift.io~v1~MachineConfigPool`) Which has an output during like:

```
NAME     CONFIG                  UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-79f05   False     True       False      3              1                   1                     0                      42h
worker   rendered-worker-59097   False     True       False      3              2                   2                     0                      42h
```
And a final output when everything is ready of:
```
NAME     CONFIG                  UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-79f05   True      False      False      3              3                   3                     0                      42h
worker   rendered-worker-59097   True      False      False      3              3                   3                     0                      42h
```

When the nodes have rolled, your cluster is now configured to pull images from the mirror repositories and now requires appropriate image pull secrets.

## Global Image Pull Secret Setup
The ‘friendliest’ way to add credentials to your Global IPS is via the OpenShift UI. The above link describes the CLI process but it’s less fiddly to navigate to `https://<YOUR-CLUSTER-URL>/k8s/ns/openshift-config/secrets/pull-secret/edit` and use the Add Credentials button at the bottom of the form to enter your credentials without encoding etc.

Each entry requires:
```
Registry Server Address -> i.e. cp.stg.icr.io
Username -> iamapikey
Password -> <your-api-key>
Email -> ibm@ibm.com
```

Clicking `Save` will again roll all the nodes of your cluster as this is a global configuration change. Remember the command `oc get machineconfigpools` to monitor the status of the roll.