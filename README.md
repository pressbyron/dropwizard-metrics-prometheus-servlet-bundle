# Configurable Prometheus Servlet Bundle for Dropwizard
Adds support for exposing Dropwizard metrics as Prometheus compatible metrics through a dedicated servlet.

Using your applications configuration file it is easy to customize metrics and map metric names to a more user friendly version including support for labels.

## Information
This bundle makes it easy to expose metrics from a Dropwizard application in a Prometheus compatible format. Through the applications configuration file the name of the exposed metrics can be changed and labels added for each metric independently.

Internally the official [Prometheus JVM Client](https://github.com/prometheus/client_java) implementation is being used to map the Dropwizard metrics as well as to expose the metrics through a dedicated servlet.
Due to an [open issue](https://github.com/prometheus/client_java/issues/518) a few classes have been copied over and altered to allow more flexible match patterns.

Information regarding the sanitizing of dropwizard metric names can be found at the [Prometheus DropwizardExports Collector](https://github.com/prometheus/client_java#dropwizardexports-collector) section.

More information regarding the [Prometheus Data Model](https://prometheus.io/docs/concepts/data_model/) in general.

## Getting Started
Dropwizard makes it easy to extend its functionality with litle work for the user.

Your main configuration class needs to implement the _PrometheusMetricsServletBundle_ interface:
```java
public class MyApplicationConfiguration extends Configuration implements PrometheusMetricsServletBundle {
  @Valid
  @NotNull
  @JsonProperty
  private PrometheusMetricsServletConfiguration prometheusMetrics;
  
  @Override
  public PrometheusMetricsServletConfiguration getPrometheusMetricsServletConfiguration() {
    return prometheusMetrics;
  }
}
```

Add the _PrometheusMetricsServletBundle_ bundle to your application:
```java
public class MyApplication extends Application<MyApplicationConfiguration> {

    @Override
    public void initialize(Bootstrap<MyApplicationConfiguration> bootstrap) {
        // Add bundle to serv metrics in prometheus compatible format at /prometheusMetrics
        bootstrap.addBundle(new PrometheusMetricsServletBundle());
    }

    @Override
    public void run(MyApplicationConfiguration configuration, Environment environment) {
        ...
    }
}
```

Add a section to your applications configuration file:
```yml
prometheusMetrics:
  sampleBuilder:
    type: default
```
The key _prometheusMetrics_ depends on the _JsonProperty_ name in your _MyApplicationConfiguration_ and can be changed.

## Dynamic Labels
By default dynamic labels are being extracted from the dropwizard metric name. Labels can be encoded as _my.metric.something\[label1:value1,label2:value2]_ and are automatically decoded and added to the prometheus metric. The prometheus metric name becomes _my.metric.something_.

The extraction of dynamic labels can be disabled through configuration options.

## Configuration Options

#### path
The URI path used to serv the prometheus metrics, default is _/prometheusMetrics_

#### sampleBuilder.type
Currently there are three support _sampleBuilder.type_'s:
* default: no explicit mapping
* simple-mapping: replaces the name of the metric, if it starts with the given string
* custom-mapping: use * as a wildcard to match and keywords $0, $1, .. to reference the matched value. The refernces can be used within the metric name as well as to add dynamic label values.

#### sampleBuilder.extractDynamicLabels
Enable or disable the automatic decoding of dynamic labels from the dropwizard metric name.

#### sampleBuilder.mappings
Defines the explicit mapping for dropwizard metrics to prometheus metrics. The format depends on the _sampleBuilder.type_.

The type _simple-mapping_ uses a _key: value_ syntax to define the _match_ part and the name that should be used instead.
```yml
  type: simple-mapping
  mappings:  
    my.metric.something: new.name
    my.metric.else: new.name
```

The type _custom-mapping_ uses * as a wildcard in the dropwizard metric name and allows to reference its value through the keywords $0, $1, .. representing the matched group.

Mappings can be defined in two formats, a _key: value_ syntax to define the _match_ part and the name that should be used instead.
The other option is to provide an object with a _name_ and an optional list of labels, each label expressed as a single string using the format _label:value_.

```yml
  type: custom-mapping
  mappings:  
    my.metric.something.*: new.name.$0
    my.metric.else.*: 
      name: new.name.$0
      labels: 
       - my_label:some_value
    my.metric.*.*: 
      name: new.name.$1
      labels: 
       - my_label:$0
```

## Dropwizard Example Configuration
Dropwizard comes with several instrumented classes by default and those metrics can easily be mapped to a more user friendly format.

```yml
prometheusMetrics:
  sampleBuilder:
    type: custom-mapping
    mappings:
      # Dropwizard FW default instrumented classes
      io.dropwizard.jetty.MutableServletContextHandler.*-requests:
        name: servlet.requests
        labels:
          - method:$0
      io.dropwizard.jetty.MutableServletContextHandler.*-responses:
        name: servlet.responses
        labels:
          - code:$0
      io.dropwizard.jetty.MutableServletContextHandler.percent-*-*:
        name: servlet.responses.percent
        labels:
          - code:$0
          - timeframe:$1
      io.dropwizard.jetty.MutableServletContextHandler.*-*: servlet.$1.$0
      io.dropwizard.jetty.MutableServletContextHandler.*: servlet.$0
      ch.qos.logback.core.Appender.*:
        name: logger
        labels:
          - level:$0
      org.apache.http.conn.HttpClientConnectionManager.*.*-connections:
        name: client.http.connections.$1
        labels:
          - name:$0
      org.eclipse.jetty.server.HttpConnectionFactory.*.connections:
        name: server.connections
        labels:
          - port:$0
      org.eclipse.jetty.util.thread.QueuedThreadPool.*.*:
        name: server.threadpools.$1
        labels:
          - name:$0
```

## Extension Support
The supported functionality is not enough, you want a more customized mapping solution or a different configuration structure?

Do not hesitate to extend _PrometheusSampleBuilderFactory_ and provide your own custom implementation.