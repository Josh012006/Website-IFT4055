---
title: Suivi du projet
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Suivi de projet

## Semaine 1 (4–9 mai 2026)

### Objectifs de la période
- Prendre contact avec l'équipe et configurer l'environnement de travail
- Lancer un premier entraînement RL sur le cluster Mila
- Identifier et résoudre les problèmes d'infrastructure

### Travail réalisé

!!! abstract "Avancement"
    - [x] Prise de contact avec l'équipe, accès au Slack du laboratoire
    - [x] Lecture de la documentation RLinf (LIBERO)
    - [x] Connexion au cluster Mila et résolution des problèmes d'accès avec l'aide du support informatique
    - [x] Clonage du dépôt et installation des dépendances sur un nœud de calcul interactif (`salloc --no-root`)
    - [x] Rédaction du script `job.sh` et soumission via `sbatch`
    - [x] Résolution progressive des erreurs : mauvaise clé de normalisation, mémoire CPU insuffisante, mémoire GPU insuffisante, timeout NCCL multi-GPU
    - [x] Passage au GPU A100L (80GB) avec `enable_offload: True` et `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
    - [x] Premier entraînement fonctionnel jusqu'à 132 steps
    - [x] Migration du projet vers `$SCRATCH`, lien symbolique depuis `$HOME`, script de nettoyage automatique des checkpoints
    - [x] Entraînement complet de 1000 steps (31h40)
    - [x] Versionnement du script et de la config sur GitHub
    - [ ] Augmentation du batch size — reportée à la semaine suivante

### Décisions et ajustements

!!! info "Décisions"
    - Abandon du GPU L40S (44GB insuffisant) au profit de l'A100L (80GB)
    - Passage à un seul GPU avec `enable_offload: True` au lieu de multi-GPU (problèmes NCCL)
    - Réduction du scope : passage de LIBERO-90 (90 tâches simultanées) à une configuration plus ciblée
    - Ajout d'un script de nettoyage des checkpoints pour éviter de saturer le stockage

### Difficultés rencontrées

!!! warning "Difficultés"
    - Le modèle est chargé deux fois en mémoire GPU (RolloutGroup + ActorGroup), ce qui double l'empreinte mémoire et cause des erreurs OOM
    - Résolu par `enable_offload: True` et `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
    - Checkpoint corrompu à l'étape 50 : le dossier `$HOME` (96GB) était insuffisant pour stocker les sauvegardes (~50GB chacune)
    - Résolu en déplaçant le projet vers `$SCRATCH` et en ajoutant le script de nettoyage
    - Conflit Ray sur le nœud assigné : un cluster Ray résiduel d'un autre utilisateur empêchait l'initialisation
    - Résolu en excluant le nœud contaminé via `#SBATCH --exclude`
    - Résultats de l'entraînement à 1000 steps : pics de récompense occasionnels (~0.13) sans tendance stable — batch size trop petit pour extraire un signal d'apprentissage robuste
        ![LIBERO-90 GRPO Training Results Success Once]({{ config.site_url }}/images/libero90_grpo_training1_success_once.png)
        ![LIBERO-90 GRPO Training Results Approx KL]({{ config.site_url }}/images/libero90_grpo_training1_approx_kl.png)


## Semaine 2 (11–15 mai 2026)

### Objectifs de la période
- Analyser les résultats de l'entraînement précédent et planifier la suite
- Changer d'environnement et de modèle selon les recommandations du superviseur
- Lancer un premier entraînement avec le nouveau setup

### Travail réalisé

!!! abstract "Avancement"
    - [x] Discussion avec l'équipe sur les résultats de la semaine 1 et les prochaines étapes
    - [x] Passage à l'environnement RoboCasa avec le modèle π0 (~3B paramètres) en PPO
    - [x] Installation des dépendances RoboCasa et téléchargement des assets de cuisine (~5GB)
    - [ ] Premier entraînement fonctionnel avec π0 — bloqué par un problème d'initialisation Ray

### Décisions et ajustements

!!! info "Décisions"
    - Abandon d'OpenVLA-OFT comme modèle principal au profit de π0 (~3B), plus adapté à la recherche
    - Abandon de LIBERO comme benchmark principal (trop saturé) au profit de RoboCasa
    - LoRA sera activé pour réduire l'empreinte mémoire lors du prochain entraînement

### Difficultés rencontrées

!!! warning "Difficultés"
    - Les pics de récompense observés la semaine précédente s'expliquent par un problème d'exploration, pas de batch size — l'accumulation de gradients n'a pas d'effet sur la dynamique d'apprentissage
    - Ray hang au démarrage du job : le processus Ray s'initialise puis bloque sans produire d'output. Cause en cours d'investigation.


## Semaine 3 (18–22 mai 2026)

### Objectifs de la période
- Lancer un entraînement fonctionnel de π0 sur RoboCasa
- Démarrer un entraînement LIBERO-Spatial avec OpenVLA-OFT
- Documenter et partager les expériences avec l'équipe

