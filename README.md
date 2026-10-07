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

<!-- Noms du binôme -->
- arosfa (seul, sans binôme)

## Labo : déployer par PR, dérive, retour arrière

Auteur : arosfa. Labo réalisé seul, sans binôme, en fin de séance.
Fork : https://github.com/arosfa/taskflow-gitops

Étapes 1 et 2 réalisées et documentées ci-dessous. Étapes 3 à 5 non réalisées faute de temps.

Environnement : cluster kind `kind-cicd` sur macOS, Argo CD v3.5.3, relecture de Git toutes les 60 s.
Argo CD surveille la branche `main`, dossier `apps/taskflow`, avec `selfHeal` et `prune` activés.

### Journal des déploiements

| Heure | Action | PR / commit | Version observée | Qui a agi |
|---|---|---|---|---|
| non relevée | `kubectl apply -f argocd/application.yaml` | - | 1.0.0, 4 pods, Synced + Healthy | moi (déclaration), Argo CD (déploiement) |
| entre 12:18 et 12:31 | Merge de « Passer TaskFlow en image 2.0.0 » | PR #1, commit `fe71d47` | 1.0.0 juste avant | moi, dans Git |
| avant 12:31:28 | Synchronisation automatique | `fe71d47` | 2.0.0 | Argo CD |
| 12:31:28 | Contrôle avec `observe.sh` | - | 40 réponses `version=2.0.0 http=200` | - |

L'heure exacte du merge est visible sur la PR #1.

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

### 3. Dérive : non réalisée faute de temps

Comportement attendu d'après la configuration, non observé : avec `selfHeal: true`, Argo CD devrait
annuler un `kubectl scale` ou un `kubectl set image` manuel et ramener le cluster à l'état de Git.

### 4. Revert : non réalisé faute de temps

### 5. Bonus prune : non réalisé faute de temps

### Réponses

**Push ou pull ?** Pull. Le merge n'a rien envoyé au cluster, qui tourne en local et que GitHub
ne peut pas joindre. C'est Argo CD, depuis le cluster, qui interroge Git toutes les 60 s et applique
ce qu'il y trouve.

**Qui a corrigé quoi ?** Pour le déploiement, j'ai corrigé Git (PR #1) et Argo CD a aligné le cluster.
La dérive n'a pas été testée. Avec `selfHeal: true`, c'est Argo CD seul qui devrait la corriger,
sans commit dans Git.

**Pourquoi git revert ?** Non testé. Git étant la source de vérité, un retour arrière fait avec kubectl
serait une dérive qu'Argo CD annulerait. Le revert ajoute un commit qui annule le précédent : l'historique
reste lisible et le changement passe par une PR.
