nginx ingress
=============

* Ref: https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/
* Version: v1.6.1


Setup
-----

    export PS1='me:nginx mymac$ '
    multipass launch -n nginx
    multipass shell nginx
    sudo sed -r -i -e 's/^#?DNS=.*/DNS=1.1.1.1/g' /etc/systemd/resolved.conf
    sudo systemctl restart systemd-resolved
    curl -sfL https://get.k3s.io | sh -
    sleep 30
    sudo kubectl delete -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
    logout


kubectl
-------

    multipass exec nginx -- sudo chmod 644 /etc/rancher/k3s/k3s.yaml
    multipass transfer nginx:/etc/rancher/k3s/k3s.yaml k3s.yaml
    IP=$(multipass info nginx | grep IPv4 | awk '{print $2}')
    echo $IP
    sed -i '' "s/127.0.0.1/$IP/" k3s.yaml
    export KUBECONFIG=$PWD/k3s.yaml
    kubectl get nodes


Install
-------

    svn export https://github.com/nginxinc/kubernetes-ingress/tags/v1.6.1/deployments nki
    kubectl apply -f nki/common/ns-and-sa.yaml
    kubectl apply -f nki/rbac/rbac.yaml
    kubectl apply -f nki/common/default-server-secret.yaml
    kubectl apply -f nki/common/nginx-config.yaml
    kubectl apply -f nki/common/custom-resource-definitions.yaml
    kubectl apply -f nki/deployment/nginx-ingress.yaml
    kubectl apply -f nki/service/loadbalancer.yaml
    kubectl -n nginx-ingress rollout status deploy nginx-ingress -w
    kubectl -n nginx-ingress get deploy

* ✓ Official install docs
* ✓ Installs


LoadBalancer
------------

    kubectl -n nginx-ingress get svc
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

* × Ingress object has no ADDRESS.
* ✓ Traffic is routed.


HTTPS
-----

    curl -ksSH 'Host: helloworld' https://$IP

* ✓ Traffic accepted
* ✓ Default certificate
* × No content served


Dashboard
---------

    kubectl -n nginx-ingress port-forward deploy/nginx-ingress 8080:8080
    # Opt-Shift-D
    curl http://localhost:8080/stub_status
    # ^D
    # ^C

* ✓ Enabled by default
* × Missing rich details (available in NGinx Plus)


Metrics
-------

* × Not enabled by default


Canary
------

    more ../app/canary-nginx.yaml
    kubectl apply -f ../app/canary-nginx.yaml
    for i in {1..10}; do echo $(curl -sSH 'Host: helloworld' http://$IP) ; done
    kubectl delete -f ../app/canary-nginx.yaml
    
* ✓ Enabled by default
* × Doesn't work


Performance
-----------

Setup:

    docker run --rm williamyeh/wrk -d 120s -H 'Host: helloworld' http://$IP && echo && kubectl top pod -A

Results:

```
Running 2m test @ http://192.168.64.25
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     6.37ms    1.78ms  31.14ms   79.46%
    Req/Sec   790.25     84.44     0.98k    75.10%
  188834 requests in 2.00m, 41.23MB read
Requests/sec:   1573.00
Transfer/sec:    351.70KB

NAMESPACE       NAME                                      CPU(cores)   MEMORY(bytes)
default         canary-58df596587-jnbx2                   1m           1Mi
default         canary-58df596587-n26kb                   0m           1Mi
default         helloworld-c569b9946-4g489                144m         5Mi
default         helloworld-c569b9946-xnvsz                145m         5Mi
default         prod-644c68fcf4-grgr2                     1m           1Mi
default         prod-644c68fcf4-vzfxl                     0m           1Mi
kube-system     coredns-d798c9dd-lzmgd                    2m           8Mi
kube-system     local-path-provisioner-58fb86bdfd-dh6b4   2m           7Mi
kube-system     metrics-server-6d684c7b5-k7bg7            1m           13Mi
nginx-ingress   nginx-ingress-f599ddd8-rg2cg              285m         12Mi
nginx-ingress   svclb-nginx-ingress-j2dbh                 0m           2Mi
```

Uninstall
---------

    kubectl delete deploy,svc,ing helloworld
    kubectl delete namespace nginx-ingress
    kubectl delete clusterrole nginx-ingress
    kubectl delete clusterrolebinding nginx-ingress
    multipass shell nginx
      sudo crictl rmi gcr.io/google-samples/hello-app:1.0
      sudo crictl rmi gcr.io/google-samples/hello-app:2.0
      sudo crictl rmi docker.io/nginx/nginx-ingress:1.6.1
      logout
    rm -r nki
    multipass delete nginx
    multipass purge
