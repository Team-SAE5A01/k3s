# Projet Kubernetes K3s

Déployer une petite stack sur un cluster **K3s** (nœud unique ou multi‑nœuds) : MySQL + phpMyAdmin + deux services applicatifs (`datamedia` et `wheeltrip-user`).

---

## Contenu

```
.
├── datamedia-configmap.yaml
├── datamedia-deployment.yaml
├── datamedia-service.yaml
├── mysql-deployment.yaml
├── mysql-pvc.yaml
├── mysql-secret.yaml
├── mysql-service.yaml
├── phpmyadmin-deployment.yaml
├── phpmyadmin-service.yaml
├── README.md
├── wheeltrip-user-configmap.yaml
├── wheeltrip-user-deployment.yaml
└── wheeltrip-user-service.yaml
```

---

## Prérequis

- Un cluster Kubernetes fonctionnel. Testé avec **K3s** ≥ 1.29 sur Linux.
- `kubectl` configuré pour accéder à votre cluster (ex. `~/.kube/config`).
- (Optionnel) Un namespace dédié (par défaut dans ce README : `wheeltrip`).

> **Installer K3s (nœud unique)**
>
> ```bash
> curl -sfL https://get.k3s.io | sh -
> # Le kubeconfig est disponible à /etc/rancher/k3s/k3s.yaml
> sudo kubectl get nodes
> ```
>
> **Utiliser kubectl sans sudo**
>
> ```bash
> mkdir -p ~/.kube
> sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
> sudo chown $(id -u):$(id -g) ~/.kube/config
> kubectl cluster-info
> ```

---

## Démarrage rapide (TL;DR)

```bash
# 1) Créer un namespace (optionnel mais recommandé)
kubectl create namespace wheeltrip

# 2) Appliquer les manifestes dans un ordre sûr
kubectl -n wheeltrip apply -f mysql-pvc.yaml
kubectl -n wheeltrip apply -f mysql-secret.yaml
kubectl -n wheeltrip apply -f mysql-deployment.yaml
kubectl -n wheeltrip apply -f mysql-service.yaml

kubectl -n wheeltrip apply -f datamedia-configmap.yaml
kubectl -n wheeltrip apply -f datamedia-deployment.yaml
kubectl -n wheeltrip apply -f datamedia-service.yaml

kubectl -n wheeltrip apply -f wheeltrip-user-configmap.yaml
kubectl -n wheeltrip apply -f wheeltrip-user-deployment.yaml
kubectl -n wheeltrip apply -f wheeltrip-user-service.yaml

kubectl -n wheeltrip apply -f phpmyadmin-deployment.yaml
kubectl -n wheeltrip apply -f phpmyadmin-service.yaml

# 3) Attendre que les pods soient prêts
kubectl -n wheeltrip get pods -w

# 4) Rediriger les services vers localhost (si Services en ClusterIP)
# phpMyAdmin (service phpmyadmin-service sur le port 80):
kubectl -n wheeltrip port-forward svc/phpmyadmin-service 8080:80
# wheeltrip-user (par défaut sur 3000):
kubectl -n wheeltrip port-forward svc/wheeltrip-user-service 3000:3000
# datamedia (remplacer par le port défini dans le service):
kubectl -n wheeltrip port-forward svc/datamedia-service 8081:80
```

Accès via navigateur :
- phpMyAdmin → http://localhost:8080
- wheeltrip-user → http://localhost:3000
- datamedia → http://localhost:8081 (ou le port choisi)

---

## Configuration

### 1) Secrets (MySQL)
Vérifiez que `mysql-secret.yaml` contient les valeurs encodées en **base64** pour vos identifiants. Exemple :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wheeltrip
type: Opaque
data:
  mysql-user: d2hlZWx0cmlw  # echo -n "wheeltrip" | base64
  mysql-password: c3VwZXJfc2VjcmV0  # echo -n "super_secret" | base64
  mysql-root-password: cm9vdF9zdXBlcl9zZWNyZXQ=  # echo -n "root_super_secret" | base64
