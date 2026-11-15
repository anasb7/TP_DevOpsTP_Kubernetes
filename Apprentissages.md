# Apprentissages

## Exercice 0 : préparer l'environnement

Installer minikube, kubectl, helm, k9s, stern, kubectx et kubens, puis démarrer un cluster local.

### Problème rencontré et pourquoi il est survenu

Sept outils sur huit répondaient correctement. Docker, lui, renvoyait :

```
The command 'docker' could not be found in this WSL 2 distro.
```

Docker fonctionnait pourtant dans cette même distro au TP précédent. L'intégration WSL de Docker Desktop avait été désactivée, ou l'application n'était pas lancée. C'est bloquant : minikube utilise le driver `docker` pour créer son nœud, et il faut Docker pour construire puis charger l'image du guestbook.

### Solution appliquée et pourquoi cette solution fonctionne

Lancer Docker Desktop, puis Settings → Resources → WSL Integration, activer Ubuntu, Apply & Restart, et rouvrir le terminal. Le socket Docker redevient accessible depuis la distro.

Le test décisif est `docker run --rm hello-world` et non `docker --version` : la seconde commande peut répondre alors que le démon reste injoignable.

Le cluster démarre ensuite avec `minikube start --driver=docker`.

### Ce que j'ai appris

Qu'un outil installé et un outil joignable sont deux choses différentes. `docker --version` interroge le client, `docker run` interroge le démon ; seul le second prouve que la chaîne complète fonctionne.

Et que sous WSL, Docker n'est pas installé dans la distro : c'est une passerelle vers Docker Desktop côté Windows. Une case décochée suffit à le faire disparaître.

---

## Exercice 1 : explorer le cluster

Découvrir les commandes `kubectl` de base et créer les namespaces `guestbook` et `data`.

### Problème rencontré et pourquoi il est survenu

`kubectl get all` ne renvoyait rien, alors que le cluster tournait manifestement. La commande ne liste que le namespace courant, ici `default`, qui est vide. Tout ce qui fait fonctionner le cluster se trouve dans `kube-system`, visible seulement avec `--all-namespaces`.

### Solution appliquée et pourquoi cette solution fonctionne

```
kubectl get pods --all-namespaces
```

révèle le control plane : CoreDNS, etcd, l'API server, le scheduler, le controller manager, kube-proxy.

Puis création des deux namespaces et bascule avec `kubens guestbook`, ce qui évite de répéter `-n guestbook` sur chaque commande.

### Ce que j'ai appris

Que la plupart des commandes `kubectl` sont implicitement limitées au namespace courant, et que c'est une source d'erreur permanente : une ressource « disparue » est souvent une ressource cherchée dans le mauvais namespace.

Que le control plane n'est pas extérieur au cluster, il y tourne comme n'importe quelle autre application, sous forme de pods.

---

## Exercice 2 : déployer un Pod

Lancer le guestbook du TP précédent sur Kubernetes sous la forme d'un Pod.

### Problème rencontré et pourquoi il est survenu

Le cluster ne voit pas les images du Docker local : minikube tourne dans son propre conteneur, avec son propre stockage d'images.

En déployant volontairement une image inexistante pour observer la réaction de Kubernetes, le pod est passé en `ErrImagePull` puis `ImagePullBackOff`, avec ce message :

```
failed to pull "docker.io/library/guestbook:v9.9.9": pull access denied,
repository does not exist
```

Un nom d'image sans registre explicite est complété en `docker.io/library/<nom>`. Ne trouvant rien localement, Kubernetes est allé chercher sur Docker Hub, où cette image n'existe pas.

### Solution appliquée et pourquoi cette solution fonctionne

```
minikube image load guestbook:v0.1.0
```

copie l'image depuis le Docker local vers le stockage du cluster. Le `describe` du pod le confirme ensuite : `Container image already present on machine`, aucun téléchargement.

L'accès se fait avec `kubectl port-forward`, car l'IP du pod (`10.244.0.3`) appartient au réseau interne du cluster et n'est pas joignable depuis la machine.

