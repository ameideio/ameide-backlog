---
title: 667 – Transformation Primitive v4 (Deploy Plan)
status: draft
owners:
  - platform
  - transformation
created: 2026-01-13
---

# 667 – Transformation Primitive v4 (Deploy Plan)

This document defines how Transformation v4 is deployed as Camunda/Zeebe “process solution” assets.

## Deployable units

Minimum v4 deployables:

1) Transformation **domain primitive**
- the system of record; emits facts; consumes intents/commands

2) Transformation **process solution**
- 4 BPMN resources deployed to Zeebe
- one worker microservice implementing all job types
- completion ingress (publish resume messages)
  - (if using Coder) a poller component to observe Coder Tasks and publish resume messages

3) **EDA→Zeebe bridge**
- consumes CloudEvents from Kafka topics
- publishes Zeebe messages (TTL + messageId + correlationKey)

## Deployment mechanism

Primary posture:
- CI deploys BPMN resources to Zeebe via Orchestration Cluster REST API `POST /v2/deployments`.
- GitOps deploys the worker service + bridge as Kubernetes workloads.

## Credentials / identity

- CI deploy uses an OIDC confidential client (machine principal) with deploy permissions.
- Cluster has the required secrets available before deploy steps run.

## Definition of Done (deploy)

- All 4 BPMNs are deployable to Zeebe (dev).
- Worker service is deployed and can activate/complete all job types.
- EDA→Zeebe bridge is deployed and reliably starts/correlates processes from domain facts.
- `ameide test smoke` is green for v4 cluster scenarios.

## Vendor deployment consideration (request size)

If BPMN deployments include additional resources (e.g., forms) in the same multipart request, enforce an explicit maximum payload size and keep deployments small enough to avoid REST API request-size limits. Configure the orchestration cluster API gateway limits explicitly if needed.