```

> Encoder avec :
> ```bash
> echo -n "valeur" | base64
> ```

Assurez-vous que `mysql-deployment.yaml` et les autres apps référencent bien ces clés comme variables d’environnement.

### 2) ConfigMaps
- `datamedia-configmap.yaml`
- `wheeltrip-user-configmap.yaml`

À remplir avec les variables nécessaires (URL d’API, hôte DB, etc.). Vérifiez que vos déploiements consomment ces ConfigMaps.

### 3) Stockage persistant
- `mysql-pvc.yaml` définit la persistance de MySQL. Ajustez `storageClassName`, `accessModes` et `requests.storage` selon votre cluster.

---

## Déploiement détaillé

1. **Namespace**
   ```bash
   kubectl create namespace wheeltrip
   ```

2. **Stockage (PVC MySQL)**
   ```bash
   kubectl -n wheeltrip apply -f mysql-pvc.yaml
   kubectl -n wheeltrip get pvc
   ```

3. **Secrets**
   ```bash
   kubectl -n wheeltrip apply -f mysql-secret.yaml
   kubectl -n wheeltrip get secrets
   ```

4. **MySQL**
   ```bash
   kubectl -n wheeltrip apply -f mysql-deployment.yaml
   kubectl -n wheeltrip apply -f mysql-service.yaml
   kubectl -n wheeltrip rollout status deployment/mysql-deployment
   kubectl -n wheeltrip get svc mysql-service
   ```

5. **Applications (ConfigMaps → Deployments → Services)**
   ```bash
   # datamedia
   kubectl -n wheeltrip apply -f datamedia-configmap.yaml
   kubectl -n wheeltrip apply -f datamedia-deployment.yaml
   kubectl -n wheeltrip apply -f datamedia-service.yaml

   # wheeltrip-user
   kubectl -n wheeltrip apply -f wheeltrip-user-configmap.yaml
   kubectl -n wheeltrip apply -f wheeltrip-user-deployment.yaml
   kubectl -n wheeltrip apply -f wheeltrip-user-service.yaml

   # phpMyAdmin
   kubectl -n wheeltrip apply -f phpmyadmin-deployment.yaml
   kubectl -n wheeltrip apply -f phpmyadmin-service.yaml
   ```

6. **Vérification**
   ```bash
   kubectl -n wheeltrip get all
   kubectl -n wheeltrip get events --sort-by=.lastTimestamp | tail -n 50
   ```

---

## Accès aux services

Par défaut, les Services sont de type `ClusterIP` (interne seulement). Plusieurs options :

### A) Port‑forward (développement local)
```bash
kubectl -n wheeltrip port-forward svc/phpmyadmin-service 8080:80
kubectl -n wheeltrip port-forward svc/wheeltrip-user-service 3000:3000
kubectl -n wheeltrip port-forward svc/datamedia-service 8081:80
```

### B) NodePort (exposé sur l’IP du nœud)
```bash
kubectl -n wheeltrip patch svc/wheeltrip-user-service \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 3000, "targetPort": 3000, "nodePort": 32000}]}}'
```
Accessible via `http://<ip-noeud>:32000`.


## Debug & Observabilité

```bash
kubectl -n wheeltrip get pods -o wide
kubectl -n wheeltrip describe pod <pod>
kubectl -n wheeltrip logs -f deployment/<nom-deployment>
kubectl -n wheeltrip get svc,endpoints
kubectl -n wheeltrip get pvc,pv
```

**Problèmes fréquents**
- `ImagePullBackOff` → image incorrecte.
- `CrashLoopBackOff` → vérifier logs/env vars/connexion DB.
- `Pending` sur MySQL → souci de PVC.
- phpMyAdmin n’accède pas à MySQL → vérifier `PMA_HOST` et readiness de MySQL.

---

## Mise à jour & redeploiement

```bash
kubectl -n wheeltrip apply -f <fichier.yaml>
# ou
kubectl -n wheeltrip apply -f .
```

Suivre le déploiement :
```bash
kubectl -n wheeltrip rollout status deployment/<nom>
```

---

## Nettoyage

```bash
kubectl -n wheeltrip delete -f .
kubectl delete namespace wheeltrip
```

---

## Pour aller plus loin (prod)

- Externaliser les secrets (SOPS, KMS, CSI driver).
- Ajouter ressources (CPU/mémoire) et probes (liveness/readiness).
- Backups MySQL (Velero, CronJob `mysqldump`).
- Ingress + TLS (cert‑manager).
- Autoscaling (HPA).
- Monitoring (Prometheus/Grafana) + logs centralisés.

---

## Aide base64

```bash
echo -n "valeur" | base64
echo -n "dmFsZXVy" | base64 -d
```

---

## One‑liners utiles

```bash
# Pods non prêts
kubectl -n wheeltrip get pods | awk 'NR==1 || $3!="Running" || $2!=$4'

# Suivre les events
kubectl -n wheeltrip get events --watch

# Pods par label
kubectl -n wheeltrip get pods -l app=wheeltrip-user
```

