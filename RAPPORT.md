# TP 2 — GitOps avec ArgoCD : DevHub Campus

## Étape 0 — Outillage

Le poste utilisé pour le TP est un Mac Apple Silicon (`darwin/arm64`).

| Outil | Version utilisée | Version minimale demandée | État |
|---|---:|---:|---|
| Docker | 29.5.3 | — | Conforme |
| kubectl | 1.34.1 | 1.30 | Conforme |
| Kustomize intégré à kubectl | 5.7.1 | — | Conforme |
| Helm | 4.2.3 | 3.14 | Conforme, compatibilité avec le TP à surveiller |
| kind | 0.32.0 | 0.22 | Conforme |
| ArgoCD CLI | 3.4.5 | 2.11 | Conforme |
| Git | 2.50.1 | 2.40 | Conforme |
| yq | 4.53.3 | 4.40 | Conforme |

Les versions ont été vérifiées avec les commandes suivantes :

```text
docker version
kubectl version --client
helm version
kind version
argocd version --client
git --version
yq --version
```

Le cluster local du TP est créé avec kind et porte le nom `devhub`. Il utilise la configuration `cluster/kind-config.yaml` fournie dans le squelette.

Le contexte Kubernetes actif est `kind-devhub`. Le cluster contient deux nœuds Kubernetes 1.36.1 : un control-plane et un worker. Après l'initialisation, les deux nœuds sont passés à l'état `Ready` et tous les pods système étaient `Running`.

## Étape 1 — GitOps en une page

### Les quatre principes de GitOps

1. **Déclaratif** : Git décrit l'état final attendu de la plateforme, par exemple la version d'une image, le nombre de répliques et la configuration de l'Ingress. Il ne contient pas une suite de commandes impératives à exécuter.
2. **Versionné et immuable** : chaque changement de l'état désiré est enregistré dans un commit. L'historique permet de connaître l'auteur et la raison d'un changement, puis de revenir à une version précédente.
3. **Tiré automatiquement** : un agent présent dans le cluster, ici ArgoCD, lit Git. La CI n'a donc pas besoin d'envoyer les manifests au cluster ni de posséder un kubeconfig de déploiement.
4. **Réconcilié continuellement** : ArgoCD compare régulièrement l'état réel du cluster à l'état désiré dans Git. Il signale les écarts et peut les corriger automatiquement.

### Comparaison des flux push et pull

```text
Modèle push

Développeur ──push──> Git ──déclenche──> CI/CD ──kubectl apply──> Kubernetes
                                         │
                                         └── possède des droits sur le cluster

Modèle pull / GitOps

Développeur ──push──> Git <──lecture et comparaison── ArgoCD ──réconcilie──> Kubernetes
                                                   installé dans le cluster
```

Dans le modèle push, la pipeline agit directement sur le cluster. Dans le modèle pull, Git contient l'état désiré et ArgoCD est responsable de faire converger Kubernetes vers cet état.

| Question | Push (`kubectl apply` en CI) | Pull (ArgoCD) |
|---|---|---|
| Qui a les droits sur le cluster ? | La CI possède un kubeconfig ou un compte de service disposant de droits de déploiement. | ArgoCD possède les droits nécessaires dans le cluster ; la CI n'en a pas besoin. |
| Où est l'historique des changements ? | Une partie se trouve dans Git et une autre dans l'historique d'exécution de la CI. Une action manuelle peut échapper aux deux. | L'état désiré et ses changements sont enregistrés dans Git. ArgoCD conserve aussi l'historique de ses synchronisations. |
| Que se passe-t-il si un développeur modifie le cluster à la main ? | La dérive peut rester invisible jusqu'au prochain déploiement, qui risque de l'écraser. | ArgoCD détecte une différence `OutOfSync` et peut la corriger si `selfHeal` est activé. |
| Comment ajouter un environnement de plus ? | Il faut adapter les manifests et la pipeline, créer les accès et autoriser la CI à cibler le nouvel environnement. | On ajoute dans Git une `Application` ou une règle `ApplicationSet` décrivant la nouvelle destination. |
| Comment faire un rollback ? | On relance la CI avec une ancienne version ou on utilise une commande Kubernetes comme `kubectl rollout undo`. | On effectue un `git revert` ; ArgoCD détecte le nouvel état désiré et réconcilie le cluster. |
| Combien de pipelines pour 30 services ? | Chaque service possède généralement une pipeline avec une logique et des identifiants de déploiement à maintenir. | Les pipelines construisent les images, tandis qu'ArgoCD peut piloter les déploiements des 30 services depuis des déclarations homogènes. |
| Qui voit en direct ce qui tourne ? | Les personnes ayant accès à Kubernetes ou aux sorties des pipelines doivent croiser plusieurs sources. | Les utilisateurs autorisés voient dans l'interface ArgoCD la révision, la santé et l'état de synchronisation. |

