---
title: Installer kubeadm
content_type: task
weight: 10
card:
  name: setup
  weight: 20
  title: Installer l'outil kubeadm
---

<!-- overview -->

<img src="https://raw.githubusercontent.com/kubernetes/kubeadm/master/logos/stacked/color/kubeadm-stacked-color.png" align="right" width="150px">Cette page présente le processus d'installation de l'outil `kubeadm`.
Pour savoir comment créer un cluster avec kubeadm une fois l'installation terminée, rendez vous sur la page [Utiliser kubeadm pour créer un Cluster](/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).



## {{% heading "prerequisites" %}}


* Une ou plusieurs machines éxecutant:
  - Ubuntu 16.04+
  - Debian 9+
  - CentOS 7
  - Red Hat Enterprise Linux (RHEL) 7
  - Fedora 25+
  - HypriotOS v1.0.1+
  - Flatcar Container Linux (testé avec 2512.3.0)
* 2 GB de RAM ou plus par machine (une valeur en deçà ne laissera pas assez de ressources pour vos applications)
* 2 CPUs ou plus
* Connectivité réseau complète entre toutes les machines du cluster (que le réseau soit privé ou publique)
* Hostname, adresse MAC et product_uuid unique pour chaque noeud. Plus d'informations [ici](#verify-mac-address).
* Les ports requis sont ouverts sur vos machines. Plus d'informations [ici](#check-required-ports)
* Swap désactivé. Il est NÉCESSAIRE que le swap soit désactivé afin que le kubelet fonctionne correctement.



<!-- steps -->

## Vérifier l'unicité que l'adresse MAC et du product_uuid pour chaque noeud {#verify-mac-address}

* Vous pouvez obtenir l'adresse MAC de l'interface réseau via les commandes `ip link` ou `ifconfig -a`
* Le product_uuid peut être obtenu via la commande `sudo cat /sys/class/dmi/id/product_uuid`

Il est très probable que des machines physiques aient des adresses uniques, cependant certains machines virtuelles 
peuvent avoir des valeurs identiques. Kubernetes utilise ces valeurs pour identifier les noeuds du cluster.
Si ces valeurs ne sont pas uniques pour chaque noeud, le processus d'installation peut [échouer](https://github.com/kubernetes/kubeadm/issues/31)

## Vérifier les adaptateurs réseau

Si vous avez plus d'un adaptateur réseau, et vos composants de Kubernetes ne sont pas accessibles via la route par défaut,
nous vous recommandons d'ajouter la ou les routes pour que les addresses IP soient routé par le bon adaptateur.

## Activer le traffic par pont (bridged) sur iptables

Assurez vous que le module `br_netfilter` est chargé. Vous pouvez le faire via la commande `lsmod | grep br_netfilter`. Pour charger explicitement le module, utilisez `sudo modprobe br_netfilter`.

Pour que iptables puisse voir le trafic par pont sur vos noeuds Linux, vous devez vous assurez que `net.bridge.bridge-nf-call-iptables` soit bien à la valeur 1 dans votre config `sysctl`, par exemple :

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

Pour plus de d'informations, consultez la page [Pré-requis plugin réseau](/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#network-plugin-requirements).

## Vérifier les ports

### Noeud(s) du plan de contrôle

| Protocole | Direction | Plage de ports | Utilisation             | Utilisé par                |
|-----------|-----------|----------------|-------------------------|----------------------------|
| TCP       | Entrée    | 6443*          | API Kubernetes          | Tous                       |
| TCP       | Entrée    | 2379-2380      | serveur etdc api client | kube-apiserver, etcd       |
| TCP       | Entrée    | 10250          | API Kubelet             | Lui même, plan de contrôle |
| TCP       | Entrée    | 10251          | kube-scheduler          | Lui même                   |
| TCP       | Entrée    | 10252          | kube-controller-manager | Lui même                   |

### Noeud(s) de travail

| Protocole | Direction | Plage de ports  | Utilisation           | Utilisé par                |
|-----------|-----------|-----------------|-----------------------|----------------------------|
| TCP       | Entrée    | 10250           | API Kubelet           | Lui même, plan de contrôle |
| TCP       | Entrée    | 30000-32767     | Services NodePort†    | All                        |

† Plage de ports par défaut pour les [Services NodePort](/docs/concepts/services-networking/service/).

Les ports marqués d'une * sont modifiables, vous devez donc vous assurer que 
chaque port personalisé que vous indiquez soit ouverts également.

