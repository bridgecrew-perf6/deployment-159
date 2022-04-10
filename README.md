# Aufbau

* VM mit Debian 11 und Kubernetes via K3S
* Das Setup erfolgt über Ansible (Hosts und Playbook im Ordner "ansible")

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

Die `kubeconfig.yaml` kann auf den eigenen Rechner kopiert werden. Darin sollte der Hostname und der Name des "Context" angepasst werden. Ich lege die Datei unter ~/.kube/config.ffle ab, dort wird sie automatisch von dem von mir verwendeten Deployment-Tool `kuploy` gefunden.

Test, dass der Zugriff auf den Cluster möglich ist: `KUBECONFIG=~/.kube/config.ffle kubectl cluster-info`

Auch das Tool "Helm-Lens" findet die Datei, so dass ich schonmal in den Cluster "reinschauen" kann.

# Deployments

Die Deployments im K3S erfolgen mittels Helm-Charts. Die Infos zu den Charts, Werte und Parameter sind in den .yaml Dateien im Ordner "deploy" enthalten. Ich verwende das Tool `kuploy`, um die passenden Charts mit den passenden Werten zu deployen.

Beispiel:

`kuploy infrastructure.yaml`

Installiert den Infrastruktur-Stack.

Das Tool `kuploy` kümmerst sich auch um die Verwaltung von geheimen Schlüsselmaterial und Passworten. Dazu erzeugt es einen AES-Key und speichert diesen als Secret im Kubernetes. Sämtliche Secrets werden mit diesem geheimen Key verschlüsselt und können in dieser Form im Git abgelegt werden.


## Infrastructur-Stack

### VM-Stack (optional)

Installiert den Victoriametrics-Stack, welcher das Monitoring des Clusters ermöglicht.

### Cert-Manager

Installiert den Cert-Manager, welcher für die Vergabe und Erneuerung von SSL-Zertifikaten über Let's Encrypt verantwortlich ist. Es wird ein Wrapper-Chart verwendet, welches erst den Cert-Manager (incl. CRDs) und anschließend einen HTTP-Issuer für Let's Encrypt installiert.

### Nginx Ingress Controller

Ingress-Controller zur Bereitstellung von Diensten via HTTP(s).

### Backup

Restic-Backup aller Daten auf einen B2 (Backblaze) Account.

## Mapserver-Stack

### Mapserver

All-in-one chart für die Bestandteile des Mapservers:

#### Fastd/yanic

* Fastd verbindet sich mit ein oder mehreren Supernodes
* Yanic lauscht auf Broadcasts von Nodes und erzeugt daraus die Node-Statistiken
* Meshviewer-Collector sammelt Mapviewer-kompatible Metriken (im Falle von FFLE vom alten Meshkit-Netz) ein und sendet diese an Yanic
* Victoriametrics stellt eine Influx-DB kompatible Zeitreihen-Datenbank zur Verfügung, in die Yanic die Statistiken schreibt
* Meshviewer gibt die Karte als Webseite aus
* Mittels prometheus-png-renderer werden Vorschaugrafiken von Statistiken für die Karte erzeugt. Dies ist wesentlich schneller, als der Grafana-Renderer

