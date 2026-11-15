# Journal de bord  TP Kubernetes



---

## Exercice 0  Préparer l'environnement

### Installation des outils

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl

curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

curl -L https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz | tar xz k9s
sudo install k9s /usr/local/bin/ && rm k9s

STERN_VER=$(curl -s https://api.github.com/repos/stern/stern/releases/latest | grep -Po '"tag_name": "v\K[^"]*')
curl -L "https://github.com/stern/stern/releases/download/v${STERN_VER}/stern_${STERN_VER}_linux_amd64.tar.gz" | tar xz stern
sudo install stern /usr/local/bin/ && rm stern

sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -sf /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -sf /opt/kubectx/kubens /usr/local/bin/kubens
```

### Vérification

```bash
minikube version
kubectl version --client
docker --version
helm version
k9s version
stern --version
kubectx --help
kubens --help
```


### Erreur : Docker introuvable

```
The command 'docker' could not be found in this WSL 2 distro.
We recommend to activate the WSL integration in Docker Desktop settings.
```

Docker fonctionnait pourtant au TP précédent dans cette même distro.

Résolution : lancer Docker Desktop, puis Settings → Resources → WSL Integration, activer Ubuntu, Apply & Restart, et rouvrir le terminal.

```bash
docker --version
docker run --rm hello-world
```

Le `hello-world` est le test décisif : `docker --version` interroge le client, qui peut répondre alors que le démon est injoignable.

### Démarrage du cluster

```bash
minikube start --driver=docker
kubectl get nodes
kubectl cluster-info
```

Le premier lancement télécharge l'image de base (~508 MB) et les images Kubernetes préchargées (~348 MB).

Le nœud apparaît en `NotReady` les premières secondes, le temps que le CNI démarre, puis passe en `Ready`.



---

## Exercice 1  Explorer le cluster

### Commandes d'observation

```bash
kubectl get all
kubectl get nodes
kubectl get namespaces
kubectl config current-context
kubectl cluster-info
kubectl describe node minikube
kubectl get pods --all-namespaces
kubectl get events --sort-by='.metadata.creationTimestamp' -A
kubectl explain pods
```



### Création des namespaces

```bash
kubectl create namespace guestbook
kubectl create namespace data
kubens guestbook
kubens -c
kubectl get pods
```

Résultat : `No resources found in guestbook namespace.`

---

## Exercice 2  Déployer un Pod

### Charger l'image dans le cluster

Le cluster ne voit pas les images du Docker local : minikube a son propre stockage.

```bash
docker images guestbook
minikube image load guestbook:v0.1.0
minikube image ls | grep guestbook
```



### Manifest

`pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guestbook
  namespace: guestbook
spec:
  containers:
    - name: guestbook
      image: guestbook:v0.1.0
      ports:
        - containerPort: 3000
```

### Déploiement

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl logs guestbook
kubectl describe pod guestbook
```


### Accès

```bash
kubectl port-forward pod/guestbook 3000:3000
```

Page accessible sur `http://localhost:3000`, avec l'avertissement « No database connection » attendu sans Redis.

### Test d'une image inexistante

```bash
kubectl run test-erreur --image=guestbook:v9.9.9
sleep 20
kubectl get pods
kubectl describe pod test-erreur | tail -15
kubectl delete pod test-erreur
```

Statut : `ImagePullBackOff`. Events :

```
Normal   Pulling  7s (x2 over 20s)  Pulling image "guestbook:v9.9.9"
Warning  Failed   7s (x2 over 19s)  Failed to pull image: failed to resolve reference
                                    "docker.io/library/guestbook:v9.9.9": pull access denied,
                                    repository does not exist
```


---

## Exercice 3  Deployment et Service

```bash
kubectl delete -f pod.yaml
```

### Manifests

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  namespace: guestbook
spec:
  replicas: 2
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
        - name: guestbook
          image: guestbook:v0.1.0
          ports:
            - containerPort: 3000
```

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook
  namespace: guestbook
spec:
  selector:
    app: guestbook
  ports:
    - port: 3000
      targetPort: 3000
```

### Déploiement

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get all
kubectl get endpointslices
```


### Test d'auto-réparation

```bash
kubectl get pods
kubectl delete pod guestbook-6cf5745bd6-7kld7
kubectl get pods
```

Un pod de remplacement apparaît une seconde après la suppression. Le ReplicaSet compare en continu l'état réel à l'état désiré.

### Mise à l'échelle

```bash
kubectl scale deployment guestbook --replicas=4
kubectl get pods
```

### Rupture du port-forward

Pendant la suppression du pod, le port-forward lancé dans l'autre terminal s'est interrompu :

```
error forwarding port 3000 to pod 99f208d1... network namespace for sandbox is closed
error: lost connection to pod
```

Bien que la commande vise `service/guestbook`, `kubectl port-forward` sélectionne un seul pod au démarrage et y reste attaché. Il ne répartit pas la charge. Il suffit de le relancer.

### Test d'un mauvais selector

Dans `service.yaml`, `app: guestbook` remplacé par `app: mauvais` :

```bash
kubectl apply -f service.yaml
kubectl get endpointslices
```

```
NAME              ADDRESSTYPE   PORTS     ENDPOINTS   AGE
guestbook-k7q29   IPv4          <unset>   <unset>     8m17s
```

La liste est vide et l'application devient injoignable, alors que le Service existe toujours avec son IP. Après remise du bon selector et réapplication, les quatre endpoints reviennent.

---

## Exercice 4  Chart Helm

```bash
kubectl delete -f deployment.yaml -f service.yaml
mkdir -p chart/templates
```

La suppression préalable est nécessaire : les ressources créées à la main portent les mêmes noms, et Helm refuse d'adopter des objets qu'il n'a pas créés.

### Structure

`chart/Chart.yaml`

```yaml
apiVersion: v2
name: guestbook
description: Application guestbook
type: application
version: 0.1.0
appVersion: "0.1.0"
```

`chart/values.yaml`

```yaml
replicaCount: 2

image:
  repository: guestbook
  tag: v0.1.0

service:
  port: 3000
```

`chart/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: guestbook
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

`chart/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
```

Les templates ne contiennent pas de `namespace:` : Helm le gère avec l'option `-n`.

### Erreur de chemin

```
Error: path "./chart" not found
```

Commande lancée depuis `chart/templates/`. Le chemin `./chart` est relatif au répertoire courant : toutes les commandes `helm` se lancent depuis la racine du projet.

### Prévisualisation et déploiement

```bash
cd ~/projects/TP_DevOps_2
helm template guestbook ./chart
helm install guestbook ./chart -n guestbook
helm list -n guestbook
kubectl get all
```

`helm template` affiche le YAML final sans rien déployer : les variables sont substituées, `{{ .Release.Name }}` devient `guestbook`.

### Mise à jour

```bash
helm upgrade guestbook ./chart -n guestbook --set replicaCount=3
kubectl get pods
helm history guestbook -n guestbook
```

Puis en modifiant `replicaCount: 3` directement dans `values.yaml` :

```bash
helm upgrade guestbook ./chart -n guestbook
kubectl get pods
```

### Rollback

```bash
helm rollback guestbook 1 -n guestbook
kubectl get pods
helm history guestbook -n guestbook
```

```
REVISION  UPDATED                   STATUS      DESCRIPTION
1         Sun Nov  1 21:02:11 2026  superseded  Install complete
2         Sun Nov  1 21:02:33 2026  superseded  Upgrade complete
3         Sun Nov  1 21:04:06 2026  superseded  Upgrade complete
4         Sun Nov  1 21:05:35 2026  deployed    Rollback to 1
```


---

## Exercice 5 — Ajouter une base de données (Redis)

