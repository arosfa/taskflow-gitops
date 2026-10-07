# taskflow-gitops — dépôt GitOps du cours CI/CD M2

Ce dépôt décrit **l'état voulu** de l'application TaskFlow dans Kubernetes.
Argo CD le surveille et aligne le cluster dessus : pour changer la production,
on ne tape pas de commande, on fait une **Pull Request**.

## Installation (à faire chez vous, avant le cours)

Prérequis : Docker Desktop démarré, 8 Go de RAM, 10 Go de disque libre.
Sous Windows : WSL2 (Ubuntu) + intégration WSL de Docker Desktop, et toutes les commandes dans WSL.

```bash
git clone https://github.com/9m7fjfpv9k-cyber/taskflow-gitops.git
cd taskflow-gitops
./scripts/install.sh
```

Le script crée un cluster local `kind`, installe Argo CD et Argo Rollouts,
puis télécharge les images des labs. Comptez 5 à 15 minutes.
Il peut être relancé sans risque.

## Structure

| Chemin | Rôle |
| --- | --- |
| `apps/taskflow/` | Les manifests surveillés par Argo CD |
| `argocd/application.yaml` | Déclare l'application dans Argo CD |
| `exemples/bluegreen/` | Manifests pour le déploiement Blue-Green |
| `exemples/canary/` | Manifests pour le déploiement Canary |
| `scripts/install.sh` | Installation de l'environnement |
| `scripts/argocd-ui.sh` | Ouvre l'interface d'Argo CD |
| `scripts/observe.sh` | Montre quelle version répond, et avec quel code HTTP |

## Images disponibles

`ghcr.io/9m7fjfpv9k-cyber/taskflow` en versions `1.0.0`, `1.1.0`, `2.0.0` et `2.1.0`.

## Équipe

- arosfa (seul, sans binôme)

## Labo : déployer par PR, dérive, retour arrière

Auteur : arosfa. Labo fait seul, sans binôme. Commencé en fin de séance le 7 octobre, continué le soir et dans la nuit.
Fork : https://github.com/arosfa/taskflow-gitops

Étapes 1 à 4 réalisées et documentées ci-dessous. Bonus non réalisé.

Environnement : cluster kind `kind-cicd` sur macOS, Argo CD v3.5.3, relecture de Git toutes les 60 s.
Argo CD surveille la branche `main`, dossier `apps/taskflow`, avec `selfHeal` et `prune` activés.

### Journal des déploiements

| Date et heure | Action | PR / commit | Ce que j'ai observé | Qui a agi |
|---|---|---|---|---|
| 7 oct., non relevée | `kubectl apply -f argocd/application.yaml` | - | 1.0.0, 4 pods, Synced + Healthy | moi (déclaration), Argo CD (déploiement) |
| 7 oct., entre 12:18 et 12:31 | Merge de « Passer TaskFlow en image 2.0.0 » | PR #1, commit `fe71d47` | 1.0.0 juste avant | moi, dans Git |
| 7 oct., avant 12:31:28 | Synchronisation automatique | `fe71d47` | 2.0.0 | Argo CD |
| 7 oct., 12:31:28 | Contrôle avec `observe.sh` | - | 40 réponses `version=2.0.0 http=200` | - |
| 8 oct., 01:18:28 | Dérive : `kubectl scale --replicas=1` | aucun commit | 1 pod, puis 4 pods prêts 7 s après | moi (dérive), Argo CD (correction) |
| 8 oct., 01:19:34 | Dérive : `kubectl set image` en 1.1.0 | aucun commit | 1.1.0, puis retour en 2.0.0 avant 01:24:54 | moi (dérive), Argo CD (correction) |
| 8 oct., 01:38 | Merge de « Revert "Passer TaskFlow en image 2.0.0" » | PR de revert, commit `1eafca6` | retour en 1.0.0 constaté à 01:38:16 | moi (dans Git), Argo CD (déploiement) |

Les heures exactes des deux merges sont visibles sur les PR.

### 1. Premier déploiement (1.0.0)

`kubectl apply -f argocd/application.yaml` ne déploie rien lui-même : il déclare l'application à Argo CD.
C'est Argo CD qui lit ensuite le dépôt et crée le namespace, le Deployment et le Service.
Résultat : 4 pods en image 1.0.0, application Synced et Healthy, `observe.sh` : 40 réponses `version=1.0.0 http=200`.

![État initial : application Synced et Healthy, 4 pods](image-1.png)

### 2. Déploiement de la 2.0.0 par PR

La PR #1 change une seule ligne de `apps/taskflow/deployment.yaml` : l'image passe de 1.0.0 à 2.0.0.
Après le merge, aucune commande n'a été lancée sur le cluster. Argo CD a vu le nouveau commit
à sa relecture suivante de Git, puis a remplacé les pods.
Délai : non chronométré précisément. La 2.0.0 était en place au contrôle de 12:31:28.

![Diff de la PR 1](image-2.png)

![observe.sh : 1.0.0 avant le merge, 2.0.0 après](image-3.png)

### 3. Dérive, puis correction par Argo CD

Git demande 4 pods en image 2.0.0. J'ai changé le cluster à la main avec `kubectl`, sans toucher à Git,
pour voir si Argo CD s'en rendait compte.

**Dérive 1 : le nombre de pods (8 oct.)**