### Prise de position personnelle

Pour un petit projet personnel, je commencerais par un modèle push, car il est plus rapide à mettre en place et le coût initial d'ArgoCD serait difficile à justifier pour un seul service et un seul environnement. Je passerais au modèle pull dès que le projet comporte plusieurs environnements, plusieurs services ou plusieurs contributeurs, car la détection du drift, la traçabilité et la centralisation des déploiements deviennent alors réellement utiles.

## Étape 2 — Glossaire ArgoCD

### Application

Une `Application` est une ressource personnalisée ArgoCD qui associe une source versionnée à une destination Kubernetes. Elle précise notamment le dépôt Git, la révision, le chemin des manifests ou du chart Helm, le cluster et le namespace cibles, ainsi que la politique de synchronisation.

**Exemple dans DevHub Campus :** l'`Application` `annuaire-dev` lit le chart du service annuaire sur la branche `main` et le déploie dans le namespace `devhub-dev`. Elle ne doit pas être confondue avec l'application métier annuaire elle-même.

### AppProject

Un `AppProject` représente une frontière de sécurité et d'organisation dans ArgoCD. Il contrôle les dépôts sources autorisés, les clusters et namespaces de destination, les types de ressources utilisables et éventuellement les rôles et fenêtres de synchronisation.

**Exemple dans DevHub Campus :** le projet `devhub` autorise seulement les dépôts des services du campus et les namespaces dont le nom commence par `devhub-`. Ce n'est ni un dépôt Git ni un namespace Kubernetes.

### Source

La `source` indique à ArgoCD où trouver l'état désiré. Elle peut désigner un dépôt Git, un chemin dans ce dépôt et une révision, ou directement un chart Helm publié.

**Exemple dans DevHub Campus :** la source de `planning-dev` est la branche `main` du dépôt DevHub, au chemin `services/planning/chart`, avec le fichier `values-dev.yaml`.

### Destination

La `destination` est l'endroit où ArgoCD doit créer les ressources rendues depuis la source. Elle est composée d'un cluster Kubernetes et d'un namespace.

**Exemple dans DevHub Campus :** les trois services stables ciblent le cluster `https://kubernetes.default.svc` et le namespace `devhub-dev`, tandis qu'une preview cible un namespace `devhub-preview-*`.

### Sync

Une synchronisation est l'opération par laquelle ArgoCD compare l'état désiré à l'état réel puis applique les changements nécessaires. Elle peut être manuelle ou automatique. L'option `selfHeal` permet de corriger automatiquement une dérive créée directement dans le cluster.

**Exemple dans DevHub Campus :** après un commit changeant le tag de l'image annuaire, ArgoCD synchronise le `Deployment` afin qu'il utilise cette nouvelle image. Contrairement à un simple `kubectl apply`, ArgoCD conserve le lien avec Git et continue ensuite à surveiller la ressource.

### Prune

Le `prune` autorise ArgoCD à supprimer du cluster une ressource qui n'existe plus dans l'état désiré stocké dans Git. Il complète la synchronisation, qui ne supprimerait sinon pas nécessairement les anciennes ressources.

**Exemple dans DevHub Campus :** si `service.yaml` est supprimé du chart annuaire et que `prune` est actif, ArgoCD supprime le `Service` correspondant. Pour les previews, `prune` est indispensable afin de nettoyer les ressources après la suppression d'une branche.

### App of Apps

