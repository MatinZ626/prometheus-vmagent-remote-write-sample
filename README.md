# Prometheus vmagent Remote Write Example

A practical example of using VictoriaMetrics vmagent to improve Prometheus metric delivery reliability during intermittent network outages.

## Overview

In distributed monitoring environments, Prometheus may experience intermittent connectivity issues when scraping Node Exporter instances running on remote servers, especially across unstable international network connections.

These interruptions can result in missing metrics and gaps in time-series data, making it difficult to maintain reliable monitoring and troubleshoot historical incidents.

This repository demonstrates an alternative architecture using `vmagent` and its Remote Write functionality to reduce metric loss during temporary network interruptions.

## The Problem

In a traditional pull-based monitoring setup, Prometheus scrapes metrics directly from remote Node Exporter instances.

When network connectivity between Prometheus and a remote exporter is interrupted, scraping fails. Metrics that are not successfully collected during the outage cannot be recovered automatically by Prometheus.

This can lead to gaps in monitoring data and incomplete historical metrics.

## The Solution

`vmagent` can collect metrics from Node Exporter instances and forward them to a compatible remote storage endpoint using Remote Write.

Its disk-backed buffering mechanism allows pending data to be retained temporarily on disk when the remote write destination is unavailable. Once connectivity is restored, vmagent can resume sending the buffered data.

This approach improves resilience against temporary connectivity issues and helps reduce metric loss.

## Architecture

```text
Remote Server
┌─────────────────────────┐
│      Node Exporter      │
│            │            │
│            ▼            │
│         vmagent         │
│            │            │
│            ▼            │
│    Disk-backed Buffer   │
└────────────┬────────────┘
             │
             │ Remote Write
             ▼
       Unstable Network
             │
             ▼
┌─────────────────────────┐
│          Nginx          │
│       Reverse Proxy     │
└────────────┬────────────┘
             │
             │ HTTP Proxy
             ▼
┌─────────────────────────┐
│       Prometheus        │
│  Remote Write Receiver  │
└─────────────────────────┘
```

> The diagram illustrates the intended data flow. The exact configuration depends on the deployment environment and the selected remote write destination.

## What This Repository Covers

- Configuring vmagent to scrape remote Node Exporter targets.
- Forwarding collected metrics using Remote Write.
- Configuring disk-backed buffering for temporary destination outages.
- Identifying configuration and operational issues that may lead to metric loss.

## Configuration

The main vmagent settings typically include:

- `-promscrape.config`: Path to the scrape configuration.
- `-remoteWrite.url`: Remote Write destination URL.
- `-remoteWrite.tmpDataPath`: Directory used for temporary persistent Remote Write data.
- `-remoteWrite.maxDiskUsagePerURL`: Maximum disk space available for pending data for a Remote Write destination.
- `-remoteWrite.forcePromProto:` Forces vmagent to use the Prometheus Remote Write protocol for the corresponding remote write destination. This is useful when the destination expects the Prometheus Remote Write protocol.
- `-remoteWrite.queues=1:` Configures a single concurrent Remote Write queue per destination. This can help avoid out-of-order sample ingestion when the remote storage does not support backfilling, as may happen when buffered samples are sent after a network outage. A single queue may reduce write throughput, so monitor the pending data backlog and adjust the setting according to your workload.

## Enable Prometheus Remote Write Receiver

To allow Prometheus to receive metrics from vmagent, enable the Remote Write Receiver.

If Prometheus is installed as a systemd service:

Add the following flag to the ExecStart command in the Prometheus service file:
```bash
--web.enable-remote-write-receiver
```
Then restart Prometheus:

```bash
sudo systemctl daemon-reload
sudo systemctl restart prometheus
```
If Prometheus is running in Docker:

Add the following flag to the Prometheus service's command section in your docker-compose.yml file:
```bash
- "--web.enable-remote-write-receiver"
```
Then recreate the container:
```bash
docker compose up -d
```

## Limitations and Considerations

Disk-backed buffering improves resilience, but it does not guarantee zero data loss.

Potential causes of data loss include:

- Buffer disk exhaustion or configured disk usage limits.
- Network outages lasting longer than the available buffering capacity can accommodate.
- Failure to collect metrics at the source while the exporter is unreachable.
- Incorrect scrape or Remote Write configuration.
- Disk or host failures and non-persistent container storage.
- Remote Write queue behavior and retry limitations.

The buffer protects pending data that vmagent has already collected; it cannot recreate metrics that were never scraped.


### NOTE: Metric Gaps After Network Recovery

Metrics queried using rate() may still show gaps after network connectivity is restored. The rate() function requires sufficient samples within the selected time window to calculate the per-second average rate of increase. Missing samples or samples rejected by the destination due to out-of-order ingestion can affect these calculations. For example, CPU usage graphs based on rate() may still show gaps after buffered metrics have been delivered.

To reduce the risk of out-of-order ingestion, based on our metric ingestion volume and the destination's behavior, we configured:

- "-remoteWrite.queues=1"

## Before You Go 

Please remember to adapt the configuration to your own needs and infrastructure. I hope this example helps you!

Contributions, feedback, and suggestions are welcome.


