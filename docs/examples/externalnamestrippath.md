# Expose an external application with a path, that is stripped by KongIngress config.

This example shows how we can expose a service located outside the Kubernetes cluster using an Ingress rule similar to the Kong [Getting Started guide](0)
We then extend the example with a custom path on the ingress resource that will be stripped by kong before the request reaches the upstream.

Requirements:

- working Kubernetes cluster
- Kong Ingress controller installed. Please check the [deploy guide](1)

1. Create a Kubernetes service

First we need to create a Kubernetes Service [type=ExternalName](2) using the hostname of the application we want to expose

```bash
echo "
kind: Service
apiVersion: v1
metadata:
  name: proxy-to-mockbin
spec:
  ports:
  - protocol: TCP
    port: 80
  type: ExternalName
  externalName: mockbin.org
" | kubectl create -f -
```

2. Configure a [request-transformer](3) plugin to remove the Host header from the original request.

This removes the Host header so when the traffic reaches `mockbin.org` does not contains `foo.bar`

```bash
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: transform-request-to-mockbin
config:
  remove:
    headers: host
" | kubectl create -f -
```

3. Create an Ingress to expose the service in the host `foo.bar`. With an annotation referencing the KongIngress.

```bash
echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: proxy-from-k8s-to-mockbin
  annotations:
    ingress.plugin.konghq.com: proxy-from-k8s-to-mockbin
    request-transformer.plugin.konghq.com: |
      transform-request-to-mockbin
spec:
  rules:
  - host: foo.bar
    http:
      paths:
      - path: /mockbin
        backend:
          serviceName: proxy-to-mockbin
          servicePort: 80
" | kubectl create -f -
```

4. Add a corresponding KongIngress to activate the strip_path functionality so that the curl request gets proxied to `mockbin.org` instead of `mockbin.org/mockbin`.

```bash
echo "
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: proxy-from-k8s-to-mockbin
upstream:
  hash_on: none
  hash_fallback: none
  healthchecks:
    active:
      concurrency: 10
      healthy:
      http_statuses:
        - 200
        - 302gi
      interval: 0
      successes: 0
      http_path: "/mockbin"
      timeout: 1
      unhealthy:
        http_failures: 0
        http_statuses:
        - 429
        interval: 0
        tcp_failures: 0
        timeouts: 0
    passive:
      healthy:
      http_statuses:
        - 200
      successes: 0
      unhealthy:
        http_failures: 0
        http_statuses:
        - 429
        - 503
        tcp_failures: 0
        timeouts: 0
    slots: 10
proxy:
  path: /
  connect_timeout: 10000
  retries: 10
  read_timeout: 10000
  write_timeout: 10000
route:
  methods:
  - POST
  - GET
  - DELETE
  - HEAD
  regex_priority: 0
  strip_path: true
  preserve_host: false
" | kubectl create -f -

```

5. Now we can test the service running:

```bash
export KONG_ADMIN_PORT=$(minikube service -n kong kong-ingress-controller --url --format "{{ .Port }}")
export KONG_ADMIN_IP=$(minikube service   -n kong kong-ingress-controller --url --format "{{ .IP }}")
export PROXY_IP=$(minikube   service -n kong kong-proxy --url --format "{{ .IP }}" | head -1)
export HTTP_PORT=$(minikube  service -n kong kong-proxy --url --format "{{ .Port }}" | head -1)
export HTTPS_PORT=$(minikube service -n kong kong-proxy --url --format "{{ .Port }}" | tail -1)

curl ${PROXY_IP}:${HTTP_PORT}/mockbin -H "Host:foo.bar"

```

6. What is configured in Kong?

```bash
curl ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/routes/ |jq .
curl ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/services/ |jq .
curl ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/upstreams/ |jq .
curl ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/upstreams/default.proxy-to-mockbin.80/targets |jq .
```

[0]: https://getkong.org/docs/0.13.x/getting-started/configuring-a-service/
[1]: https://github.com/Kong/kubernetes-ingress-controller/tree/master/deploy
[2]: https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors
[3]: https://getkong.org/plugins/request-transformer/