Le pattern App of Apps utilise une `Application` racine dont le contenu est un ensemble de manifests d'autres `Application`. La racine permet ainsi à ArgoCD de créer et de piloter déclarativement les applications enfants.

**Exemple dans DevHub Campus :** l'Application `root` lit `platform/apps/dev/` et crée `annuaire-dev`, `planning-dev` et `notif-dev`. Ce n'est pas un chart Helm qui embarque les trois charts des services.

### ApplicationSet

Un `ApplicationSet` est une ressource qui génère plusieurs `Application` à partir d'un modèle et d'une source de paramètres appelée generator. Il évite de copier manuellement presque le même manifeste pour chaque branche, PR, cluster ou environnement.

**Exemple dans DevHub Campus :** un générateur inspecte les branches `feature/*` et produit pour chacune une Application de preview avec un nom, un namespace et un Ingress propres.

### Sync wave

Une sync wave donne un ordre relatif aux ressources d'une même synchronisation. Les valeurs les plus basses sont traitées en premier ; ArgoCD attend que la vague précédente soit correctement appliquée avant de poursuivre.

**Exemple dans DevHub Campus :** un `ConfigMap` annoté avec la vague `-1` est appliqué avant le `Deployment` annoté avec la vague `0`. Une vague n'est pas une boucle de nouvelle tentative.

### Hooks PreSync, Sync et PostSync

Les hooks sont des ressources exécutées pendant une phase précise de la synchronisation. Un hook `PreSync` s'exécute avant les ressources normales, un hook `Sync` pendant la synchronisation et un hook `PostSync` après une synchronisation réussie. Un échec de `PreSync` bloque la suite du déploiement.

**Exemple dans DevHub Campus :** un `Job` `PreSync` simule une migration de base de données et affiche `migration ok` avant le déploiement du service annuaire. Ce mécanisme ne doit pas être confondu avec un webhook ou un déclencheur Git.

### Différences importantes

- `Refresh` demande à ArgoCD de relire la source et de recalculer l'écart ; `Sync` applique cet état au cluster.
- `selfHeal` corrige une ressource modifiée hors de Git ; `prune` supprime une ressource qui a disparu de Git.
- Une sync wave détermine un ordre entre ressources ; un hook associe une ressource à une phase du déploiement.
- Une `Application` déploie un état ; un `ApplicationSet` produit plusieurs objets `Application` à partir d'un modèle.

## Étape 3 — Conteneurisation du service annuaire

Le service choisi pour détailler la conteneurisation est `annuaire`, écrit en Node.js avec Express. Avant sa publication, son image a été construite et testée localement sous le nom `devhub-annuaire:test`.

### Choix de construction

Le Dockerfile utilise deux stages basés sur Node.js 20 Alpine. Le premier stage installe les dépendances de production de façon reproductible avec `npm ci --omit=dev`. Le second ne récupère que `node_modules`, le code source et `package.json`. Cette séparation évite de copier le cache npm et les fichiers de construction inutiles dans l'image finale.

Le processus s'exécute avec l'UID et le GID `1001`, et non avec `root`. Cet identifiant sera également déclaré dans le `securityContext` Kubernetes du chart Helm. Aucun secret n'est transmis par `ARG` ou `ENV`. Le label OCI `org.opencontainers.image.source` est renseigné au moment du build avec l'URL du dépôt Git.

### Validation locale

L'image a été démarrée avec `LOG_LEVEL=debug`. Le service a produit un message de niveau debug puis un message indiquant son écoute sur le port 8080 :

```text
{"level":"debug","msg":"LOG_LEVEL=debug"}
{"level":"info","msg":"annuaire up on :8080"}
```

Les deux endpoints ont été testés :

| Endpoint | Résultat |
|---|---|
| `GET /healthz` | `200 OK`, avec `{"ok":true,"service":"annuaire"}` |
| `GET /students` | `200 OK`, avec la liste JSON des étudiants |

Le fichier `package-lock.json` a été généré afin de verrouiller les versions transitives et de permettre l'utilisation de `npm ci` dans le build.

L'inspection finale a confirmé que le conteneur s'exécute avec `user=1001:1001`. Docker Desktop indique une occupation disque de 197 Mo et une taille de contenu de 48,8 Mo. L'image respecte donc la limite de 200 Mo demandée, avec une marge faible sur la mesure d'occupation locale.

