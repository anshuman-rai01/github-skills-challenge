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

## Logs and Metrics Analysis

The observations below are based on the 10 records in
[`data/service_data.json`](data/service_data.json).

1. **Metric fields:** `response_time_ms` is the request-latency metric, while
	`cpu_percent` and `memory_percent` are resource-utilization metrics. Their
	numeric values can be compared over time and against operational thresholds.

2. **Log fields:** `log_level` and `message` represent application log
	information. The dataset uses `INFO` for successful requests and `ERROR` for
	timeout conditions. `service` identifies the emitting service, while
	`timestamp` provides context for both the metrics and the log entry.

3. **Timestamp usage:** Timestamps are ISO-style date-time strings, all on
	`2026-09-20`, recorded at one-minute intervals from `10:00:00` through
	`10:09:00`. They provide the chronological order needed to compare the
	service baseline, incident window, and recovery.

4. **Normal behaviour:** The records at `10:00`–`10:04` and `10:07`–`10:09`
	appear normal. They have response times from 120–150 ms, CPU utilization from
	42–50%, memory utilization from 51–57%, `INFO` log levels, and successful
	payment messages. This accounts for 8 of the 10 observations.

5. **Unusual behaviour:** The records at `10:05` and `10:06` form a short
	incident. Response time rises to 610 ms and 640 ms, and the log messages
	report a payment-service timeout and a database connection timeout. At
	`10:06`, CPU reaches 94% and memory reaches 91%; both exceed the detector's
	80% thresholds. The two records also exceed the detector's 500 ms response
	time threshold and have `ERROR` log levels. Metrics and logs return to the
	normal range at `10:07`, suggesting the issue is temporary in this sample.

## Anomaly Detection Analysis

The provided `AnomalyDetector` was used with its configured thresholds:
response time greater than 500 ms, CPU utilization greater than 80%, and memory
utilization greater than 80%. `WARNING` and `ERROR` log levels are also treated
as concerning events. The pipeline processed all 10 records and published the
detected events through the existing in-memory topic and consumer components.

### Detection Report

| Timestamp | Metric and log evidence | Detection reasons |
| --- | --- | --- |
| `2026-09-20T10:05:00` | 610 ms response time; `ERROR`; `Payment service timeout` | High response time; error log detected |
| `2026-09-20T10:06:00` | 640 ms response time, 94% CPU, 91% memory; `ERROR`; `Database connection timeout` | High response time; high CPU utilization; high memory utilization; error log detected |

The result distinguishes the 2 incident observations from the 8 normal
observations. No expected anomaly was missed in this dataset: both `ERROR`
timeout records were detected, including their relevant metric and log reasons.
No normal event was incorrectly flagged; all `INFO` records remain below the
configured metric thresholds and were not reported as anomalies.

One limitation is that the detector uses fixed, global thresholds. It does not
learn the service's normal baseline or correlate events across time, so a
gradual performance change that remains below the thresholds could be missed.

## AIOps Event Flow Verification

The event-processing workflow uses the existing components in this order:

1. **Event/message:** `AnomalyDetector.detect` creates an `ANOMALY` event when a
	record has an abnormal metric or concerning log level. The event contains the
	service, timestamp, type, detection reasons, and original source record.
2. **Producer:** `EventProducer.publish` accepts the event and forwards it to
	its configured topic, returning `True` for a published event.
3. **Topic:** `EventTopic` stores the event in its in-memory message list. The
	pipeline uses the same `service-events` topic for both publication and
	consumption.
4. **Consumer:** `EventConsumer.consume` reads the topic messages and returns
	the received events to the pipeline.
5. **Downstream AIOps component:** `run_pipeline` collects the consumed events
	in `events_consumed`, and the CLI prints their service, timestamp, type, and
	reasons as the final AIOps result.

### Execution Result

Running `python3 src/aiops_pipeline.py` processed 10 records and reported:

| Result | Value |
| --- | ---: |
| Anomalies detected | 2 |
| Events consumed | 2 |
| Event timestamps | `2026-09-20T10:05:00`, `2026-09-20T10:06:00` |

Both consumed events retained their anomaly type, payment-service identity,
source timestamp, metric/log reasons, and source record. This verifies that an
anomaly can travel through detection, production, topic publication,
consumption, and the downstream AIOps report.

## Repository Components

| Concern | File or component | Purpose |
| --- | --- | --- |
| Operational data | [`data/service_data.json`](data/service_data.json) | Sample `payment-service` telemetry and log records. |
| Metrics and logs | [`data/service_data.json`](data/service_data.json) | Provides response time, CPU, memory, log level, message, service, and timestamp fields. |
| Anomaly detection | [`src/anomaly_detector.py`](src/anomaly_detector.py) | Applies response-time, CPU, and memory thresholds and flags warning/error-level logs. It returns an `ANOMALY` event with reasons and the source record. |
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