Bien que les ports etcd soient inclus dans les noeuds du plan de contrôle, vous pouvez également
héberger votre propre cluster etcd à l'extérieur ou sur des ports personalisés.

Le plugin du réseau de pod que vous utilisez (voir ci-dessous) peut également nécessiter l'ouverture
de certains ports. Puisque cela dépend du plugin réseau, veuillez consulter la 
documentation pour le plugin vous concernant.

## Installer l'environnement d'éxecution des conteneurs {#installing-runtime}

Pour éxecuter des conteneurs dans les Pods, Kubernetes utilise un
{{< glossary_tooltip term_id="container-runtime" text="container runtime" >}}.

{{< tabs name="container_runtime" >}}
{{% tab name="noeuds Linux" %}}

Par défaut, Kubernetes utilise la
{{< glossary_tooltip term_id="cri" text="Container Runtime Interface">}} (CRI)
pour s'interfacer avec votre environnement d'éxecution.

Si vous ne spécifiez pas d'environnement, kubeadm va automatiquement essayer de détecter
un environnement installé en scannant une list de sockets Unix bien définie.
Le tableau suivant liste les environnements et le chemin de leur socket associé :

{{< table caption = "Container runtimes and their socket paths" >}}
| Environnement | Chemin du socket Unix             |
|---------------|-----------------------------------|
| Docker        | `/var/run/docker.sock`            |
| containerd    | `/run/containerd/containerd.sock` |
| CRI-O         | `/var/run/crio/crio.sock`         |
{{< /table >}}

<br />
Si à la fois Docker et containerd sont détectés, Docker sera utilisé en priorité.
Cela est rendu nécessaire car Docker 18.09 contient également containerd, et sont tous deux
détectables même si seul Docker est installé.
Si d'autres environnement sont également détectés, kubeadm retournera une erreur.

Le kubelet s'interface à Docker via l'implémentation CRI intégrée `dockershim`.

Pour plus d'informations, rendez vous sur la page [environnement d'éxecution des conteneurs](/docs/setup/production-environment/container-runtimes/)
{{% /tab %}}
{{% tab name="autres systèmes d'exploitation" %}}
Par défaut, kubeadm utilise {{< glossary_tooltip term_id="docker" >}} comme environnement d'éxecution.

Le kubelet s'interface à Docker via l'implémentation CRI intégrée `dockershim`.

Pour plus d'informations, rendez vous sur la page [environnement d'éxecution des conteneurs](/docs/setup/production-environment/container-runtimes/)
{{% /tab %}}
{{< /tabs >}}


## Installer kubeadm, kubelet et kubectl

Vous allez installer les logiciels suivants sur votre machine :

* `kubeadm`: la commande pour créer votre cluster.

* `kubelet`: le composant qui s'éxecute sur toutes les machines de votre cluster
    et qui est chargé de démarrer les pods et les conteneurs.

* `kubectl`: l'utilitaire en ligne de commande pour gérer votre cluster.

kubeadm **n'installera pas** ou ne gérera le `kubelet` ou `kubectl` pour vous, vous devrez donc
vous assurer que leur version soit celle du plan de contrôle que vous souhaitez installer.
Si vous ne le faite pas, il y a un risque d'incompatibilité de versions, qui peut mener à un comportement 
indéfinis et erratique du cluster. Cependant, _une_ différence de version mineure entre le kubelet et le plan de contrôle
est accepté, mais la version du kubelet ne peut jamais être supérieure à la version du serveur d'API.
Par exemple, les kubelets exécutant la version 1.7.0 seront entièrement compatibles avec la verion 1.8.0 du serveur d'API,
mais le contraire ne sera pas possible.

Pour plus d'informations sur l'installation de `kubectl`, rendez vous sur la page [installer et configurer kubectl](/docs/tasks/tools/install-kubectl/).

{{< warning >}}
Ces instructions bloquent les paquets Kubernetes des mises à jour système.
Cela est nécessaire car kubeadm et Kubernetes requirents une [attention particulière pour les mises à jour](/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).
{{</ warning >}}

Pour plus d'informations sur les comptabilité de versions, rendez vous sur :