### Publication dans GHCR

L'image finale a été taguée avec le SHA court du commit, sans utiliser le tag `latest`, puis publiée dans GitHub Container Registry :

```text
ghcr.io/louis-guidez/annuaire:d022547
```

Le registre a retourné le digest immuable suivant :

```text
sha256:8ff8d6ecd2d26e859884c4b971aac6d5b5a833a64e048825139ba0c0f24710bd
```

## Étape 4 — Chart Helm du service annuaire

Le chart `services/annuaire/chart` produit un `Deployment`, un `Service` et, selon les values, un `Ingress`. Les éléments communs de nommage et de labels sont centralisés dans `_helpers.tpl` afin d'éviter les divergences entre ressources.

Les paramètres principaux sont exposés dans `values.yaml` : dépôt et tag de l'image, nombre de répliques, ports, niveau de log, Ingress, ressources, sécurité et probes. Trois surcharges limitées aux différences de chaque environnement sont disponibles :

- `values-dev.yaml` active l'Ingress local, le niveau debug et une seule réplique ;
- `values-staging.yaml` active l'Ingress de staging avec deux répliques héritées des valeurs par défaut ;
- `values-preview.yaml` réduit le nombre de répliques et les ressources, et prépare un hôte qui sera remplacé par l'`ApplicationSet`.

Toutes les ressources portent les labels `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/part-of: devhub-campus` et `app.kubernetes.io/managed-by: Helm`. Le sélecteur des Pods utilise uniquement les labels stables `name` et `instance`.

Le Pod impose l'UID/GID `1001`, `runAsNonRoot` et le profil seccomp `RuntimeDefault`. Le conteneur interdit l'élévation de privilèges, utilise un système de fichiers racine en lecture seule et abandonne toutes les capabilities Linux. Les probes de disponibilité et de vivacité interrogent `/healthz` sur le port HTTP nommé.

La commande `helm lint services/annuaire/chart` a produit le résultat suivant :

```text
1 chart(s) linted, 0 chart(s) failed
```

Le rendu avec `values-dev.yaml` contient une réplique, l'image `ghcr.io/louis-guidez/annuaire:d022547`, les deux probes, les contextes de sécurité et l'Ingress `annuaire.devhub.local`.

La validation Kubernetes locale a également réussi :

```text
service/annuaire-annuaire created (dry run)
deployment.apps/annuaire-annuaire created (dry run)
ingress.networking.k8s.io/annuaire-annuaire created (dry run)
```

## Étape 5 — Installation d'ArgoCD et première Application

ArgoCD a été installé avec le chart officiel `argo/argo-cd` dans le namespace `argocd`. Le contrôleur ingress-nginx a été installé auparavant afin d'exposer l'interface sur `argocd.devhub.local`. Tous les composants ArgoCD, y compris l'application controller, l'ApplicationSet controller, le repo server, Redis et le server, ont atteint l'état `Running` et `Ready`.

Le mot de passe initial du compte `admin` a été récupéré depuis le secret de bootstrap, utilisé pour la première connexion, puis immédiatement remplacé. Le secret initial a ensuite été supprimé. Aucun mot de passe n'est conservé dans le dépôt ni dans ce rapport.

### Première synchronisation

L'Application `annuaire-dev` a d'abord été créée avec une politique de synchronisation manuelle. Avant la première sync, ArgoCD indiquait `OutOfSync + Missing`, car le Service, le Deployment et l'Ingress existaient dans Git mais pas encore dans Kubernetes. Après la synchronisation manuelle, elle est passée à `Synced + Healthy` sur la révision `92a2164`.

Le Pod était `1/1 Running`. Les endpoints `/healthz` et `/students`, exposés sur `http://annuaire.devhub.local`, ont tous les deux répondu avec le statut `200 OK`. Une fois ce déploiement validé, l'auto-sync et `selfHeal` ont été activés, tandis que `prune` est resté désactivé.

### Différence entre self-heal et prune

