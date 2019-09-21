# knative-study-doc

## Introduction à Knative

L'objectif de Knative est d'utiliser les conteneurs et leur orchestration de Kubernetes pour répondre à des besoins principaux de :
- Autoscaling
- Minimisation de la consommation de ressources
- Routage de requêtes
- Gestionnaire d'événements

Le tout en y repondant de la manière la plus simple possible pour les développeurs.

Pour y parvenir, Knative s'installe comme un ensemble de plugins pour Kubernetes :

* **Serving** : gère les déploiements sur K8s avec des objets d'API dédiés
* **Eventing** : offre un système d'événements pour les communications inter-conteneur

Serving étant l'élément central de Knative, tandis que Eventing être utilisés optionnellement.

Jusqu'en version 0.7, Knative embarquait également un plugin Build dédié aux builds d'images. Cependant celui-ci a été abandonné au profit d'un projet tiers : [Tekton Pipelines](https://github.com/tektoncd/pipeline).

La première version de Knative remonte à juillet 2018, ce qui en fait un projet encore très jeune. En effet, alors que la version 0.9 a été publié courant septembre 2019, toutes les versions sont marquées par l'équipe du projet comme des pre-releases. Cependant au rythme où vont les versions, le projet semble proche d'une première version stable d'ici la fin de l'année 2019.