* Kubernetes [version and version-skew policy](/docs/setup/release/version-skew-policy/)
* Kubeadm-specific [version skew policy](/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#version-skew-policy)

{{< tabs name="k8s_install" >}}
{{% tab name="Ubuntu, Debian ou HypriotOS" %}}
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
{{% /tab %}}
{{% tab name="CentOS, RHEL ou Fedora" %}}
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Passez SELinux en mode permissif (ce qui le rendra inactif)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

  **Notes :**
  
  - L'activation du mode permissif de SELinux via `setenforce 0` et `sed ...` désactivera SELinux.
    C'est nécessaire afin que les processus des conteneurs aient accès au système de fichiers de l'hôte, ce qui est requis pas les réseaux de pod par exemple.
    Vous devez activer ce mode en attendant que le support de SELinux soit amélioré dans le kubelet.

  - Vous pouvez laisser SELinux activé si vous savez comment le configurer mais cela peut nécessiter des paramétrages non supportés par kubeadm.

{{% /tab %}}
{{% tab name="Fedora CoreOS ou Flatcar Container Linux" %}}
Istaller le plugin CNI (nécessaire pour la plupar des réseaux de pod) : 

```bash
CNI_VERSION="v0.8.2"
sudo mkdir -p /opt/cni/bin
curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-amd64-${CNI_VERSION}.tgz" | sudo tar -C /opt/cni/bin -xz
```

Définissez le dossier où télécharger les fichiers de commandes

{{< note >}}
La variable DOWNLOAD_DIR doit pointer vers un dossier accessible en écriture.
Si vous utilisez Flatcar Container Linux, utilisez DOWNLOAD_DIR=/opt/bin.
{{< /note >}}

```bash
DOWNLOAD_DIR=/usr/local/bin
sudo mkdir -p $DOWNLOAD_DIR
```

Installer crictl (nécessaire pour kubeadm / Kubelet Container Runtime Interface (CRI))

```bash
CRICTL_VERSION="v1.17.0"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz
```

Installer `kubeadm`, `kubelet`, `kubectl` et ajouter `kubelet` en tant que service systemd:

```bash
RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/${RELEASE}/bin/linux/amd64/{kubeadm,kubelet,kubectl}
sudo chmod +x {kubeadm,kubelet,kubectl}

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

Activer et démarrer le `kubelet`:

```bash
systemctl enable --now kubelet
```

{{< note >}}
La distribution Flatcar Container Linux monte le dossier `/usr` en tant que système de fichiers en lecture seule.
Avant de créer votre cluster, vous devez réaliser des étapes supplémentaires pour rendre ce dossier inscriptible.
Consultez le [guide de dépannage Kubeadm](/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#usr-mounted-read-only/) afin d'autoriser l'écriture dans ce dossier.
{{< /note >}}
{{% /tab %}}
{{< /tabs >}}


Le kubelet va désormais redémarrer en boucle, en attendant que kubeadm lui donne des instructions supplémentaires.

## Configure le driver cgroup utilisé par le kubelet sur le noeud du plan de contrôle

Quand vous utilisez Docker, kubeadm détectera automatiquement le driver cgroup pour le kubelet
et le configurera directement dans le fichier `/var/lib/kubelet/config.yaml` lors de l'éxecution.

Si vous utilisez un CRI différent, vous devez passer la valeur `cgroupDriver` à la commande `kubeadm init` de cette manière :

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: <value>
```

Pour plus d'informations, rendez vous sur la pahe [utiliser kubeadm init avec un fichier de configuration](/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file).

Il faut noter que vous devez réaliser ce changement **seulement** si le driver cgroup de votre CRI
n'est pas `cgroupfs`, car c'est déjà la valeur par défaut du kubelet.

{{< note >}}
Vu que le flag `--cgroup-driver` est désormais déprécié par le kubelet, si il est présent dans `/var/lib/kubelet/kubeadm-flags.env`
ou dans `/etc/default/kubelet`(`/etc/sysconfig/kubelet` for RPMs), vous devez le retirer et le remplacer par KubeletConfiguration
(présent dans `var/lib/kubelet/config.yaml` par défaut).
{{< /note >}}

Le redémarrage du kubelet est nécessaire:

```bash
systemctl daemon-reload
systemctl restart kubelet
```

La détection automatique du driver cgroup pour les autres environnements 
d'éxecution de conteneurs est en cours d'implémentation.

## Troubleshooting

Si vous rencontrez des difficultés avec kubeadm, consultez le [guide de dépanage](/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/).

## {{% heading "whatsnext" %}}


* [Utiliser kubeadm pour créer un Cluster](/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
