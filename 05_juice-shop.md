# 05 — Déploiement d’OWASP Juice Shop sur Kubernetes avec LoadBalancer et stockage persistant

## Objectif

Cette partie explique comment déployer **OWASP Juice Shop**, une application vulnérable pour l’apprentissage de la sécurité web, sur un cluster Kubernetes.
Le déploiement utilisera :

* **MetalLB** pour exposer l’application via une **IP locale publique** sur le réseau du laboratoire.
* **Longhorn** pour stocker les données de l’application de façon **persistante** via un **Persistent Volume Claim (PVC)**.

---

## 1. Pré-requis

Avant de commencer :

* [Cluster Kubernetes avec **k3s** installé](https://github.com/mael-chouteau/K3S_TP/blob/main/02_keepalived_k3s.md).
* [MetalLB configuré avec un pool d’IP disponible](https://github.com/mael-chouteau/K3S_TP/blob/main/03_metallb.md).
* [Longhorn installé et fonctionnel](https://github.com/mael-chouteau/K3S_TP/blob/main/04_longhorn.md).

---

## 2. Créer le namespace

```bash
kubectl create ns juice-shop
```

---

## 3. Créer un Persistent Volume Claim (PVC)

Créer un fichier `juice-shop-pvc.yaml` pour stocker les données persistantes de Juice Shop :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: juice-shop-pvc
  namespace: juice-shop
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: longhorn
```

Appliquer le PVC :

```bash
kubectl apply -f juice-shop-pvc.yaml
```

> Vérifier qu’un volume est créé automatiquement via Longhorn :

```bash
kubectl get pvc -n juice-shop
kubectl get pv
```

---

## 4. Déploiement de Juice Shop

Créer un fichier `juice-shop-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: juice-shop
  namespace: juice-shop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: juice-shop
  template:
    metadata:
      labels:
        app: juice-shop
    spec:
      nodeSelector:
          node-role.kubernetes.io/worker: "true"
      containers:
      - name: juice-shop
        image: bkimminich/juice-shop:latest
        ports:
        - containerPort: 3000
        volumeMounts:
        - mountPath: /app/data
          name: juice-shop-data
      volumes:
      - name: juice-shop-data
        persistentVolumeClaim:
          claimName: juice-shop-pvc
```

Appliquer le déploiement :

```bash
kubectl apply -f juice-shop-deployment.yaml
```

---

## 5. Exposer Juice Shop via LoadBalancer

Créer un service `juice-shop-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: juice-shop
  namespace: juice-shop
spec:
  selector:
    app: juice-shop
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
```

Appliquer le service :

```bash
kubectl apply -f juice-shop-service.yaml
```

> Vérifier l’IP attribuée par MetalLB :

```bash
kubectl get svc -n juice-shop
```

L’IP affichée dans `EXTERNAL-IP` sera celle à utiliser pour accéder à Juice Shop depuis le réseau local.

---

## 6. Vérification

Vérifier que l’application est fonctionnelle et que les données sont persistantes :

   * Créer un utilisateur ou un score dans Juice Shop.
   * Redémarrer un pod et vérifier que les données persistent grâce au PVC Longhorn.

---

## 7. Notes

* Les réplicas permettent de tester la **réplication et la tolérance aux pannes**.
* MetalLB assure que **l’application est accessible depuis le LAN**, même sur un cluster bare-metal.
* Longhorn garantit que les **données ne sont pas perdues** si un pod ou un nœud tombe.
