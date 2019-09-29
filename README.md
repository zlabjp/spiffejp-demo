# spiffejp-demo

This repository contains the manifests used in the demonstration of `Securing the Service Mesh with SPIRE` at [SPIFFE Meetup Tokyo #2](https://spiffe-jp.connpass.com/event/142393/) . The manifests have been created based on [spiffe/spire-examples](https://github.com/spiffe/spire-examples/tree/54dbcbc8c378bb11c5040eb261ec51bc82f8a79b) . Thanks, SPIFFE Community!

- Use SPIRE 0.8.1 and Envoy 1.11.1
- Use [NodeAttestor "k8s_psat"](https://github.com/spiffe/spire/blob/0.8.1/doc/plugin_server_nodeattestor_k8s_psat.md)
- Enable Envoy SDS Support

## Overview

![figure](/img/figure.png)

Three services are running.

- API（ `spiffe://example.org/api` ）
    - Simple API that returns ok using [Mmock](https://github.com/jmartin82/mmock)
    - mTLS authentication with Valid API Client is allowed
- Valid API Client（ `spiffe://example.org/valid-api-client` ）
    - curl container
    - mTLS authentication with API is allowed
- Invalid API Client（ `spiffe://example.org/invalid-api-client` ）
    - curl container
    - mTLS authentication with API is **NOT** allowed

Certificate for mtls authentication is delivered from SPIRE Agent to Envoy via SDS Server.

## 1. Create Kubernetes Cluster

Create a Kubernetes cluster with the required flags to run NodeAttestor "k8s_psat".

```bash
minikube start \
    --extra-config=apiserver.authorization-mode=RBAC \
    --extra-config=apiserver.service-account-signing-key-file=/var/lib/minikube/certs/sa.key \
    --extra-config=apiserver.service-account-key-file=/var/lib/minikube/certs/sa.pub \
    --extra-config=apiserver.service-account-issuer=api \
    --extra-config=apiserver.service-account-api-audiences=api,spire-server
```

## 2. Deploy SPIRE Server

Deploy SPIRE Server as StatefulSet.

```bash
kubectl apply -f spire-server.yaml
```

## 3. Deploy SPIRE Agent

Deploy SPIRE Agent as DaemonSet.

```bash
kubectl apply -f spire-agent.yaml
```

## 4. Create Registration Entries

Create node entry.

```bash
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/node \
    -selector k8s_psat:cluster:demo-cluster \
    -node
```

Create workload entries.

```bash
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -parentID spiffe://example.org/node \
    -spiffeID spiffe://example.org/api \
    -selector k8s:pod-label:app:api \
    -ttl 30

kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -parentID spiffe://example.org/node \
    -spiffeID spiffe://example.org/valid-api-client \
    -selector k8s:pod-label:app:valid-api-client \
    -ttl 30

kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -parentID spiffe://example.org/node \
    -spiffeID spiffe://example.org/invalid-api-client \
    -selector k8s:pod-label:app:invalid-api-client \
    -ttl 30
```

Confirm the created registration entries.

```bash
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry show
```

## 5. Confirm that API and Valid API Client can mTLS authenticate

### Deploy API

```bash
kubectl apply -f api.yaml
```

Confirm that Envoy has obtained the following certificates from SPIRE Agent via SDS Server.

- Trust Bundle as ca certificate
- X.509-SVID as server certificate ( `spiffe://example.org/api` )

```bash
API_ENVOY_ADMIN_NODE_PORT=$(kubectl get svc api -o jsonpath='{.spec.ports[?(@.name=="envoy-admin-port")].nodePort}')
curl $(minikube ip):${API_ENVOY_ADMIN_NODE_PORT}/certs
```

### Deploy Valid API Client

```bash
kubectl apply -f valid-api-client.yaml
```

Confirm that Envoy has obtained the following certificates from SPIRE Agent via SDS Server.

- Trust Bundle as ca certificate
- X.509-SVID as client certificate ( `spiffe://example.org/valid-api-client` )

```bash
VALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT=$(kubectl get svc valid-api-client -o jsonpath='{.spec.ports[?(@.name=="envoy-admin-port")].nodePort}')
curl $(minikube ip):${VALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT}/certs
```

### Confirm that API and Valid API Client can mTLS authenticate

Request from Valid API Client to API via Envoy.

```bash
VALID_API_CLIENT_POD=$(kubectl get pods -l app=valid-api-client -o jsonpath='{.items[0].metadata.name}')
kubectl exec ${VALID_API_CLIENT_POD} -c curl -- curl -s 127.0.0.1:8000
```

Mutual authentication succeeds and the following response is returned.

```bash
ok
```

## 6. Confirm that API and Invalid API Client can *NOT* mTLS authenticate

### Deploy Invalid API Client

```bash
kubectl apply -f invalid-api-client.yaml
```

Confirm that Envoy has obtained the following certificates from SPIRE Agent via SDS Server.

- Trust Bundle as ca certificate
- X.509-SVID as client certificate ( `spiffe://example.org/invalid-api-client` )

```bash
INVALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT=$(kubectl get svc invalid-api-client -o jsonpath='{.spec.ports[?(@.name=="envoy-admin-port")].nodePort}')
curl $(minikube ip):${INVALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT}/certs
```

### Confirm that API and Invalid API Client can *NOT* mTLS authenticate

Request from Invalid API Client to API via Envoy.

```bash
INVALID_API_CLIENT_POD=$(kubectl get pods -l app=invalid-api-client -o jsonpath='{.items[0].metadata.name}')
kubectl exec ${INVALID_API_CLIENT_POD} -c curl -- curl -s 127.0.0.1:8000
```

Mutual authentication fails and the following response is returned

```bash
upstream connect error or disconnect/reset before headers. reset reason: connection failure
```

Since the details of `upstream connect error` are not included in the above response (client certificate error is expected), change the log level of Envoy and debug.

```bash
INVALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT=$(kubectl get svc invalid-api-client -o jsonpath='{.spec.ports[?(@.name=="envoy-admin-port")].nodePort}')
curl -X POST "$(minikube ip):${INVALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT}/logging?connection=debug"
```

Request again after changing log level.

```bash
INVALID_API_CLIENT_POD=$(kubectl get pods -l app=invalid-api-client -o jsonpath='{.items[0].metadata.name}')
kubectl exec ${INVALID_API_CLIENT_POD} -c curl -- curl -s 127.0.0.1:8000
```

Confirm Envoy log.

```bash
INVALID_API_CLIENT_POD=$(kubectl get pods -l app=invalid-api-client -o jsonpath='{.items[0].metadata.name}')
kubectl logs ${INVALID_API_CLIENT_POD} -c envoy -f
```

The following log shows that mutual authentication failed due to client certificate error.

```
[2019-10-01 05:16:35.506][11][debug][connection] [source/extensions/transport_sockets/tls/ssl_socket.cc:201] [C10] TLS error: 268436502:SSL routines:OPENSSL_internal:SSLV3_ALERT_CERTIFICATE_UNKNOWN
```

## 7. Confirm that SPIRE Agent rotates certificates in Envoy

In `4. Create Registration Entries`, the entries were registered so that the SVID rotates every 30 seconds, so you can confirm that the rotation is executed in the SPIRE Agent log.

```bash
SPIRE_AGENT_POD=$(kubectl get pods -n spire -l app=spire-agent -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n spire ${SPIRE_AGENT_POD} -f
```

The following log will be output.

```bash
# Renewing X509-SVID
time="2019-10-01T06:12:40Z" level=info msg="Renewing X509-SVID" expires_at="2019-10-01T06:13:00Z" spiffe_id="spiffe://example.org/api" subsystem_name=manager
time="2019-10-01T06:12:40Z" level=info msg="Renewing X509-SVID" expires_at="2019-10-01T06:13:00Z" spiffe_id="spiffe://example.org/valid-api-client" subsystem_name=manager
time="2019-10-01T06:12:40Z" level=info msg="Renewing X509-SVID" expires_at="2019-10-01T06:13:00Z" spiffe_id="spiffe://example.org/invalid-api-client" subsystem_name=manager
time="2019-10-01T06:12:40Z" level=debug msg="Entry updated" entry=eaaf39ca-a387-432b-a19d-f597280a32fc spiffe_id="spiffe://example.org/api" subsystem_name=cache_manager svid_updated=true
time="2019-10-01T06:12:40Z" level=debug msg="Entry updated" entry=965f582a-299f-4187-bd4d-58fce2de5160 spiffe_id="spiffe://example.org/valid-api-client" subsystem_name=cache_manager svid_updated=true
time="2019-10-01T06:12:40Z" level=debug msg="Entry updated" entry=172b6844-f1d0-4877-8902-d83aed0cc6a8 spiffe_id="spiffe://example.org/invalid-api-client" subsystem_name=cache_manager svid_updated=true

# Received StreamSecrets request from Envoy
time="2019-10-01T06:12:40Z" level=debug msg="Received StreamSecrets request" nonce=e5f75599 resource_names="[spiffe://example.org]" subsystem_name=sds_api version_info=67
time="2019-10-01T06:12:40Z" level=debug msg="Received StreamSecrets request" nonce=db7e5697 resource_names="[spiffe://example.org/api]" subsystem_name=sds_api version_info=67
time="2019-10-01T06:12:40Z" level=debug msg="Received StreamSecrets request" nonce=65a4585c resource_names="[spiffe://example.org]" subsystem_name=sds_api version_info=58
time="2019-10-01T06:12:40Z" level=debug msg="Received StreamSecrets request" nonce=cd2a0db6 resource_names="[spiffe://example.org/valid-api-client]" subsystem_name=sds_api version_info=58
time="2019-10-01T06:12:40Z" level=debug msg="Received StreamSecrets request" nonce=a9f10bc3 resource_names="[spiffe://example.org]" subsystem_name=sds_api version_info=54
time="2019-10-01T06:12:40Z" level=debug msg="Received StreamSecrets request" nonce=746562b9 resource_names="[spiffe://example.org/invalid-api-client]" subsystem_name=sds_api version_info=54

# Sending StreamSecrets response to Envoy
time="2019-10-01T06:12:40Z" level=debug msg="Sending StreamSecrets response" count=1 nonce=db7e5697 subsystem_name=sds_api version_info=67
time="2019-10-01T06:12:40Z" level=debug msg="Sending StreamSecrets response" count=1 nonce=e5f75599 subsystem_name=sds_api version_info=67
time="2019-10-01T06:12:40Z" level=debug msg="Sending StreamSecrets response" count=1 nonce=cd2a0db6 subsystem_name=sds_api version_info=58
time="2019-10-01T06:12:40Z" level=debug msg="Sending StreamSecrets response" count=1 nonce=65a4585c subsystem_name=sds_api version_info=58
time="2019-10-01T06:12:40Z" level=debug msg="Sending StreamSecrets response" count=1 nonce=746562b9 subsystem_name=sds_api version_info=54
time="2019-10-01T06:12:40Z" level=debug msg="Sending StreamSecrets response" count=1 nonce=a9f10bc3 subsystem_name=sds_api version_info=54
```

You can also confirm that the rotation is executed from the `valid_from` and `expiration_time` values in Envoy's Administration interface `/certs`.

```bash
# API
API_ENVOY_ADMIN_NODE_PORT=$(kubectl get svc api -o jsonpath='{.spec.ports[?(@.name=="envoy-admin-port")].nodePort}')
curl $(minikube ip):${API_ENVOY_ADMIN_NODE_PORT}/certs

# Valid API Client
VALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT=$(kubectl get svc valid-api-client -o jsonpath='{.spec.ports[?(@.name=="envoy-admin-port")].nodePort}')
curl $(minikube ip):${VALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT}/certs

# Invalid API Client
INVALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT=$(kubectl get svc invalid-api-client -o jsonpath='{.spec.ports[?(@.name=="envoy-admin-port")].nodePort}')
curl $(minikube ip):${INVALID_API_CLIENT_ENVOY_ADMIN_NODE_PORT}/certs
```
