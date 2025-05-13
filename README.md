# OpenShift Routes for AAP Controller Service

This repository contains the OpenShift Route definitions for the `controller-service` in the `aap` namespace.

## Routes

1.  **`controller-private-route.yaml`**:
    * Hostname: `private-controller.apps.openshift.example.com`
    * Purpose: Provides access to the service for whitelisted IPs only (`172.0.0.0/24`).
    * Rate Limiting: Configured with minimal or very generous rate limits (adjust as needed).

2.  **`controller-public-route.yaml`**:
    * Hostname: `public-controller.apps.openshift.example.com`
    * Purpose: Provides general access to the service.
    * Rate Limiting: Configured with stricter rate limits for public traffic.

## Applying the Routes

Ensure you are logged into your OpenShift cluster and have selected the correct project/namespace for (`aap`).

To apply the routes:
```
oc apply -f controller-private-route.yml controller-public-route.yml -n <namespace_for_aap>
```
