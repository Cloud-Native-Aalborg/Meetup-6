Ingress Controller Evaluation
=============================

Environment
-----------

Rancher K3S in Canonical Multipass

### Edit Session -> Window -> Transparency: 0
### Terminal Prompt
### Resize to 87x40 chars

```
export PS1='me:~ mymac$ '
export PS1='me:traefik1 mymac$ '
sudo id
brew update
cd /tmp
clear
```

##* Multipass

```
brew cask install multipass
multipass ls
multipass launch -n vm01
multipass ls
multipass info vm01
```

### VM

```
multipass shell vm01
```


### DNS

```
dig dr.dk
systemd-resolve --status | grep Servers
sudo sed -r -i -e 's/^#?DNS=.*/DNS=1.1.1.1/g' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
systemd-resolve --status | grep Servers
dig dr.dk
```

### K3S

```
curl -sfL https://get.k3s.io | sh -
kubectl get nodes
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
kubectl get nodes
```

### Cri-tools

```
docker ps
sudo crictl ps
sudo crictl images
```

### kubectl

```
logout
multipass transfer vm01:/etc/rancher/k3s/k3s.yaml k3s.yaml
multipass ls
IP=$(multipass info vm01 | grep IPv4 | awk '{print $2}')
echo $IP
sed -i '' "s/127.0.0.1/$IP/" k3s.yaml
export KUBECONFIG=$PWD/k3s.yaml
brew install kubectl
kubectl get pod -A
```


### Performance monitoring

```
kubectl -n kube-system patch deployment metrics-server -p '{"spec":{"template":{"spec":{"containers":[{"name":"metrics-server","args":["--metric-resolution=2s"]}]}}}}'
watch --no-title -d kubectl top pod -A
```

```
watch -d --no-title multipass info vm01
```


Evaluation
==========

Criteria:

* Load Balancer reporting
* http vs https
* dashboard
* metrics
* canary
* scaling
* performance
* knative serverless


### Performance testing program

alias wrk='docker run --rm williamyeh/wrk'


### Monitoring Performance

sudo crictl stats -w


Cleanup
-------

```
k3s-uninstall.sh
multipass delete vm01
multipass purge
brew cask uninstall multipass
brew uninstall kubectl
rm /tmp/k3s.yaml
```
