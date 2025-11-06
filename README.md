#  Projet Kubernetes — Cluster HA avec K3s, Keepalived, MetalLB, Longhorn & Prometheus

##  Objectif

Ce projet a pour but de déployer un cluster Kubernetes **hautement disponible (HA)** sur une infrastructure locale à l’aide de :

* **K3s** (distribution légère de Kubernetes)
* **Keepalived** (IP virtuelle haute disponibilité)
* **MetalLB** (LoadBalancer local)
* **Longhorn** (stockage distribué)

Chaque équipe dispose de plusieurs **Control Planes (CP)** et de **workers**, configurés avec des IP virtuelles distinctes pour assurer la redondance et la tolérance de panne.
Pour des raisons de clarté seul les fichiers de config et IPs de l'équipe 1 seront utilisés.

---

## Architecture

* **Réseau** : 10.2.0.0/16
* **IP virtuelle** :

  * `10.2.100.0` → Cluster de l’équipe 1 (CP1–CP3)
* **Keepalived** gère la bascule automatique de ces adresses IP.
* **K3s** fournit le plan de contrôle et l’orchestration des conteneurs.
* **MetalLB** attribue des IP locales aux services LoadBalancer.
* **Longhorn** gère le stockage persistant sur les nœuds workers.

---

## Étapes d’installation

Les étapes détaillées sont réparties dans plusieurs sous-guides pour plus de clarté :

**[Bonus](https://github.com/mael-chouteau/K3S_TP/blob/main/Bonus.md)**

1. **[01_base_setup.md](./01_base_setup.md)**
   Configuration de base commune à tous les nœuds :

   * Fichier `/etc/hosts`
   * Configuration réseau et swap
   * Activation du module `br_netfilter`
   * Paramètres `sysctl`
   * Installation de base (`iptables`, `curl`, etc.)

2. **[02_keepalived_k3s.md](./02_keepalived_k3s.md)**
   Mise en place de Keepalived et K3s pour chaque CP :

   * CP1 → `vip1` (10.2.100.0) — Master
   * CP2–CP3 → Backup
   * Rejoindre le cluster avec le `node-token`

3. **[03_metallb.md](./03_metallb.md)**
   Installation et configuration de MetalLB :

   * Choix de la plage IP locale
   * Déploiement de MetalLB
   * Tests avec un service Nginx
   * Exposition de Traefik

4. **[04_longhorn.md](./04_longhorn.md)**
   Déploiement du stockage Longhorn :

   * Installation des dépendances (`open-iscsi`)
   * Configuration du répertoire de stockage
   * Installation de Longhorn via Helm

5. **[05_juice-shop.md](./05_juice-shop.md)**
   Mise en place de juice-shop :

   * Création d'un service web
   * Utilisation du loadbalancer
   * Presistance des données avec longhorn

---

## Réinitialisation d’un nœud de contrôle

En cas d’erreur ou de corruption de cluster :

```bash
sudo systemctl stop k3s
sudo /usr/local/bin/k3s-killall.sh
sudo /usr/local/bin/k3s-uninstall.sh
sudo systemctl daemon-reload
sudo rm -rf /var/lib/rancher/k3s /etc/rancher/k3s /var/lib/cni /etc/cni /var/lib/kubelet
```
## Suppression d’un namespace qui reste bloqué
Exemple avec le namespace longhorn-system :
```bash
kubectl get namespace "longhorn-system" -o json   | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/"   | kubectl replace --raw /api/v1/namespaces/longhorn-system/finalize -f -
```
---

## Outils utilisés

| Composant      | Rôle                           |
| -------------- | ------------------------------ |
| **K3s**        | Distribution Kubernetes légère |
| **Keepalived** | IP virtuelle HA entre les CP   |
| **MetalLB**    | LoadBalancer local             |
| **Longhorn**   | Stockage distribué persistant  |

---

Projet pédagogique collaboratif réalisé dans le cadre du TP Kubernetes à l’**ESAIP**.
Chaque équipe dispose de son propre cluster et de son IP virtuelle.
