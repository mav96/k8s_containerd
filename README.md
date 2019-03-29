# k8s_containerd
Install etcd, containerd and single-node k8s

**1. Install etcd:**

[Install etcd on Ubuntu](https://computingforgeeks.com/how-to-install-etcd-on-ubuntu-18-04-ubuntu-16-04/)
```
sudo apt -y install wget
export RELEASE="3.2.24"
wget https://github.com/etcd-io/etcd/releases/download/v${RELEASE}/etcd-v${RELEASE}-linux-amd64.tar.gz
tar xvf etcd-v${RELEASE}-linux-amd64.tar.gz
cd etcd-v${RELEASE}-linux-amd64
sudo mv etcd etcdctl /usr/local/bin 
```

Create Etcd configuration file and data directory.
```
sudo mkdir -p /var/lib/etcd/
sudo mkdir /etc/etcd
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
sudo chown -R etcd:etcd /var/lib/etcd/
```


Add /etc/systemd/system/etcd.service
```
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
Environment=ETCD_DATA_DIR=/var/lib/etcd
Environment=ETCD_NAME=%m
ExecStart=/usr/local/bin/etcd --advertise-client-urls=http://ETCD_IP:2379  --initial-advertise-peer-urls=http://ETCD_IP:2380 --listen-client-urls=http://127.0.0.1:2379,http://ETCD_IP:2379 --listen-peer-urls=http://ETCD_IP:2380
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

Reload systemd service and start etcd on Ubuntu 18,04 / Ubuntu 16,04
```
sudo systemctl daemon-reload
sudo systemctl enable etcd.service
sudo systemctl start etcd.service
sudo systemctl status etcd.service
ss -tunelp | grep 2379
etcdctl member list
```


**2. Install containerd**

[containerd](https://kubernetes.io/docs/setup/cri/#containerd)

```
export CONTAINERD_VERSION="1.2.4"
sudo systemctl enable containerd
sudo systemctl start containerd
```



**3. Installing k8s**

[kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

Add/etc/default/kubelet
```
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd --container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"

sudo systemctl enable kubelet
sudo systemctl restart kubelet
journalctl -f -u kubelet
```



**4. Init cluster**

Add kubeadm-config.yaml
```
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - "LOAD_BALANCER_DNS_OR_IP"
controlPlaneEndpoint: "LOAD_BALANCER_DNS_OR_IP:LOAD_BALANCER_PORT"
etcd:
    external:
        endpoints:
        - http://ETCD_DNS_OR_IP:2379
networking:
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.32.0.0/12
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```

Init single-node cluster
```
sudo kubeadm init --config=kubeadm-config.yaml
sudo kubeadm init --config kubeadm-config.yaml --cri-socket /run/containerd/containerd.sock

kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
kubectl get pod -n kube-system
```

**5. Reset cluster**
```
kubeadm reset --cri-socket=/var/run/containerd/containerd.sock
sudo systemctl stop etcd.service
sudo rm -rf  /var/lib/etcd/*
```
