---
title: "Install and Configure OpenSearch Suite on Linux"
date: 2025-11-08 08:35:00 -0500
categories: [Linux, Install Guide]
tags: [opensearch]
author: Th3_4ngl3r
toc: true
comments: true
---

This guide walks through a Bash script that installs and configures the OpenSearch suite on a Linux system using RPM packages. It includes user setup, plugin activation, and service management.

## Table of Contents

- [Overview](#overview)
- [Installation Script](#installation-script)
- [Plugin Descriptions](#plugin-descriptions)
- [Final Notes](#final-notes)

---

## Overview

OpenSearch is an open-source search and analytics suite derived from Elasticsearch. This script installs OpenSearch and OpenSearch Dashboards, configures security, and enables key plugins like Observability, Alerting, ML Commons, and k-NN.

---

## Installation Script

```bash
#!/bin/bash
set -e  # Exit immediately if any command fails

# Define version and user/group for OpenSearch
OPENSEARCH_VERSION="3.2.0"
OPENSEARCH_USER="opensearch"
OPENSEARCH_GROUP="opensearch"

# Update system and install dependencies
echo "Updating system and installing dependencies..."
sudo dnf update -y
sudo dnf install -y wget curl unzip java-17-openjdk java-17-openjdk-devel

# Create system user and group for OpenSearch
echo "Creating OpenSearch user and group..."
sudo groupadd --system $OPENSEARCH_GROUP || true
sudo useradd --system --gid $OPENSEARCH_GROUP --shell /sbin/nologin --home-dir /usr/share/opensearch $OPENSEARCH_USER || true

# Download RPM packages for OpenSearch and Dashboards
echo "Downloading OpenSearch RPM packages..."
wget https://artifacts.opensearch.org/releases/bundle/opensearch/${OPENSEARCH_VERSION}/opensearch-${OPENSEARCH_VERSION}-linux-x64.rpm
wget https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/${OPENSEARCH_VERSION}/opensearch-dashboards-${OPENSEARCH_VERSION}-linux-x64.rpm

# Install the downloaded packages
echo "Installing OpenSearch and Dashboards..."
sudo rpm -ivh opensearch-${OPENSEARCH_VERSION}-linux-x64.rpm
sudo rpm -ivh opensearch-dashboards-${OPENSEARCH_VERSION}-linux-x64.rpm

# Set correct ownership for OpenSearch directories
echo "Setting permissions..."
sudo chown -R $OPENSEARCH_USER:$OPENSEARCH_GROUP /usr/share/opensearch
sudo chown -R $OPENSEARCH_USER:$OPENSEARCH_GROUP /etc/opensearch
sudo chown -R $OPENSEARCH_USER:$OPENSEARCH_GROUP /var/lib/opensearch
sudo chown -R $OPENSEARCH_USER:$OPENSEARCH_GROUP /var/log/opensearch

# Enable and start OpenSearch services
echo "Enabling and starting services..."
sudo systemctl enable opensearch.service
sudo systemctl start opensearch.service
sudo systemctl enable opensearch-dashboards.service
sudo systemctl start opensearch-dashboards.service

# Configure security plugin with SSL
echo "Configuring Security plugin..."
sudo cp /etc/opensearch/opensearch.yml /etc/opensearch/opensearch.yml.bak
sudo tee -a /etc/opensearch/opensearch.yml > /dev/null <<EOF

# Enable Security plugin
plugins.security.disabled: false
plugins.security.ssl.transport.enabled: true
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: certs/opensearch.pem
plugins.security.ssl.http.pemkey_filepath: certs/opensearch-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: certs/root-ca.pem
EOF

# 🔌 Enable core plugins
echo "Enabling Observability, Alerting, ML, and k-NN plugins..."
sudo tee -a /etc/opensearch/opensearch.yml > /dev/null <<EOF

# Observability
plugins.observability.enabled: true

# Alerting
plugins.alerting.enabled: true

# ML Commons
plugins.ml_commons.enabled: true
plugins.ml_commons.allow_registering_model: true

# k-NN
plugins.knn.enabled: true

# Neural Search (optional)
plugins.ml_commons.neural_search.enabled: true
EOF

# Configure Dashboards for plugin support
echo "Configuring OpenSearch Dashboards for plugins..."
sudo cp /etc/opensearch-dashboards/opensearch_dashboards.yml /etc/opensearch-dashboards/opensearch_dashboards.yml.bak
sudo tee -a /etc/opensearch-dashboards/opensearch_dashboards.yml > /dev/null <<EOF

# Enable security plugin
opensearch_security.auth.type: "basicauth"

# Enable observability and ML dashboards
opensearchDashboardsObservability.enabled: true
opensearchDashboardsMlCommons.enabled: true
EOF

# Restart services to apply changes
echo "Restarting services to apply changes..."
sudo systemctl restart opensearch.service
sudo systemctl restart opensearch-dashboards.service

# Final output
echo "OpenSearch suite installed and configured!"
echo "Access OpenSearch Dashboards at http://localhost:5601"
echo "Default credentials: admin / admin (change ASAP)"
```

## Plugin Descriptions

- **Security Plugin**  
  Adds authentication, role-based access control, and SSL encryption to protect your cluster.

- **Observability**  
  Enables metrics, logs, and traces to monitor system performance and troubleshoot issues.

- **Alerting**  
  Allows you to define monitors and triggers to notify you of anomalies or thresholds.

- **ML Commons**  
  Provides machine learning capabilities like training models and running inference tasks.

- **k-NN**  
  Enables vector-based search using approximate nearest neighbor algorithms for similarity queries.

- **Neural Search**  
  Enhances ML Commons with semantic search powered by neural networks.

## Final Notes

This script sets up a secure and feature-rich OpenSearch environment on a Linux system. Be sure to:

- **Replace certificate paths** with your actual SSL files.
- **Change the default credentials** immediately to secure your cluster.
- **Open port 5601** in your firewall to access OpenSearch Dashboards remotely.
- **Consider automating** this setup with tools like Ansible or scheduling backups using cron.
