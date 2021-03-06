[![FIWARE Banner](https://fiware.github.io/tutorials.IoT-over-MQTT/img/fiware.png)](https://www.fiware.org/developers)
[![FIWARE Banner](https://github.com/PietroGreco/iotagent-api-adoption/blob/master/docs/img/opcua-logo.png)](https://opcfoundation.org/)

# OPC UA Agent Tutorial

This is a step-by-step tutorial that will introduce in detail how to enable OPC UA to FIWARE connecting an OPC UA server
to Orion Context Broker using the agent. The OPC UA data will be automatically published in a FIWARE Orion Context
Broker using NGSI data model.

## What is OPC UA ?

OPC UA is a well-known client-server protocol used in the Industry.

Usually an OPC UA server is responsible for fetching sensor data from factory-level machinery and make them available to
an OPC UA client (the Agent in this case).

Sensors are mapped to the OPC UA Server Address Space as variables (or attributes). A client can then retrieve their
values. Moreover, it is also possible to control the machinery invoking methods.

In particular, an OPC UA client chooses a set of variables to monitor creating a set of subscriptions, one for each
variable. During the subscription stage the client specifies, through some parameters, how it should receive sensor data
from the server.

In our case OPC UA Agent acts as bridge between the OPC UA server and the Orion Context Broker, behaving as an OPC-UA
client.

## Actors

The actors involved in the scenario are:

-   **OPC UA Server**, representing the data source
-   **OPC UA Agent**, the connector to join industrial environment to FIWARE
-   **Orion Context Broker**, the broker as entry point of FIWARE platform

#### OPC-UA Server

For tutorial purposes, you will use a simple OPC-UA server (source code
[here](https://github.com/Engineering-Research-and-Development/opc-ua-car-server)).

![Car Schema](https://raw.githubusercontent.com/Engineering-Research-and-Development/opc-ua-car-server/master/img/car_schema.png)

It represents a car with the following structure:

-   Car (obj)

    -   Speed (attr)

    -   Accelerate (meth)

-   Stop (meth)

    -   Engine (obj)

        -   Temperature (attr)
        -   Oxygen (attr)

-   Sensors

    -   Any number of user-defined Boolean sensor simulating a square-wave

        For each sensor it is possible to define the semi-period

#### OPC UA Agent

IoT Agent can be configured as described in the
[user guide](https://github.com/Engineering-Research-and-Development/iotagent-opcua/blob/master/docs/user_and_programmers_manual.md).
In order to play with OPC UA server above-mentioned, configuration files are already edited and available in AGECONF
folder.

#### Orion Context Broker

Orion Context Broker can be external, however to have a black box for testing, it will be included in docker compose in
order to have a self-supporting environment.

## Step-by-step Tutorial

In this paragraph we are going to describe how to quickly deploy a working testbed consisting of all the actors: Car,
Agent, Orion Context Broker and the two MongoDB instances.

#### Requirements

-   Docker (Version 19.03.1+)
-   Docker-compose (Version 1.24.1+)

Install docker and docker-compose by following the instructions on the official web site

-   Docker: https://docs.docker.com/install/linux/docker-ce/ubuntu/
-   Docker-Compose: https://docs.docker.com/compose/install/

Once docker has been correctly installed you can continue with the other steps

#### Step 1 - Clone the OPCUA Agent Repository

Open a terminal and move into a folder in which to create the new folder containing the IotAgent testbed.

Then run:

```bash
git clone "https://github.com/Engineering-Research-and-Development/iotagent-opcua"
```

#### Step 2 - Run the testbed

To launch the whole testbed:

```bash
cd iotagent-opcua
docker-compose up -d
```

After that you can run:

```bash
docker ps
```

to check if all the required components are running.

Running the docker environment (using configuration files as is) creates the following situation:
![Docker Containers Schema](https://raw.githubusercontent.com/Engineering-Research-and-Development/iotagent-opcua/master/docs/images/OPC%20UA%20Agent%20tutorial%20Containers.png)

#### Step 3 - Start using the testbed

For the Agent to work an initialization phase is required. During this phase the Agent becomes aware of what variables
and methods are available on OPC UA server side. These information can be provided to the agent by means of a
configuration file (config.json) or through the REST API.

Three different initialization modalities are available:

-   Use a preloaded config.json
-   Invoke a mapping tool responsible of automatically building the config.json
-   Use the REST API

In the following parts of this tutorial we are going to use the REST API (default config.json is empty)

#### Step 4 - Provision a new Device

By Device we mean the set of variables (attributes) and methods available on OPC UA Server side.

To provision the Device corresponding to what the CarServer offers, use the following REST call:

```bash
curl http://localhost:4002/iot/devices \
     -H "fiware-service: opcua_car" \
     -H "fiware-servicepath: /demo" \
     -H "Content-Type: application/json" \
     -d @add_device.json
```

Where add_device.json is the one you find inside iotagent-opcua/API_Server_Tests folder

add_device.json sample payload contains several attributes even of different type. Some of them are missing on the OPC
UA Server side but have been included to prove that the Agent is able to manage such situations.

#### Step 5 - Get devices

Check if the operation gone well, by sending the following REST call:

```bash
curl http://localhost:4002/iot/devices \
     -H "fiware-service: opcua_car" \
     -H "fiware-servicepath: /demo"
```

You should obtain a JSON indicating that there is one device:

[aggiungere estratto corpo JSON]

#### Interlude

You can interact with the CarServer through the Agent in three different ways:

-   **Active attributes** For attributes mapped as **active** the Agent receives in real-time the updated values

-   **Lazy attributes** For this kind of attribute the OPC UA Server is not willing to send the value to the Agent,
    therefore this can be obtained only upon request. The agent registers itself as lazy attribute provider being
    responsible for retrieving it

-   **Commands**

    Through the requests described below it is possible to execute methods on the OPC UA server.

Examples of what has been just illustrated can be found on add_device.json file.

#### Step 6 - Monitor Agent behaviour

Any activity regarding the Agent can be monitored looking at the logs. To view docker testbed logs run:

```bash
cd iotagent-opcua
docker-compose logs -f
```

Looking at these logs is useful to spot possible errors.

#### Step 7 - Accelerate

In order to send the Accelerate command (method in OPC UA jargon), the request has to be sent to Orion that forwards the
request to the OPC UA Agent:

```bash
curl -X PUT \
  'http://localhost:1026/v2/entities/age01_Car/attrs/Accelerate?type=Device' \
  -H 'content-type: application/json' \
  -H 'fiware-service: opcua_car' \
  -H 'fiware-servicepath: /demo'
  -d '{
  "value": [2],
  "type": "command"
}
```

To proof that the method Accelerate is arrived to the device, It is sufficient to evaluate the speed attribute (must be
greater than zero):

```bash
curl -X GET \
  http://localhost:1026/v2/entities/age01_Car/attrs/Speed \
  -H 'fiware-service: opcua_car' \
  -H 'fiware-servicepath: /demo'
```

The OPC UA Agent monitors all attributes defined as active into config.json file, after creation of NGSI entity as
proof:

```bash
curl -X GET \
  http://orion:1026/v2/entities \
  -H 'fiware-service: opcua_car' \
  -H 'fiware-servicepath: /demo'
```

Every value change can be seen into docker-compose logs and performing requests to Orion as:

```bash
curl -X GET \
  http://orion:1026/v2/entities \
  -H 'fiware-service: opcua_car' \
  -H 'fiware-servicepath: /demo'
```

#### Appendix A - Customize the environment

Docker Compose can be downloaded here
[docker-compose.yml](https://github.com/Engineering-Research-and-Development/iotagent-opcua/blob/api_adoption/docker-compose.yml):

```yaml
version: "3"
#secrets:
#   age_idm_auth:
#      file: age_idm_auth.txt

services:
    iotcarsrv:
        hostname: iotcarsrv
        image: pietrogreco4991/opcuacarsrv:1.3-PG_2.0.0-NODEOPCUA
        networks:
            - hostnet
        ports:
            - "5001:5001"

    iotage:
        hostname: iotage
        image: pietrogreco4991/iotagent-opcua:1.1_API_ADOPTION
        networks:
            - hostnet
            - iotnet
        ports:
            - "4002:4002"
            - "4081:8080"
        depends_on:
            - iotcarsrv
            - iotmongo
            - orion
        volumes:
            - ./AGECONF:/opt/iotagent-opcua/conf
        command: /usr/bin/tail -f /var/log/lastlog

    iotmongo:
        hostname: iotmongo
        image: mongo:3.4
        networks:
            - iotnet
        volumes:
            - iotmongo_data:/data/db
            - iotmongo_conf:/data/configdb

    ################ OCB ################

    orion:
        hostname: orion
        image: fiware/orion:latest
        networks:
            - hostnet
            - ocbnet
        ports:
            - "1026:1026"
        depends_on:
            - orion_mongo
        #command: -dbhost mongo
        entrypoint:
            /usr/bin/contextBroker -fg -multiservice -ngsiv1Autocast -statCounters -dbhost mongo -logForHumans -logLevel
            DEBUG -t 255

    orion_mongo:
        hostname: orion_mongo
        image: mongo:3.4
        networks:
            ocbnet:
                aliases:
                    - mongo
        volumes:
            - orion_mongo_data:/data/db
            - orion_mongo_conf:/data/configdb
        command: --nojournal

volumes:
    iotmongo_data:
    iotmongo_conf:
    orion_mongo_data:
    orion_mongo_conf:

networks:
    hostnet:
    iotnet:
    ocbnet:
```

By modifying this file you can:

-   Change exposed ports
-   Extend the stack with other services (e.g. Cygnus)

#### Customize CarServer

Coming soon ... How to add new sensors

#### Customize Agent

Below, a copy of config.properties file is included.

This file contains all the information required to establish connections to othe components. You find OCB hostname and
port, Agent port, Device Registry Mo

-   OCB hostname and port
-   Agent port
-   Information about a MongoDB instance that could be used as Device Registry (depending whether you have specified
    "memory" or "mongo" as device-registry-type)
-   OPC UA Server endpoint to which establish a connection
-   OPC UA Session and Monitoring parameters
-   Security Policies

```yaml
namespace-ignore=2,7

context-broker-host=orion
context-broker-port=1026
server-base-root=/
server-port=4002
device-registry-type=memory
mongodb-host=iotmongo
mongodb-port=27017
mongodb-db=iotagent
mongodb-retries=5
mongodb-retry-time=5
fiware-service=opcua_car
fiware-service-path=/demo
provider-url=http://iotage:4002
device-registration-duration=P1M
endpoint=opc.tcp://iotcarsrv:5001/UA/CarServer
log-level=DEBUG
#DATATYPE MAPPING OPCUA --> NGSI
OPC-datatype-Number=Number
OPC-datatype-Decimal128=Number
OPC-datatype-Double=Number
OPC-datatype-Float=Number
OPC-datatype-Integer=Integer
OPC-datatype-UInteger=Integer
OPC-datatype-String=Text
OPC-datatype-ByteString=Text
#END DATATYPE MAPPING OPCUA --> NGSI

#SESSION PARAMETERS
requestedPublishingInterval=10
requestedLifetimeCount=1000
requestedMaxKeepAliveCount=10
maxNotificationsPerPublish=100
publishingEnabled=true
priority=10

#MONITORING PARAMETERS
samplingInterval=1
queueSize=10000
discardOldest=false

#SERVER CERT E AUTH
securityMode=1
securityPolicy=0
userName=
password=

#securityMode=SIGNANDENCRYPT
#securityMode=1Basic256
#password=password1
#userName=user1

#api-ip=192.168.13.153

#Administration Services
api-port=8080
#End Administration Services

#POLL COMMANDS SETTINGS
polling=false
polling-commands-timer=30000
pollingDaemonFrequency=20000
pollingExpiration=200000
#END POLL COMMANDS SETTINGS

#AGENT ID
agent-id=age01_
entity-id=age01_Car

#CONFIGURATION
configuration=api
#CHECK TIMER POLLING DEVICES
checkTimer=2000
```

A recap of the most important properties:

```yaml
# OCB
context-broker-host=<ORIONHOSTIP>
context-broker-port=<ORIONPORT>

# Needed to identify Devices
fiware-service=<SERVICE>
fiware-service-path=<SERVICE-PATH>

# OPC UA Server endpoint
endpoint=opc.tcp://<IPADDR>:<PORT>

agent-id=<PREFIX>
```
