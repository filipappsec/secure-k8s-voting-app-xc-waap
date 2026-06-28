# Secure Kubernetes Voting App with F5 XC WAAP

## Overview

This project is a production-style Kubernetes security lab focused on deploying, hardening and protecting a multi-tier application exposed through F5 XC WAAP.

The application is based on the Example Voting App architecture and contains:

* Python vote frontend
* Redis queue
* .NET worker
* PostgreSQL database
* Node.js result frontend

The goal is not only to deploy an application to Kubernetes, but to understand and document how traffic flows through the system, how the internal services communicate, where security controls should be placed, and how F5 XC WAAP can protect public north-south traffic.

## Project Goals

* Deploy a distributed application on Kubernetes
* Understand Kubernetes Services, Ingress, and internal DNS
* Separate public and private application components
* Harden Kubernetes workloads using securityContext, probes, and resource limits
* Restrict east-west traffic using NetworkPolicies
* Reduce Kubernetes API exposure using ServiceAccount hardening
* Configure public DNS records for the application
* Configure TLS termination on F5 XC HTTP Load Balancer
* Expose the application through F5 XC HTTP Load Balancer
* Attach WAAP, rate limiting, and service policies using Terraform
* Test common web attack payloads and observe WAAP behavior
* Document traffic flow, threat model, security decisions, TLS design, and troubleshooting scenarios

## Target Architecture

```text
Internet
  |
  v
Public DNS Record
  |
  v
F5 Distributed Cloud HTTP Load Balancer
  |
  |-- TLS Termination
  |-- WAAP Policy
  |-- Rate Limiting
  |-- Service Policy
  |
  v
Kubernetes Ingress
  |
  v
Kubernetes Services
  |
  |-- vote-service -> vote pods
  |-- result-service -> result pods
  |-- redis-service -> Redis
  |-- postgres-service -> PostgreSQL
  |-- worker -> background processor
```

## Domain and TLS

The application will be exposed through dedicated public hostnames.

Planned public endpoints:

```text
https://vote.example.com
https://result.example.com
```

TLS will be configured on the F5 Distributed Cloud HTTP Load Balancer. Public client traffic will terminate at the XC edge, where WAAP, rate limiting, and service policies will be applied before forwarding traffic to the Kubernetes origin.

Expected request flow:

```text
User Browser
  |
  v
Public DNS Record
  |
  v
F5 Distributed Cloud HTTP Load Balancer
  |
  |-- TLS Termination
  |-- WAAP Inspection
  |-- Rate Limiting
  |-- Service Policy Enforcement
  |
  v
Kubernetes Origin
  |
  v
Ingress Controller
  |
  v
Kubernetes Services
  |
  v
Application Pods
```

## TLS Design

Planned TLS model:

```text
Client -> F5 XC: HTTPS
F5 XC -> Origin: HTTP or HTTPS depending on lab phase
```

Initial implementation may use HTTP between F5 XC and the Kubernetes origin for simplicity.

A later phase may add end-to-end TLS:

```text
Client -> F5 XC: HTTPS
F5 XC -> Kubernetes Ingress: HTTPS
```

The final documentation will clearly describe:

* Where TLS is terminated
* Whether origin traffic is encrypted
* How certificates are managed
* How direct origin access is restricted
* What risks exist if the origin is reachable outside F5 XC

## Security Model

Only the vote and result applications should be reachable from outside the cluster.

Redis, PostgreSQL, and the worker should remain internal-only components.

Expected communication paths:

```text
Ingress -> vote-service
Ingress -> result-service
vote -> redis
worker -> redis
worker -> postgres
result -> postgres
```

Everything else should be denied by default using Kubernetes NetworkPolicies.

## Origin Bypass Risk

WAAP only protects traffic that passes through F5 XC.

If the Kubernetes origin is directly reachable from the internet, an attacker may bypass the WAAP layer and attack the application directly.

Bad flow:

```text
Attacker -> Kubernetes Origin directly
```

Expected protected flow:

```text
User -> Public DNS -> F5 XC WAAP -> Kubernetes Origin
```

The project will document origin protection options such as:

* Restricting direct origin access
* Allowing only trusted edge sources
* Using private connectivity where possible
* Avoiding public exposure of internal Kubernetes services
* Ensuring Redis, PostgreSQL, and worker components are never public

## Repository Status

This project is currently in the design and planning phase.

Implementation will be developed in stages:

1. Local Docker Compose validation
2. Kubernetes base deployment
3. Ingress exposure
4. ConfigMaps, Secrets, and persistent storage
5. Workload hardening
6. NetworkPolicy enforcement
7. RBAC and ServiceAccount hardening
8. F5 XC integration using Terraform
9. Domain and DNS configuration
10. TLS termination on F5 XC HTTP Load Balancer
11. WAAP and rate limiting tests
12. Origin bypass validation
13. Final documentation and walkthrough

## Learning Focus

This project is designed to build practical knowledge in:

* Kubernetes networking
* Kubernetes workload security
* DevSecOps and infrastructure as code
* F5 WAAP architecture
* DNS and TLS design for public applications
* Edge-to-origin traffic flow
* Origin protection and origin bypass risks
* Web application attack testing
* Production-style troubleshooting

## Planned Security Controls

The project will include the following security controls:

| Layer      | Control                                     |
| ---------- | ------------------------------------------- |
| Edge       | F5 XC HTTP Load Balancer                    |
| Edge       | TLS termination                             |
| Edge       | WAAP policy                                 |
| Edge       | Rate limiting                               |
| Edge       | Service policy                              |
| Kubernetes | Ingress Controller                          |
| Kubernetes | NetworkPolicies                             |
| Kubernetes | securityContext                             |
| Kubernetes | resource requests and limits                |
| Kubernetes | readiness and liveness probes               |
| Kubernetes | ServiceAccount hardening                    |
| Kubernetes | Internal-only Redis and PostgreSQL services |

## Final Goal

The final goal is to build a realistic Kubernetes security lab that demonstrates how a multi-tier application can be deployed, hardened, exposed through a public domain, protected with TLS and WAAP, and documented from both platform engineering and security architecture perspectives.
