# Generic Profiling for Ingress Handlers

This example demonstrates how to add automatic Prometheus metrics profiling to any Ingress handler without modifying the handler code itself.

## Overview

The new `IngressMetrics` system provides automatic profiling capabilities that can be applied to any Ingress handler. It automatically tracks:

- **Request Count**: Total number of requests processed
- **Request Duration**: Time spent processing each request
- **Request/Response Bytes**: Total bytes received and sent
- **Error Count**: Total number of errors encountered

Additionally, the example demonstrates custom metrics with data bytes tracking in `MySystemStatsMetrics`.

## How It Works

Simply pass the endpoint directly to the `_with_metrics` methods:

```rust
use dynamo_runtime::pipeline::network::Ingress;

// Automatic profiling - endpoint is passed directly
let ingress = Ingress::for_engine_with_metrics(my_handler, &endpoint)?;
```

The endpoint automatically provides proper labeling (namespace, component, endpoint) for all metrics.

## Available Methods

The `Ingress` struct now provides a method for creating profiled instances:

- `Ingress::for_engine_with_metrics(engine, &endpoint)` - For engines with automatic profiling

## Metrics Generated

When you use the `_with_metrics` methods, the following Prometheus metrics are automatically created:

### Counters
- `ingress_requests_total` - Total requests processed
- `ingress_request_bytes_total` - Total bytes received in requests
- `ingress_response_bytes_total` - Total bytes sent in responses
- `ingress_errors_total` - Total errors encountered (with error_type labels)

### Custom System Metrics (MySystemStatsMetrics)
- `system_data_bytes_processed_total` - Total data bytes processed by system handler

### Error Types
The `ingress_errors_total` metric includes the following error types:
- `deserialization` - Errors parsing request messages
- `invalid_message` - Unexpected message format
- `response_stream` - Errors creating response streams
- `generate` - Errors in request processing
- `publish_response` - Errors publishing response data
- `publish_final` - Errors publishing final response

### Histograms
- `ingress_request_duration_seconds` - Request processing time

### Gauges
- `ingress_concurrent_requests` - Number of requests currently being processed

### Labels
All metrics automatically include these labels from the endpoint:
- `namespace` - The namespace name
- `component` - The component name
- `endpoint` - The endpoint name

## Examples

### Example: Simple Handler with Automatic Profiling

```rust
struct SimpleHandler;

#[async_trait]
impl AsyncEngine<SingleIn<String>, ManyOut<Annotated<String>>, Error> for SimpleHandler {
    async fn generate(&self, input: SingleIn<String>) -> Result<ManyOut<Annotated<String>>> {
        // Your business logic here
        // No need to add any metrics code!
        Ok(ResponseStream::new(Box::pin(stream), ctx.context()))
    }
}

// Automatic profiling - just pass the endpoint
let ingress = Ingress::for_engine_with_metrics(SimpleHandler::new(), &endpoint)?;
```

### Example: Custom Handler with Data Bytes Tracking

```rust
struct RequestHandler {
    metrics: Option<Arc<MySystemStatsMetrics>>,
}

#[async_trait]
impl AsyncEngine<SingleIn<String>, ManyOut<Annotated<String>>, Error> for RequestHandler {
    async fn generate(&self, input: SingleIn<String>) -> Result<ManyOut<Annotated<String>>> {
        let (data, ctx) = input.into_parts();

        // Track data bytes processed
        if let Some(metrics) = &self.metrics {
            metrics.data_bytes_processed.inc_by(data.len() as u64);
        }

        // Your business logic here...

        Ok(ResponseStream::new(Box::pin(stream), ctx.context()))
    }
}

// Create custom metrics and handler
let system_metrics = MySystemStatsMetrics::from_endpoint(&endpoint)?;
let handler = RequestHandler::with_metrics(system_metrics);
let ingress = Ingress::for_engine_with_metrics(handler, &endpoint)?;
```

## Benefits

1. **Ultra-Simple API**: Just pass the endpoint to `_with_metrics` methods
2. **Zero Code Changes**: Existing handlers work without modification
3. **Automatic Profiling**: Request count, duration, and error tracking out of the box
4. **Automatic Labeling**: Endpoint provides proper namespace/component/endpoint labels
5. **Performance**: Minimal overhead, metrics are only recorded when provided
6. **Backward Compatible**: Old code continues to work unchanged

## Running the Example

```bash
# Run the system metrics example
cargo run --bin system_server
```

## Querying Metrics

Once running, you can query the metrics:

```bash
# Get all ingress metrics
curl http://localhost:9091/metrics | grep ingress_

# Get request count for specific endpoint
curl http://localhost:9091/metrics | grep 'ingress_requests_total{endpoint="myendpoint"}'

# Get request duration histogram
curl http://localhost:9091/metrics | grep 'ingress_request_duration_seconds'
```