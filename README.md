# OpenShift Routes for Controller EDB Service

This repository contains the OpenShift Route definitions for the `controller-edb-01-service` in the `aap-24` namespace.

## Routes

1.  **`controller-edb-01-private-route.yaml`**:
    * Hostname: `private-controller-edb-01-aap-24.apps.hammer-sno.arsalan.io`
    * Purpose: Provides access to the service for whitelisted IPs only (`172.16.0.0/24`).
    * Rate Limiting: Configured with minimal or very generous rate limits (adjust as needed).

2.  **`controller-edb-01-public-route.yaml`**:
    * Hostname: `public-controller-edb-01-aap-24.apps.hammer-sno.arsalan.io`
    * Purpose: Provides general access to the service.
    * Rate Limiting: Configured with stricter rate limits for public traffic.

## Applying the Routes

Ensure you are logged into your OpenShift cluster and have selected the correct project/namespace (`aap-24`).

To apply a route:
```bash
oc apply -f <route-filename.yaml> -n aap-24
