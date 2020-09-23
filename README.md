# abp-demo-cartridge
A demo cartridge to act as a real cartridge would

# The producer

The producer generates sequences of timestamped BPMNX Process/Activity messages identified by a unique sequenceId. Sequences may fail in approx 10% of cases.

Each sequence runs as follows:

```
1.bpmnx:PROCESS_STARTED
2. For each activity but the last one:
  a) bpmnx:ACTIVITY_READY
  b) bpmnx:ACTIVITY_STARTED
  c) bpmnx:ACTIVITY_COMPLETED
  d) bpmnx:ACTIVITY_TERMINATED
3. For the last activity:
  a) On success:
     i) bpmnx:ACTIVITY_READY
     ii) bpmnx:ACTIVITY_STARTED
     iii) bpmnx:ACTIVITY_COMPLETED
     iv) bpmnx:ACTIVITY_TERMINATED
     v) bpmnx:PROCESS_COMPLETED
  d) On failure:
     i) bpmnx:ACTIVITY_READY
     ii) bpmnx:ACTIVITY_STARTED
     iii) bpmnx:ACTIVITY_FAILED
     iv) bpmnx:ACTIVITY_TERMINATED
     v) bpmnx:PROCESS_FAILED
4. bpmnx:PROCESS_TERMINATED
```
A message looks similar to:
```
{
  "bpm": {
    "sequenceId": 1  // A unique identifier for the sequence, common to all messages in the sequence
  },
  "id": 5577006791947779000,
  "timestamp": "2020-09-08T13:23:31Z",
  "type": "bpmnx:PROCESS_STARTED"   // The message type
}
```
The CloudEvents protocol defines that the above payload is base64-encoded, along with various CloudEvents headers and keys in the Kafka protocol message. Details here: https://github.com/cloudevents/spec/blob/v1.0/kafka-protocol-binding.md

# Test-running the producer

You can run the producer locally for test purposes by following these steps:

1. Build this repo using `CGO_ENABLED=0 GO111MODULE=on go build -a -o manager main.go`

2. If necessary, make the binary executable using `chmod +x manager`

3. Run `docker run -it -p2181:2181 -p9092:9092 -e "KAFKA_ADVERTISED_LISTNERS=localhost" paspaola/kafka-one-container`
(Note: `LISTNERS` is not a mistake. It has been misspelled in the docker image). This image runs a single container with both Kafka and Zookeeper together. It exposes Kafka on port 9092 and Zookeeper on 2181.

4. Run the producer using `FUNCTION=producer BOOTSTRAP_SERVERS=localhost:9092 ./manager`. This will start producing messages to Kafka.

You can monitor the Kafka broker from the command line using [kafkacat](https://github.com/edenhill/kafkacat). Use the following command:
`kafkacat -b localhost:9092 -C -t demo-raw -f "\n=====\n%o %h %s"`

If you prefer a GUI, use [Conduktor](https://www.conduktor.io/) which has a free single-broker version for testing/evaluation purposes.

Note : If you see the below error in building the docker image :
fatal: could not read Username for 'https://github.ibm.com': terminal prompts disabled
then add the below line in ./Dockerfile (after ENV GOPRIVATE="github.ibm.com" line)
ARG GIT_USER
ARG_GIT_TOKEN

Run git config --global url."https://${GIT_USER}:${GIT_TOKEN}@github.ibm.com/".insteadOf "https://github.ibm.com/"

# Building the abp-demo-cartridge operator

  1.  Clone the demo-cartridge project

```
  git clone git@github.ibm.com:automation-base-pak/abp-demo-cartridge.git
```
2. Enter the project

```
	cd abp-demo-cartridge
```

3. Set the below environment variables

```
 	export GITHUB_USERNAME=<GITHUB_USERNAME>
 	export GITHUB_TOKEN=<GITHUB_TOKEN>
 	export IMG=<image_registry>
```
  eg. image_registry - quay.io/test/abp-demo-cartridge:v0.0.1
```
 	export BUNDLE_IMG=<bundle_image_registy>
```
   eg. bundle_image_registy - quay.io/test/abp-demo-cartridge-bundle:v0.0.1

4. Run the below to command to git access to the private git repository.

    ```git config --global url."https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.ibm.com/".insteadOf "https://github.ibm.com/"```

5. Change the registry name in Dockerfile (Only if you see any access issue) from _us.icr.io/abp-ubi8/abp-minimal_ to _gcr.io/distroless/static:nonroot_

6. Replace the below lines with your image registy location in config/manager/manager.yaml file
```
image: image-registry.openshift-image-registry.svc:5000/test/cartridge:latest
	env:
	    value: image-registry.openshift-image-registry.svc:5000/test/cartridge:latest
```

7. Run ```make generate```

8. Run ``make manifests``

9. oc login to you openshift cluster

10. Run the below commands to buid the image

	- ```make docker-build```

  - ```make docker-push```

  - Run the below commands to build the Bundle Images

      - ```make bundle```

      - ```make bundle-build```

      - ```docker push ${BUNDLE_IMG}```

11. To directly deploy on the openshift cluster and test abp-demo-cartridge operator.

    - set the namespace, by running the below command

	 ```cd config/default/ && kustomize edit set namespace "<namespacename>" && cd ../..```

    - oc login to you openshift cluster

  - Run below command to deploy to the specified namespace

      ```make deploy```
