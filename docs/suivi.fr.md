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

## Semaine 1 (5–9 mai 2026)

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
