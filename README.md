## Labo — déployer par PR, dérive, retour arrière

Équipe : arosfa. Fork : https://github.com/arosfa/taskflow-gitops

Environnement : cluster kind `kind-cicd` sur macOS, Argo CD v3.5.3, relecture de Git toutes les 60 s.
Argo CD surveille la branche `main`, dossier `apps/taskflow`, avec `selfHeal` et `prune` activés.
Ruleset sur `main` : pull request obligatoire.

### Journal des déploiements

| Heure | Action | PR / commit | Version observée | Qui a agi |

 `kubectl apply -f argocd/application.yaml` | - | 1.0.0, 4 pods, Synced + Healthy | moi (déclaration), Argo CD (déploiement) |
Merge de « Passer TaskFlow en image 2.0.0 » | PR #1, commit `fe71d47` | encore 1.0.0 | moi, dans Git |
Synchronisation automatique | `fe71d47` | 2.0.0 | Argo CD |
| 12:31:28 | Contrôle avec `observe.sh` | - | 40 réponses `version=2.0.0 http=200` | - |

### 1. Premier déploiement (1.0.0)

`kubectl apply -f argocd/application.yaml` ne déploie rien lui-même : il déclare l'application à Argo CD.
C'est Argo CD qui lit ensuite le dépôt et crée le namespace, le Deployment et le Service.
Résultat : 4 pods en image 1.0.0, application Synced et Healthy, `observe.sh` : 40 réponses `version=1.0.0 http=200`.

![État initial]
![alt text](image-1.png)

### 2. Déploiement de la 2.0.0 par PR

La PR #1 change une seule ligne de `apps/taskflow/deployment.yaml` : l'image passe de 1.0.0 à 2.0.0.
Après le merge, aucune commande n'a été lancée sur le cluster. Argo CD a vu le nouveau commit
à sa relecture suivante de Git, puis a remplacé les pods.
Délai mesuré entre le merge et la 2.0.0 : __ s.

![Diff de la PR 1]
![alt text](image-2.png)

![observe.sh en 2.0.0]
![alt text](image-3.png)



### 3. Dérive - à compléter

### 4. Revert - à compléter

### 5. Bonus prune - à compléter

### Réponses

**Push ou pull ?** Pull. Le merge n'a rien envoyé au cluster, qui tourne en local et que GitHub
ne peut pas joindre. C'est Argo CD, depuis le cluster, qui interroge Git toutes les 60 s et applique
ce qu'il y trouve. Le délai observé après le merge correspond à cette attente.

**Qui a corrigé quoi ?** Pour le déploiement, j'ai corrigé Git (PR #1) et Argo CD a aligné le cluster.
Dérive : à compléter.

**Pourquoi git revert ?** À compléter après l'étape 4.