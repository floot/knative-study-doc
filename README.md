# knative-study-doc

## Introduction à Knative

L'objectif de Knative est d'utiliser les conteneurs et leur orchestration de Kubernetes pour répondre à des besoins principaux de :
- Autoscaling
- Minimisation de la consommation de ressources

Pour y parvenir, Knative s'installe comme un ensemble de plugins pour Kubernetes :

* **Serving** : gère les déploiements sur K8s avec des objets d'API dédiés
* **Build** : apporte des facilités de build d'images de conteneurs à K8s
* **Eventing** : offre un système d'événements pour les communications inter-conteneur

Serving étant l'élément central de Knative, tandis que les deux autres peuvent être utilisés optionnellement.

## Déploiement

Comme Knative n'est qu'une surcouche à Kubernetes, **l'unité de base reste le Pod**.
Ce n'est donc pas un système de FaaS (function as a service) tel que AWS Lambda, car il reste toujours au développeur à gérer la création d'un `Dockerfile` qui va spécifier finement dans quel environnement son code va fonctionner.

Ce fonctionnement est donc plus adapté à des applications de petite taille telles que des microservices, avec potentiellement des environnements d'exécution hétérogènes pour chacune d'entre elles.

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

### Avantages / Inconvénients

[+] Knative permet de rendre une application disponible et scalable à partir de zéro pod par déploiement (en cas d'inactivité), ce qui permet plus d'**économies de ressources** que K8s qui nécessite qu'au moins un pod par déploiement existe.

[-] Cependant pour que le scale-up soit le plus transparent possible, **le développeur doit s'assurer que l'application démarre le plus rapidement possible**. Car pendant que l'application démarre, les requêtes qui lui parviennent sont mises en attente.
