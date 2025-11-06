# 02 — Configuration de Keepalived et installation du Control Plane K3s

Cette partie explique comment configurer **Keepalived** pour la haute disponibilité (HA) entre plusieurs Control Planes au niveau de l'adresse IP utilisée, puis installer **K3s** sur ces nœuds.

---

## 1. Principe général

Keepalived permet de créer une **adresse IP virtuelle (VIP)** flottante entre plusieurs serveurs.
Cette IP sera utilisée par les clients et les workers pour rejoindre le cluster, quel que soit le Control Plane actif.

Exemple d’architecture :

| Nom | IP réelle  | Rôle                     | VIP        |
| --- | ---------- | ------------------------ | ---------- |
| cp1 | 10.2.1.1   | Control Plane principal  | 10.2.100.0 |
| cp2 | 10.2.0.244 | Control Plane secondaire | 10.2.100.0 |
| cp3 | 10.2.0.255 | Control Plane secondaire | 10.2.100.0 |

---

## 2. Installation de Keepalived

Sur **chaque Control Plane** :

```bash
sudo apt update
sudo apt install keepalived -y
```

---

## 3. Configuration de Keepalived

Édite le fichier de configuration :

```bash
sudo nano /etc/keepalived/keepalived.conf
```

### Exemple pour `cp1` (maître initial) :

```bash
vrrp_instance VI_1 {
    state MASTER
    interface ens18
    virtual_router_id 1
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass team1
    }
    virtual_ipaddress {
        10.2.100.0/24 dev ens18
    }
}
```

### Exemple pour `cp2` et `cp3` :

```bash
vrrp_instance VI_1 {
    state BACKUP
    interface ens18
    virtual_router_id 1
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass team1
    }
    virtual_ipaddress {
        10.2.100.0/24 dev ens18
    }
}
```

**Remarques :**

* `interface ens18` doit être remplacé par le nom réel de l’interface réseau si elle est différente(`ip a` pour le vérifier).
* `priority` détermine qui devient maître. Le plus haut gagne.
* `auth_pass` doit être identique sur tous les serveurs.

---

## 4. Activation et test de Keepalived

Activer le service :

```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
```

Vérifie son statut :

```bash
sudo systemctl status keepalived
```

Vérifier que l'ip est bien affectée sur cp1 
  ```bash
  ip a
  ```

---

## 5. Installation du premier Control Plane K3s

Sur le **nœud maître initial** (ex. `cp1`) :

```bash
curl -sfL https://get.k3s.io | sudo sh -s - server --cluster-init --tls-san=10.2.100.0 --tls-san=cp1 --write-kubeconfig-mode=644
```

Vérifie l’état :

```bash
sudo k3s kubectl get nodes
```

Récupère le token d’installation :

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Note-le, il sera utilisé pour les autres Control Planes.

---
## 5.1 Envoi du node token entre deux VM
Vous êtes sur le même réseau donc vous pouvez vous envoyer le token par netcat.

Sur la vm qui va recevoir le token par ex cp2:
```bash
nc -lp 4444
```
Sur la vm cp1 :
```bash
cat /var/lib/rancher/k3s/server/node-token | nc -N 10.2.0.244 4444
```

---
## 6. Installation des autres Control Planes

Sur `cp2` et `cp3` :

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=K10d249a6392322a15ee2efc58b7a13dddcf0a2eee40e365419bd0544acf2c5e3ae::server:3a64a156e2c240566fcb20354ded531e sh -s - server --server=https://10.2.100.0:6443 --tls-san=10.2.100.0

```

Vérifie que les nœuds rejoignent le cluster :

```bash
sudo k3s kubectl get nodes -o wide
```

---
## 7. Installation des workers

Les cp doivent envoyer le token aux workers en suivant l'étape 5.1.

Puis executer la commande suivante pour installer kubelet sur les workers :

```bash
curl -sfL https://get.k3s.io | sudo sh -s - agent --token=K10d249a6392322a15ee2efc58b7a13dddcf0a2eee40e365419bd0544acf2c5e3ae::server:3a64a156e2c240566fcb20354ded531e --server=https://10.2.100.0:6443 --kubelet-arg="provider-id=$(hostname)"
```
Les cp doivent assigner le rôle de worker aux vm qui viennent de rejoindre leur cluster avec la commande suivant (exemple avec le worker w555) :

```bash
kubectl label node w555 node-role.kubernetes.io/worker=true
```

Vérifiez votre cluster avec :

```bash
sudo k3s kubectl get nodes -o wide
```

## 8. Vérification de la haute disponibilité

Teste la continuité :

1. Stoppe K3s sur le maître actif :

   ```bash
   sudo systemctl stop k3s
   ```
2. Observe que le cluster reste accessible depuis un autre Control Plane :

   ```bash
   sudo k3s kubectl get nodes
   ```

---

## 9. Étape suivante

Continue avec la configuration du **Load Balancer MetalLB** :
[03_metallb.md](./03_metallb.md)

---
 ## Bonus
 En cas de problème avec un des cp vous pouvez reset votre installation avec les commandes suivantes :
```bash
sudo systemctl stop k3s
sudo /usr/local/bin/k3s-killall.sh 
sudo /usr/local/bin/k3s-uninstall.sh
sudo systemctl daemon-reload
sudo rm -rf /var/lib/rancher/k3s /etc/rancher/k3s /var/lib/cni /etc/cni /var/lib/kubelet
```

