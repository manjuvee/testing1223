# abp-deploy

This repository provides the required steps to install `IBM Automation Foundation` aka IAF / ABP

Alternatively, you can try the [install-demo.sh](install-demo.sh) script. The script does several additional steps including an install of the [Demo Cartridge](https://github.ibm.com/automation-base-pak/abp-demo-cartridge)

**Note:** All of the commands discussed here are `bash` assumed

## Prerequisites

- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git/)
- Red Hat Openshift Container Platform - [4.6](https://docs.openshift.com/container-platform/4.6/welcome/index.html) or later
  - IBM Automation Foundation developers can provision the Red Hat Openshift Container Platform with OCP+ in [Fyre](https://fyre.ibm.com/ocp).
    - Because of CPU and memory resource requirements, choose the large cluster configuration:
       - `inf:1, cpu:4, ram:8`
       - `master:3, cpu:8, ram:16`
       - `worker:3, cpu:16, ram:32`
- Red Hat Openshift Container Platform [CLI](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html)
- Access to images for `IBM Automation Foundation` from the IBM Cloud Container Registry
    - IAF developers use images from the build pipelines in registry: `us.icr.io`
      - [Retrieve your API key](https://cloud.ibm.com/docs/account?topic=account-userapikey#create_user_key) or [Request Access](https://github.ibm.com/automation-base-pak/abp-project-requests/issues/new?assignees=abpbuild,+dnastaci&labels=access+request&template=request-read-access-to-abp-container-images.md&title=%5BRequest+read+access+to+ABP+container+images%5D)
    - IAF consumers use images promoted to staging in registry: `cp.stg.icr.io`. The access can be obtained following instructions [here](https://github.ibm.com/alchemy-registry/image-iam/pull/866#issuecomment-24741021).

  **Note:** If you want to use the `IBM Automation Foundation` images from any other container registry, you will have to obtain the credentials to [access the registry](https://docs.openshift.com/container-platform/4.5/openshift_images/managing_images/using-image-pull-secrets.html#images-allow-pods-to-reference-images-from-secure-registries_using-image-pull-secrets)
- The install of IAF currently depends on `Zen` packages and these packages come from `cp.stg.icr.io` (IBM Cloud Container Registry - Staging).
- The install of Redis Sentinel currently depends on `IBM Operator for Redis` packages and these packages come from `cp.icr.io` (IBM Cloud Container Registry).
  - Get an IBM Cloud Container Registry entitelment key here: https://myibm.ibm.com/products-services/containerlibrary.
  - Once you have a IBM Cloud Container Registry entitelment key upload it to the global Image Pull Secret (IPS) following [these steps which is at the end of doc link.](ImageMirrorAndGlobalIPS.md)
- Login to Openshift Cluster using [oc](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html#cli-logging-in_cli-developer-commands) tools. Typically, the following form:
  ```
  oc login --token=<OC_TOKEN> --server=<OC_SERVER_URL>
  ```
  **NOTE:** If you're using the OCP UI, this can be obtained via the "Copy Login Command" option under the top right drop-down menu.
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
  **NOTE:** For development, the below step is currently required. There is an alternate approach of using the `latest-dev` tag that needs to be documented.
- Setup the required `ImageContentSourcePolicy` and global Image Pull Secret (IPS) following [these steps](ImageMirrorAndGlobalIPS.md).

## Add `IBM Automation Foundation` Operators to the catalog

1. Create a [project](https://docs.openshift.com/container-platform/4.6/applications/projects/working-with-projects.html) into which the `IBM Automation Foundation` will be installed. Ensure the environment variable is exported for forward use

    ```
    export ABP_PROJECT=acme-abp
    oc new-project $ABP_PROJECT
    ```

2. **(Optional): Skip the step if a global Image Pull Secret (IPS) is configured for IAF images**
  Set the following environment variables with registry access detailes obtained for IAF in prerequisites.


    ```
    export IMG_REG_HOST=<REGISTRY HOST>
    export IMG_REG_USER=<REGISTRY USER>
    export IMG_REG_PASSWORD=<PASSWORD | API_KEY>
    ```

3. **(Optional): Skip the step if a global Image Pull Secret (IPS) is configured for IAF images**
  Create an image pull secret called `ibm-entitlement-key` for the cluster to be able to pull the images from the container registry. We'll have to create the key in both the projects:
    - `openshift-marketplace` to be able to initialize the `CatalogSource` that provides the Openshift Operator Catalog with the operators of `IBM Automation Foundation` (and)
    - the project in which `IBM Automation Foundation` is being installed

    ```
    oc create secret docker-registry ibm-entitlement-key --docker-server=$IMG_REG_HOST --docker-username=$IMG_REG_USER --docker-password=$IMG_REG_PASSWORD -n openshift-marketplace
    oc secrets link default ibm-entitlement-key --for=pull -n openshift-marketplace

    oc create secret docker-registry ibm-entitlement-key --docker-server=$IMG_REG_HOST --docker-username=$IMG_REG_USER --docker-password=$IMG_REG_PASSWORD -n $ABP_PROJECT
    ```

4. Set the following environment variables with registry access detailes obtained for Zen images in prerequisites.

    ```
    export ZEN_IMG_REG_HOST=cp.stg.icr.io
    export ZEN_IMG_REG_USER=cp
    export ZEN_IMG_REG_PASSWORD=<PASSWORD | API_KEY>
    ```

5. **(Optional): Skip the step if a global Image Pull Secret (IPS) is configured for Zen images.** Create an image pull secret called `ibm-entitlement-key` for the cluster to be able to pull Zen images from the container registry.

    ```
    oc create secret docker-registry stg-key --docker-server=$ZEN_IMG_REG_HOST --docker-username=$ZEN_IMG_REG_USER --docker-password=$ZEN_IMG_REG_PASSWORD -n openshift-marketplace
    oc secrets link default stg-key --for=pull -n openshift-marketplace
    ```

6. We currently need to add the below secret (with name `ibm-entitlement-key`) in the `ibm-common-services` namespace for Zen to install

   ```
   oc new-project ibm-common-services
   oc create secret docker-registry ibm-entitlement-key --docker-server=$ZEN_IMG_REG_HOST --docker-username=$ZEN_IMG_REG_USER --docker-password=$ZEN_IMG_REG_PASSWORD -n ibm-common-services
   ```

7. Add the `IBM Common Services` operators to the Openshift Operator Hub catalog

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

5. Add the `IBM Automation Foundation` operators to the Openshift Operator Hub catalog

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
    EOF
    ```

    **Verify:**

    ```
    $ oc get pods -n openshift-marketplace | grep abp-operators
    abp-operators-sz64q 1/1 Running 0 32s
    ```


6. Add the `Zen` operators to the Openshift Operator Hub catalog

    ```sh
    cat <<EOF | oc apply -f -
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
       name: ibm-operator-catalog
       namespace: openshift-marketplace
    spec:
       displayName: "IBM Operator Catalog"
       publisher: IBM
       sourceType: grpc
       image: docker.io/ibmcom/ibm-operator-catalog
       updateStrategy:
         registryPoll:
           interval: 45m
    EOF
    ```

    **Verify:**

    ```
    $ oc get pods -n openshift-marketplace | grep ibm-operator-catalog
    ibm-operator-catalog-nghtl  1/1 Running 0 81m
    ```


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
        |_ Installed Operators
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

- Run the below command to apply the demo cartridge CR yaml. The `democartridge_v1_abpdemo.yaml` should be available under the `abp-deploy` folder.

  `oc apply -f democartridge_v1_abpdemo.yaml`

**Note:** Zen is installed by default when the democartridge_v1_abpdemo.yaml is applied. To skip the Zen deployment, you can set `zen: false` to the CR `spec`.

  **Verify:**

  ```
  $ oc get pods -n $ABP_PROJECT | grep abp-kafka-kafka
  abp-kafka-kafka-0         2/2     Running   0    110s
  abp-kafka-kafka-1         2/2     Running   0    110s
  abp-kafka-kafka-2         2/2     Running   0    110s
  ```

## User Interface of `IBM Cloud Pak for Automation`

  ```
  $ oc get routes
  ```

  and proceed to the `abp-ui-route`/basepakui in your browser.


## Zen Service & dashboard

Zen Operator (CloudPak for Data Operator) gets installed within the `ibm-common-services` namespace. It takes close to 15 minutes for a complete deployment of Zen service.

a. [OPTIONAL] Run the following command to check the status of the Zen Service deployment. The Ready status confirms the succes of zen service deployment.

  ```
  oc get cpdservice -n ibm-common-services abp-zen-cpdservice --output="jsonpath={.status} {'\n'} "
  ```
b. Under `ibm-common-services` project, go to **Networking > Routes** to reach the zen dashboard url.


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

4.  Run the following commands to delete all the zen resources from the `ibm-common-services` namespace.

	```
	oc delete deploy cpd-install-operator dsx-influxdb ibm-nginx icpd-till meta-api-deploy secretshare usermgmt zen-audit zen-core zen-core-api zen-watchdog zen-data-sorcerer zen-watchdog zen-watcher -n ibm-common-services

	oc delete cronjob diagnostics-cronjob usermgmt-ldap-sync-cron-job watchdog-alert-monitoring-cronjob watchdog-alert-monitoring-purge-cronjob zen-watchdog-cronjob -n ibm-common-services

	oc delete statefulset zen-metastoredb  -n ibm-common-services

	```

5. Delete your **Namespace/Project**

6.  Delete **ibm-entitlement-key** secret from the **openshift-marketplace** namespace

**Note:** Before you can install again using install-demo.sh please change your environment variables
`ABPDEMO_PROJECT`, `TMPDIR` which are used by the script. Failing to change these may result in older environments being stood up.

### Uninstall/Clean Up Script

A cluster previously having run the BasePak Demo, can now clean up using a script. See [here](https://github.ibm.com/automation-base-pak/cloudpak-svt-cluster-tools/blob/main/scripts/abp_cleanup.sh).

## Build and push images to the registry on Openshift Cluster

To build and push images locally during development, you may need to do the following (replacing `<cluster_name>` with the name of your cluster):

- Add the host information to your `/etc/hosts` file

  ```
  <cluster_ip>  console-openshift-console.apps.<cluster_name>.cp.fyre.ibm.com api.<cluster_name>.cp.fyre.ibm.com <cluster_name>.cp.fyre.ibm.com cp-console.apps.<cluster_name>.cp.fyre.ibm.com oauth-openshift.apps.<cluster_name>.cp.fyre.ibm.com
  ```

You can obtain the `<cluster_ip>` via `ping https://console-openshift-console.apps.<cluster_name>.cp.fyre.ibm.com`

- Update your insecure registries under Docker Engine in your Docker preferences

  ```
  "insecure-registries": [
    "default-route-openshift-image-registry.apps.<cluster_name>.cp.fyre.ibm.com"
  ]
  ```
