# Aufbau

* VM mit Debian 11 und Kubernetes via K3S

# System-Setup

* Debian 11 Standard
* Docker + Docker-compose

```/etc/docker/daemon.json
{
  "exec-opts": [
    "native.cgroupdriver=cgroupfs"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m"
  },
  "storage-driver": "overlay2"
}
```

* systemctl restart docker

## K3S

Ich habe mich entschieden, K3S selbst als Docker-Container auszuführen und so zu "verbiegen", dass es den Docker-Daemon auf dem Host für alle PODs verwendet. Damit lassen sich sowohl k3s als auch die PODs später per Docker-CLI recht einfach ansprechen.

* mkdir -p /var/lib/kubelet
* mkdir -p /var/k3s
* echo "K3S_VERSION=v1.23.3-k3s1" /data/k3s/.env

```/data/k3s/docker-compose.yaml
version: '3.3'
services:

  server:
    image: "rancher/k3s:${K3S_VERSION:?err}"
    network_mode: host
    pid: host
    command:
    - server
    - --snapshotter=native
    - --https-listen-port=6443
    - --disable=traefik
    - --tls-san=gw1.wyraz.de
    - --default-local-storage-path=/data/k3s/volumes
    - --docker
    tmpfs:
    - /run
    ulimits:
      nproc: 65535
      nofile:
        soft: 65535
        hard: 65535
    privileged: true
    restart: unless-stopped
    environment:
    - K3S_KUBECONFIG_OUTPUT=/output/kubeconfig.yaml
    - K3S_KUBECONFIG_MODE=600
    volumes:
    - ./k3s-server:/var/lib/rancher/k3s
    - /data:/data
    - /var/run/docker.sock:/var/run/docker.sock
    - type: bind
      source: /var/lib/docker
      target: /var/lib/docker
      bind:
        propagation: rshared
    - type: bind
      source: /var/lib/kubelet
      target: /var/lib/kubelet
      bind:
        propagation: rshared   
    - /sys:/sys
    # This is just so that we get the kubeconfig file out
    - .:/output
```

* docker-compose up -d

* K3S startet. Anschließend liegt ein Kubeconfig-File unter `/data/k3s/kubeconfig.yaml`, mit dem der "Cluster" administriert werden kann.