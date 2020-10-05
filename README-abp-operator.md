- [abp-operator](#abp-operator)
  - [Pre-requisites](#pre-requisites)
  - [To start with](#to-start-with)
  - [Build Operator and Bundle Images](#build-operator-and-bundle-images)
  - [Manifest Generation](#manifest-generation)
  - [Troubleshooting](#troubleshooting)
  - [For Developer](#for-developer)

# abp-operator

**Operator for Automation Base Pak**

This Repo has been built using operator-sdk v1.0.0. The operator-sdk v1.0.0 or later should still be used to expand the repo, create new api's and webhooks.

## Pre-requisites

Install [ibmcloud](https://cloud.ibm.com/docs/cli?topic=cli-getting-started), [go](https://golang.org/doc/install), [operator-sdk](https://sdk.operatorframework.io/docs/installation/install-operator-sdk/) and [kustomize](https://kubernetes-sigs.github.io/kustomize/installation/).

Make sure all the required binaries are present in the path. This is required to build and publish the operator.

## To start with

- You need to login to a ibmcloud container private registry to build and push the abp-operator images. To push the docker images, you will need access to ibmcloud private registry
  Login to registry before running the build. For more info [Click here](https://github.ibm.com/automation-base-pak/abp-planning/wiki/Container-Registry-Service-in-IBM-Cloud#registry-urls)

> Note : This needs <API_KEY> to push/pull images from ibm cloud registry(us.icr.io), contact @charankumar.p@in.ibm.com/@jjayabal@in.ibm.com

- Create [Personal access token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) for Git operations. Refer the [**Troubleshooting**](#troubleshooting) section for more details.

## Build Operator and Bundle Images

**Export the Required env variables**

```
    export API_KEY=<API_KEY>
    export GIT_USER=<GIT_USER>
    export GIT_TOKEN=<GIT_TOKEN>
```

**a.** Clone the abp-operator project

```
    git clone git@github.ibm.com:automation-base-pak/abp-operator.git
```

**b.** Navigate to the project

```
    cd abp-operator
```

**c.** Run the below command to **Build and Push** the Operator and Bundle Images to image registry

```
     make build
```

> Note : Default tag for images is `latest`, To build custom image tag provide parameter as `make build IMAGE_VERSION=test`

the above command does series of actions like - `docker build, docker push ,bundle , bundle-build & bundle-push`

This will build and push respective images as

```
    us.icr.io/abp-scratchpad/abp-operator:latest
    us.icr.io/abp-scratchpad/abp-operator-bundle:latest
```

## Manifest Generation

**a.** Clone the abp-operator project

```
    git clone git@github.ibm.com:automation-base-pak/abp-operator.git
```

**b.** Navigate to the project

```
    cd abp-operator
```

**c.** Run the command `make manifests` to generate the CRDs.

**d.** This will inturn run the controller-gen tool, to generate the required CRDs and would save them to the `config/crd` folder.

**e.** The tool also generates the **rbac role** and saves the respective configuration in the `config/rbac` folder. The configuration is based on the annotations found in the code.

## Manual steps till ibm-events-operator is part of the common services.

**a.** Clone the abp-event-operator project

```
    git clone git@github.ibm.com:mhub/ibm-events-operator.git
```

**b.** Navigate to the project

```
    cd ibm-events-operator
```

**c.** Follow the Readme of the ibm events operator for the below steps:
```
     https://github.ibm.com/mhub/ibm-events-operator/blob/master/README.md
     1. Install the Common Services Operator
     2. Add pull secrets
     3. Update the Common Services registry
```

**d.** Update the Common Services registry Config

```
oc patch operandconfig \
-n ibm-common-services \
common-service \
--type=json \
-p '[ {"op":"add","path":"/spec/services/-","value":{"name":"ibm-events-operator","spec":{"operandRequest" : {}}}} ]'

```

## Troubleshooting

- If you notice the below error while building the operator -

```
    fatal: could not read Username for 'https://github.ibm.com': terminal prompts disabled
```

Adding the below line in the ./Dockerfile (after ENV GOPRIVATE="github.ibm.com" line) should resolve the issue

`RUN git config --global url."https://<replace_with_git_userid>:<replace_with_git_token>@github.ibm.com/".insteadOf "https://github.ibm.com/"`

## For Developer

If any change in `CRD`(api/types folder). Run `make bundle` to update the Bundle manifests
