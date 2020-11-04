# abp-deploy

This repository provides the required steps to install `IBM Cloud Pak for Automation`

Alternatively, you can try the [install-demo.sh](install-demo.sh) script. The script does several additional steps including an install of the [Demo Cartridge](https://github.ibm.com/automation-base-pak/abp-demo-cartridge)

**Note:** All of the commands discussed here are `bash` assumed

## Prerequisites

- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git/)
- Red Hat Openshift Container Platform - [4.6](https://docs.openshift.com/container-platform/4.6/welcome/index.html) or later
- Red Hat Openshift Container Platform [CLI](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html)
- Access to images for `IBM Cloud Pak for Automation` from the IBM Cloud Container Registry
    - ABP developers use images from the build pipelines in regsitry: `us.icr.io` - [Retrieve your API key](https://cloud.ibm.com/docs/account?topic=account-userapikey#create_user_key) or [Request Access](https://github.ibm.com/automation-base-pak/abp-project-requests/issues/new?assignees=abpbuild,+dnastaci&labels=access+request&template=request-read-access-to-abp-container-images.md&title=%5BRequest+read+access+to+ABP+container+images%5D)
    - ABP consumers use images promoted to staging in registry: `cp.stg.icr.io`

Set the following environment variables, where the value of `IMG_REG_PASSWORD` should be set to the `API Key` obtained above.
  ```
  export IMG_REG_HOST=us.icr.io
  export IMG_REG_USER=iamapikey
  export IMG_REG_PASSWORD=<PASSWORD | API_KEY>
  ```
**Note:** If you want to use the `IBM Cloud Pak for Automation` images from any other container registry, you will have to obtain the credentials to [access the registry](https://docs.openshift.com/container-platform/4.5/openshift_images/managing_images/using-image-pull-secrets.html#images-allow-pods-to-reference-images-from-secure-registries_using-image-pull-secrets) and set them in the environment variables (as above)

- The install currently depends on `Zen` packages and as these packages come from a different image registry (IBM Cloud Container Registry - Staging). The access can be obtained from [here](https://github.ibm.com/alchemy-registry/image-iam/pull/866#issuecomment-24741021). We'll need the following environment variables set with the right values.
  ```
  export ZEN_IMG_REG_HOST=cp.stg.icr.io
  export ZEN_IMG_REG_USER=cp
  export ZEN_IMG_REG_PASSWORD=<PASSWORD | API_KEY>
  ```
- Login to Openshift Cluster using [oc](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html#cli-logging-in_cli-developer-commands) tools. Typically, the following form:
  ```
  oc login --token=<OC_TOKEN> --server=<OC_SERVER_URL>
  ```
- Run the below commands to configure the ceph storage :
  ```
  git clone https://github.com/rook/rook.git
  cd rook/cluster/examples/kubernetes/ceph
  oc create -f common.yaml
  oc create -f operator-openshift.yaml
  oc create -f cluster.yaml
  oc create -f ./csi/rbd/storageclass.yaml
  oc create -f ./csi/rbd/pvc.yaml
  oc create -f filesystem.yaml
  oc create -f ./csi/cephfs/storageclass.yaml
  oc create -f ./csi/cephfs/pvc.yaml
  oc create -f toolbox.yaml
  ```
- Setup the required `ImageContentSourcePolicy` and global Image Pull Secret (IPS) following [these steps](ImageMirrorAndGlobalIPS.md).

### (Optional) Validate the prerequisites

```
$ oc version
Client Version: openshift-clients-4.5.0-202006231303.p0-13-gd7f3ccf9a
Server Version: 4.6.1
Kubernetes Version: v1.19.0+d59ce34
```

## Add `IBM Cloud Pak for Automation` Operators to the catalog

1. Create a [project](https://docs.openshift.com/container-platform/4.6/applications/projects/working-with-projects.html) into which the `IBM Cloud Pak for Automation` will be installed. Ensure the environment variable is exported for forward use

    ```
    export ABP_PROJECT=acme-abp
    oc new-project $ABP_PROJECT
    ```

2. Create an image pull secret called `ibm-entitlement-key` for the cluster to be able to pull the images from the container registry. We'll have to create the key in both the projects:
    - `openshift-marketplace` to be able to initialize the `CatalogSource` that provides the Openshift Operator Catalog with the operators of `IBM Cloud Pak for Automation` (and)
    - the project in which `IBM Cloud Pak for Automation` is being installed

    ```
    oc create secret docker-registry ibm-entitlement-key --docker-server=$IMG_REG_HOST --docker-username=$IMG_REG_USER --docker-password=$IMG_REG_PASSWORD -n openshift-marketplace
    oc secrets link default ibm-entitlement-key --for=pull -n openshift-marketplace

    oc create secret docker-registry stg-key --docker-server=$ZEN_IMG_REG_HOST --docker-username=$ZEN_IMG_REG_USER --docker-password=$ZEN_IMG_REG_PASSWORD -n openshift-marketplace
    oc secrets link default stg-key --for=pull -n openshift-marketplace

    oc create secret docker-registry ibm-entitlement-key --docker-server=$IMG_REG_HOST --docker-username=$IMG_REG_USER --docker-password=$IMG_REG_PASSWORD -n $ABP_PROJECT
    ```

    ## OR
    Add the image pull secret ibm-entitlement-key as part of the Global pull-secret

      a. Run below command. This will create a .dockerconfigjson file in your current directory.
      ```
      oc extract secret/pull-secret -n openshift-config --to=.
      ```
      b.Encode your API key, using the below command
      ```
      printf "iamapikey:My_API_KEY" | base64
      ```
      c. Take a backup of the .dockerconfigjson and then add the following snippet to the .dockerconfigjson file :
      ```
      "us.icr.io": {
        "auth": "The_encoded_API_Key_you_got_in_step_3",
        "email": "your_ibm_email_id"
      }
      ```
      d. Update the data keys by running the below command
      ```
      oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson
      ```
      e. Try oc get nodes. Wait till all the nodes arrive at the Ready Status.
    
      f. Once all the nodes are Ready, re-check your pod status. All the pods should be running now.

3. Add the `IBM Common Services` operators to the Openshift Operator Hub catalog

    ```sh
    cat <<EOF | oc apply -f -
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
      name: opencloud-operators
      namespace: openshift-marketplace
    spec:
      displayName: IBMCS Operators
      publisher: IBM
      sourceType: grpc
      image: docker.io/ibmcom/ibm-common-service-catalog:latest
      updateStrategy:
        registryPoll:
          interval: 45m
    EOF
    ```

    **Verify:**

    ```
    $ oc get pods -n openshift-marketplace | grep opencloud-operators
    opencloud-operators-59fc5 1/1 Running 0 53s
    ```

4. Add the `Zen` operators to the Openshift Operator Hub catalog

    ```sh
	cat <<EOF | oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: CatalogSource
	metadata:
	  name: ibm-cp-data-operator-catalog
	  namespace: openshift-marketplace
	spec:
	  displayName: Cloud Pak for Data
	  image: cp.stg.icr.io/cp/ibm-cp-data-operator-catalog:staging-latest
	  publisher: IBM
	  sourceType: grpc
	  imagePullPolicy: Always
	EOF
    ```

    **Verify:**

    ```
    $ oc get pods -n openshift-marketplace | grep ibm-cp-data-operator-catalog
    ibm-cp-data-operator-catalog-89fc5 1/1 Running 0 42s
    ```

5. Add the `IBM Cloud Pak for Automation` operators to the Openshift Operator Hub catalog
    **Note:** Consumers of ABP using the staging registry will use `image: cp.stg.icr.io/cp/abp-catalog:0.0.8` (where `0.0.8` is the sprint 8 driver tag)

    ```sh
    cat <<EOF | oc apply -f -
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
      name: abp-operators
      namespace: openshift-marketplace
    spec:
      displayName: IBM ABP Operators
      publisher: IBM
      sourceType: grpc
      image: us.icr.io/abp-builds/abp-catalog:latest
      updateStrategy:
        registryPoll:
            interval: 45m
    EOF
    ```

    **Verify:**

    ```
    $ oc get pods -n openshift-marketplace | grep abp-operators
    abp-operators-sz64q 1/1 Running 0 32s
    ```

**Note:** Once Zen Operator CR is issued, setting of the Zen Dashboard takes close to 25 min to complete.

## Install `IBM Cloud Pak for Automation`

If you would like to explore `IBM Cloud Pak for Automation` via a `Demo Cartridge`, skip to the section [here](install-demo-cartridge-for-ibm-cloud-pak-for-automation)

**Note:** Installing `Demo Cartridge` will also install `IBM Cloud Pak for Automation`

This can be done either via the `Red Hat Openshift Cluster Console` or by directly applying the corresponding Kubernetes resources using `oc apply`.

- From the Openshift Console, follow the below navigation flow:

  ```
  Red Hat Openshift Container Platform Cluster Administrator Console
  |_ Operators
    |_ Operator Hub
      |_ Automation Base Pak Operator
        |_ Install
          |_ Choose a specific installation mode - Both namespace mode or all namespaces mode are supported
          |_ When a namespace mode is selected, select the namespace set for $ABP_PROJECT
          |_ Install
  ```

- Run the below command for installing through CLI

  ```
  cat <<EOF | oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: abp-operator
    namespace: ${ABP_PROJECT}
  spec:
    channel: alpha
    installPlanApproval: Automatic
    name: abp-operator
    source: abp-operators
    sourceNamespace: openshift-marketplace
  EOF
  ```

>**Note:** Once the `IBM Cloud Pak for Automation` is installed, can be configured for use through the API made available via [Custom Resources](https://pages.github.ibm.com/automation-base-pak/abp-playbook/cartridges/custom-resources)

>Refer the [Troubleshooting](TroubleShoot.md) section, to resolve the issues related to **ImagePullBackOff** error.


## Install `Demo Cartridge for IBM Cloud Pak for Automation`

This can be done either via the `Red Hat Openshift Cluster Console` or by directly applying the corresponding Kubernetes resources using `oc apply`.

**Note:** If the Cartridge uses `AIModel` Custom Resource, follow the instructions [here](user-interface-of-ibm-cloud-pak-for-automation) before proceeding

- From the Openshift Console, follow the below navigation flow:

  ```
  Red Hat Openshift Container Platform Cluster Administrator Console
  |_ Operators
    |_ Operator Hub
      |_ ABP Demo Cartridge
        |_ Install
          |_ Choose a specific installation mode - Both namespace mode or all namespaces mode are supported
          |_ When a namespace mode is selected, select the namespace set for $ABP_PROJECT
          |_ Install
  ```

  **Verify:**

  Check the status of the Installed Operator:

    ```
    Red Hat Openshift Container Platform Cluster Administrator Console
      |_ Operators
        |_ Intalled Operators
    ```

- Run the below command for installing through CLI

```
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: abp-demo-cartridge-operator
  namespace: ${ABP_PROJECT}
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: abp-demo-cartridge
  source: abp-operators
  sourceNamespace: openshift-marketplace
EOF
```

## `Zen` specifics

- Once the `ABP Operator` is installated, the OperandRegistry & OperandConfig have to be patched for `Zen`.

	```
	oc -n ibm-common-services patch opreg common-service --type='json' -p='[{"op": "add", "path": "/spec/operators/0", "value": {"channel": "v1.0", "name": "ibm-cp-data-operator", "namespace": "ibm-common-services", "packageName": "ibm-cp-data-operator", "scope": "public", "sourceName": "ibm-cp-data-operator-catalog", "sourceNamespace": "openshift-marketplace" } }]'

	oc -n ibm-common-services patch opcon common-service --type='json' -p='[{"op": "add", "path": "/spec/services/0", "value": {"name": "ibm-cp-data-operator", "spec": {}}}]'
	```

- After the `Cartridge CR` is issued, the Zen Operand (within `ibm-common-services` project) initialization is triggered. At this point, you see the ImgErrPull for Zen Operand (ibm-cp-data-operator). Please follow the below:

	```
  oc create secret docker-registry ibm-entitlement-key --docker-server=$ZEN_IMG_REG_HOST --docker-username=$ZEN_IMG_REG_USER --docker-password=$ZEN_IMG_REG_PASSWORD -n ibm-common-services
	oc secrets link ibm-cp-data-operator-serviceaccount ibm-entitlement-key --for=pull -n ibm-common-services

  oc -n ibm-common-services delete $(oc get pod -n ibm-common-services -l app.kubernetes.io/managed-by=ibm-cp-data-operator,app.kubernetes.io/name=ibm-cp-data-operator -o name)
	```

## Instance creation of `Demo Cartridge` aka Custom Resource

- From the Openshift Console, follow the below navigation flow:

  ```
  Red Hat Openshift Container Platform Cluster Administrator Console
  |_ Operators
    |_ Installed Operators
      |_ ABP Demo Cartridge
        |_ ABPDemo
          |_ Create ABPDemo
          |_ Create
  ```

  **Verify:**

  ```
  $ oc get pods -n $ABP_PROJECT | grep abp-kafka-kafka
  abp-kafka-kafka-0         2/2     Running   0    110s
  abp-kafka-kafka-1         2/2     Running   0    110s
  abp-kafka-kafka-2         2/2     Running   0    110s
  ```

- Wait for the pods to stand up.

    ```
    $ oc get pods -n $ABP_PROJECT | grep kafka
    ```

- Link secret for `kafka-connect-cluster-connect` so that Image can be pulled.

    ```
    $ oc secrets link kafka-connect-cluster-connect ibm-entitlement-key --for=pull
    ```

- Delete `kafka-connect-cluster-connect` pod to restart with Secrets and Refresh Pods again. Change directory to `abp-deploy` and run

    ```
    $ oc delete pod -l app.kubernetes.io/name=kafka-connect
    $ oc get pods -n $ABP_PROJECT | grep kafka-connect-cluster-connect
    kafka-connect-cluster-connect-69fbd5c4b8-blq5t    1/1    Running   0   118
    ```

## User Interface of `IBM Cloud Pak for Automation`

  ```
  $ oc get routes
  ```

  and proceed to the `abp-ui-route`/basepakui in your browser.

## Install Open Data Hub Operator

1. Create a [project](https://docs.openshift.com/container-platform/4.6/applications/projects/working-with-projects.html) into which the `Kubeflow` will be installed.

    ```
    oc new-project kubeflow
    ```

2. From the Openshift Console, follow the below navigation flow:

    ```
    Red Hat Openshift Container Platform Cluster Administrator Console
    |_ Operators
      |_ Operator Hub
        |_ Open Data Hub Operator
          |_ Install
            |_ Choose a specific installation mode - Both namespace mode or all namespaces mode are supported
            |_ When a namespace mode is selected, select the namespace `kubeflow`
            |_ Install
    ```

3. Create the `kfDef` resource that will create instances of both `Kubeflow` and `KfServing`

    ```
    apiVersion: kfdef.apps.kubeflow.org/v1
    kind: KfDef
    metadata:
      annotations:
        kfctl.kubeflow.io/force-delete: 'false'
      name: kubeflow
      namespace: kubeflow
    spec:
      applications:
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: openshift/openshift-scc/base
          name: openshift-scc
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/application/istio-stack
          name: istio-stack
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/application/cluster-local-gateway
          name: cluster-local-gateway
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/application/istio
          name: istio
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/application/add-anonymous-user-filter
          name: add-anonymous-user-filter
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: application/v3
          name: application
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/application/cert-manager-crds
          name: cert-manager-crds
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/application/cert-manager-kube-system-resources
          name: cert-manager-kube-system-resources
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/application/cert-manager
          name: cert-manager
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/base
          name: kubeflow-apps
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/components/profile-control-plane
          name: kubeflow-apps
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/components/tektoncd
          name: kubeflow-apps
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/components/kfp-tekton
          name: kubeflow-apps
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/components/metadata
          name: kubeflow-apps
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/components/notebooks
          name: kubeflow-apps
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/components/pytorch-job
          name: kubeflow-apps
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/components/tf-job
          name: kubeflow-apps
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: metacontroller/base
          name: metacontroller
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: stacks/openshift/application/spark-operator
          name: spark-operator
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: knative/installs/generic
          name: knative
        - kustomizeConfig:
            repoRef:
              name: manifests
              path: kfserving/installs/generic
          name: kfserving
      repos:
        - name: manifests
          uri: 'https://github.com/IBM/manifests/archive/master.tar.gz'
      version: master
    ```

  **Verify:**

  - Ensure all the pods are running:

      ```
      $ oc get pods -n kubeflow
      ```

## Cleanup

**Note:** These steps must be performed before using the install-demo.sh script on the cluster again (or) to repeat the installation

1.  Delete the instance of the **ABPDemo** under the ABP Demo Cartridge operator.

    ```
    Red Hat Openshift Container Platform Cluster Administrator Console
    |_ Operators
      |_ Installed Operators
        |_ ABP Demo Cartridge
          |_ ABPDemo
            |_ Delete All Instances
    ```

2.  In your Namespace/Project under Installed operators, delete all the operators that are installed Namespace Scoped. Any operators on the project which are installed on "**All Namespaces**" should not be deleted.

3.  Under **Administration** go to view the Custom Resource Definitions
    - Go to CatalogSource, under Instances delete **abp-operators** (may be called abp-catalog) and **opencloud-operators**
    - Back on the CRDs page delete **ABPDemo** and **AutomationBase** Custom Resource Definitions.

4.  Delete your **Namespace/Project**

5.  Delete **ibm-entitlement-key** secret from the **openshift-marketplace** namespace

**Note:** Before you can install again using install-demo.sh please change your environment variables
`ABPDEMO_PROJECT`, `TMPDIR` which are used by the script. Failing to change these may result in older environments being stood up.

### Uninstall/Clean Up Script

A cluster previously having run the BasePak Demo, can now clean up using a script. See [here](https://github.ibm.com/automation-base-pak/cloudpak-svt-cluster-tools/blob/main/scripts/abp_cleanup.sh).
