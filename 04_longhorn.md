# 04 — Installation et fonctionnement de Longhorn (Stockage distribué)

## Objectif

Cette partie explique comment installer **Longhorn**, un système de stockage distribué pour Kubernetes, et détaille son architecture et son fonctionnement.
Longhorn fournit des **volumes persistants répliqués**, garantissant la **tolérance aux pannes** et la **haute disponibilité** des données dans le cluster.

---

## 1. Architecture de Longhorn

Longhorn est conçu pour être **léger** et entièrement intégré à Kubernetes. Il repose sur plusieurs composants qui travaillent ensemble pour fournir un stockage distribué fiable :

* **Longhorn Manager** :
  S’exécute sur chaque nœud du cluster. Responsable de l’orchestration, de la planification, de la détection des pannes et du maintien de l’état du cluster.

* **Longhorn Engine** :
  C’est le composant de **lecture/écriture** des volumes. Chaque volume Longhorn a son Engine dédié. Il gère les snapshots, les backups et la réplication des données sur plusieurs nœuds.

* **Longhorn UI** :
  Interface graphique pour gérer les volumes, snapshots, backups et surveiller l’état du stockage.

* **Longhorn CSI Driver (Container Storage Interface)** :
  Permet à Kubernetes d’utiliser Longhorn comme fournisseur de stockage. Il traduit les opérations Kubernetes sur les volumes (create, attach, mount…) en actions Longhorn.

* **Longhorn Instance Manager** :
  Deux types d’instance manager : pour les Engines et pour les Réplicas. Chaque nœud exécute un pod instance manager de chaque type.

* **Longhorn Replicas** :
  Copies synchronisées des données créées sur différents nœuds pour garantir la redondance et la haute disponibilité.

---

## 2. Fonctionnement de Longhorn

1. Lorsqu’une **Persistent Volume Claim (PVC)** est créée dans Kubernetes, le CSI Driver informe le **Longhorn Manager**.
2. Le Manager provisionne un **volume Longhorn** et un Engine correspondant.
3. L’Engine stocke les données et réplique les données vers les **Longhorn Replicas** sur d’autres nœuds.
4. Les snapshots sont des copies ponctuelles du volume, utilisables pour les backups, sans interruption des workloads.
5. Les backups sont incrémentaux et peuvent être stockés sur des services externes (AWS S3, NFS…).

---

## 3. Préparation des workers

Sur **chaque nœud worker** :

```bash
sudo apt install open-iscsi -y
sudo mkdir -p /var/lib/longhorn
sudo chmod 777 /var/lib/longhorn
```

---

## 4. Préparation d’un Control Plane

Sur **un nœud CP**, préparez le kubeconfig pour Helm et kubectl :

```bash
sudo sed -i 's/127.0.0.1/10.2.100.0/' /etc/rancher/k3s/k3s.yaml
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

---

## 5. Installation de Helm

```bash
sudo apt-get install curl gpg apt-transport-https -y
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm -y
```

Ajouter le dépôt Longhorn et mettre à jour :

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Créer le namespace :

```bash
kubectl create namespace longhorn-system
```

> Si le namespace `longhorn-system` est bloqué :

```bash
kubectl get namespace "longhorn-system" -o json | tr -d "\n" | \
sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/" | \
kubectl replace --raw /api/v1/namespaces/longhorn-system/finalize -f -
```

---

## 6. Déploiement de Longhorn

Créer un fichier de configuration **`longhorn.yaml`** .

Installer Longhorn via Helm :

```bash
helm install longhorn longhorn/longhorn --namespace longhorn-system -f longhorn.yaml
```

Vérifier le déploiement :

```bash
kubectl -n longhorn-system get pods -w
```

---

## 7. Accès à l’interface Longhorn

Longhorn fournit une interface web via le service `longhorn-frontend`.

Vérifier le service :

```bash
kubectl -n longhorn-system get svc longhorn-frontend
```

Si le service n’est pas de type `LoadBalancer` :

```bash
kubectl -n longhorn-system patch svc longhorn-frontend -p '{"spec":{"type":"LoadBalancer"}}'
```

Une IP du pool MetalLB sera affichée dans `EXTERNAL-IP`.
Exemple :

```
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
longhorn-frontend  LoadBalancer   10.43.210.15   10.2.100.12     80:32435/TCP
```

Accéder au dashboard :

```
http://10.2.100.12/
```

---

## 8. Vérification

* Tous les nœuds doivent apparaître comme **Ready** dans l’interface Longhorn.
* Tester la création d’un volume via l’UI ou avec un PVC Kubernetes.
* Vérifier la réplication sur plusieurs nœuds pour confirmer le fonctionnement.
