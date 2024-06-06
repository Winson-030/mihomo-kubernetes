# mihomo-kubernetes

Deploy mihomo(clash-meta) as transparent proxy gateway in Kubernetes

## Notice

Recently, I found that Mihomo has some problems after being deployed for a while. It appears to be a memory leak issue that has been raised on [Github](https://github.com/MetaCubeX/mihomo/issues/405).
Some Mihomo users have reported the memory leak problem. To address this issue, I have created a cronjob to redeploy Mihomo every day. Mihomo functions well after being redeployed.

The `cronjob-restart.yaml` is the configuration for this cronjob.

easy to use `kubectl apply -f cronjob-restart.yaml`, don't forget to set `<YOUR NAMESPACE>`.

## Environments

My cluster is a 3 nodes K3s cluster, upstream router is Openwrt. Mihomo will be deployed in the master node, and will be used as a gateway for transparent proxying.

## Install multus-cni

Deploy multus-cni to kubernetes cluster using Helm. Multus-cni is a CNI plugin that allows you to attach multiple network interfaces to a pod. This allows you to expose interfaces to your local network, this is useful for setting as a gateway for transparent proxying.

```shell
helm repo add rke2-charts https://rke2-charts.rancher.io/
# Recommend to specify values.yaml
helm install --namespace kube-system --name multus-cni rke2-charts/rke2-multus --version 4.0.204

```

## Create a network attachment definition for mihomo

This is the configuration for the pod to attach to the interface which expose the local network(not cluster internal network).

```shell
# Namespace for mihomo
kubectl create namespace mihomo
```

```yaml
# mihomo-home-lan.yaml
# Interface enp1s0 is the Lan interface of master node and it is Debian11 operating system. Specify the interface name on your node operating system.
# 'address' is a static IP address of the interface, and 'gateway' is the gateway IP address of master node
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: mihomo-home-lan
  namespace: mihomo
spec:
  config: '{ "cniVersion": "1.0.0", "type": "macvlan", "name": "mihomo-home-lan", "master": "enp1s0", "mode": "bridge", "ipam": { "type": "static", "addresses": [ { "address": "10.0.0.245/24", "gateway": "10.0.0.254" } ], "routes": [ { "dst": "0.0.0.0/0" } ], "dns": { "nameservers": [ "10.0.0.254"], "domain": "example.local", "search": [ "example.local" ] } } }'

```

```shell
kubectl create -f mihomo-home-lan.yaml
```

## Prepare configuration file and script for setting up mihomo and transparent proxy

```yaml
# mihomo-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mihomo-config
  namespace: mihomo
data:

  # file-like keys
  config.yaml: |
    xxxx
---
# mihomo-iptables.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mihomo-iptables
  namespace: mihomo
data:

  # file-like keys
  iptables.sh: >
    #!/bin/sh
    set -ex

    # ENABLE ipv4 forward
    sysctl -w net.ipv4.ip_forward=1

    # ROUTE RULES
    ip rule add fwmark 666 lookup 666
    ip route add local 0.0.0.0/0 dev lo table 666

    # clash Chain
    iptables -t mangle -N clash


    iptables -t mangle -A clash -d 0.0.0.0/8 -j RETURN
    iptables -t mangle -A clash -d 127.0.0.0/8 -j RETURN
    iptables -t mangle -A clash -d 10.0.0.0/8 -j RETURN
    iptables -t mangle -A clash -d 172.16.0.0/12 -j RETURN
    iptables -t mangle -A clash -d 192.168.0.0/16 -j RETURN
    iptables -t mangle -A clash -d 169.254.0.0/16 -j RETURN

    iptables -t mangle -A clash -d 224.0.0.0/4 -j RETURN
    iptables -t mangle -A clash -d 240.0.0.0/4 -j RETURN

    iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 666
    iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 666

    iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to 1053
  
    iptables -t nat -I PREROUTING -p udp --dport 53 -d 10.0.0.0/24 -j REDIRECT --to 1053


    iptables -t mangle -A PREROUTING -j clash

    iptables -t mangle -N clash_local

 
    iptables -t mangle -A clash_local -d 0.0.0.0/8 -j RETURN
    iptables -t mangle -A clash_local -d 127.0.0.0/8 -j RETURN
    iptables -t mangle -A clash_local -d 10.0.0.0/8 -j RETURN
    iptables -t mangle -A clash_local -d 172.16.0.0/12 -j RETURN
    iptables -t mangle -A clash_local -d 192.168.0.0/16 -j RETURN
    iptables -t mangle -A clash_local -d 169.254.0.0/16 -j RETURN

    iptables -t mangle -A clash_local -d 224.0.0.0/4 -j RETURN
    iptables -t mangle -A clash_local -d 240.0.0.0/4 -j RETURN

 
    iptables -t mangle -A clash_local -p tcp -j MARK --set-mark 666
    iptables -t mangle -A clash_local -p udp -j MARK --set-mark 666

    # uid owner 0
    iptables -t mangle -A OUTPUT -p tcp -m owner --uid-owner 0 -j RETURN
    iptables -t mangle -A OUTPUT -p udp -m owner --uid-owner 0 -j RETURN
    iptables -t mangle -A OUTPUT -j clash_local
    sysctl -w net.ipv4.conf.all.route_localnet=1
    iptables -t nat -A PREROUTING -p icmp -d 198.18.0.0/16 -j DNAT --to-destination 127.0.0.1
    
```

```shell

kubectl create -f mihomo-config.yaml

kubectl create -f mihomo-iptables.yaml

```

## Prepare Deployment for Mihomo

```yaml
# mihomo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mihomo-deployment
  namespace: mihomo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mihomo
  template:
    metadata:
      labels:
        app: mihomo
      # Use annotation to specify the network interface
      annotations:
        k8s.v1.cni.cncf.io/networks: 'mihomo-home-lan'
    spec:
      # User nodeselector if needed
      nodeName: master
      containers:
      - name: mihomo
        image: metacubex/mihomo:Alpha
        imagePullPolicy: IfNotPresent
        # This will execute the script to configure iptables after mihomo starts up
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "/root/iptables.sh"]
        securityContext:
          privileged: true
          capabilities:
            add: ["NET_ADMIN"]
        resources:
          limits:
            cpu: "1000m"
            memory: 512Mi
          requests:
            cpu: "250m"
            memory: 256Mi
        env:
        - name: TZ
          value: Asia/Shanghai
        ports:
        - containerPort: 53
          name: dns
        - containerPort: 7890
          name: mixed-port
        - containerPort: 9999
          name: api
        - containerPort: 7893
          name: tproxy-port
        volumeMounts:
        - name: mihomo-config
          mountPath: /root/.config/mihomo/config.yaml
          subPath: config.yaml
        - name: mihomo-volume
          mountPath: /root/.config/mihomo
        - name: iptables
          mountPath: /root/iptables.sh
          subPath: iptables.sh
      volumes:
      - name: mihomo-volume
        hostPath:
          path: /root/mihomo
      - name: mihomo-config
        configMap:
          name: mihomo-config
      - name: iptables
        configMap:
          name: mihomo-iptables
          # This will change the iptables.sh file mode to 755, so that it can be executed by root user
          defaultMode: 0744
```

```shell
kubectl apply -f mihomo-deployment.yaml
```

Interface net1@eth0 will be exposed to the local network, with static IP 10.0.0.245 configured.

```shell
# Interfaces of mihomo pod
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if67: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1230 qdisc noqueue state UP 
    link/ether f6:b8:43:13:f9:ad brd ff:ff:ff:ff:ff:ff
    inet 10.42.3.75/24 brd 10.42.3.255 scope global eth0
       valid_lft forever preferred_lft forever
3: net1@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether ce:8b:0e:51:4d:0c brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.245/24 brd 10.0.0.255 scope global net1

```

Finally, set the IP as gateway and dns on the dhcp server, all devices will route to mihomo.
