Gloo & Knative
==============

Ref: 

* Version: 1.3.4
* https://docs.solo.io/gloo/latest/gloo_integrations/ingress
* https://knative.dev/docs/install/knative-with-gloo/


Setup
-----

    export PS1='me:gloo mymac$ '
    multipass launch -n gloo
    multipass shell gloo
    sudo sed -r -i -e 's/^#?DNS=.*/DNS=1.1.1.1/g' /etc/systemd/resolved.conf
    sudo systemctl restart systemd-resolved
    curl -sfL https://get.k3s.io | sh -
    sudo kubectl delete -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    logout


kubectl
-------

    multipass exec gloo -- sudo chmod 644 /etc/rancher/k3s/k3s.yaml
    multipass transfer gloo:/etc/rancher/k3s/k3s.yaml k3s.yaml
    IP=$(multipass info gloo | grep IPv4 | awk '{print $2}')
    echo $IP
    sed -i '' "s/127.0.0.1/$IP/" k3s.yaml
    export KUBECONFIG=$PWD/k3s.yaml
    kubectl get nodes
    sudo id
    brew update


Install
-------

Use glooctl to install ingress controller

    brew install glooctl
    glooctl install ingress
    kubectl -n gloo-system get deploy -w

* ✓ Official install docs
* ✓ Installs


LoadBalancer
------------

    kubectl -n gloo-system get svc
    curl -i http://$IP

* × 404 Status code
* × No descriptive content


Hello-World
-----------

    kubectl apply -f ../app/helloworld.yaml
    kubectl rollout status deploy helloworld -w
    kubectl get deploy,svc,ing helloworld
    #echo $(curl -sSH 'Host: helloworld' http://$IP)
    curl -sSH 'Host: helloworld' http://$IP

* × Ingress object has no ADDRESS.
* ✓ Traffic is routed.


HTTPS
-----

    curl -ksSH 'Host: helloworld' https://$IP

* × Traffic accepted
* × Default certificate
* × No content served


Dashboard
---------

* × Not available. (Coming in future version).


Metrics
-------

    kubectl -n gloo-system get svc
    kubectl -n gloo-system port-forward deploy/discovery 9091
    curl http://localhost:9091/metrics

* ✓ Enabled
* ✓ Prometheus format


Canary
------

* × Not included. Have to use Flagger instead.


Performance
-----------

Setup:

    docker run --rm williamyeh/wrk -d 120s -H 'Host: helloworld' http://$IP && echo && kubectl top pod -A

Results:

```
Running 2m test @ http://192.168.64.23
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     4.02ms    1.67ms  52.10ms   78.84%
    Req/Sec     1.26k   135.13     1.58k    74.02%
  301661 requests in 2.00m, 66.74MB read
Requests/sec:   2512.61
Transfer/sec:    569.27KB

NAMESPACE     NAME                                      CPU(cores)   MEMORY(bytes)
default       helloworld-c569b9946-8dczb                158m         6Mi
default       helloworld-c569b9946-sv9fg                157m         5Mi
gloo-system   discovery-5f97667c4c-q4b2z                4m           27Mi
gloo-system   gloo-d7c867cf4-dlr4q                      4m           29Mi
gloo-system   ingress-fb4bcff75-gc9wn                   2m           22Mi
gloo-system   ingress-proxy-69568c6649-4zzcj            387m         16Mi
gloo-system   svclb-ingress-proxy-4l2bt                 0m           1Mi
kube-system   coredns-d798c9dd-kcddw                    2m           11Mi
kube-system   local-path-provisioner-58fb86bdfd-4hr2d   2m           7Mi
kube-system   metrics-server-6d684c7b5-9w942            1m           14Mi
```


Uninstall
---------

    glooctl uninstall
    kubectl delete deploy,svc,ing helloworld
    multipass shell gloo
      sudo crictl rmi $(sudo crictl images | egrep 'solo|google' | awk '{print $3}')
    brew uninstall glooctl
    multipass delete gloo
    multipass purge

