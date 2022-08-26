# go-carbon, gostatsd, grafana with docker-compose

### What is this about?

This repository provides docker-compose file for collection and a combination of metric tools including grafana, go-carbon, carbonapi and gostatsd (go version of graphite and statsd). Contributions are very welcome! Feel free to create a pull request or issue about how to improve it.

### Components

| component     | link                                      | version   | license               |
|---------------|-------------------------------------------|-----------|-----------------------|
| gocarbon      | https://github.com/go-graphite/go-carbon  | 0.16.2    | MIT                   |
| carbonapi     | https://github.com/go-graphite/carbonapi  | 0.15.6    | MIT                   |
| gostatsd      | https://github.com/atlassian/gostatsd     | 35.1.0    | MIT                   |
| grafana       | https://github.com/grafana/grafana        | latest    | GNU Affero GPL v3.0   |

### Architecture


    ┌───────────────────────┐
    │                       │
    │                       │
    │                       │
    │    grafana            │
    │    public port 3000   │
    │    web interface      │
    │                       │
    │                       │               ┌─────────────────────┐
    │                       ├─────────────► │                     │
    └───────────────────────┘ requests      │                     │
                                            │                     │
                                            │    carbonapi        │
                                            │    private port 8081│
                                            │    carbon api       │
    ┌───────────────────────┐               │                     │
    │                       │               │                     │
    │                       │ ◄─────────────┤                     │
    │                       │  requests     └─────────────────────┘
    │    gocarbon           │
    │    public port 2003   │
    │    carbon receiver    │
    │    metric storage     │
    │                       │
    │                       │
    └───────────────────────┘
                      ▲
                      │
                sends │
                      │
    ┌─────────────────┴─────┐
    │                       │
    │                       │
    │                       │
    │    gostatsd           │
    │    public port 8125   │
    │    statsd receiver    │
    │    aggregate and send │
    │                       │
    │                       │
    └───────────────────────┘

### Configuration and storage behaviour

The configuration files come with sensible defaults and are pre-configured for running components together, which they're battle-tested on production since 2019.

go-carbon and gostatsd were configured to store and aggregate the metrics with 10 seconds precision. You can change the frequency and history behaviour by configuring `storage-schemas.conf` file under the `config/carbon` directory.

Let's take a look at the existing configuration:

    # storage-schemas.conf
    10s:4d,1m:30d,1h:1y,1d:5y

This means 10 seconds for 4 days, 1 minute for 30 days, 1 hour for 1 year and 1 day for 5 years resulting in 1.01 Megabytes per metric (The first two high-precision archive takes up most of the space). If you're planning to use a different configuration, you can use [whisper calculator](https://m30m.github.io/whisper-calculator/) to calculate how much disk size will be used by carbon. Also, [graphite documentation](https://graphite.readthedocs.io/en/latest/config-carbon.html#storage-schemas-conf) has good tips about how to define retention rates in the right way.

Keep in mind that

* Changing this file will not affect already-created .wsp files.
* Carbon implements a fixed-size database. This means time-slots within an archive take up space whether or not a value is stored.

If you're planning to use a different precision rather than 10 seconds, you also have to change the following configuration in the `gostatsd.toml` file under the `config/gostatsd` directory.

    # gostatsd.toml
    flush-interval='10s'

### Running

Run the following command to kick the containers

    docker compose up -d

#### Adding the new datasource in grafana

* Login to grafana (port 3000)
* Go to Configuration > Data sources
* Add a new graphite Datasource with the following configuration
    * HTTP > URL: http://carbonapi:8081
    * Graphite details > Version: 1.1.x
* Save & test

### HA ideas

This setup is not designed to be highly available. The possible improvements can be:

* Adding [carbon-c-relay](https://github.com/grobian/carbon-c-relay) or [carbon-relay-ng](https://github.com/grafana/carbon-relay-ng) to the front of the gocarbon(s).
* gostatsd can be configured as a forwarder for running in front of the gostatsd(s).
* gocarbon supports receiving metrics from kafka and google pubsub.
* [carbon-clickhouse](https://github.com/go-graphite/graphite-clickhouse)

