Traefik 2
---------

* Ref: https://docs.traefik.io/getting-started/install-traefik/
* Version: v2.1.3


Setup
-----

    export PS1='me:traefik2 mymac$ '
    multipass launch -n traefik2
    multipass shell traefik2
    sudo sed -r -i -e 's/^#?DNS=.*/DNS=1.1.1.1/g' /etc/systemd/resolved.conf
    sudo systemctl restart systemd-resolved
    curl -sfL https://get.k3s.io | sh -
    sleep 30
    sudo kubectl delete -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    logout


kubectl
-------

    multipass exec traefik2 -- sudo chmod 644 /etc/rancher/k3s/k3s.yaml
    multipass transfer traefik2:/etc/rancher/k3s/k3s.yaml k3s.yaml
    IP=$(multipass info traefik2 | grep IPv4 | awk '{print $2}')
    sed -i '' "s/127.0.0.1/$IP/" k3s.yaml
    export KUBECONFIG=$PWD/k3s.yaml
    multipass exec traefik2 -- sudo kubectl delete -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    kubectl get nodes


Install
-------

Custom manifest file.

     more traefik2.yaml
     kubectl apply -f traefik2.yaml
     kubectl -n traefik rollout status deploy traefik -w
     kubectl -n traefik get deploy

* × Official install docs
* ✓ Installs


LoadBalancer
------------

    kubectl -n traefik get svc
    curl -i http://$IP

* ✓ 404 Status code
* ✓ Descriptive content


Hello-World
-----------

    kubectl apply -f ../app/helloworld.yaml
    kubectl rollout status deploy helloworld -w
    kubectl get deploy,svc,ing helloworld
    #echo $(curl -sSH 'Host: helloworld' http://$IP)
    curl -sSH 'Host: helloworld' http://$IP

* ✓ Ingress has ADDRESS.
* ✓ Ingress works.


HTTPS
-----

    curl -vksSH 'Host: helloworld' https://$IP

* ✓ Traffic accepted
* ✓ Default certificate
* × No content served


Dashboard
---------

    kubectl -n traefik get svc
    kubectl -n traefik port-forward svc/traefik-dashboard 8080
    # Opt-Shift-D
    curl http://localhost:8080/dashboard
    # ^D
    # ^C

* ✓ Enabled
* ✓ Rich details


Metrics
-------

    kubectl -n traefik port-forward service/traefik-dashboard 8080
    curl http://localhost:8080/metrics

* ✓ Enabled
* ✓ Prometheus format


Canary
------

* × Doesn't work. Have to use CRD middleware instead.


Performance
-----------

Setup:

    docker run --rm williamyeh/wrk -d 120s -H 'Host: helloworld' http://$IP && echo && kubectl top pod -A

Results:

```
Running 2m test @ http://192.168.64.24
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     6.71ms    2.82ms  39.06ms   74.09%
    Req/Sec   756.83     78.33     0.94k    73.02%
  180844 requests in 2.00m, 31.56MB read
Requests/sec:   1506.52
Transfer/sec:    269.23KB

NAMESPACE     NAME                                      CPU(cores)   MEMORY(bytes)
default       helloworld-c569b9946-gnvv2                91m          5Mi
default       helloworld-c569b9946-s9j72                90m          5Mi
kube-system   coredns-d798c9dd-6bbxp                    2m           15Mi
kube-system   local-path-provisioner-58fb86bdfd-nxhkc   1m           8Mi
kube-system   metrics-server-6d684c7b5-mn6kx            1m           13Mi
traefik       svclb-traefik-ingress-6v6rn               0m           1Mi
traefik       traefik-774fbb9f48-sfk4x                  502m         9Mi
```

* ✓ Results can be obtained


Uninstall
---------

    kubectl delete deploy,svc,ing helloworld
    kubectl delete -f traefik2.yaml
