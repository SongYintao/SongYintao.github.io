---
title: Kubernetes Service Discovery Based DNS
subtitle: Kubernetes Service Discovery Based DNS
layout: post
tags: [k8s]
---

### Resource Records

Any DNS-based service discovery solution for Kubernetes must provide the **resource records** (**RR**s) described below to be considered compliant with this specification.



#### Definitions

In the RR descriptions below, values not in angle brackets, `< >`, are literals. The meaning of the values in angle brackets are defined below or in the description of the specific record.

- `<zone>` = configured **cluster domain**, e.g. cluster.local
- `<ns>` = a **Namespace**
- `<ttl>` = the standard DNS **time-to-live** value for the record

In the RR descriptions below, the following definitions should be used for words in *italics*.

*hostname*

- In order of precedence, the hostname  of an endpoint is:
  - The value of the endpoint's `hostname` field.
  - A unique, system-assigned identifier for the endpoint. The exact format and source of this identifier is not prescribed by this specification. However, it must be possible to use this to identify a specific endpoint in the context of a Service. This is used in the event no explicit endpoint hostname is defined.

*ready*

- An endpoint is considered *ready* if its address is in the `addresses` field of the EndpointSubset object, or the corresponding service has the `service.alpha.kubernetes.io/tolerate-unready-endpoints` annotation set to `true`.

All comparisons between query data and data in Kubernetes are **case-insensitive**.

 

### 2.2 - Record for Schema Version

