# go-carbon, gostatsd, grafana, telegraf with docker-compose

### What is this about?

This repository provides a Docker Compose file for collection and a combination of metric tools including Grafana, telegraf, go-carbon, carbonapi, and gostatsd (Go version of Graphite and Statsd). Contributions are very welcome! Feel free to create a pull request or issue about how to improve it.

### Why go-graphite?

* Minimal configuration required
* Consistent and predictable disk consumption
* Efficient in terms of CPU and memory usage
* Well-supported by Grafana
* Integrates smoothly with existing tools

### Components

| component     | link                                      | version   | license               |
|---------------|-------------------------------------------|-----------|-----------------------|
| gocarbon      | https://github.com/go-graphite/go-carbon  | 0.17.0    | MIT                   |
| carbonapi     | https://github.com/go-graphite/carbonapi  | 0.16.0    | MIT                   |
| gostatsd      | https://github.com/atlassian/gostatsd     | 35.2.1    | MIT                   |
| grafana       | https://github.com/grafana/grafana        | latest    | GNU Affero GPL v3.0   |
| telegraf      | https://github.com/influxdata/telegraf    | latest    | MIT                   |


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

The configuration files come with sensible defaults and are pre-configured for running components together, which they’ve been battle-tested on production since 2019.

go-carbon and gostatsd were configured to store and aggregate the metrics with 10-second precision. You can change the frequency and history behaviour by configuring the `storage-schemas.conf` file under the `config/carbon` directory.

Let's take a look at the existing configuration:

    # storage-schemas.conf
    10s:8h,1m:7d,5m:30d,1h:1y,1d:5y

This means 10 seconds for 8 hours, 1 minute for 7 days, 5 minutes for 30 days, 1 hour for 1 year and 1 day for 5 years, resulting in 377 kilobytes per metric (The first two high-precision archives take up most of the space). If you're planning to use a different configuration, you can use [whisper calculator](https://m30m.github.io/whisper-calculator/) to calculate how much disk size will be used by Carbon. Also, [graphite documentation](https://graphite.readthedocs.io/en/latest/config-carbon.html#storage-schemas-conf) has good tips about how to define retention rates in the right way.

Keep in mind that

* Changing this file will not affect already-created .wsp files.
* Carbon implements a fixed-size database. This means time-slots within an archive take up space whether or not a value is stored.

If you're planning to use a different precision rather than 10 seconds, you also have to change the following configuration in the `gostatsd.toml` file under the `config/gostatsd` directory.

    # gostatsd.toml
    flush-interval='10s'

### Running

Don't forget to change the hostname of Telegraf containers, and the Graphite hostname if you're running on a different host. 

    telegraf:
      hostname: "luminary" # Don't forget to customize this one.
      environment:
        - GRAPHITE_URL=127.0.0.1:2003 # If you’re running separately, use the correct hostname or IP address.

Run the following command to kick the containers

    # Give necessary permissions before running docker compose
    chmod -R a+rw storage/
    docker compose up -d

### Running Telegraf separately in the different hosts

There is a separate `docker-compose-telegraf.yml` file for running Telegraf in the different hosts. You should customize the hostname and Graphite URL in the environment variables. You also need to copy the `config/telegraf/telegraf.conf` file. You can run with the following: 

    docker compose -f docker-compose-telegraf.yml up -d

### Future ideas

This setup is not designed to be highly available. The possible improvements can be:

* Adding [carbon-c-relay](https://github.com/grobian/carbon-c-relay) or [carbon-relay-ng](https://github.com/grafana/carbon-relay-ng) to the front of the gocarbon(s). 
* gostatsd can be configured as a forwarder for running in front of the gostatsd(s).
* telegraf statsd plugin can replace the centralized gostatsd
* gocarbon supports receiving metrics from kafka and google pubsub.
* [carbon-clickhouse](https://github.com/go-graphite/graphite-clickhouse) as a storage backend
