# betterfactory-cognitive-hri

The repository contains a partial deployment involving some of the Cognitive
Human Robot Interaction (C-HRI) scenario components, as defined within the
Better Factory project. The deployment is provided by means of Docker Compose,
and the set of initialized components is depicted in the picture here below:

![docker-deployment](./docker-deployment.png)

The blue-colored components represent the core components for which SUPSI
provides and maintains a Docker image; the other components (grey- and
green-colored) represent dependencies that are provided as Docker images by
third parties.

> NOTE: In this deployment version, the only external dependencies that is
> available only in a private registry is *models*. The *models* image is
> provided and maintained by Holonix (HOL) within the Better Factory project.

## Core Components

### fams

The *fams* component detects possible psychological (e.g., loss of attention,
mental fatigue) or physical (e.g., tiredness) discomfort or harmful situations
for a worker. The situations identified by this component can be targeted by a
short term intervention, which is an intervention that can be triggered and
executed to support a specific worker during the current working shift. Examples
of possible actions include:

- providing visual support by highlighting which component is the next one
  assemble, or where to insert a connector;

- increasing the support provided by the exoskeleton;

- taking a break;

- shut-down all information

### im

The *im* component allows users to easily define intervention rules to
orchestrate a production system. The component monitors the status of the
worker-factory ecosystem in real-time, by elaborating data from sensors,
machines, workers monitoring systems, ERP, and more. The set of intervention
rules are known to the *im*, which decides which is the best one to trigger.

### kafka-orion-gateway

The *kafka-orion-gateway* component serves as a data gateway between two
brokers, i.e., Kafka Broker (KB) and Orion Context Broker by FIWARE (OCB).

A specific service subscribes a set of predefined topics in KB, then converts
the information to an OCB entity and forwards the message to OCB using the OCB
REST API endpoint.

At the same time, a REST controller is defined to receive notifications from
OCB. All the notifications are converted into an Avro-serializable object and
forwarded to dedicated KB topics.

### worker-data-importer

The *worker-data-importer* component downloads the worker responses collected
with the "Consensus" questionnaire (GForm). Each response contains static data
about the worker, which are pushed to the *models* component by means of its
REST API. A cron job is exploited to download new responses.

### Dependencies

#### middleware

The image is based on the [fast-data-dev
(v2.6.2)](https://github.com/lensesio/fast-data-dev/tree/fdd/2.6.2) project by
Lenses.io and runs a full fledged Kafka installation (including extra services,
e.g., UIs).

In addition, the Schema Registry is automatically populated with schemas
available under the `/schemas` directory. In our deployment, the `/schemas`
directory is read from the *kafka-message-model* component.

The middleware is run in secure mode and can be accessed at
[localhost:3040](localhost:3040) (credentials are stored in the docker-compose
file).

#### kafka-message-model

This component embeds the data model shared within the Better Factory project.
The data model is automatically uploaded to the Schema Registry available within
the *middleware*.


#### orion

The *orion* component runs an instance of the Orion Context Broker (v3.1.0). It
requires a MongoDB instance to use as storage system, which is provided by the
*mongo* component.

#### mongo

The *mongo* component runs an official MongoDB docker image (v4.4).


#### models

The *models* component exposes a REST API to access the data model shared among
all the components involved in the C-HRI scenario. The API is accessed by both
the *fams* and *im* components to fetch information about workers and other
factory elements.

> NOTE: this image is available in a private Docker registry hosted at GitLab.
> Please ask HOLONIX to get access to this image.

#### models-db
The *models-db* component runs an official MySql docker image (v5.7).

## How to Use

### Requirements

All the components are provided as Docker images, thus the following
software is required:

- Docker
- Docker Compose

We tested our deployment on a machine running Ubuntu 21, with Docker v20.10.8,
and Docker Compose v1.29.2.

### Install

Before running the containers, it is required to download the Docker images from
their respective registries. While some images are publicly available, some
other require credentials to be downloaded from private registries.

> NOTE: Images provided by SUPSI can be download from the GitLab container
> registry, which supports the token-based authentication. Please send your
> request for a new token to the repository maintainers.

Once you are provided with a username and a token, you can issue the following
command to login to the private GitLab Docker registry and download the images:

```bash
docker login registry.example.com -u <username> -p <token>
docker-compose pull
```

### Usage

We use Docker Compose to automatically runs all the components as Docker
containers. Each container can be customized by editing its environment
parameters. The current deployment already set the right values for all the
parameters. We suggest the user to only modify variables listed in the `.env`
file, if needed.

| Parameter name             | Parameter value | Default |
| -------------------------- | --------------- | ------  |
| BF_USER | A username shared by all the components | **bfuser** |
| BF_PASSWORD | A password shared by all the components | **bfpwd123** |
| MODELS_MYSQL_DATABASE | Name of the database to store the shared model | **models**|
| MODELS_MYSQL_ROOT_PASSWORD | Password for the root user in MySQL | **root** |
| CREDENTIALS | A JSON file containing the Google Service account credentials to access the GForm responses. | **/app/service_account.json** |
| SPREADSHEET | The name of the Google Spreadsheet containing the GForm responses. | **[BF] Worker Profiling (Risposte)** |
| FIELDS_MAP | A JSON file providing the mapping between the GForm questions and the ConsensusWorker parameters | **/app/fields_map.json** |

> NOTE: this repo contains submodules. Please run the following commands to fetch the submodules content, if needed:
```bash
git submodule init
git submodule update
```

> NOTE: To download responses from the Google Spreadsheet, a JSON file containing the user credentials is needed. The file will be provided upon request.

Run the **up** command to start all the containers:

```bash
docker-compose \
  -f bf-worker-data-importer/docker-compose.yml \
  -f bf-fams/docker-compose.yml \
  -f bf-im/docker-compose.yml \
  -f bf-kafka-orion-gateway/docker-compose.yml \
  up -d
```

You can now access different components:

- The IM is available at [localhost:8666](http://localhost:8666); you can use
  the IM UI to edit the possible interventions available for the factory. The
  intervention will be triggered by the IM and written to the *middleware*
  component.

- Kafka is accessible from the Kafka Development Environment, which is available
  at [localhost:3040](http://localhost:3040/). Here you can check schemas
  uploaded to the Schema Registry, as well as existing Kafka topics and their
  content.

- Orion is available at [localhost:1026](http://localhost:1026/) and can be
  queried using its
  [API](https://fiware-orion.readthedocs.io/en/1.13.0/user/walkthrough_apiv2/index.html).
  For example, the following query retrieves all the entities stored to the
  current Orion instance:

```bash
curl localhost:1026/v2/entities -s -S -H 'Accept: application/json' | \
  python -mjson.tool
```

## Maintainers

- Vincenzo Cutrona - vincenzo.cutrona@supsi.ch
- Niko Bonomi - niko.bonomi@supsi.ch
- Giuseppe Landolfi - giuseppe.landolfi@supsi.ch