There must be a `TXT` record named `dns-version.<zone>.` that contains the [semantic version](http://semver.org/) of the **DNS schema** in use in this cluster.

- Record Format:
  - `dns-version.<zone>. <ttl> IN TXT <schema-version>`
- Question Example:
  - `dns-version.cluster.local. IN TXT`
- Answer Example:
  - `dns-version.cluster.local. 28800 IN TXT "1.0.0"`

### 2.3 - Records for a Service with ClusterIP

Given a Service named `<service>` in Namespace `<ns>` with ClusterIP `<cluster-ip>`, the following records must exist.

#### 2.3.1 - `A` Record

- Record Format:
  - `<service>.<ns>.svc.<zone>. <ttl> IN A <cluster-ip>`
- Question Example:
  - `kubernetes.default.svc.cluster.local. IN A`
- Answer Example:
  - `kubernetes.default.svc.cluster.local. 4 IN A 10.3.0.1`

#### 2.3.2 - `SRV` Records

For each port in the Service with name `<port>` and number `<port-number>` using protocol `<proto>`, an `SRV` record of the following form must exist.

- Record Format:
  - `_<port>._<proto>.<service>.<ns>.svc.<zone>. <ttl> IN SRV <weight> <priority> <port-number> <service>.<ns>.svc.<zone>.`

The priority `<priority>` and weight `<weight>` are numbers as described in [RFC2782](https://tools.ietf.org/html/rfc2782) and whose values are not prescribed by this specification.

Unnamed ports do not have an `SRV` record.

- Question Example:
  - `_https._tcp.kubernetes.default.svc.cluster.local. IN SRV`
- Answer Example:
  - `_https._tcp.kubernetes.default.svc.cluster.local. 30 IN SRV 10 100 443 kubernetes.default.svc.cluster.local.`

**The Additional section of the response may include the Service `A` recor**d **referred to** in the `SRV` record.

#### 2.3.3 - `PTR` Record

Given Service ClusterIP `<a>.<b>.<c>.<d>`, a `PTR` record of the following form must exist.

- Record Format:
  - `<d>.<c>.<b>.<a>.in-addr.arpa. <ttl> IN PTR <service>.<ns>.svc.<zone>.`
- Question Example:
  - `1.0.3.10.in-addr.arpa. IN PTR`
- Answer Example:
  - `1.0.3.10.in-addr.arpa. 14 IN PTR kubernetes.default.svc.cluster.local.`

### 2.4 - Records for a Headless Service

Given a headless Service `<service>` in Namespace `<ns>` (i.e., a Service with no ClusterIP), the following records must exist.

#### 2.4.1 - `A` Records

There must be an `A` record for each *ready* endpoint of the headless Service with IP address `<endpoint-ip>` as shown below. If there are no *ready* endpoints for the headless Service, the answer should be `NXDOMAIN`.

- Record Format:
  - `<service>.<ns>.svc.<zone>. <ttl> IN A <endpoint-ip>`
- Question Example:
  - `headless.default.svc.cluster.local. IN A`
- Answer Example:

```
    headless.default.svc.cluster.local. 4 IN A 10.3.0.1
    headless.default.svc.cluster.local. 4 IN A 10.3.0.2
    headless.default.svc.cluster.local. 4 IN A 10.3.0.3
```

There must also be an `A` record of the following form for each *ready* endpoint with *hostname* of `<hostname>` and IP address `<endpoint-ip>`. If there are multiple IP addresses for a given *hostname*, then there must be one such `A`record returned for each IP.

- Record Format:
  - `<hostname>.<service>.<ns>.svc.<zone>. <ttl> IN A <endpoint-ip>`
- Question Example:
  - `my-pet.headless.default.svc.cluster.local. IN A`
- Answer Example:
  - `my-pet.headless.default.svc.cluster.local. 4 IN A 10.3.0.100`

#### 2.4.2 - `SRV` Records

For each combination of *ready* endpoint with *hostname* of `<hostname>`, and port in the Service with name `<port>` and number `<port-number>` using protocol `<proto>`, an `SRV` record of the following form must exist.

- Record Format:
  - `_<port>._<proto>.<service>.<ns>.svc.<zone>. <ttl> IN SRV <weight> <priority> <port-number> <hostname>.<service>.<ns>.svc.<zone>.`

This implies that if there are **N** *ready* endpoints and the Service defines **M** named ports, there will be **N** ✖️ **M** `SRV` RRs for the Service.

The priority `<priority>` and weight `<weight>` are numbers as described in [RFC2782](https://tools.ietf.org/html/rfc2782) and whose values are not prescribed by this specification.

Unnamed ports do not have an `SRV` record.

- Question Example:
  - `_https._tcp.headless.default.svc.cluster.local. IN SRV`
- Answer Example:

```
        _https._tcp.headless.default.svc.cluster.local. 4 IN SRV 10 100 443 my-pet.headless.default.svc.cluster.local.
        _https._tcp.headless.default.svc.cluster.local. 4 IN SRV 10 100 443 my-pet-2.headless.default.svc.cluster.local.
        _https._tcp.headless.default.svc.cluster.local. 4 IN SRV 10 100 443 438934893.headless.default.svc.cluster.local.
```

The Additional section of the response may include the `A` records referred to in the `SRV` records.

#### 2.4.3 - `PTR` Records

Given a *ready* endpoint with *hostname* of `<hostname>` and IP address `<a>.<b>.<c>.<d>`, a `PTR` record of the following form must exist.

- Record Format:
  - `<d>.<c>.<b>.<a>.in-addr.arpa. <ttl> IN PTR <hostname>.<service>.<ns>.svc.<zone>.`
- Question Example:
  - `100.0.3.10.in-addr.arpa. IN PTR`
- Answer Example:
  - `100.0.3.10.in-addr.arpa. 14 IN PTR my-pet.headless.default.svc.cluster.local.`

### 2.5 - Records for External Name Services

Given a Service named `<service>` in Namespace `<ns>` with ExternalName `<extname>`, a `CNAME` record named `<service>.<ns>.svc.<zone>` pointing to `<extname>` must exist.

- Record Format:
  - `<service>.<ns>.svc.<zone>. <ttl> IN CNAME <extname>.`
- Question Example:
  - `foo.default.svc.cluster.local. IN A`
- Answer Example:
  - `foo.default.svc.cluster.local. 10 IN CNAME www.example.com.`
  - `www.example.com. 28715 IN A 192.0.2.53`

### 2.6 - Deprecated Records

Kube-DNS versions prior to implementation of this specification also replied with an `A` record of the form below for any values of `<a>`, `<b>`, `<c>`, and `<d>` between 0 and 255:

- Record Format:
  - `<a>-<b>-<c>-<d>.<ns>.pod.<zone>. <ttl> IN A <a>.<b>.<c>.<d>`

This behavior is deprecated but is required to satisfy this specification. It will be removed from a future version of the specification.

## 