En tant que principal contributeur, Google commercialise déjà un produit basé sur Knative : [Cloud Run](https://cloud.google.com/run/). Mais pour les raisons évoquées ci-dessus, Cloud Run est toujours marqué comme "beta" au 4e trimestre 2019.

## Déploiement

Comme Knative n'est qu'une surcouche à Kubernetes, **l'unité de base reste le Pod**.
Ce n'est donc pas un système de FaaS (function as a service) tel que AWS Lambda, car il reste toujours au développeur à gérer la création d'un `Dockerfile` qui va spécifier finement dans quel environnement son code va fonctionner.

Ce fonctionnement est donc plus adapté à des applications de petite taille telles que des microservices, avec potentiellement des environnements d'exécution hétérogènes pour chacune d'entre elles. À noter que **les applications fonctionnant avec Knative doivent être Stateless**.

En revanche, **la spécification du déploiement est facilitée** par rapport à K8s grâce à des ressources spécifiques à Knative.
Au plus court, un développeur peut par exemple spécifier en YAML dans une seule ressource Knative de type "Service" le déploiement équivalent à : 1 Deployment + 1 Service + 1 Ingress de Kubernetes.

Pour plus de flexibilité, un "Service" de Knative peut être subdivisé en une ressource "Configuration" pour le déploiement à proprement parler et une "Route" pour son accès depuis l'extérieur du cluster.

### Avantages / Inconvénients

[+] Knative permet donc d'**abstraire** et de **simplifier la manipulation des objets de K8s aux développeurs**. 

[+] Les ops peuvent quant à eux continuer d'observer les objets de bas niveau (Pods, ReplicaSets...) comme ils en ont l'habitude, mais laissent Knative les gérer avec ses propres mécanismes de configuration.

[-] L'environnement d'exécution (l'image de conteneur) doit toujours être maîtrisé par les développeurs, au contraire de AWS Lambda qui permet de s'en abstraire. 

[+] Mais en contrepartie cela permet plus de flexibilité qu'un système de FaaS où les environnements d'exécution sont prédéfinis et limités.

## Scaling

### Fonctionnement

Knative propose un **autoscaling basé sur le nombre de requêtes HTTP concurrentes par pod**. Alors dès qu'il n'y a plus assez de pods pour être en capacité de se répartir un afflux de requêtes, Knative va créer un ou plusieurs nouveaux pods pour absorber ce trafic. De plus, aucune requête n'est perdue puisque celles qui vont être traitées par les pods supplémentaires sont mises en attente jusqu'à ce qu'ils soient prêts à les traiter.

Par contraste, l'autoscaling standard proposé par K8s se base sur la consommation de CPU ou de RAM pour déclencher des créations/destructions de pods d'une même application. Dans certains cas (ex : consommation de RAM sur une app Java), cela implique que les applications puissent réagir au scaling pour répartir l'excès de consommation de ressource sur les nouveaux pods créés par l'autoscaling.

L'approche par nombre de requêtes concurrentes de Knative rend donc l'application globalement plus agnostique du scaling qu'avec K8s utilisé seul. De cette manière le scaling n'a pas d'impact sur les développements de l'application, qui restent de simples mini-serveurs HTTP.

### Serverless

Ce qui fait de Knative une solution "Serverless" est qu'en cas d'inactivité après un certain temps (par défaut 5 minutes) sur une application, l'autoscaling peut réduire son nombre de pods à zéro.

Dans un environnement de cloud public facturé à la consommation de ressources, cela permet alors de générer des économies en ne laissant pas tourner des pods "dans le vide".

Lorsqu'une requête HTTP arrive à destination d'une application avec zéro pod, Knative la met en attente et déclenche la création d'un pod pour la traiter.

### Aperçu technique pour les développeurs

Pour être déployée avec Knative, **une application doit avoir ses images de conteneurs construites puis stockées dans un registre accessible par le cluster Kubernetes**. Pour rappel, le "build" des images n'entre pas dans le périmètre de Knative.

À partir de là, le principal (voire unique) type de ressource Knative qu'un développeur doit connaître est Service, dont voici un exemple :

```yaml
apiVersion: serving.knative.dev/v1beta1
kind: Service
metadata:
  # Nom du service
  name: my-knative-service
  # Namespace auquel le service doit appartenir
  namespace: my-project

spec:

  # Partie "Configuration" du Service
  template:
    metadata:
      name: my-deployment-rev1
      annotations:
        # Fixe le nombre de requêtes HTTP simultanées pour l'autoscaling à 100
        autoscaling.knative.dev/target: "100"
    spec:
      # Définition des conteneurs fournissant le Service
      containers:
      - image: gcr.io/knative-samples/autoscale-go:0.1

  # Partie "Route" du Service
  traffic:
  - tag: current
    revisionName: my-deployment-rev1
    percent: 100
  - tag: latest
    latestRevision: true
    percent: 0
```

La partie `template` indique à Knative de générer une ressource Revision nommée `my-deployment-rev1` afin de déployer le conteneur.

D'autre part, la partie `traffic` provoque la génération d'une ressource Knative Route dirigeant 100% du trafic de requêtes vers l'application définie dans la Revision `my-deployment-rev1`.

Dans cet exemple, si seule la partie `template` est modifiée et qu'elle porte un nouveau nom de Revision, alors 100% du trafic restera malgré cela affecté à la Revision initiale nommée `my-deployment-rev1`, comme définit dans la section `traffic`.

C'est avec ce mécanisme d'allocation de trafic qu'il est possible de procéder à de la répartition différentiée (blue/green) :

```yaml
  traffic:
  - tag: current
    revisionName: my-deployment-rev1
    percent: 80
  - tag: candidate
    revisionName: my-deployment-rev2
    percent: 20
  - tag: latest
    latestRevision: true
    percent: 0
```

Cet exemple de définition de Route dirige 80% du trafic à `my-deployment-rev1` et seulement 20% à `my-deployment-rev2` qui peut correspondre à une nouvelle version de l'application en déploiement progressif.

Note : l'URL d'accès à l'application s'obtient en consultant la clé `.status.url` de la ressource Knative Service. Le nom de domaine est normalement composé du nom du Service, du nom du Namespace, et du suffixe de domaine (ex : `my-knative-service.my-project.example.com`).

### Avantages / Inconvénients

[+] Knative permet de rendre une application disponible et scalable à partir de zéro pod par déploiement (en cas d'inactivité), ce qui permet plus d'**économies de ressources** que K8s qui nécessite qu'au moins un pod par déploiement existe.

[-] Cependant pour que le scale-up soit le plus transparent possible, **le développeur doit s'assurer que l'application démarre le plus rapidement possible**. Car pendant que l'application démarre, les requêtes qui lui parviennent sont mises en attente.


## Monitoring

Knative apporte des facilités d'installation de composants de monitoring. D'autre part, aucune configuration additionnelle n'est normalement requise au niveau des applications pour faire remonter les données de monitoring aux composants de Knative.

### Métriques

Le composant Serving de Kubernetes embarque par défaut une installation de Prometheus et Grafana. Ce dernier embarque un certain nombre de tableaux de bord préinstallés :

* Revision HTTP Requests : compteur (+ latence + taille) des requêtes HTTP par Revision. Une Revision étant une "version" de Configuration, tout comme les ReplicaSets sont des versions de Deployment dans K8s
* Nodes : métriques système et réseau de niveau Node
* Pods : métriques système et réseau de niveau Pod
* Deployment : métriques système et réseau de niveau Deployment
* Istio, Mixer and Pilot : métriques de ces composants utilisés en interne par Knative
* Kubernetes : métriques de niveau cluster K8s

### Logs

Serving propose 3 systèmes de logs au choix :

* Elasticsearch + Kibana embarqué
* Stackdriver dans GCP
* Autre système de logs personnalisé

Les catégories de logs proposées sont :

* Les sorties standard (`stdout`, `stderr`) des applications
* Les requêtes HTTP reçues par les applications
* Les changements de Configuration (et Revision) de déploiements Knative

### Traces

Knative propose d'embarquer 2 outils de traces au choix :

* Zipkin
* Jaeger

Quel que soit le choix, Knative s'assure de la collecte des traces depuis les applications vers l'outil de tracing, ce qui évite d'avoir aux devs et aux ops de s'en préoccuper.

## Conclusion

Knative étend Kubernetes en lui apportant une solution pour le déploiement d'applications web conteneurisées Serverless et scalables. De plus, ses ressources de configuration spécifiques évitent aux développeurs de rentrer trop dans les détails des objets standards de Kubernetes, ce qui permet une meilleure maintenabilité et plus de clarté. 

Son fonctionnement basé sur les conteneurs le rend suffisamment générique et flexible pour couvrir le plus de cas d'utilisation "Serverless" possible. Seulement, les développeurs doivent veiller à ce que le temps de "warm-up" des conteneurs reste le plus réduit possible.

En plus de ces apports centrés sur le déploiement, Knative vient avec des solutions préparamétrées pour les métriques, les logs et les traces, toutes basées sur des logiciels éprouvés dans ces domaines (Prometheus, Grafana, Elasticsearch, Kibana...).

En revanche, Knative ne propose plus d'aide au "build" d'images Docker pour rester centré sur le déploiement (Serving) et la gestion d'événements (Eventing).

Le projet est encore jeune mais en cours de stabilisation, porté notamment par Google qui propose déjà une offre commerciale basée dessus.
