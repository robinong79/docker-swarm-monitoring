# docker-swarm-monitoring
Monitoring Docker Swarm with Prometheus and ELK stack.

This repository describes and publishes our setup of monitoring a Docker Swarm with the help of the ELK repository and Prometheus with it's scrapers.

## Important note regarding Prometheus 2.x.x
As of version 2.x.x. of Prometheus they changed the user to nobody. This means that if you use a persistent data directory is has to be chmod'ed to 777.
In this example this will be the directory /var/dockerdata/prometheus/data.

If you use this repo i.c.m. with the v.1.8.2 yml files you can leave the user as it is. Just make sure to copy and rename the prometheus.v1.8.2.yml file from configs to prometheus.yml.

## Prerequisites

- Ubuntu (16.04 or higher) or RHEL host(s)
- Docker v1.13.1 (minimum)
    - Experimental Mode must be set to true (to be able to use "docker deploy" with compose v3 files)
    - Must run in Swarm Mode
    - 2 overlay networks ("monitoring" and "logging")

## Used components

We have split up the monitoring into 2 basic parts:

#### Monitoring Stack

| Service | Purpose |
| ------ | ----- |
| [Prometheus](https://hub.docker.com/r/prom/prometheus/) | Central Metric Collecting |
| [CAdvisor](https://hub.docker.com/r/google/cadvisor/) | Collecting Container information  |
| [Node-Exporter](https://hub.docker.com/r/basi/node-exporter/) | Collecting Hardware and OS information |
| [AlertManager](https://hub.docker.com/r/prom/alertmanager/) | Sending out alerts raised from Prometheus |
| [Grafana](https://hub.docker.com/r/grafana/grafana/) | Dashboard on top of Prometheus |

#### Logging Stack

| Service | Purpose |
| ------ | ----- |
| [ElasticSearch](https://hub.docker.com/_/elasticsearch/) | Central storage for Logdata |
| [LogStash](https://hub.docker.com/_/logstash/) | Log formatter and processing pipeline |
| [ElastAlert](https://hub.docker.com/r/ivankrizsan/elastalert/) | Sending out alerts raised on Logs |
| [Kibana](https://hub.docker.com/_/kibana/) | Dashboard on top of Elasticsearch |

## Schema of the stacks
![stackflow](https://raw.githubusercontent.com/robinong79/docker-swarm-monitoring/master/Monitoring_Logging_Stack.png "Monitoring Logging Stack")

## Preparation

Host setting for ElasticSearch (Look [here](https://www.elastic.co/guide/en/elasticsearch/reference/5.0/vm-max-map-count.html) for more information)
```
$ sysctl -w vm.max_map_count=262144
```

Changing mod of persistent data directory (Only for Prometheus 2.x.x)


```
$ chmod 777 /var/dockerdata/prometheus/data
```

#### Docker

```
$ docker swarm init
$ docker network create -d overlay monitoring
$ docker network create -d overlay logging
```

#### Compose files

Make sure to look at the compose files for the volume mappings.
In this example everything is mapped to /var/dockerdata/<servicename>/<directories>. Adjust this to your own liking or create the same structure as used in this example.

#### Config Files

| Config file | Needs to be in <Location> | Remarks |
| ----- | ----- | ----- | 
| alertmanagerconfig.yml | /var/dockerdata/alertmanager/ | The alerts go through Slack. Use your Slack Key and channel name for it to work |
| elastalert_supervisord.conf | /var/dockerdata/elastalert/config | - |
| elastalertconfig.yaml | /var/dockerdata/elastalert/config | - |
| prometheus.yml | /var/dockerdata/prometheus | - |

#### Alert Files

| Alert file | Needs to be in <Location> | Remarks |
| ----- | ----- | ----- | 
| alertrules.nodes | /var/dockerdata/prometheus/rules | - |
| alertrules.task | /var/dockerdata/prometheus/rules | - |
| elastrules.error.yaml| /var/dockerdata/elastalert/rules | The alerts go through Slack. Use your Slack Key and channel name for it to work |


## Installation

#### Logging Stack

```
$ docker deploy --compose-file docker-compose-logging.yml logging
```

#### Monitoring Stack

```
$ docker deploy --compose-file docker-compose-monitoring.yml monitoring
```

#### Container/Service logging to Logstash

In order to get the logs from the services/containers to Logstash you need to start them with a different logdriver.

Compose file:

```
logging:
    driver: gelf
    options:
        gelf-address: "udp://127.0.0.1:12201"
        tag: "<name of container for filtering in elasticsearch>" 
```

Run command:

```
$ docker run \
         --log-driver=gelf \
         --log-opt gelf-address=udp://127.0.0.1:12201 \
         --log-opt tag="<name of container for filtering in elasticsearch>" \
         ....
         ....
```         

## Updates

| Date | Update |
| ----- | ----- | 
| 23-02-2018 | Updated repo to work with Prometheus version 2.1.0 |


## Credits and License

Basilio Vera's repo's (https://hub.docker.com/u/basi/) have been used for information. This got me a long way with building up a monitoring stack.
Also using his version of Node-Exporter and some alert files so we have access to HOST_NAME and some startup alerts.
He made a really nice Grafana Dashboard too which we used as a base. You can check it out here (https://grafana.net/dashboards/609).

The files are free to use and you can redistribute it and/or modify it under the terms of the GNU Affero General Public License as published by the Free Software Foundation.