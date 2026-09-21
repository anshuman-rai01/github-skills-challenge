# AIOps Payment Service Assessment

This repository contains a small AIOps pipeline for monitoring a `payment-service`.
The sample operational data includes request timestamps, response times, CPU and
memory utilization, log levels, and log messages. Most records represent healthy
requests; a short incident includes slow responses, high resource utilization,
and timeout errors.

## Operational Context

The operational problem is detecting service degradation early from a mixture of
telemetry and logs. High latency, resource saturation, and warning/error-level
logs can indicate that payment requests or their database dependency are failing.
Without automated detection, an operator would need to inspect these signals
manually and could miss a short-lived incident.

AIOps in this assessment means applying simple automated detection and event
processing to operational data. The pipeline identifies records that cross
configured thresholds, explains the anomaly with one or more reasons, and
publishes the resulting event for downstream consumption. It is a deliberately
small in-memory simulation rather than a production monitoring platform.

## Repository Components

| Concern | File or component | Purpose |
| --- | --- | --- |
| Operational data | [`data/service_data.json`](data/service_data.json) | Sample `payment-service` telemetry and log records. |
| Metrics and logs | [`data/service_data.json`](data/service_data.json) | Provides response time, CPU, memory, log level, message, service, and timestamp fields. |
| Anomaly detection | [`src/anomaly_detector.py`](src/anomaly_detector.py) | Applies response-time, CPU, and memory thresholds and flags warning-level logs. It returns an `ANOMALY` event with reasons and the source record. |
| Event production | [`src/event_producer.py`](src/event_producer.py) | Publishes non-empty anomaly events to an `EventTopic`. |
| Event topics | [`src/event_topic.py`](src/event_topic.py) | Implements a named, in-memory message list with publish, read, and clear operations. |
| Event consumption | [`src/event_consumer.py`](src/event_consumer.py) | Reads messages from an `EventTopic` for downstream processing. |
| Final AIOps processing | [`src/aiops_pipeline.py`](src/aiops_pipeline.py) | Loads the data, runs detection, publishes detected events, consumes events, and returns processing results. |
| Supporting calculations | [`src/calculations.py`](src/calculations.py) | Contains standalone circle-area and Fibonacci examples covered by the general test suite; it is not part of the AIOps flow. |
| Tests | [`tests/test_aiops_pipeline.py`](tests/test_aiops_pipeline.py) and [`tests/calculations_test.py`](tests/calculations_test.py) | Verify detector, producer, consumer, topic, and calculation behavior. |

## Running the Pipeline

```bash
python3 src/aiops_pipeline.py
```

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

