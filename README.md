# DevOps Task Home Assignment #


## Summary  ##
Here is a description of our home assignment for a Senior DevOps position.
The task includes several steps: In the first step, you will need to set up 
a local Kubernetes Cluster together with an ingress controller.
Next, you will need to install the http-echo helm chart package 
and fix the environment (troubleshooting). 
The last step is to add https support.

### Step 1: Installation and deployment  ###
Here you need to set up a local Kubernetes cluster using minikube (or any other tool)
and install the ingress controller and helm. Finally, you will need to deploy a simple
http-echo helm chart package


* Create local Kubernetes Cluster (you can use [minikube](https://minikube.sigs.k8s.io/docs/start/) / [MicroK8s](https://microk8s.io/docs) / etc.)
* Install [Helm](https://helm.sh/docs/intro/install/)
* Deploy http-echo helm chart package

### Step 2:  Fix Environment  ###
The Helm package is **broken** you need to investigate what went wrong
and fix the environment. Below you can see instructions on how to check
if the environment is up and running.

* how to make sure the deployment is fully working:
    * All the pods and services are up and running without any errors
    * The echo server is up and running:
        * Add to /etc/hosts the following mapping: 127.0.0.1 http-echo.k8s-server.local(In case of Minikube add ```minikube ip``` instead of 127.0.0.1)
        * Open browser at http://http-echo.k8s-server.local/ or curl to the url. You should get a json response that describe the request:
    ```
    {"host":{"hostname":"http-echo.k8s-server.local","ip":"::ffff:10.1.0.14","ips":[]},"http":
  {"method":"GET","baseUrl":"","originalUrl":"/","protocol":"http"},"request":{"params":
  {"0":"/"},"query":{},"cookies":{},"body":{},"headers":{"host":"http-echo.k8s-server.local"
  ,"x-request-id":"cc0fb6714354cf630bcc345cd0008c56","x-real-ip":"192.168.65.3",
  "x-forwarded-for":"192.168.65.3","x-forwarded-host":"http-echo.k8s-server.local",
  "x-forwarded-port":"80","x-forwarded-proto":"http","x-forwarded-scheme":"http",
  "x-scheme":"http","upgrade-insecure-requests":"1","user-agent":"Mozilla/5.0 (Macintosh;Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36","accept":"text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,/;q=0.8,application/signed-exchange;v=b3;q=0.9","accept-encoding":"gzip, deflate","accept-language":"en-US,en;q=0.9","apikey":"JdazXPS4F4rySrosZU/uzfnDu2vBUlyu0zJMgkC6LEQ="}},"environment":{"PATH":"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","HOSTNAME":"http-echo-69f585ff95-srk6l","HTTP_ECHO_SERVICE_HOST":"10.102.9.43","HTTP_ECHO_PORT":"tcp://10.102.9.43:5678","HTTP_ECHO_PORT_5678_TCP_PROTO":"tcp","KUBERNETES_PORT_443_TCP_PORT":"443","HTTP_ECHO_PORT_5678_TCP":"tcp://10.102.9.43:5678","HTTP_ECHO_PORT_5678_TCP_PORT":"5678","HTTP_ECHO_PORT_5678_TCP_ADDR":"10.102.9.43","KUBERNETES_SERVICE_PORT_HTTPS":"443","KUBERNETES_PORT":"tcp://10.96.0.1:443","KUBERNETES_PORT_443_TCP_ADDR":"10.96.0.1","HTTP_ECHO_SERVICE_PORT":"5678","HTTP_ECHO_SERVICE_PORT_HTTP":"5678","KUBERNETES_SERVICE_HOST":"10.96.0.1","KUBERNETES_SERVICE_PORT":"443","KUBERNETES_PORT_443_TCP":"tcp://10.96.0.1:443","KUBERNETES_PORT_443_TCP_PROTO":"tcp","NODE_VERSION":"12.19.0","YARN_VERSION":"1.22.5","HOME":"/root"}}
    ```


### Step 3:  (Bonus) Add feature  ###

Please add TLS  support to nginx ingress bind to port 443

#### how to make sure the TLS is fully working?  ####
Do the same test from previous step with https (https://http-echo.k8s-server.local/)

### Guidelines and Preuest Information ###

* Please treat your solution as a task that need to be deployed in production, so we expect to a high level of maturity of the solution 
* Pay attention to the performance of the solution you use and make sure it runs efficiently with minimal overhead as possible. 
* The task should take about 2-4 hours. 
* Please read the instructions carefully. 
* If there is anything incomprehensible - please contact us for clarifications.

### Submission ###
* Please provide a working solution, **submitted as a new git local branch**, and in the comment please provide a short description which explains any important issue or comments about your work 



---

## My solution

I fixed the chart's port configuration, probes, Service protocol, resource requests, and deprecated HPA API. I added TLS through cert-manager and basic container hardening. The commands below reproduce the lab on macOS with Docker Desktop.

### How to run it

```bash
# cluster and ingress controller
minikube start --driver=docker
minikube addons enable ingress

# cert-manager and the local CA (needed for https)
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm upgrade --install cert-manager jetstack/cert-manager --version v1.21.2 \
  -n cert-manager --create-namespace -f deploy/cert-manager/values.yaml --wait
kubectl apply -f deploy/cert-manager/cluster-issuer.yaml

# the app
helm upgrade --install http-echo . -n http-echo --create-namespace --wait
kubectl -n http-echo wait --for=condition=Ready certificate/http-echo-tls --timeout=120s
helm test http-echo -n http-echo
```

On macOS with the docker driver, `minikube ip` isn't reachable from the host. So run `minikube tunnel` in a separate terminal (it asks for sudo) and add `127.0.0.1 http-echo.k8s-server.local` to `/etc/hosts`. Then:

```bash
kubectl -n http-echo get secret http-echo-tls -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
curl -L --cacert ca.crt http://http-echo.k8s-server.local/
```

HTTP redirects to HTTPS. Curl needs `-L` to follow the redirect and `--cacert` to trust the lab CA.

### What was broken

The original chart, deployed as-is:

```console
$ kubectl -n http-echo get pods,svc
pod/http-echo-75d449b66-sn7l8   0/1     Running   0   60s
service/http-echo   ClusterIP   10.97.112.141   5678/UDP

$ kubectl -n http-echo describe pod ...    # Events
Liveness probe errored: exec: "/bin/bash": stat /bin/bash: no such file or directory
Readiness probe failed: ... dial tcp 10.244.0.6:5678: connect: connection refused

$ kubectl -n http-echo logs ...
Listening on port 80.
```

1. **Wrong port.** echo-server reads its port from the `PORT` env var and ignored the chart's `-port` argument, so it listened on 80 instead of 5678. I now set `PORT` from a `containerPort` value.
2. **Liveness probe used `/bin/bash`**, which the Alpine image doesn't have. Both probes are now HTTP checks on `/ping`.
3. **The Service was UDP.** HTTP needs TCP.
4. **Resource requests of 6 CPUs and 1Mi memory.** Changed to 50m CPU and 128Mi requests with a 256Mi memory limit. That's a starting point based on idle usage (about 26Mi), not a load test. I left out a CPU limit to avoid throttling short bursts.
5. **The HPA used `autoscaling/v2beta1`**, which was removed in Kubernetes 1.25. Moved to `autoscaling/v2`. Autoscaling stays off: there's no metrics-server here, and 2 replicas are enough.
6. **The helm test used an unpinned busybox image** and never cleaned up. I pinned it and added a delete policy.

### Other changes

- **Hardening:** the container runs as non-root with a read-only filesystem, no capabilities and no service account token. I turned off echo-server's `?echo_file=` feature, which lets anyone read files from the container.
- **Availability:** 2 replicas, rolling updates that keep both serving, and a PodDisruptionBudget. On minikube both pods share one node, so this only shows the setup.
- **NetworkPolicy:** only the ingress controller can reach the pods, plus pods labelled `http-echo-client=true` (used by the helm test). Outgoing traffic is blocked. I tested it on minikube 1.39, Kubernetes 1.37 and kindnet `v20260820`, which enforces it. If yours doesn't, use `minikube start --cni=calico`.

### TLS

cert-manager creates a local CA (`deploy/cert-manager/`) and issues a 90-day certificate for the Ingress, which it renews automatically. The local CA is only for this lab, because no public CA issues certificates for `.local`. That's why the check uses `curl --cacert` instead of trusting the CA on the machine. In production you'd need a real domain and an issuer like Let's Encrypt.

### Limitations

Everything runs on one node, so the availability settings aren't really exercised. Resource values aren't load-tested. The local CA has no tested rollover plan. echo-server echoes request headers and its environment, so it shouldn't be given secrets or be exposed publicly. There's no monitoring yet.
