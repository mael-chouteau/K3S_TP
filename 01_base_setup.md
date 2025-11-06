# 01 — Setup de Base pour tous les nœuds

Ce guide décrit la configuration initiale commune à **tous les Control Planes (CP)** et **workers** du cluster Kubernetes.
Ces étapes préparent le système à l’installation de **K3s** et des composants HA (Keepalived, MetalLB, etc.).
les hostnames doivent être en minuscule dans ce style : (cp1, w35, W923, cp4)

---
**Changez et notez votre MDP de VM.**
```bash
passwd
```

## 1. Configuration du fichier hosts

Ajoute la correspondance des noms de machines :

```bash
sudo tee -a /etc/hosts > /dev/null <<'EOF'
10.2.1.1   cp1
10.2.0.244 cp2
10.2.0.255 cp3
10.2.100.0 vip1
EOF
```

---

## 2. Définir le nom d’hôte

Définis le **hostname** correspondant à la machine :

```bash
sudo nano /etc/hostname
sudo hostnamectl set-hostname <nom_du_noeud>
```

Puis vérifie sa cohérence dans `/etc/hosts` :

```bash
sudo nano /etc/hosts
```

---

## 3. Mise à jour du système

Assure-toi que ton système est à jour :

```bash
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

Installe les outils nécessaires :

```bash
sudo apt install iptables curl -y
```

---

## 4. Désactivation du swap

```bash
sudo nano /etc/fstab
```

Commente la ligne contenant la partition swap (ajoute `#` au début).

---

##  5. Activation du module réseau `br_netfilter`

```bash
echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf
```

---

##  6. Configuration des paramètres sysctl

Active le routage et le filtrage réseau pour Kubernetes :

```bash
sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
vm.swappiness=0
EOF
```

Applique la configuration :

```bash
sudo sysctl --system
```

---
##  7. Redémarrage

Redémarre la machine pour appliquer tous les changements :

```bash
sudo reboot
```
---

## Étape suivante

Passe à la configuration des **Control Planes et Keepalived** :
[02_keepalived_k3s.md](./02_keepalived_k3s.md)