### Travail réalisé

!!! abstract "Avancement"
    - [x] Activation de LoRA (rank 32) pour π0 et tuning de la config pour un seul A100L (80GB)
    - [x] Premier run fonctionnel π0 + RoboCasa CloseDrawer (PPO) — 24 steps, `success_once` atteignant 0.8125 au step 20
    - [x] Résolution du conflit de versions `robosuite` entre LIBERO et RoboCasa — création de deux venvs séparés (`.venv-robocasa` et `.venv-libero`)
    - [x] Mise à jour de tous les scripts `job.sh` pour pointer vers le bon venv
    - [x] Ouverture d'une PR sur `montrealrobotics/RLinf` avec les configs, scripts et `EXPERIMENTS_README.md`
    - [x] Démarrage du run LIBERO-Spatial GRPO — `success_once` oscillant entre 0.22 et 0.875 sur 38 steps
    - [x] Run π0 + RoboCasa CloseDrawer complété à 150 steps — métriques finales : `success_once` ~0.82, `critic/value_loss` ~0.005, `approx_kl` ~0.036
        ![RoboCasa CloseDrawer PPO Training Results Success Once]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_success_once.png)
        ![RoboCasa CloseDrawer PPO Training Results Approx KL]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_approx_kl.png)
        ![RoboCasa CloseDrawer PPO Training Results Value Loss]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_value_loss.png)
    - [x] Visualisation des résultats via TensorBoard, sauvegarde des vidéos d'épisodes

        <div style="text-align: center">
            <video width="480" controls poster="{{ config.site_url }}/images/robocasa_close_drawer_thumbnail.png">
                <source src="{{ config.site_url }}/images/robocasa_close_drawer_training1_ppo_step_150.mp4" type="video/mp4">
            </video>
        </div>

        *π0 + RoboCasa CloseDrawer — épisode au step 150 (après entraînement PPO)*

    - [x] Upload des poids finaux sur HuggingFace (`josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO`)
    - [x] Partage des résultats et vidéos avec le groupe via Slack
    - [ ] Run LIBERO-Spatial toujours en cours

### Décisions et ajustements

!!! info "Décisions"
    - LIBERO et RoboCasa nécessitent des venvs séparés en raison de conflits de dépendances — cette contrainte est documentée dans le README
    - RoboCasa CloseDrawer s'avère trop facile comme tâche de départ : le modèle SFT de base atteint déjà 0.6–0.7 dès le step 1, laissant peu de place à l'amélioration par RL
    - LIBERO-Long identifié comme prochaine tâche cible — plus difficile pour le SFT (11.9% de base), meilleur candidat pour démontrer l'apport du RL
    - `unset RAY_ADDRESS` ajouté aux scripts de job pour éviter les conflits avec des clusters Ray résiduels d'autres utilisateurs

### Difficultés rencontrées

!!! warning "Difficultés"
    - Job crashé au step 24 avec `FileNotFoundError: omron_mobile_base.xml` — assets RoboCasa manquants après réinstallation du venv. Résolu en téléchargeant les assets séparément avec `python -m robocasa.scripts.download_kitchen_assets`
    - Conflits de versions Ray récurrents sur plusieurs nœuds (`cn-g008`, `cn-g012`, `cn-g026`) — clusters résiduels d'autres utilisateurs. Résolu par `unset RAY_ADDRESS` et exclusion des nœuds contaminés
    - Run LIBERO-Spatial préempté après ~19h — reprise depuis le dernier checkpoint


## Semaine 4 (25–29 mai 2026)

### Objectifs de la période
- Analyser les résultats de LIBERO-Spatial et planifier la suite
- Lancer l'entraînement LIBERO-Long
- Approfondir la revue de littérature

### Travail réalisé

!!! abstract "Avancement"
    - [x] Analyse des résultats du run LIBERO-Spatial — performances insuffisantes, suspicion d'un learning rate inadapté après réduction du nombre d'epochs
        ![LIBERO-Spatial GRPO Training Results Success Once]({{ config.site_url }}/images/libero_spatial_grpo_training1_success_once.png)
        ![LIBERO-Spatial GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_spatial_grpo_training1_approx_kl.png)
    - [x] Lecture complète du papier de recherche OpenVLA-OFT — confirme que la configuration A100 + LoRA est celle recommandée par les auteurs eux-mêmes
    - [x] Revue exhaustive du dépôt et de la documentation RLinf — inventaire des tâches disponibles par benchmark avec leurs modèles SFT de base respectifs
    - [x] Configuration et lancement du run LIBERO-Long (150 steps) avec OpenVLA-OFT + GRPO
    - [x] Run préempté au step 145 — reprise depuis le dernier checkpoint, fin imminente
    - [ ] Analyse des résultats LIBERO-Long — en attente de la fin du run

