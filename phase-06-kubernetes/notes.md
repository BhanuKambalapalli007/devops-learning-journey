
# Kubernetes overview — Phase 06

## What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform.

It automates deployment, scaling, and management of containerised apps.

## Why DevOps engineers use it

- Run hundreds of containers across multiple servers

- Automatic failover — dead containers restart automatically

- Scale up or down based on traffic

- Rolling deployments — zero downtime updates

- Foundation of modern cloud-native infrastructure

## Core concepts

- Pod       — smallest deployable unit, wraps one or more containers

- Node      — a server (VM or physical) that runs pods

- Cluster   — group of nodes managed by Kubernetes

- Service   — stable network endpoint for a set of pods

- Deployment — defines desired state, manages pod replicas

- Namespace  — logical isolation within a cluster

## AWS managed service

Amazon EKS (Elastic Kubernetes Service) — managed K8s on AWS

No need to manage the control plane yourself

## Status

⏳ Starting after Phase 05 Terraform is complete