### Ce que j'ai appris

Que `ImagePullBackOff` n'est pas un échec définitif : les Events montrent `x2 over 20s`, Kubernetes réessaie indéfiniment avec un délai croissant. Il attend que l'image devienne disponible plutôt que d'abandonner.

Que le `describe` est l'outil de diagnostic principal, et que sa section `Events` en bas est la plus utile.

Et, en lisant ce `describe`, que mon pod avait `Labels: <none>`, ce qui a pris son sens à l'exercice suivant.

---

## Exercice 3 : Deployment et Service

Remplacer le pod isolé par un Deployment répliqué, et l'exposer via un Service.

### Problème rencontré et pourquoi il est survenu

Deux observations.

En supprimant un pod pour tester l'auto-réparation, le `port-forward` lancé dans l'autre terminal s'est interrompu :

```
error forwarding port 3000 to pod 99f208d1... network namespace for sandbox is closed
```

Alors que la commande visait `service/guestbook`, le port-forward avait choisi un pod précis au démarrage et y restait attaché. Il ne répartit pas la charge.

Ensuite, en remplaçant volontairement `app: guestbook` par `app: mauvais` dans le selector du Service, la liste des endpoints est devenue vide et l'application est devenue injoignable, alors que le Service existait toujours avec son IP.

### Solution appliquée et pourquoi cette solution fonctionne

Relancer le port-forward suffit, puisqu'il se rattache alors à un pod vivant.

Pour le Service, remettre le selector en accord avec les labels du template du Deployment. La vérification se fait avec :

```
kubectl get endpointslices
```

qui liste les IP des pods effectivement trouvés. Une liste vide signifie que le selector ne correspond à rien.

### Ce que j'ai appris

Que rien n'est câblé en dur dans Kubernetes : le Service ne connaît pas le Deployment, il cherche en continu les pods portant les bons labels. Une faute de frappe dans un label casse la liaison sans produire la moindre erreur au `apply`.

Que la hiérarchie est Deployment → ReplicaSet → Pods. Le Deployment ne gère pas les pods directement, c'est ce qui rend le rolling update possible : deux ReplicaSets coexistent pendant la transition.

Que `kubectl port-forward` est un outil de test, pas un mode d'accès : il vise un seul pod, ce qui fausserait un test de charge.

---

## Exercice 4 : chart Helm

Packager le Deployment et le Service dans un chart Helm paramétrable.

### Problème rencontré et pourquoi il est survenu

Deux erreurs.

```
Error: path "./chart" not found
```

Le chemin `./chart` est relatif au répertoire courant, et j'étais dans `chart/templates/`.

Ensuite, le Deployment et le Service créés à la main portaient déjà les noms `guestbook`. Helm refuse de prendre le contrôle de ressources qu'il n'a pas créées, faute des annotations de propriété qu'il pose lui-même.

### Solution appliquée et pourquoi cette solution fonctionne

Lancer les commandes `helm` depuis la racine du projet, et supprimer les ressources manuelles avant l'installation :

```
kubectl delete -f deployment.yaml -f service.yaml
helm install guestbook ./chart -n guestbook
```

Les templates utilisent `{{ .Release.Name }}` et les valeurs de `values.yaml`, ce qui permet de modifier le nombre de replicas ou le tag de l'image sans toucher aux manifests.

`helm template` affiche le YAML final sans rien déployer : c'est le réflexe à prendre avant chaque `install` ou `upgrade`.

### Ce que j'ai appris

Que Helm ne se contente pas d'appliquer du YAML, il tient un registre de ce qu'il possède. D'où le refus d'adopter des ressources préexistantes, et la possibilité de faire des rollbacks.

Que `helm rollback` crée une nouvelle révision au lieu de supprimer les précédentes : l'historique n'est jamais réécrit, on peut donc revenir en arrière d'un rollback.

Qu'après un rollback, `values.yaml` et l'état déployé divergent : le fichier indiquait 3 replicas alors que le cluster en avait 2. Le fichier sur disque décrit une intention, pas l'état réel.