### Décisions et ajustements

!!! info "Décisions"
    - LIBERO-Long priorisé comme tâche principale : le modèle SFT de base n'atteint que 11.9% de succès, ce qui laisse une large marge d'amélioration pour le RL
    - Learning rate à ajuster pour les prochains runs suite aux instabilités observées sur LIBERO-Spatial

### Difficultés rencontrées

!!! warning "Difficultés"
    - Run LIBERO-Long préempté au step 145 avant la fin des 150 steps prévus — repris depuis le checkpoint disponible


## Semaine 5 (01–05 juin 2026)

### Objectifs de la période
- Visualiser les datasets d'entraînement des VLAs pour analyser la distribution des endpoints et identifier des tâches hors distribution
- Étudier dans quelle mesure on peut maintenir de bonnes performances tout en réduisant le temps d'entraînement via les hyperparamètres
- Réfléchir conceptuellement à comment orienter le RL pour atteindre notre objectif de généralisation hors distribution
- Lire les papiers de recherche pertinents sur le sujet
- Analyser les résultats du run LIBERO-Long

### Travail réalisé

!!! abstract "Avancement"
    - [x] Discussion avec l'équipe sur le vrai objectif du projet : démontrer que le RL permet une généralisation hors distribution au-delà de ce que le SFT permet d'atteindre, et non simplement améliorer les performances sur des tâches déjà maîtrisées par le SFT
    - [x] Visualisation des datasets et analyse de la distribution des endpoints
        - Création d'un notebook Jupyter pour la visualisation d'ensembles de données et l'analyse de la répartition des points d'extrémité
        - [Notebook](https://colab.research.google.com/github/Josh012006/OpenX-Embodiment-Datasets-Visualization/blob/main/colabs/Open_X_Embodiment_Datasets.ipynb)
        - [Répertoire](https://github.com/Josh012006/OpenX-Embodiment-Datasets-Visualization)
    - [x] Analyse des résultats du run LIBERO-Long : comme indiqué dans la documentation RLinf, LIBERO-Long est la suite la plus difficile avec un taux de base SFT de seulement 11.9%. L'entraînement s'est déroulé sur 4 sessions reprises totalisant ~150 steps (~37 heures). `success_once` a oscillé entre 0.06 et 0.19 sans tendance à la hausse, restant proche du niveau SFT de base. `approx_kl` est resté stable entre 0.035 et 0.048 tout au long des sessions, indiquant que les mises à jour de politique étaient contrôlées — le manque d'amélioration est donc davantage lié à un problème d'exploration qu'à une instabilité d'optimisation.
        ![LIBERO-Long GRPO Training Results Success Once]({{ config.site_url }}/images/libero_long_grpo_training1_success_once.png)
        ![LIBERO-Long GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_long_grpo_training1_approx_kl.png)
    - [x] Exploration des hyperparamètres pour réduire le temps d'entraînement : l'objectif est de voir si l'augmentation du replay ratio de manière appropriée peut réduire le temps d'entraînement tout en maintenant de bonnes performances
        - Relancement d'un run LIBERO-Spatial GRPO (OpenVLA-OFT) avec `lr=1.0e-6`, `rollout_epoch=8` (au lieu de 4) et moins d'epochs (`max_epochs=100` au lieu de 150)
        - Relancement d'un run RoboCasa CloseDrawer PPO (π0) avec `lr=1.0e-6` et `update_epoch=4` (au lieu de 2)
    - [ ] Lecture de papiers de recherche pertinents
    - [ ] Réflexion conceptuelle sur les leviers RL pour la généralisation hors distribution

### Décisions et ajustements

!!! info "Décisions"
    - Décision de se concentrer sur des tâches précises à horizon court pour la phase initiale du projet, plutôt que sur des tâches longues à multiples sous-objectifs.



## Semaine 6 (8–12 juin 2026)

### Objectifs de la période
- Analyser les distributions d'endpoints et identifier des tâches hors distribution
- Continuer à étudier dans quelle mesure on peut maintenir de bonnes performances tout en réduisant le temps d'entraînement
- Réfléchir conceptuellement à comment orienter le RL pour atteindre notre objectif de généralisation hors distribution
- Lire les papiers de recherche pertinents sur le sujet

### Travail réalisé

!!! abstract "Avancement"
    - [ ] Discussion avec l'équipe sur l'analyse de visualisation
    - [ ] Identification des tâches hors distribution à partir de l'analyse de distribution des endpoints
    - [ ] Construction d'un dataset OOD pour nos futurs travaux sur la généralisation hors distribution
    - [ ] Analyse de l'exploration des hyperparamètres pour la réduction du temps d'entraînement
    - [ ] Fin de la lecture des papiers de recherche pertinents

<!-- ### Décisions et ajustements

!!! info "Décisions"

### Difficultés rencontrées

!!! warning "Difficultés" -->