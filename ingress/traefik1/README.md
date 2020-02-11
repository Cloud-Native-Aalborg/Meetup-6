K3S Built-in Traefik 1.7
========================

* Ref: https://rancher.com/docs/k3s/latest/en/advanced/#auto-deploying-manifests
* Version: 1.7.19


Setup
-----

    export PS1='me:traefik1 mymac$ '
    multipass launch -n traefik1
    multipass shell traefik1
    sudo sed -r -i -e 's/^#?DNS=.*/DNS=1.1.1.1/g' /etc/systemd/resolved.conf
    sudo systemctl restart systemd-resolved
    curl -sfL https://get.k3s.io | sh -
    sleep 30
    sudo kubectl delete -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    logout


kubectl
-------

    multipass exec traefik1 -- sudo chmod 644 /etc/rancher/k3s/k3s.yaml
    multipass transfer traefik1:/etc/rancher/k3s/k3s.yaml k3s.yaml
    IP=$(multipass info traefik1 | grep IPv4 | awk '{print $2}')
    sed -i '' "s/127.0.0.1/$IP/" k3s.yaml
    export KUBECONFIG=$PWD/k3s.yaml
    kubectl get nodes


Install
-------

    #multipass exec traefik1 -- \
    #  sudo kubectl apply -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    multipass shell traefik1
    sudo kubectl apply -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    kubectl -n kube-system rollout status deploy traefik -w
    kubectl -n kube-system get deploy traefik
    logout

* ✓ Bundled
* ✓ Installs


LoadBalancer
------------

    kubectl -n kube-system get svc -l app=traefik
    curl -i http://$IP

* ✓ 404 Status code
* ✓ Descriptive error


Hello-World
-----------

    more ../app/helloworld.yaml
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
* ✓ Content served


Dashboard
---------

    kubectl -n kube-system get svc

* × Not enabled


Metrics
-------

    kubectl -n kube-system port-forward service/traefik-prometheus 9100
    # Opt-Shift-D
    curl http://localhost:9100/metrics
    # ^D
    # ^C

* ✓ Enabled
* ✓ Prometheus format


Canary
------

    more ../app/canary-traefik1.yaml
    kubectl apply -f ../app/canary-traefik1.yaml
    kubectl rollout status deploy canary -w
    kubectl rollout status deploy canary2 -w
    kubectl get deploy,svc canary canary2
    kubectl describe ingress canary
    for i in {1..10}; do echo $(curl -sSH 'Host: canary' http://$IP) ; done
    kubectl delete -f ../app/canary-traefik1.yaml

* ✓ Enabled
* ✓ Works


Performance
-----------

Setup:

     docker run --rm williamyeh/wrk -d 120s -H 'Host: helloworld' http://$IP && echo && kubectl top pod -A

Results:

```
Running 2m test @ http://192.168.64.26
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     5.24ms    2.25ms  32.29ms   77.65%
    Req/Sec     0.97k   113.40     1.27k    70.48%
  232696 requests in 2.00m, 45.71MB read
Requests/sec:   1938.19
Transfer/sec:    389.91KB

NAMESPACE     NAME                                      CPU(cores)   MEMORY(bytes)
default       helloworld-c569b9946-2rlp8                113m         6Mi
default       helloworld-c569b9946-chxvq                111m         9Mi
kube-system   coredns-d798c9dd-z6h8b                    2m           8Mi
kube-system   local-path-provisioner-58fb86bdfd-qjnjp   2m           9Mi
kube-system   metrics-server-6d684c7b5-jwt7p            1m           10Mi
kube-system   svclb-traefik-gvpcl                       0m           1Mi
kube-system   traefik-6787cddb4b-vg98n                  557m         24Mi
```

* ✓ Results can be obtained


Uninstall
---------

    kubectl delete -f ../app/canary-traefik1.yaml
    kubectl delete deploy,svc,ing helloworld
    multipass exec traefik1 -- \
      sudo kubectl delete -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    multipass delete traefik1
    multipass purge
