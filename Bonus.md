# Exécuter des commandes `kubectl` en tant que Worker

Vous pouvez déployer et gérer des services directement depuis un nœud Worker.

1. **Récupérer le fichier de configuration Kubeconfig**
   Sur un des nœuds de contrôle (CP), récupérez le fichier :

   ```
   /etc/rancher/k3s/k3s.yaml
   ```

   Pour voir comment transférer le contenu d’un fichier entre deux VM, référez-vous à la partie [5.1 du TP02](https://github.com/mael-chouteau/K3S_TP/blob/main/02_keepalived_k3s.md#51-envoi-du-node-token-entre-deux-vm).

2. **Créer le dossier et copier le fichier**
   Sur votre worker, créez le dossier `~/.kube` et copiez le fichier `k3s.yaml` dedans :

   ```bash
   mkdir -p ~/.kube
   cp <chemin_du_fichier_transféré> ~/.kube/config
   chown $(id -u):$(id -g) ~/.kube/config
   ```

3. **Modifier l’adresse IP**
   Dans le fichier `k3s.yaml`, remplacez l’IP `127.0.0.1` par **l’adresse IP virtuelle** de votre cluster.

4. **Tester la connexion**

   ```bash
   kubectl get nodes
   ```

   Si tout est correct, vous devriez voir la liste des nœuds de votre cluster.

---

# Utilisation de OpenLens

Vous pouvez piloter et monitorer votre cluster à partir de [OpenLens](https://github.com/MuhammedKalkan/OpenLens/releases/tag/v6.5.2-366).

* Pour Windows, prenez **la version MSI** (la version EXE ne semble pas fonctionner).
* Versions disponibles :

  * Windows MSI : [OpenLens-6.5.2-366.msi](https://github.com/MuhammedKalkan/OpenLens/releases/download/v6.5.2-366/OpenLens.6.5.2-366.msi)
  * Linux DEB : [OpenLens-6.5.2-366.amd64.deb](https://github.com/MuhammedKalkan/OpenLens/releases/download/v6.5.2-366/OpenLens-6.5.2-366.amd64.deb)
  * Linux RPM : [OpenLens-6.5.2-366.x86_64.rpm](https://github.com/MuhammedKalkan/OpenLens/releases/download/v6.5.2-366/OpenLens-6.5.2-366.x86_64.rpm)
  * Linux AppImage : [OpenLens-6.5.2-366.x86_64.AppImage](https://github.com/MuhammedKalkan/OpenLens/releases/download/v6.5.2-366/OpenLens-6.5.2-366.x86_64.AppImage)
  * MacOS DMG : [OpenLens-6.5.2-366.dmg](https://github.com/MuhammedKalkan/OpenLens/releases/download/v6.5.2-366/OpenLens-6.5.2-366.dmg)
  * MacOS ARM64 DMG : [OpenLens-6.5.2-366-arm64.dmg](https://github.com/MuhammedKalkan/OpenLens/releases/download/v6.5.2-366/OpenLens-6.5.2-366-arm64.dmg)

## Ajouter votre cluster dans OpenLens

1. Ouvrez OpenLens.
2. Allez dans **File → Add Cluster**.
3. Copiez le contenu du fichier `k3s.yaml` récupéré depuis un CP.
4. Modifiez l’IP `127.0.0.1` par **l’adresse IP virtuelle** de votre cluster.
5. Enregistrez et vérifiez que le cluster apparaît correctement.