| Heure | Ce qui se passe |
|---|---|
| 01:18:28 | `kubectl scale --replicas=1` : le Deployment affiche `1/1` |
| 01:18:31 | `1/4` : Argo CD a remis 4 pods voulus, 3 s après |
| 01:18:35 | `4/4` : tout est revenu, 7 s après ma commande |

![Dérive sur le nombre de pods](image-4.png)

**Dérive 2 : l'image (8 oct.)**

| Heure | Ce qui se passe |
|---|---|
| 01:19:34 | `kubectl set image` : l'image passe en 1.1.0 |
| 01:22:36 | toujours 1.1.0, 3 minutes après |
| 01:24:54 | l'image est revenue en 2.0.0, application Synced et Healthy |

![Dérive sur l'image](image-5.png)

**Mon analyse**

Kubernetes a obéi à mes commandes : c'est l'ouvrier, il fait ce qu'on lui dit. Argo CD, lui, compare
le cluster à Git, comme un chef de chantier qui vérifie que le chantier suit le plan. Quand il voit
un écart, il redonne la consigne de Git.

Les deux fois, je n'ai rien corrigé moi-même et Git n'a reçu aucun commit. Ce que j'avais fait
à la main a disparu.

Ce qui m'a surpris, c'est le délai. Les pods sont revenus en 3 s, mais l'image a mis entre 3 et 5 minutes.
La veille, la même dérive sur l'image avait été corrigée en 2 s. Mon hypothèse, non vérifiée : Argo CD
attend de plus en plus longtemps quand les dérives s'enchaînent. `selfHeal` est donc automatique,
mais pas toujours immédiat.

Ce que je retiens : pour changer le cluster pour de bon, il faut changer Git, par une PR.

### 4. Revert : retour à l'image 1.0.0

J'ai cliqué sur le bouton Revert de la PR #1. GitHub a créé tout seul une nouvelle PR,
« Revert "Passer TaskFlow en image 2.0.0" », qui fait l'inverse de la première : la ligne de l'image
repasse de 2.0.0 à 1.0.0. Je n'ai rien eu à pousser depuis ma machine.

| Heure (8 oct.) | Ce qui se passe |
|---|---|
| juste avant 01:38:04 | merge de la PR de revert (heure exacte sur la PR) |
| 01:38:04 | je lance la surveillance du cluster |
| 01:38:16 | 4 pods prêts sur 4 en image 1.0.0 |

![Diff de la PR de revert : 2.0.0 redevient 1.0.0](image-6.png)

![Terminal : retour en 1.0.0 à 01:38:16](image-7.png)

**Mon analyse**

Cette fois, le changement a tenu, contrairement à la dérive. La différence : j'ai changé Git, le plan,
et pas le cluster directement. Argo CD a lu le nouveau commit et a déployé la 1.0.0 lui-même.

Le retour arrière n'efface rien : l'historique garde la PR #1 (passage en 2.0.0) puis la PR de revert
qui l'annule. On voit qui a fait quoi, et quand.

Le délai a été court, une douzaine de secondes. Argo CD relit Git toutes les 60 s : j'ai sans doute
mergé juste avant une relecture.

### 5. Bonus prune : non réalisé

### Réponses

**Push ou pull ?** Pull. Le merge n'a rien envoyé au cluster, qui tourne en local et que GitHub
ne peut pas joindre. C'est Argo CD, depuis le cluster, qui interroge Git toutes les 60 s et applique
ce qu'il y trouve.

**Qui a corrigé quoi ?** Pour le déploiement, j'ai corrigé Git (PR #1) et Argo CD a aligné le cluster.
La dérive a été corrigée par Argo CD seul, grâce à `selfHeal` : je n'ai rien fait et Git n'a reçu aucun commit.
Pour le retour arrière, j'ai corrigé Git avec la PR de revert, puis Argo CD a remis la 1.0.0 dans le cluster.

**Pourquoi git revert ?** Parce que Git est la source de vérité. La dérive me l'a montré : un changement fait
à la main avec `kubectl` est annulé par Argo CD. Pour revenir en 1.0.0 pour de bon, il fallait changer Git.
Le revert le fait proprement : il ajoute un commit qui annule le précédent, sans réécrire l'historique,
et il passe par une PR comme tout le reste.

### Ce qui m'a posé problème

- Le script d'installation s'arrêtait sur Argo Rollouts (« annotations: Too long »). Corrigé en ajoutant
  `--server-side --force-conflicts` au `kubectl apply` de Rollouts.
- Ma première PR partait vers le dépôt du prof au lieu de mon fork : GitHub propose le dépôt d'origine
  par défaut. Il faut changer le « base repository ».
- J'ai committé mes captures sans avoir enregistré le README : la PR ne contenait que les images.
- Le cluster ne répondait plus (« connection refused ») : Docker Desktop s'était arrêté.
  C'est là que j'ai compris que le cluster kind vit dans Docker.
- Pour le revert, je cherchais comment pousser depuis mon terminal. En fait, le bouton Revert fait tout
  sur GitHub : il crée la branche, le commit et la PR.
- Au début, je mélangeais les rôles de Git, Kubernetes et Argo CD. L'image du chantier m'a débloqué.

### Ce que je retiens

Pendant tout le labo, je n'ai lancé aucune commande de déploiement. J'ai changé Git, et Argo CD a fait le reste.
Les seules fois où j'ai touché au cluster directement, ça n'a pas tenu.