`selfHeal: true` demande à ArgoCD de corriger automatiquement une ressource existante qui a été modifiée directement dans le cluster. Par exemple, si un utilisateur change manuellement le nombre de répliques du Deployment annuaire, ArgoCD restaure la valeur déclarée dans Git. Cette option serait dangereuse pendant un diagnostic d'urgence si une correction manuelle indispensable était immédiatement écrasée avant d'avoir pu être reportée dans Git.

`prune: true` autorise ArgoCD à supprimer une ressource réelle lorsqu'elle n'existe plus dans la source Git rendue. Par exemple, retirer le template du Service entraîne la suppression du Service Kubernetes. Cette option serait dangereuse si le chemin source était accidentellement modifié vers un dossier vide ou si une ressource importante était retirée du dépôt par erreur : ArgoCD pourrait propager immédiatement cette suppression.

## Étape 6 — Préparation de l'App of Apps

Afin que les trois Applications enfants puissent devenir saines, les services `planning` et `notif` ont également été conteneurisés et publiés. Leurs images respectent les mêmes contraintes de sécurité que l'image annuaire.

| Service | Image | Utilisateur | Taille disque | Taille du contenu | Digest |
|---|---|---:|---:|---:|---|
| planning | `ghcr.io/louis-guidez/planning:9f35860` | `1001:1001` | 118 Mo | 27,3 Mo | `sha256:6b98379c35d3497ee86f7c91d2078ba916f4b9e2ae5e663bf9f286c79ccffe9b` |
| notif | `ghcr.io/louis-guidez/notif:a9cf89c` | `65532:65532` | 13,3 Mo | 2,85 Mo | `sha256:bae55d0066ec800cae9852afd5835391ab94504583f273345abe99ed834cd9f4` |

Les endpoints `/healthz`, `/slots` et `/events` ont répondu avec le statut `200 OK`. Le service planning a notamment produit un message de niveau debug au démarrage. Son image initiale Debian Slim occupait 273 Mo ; le passage à Python Alpine et le retrait des extensions Uvicorn inutilisées ont ramené cette taille à 118 Mo.

La root Application utilise le projet intégré `default` uniquement pour le bootstrap. Elle lit deux sources dans le dépôt de plateforme : `platform/projects`, puis `platform/apps/dev`. L'AppProject `devhub` porte la sync wave `-1`, afin d'être créé avant les Applications qui le référencent. Il autorise uniquement le dépôt DevHub, le cluster local et les namespaces `devhub-*`. La seule ressource cluster-scoped autorisée est `Namespace`.

Après le bootstrap, la root est passée à `Synced + Healthy` et a créé ou configuré les trois Applications enfants `annuaire-dev`, `planning-dev` et `notif-dev`. Les trois services répondent avec le statut `200 OK` sur leurs Ingress respectifs :

- `http://annuaire.devhub.local/healthz` ;
- `http://planning.devhub.local/healthz` ;
- `http://notif.devhub.local/healthz`.

Un problème de compatibilité a été détecté après le bootstrap : kind 0.32 a créé un cluster Kubernetes 1.36, alors que la version 7.6.12 du chart fournie par le squelette installait ArgoCD 2.12.6. Le schéma statique de cette ancienne version ne connaissait pas le champ Kubernetes `Deployment.status.terminatingReplicas`, ce qui provoquait un `ComparisonError` malgré des workloads sains. Le chart ArgoCD a donc été mis à niveau vers la version 10.1.4, qui installe ArgoCD 3.4.5, afin d'aligner le contrôleur avec la version récente du cluster.

### Pourquoi App of Apps n'est pas un simple kubectl apply

Un `kubectl apply -f platform/apps/dev` ne ferait qu'envoyer une fois les manifests présents sur le poste de l'opérateur. Après cette commande, aucun composant ne garantirait que les ressources restent conformes au dépôt ni ne réagirait automatiquement à un nouveau commit. Avec App of Apps, la root conserve au contraire un lien permanent avec Git : elle détecte les ajouts, modifications et suppressions d'Applications, corrige leur drift avec `selfHeal` et nettoie les enfants retirés avec `prune`. Le bootstrap devient ainsi reproductible et continuellement réconcilié, au lieu d'être une opération ponctuelle dépendant du poste local.

