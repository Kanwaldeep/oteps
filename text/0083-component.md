# InstrumentationLibrary

Introducing the notion of `InstrumentationLibrary` as a first class concept in
OpenTelemetry which describes the named `Tracer|Meter` in the data model.

## Motivation

The main motivation for this is to better model telemetry coming from different
instrumentation libraries inside an Application (`Resource`). This change
affects only the OpenTelemetry protocol and exporters, and it does not require
any API changes.

The proposal is to make `InstrumentationLibrary` a first class concept in
OpenTelemetry, with a scope defined by the `named Tracer|Meter`. It describes
all telemetry generated by one `named Tracer|Meter`.

## Explanation

```
                                Application
+--------------------------------------------------------------------------+
|                         TracerProvider(Resource)                         |
|                         MeterProvider(Resource)                          |
|                                                                          |
|      Instrumentation Library 1           Instrumentation Library 2       |
|  +--------------------------------+  +--------------------------------+  |
|  | Tracer(InstrumentationLibrary) |  | Tracer(InstrumentationLibrary) |  |
|  | Meter(InstrumentationLibrary)  |  | Meter(InstrumentationLibrary)  |  |
|  +--------------------------------+  +--------------------------------+  |
|                                                                          |
|      Instrumentation Library 3           Instrumentation Library 4       |
|  +--------------------------------+  +--------------------------------+  |
|  | Tracer(InstrumentationLibrary) |  | Tracer(InstrumentationLibrary) |  |
|  | Meter(InstrumentationLibrary)  |  | Meter(InstrumentationLibrary)  |  |
|  +--------------------------------+  +--------------------------------+  |
|                                                                          |
+--------------------------------------------------------------------------+
```

Every application has one `TracerProvider` and one `MeterProvider` which have a
`Resource` associated with them that describes the Application.

Every instrumentation library has one `Tracer` and one `Meter` which have a
`InstrumentationLibrary` associated with them that describes the instrumentation
library.

In case of multi-application deployments like Spring boot (or old Java
Application servers) every Application will have it's own `TracerProvider` and
`MeterProvider` instances.

## Internal details

This proposal affects only the OpenTelemtry protocol, and proposes a way to
represent the telemetry data in a structured way.
For example, here is the protobuf definition for metrics:
metrics:

```proto
// Resource information.
message Resource {
  // Set of labels that describe the resource.
  repeated opentelemetry.proto.common.v1.AttributeKeyValue attributes = 1;
}

// InstrumentationLibrary is a message representing the instrumentation library
// informations such as the fully qualified name and version.
message InstrumentationLibrary {
  string name = 1;
  string version = 2;
  // Can be extended with attributes.
}

// A collection of InstrumentationLibraryMetrics from a Resource.
message ResourceMetrics {
  // The Resource for the metrics in this message.
  // If this field is not set then no Resource is known.
  Resource resource = 1;

  // A list of metrics that originate from a Resource.
  repeated InstrumentationLibraryMetrics instrumentation_library_metrics = 2;
}

// A collection of Metrics from a InstrumentationLibrary.
message InstrumentationLibraryMetrics {
  // The Instrumentation library information for the metrics in this message.
  // If this field is not set then no InstrumentationLibrary is known.
  InstrumentationLibrary instrumentation_library = 1;

  // A list of metrics that originate from a Instrumentation library.
  repeated Metric metrics = 2;
}
```

## Trade-offs and mitigations

Adding a new concept into the OpenTelemetry protocol and the exporter framework
may be overkill, but this concept maps easily to an already existing concept
in the API, and it is easy to explain to users.

## Prior art and alternatives

An alternative approach was proposed in the proto [PR comment](
https://github.com/open-telemetry/opentelemetry-proto/pull/94#discussion_r369952371)
which suggested to enclose `Resource` at multiple levels including the
named `Tracer|Meter`.

## Open questions

1. Should we support more than name and version for the InstrumentationLibrary
now?

## Future possibilities

In the future the `InstrumentationLibrary` can be extended to support multiple
properties (attributes) that apply to the specific instance of the
instrumenting library.
Also, information about the instrumented library could be added, but that will require additional consideration about grouping, like grouping by the pair (instrumenting lib, instrumenting lib) instead of just by instrumenting lib.