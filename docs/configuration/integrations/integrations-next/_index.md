+++
title = "(Beta) Integrations Revamp"
weight = 100
+++

# (Beta) Integrations Revamp

Release v0.22.0 of Grafana Agent includes experimental support for a revamped
integrations subsystem. The integrations subsystem is the second oldest part of
Grafana Agent, and has started to feel out of place as we built out the
project.

The revamped integrations subsystem can be enabled by passing
`integrations-next` to the `-enable-features` command line flag. As an
experimental feature, there are no stability guarantees, and it may receive a
higher frequency of breaking changes than normal.

The revamped integrations subsystem has the following benefits over the
original subsystem:

* Integrations can opt in to supporting multiple instances. For example, you
  may now run any number of `redis_exporter` integrations, where before you
  could only have one per agent. Integrations such as `node_exporter` still
  only support a single instance, as it wouldn't make sense to have multiple
  instances of those.

* Autoscrape (previously called "self-scraping"), when enabled, now supports
  sending metrics for an integration directly to a running metrics instance.
  This allows you configuring an integration to send to a specific Prometheus
  remote_write endpoint.

* A new service discovery HTTP API is included. This can be used with
  Prometheus' [http_sd_config][http_sd_config]. The API returns extra labels
  for integrations that previously were only availble when autoscraping, such
  as `agent_hostname`.

* Integrations that aren't Prometheus exporters may now be added, such as
  integrations that generate logs or traces.

[http_sd_config]: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#http_sd_config

## Config changes

The revamp contains a number of breaking changes to the config. The schema of the
`integrations` key in the config file is now the following:

```yaml
integrations:
  # Controls settings for integrations that generate metrics.
  metrics:
    # Controls default settings for autoscrape. Individual instances of
    # integrations inherit the defaults and may override them.
    autoscrape:
      # Enables autoscrape of integrations.
      [enable: <boolean> | default = true]

      # Specifies the metrics instance name to send metrics to. Instance
      # names are located at metrics.configs[].name from the top-level config.
      # The instance must exist.
      #
      # As it is common to use the name "default" for your primary instance,
      # we assume the same here.
      [metrics_instance: <string> | default = "default"]

      # Autoscrape interval and timeout. Defaults are inherited from the global
      # section of the top-level metrics config.
      [scrape_interval: <duration> | default = <metrics.global.scrape_interval>]
      [scrape_timeout: <duration> | default = <metrics.global.scrape_timeout>]

  # Override settings for agent to self-communivate for autoscrape. This is
  # currently required if you are using TLS for the agent server. This field is
  # temporary and will be removed in the near future once autoscrape can work #
  # without using the network.
  #
  # Settings are omitted for brevity, but the schema is from:
  # https://github.com/prometheus/common/blob/2af6d036253eee1a9a08c6ddf6be6d67537bcdff/config/http_config.go#L177
  client_config:
    # <settings omitted>

  # Configs for integrations which do not support multiple instances.
  [agent: <agent_config>]
  [cadvisor: <cadvisor_config>]
  [node_exporter: <node_exporter_config>]
  [process_exporter: <process_exporter_config>]
  [statsd_exporter: <statsd_exporter_config>]
  [windows_exporter: <windows_exporter_config>]

  # Configs for integrations that do support multiple instances. Note that
  # these must be arrays.
  consul_exporter_configs:
    [- <consul_exporter_config> ...]

  dnsmasq_exporter_configs:
    [- <dnsmasq_exporter_config> ...]

  elasticsearch_expoter_configs:
    [- <elasticsearch_expoter_config> ...]

  github_exporter_configs:
    [- <github_exporter_config> ...]

  kafka_exporter_configs:
    [- <kafka_exporter_config> ...]

  memcached_exporter_configs:
    [- <memcached_exporter_config> ...]

  mongodb_exporter_configs:
    [- <mongodb_exporter_config> ...]

  mysqld_exporter_configs:
    [- <mysqld_exporter_config> ...]

  postgres_exporter_configs:
    [- <postgres_exporter_config> ...]

  redis_exporter_configs:
    [- <redis_exporter_config> ...]
```

## Integrations changes

Integrations no longer support an `enabled` field; they are enabled by being
defined in the YAML. To disable an integration, comment it out or remove it.

Metrics-based integrations now use this common set of options:

```yaml
# Provide an explicit value to uniquely identify this instance of the
# integration. If not provided, a reasonable default will be inferred based
# on the integration.
#
# The value here must be unique across all instances of the same integration.
[instance: <string>]

# Override autoscrape defaults for this integration.
autoscrape:
  # Enables autoscrape of integrations.
  [enable: <boolean> | default = <integrations.metrics.autoscrape.enable>]

  # Specifies the metrics instance name to send metrics to.
  [metrics_instance: <string> | default = <integrations.metrics.autoscrape.metrics_instance>]

  # Autoscrape interval and timeout.
  [scrape_interval: <duration> | default = <integrations.metrics.autoscrape.scrape_interval>]
  [scrape_timeout: <duration> | default = <integrations.metrics.autoscrape.scrape_timeout>]
```

The old set of common options have been removed and do not work when the revamp
is being used:

```yaml
# OLD SCHEMA: NO LONGER SUPPORTED

[enabled: <boolean> | default = false]
[instance: <string>]
[scrape_integration: <boolean> | default = <integrations_config.scrape_integrations>]
[scrape_interval: <duration> | default = <global_config.scrape_interval>]
[scrape_timeout: <duration> | default = <global_config.scrape_timeout>]
[wal_truncate_frequency: <duration> | default = "60m"]
relabel_configs:
  [- <relabel_config> ...]
metric_relabel_configs:
  [ - <relabel_config> ...]
```