## Étape 7 — Environnements de preview

Le générateur `pullRequest` a été retenu. La documentation actuelle d'ArgoCD limite le Git generator à la découverte de fichiers et de répertoires ; il n'énumère pas directement les branches d'un dépôt. Le pull request generator fournit en revanche la branche, son nom normalisé et son SHA, et supprime naturellement les Applications générées lorsque la PR est fermée.

Trois ApplicationSets suivent les PR dont la branche correspond à `feature/*`. Pour une branche `feature/demo-prof`, ils génèrent :

- `annuaire-preview-feature-demo-prof` ;
- `planning-preview-feature-demo-prof` ;
- `notif-preview-feature-demo-prof`.

Les trois Applications ciblent le namespace partagé `devhub-preview-feature-demo-prof`. Elles utilisent `values-preview.yaml`, remplacent le tag d'image par le SHA complet de la tête de PR produit par la CI, et exposent des Ingress distincts préfixés par le nom du service. `CreateNamespace=true`, `selfHeal=true` et `prune=true` garantissent respectivement la création, la réconciliation et le nettoyage de la preview.

Les charts ajoutent `devhub.io/env: preview` à toutes les ressources de preview. Le token utilisé pour interroger l'API GitHub est stocké uniquement dans un Secret Kubernetes du namespace `argocd` et n'est jamais versionné dans Git.

Le premier test de suppression a mis en évidence que `CreateNamespace=true` crée le namespace mais ne le place pas automatiquement dans l'inventaire des ressources à pruner. Les Applications et leurs workloads avaient disparu, tandis que le namespace vide restait présent. Le chart annuaire a donc reçu un template `Namespace` conditionnel, activé uniquement par l'ApplicationSet de preview et appliqué en sync wave `-1`. Le namespace fait ainsi partie de l'état GitOps et peut être supprimé par le finalizer et `prune` lors de la fermeture de la PR.

Un second test a permis d'isoler une autre condition nécessaire : les Applications générées doivent elles-mêmes porter `resources-finalizer.argocd.argoproj.io`. Sans ce finalizer, la suppression de l'objet Application ne déclenche pas la suppression en cascade de son inventaire, même si sa politique contient `prune: true`. Le finalizer a donc été ajouté aux trois templates ApplicationSet.

Après ajout du finalizer, la fermeture de la PR a supprimé les trois Applications, leurs workloads et le namespace de preview. La commande `kubectl get namespace -l devhub.io/env=preview` n'a retourné aucune ressource. Le cycle création puis destruction automatique est donc validé.

## Étape 8 — Bestiaire ArgoCD

### Scénario 1 — Drift manuel et self-heal

**Manipulation.** Le Deployment `annuaire-dev-annuaire`, déclaré avec une réplique dans Git, a été modifié directement avec `kubectl scale --replicas=5`.

**Observation.** Kubernetes a immédiatement affiché `DESIRED=5`. Cinq secondes plus tard, le Deployment était revenu à `DESIRED=1` et l'Application affichait de nouveau `Synced + Healthy`.

**Hypothèse et vérification.** La modification directe créait un écart entre l'état réel et le chart stocké dans Git. La politique automatisée avec `selfHeal: true` a détecté cet écart et réappliqué la valeur désirée sans commit ni intervention supplémentaire.

**Conclusion.** Une modification manuelle du cluster n'est pas durable lorsque self-heal est actif. Toute correction d'urgence que l'on souhaite conserver doit être reportée dans Git, sinon ArgoCD l'écrase lors de sa réconciliation.

### Scénario 2 — Image inexistante et état Synced + Degraded

**Manipulation.** Le tag de l'image annuaire a été remplacé dans Git par `tag-inexistant`, puis le commit `b61980f` a été poussé. Le délai d'échec du rollout a été fixé à 60 secondes afin que Kubernetes déclare rapidement un rollout bloqué.

**Observation.** ArgoCD a appliqué le nouveau Deployment et affiché `Synced`, mais le nouveau Pod est resté en `ImagePullBackOff`. Les événements Kubernetes indiquaient que la référence `ghcr.io/louis-guidez/annuaire:tag-inexistant` était introuvable. Après expiration du délai de progression, l'Application est passée à `Synced + Degraded`.

