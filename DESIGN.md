# Design

## Goals

* Remove need for additional storage when Kafka is in place.
* Provide a fast and reliable storage that enable extensability via Kafka
    Consumers.
* Provide dependency graph building on real-time via stream-processing.

### Zipkin Storage Component

A Zipkin Storage component has the following internal parts:

* `Builder`: which configures if
    - `strictTraceId(boolean strictTraceId)`
    - `searchEnabled(boolean searchEnabled)`
    - `autocompleteKeys(List<String> keys)`
    - `autocompleteTtl(int autocompleteTtl)`
    - `autocompleteCardinality(int autocompleteCardinality)`
* `SpanStore`: main component
    - `Call<List<List<Span>>> getTraces(QueryRequest request);`
    - `Call<List<Span>> getTrace(String traceId);`
    - `Call<List<String>> getServiceNames();`
    - `Call<List<String>> getSpanNames(String serviceName);`
    - `Call<List<DependencyLink>> getDependencies(long endTs, long lookback);`
* `SpanConsumer`: which ingest spans
    - `Call<Void> accept(List<Span> spans)`
* `QueryRequest`: which includes
    - `String serviceName, spanName;`
    - `Map<String, String> annotationQuery;`
    - `Long minDuration, maxDuration;`
    - `long endTs, lookback;`
    - `int limit;`

Then we have 2 main interfaces to implement: `SpanConsumer` and `SpanStore`.

### Kafka Zipkin Storage

#### `KafkaSpanConsumer`

Kafka Span Consumer will be supported by a Kafka Producer which will receive a list
of Spans and will store them on a Kafka Topic (e.g. `zipkin-spans_v2`).

Then this Kafka topic will be used as a source for stream processors in charge
of support queries, covered by Span Store.

#### `KafkaSpanStore`

Kafka Span Store will need to support different kind of queries:

##### Get Trace

A Stream processor will take `spans` as input, and aggragate them into a trace
with key/value format: `key=trace_id, value=trace_payload` that will be used as 
a source for this query. Target topic will be called: `zipkin-traces_v2`

##### Get Service Names

This query will be served by a Stream processor that will take `spans` as input
and turn it into a key/value representation with `key=service_name,
value=operation_name`. All keys will be used as a source.

##### Get Span Names

Based on the stream processor described above, and in the key (i.e., service
name) selected by the previous operation, this query will be serve by a look by
key on the same store.

##### Get Dependencies

This query will be supported by a topic sourced from a stream-processor that
will be based on a previous work done here: 
<https://github.com/sysco-middleware/zipkin-dependencies-streaming>

##### Get Traces

This query is the most complex one and will require additional capabilities
that are not supported by Kafka Streams yet: we need an index that support
`QueryRequest` properties. As an initial option, Lucene or 
[Luwak](https://github.com/flaxsearch/luwak) will be used to create an
in-memory index that can handle these queries.

## Implementation

This Zipkin storage implementation is based on Kafka Streams State Store.

### Kafka Span Consumer

`KafkaSpanConsumer` implementation is based on Kafka Producer API, that stores spans individually on
a Spans Kafka topic.

### Kafka Span Store

`KafkaSpanStore` is based on Kafka Streams and [Lucene](https://lucene.apache.org/).

To support `getTrace`, `getServiceNames` and `getSpanNames` a stream-processor is:

- Polling data from incoming spans created by `KafkaSpanConsumer`.
- Group spans on traces.
- Record service names and span names related.
- Creating service dependencies.

To support `getTraces` requires an index of spans tagged by service names, span names, id, 
timestamp, tags, etc.

A custom Kafka Stream store was implemented to collect traces and index them using Lucene.

**All Stores are using Global state, which consumes all partitions. This means that every instance
stores all data on a defined directory.** All these with the trade-off of removing the need for an
additional database.

### Stream processors

#### Aggregation Stream Processor

This is the main processors that take incoming spans and aggregate them into:

- Traces
- Service Names
- Dependencies

#### Store Stream Processor

Global tables for traces, service names and dependencies to be available on local state.

#### Index Stream Processor

Custom processor to full-text indexing of traces using Lucene as back-end.

#### Retention Stream Processor

This is the processor that keeps track of trace timestamps for cleanup.