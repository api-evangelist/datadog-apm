---
title: "When failover isn't safe: Building high-availability PostgreSQL on Kubernetes"
url: "https://www.datadoghq.com/blog/engineering/postgresql-ha-kubernetes/"
date: "2026-06-04"
author: ""
feed_url: "https://www.datadoghq.com/blog/index.xml"
---
Datadog engineers redesigned their PostgreSQL clusters for high availability on Kubernetes using Patroni and synchronous replication, addressing edge cases where naive failover can cause data loss or split-brain scenarios. The post details the architectural decisions, trade-offs, and operational lessons learned from running production HA PostgreSQL at scale.
