# MVISION EDR - ACTIVITY FEED [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![build status](https://travis-ci.org/mcafee/mvision-edr-activity-feed.svg?branch=master)](https://travis-ci.org/mcafee/mvision-edr-activity-feed)


## OVERVIEW
- Open Source ActivityFeed integrated with OpenDXL streaming client (https://github.com/opendxl/opendxl-streaming-client-python).
- ``mvision-edr-activity-feed`` is a command line tool to consume and subscribe to events from MVISION EDR


## INSTALL
Run either:
.. code:: shell

    python setup.py install
   
   
## COMMAND LINE USAGE

In order to use the CLI, you need credentials in MVEDR.
The CLI has several parameters (as described with
``mvision-edr-activity-feed  -h``):

::

    usage: mvision-edr-activity-feed  [-h] [--version] --url URL --username
                                      USERNAME --password PASSWORD --module MODULE
                                      [--wait WAIT]
                                      [--loglevel {critical,error,warning,info,debug,notset}]
                                      [--logfile LOGFILE] [--reset]
                                      [--consumer-group CONSUMER_GROUP]
                                      [--config CONFIG]

    MVISION EDR Event Dispatcher CLI

    optional arguments:
      -h, --help            show this help message and exit
      --version             show program's version number and exit
      --url URL             Base URL for MVISION EDR (default: None)
      --username USERNAME   Username for the service (default: None)
      --password PASSWORD   Password for the service (default: None)
      --module MODULE       Module to register events (default: None)
      --wait WAIT           Time to wait between queries (default: 5)
      --loglevel {critical,error,warning,info,debug,notset}
      --logfile LOGFILE
      --topic TOPIC         Topic to subscribe to (default: ['case-mgmt-events',
                            'BusinessEvents', 'threatEvents'])
      --reset               Starts the consumer group from the earliest event
                            available. This only works when initializing the
                            consumer group the first time (default: False)
      --consumer-group CONSUMER_GROUP
                            Name for the consumer group to use (default:
                            mcafee_investigator_events)
      --config CONFIG       Configuration key/value pairs for callbacks (default:
                            None)


## SUBSCRIPTIONS

You need to provide at least one module with your subscriptions for the
CLI to load. The following is an sample subscription:

.. code:: python

    import logging
    from mvision_edr_activity_feed import subscribe


    @subscribe(entity='case', subtype='creation')
    def case_created(event):
        logging.info("CASE CREATED: %s", event)


    @subscribe(entity='case', subtype='priority-update')
    def case_updated(event):
        logging.info("CASE PRIORITY UPDATED: %s", event)


    @subscribe(entity='case', subtype='status-update')
    def case_status_updated(event):
        logging.info("CASE STATUS UPDATED: %s", event)


    @subscribe("user == 'some_user'")
    def custom_subscription(event):
        logging.info("EVENT FOR USER 'some_user': %s", event)

In the first three examples, we are subscribing to the following events: **Case
creation**, **Case priority updates**, and **Case status updates**. On the
last example we are subscribing to events that have a property ``user`` with a
value of ``some_user`` (as defined by the corresponding `JMESPath <http://jmespath.org/>`_ expression)

Note that there are two ways to subscribe to events:

* **Basic**: This is for events that follow out `Event Specification <EVENT_SPEC.rst>`__

    * **entity**: The entity affected by the event.
    * **subtype**: The event subtype, related to the entity.

* **Advanced**: This is for generic events, and uses a `JMESPath <http://jmespath.org/>`_ expression to determine the subscription


## CONFIGURE RSYSLOG IN CASE OF REMOTE LOGGING

In case of using rsyslog for remote logging please follow the documentation explained here: https://www.tecmint.com/setup-rsyslog-client-to-send-logs-to-rsyslog-server-in-centos-7/

rsyslog.conf that can be used as an example: https://github.com/mcafee/mvision-edr-activity-feed/blob/feature/activity_feed_integration/rsyslog.conf

## RUNNING THE EXAMPLES

There are a couple of simple examples that will log event information to
the console. These are executed as follows:

.. code:: shell

    mvision-edr-activity-feed --url https://api-int-cop.soc.mcafee.com/ --username YOUR_USERNAME --password YOUR_PASSWORD --module samples.generic --loglevel=debug

.. code:: shell

    mvision-edr-activity-feed --url https://api-int-cop.soc.mcafee.com/ --username YOUR_USERNAME --password YOUR_PASSWORD --module samples.individual --loglevel=debug

You can also mix several modules in a single call:

.. code:: shell

    mvision-edr-activity-feed --url https://api-int-cop.soc.mcafee.com/ --username YOUR_USERNAME --password YOUR_PASSWORD --module samples.generic --module samples.individual --loglevel=debug

## PARSING RULE ON McAfee SIEM

For convenience a `Docker` image is provided. For running `MVISION EDR` activity feed client and forward threat events to `McAfee ESM` via `syslog`, follow instructions below.

### BUILDING DOCKER IMAGE

To forward events gather from the cloud, a `rsyslog` daemon will run inside the `Docker` container. To reduce the number of events sent to the `ESM` receiver, a filter is applied to discard all logs that doesn't contain `"Threat Detection Summary"` string. All other events will be forwarded to the `ESM` receiver (see [`Dockerfile`](./Dockerfile#22)). `ESM` reciever IP must be provided when building the `Docker` image and cannot be changed later. That means if you need to change the receiver IP, the `Docker` image must be rebuilt.

```bash
docker build --rm --build-arg esm_ip=<ESM_RECEIVER_IP> -t mvision-edr-activity-feed .
```

If you are behind a proxy, add the following parameter while building the image:

```bash
--build-arg HTTP_PROXY="<PROXY_URL_OR_IP:PROXY_PORT>" \
--build-arg HTTPS_PROXY="<PROXY_URL_OR_IP:PROXY_PORT>" \
--build-arg http_proxy="<PROXY_URL_OR_IP:PROXY_PORT>" \
--build-arg http_proxys="<PROXY_URL_OR_IP:PROXY_PORT>" 
```

### RUNNING DOCKER IMAGE

As mentioned before, the `Docker` container spins it's own `rsyslog` daemon. To access `MVISION EDR` resources on the cloud, `username` and `password` must be provided.

*Note*: using a service account is advised.

```
docker run mvision-edr-activity-feed \
    --url https://api.soc.mcafee.com \
    --username user \
    --password pass \
    --module esm11 \
    --loglevel debug

```

If you are behind a proxy, add the following parameter:

```
--env HTTPS_PROXY="<PROXY_URL_OR_IP:PROXY_PORT>"
```

## Setup ESM Datasource

An `ESM` data source holds the location and connection information of your network's sources of data. It acts as a connector to your source of data. To instruct `ESM` to parse `MVISION EDR` threat events an `Advanced Syslog Parser` rule is provided (see [sample rule](./RULE_MVISION_EDR_THREAT.xml)). More information can be found at [McAfee Knowledge Center](https://kc.mcafee.com/corporate/index?page=content&id=PD26993).

### How to setup ESM for parsing MVISION EDR Threat events

* Go to `Policy Editor`. On the menu bar go to `File`, `Import`, `Rules` and click on `Import Rules`. A pop-up windows will prompt.
* Upload `RULE_MVISION_EDR_THREAT.xml` policy.
* Go back to the `Policy Editor` window, select `Advanced Syslog Parser`, and filter the list to locate the rule uploaded on the previous step (i.e. using `Signature ID` or `Name`). Make sure the rule is enabled.
* Disable aggregation (go to Datasources). Make sure rollout policy.
* Rollout the rule if needed (top right corner).

### Add client data sources

* From the `McAfee ESM` dashboard, click the hamburger menu and select `Configuration`.
* On the system navigation tree, select the Receiver, then click the `Properties` icon (small cog wheel).
* Click `Data Sources`, the `Add`.
* `Data Source Vendor` should be `Generic`, and select `Advanced Syslog Parser` as `Data Source Model`.
* Choose a `Name` and provide the `IP` address from which the client will run.
* Make sure port is `514` (`TLS` can be enabled but the port should be changed on the `Dockerfile` as well).
* Click `OK` and `Write` the data source settings to the receiver.


## BUGS AND FEEDBACK

For bugs, questions and discussions please use the
[GitHub Issues](https://github.com/mcafee/mvision-edr-activity-feed/issues).


## LICENSE

Copyright 2019, McAfee LLC

Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed
under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

