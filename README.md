# OpenShift Routes for AAP Controller Service

This repository contains the OpenShift Route definitions for the `controller-service` in the `aap` namespace. These OpenShift Route configurations provide external access to the AAP Controller Service, with options for both restricted private access and rate-limited public access. 

## Pre-reqs

- OpenShift CLI (oc) installed and configured.
- Ensure you are logged into your OpenShift cluster and have selected the correct project/namespace for (aap)" 
- Access to the aap namespace (or whatever namespace will be used).
- The AAP 2.4 controller-service already deployed and running in the desired namespace.

## Routes

1.  **`controller-private-route.yaml`**:
    * Hostname: `private-controller.apps.openshift.example.com`
    * Purpose: Provides access to the service for whitelisted IPs only (`172.0.0.0/24`).

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

## Testing the Routes

Run the curl command below from the whitelisted IP or CIDR that's specified in **`controller-private-route.yaml`**.
```
# You'll see 200 returned for successful attempts
$ for i in $(seq 1 10); do  curl -s -o /dev/null -w "%{http_code}\n" https://private-controller.apps.openshift.example.com/api/v2/ping/; done
200
200
200
```

Run the curl command below to test the public route and test rate limiting behavior from the public route created by **`controller-public-route.yaml`**.
```
$ for i in $(seq 1 15); do  curl -s -o /dev/null -w "%{http_code}\n" https://public-controller.apps.openshift.example.com/api/v2/ping/; done
200
200
200
200
200
200
200
200
200
200
000  # Rate limit kicks in
000
000
000
```

## Final Notes

1. `haproxy.router.openshift.io/rate-limit-connections: "true"`

   - Purpose: This is the master switch for enabling basic rate-limiting functionality for the route it's applied to.
   - Mechanism:
      - When set to "true", it instructs the OpenShift router (HAProxy) to prepare to track client connections and/or request rates for this specific route.
      - Behind the scenes, HAProxy uses a feature called "stick-tables." A stick-table is an in-memory storage area within HAProxy that can store data (counters, rates, flags) associated with entries, typically keyed by client source IP addresses.
      - This annotation essentially enables the creation or utilization of a stick-table associated with this route's backend services. The subsequent, more specific rate-limiting annotations will then define how this stick-table is used and what actions are taken.
   - Scope: It applies only to the route where this annotation is defined.
   - Behavior without it: If this annotation is absent or set to "false" (or any other value not interpreted as true), the other haproxy.router.openshift.io/rate-limit-connections.* annotations will generally have no effect, as the fundamental rate-limiting mechanism isn't activated for the route.
   - Impact: It provides a foundational layer of protection against denial-of-service (DoS) or abuse by overly aggressive clients by allowing you to define limits on how frequently or concurrently they can connect.


2. `haproxy.router.openshift.io/rate-limit-connections.rate-http: "10"`
   - Purpose: This annotation sets a specific limit on the rate of incoming HTTP/HTTPS requests from a single client IP address to this route.
   - Prerequisite: `haproxy.router.openshift.io/rate-limit-connections: "true"` must also be set on the same route.
   - Mechanism:
      - HAProxy tracks the number of HTTP(S) requests made by each source IP address over a defined time period (this period is often a default like 1 second or 10 seconds, or can sometimes be influenced by other less common annotations or global router configurations, though it's typically a short interval for HTTP rate limiting).
      - The value "10" means that if a client IP sends more than 10 HTTP(S) requests within that tracking period, subsequent requests from that same IP will be denied by HAProxy until their request rate drops below the configured threshold.
      - The denial usually results in an HTTP error response. The specific error code can sometimes be configured (e.g., 429 Too Many Requests or 503 Service Unavailable), but often defaults to what the HAProxy template is set to provide.
   - Tracking Key: The client is identified by its source IP address.
   - Stick-Table Usage: It utilizes a counter in the stick-table associated with the client's IP, specifically tracking http_req_rate() or a similar HAProxy counter.
   - Use Case: This is very effective for protecting web applications and APIs from rapid-fire requests, brute-force login attempts on HTTP forms, content scraping, or general abuse that involves a high volume of HTTP requests in a short time.
   - Considerations:
      - Setting this value too low might inadvertently block legitimate clients, especially if your application involves many quick, small AJAX requests or if users are behind a NAT gateway (where many users share a single public IP).
      - Setting it too high might not provide adequate protection. The "right" value depends heavily on your application's expected traffic patterns.


3. `haproxy.router.openshift.io/rate-limit-connections.concurrent-tcp: "5"`  
   - Purpose: This annotation limits the number of simultaneous (concurrent) active TCP connections that a single client IP address can have open to the backend service(s) associated with this route.
   - Prerequisite: `haproxy.router.openshift.io/rate-limit-connections: "true"` must also be set.
   - Mechanism:
      - HAProxy maintains a count of active TCP connections originating from each source IP address to the services backing this route.
      - The value "5" means that if a client IP attempts to establish a 6th TCP connection while it already has 5 active connections to this route, the new connection attempt will be rejected by HAProxy.
      - This is different from HTTP request rate. A single TCP connection can carry multiple HTTP requests (if using HTTP keep-alive). This annotation focuses on the raw number of open TCP sockets.
   - Tracking Key: The client is identified by its source IP address.
   - Stick-Table Usage: It utilizes a counter in the stick-table associated with the client's IP, specifically tracking the number of active connections (e.g., conn_cur in HAProxy terms for a given source).
