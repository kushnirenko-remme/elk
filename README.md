# System resources

This repository installs ELK stack (metricbeat, kibana, elasticsearch) on the local host and creates dashboard with system resources.
 
## Pre-requisites

1. Docker version > v17.07.0
1. Docker-Compose > 1.15.0
1. Ensuring the following ports are free on the host, as they are mounted by the containers:

    - `5601` (Kibana)
    - `9200` (Elasticsearch)
    
1. Atleast 4Gb of available RAM

The example file uses docker-compose v2 syntax.

**We assume prior knowledge of docker**

## Versions

All Elastic Stack components are version 7.2.0

## Architecture 

Metricbeats (send metrics to Elasticsearch) ----->  Elasticserach <----- Kibana (gets indexes and shows dashboard) 

Summarising the above, the following containers are deployed:

* `Elasticsearch`
* `Kibana`
* `Metricbeat` - Monitors the host system with respect cpu, disk, memory and network. 

In addition to the above containers, a `configure_stack` container is deployed at startup.  This is responsible for:

* Setting the Elasticsearch passwords
* Importing dashboard

## Modules & Data

The following Beats module is utilised in this stack example to provide data and dashboards:

1. Metricbeat
    - `system` module with `core`,`cpu`,`load`,`diskio`,`filesystem`,`fsstat`,`memory`,`network`,`process`,`socket`
    
## Step by Step Instructions - Deploying the Stack

1. Clone the repository


    ```shell
    git clone https://github.com/kushnirenko-remme/elk
    ```

1. Navigate into the elk folder and issue the following command. 

    ```shell
    cd elk
    sudo docker-compose -f docker-compose-linux.yml up -d
    ```

**The above command may take some time if you don't have the base centos7 images**

1. Confirm the containers are available, by issuing the following command:
    
    ```shell
    docker ps -a
    ```

    this will return a response such as the following:

    ```shell
    CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS                     PORTS                                          NAMES
    076e40495ef8        docker.elastic.co/beats/metricbeat:7.2.0              "metricbeat -e -E ..."   4 minutes ago       Up 4 minutes                                                              metricbeat
    946a5034e37a        docker.elastic.co/beats/metricbeat:7.2.0              "/bin/bash -c 'cat..."   5 minutes ago       Exited (0) 4 minutes ago                                                  configure_stack
    c6173ed51209        docker.elastic.co/kibana/kibana:7.2.0                 "/bin/sh -c /usr/l..."   5 minutes ago       Up 5 minutes (healthy)     127.0.0.1:5601->5601/tcp                       kibana
    198afa6b6325        docker.elastic.co/elasticsearch/elasticsearch:7.2.0   "/bin/bash bin/es-..."   5 minutes ago       Up 5 minutes (healthy)     127.0.0.1:9200->9200/tcp, 9300/tcp             elasticsearch
    ```
    
    Whilst the container ids will be unique, other details should be similar. Note the `configure_stack` container will have exited on completion of the configuration of stack.  This occurs before the beat containers start.  Other containers should be "Up".

1. On confirming the stack is started, navigate to kibana at http://localhost:5601/app/kibana#/dashboard/79ffd6e0-faa0-11e6-947f-177f697178b8-ecs.  Assuming you haven't changed the default password, see [Customising the Stack](TODO), the default credentials of `elastic` and `changeme` should apply.

1. Navigate to the dashboard view. Open any of the dashboards listed as having data below. The following shows the Metricbeat-Docker dashboard.

![Metricbeat Docker Dashboard](https://user-images.githubusercontent.com/12695796/29227415-a3413aec-7ecd-11e7-8824-cfc48982b124.png)

## Dashboards with data

The following dashboards are accessible and populated. Other dashboards, whilst loaded, will not have data due to the absence of an appropriate container e.g. Packetbeat Cassandra.

* Metricbeat filesystem per Host
* Metricbeat system overview
* Metricbeat-cpu
* Metricbeat-filesystem
* Metricbeat-memory
* Metricbeat-network
* Metricbeat-overview
* Metricbeat-processes
* Packetbeat Dashboard (limited)
* Packetbeat Flows
* Packetbeat HTTP

## Technical notes

The following summarises some important technical considerations:

1. The Elasticsearch instances uses a named volume `esdata` for data persistence between restarts. It exposes HTTP port 9200 for communication with other containers. 
1. Environment variable defaults can be found in the file .env`
1. The Elasticsearch container has its memory limited to 1g. This can be adjusted using the environment parameter `ES_MEM_LIMIT`. Elasticsearch has a heap size of 1g. This can be adjusted through the environment variable `ES_JVM_HEAP` and should be set to 50% of the `ES_MEM_LIMIT`.  **Users may wish to adjust this value on smaller machines**.
1. The Elasticsearch password can be set via the environment variable `ES_PASSWORD`. This sets the password for the `elastic` and `kibana` user.
1. The Kibana container exposes the port 5601.
1. All configuration files can be found in the extracted folder `./config`.
1. The Metricbeat container mounts both `/proc` and `/sys/fs/cgroup` on linux.  This allows Metricbeat to use the `system` module report on disk, memory, network and cpu of the host. 
1. On systems with POSIX file permissions, all Beats configuration files are subject to ownership and file permission checks. The purpose of these checks is to prevent unauthorized users from providing or modifying configurations that are run by the Beat.  The owner of the configuration file must be either root or the user who is executing the Beat process. The permissions on the file must disallow writes by anyone other than the owner.  As we mount our configurations from the host, where the user is likely different than that used to run the container and the beat process, we disable this check for all beats with  -strict.perms=false.

## Customising the Stack

With respect to the current example, we have provided a few simple entry points for customisation:

1. The example includes an .env file listing environment variables which alter the behaviour of the stack.  These environment variables allow the user to change:
    * `ELASTIC_VERSION` - the Elastic Stack version (default 7.2.0) 
    * `ES_PASSWORD` - the password used for authentication with the elastic user. This password is applied for all system users i.e. kibana and logstash_system. Defaults to “changeme”.
    * `DEFAULT_INDEX_PATTERN` - The index pattern used as the default in Kibana. Defaults to “metricbeat-*”.
    * `ES_MEM_LIMIT` - The memory limit used for the Elasticsearch container. Defaults to 1g. Consider reducing for smaller machines.
    * `ES_JVM_HEAP` - The Elasticsearch JVM heap size. Defaults to 1024m and should be set to half of the ES_MEM_LIMIT.
1. Modules and Configuration - All configuration to the containers is provided through a mounted “./config” directory.  Where possible, this exploits the dynamic configuration loading capabilities of Beats. For example, an additional module could be added by simply adding a file to the directory “./config/beats/metricbeat/modules.d/” in the required format.

## Shutting down the stack

`sudo docker-compose -f docker-compose-linux.yml stop`

will exit the containers and ensure they are shut down gracefully. To remove all containers, including their mounted named volumes:

```shell
sudo docker-compose -f docker-compose-linux.yml down -v
```
