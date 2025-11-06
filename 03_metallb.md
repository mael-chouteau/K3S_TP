# 03 — Mise en place de MetalLB (LoadBalancer sur réseau local)

Ce guide explique étape par étape comment installer et configurer **MetalLB** sur un cluster Kubernetes K3s.
MetalLB permet d’attribuer des adresses IP locales aux services de type **LoadBalancer**, afin de les rendre accessibles depuis le réseau local du laboratoire.

---

## 1. Pourquoi utiliser MetalLB

Dans un environnement cloud (AWS, Azure, GCP), la création d’un **Service** de type `LoadBalancer` attribue automatiquement une adresse IP publique.
Sur un cluster local (bare-metal), ce comportement n’existe pas, car il n’y a pas de fournisseur de Load Balancer.

**MetalLB** comble ce manque en simulant ce comportement sur le réseau local.

En **mode L2** (couche liaison), un des nœuds Kubernetes “annonce” l’adresse IP du service via **ARP**, comme si cette machine possédait réellement cette adresse.
C’est simple, efficace et idéal pour un TP.

---

## 2. Étape 0 — Choisir une plage d’IP disponible

Avant d’installer MetalLB, il faut choisir une **plage d’adresses IP libres** sur le réseau local.

1. Identifier le réseau local (exemple : `10.2.0.0/16`, passerelle `10.2.0.1`)
2. Identifier la plage DHCP utilisée par le routeur (exemple : `10.2.0.30–10.2.50.255`)
3. Choisir une plage **hors DHCP**, par exemple :

   ```
   10.2.100.10 – 10.2.100.20
   ```

Cette plage sera réservée à MetalLB pour attribuer des adresses aux services `LoadBalancer`.

---

## 3. Étape 1 — Installation de MetalLB

Sur un des **Control Planes** du cluster :

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Vérifie le déploiement :

```bash
kubectl get pods -n metallb-system
```

Cette commande installe :

* le **contrôleur MetalLB**, qui gère les IP attribuées,
* le **speaker**, qui annonce les IP sur le réseau via ARP.

---

## 4. Étape 2 — Déclarer le pool d’IP et l’annonce L2

Crée un fichier nommé `metallb-pool.yaml` :

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.2.100.10-10.2.100.20

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - lab-pool
```

Applique la configuration :

```bash
kubectl apply -f metallb-pool.yaml
```

Vérifie la création :

```bash
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```

**Documentation utile :**
[https://docs.mirantis.com/mke/3.7/ops/deploy-apps-k8s/deploy-metallb/modify-ip-address-pools.html](https://docs.mirantis.com/mke/3.7/ops/deploy-apps-k8s/deploy-metallb/modify-ip-address-pools.html)

---

## 5. Étape 3 — Exposer Traefik via MetalLB

Traefik est déjà déployé par défaut avec K3s, mais souvent en `ClusterIP`.
Pour le rendre accessible depuis le LAN :

### Vérifie le service Traefik :

```bash
kubectl -n kube-system get svc traefik
```

### Modifie son type :

```bash
kubectl -n kube-system patch svc traefik -p '{"spec":{"type":"LoadBalancer"}}'
```

### Vérifie qu’une IP lui est attribuée :

```bash
kubectl -n kube-system get svc traefik -w
```

Traefik recevra une IP du pool MetalLB, permettant d’accéder aux **Ingress** depuis le réseau local.

---

## 6. Étape suivante

Passe à l’installation du **système de stockage distribué Longhorn** :
[04_longhorn.md](./04_longhorn.md)
