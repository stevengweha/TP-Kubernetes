

**Fichiers utiles**
- `docker-compose.yml` : orchestrateur local
- `k8s/` : manifests Kubernetes
- `routes/web.php` : routes Laravel exposées
+ 
+ **Déploiement Kubernetes — notes réelles d'exécution**
+ Ces notes reflètent la session d'installation et de debug réalisée sur un cluster multi-nœuds (Master / Worker) sous Ubuntu 24.04.
+ 
+ - Adresses utilisées :
+ 
+ 	- Master (control-plane) : 192.168.1.196
+ 	- Worker : 192.168.1.197
+ 
+ - Principales étapes et corrections appliquées :
+ 
+ 	- Désactiver le swap :
+ 
+ 	```bash
+ 	sudo swapoff -a
+ 	sudo sed -i '/ swap / s/^\(.*\\)$/#\1/g' /etc/fstab
+ 	```
+ 
+ 	- Charger les modules et activer le forwarding :
+ 
+ 	```bash
+ 	cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
+ 	overlay
+ 	br_netfilter
+ 	EOF
+ 
+ 	sudo modprobe overlay
+ 	sudo modprobe br_netfilter
+ 
+ 	cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
+ 	net.bridge.bridge-nf-call-iptables  = 1
+ 	net.bridge.bridge-nf-call-ip6tables = 1
+ 	net.ipv4.ip_forward                 = 1
+ 	EOF
+ 
+ 	sudo sysctl --system
+ 	```
+ 
+ 	- Installer et configurer le runtime (containerd) :
+ 
+ 	```bash
+ 	sudo apt-get install -y containerd
+ 	sudo mkdir -p /etc/containerd
+ 	containerd config default | sudo tee /etc/containerd/config.toml
+ 	sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
+ 	sudo systemctl restart containerd
+ 	```
+ 
+ 	- Installer kubeadm / kubelet / kubectl et dépendances (ex. `conntrack`) :
+ 
+ 	```bash
+ 	# ajouter le repo k8s, puis:
+ 	sudo apt-get update
+ 	sudo apt-get install -y conntrack kubeadm kubelet kubectl
+ 	sudo apt-mark hold kubelet kubeadm kubectl
+ 	```
+ 
+ 	- Initialiser le master (exemple avec Flannel CIDR) :
+ 
+ 	```bash
+ 	sudo kubeadm init --pod-network-cidr=10.244.0.0/16
+ 	mkdir -p $HOME/.kube
+ 	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
+ 	sudo chown $(id -u):$(id -g) $HOME/.kube/config
+ 	kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
+ 	```
+ 
+ 	- Problèmes rencontrés et corrections appliquées durant l'exécution :
+ 
+ 		- Erreur pré-flight : installer `conntrack` avant `kubeadm init`.
+ 		- Worker en status `NotReady` : déployer Flannel et redémarrer `kubelet`.
+ 		- Erreurs `exec format error` pour certains containers : images incompatibles avec l'architecture (solution : remplacer par images compatibles amd64, ex. `rmohr/activemq:5.15.9` ou `richardchesterwood/*` pour web/api/tracker/simulator).
+ 		- Problème PersistentVolumeClaim `mongo-pvc` en Pending → création d'un PV local et bind sur `/mnt/data` :
+ 
+ 	```bash
+ 	sudo mkdir -p /mnt/data
+ 	cat <<EOF > pv.yaml
+ 	apiVersion: v1
+ 	kind: PersistentVolume
+ 	metadata:
+ 		name: mongo-pv
+ 	spec:
+ 		capacity:
+ 			storage: 1Gi
+ 		accessModes:
+ 			- ReadWriteOnce
+ 		hostPath:
+ 			path: "/mnt/data"
+ 		storageClassName: ""
+ 	EOF
+ 
+ 	kubectl apply -f pv.yaml
+ 	```
+ 
+ 		- Si un PVC reste en `Terminating`, supprimer le finalizer :
+ 
+ 	```bash
+ 	kubectl patch pvc mongo-pvc -p '{"metadata":{"finalizers":null}}' --type=merge
+ 	```
+ 
+ 	- ActiveMQ : le déploiement attendait un `ConfigMap` `activemq-config`. Exemple de création (XML compact) :
+ 
+ 	```bash
+ 	kubectl create configmap activemq-config \
+ 		--from-literal=activemq.xml='<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd"><broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" dataDirectory="${activemq.data}" persistent="false"></broker></beans>'
+ 	kubectl delete pod -l app=fleetman-queue
+ 	```
+ 
+ 	- Pour corriger les erreurs `exec format error` (NGINX/Java/ActiveMQ), remplacer les images non compatibles par des images amd64 compatibles :
+ 
+ 	```bash
+ 	kubectl set image deployment/fleetman-queue fleetman-queue=rmohr/activemq:5.15.9
+ 	kubectl set image deployment/fleetman-webapp fleetman-webapp=richardchesterwood/k8s-fleetman-webapp-angular:release2
+ 	kubectl set image deployment/fleetman-api-gateway fleetman-api-gateway=richardchesterwood/k8s-fleetman-api-gateway:release2
+ 	kubectl set image deployment/fleetman-position-tracker fleetman-position-tracker=richardchesterwood/k8s-fleetman-position-tracker:release3
+ 	kubectl set image deployment/fleetman-position-simulator fleetman-position-simulator=richardchesterwood/k8s-fleetman-position-simulator:release1
+ 	```
+ 
+ 	- Taint / untaint master pour diriger les pods vers le worker :
+ 
+ 	```bash
+ 	# empêcher scheduling sur le control-plane
+ 	kubectl taint nodes steve-virtualbox node-role.kubernetes.io/control-plane:NoSchedule --overwrite
+ 
+ 	# annuler le taint (si besoin)
+ 	kubectl taint nodes --all node-role.kubernetes.io/control-plane-
+ 	```
+ 
+ 	- Forcer le redéploiement des workloads si nécessaire :
+ 
+ 	```bash
+ 	kubectl rollout restart deployment fleetman-api-gateway fleetman-mongodb fleetman-position-simulator fleetman-position-tracker fleetman-queue fleetman-webapp
+ 	```
+ 
+ **Accès & vérifications courantes**
+ - Vérifier les nœuds :
+ 
+ ```bash
+ kubectl get nodes
+ kubectl get pods -o wide
+ kubectl get svc
+ ```
+ 
+ - Suivre les logs d'un déploiement :
+ 
+ ```bash
+ kubectl logs -f deployment/fleetman-webapp
+ kubectl logs -f pod/<pod-name>
+ ```
+ 
+ - Vérifier un Pod problématique :
+ 
+ ```bash
+ kubectl describe pod <pod-name>
+ kubectl describe node <node-name>
+ ```
+ 
+ **Observations finales**
+ - Le cluster a été initialisé avec succès sur `steve-virtualbox` (192.168.1.196) et le `worker` (192.168.1.197) a rejoint le cluster après l'installation de Flannel et la configuration containerd.
+ - Les problèmes récurrents rencontrés pendant le déploiement étaient principalement liés à : images non compatibles (architecture), volumes (PV/PVC) et ConfigMap manquants. Toutes ces corrections sont listées ci‑dessous et les commandes reproduisent exactement les actions réalisées.
+ 
+ Si tu veux, j'intègre ces sections dans une version récapitulative encore plus courte pour un correcteur (1 page) ou je commente chaque manifest `k8s/` pour expliquer pourquoi chaque changement a été nécessaire.

