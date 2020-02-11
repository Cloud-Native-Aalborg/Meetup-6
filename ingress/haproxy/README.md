HAProxy Kubernetes Ingress Controller
=====================================

* Ref: https://github.com/haproxytech/kubernetes-ingress
* Version: v1.3.1


Setup
-----

    export PS1='me:haproxy mymac$ '
    multipass launch -n haproxy
    multipass shell haproxy
    sudo sed -r -i -e 's/^#?DNS=.*/DNS=1.1.1.1/g' /etc/systemd/resolved.conf
    sudo systemctl restart systemd-resolved
    curl -sfL https://get.k3s.io | sh -
    sudo kubectl delete -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    logout


kubectl
-------

    multipass exec haproxy -- sudo chmod 644 /etc/rancher/k3s/k3s.yaml
    multipass transfer haproxy:/etc/rancher/k3s/k3s.yaml k3s.yaml
    IP=$(multipass info haproxy | grep IPv4 | awk '{print $2}')
    echo $IP
    sed -i '' "s/127.0.0.1/$IP/" k3s.yaml
    export KUBECONFIG=$PWD/k3s.yaml
    kubectl get nodes


Install
-------

    kubectl apply -f https://raw.githubusercontent.com/haproxytech/kubernetes-ingress/v1.3.1/deploy/haproxy-ingress.yaml
    kubectl -n haproxy-controller get deploy -w

* ✓ Official install docs
* ✓ Installs


LoadBalancer
------------

* × Not exposed to LoadBalancer.

Fix:

Change from NodePort to LoadBalancer type:

    kubectl -n haproxy-controller get svc
    kubectl -n haproxy-controller patch service haproxy-ingress -p '{"spec":{"type":"LoadBalancer"}}'
    kubectl -n haproxy-controller get svc haproxy-ingress
    #IP=$(ip route show default | awk '{print $9}')
    curl -i http://$IP

* ✓ 404 Status code
* ✓ Descriptive content


Hello World App
---------------

    kubectl apply -f ../app/helloworld.yaml
    kubectl rollout status deploy helloworld -w
    kubectl get deploy,svc,ing helloworld
    #echo $(curl -sSH 'Host: helloworld' http://$IP)
    curl -sSH 'Host: helloworld' http://$IP

* × Ingress object has no ADDRESS.
* ✓ Traffic is routed.


HTTPS
-----

    curl -vksSH 'Host: helloworld' https://$IP

* ✓ Traffic accepted
* × Default certificate
* × No content served


Dashboard
---------

    # dashboard & metrics
    kubectl -n haproxy-controller get svc
    kubectl -n haproxy-controller port-forward service/haproxy-ingress 1024
    # Opt-Shift-D
    curl http://localhost:1024

* ✓ Enabled
* ✓ Rich details


Metrics
-------

Same service as dashboard

    kubectl -n haproxy-controller port-forward service/haproxy-ingress 1024
    curl http://localhost:1024/metrics
    # ^D
    # ^C

* ✓ Enabled
* ✓ Prometheus format


Canary
------

* × Not included.


Performance
-----------

Setup:

    docker run --rm williamyeh/wrk -d 120s -H 'Host: helloworld' http://$IP && echo && kubectl top pod -A

Results:

```
Running 2m test @ http://192.168.64.27
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     4.83ms    4.74ms  86.42ms   86.79%
    Req/Sec     1.24k   172.68     1.70k    68.75%
  297071 requests in 2.00m, 51.85MB read
Requests/sec:   2474.04
Transfer/sec:    442.14KB

NAMESPACE            NAME                                       CPU(cores)   MEMORY(bytes)
default              helloworld-c569b9946-4qjnw                 152m         5Mi
default              helloworld-c569b9946-c8wsl                 152m         5Mi
haproxy-controller   haproxy-ingress-596fb4b4f4-pwfg9           376m         81Mi
haproxy-controller   ingress-default-backend-558fbc9b46-w9spl   1m           1Mi
haproxy-controller   svclb-haproxy-ingress-t22ss                0m           2Mi
kube-system          coredns-d798c9dd-jvdvh                     2m           9Mi
kube-system          local-path-provisioner-58fb86bdfd-lwl8c    2m           8Mi
kube-system          metrics-server-6d684c7b5-xlmqd             1m           14Mi
```


Uninstall
---------

    kubectl delete deploy,svc,ing helloworld
    kubectl delete -f https://raw.githubusercontent.com/haproxytech/kubernetes-ingress/v1.3.1/deploy/haproxy-ingress.yaml