L'ancien Pod utilisant l'image `d022547` est resté `Running`, ce qui a permis au Service de continuer à répondre pendant l'échec du rollout.

**Hypothèse et vérification.** ArgoCD considère la synchronisation réussie lorsque l'état désiré a été transmis à Kubernetes. La santé est une dimension distincte : Kubernetes avait accepté le manifeste mais ne pouvait pas exécuter l'image demandée.

**Conclusion.** `Synced` ne signifie pas que l'application fonctionne. Face à `Synced + Degraded`, il faut examiner la ressource dégradée, puis l'état et les événements des Pods, et enfin leurs logs si le conteneur a pu démarrer.

### Scénario 3 — Rollback par git revert

**Manipulation.** Le commit fautif `b61980f` a été annulé avec `git revert`, sans modifier directement le cluster. Le revert a restauré le tag d'image `d022547` dans l'état désiré.

**Observation.** L'Application est d'abord passée à `OutOfSync + Degraded`, puis ArgoCD a appliqué la révision de revert `d8f9bab`. Elle est revenue à `Synced + Healthy`, le Pod fautif a disparu et `/healthz` a répondu `200 OK`.

**Mesure.** La convergence complète entre le push du revert et le retour à l'état sain a pris 33 secondes.

**Conclusion.** Le rollback GitOps ne contourne pas le contrôleur avec `kubectl rollout undo` : il crée un nouveau commit traçable qui restaure l'état désiré précédent. Sa réussite dépend toutefois de la disponibilité de Git, d'ArgoCD et de l'ancienne image dans le registre.

### Scénario 4 — Hook PreSync

**Manipulation.** Un Job Helm annoté `argocd.argoproj.io/hook: PreSync` a été ajouté au chart annuaire. Il exécute une migration simulée avec l'image du service et affiche `migration ok`. La politique `BeforeHookCreation` permet de remplacer l'ancien Job lors d'une sync ultérieure.

**Observation.** L'ajout du hook seul n'a pas déclenché l'auto-sync, car les hooks ne participent pas de la même manière au calcul `OutOfSync` des ressources normales. Une synchronisation manuelle a été lancée. Une première tentative avec `--force` a échoué, car cette option est incompatible avec `ServerSideApply=true`. Sans `--force`, l'opération a réussi en six secondes et ArgoCD a affiché le Job `Succeeded`, avec la phase `PreSync`, avant le Service, le Deployment et l'Ingress. Les logs contenaient `migration ok`.

**Conclusion.** Un hook PreSync convient aux opérations qui doivent réussir avant le déploiement, comme une migration de schéma. Son échec bloque volontairement toute la sync. Il faut donc prévoir des logs, une stratégie de reprise et des migrations rétrocompatibles.

### Scénario 5 — Sync waves

**Manipulation.** Un ConfigMap a été placé dans la wave `-1` et le Deployment dans la wave `0`. Le Deployment portait une annotation `scenario-version` permettant de vérifier si sa wave avait réellement été appliquée. Après une première sync valide sur la valeur `baseline`, le ConfigMap a été rendu immutable. Un commit suivant a demandé simultanément la valeur `blocked` dans ses données et dans l'annotation du Deployment.

**Observation.** Le Job PreSync a d'abord réussi. Kubernetes a ensuite rejeté le ConfigMap avec le message `field is immutable when immutable is set`. L'opération ArgoCD est restée en échec et a effectué des retries. Le ConfigMap réel et le Deployment ont tous les deux conservé la valeur `baseline` : la wave `0` n'a donc pas été appliquée après l'échec de la wave `-1`.

Une première tentative utilisant un label de type numérique avait produit un `ComparisonError` avant toute sync. Elle ne démontrait pas les waves et a été remplacée par le cas du ConfigMap immutable, qui échoue réellement pendant l'application de la wave `-1`.

**Conclusion.** Les sync waves imposent un ordre et empêchent les vagues suivantes de démarrer après un échec. Elles ne remplacent pas les hooks : les hooks déterminent une phase fonctionnelle, tandis que les waves ordonnent les ressources au sein des phases.
