# GreenIT

**Description**
- **GreenIT** est une application de démonstration qui orchestre plusieurs microservices (webapp, api-gateway, position-tracker, position-simulator, queue, mongodb). Le dépôt contient la configuration Docker Compose pour un démarrage local et des manifests Kubernetes dans le dossier `k8s/` pour un déploiement en cluster.

**Architecture**
- **fleetman-webapp**: interface web (port 30080)
- **fleetman-api-gateway**: passerelle API (port 30020)
- **fleetman-position-tracker**: service de tracking (port 30010)
- **fleetman-position-simulator**: simulateur de positions
- **fleetman-queue**: broker (ActiveMQ) exposant 61616/8161
- **fleetman-mongodb**: base MongoDB (port 27017)

**Prérequis**
- Docker et Docker Compose installés
- (Pour Kubernetes) `kubectl` et un cluster (minikube, kind, ou un cluster distant)
- Recuperer le code source sur github 

**Démarrage local (Docker Compose)**
1. Ouvrez un terminal et placez-vous dans la racine du projet:

```
cd "c:\\Users\\Steve\\Downloads\\GreenIT-main 1\\GreenIT-main"
```

2. Démarrer tous les services en arrière-plan:

```
docker-compose up -d
```

3. Vérifier les logs (ex. webapp):

```
docker-compose logs -f fleetman-webapp
```

4. Accéder aux services depuis l'hôte:
- Webapp: http://localhost:30080
- API Gateway: http://localhost:30020
- Position Tracker: http://localhost:30010

Ports exposés (depuis `docker-compose.yml`): 30080 (web), 30020 (api), 30010 (tracker), 61616/8161 (queue), 27017 (mongodb).

**Déploiement sur Kubernetes**
1. Vérifier que votre cluster est prêt et que `kubectl` cible le bon contexte.
2. Appliquer tous les manifests:

```
kubectl apply -f k8s/
```

3. Vérifier les ressources:

```
kubectl get pods,svc -n default
```

Remarque: le manifest `fleetman-mongodb-deployment.yaml` crée un `PersistentVolumeClaim` nommé `mongo-pvc`. Assurez-vous que votre cluster a un provisioner de volumes dynamiques ou créez manuellement un PV adapté.

**Routes et logs**
- Le projet Laravel fourni contient une route POST `/log/scene-view` (voir [routes/web.php](routes/web.php#L1)). Les logs pour la barre latérale sont envoyés sur le channel `sidebar`.

**Dépannage rapide**
- Si un port est déjà utilisé, arrêtez le service local occupant le port ou modifiez `docker-compose.yml`.
- Si Mongo n'arrive pas à démarrer sur Kubernetes, vérifiez le `PersistentVolume` et les permissions.

**Contribuer**
- Créez une branche feature, faites vos modifications puis ouvrez une pull request.

**Fichiers utiles**
- `docker-compose.yml` : orchestrateur local
- `k8s/` : manifests Kubernetes
- `routes/web.php` : routes Laravel exposées

