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
- Prendre contact avec l'équipe et mettre en place l'environnement de travail
- Lancer un premier entraînement RL sur le cluster Mila
- Identifier et résoudre les problèmes d'infrastructure

### Travail réalisé

!!! abstract "Avancement"
    - [x] Prise de contact avec l'équipe, ajout au Slack du laboratoire
    - [x] Lecture de la documentation RLinf (LIBERO)
    - [x] Connexion au cluster Mila et résolution des problèmes d'accès avec le support informatique
    - [x] Clonage du dépôt et installation des dépendances sur un nœud de calcul interactif
    - [x] Rédaction du script `job.sh` et soumission via `sbatch`
    - [x] Résolution progressive des erreurs d'infrastructure (clé de normalisation, RAM CPU/GPU, timeout NCCL)
    - [x] Premier entraînement réussi atteignant 132 steps
    - [x] Migration du projet vers `$SCRATCH`, création d'un lien symbolique depuis `$HOME`, ajout d'un script de nettoyage automatique des checkpoints
    - [x] Entraînement complet de 1000 steps terminé (31h40)
    - [x] Versionnement du script de job et de la config sur GitHub
    - [ ] Augmentation du batch size — reportée à la semaine suivante

### Décisions et ajustements

!!! info "Décisions"
    - Abandon du GPU L40S (44GB insuffisant) au profit du A100L (80GB)
    - Passage à une configuration mono-GPU avec `enable_offload: True` plutôt que multi-GPU (problèmes NCCL)
    - Ajout d'un script de nettoyage des checkpoints pour éviter de saturer le stockage

### Difficultés rencontrées

!!! warning "Difficultés"
    - Erreurs OOM dues au modèle chargé deux fois en mémoire GPU — résolu avec `enable_offload: True`
    - Checkpoint corrompu au step 50 à cause de la limite de taille de `$HOME` — résolu en migrant vers `$SCRATCH`
    - Conflit Ray sur le nœud assigné — résolu en excluant les nœuds contaminés via `#SBATCH --exclude`
    - Les résultats à 1000 steps montraient des pics de reward occasionnels sans tendance d'apprentissage stable — hypothèse du batch size envisagée initialement, révisée par la suite (voir Semaine 2)


## Semaine 2 (11–15 mai 2026)

### Objectifs de la période
- Analyser les résultats de l'entraînement précédent et planifier les prochaines étapes
- Changer d'environnement et de modèle selon les recommandations du superviseur
- Lancer un premier entraînement avec la nouvelle configuration

### Travail réalisé

!!! abstract "Avancement"
    - [x] Discussion avec l'équipe sur les résultats de la Semaine 1 et les prochaines étapes
    - [x] Passage à l'environnement RoboCasa avec le modèle π₀ (~3B paramètres) en PPO
    - [x] Installation des dépendances RoboCasa et téléchargement des assets de cuisine (~5GB)
    - [ ] Premier entraînement réussi avec π₀ — bloqué par un blocage à l'initialisation de Ray

### Décisions et ajustements

!!! info "Décisions"
    - Abandon d'OpenVLA-OFT comme modèle principal au profit de π₀ (~3B paramètres), mieux adapté à la recherche
    - Abandon de LIBERO comme benchmark principal (majoritairement saturé) au profit de RoboCasa
    - LoRA sera activé pour réduire l'empreinte mémoire du prochain entraînement

### Difficultés rencontrées

!!! warning "Difficultés"
    - Les pics de reward de la Semaine 1 s'expliquent par un problème d'exploration, pas par le batch size — l'accumulation de gradient n'a aucun effet sur la dynamique d'apprentissage
    - Blocage de Ray au démarrage du job : le processus Ray s'initialise puis se bloque sans plus de sortie — cause encore en investigation


## Semaine 3 (18–22 mai 2026)

### Objectifs de la période
- Lancer un entraînement π₀ réussi sur RoboCasa
- Démarrer un entraînement LIBERO-Spatial avec OpenVLA-OFT
- Documenter et partager les expériences avec l'équipe

### Travail réalisé

!!! abstract "Avancement"
    - [x] Activation de LoRA (rank 32) pour π₀ et ajustement de la config pour un seul A100L (80GB)
    - [x] Premier entraînement réussi π₀ + RoboCasa CloseDrawer (PPO) — 24 steps, `success_once` atteignant 0.8125 au step 20
    - [x] Résolution du conflit de version `robosuite` entre LIBERO et RoboCasa — création de deux venvs séparés
    - [x] Mise à jour de tous les scripts de job pour pointer vers le bon venv
    - [x] Ouverture d'une PR sur `montrealrobotics/RLinf` avec les configs, scripts de job et `EXPERIMENTS_README.md`
    - [x] Démarrage de l'entraînement LIBERO-Spatial GRPO — `success_once` oscillant entre 0.22 et 0.875 sur 38 steps
    - [x] Entraînement π₀ + RoboCasa CloseDrawer terminé à 150 steps
    - [x] Visualisation des résultats via TensorBoard, sauvegarde des vidéos d'épisodes
    - [x] Upload des poids finaux sur HuggingFace (`josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO`)
    - [x] Partage des résultats et vidéos avec le groupe via Slack
    - [ ] Entraînement LIBERO-Spatial toujours en cours

### Décisions et ajustements

!!! info "Décisions"
    - LIBERO et RoboCasa nécessitent des venvs séparés à cause de conflits de dépendances — documenté dans le README
    - RoboCasa CloseDrawer est trop facile comme tâche de départ (base SFT déjà ~0.6–0.7) — marge limitée pour la contribution du RL
    - LIBERO-Long identifiée comme prochaine tâche cible — taux de base SFT plus difficile (11.9%), meilleure candidate pour démontrer les gains du RL
    - Ajout de `unset RAY_ADDRESS` dans les scripts de job pour éviter les conflits de clusters Ray résiduels

### Difficultés rencontrées

!!! warning "Difficultés"
    - Job planté au step 24 avec `FileNotFoundError: omron_mobile_base.xml` — résolu en téléchargeant séparément les assets RoboCasa
    - Conflits de version Ray récurrents sur plusieurs nœuds — résolu avec `unset RAY_ADDRESS` et exclusion de nœuds
    - Entraînement LIBERO-Spatial préempté après ~19h — repris depuis le dernier checkpoint


## Semaine 4 (25–29 mai 2026)

### Objectifs de la période
- Analyser les résultats de LIBERO-Spatial et planifier les prochaines étapes
- Lancer l'entraînement LIBERO-Long
- Approfondir la revue de littérature

### Travail réalisé

!!! abstract "Avancement"
    - [x] Analyse des résultats de l'entraînement LIBERO-Spatial — performance insuffisante, learning rate suspecté non adapté après la réduction du nombre d'epochs
    - [x] Lecture complète du papier de recherche OpenVLA-OFT
    - [x] Revue exhaustive du dépôt et de la documentation RLinf
    - [x] Configuration et lancement de l'entraînement LIBERO-Long (150 steps) avec OpenVLA-OFT + GRPO
    - [x] Entraînement préempté au step 145 — repris depuis le dernier checkpoint, fin imminente
    - [ ] Analyse des résultats LIBERO-Long — en attente de la fin de l'entraînement

### Décisions et ajustements

!!! info "Décisions"
    - LIBERO-Long priorisée comme tâche principale : large marge d'amélioration RL par rapport au taux de base SFT de 11.9%
    - Learning rate à ajuster pour les prochains entraînements suite aux instabilités observées sur LIBERO-Spatial

### Difficultés rencontrées

!!! warning "Difficultés"
    - Entraînement LIBERO-Long préempté au step 145 — repris depuis le checkpoint disponible


## Semaine 5 (1–5 juin 2026)

### Objectifs de la période
- Visualiser les datasets d'entraînement VLA pour analyser les distributions des endpoints et identifier les tâches OOD
- Étudier comment réduire le temps d'entraînement via l'ajustement des hyperparamètres
- Réfléchir conceptuellement à comment guider le RL vers notre objectif de généralisation OOD
- Lire les papiers de recherche pertinents
- Analyser les résultats de LIBERO-Long

### Travail réalisé

!!! abstract "Avancement"
    - [x] Discussion avec l'équipe sur le véritable objectif du projet : démontrer que le RL permet une généralisation OOD au-delà de ce que le SFT peut atteindre
    - [x] Visualisation des datasets et analyse de la distribution des endpoints — notebook Jupyter créé ([Colab](https://colab.research.google.com/github/Josh012006/OpenX-Embodiment-Datasets-Visualization/blob/main/colabs/Open_X_Embodiment_Datasets.ipynb), [Dépôt](https://github.com/Josh012006/OpenX-Embodiment-Datasets-Visualization))
    - [x] Résultats de LIBERO-Long analysés : `success_once` a oscillé entre 0.06 et 0.19 sans tendance à la hausse — bottleneck d'exploration confirmé (pas une instabilité d'optimisation, `approx_kl` étant resté stable autour de 0.035–0.048)
    - [x] Exploration d'hyperparamètres pour réduire le temps d'entraînement : test de l'augmentation du replay ratio (`rollout_epoch` 4→8 pour GRPO, `update_epoch` 2→4 pour PPO)
    - [x] Lecture des papiers de recherche pertinents

### Décisions et ajustements

!!! info "Décisions"
    - Décision de se concentrer sur des tâches courtes et précises pour la phase initiale, plutôt que sur des tâches à long horizon avec plusieurs sous-objectifs

### Difficultés rencontrées

!!! warning "Difficultés"
    - L'augmentation du replay ratio a déplacé le bottleneck de la collecte de rollouts vers les mises à jour de gradient — effet net : entraînement plus lent (~27 min/step vs ~14 min/step). Changement annulé.


## Semaine 6 (8–12 juin 2026)

### Objectifs de la période
- Analyser les distributions des endpoints et identifier les tâches OOD
- Poursuivre l'exploration d'hyperparamètres pour réduire le temps d'entraînement
- Réfléchir conceptuellement aux leviers du RL pour la généralisation OOD
- Lire les papiers de recherche pertinents

### Travail réalisé

!!! abstract "Avancement"
    - [x] Discussion avec l'équipe sur l'analyse de visualisation
    - [x] Identification des tâches OOD à partir de l'analyse de distribution des endpoints
    - [x] Sélection de ManiSkill comme benchmark OOD
    - [x] Analyse des résultats de l'exploration d'hyperparamètres — voir Décisions ci-dessous
    - [ ] Terminer la lecture des papiers de recherche pertinents

### Décisions et ajustements

!!! info "Décisions"
    - Augmenter le replay ratio ne réduit pas le temps d'entraînement sur de gros modèles — les mises à jour de gradient sont le bottleneck. Retour aux valeurs d'origine.
    - Stratégie efficace pour réduire le temps d'entraînement : réduire directement `max_episode_steps` et `max_epochs`
    - Changement d'orientation de la recherche : au lieu d'OpenVLA-OFT, étude du fine-tuning RL du OpenVLA de base sur la tâche ManiSkill `StackCube-v1`


## Semaine 7 (15–19 juin 2026)

### Objectifs de la période
- Affiner la définition de l'OOD pour ce projet de recherche
- Apprendre à ajouter de nouvelles tâches au framework RLinf
- Analyser la performance RL sur la tâche StackCube

### Travail réalisé

!!! abstract "Avancement"
    - [x] Intégration de StackCube dans le pipeline RLinf
    - [x] Documentation du processus d'ajout d'une nouvelle tâche à RLinf (`ADDING_TASKS.md`)
    - [x] Lancement d'entraînements GRPO et PPO avec le OpenVLA de base sur StackCube-v1
    - [x] Discussion avec l'équipe sur les résultats des entraînements

### Décisions et ajustements

!!! info "Décisions"
    - Définition de l'OOD gardée simple : une tâche difficile sur laquelle le modèle ne performe pas bien
    - Le RL on-policy (PPO/GRPO) directement à partir du modèle de base est insuffisant sans SFT préalable — taux de succès de 0% confirmé
    - Pivot vers la reproduction du [projet ExpoFT](https://pd-perry.github.io/expo-ft/) comme nouvelle direction de recherche

### Difficultés rencontrées

!!! warning "Difficultés"
    - Les entraînements PPO et GRPO sur `StackCube-v1` ont produit 0% de succès — sans warmup SFT, l'exploration on-policy ne peut pas s'amorcer


## Semaine 8 (22–26 juin 2026)

### Objectifs de la période
- Lire le papier ExpoFT et comprendre l'architecture de la residual policy
- Reproduire les résultats d'ExpoFT dans leur codebase originale sur StackCube
- Intégrer l'architecture de residual policy dans le framework RLinf
- Lire davantage sur les algorithmes off-policy

### Travail réalisé

!!! abstract "Avancement"
    - [x] Lecture du papier ExpoFT et compréhension de l'architecture de la residual policy
    - [x] Reproduction des résultats d'ExpoFT dans la codebase originale sur StackCube
    - [ ] Intégration de l'architecture de residual policy dans le framework RLinf
    - [ ] Lancement d'entraînements avec l'architecture de residual policy sur StackCube
    - [x] Discussion avec l'équipe sur les résultats des entraînements
    - [x] Début de l'adaptation de la codebase originale d'ExpoFT (real-robot / DROID) vers un nouveau dépôt autonome (`expo-ft`) piloté entièrement par la simulation ManiSkill
    - [x] Conversion des premières démonstrations de motion-planning sur StackCube-v1 vers les formats DROID/LeRobot attendus par le pipeline d'entraînement d'ExpoFT
    - [x] Lancement d'un premier entraînement de warmup SFT sur les démonstrations converties

### Décisions et ajustements

!!! info "Décisions"
    - ExpoFT sera d'abord reproduit dans un dépôt dédié basé sur ManiSkill plutôt que dans RLinf, afin de garder le code de l'algorithme original aussi intact que possible en ne remplaçant que la couche environnement
    - ExpoFT complet (warmup SFT + démonstrations + RL off-policy) sera tenté en premier, avec un retrait progressif des composants par la suite pour identifier la supervision minimale nécessaire

### Difficultés rencontrées

!!! warning "Difficultés"
    - Le premier entraînement de warmup SFT a montré un taux de succès proche de 0%, le bras se contentant d'approcher les cubes sans compléter l'empilement — encore en investigation pour déterminer si cela reflète un vrai plafond de difficulté de la tâche StackCube ou un problème de configuration ailleurs dans le pipeline nouvellement adapté
    - Face à ces premiers résultats décevants, début de réflexion sur le fait que StackCube pourrait simplement être une tâche de départ trop difficile pour cette approche, et si une tâche ManiSkill plus facile devrait être essayée à la place


## Semaine 9 (29 juin – 3 juillet 2026)

### Objectifs de la période
- Déterminer si les mauvais résultats SFT sur StackCube reflètent une vraie difficulté de tâche ou un problème de configuration
- Obtenir une baseline SFT pleinement validée et fiable avant de toucher à l'étape RL
- Construire l'outillage d'évaluation nécessaire pour faire confiance aux résultats futurs
- Explorer si d'autres tâches ManiSkill peuvent diversifier la suite de tâches OOD

### Travail réalisé

!!! abstract "Avancement"
    - [x] Traçage des résultats SFT proches de 0% jusqu'à un problème fondamental d'espace d'action : le pré-entraînement DROID de π₀.₅ attend des actions en position articulaire en unités physiques, alors que le pipeline utilisait le contrôle cartésien `pd_ee_delta_pose` de ManiSkill avec normalisation — causant l'écrêtage (clipping) de la majorité des actions de démonstration
    - [x] Passage du pipeline complet au contrôle en espace articulaire de bout en bout (`pd_joint_delta_pos`, checkpoint `pi05_droid_jointpos`, état `observation/joint_position`) — la validation par replay est immédiatement passée d'une petite minorité d'actions dans la plage valide à la grande majorité
    - [x] Relance du SFT avec l'espace d'action corrigé — résultats nettement plus prometteurs, mais plusieurs incohérences restantes signalées à approfondir avant de faire confiance aux chiffres
    - [x] Découverte et correction d'un ensemble de bugs silencieux supplémentaires lors d'une revue systématique pré-SFT (« MAJOR RESET ») : un script d'évaluation écrasant sans condition les normalisation stats officielles de DROID, y compris pour l'évaluation de la baseline ; un reliquat de rescaling d'action en espace cartésien issu du mode de contrôle abandonné corrompant une part importante des actions désormais en espace articulaire ; un désalignement de longueur de troncature entre l'environnement et la boucle d'évaluation ; et un mismatch de backend de simulation empêchant la bonne conversion du mode de contrôle
    - [x] Construction de `validate_demos_full_pipeline.py`, un outil de replay de bout en bout rigoureux (vraies normalisation stats, vrai wrapper d'environnement, seeds exactement enregistrés) — confirme 100% de succès de replay des démonstrations sur StackCube après les corrections, donnant une vraie confiance dans le pipeline de données
    - [x] Construction d'un outillage d'évaluation approprié : seeds d'épisodes fixes partagés entre checkpoints pour une comparaison équitable, barres d'erreur standard (±1 SE), script de sweep de checkpoints reprenable qui reconstruit son graphique après chaque checkpoint
    - [x] Sweeps complets de checkpoints SFT sur 200 seeds fixes retenus — StackCube atteignant une vraie généralisation validée (pas de la mémorisation) avec des taux de succès croissants au fil des itérations SFT ; PushCube performant encore mieux
    - [x] Ajout de PushCube-v1 comme deuxième tâche, convertie avec le même pipeline corrigé en espace articulaire
    - [x] Ajout de PickCube-v1 comme troisième tâche pour diversifier davantage la suite de tâches OOD, et début d'investigation d'un taux de succès persistant de 0% spécifiquement sur cette tâche
    - [x] Traçage du taux de 0% de PickCube jusqu'à un problème de conception de tâche plutôt qu'un problème de policy : ManiSkill cache le marqueur d'objectif à la caméra que la policy voit réellement, et le schéma d'observation de style DROID n'a aucun autre canal exposant la position de l'objectif — la policy n'avait aucun moyen de savoir où placer le cube
    - [x] Construction d'un monkeypatch ciblé qui démasque le marqueur d'objectif spécifiquement pour la caméra capteur (plutôt que d'ajouter un canal d'état privilégié qu'un vrai robot n'aurait pas), et confirmation par inspection au niveau caméra que l'objectif est maintenant visible pour la policy
    - [x] Re-validation de PickCube de bout en bout avec le patch appliqué (replay des démonstrations, configuration caméra, SFT) — réussie, donc PickCube-v1 est finalement conservée comme troisième tâche
    - [x] Discussion de la question de la taille du dataset soulevée par l'équipe (Jesse/Albert) et construction d'un support de type `--num-demos` pour que le SFT puisse se limiter à un sous-ensemble d'un dataset déjà converti sans reconversion

### Décisions et ajustements

!!! info "Décisions"
    - Les mauvais résultats SFT initiaux n'étaient finalement pas un plafond de difficulté de StackCube — ils s'expliquaient entièrement par le mismatch d'espace d'action et les bugs supplémentaires trouvés durant le MAJOR RESET ; StackCube est conservée comme tâche
    - PushCube-v1 ajoutée comme deuxième tâche, plus facile, pour la vérification de bon fonctionnement du pipeline et la comparaison
    - PickCube-v1 initialement envisagée pour un retrait à cause du problème de marqueur d'objectif caché, mais conservée après qu'un patch ciblé de visibilité caméra ait résolu le problème sans recourir à un canal d'état privilégié qu'un vrai robot n'aurait pas — les trois tâches (StackCube, PushCube, PickCube) restent dans la suite active
    - Erreur standard (`SE = sqrt(p(1-p)/n)`), et non un intervalle de Wilson, adoptée pour toutes les barres d'erreur rapportées, à la demande explicite de l'équipe
    - Prochaine priorité fixée par l'équipe : une ablation sur la taille du dataset (des ~550 démonstrations complètes jusqu'à aussi peu que ~50), suivie d'une ablation sur la nécessité du warmup SFT (sauter le SFT, démarrer le RL directement depuis le modèle de base) — trois prédictions concurrentes enregistrées au sein de l'équipe pour cette dernière

### Difficultés rencontrées

!!! warning "Difficultés"
    - Diagnostiquer le mismatch d'espace d'action a demandé un effort réel : les symptômes (succès proche de 0%, bras approchant sans compléter la tâche) semblaient identiques à un vrai problème de difficulté de tâche
    - Plusieurs des bugs du MAJOR RESET étaient silencieux — ils ne faisaient rien planter, ils produisaient simplement des chiffres plausibles mais faux, ce qui est justement ce qui a rendu la revue systématique nécessaire
    - Le mode d'échec de PickCube ressemblait au départ à un problème de policy/exploration et a nécessité des diagnostics au niveau caméra pour être réellement rattaché à la conception d'observabilité de l'environnement


## Semaine 10 (6–10 juillet 2026)

### Objectifs de la période
- Réviser l'étape d'entraînement RL/ExpoFT aussi rigoureusement que l'étape SFT l'a été
- Lancer les premiers vrais entraînements RL sur StackCube, PushCube et PickCube
- Mettre l'infrastructure (structure du dépôt, usage disque, submodules) dans un état durable
- Comprendre si le RL améliore réellement la policy SFT, et sinon pourquoi

### Travail réalisé

!!! abstract "Avancement"
    - [x] Fork de ManiSkill et passage (avec `openpi`) d'une dépendance non trackée/installée via `pip` à un vrai git submodule éditable, permettant des modifications directes et trackées plutôt que des monkeypatches à l'exécution
    - [x] Migration du dépôt d'un stockage entièrement sur `$SCRATCH` vers `$HOME`, en ne gardant que les artefacts volumineux et régénérables (`logs/`, `demos/`) en lien symbolique vers `$SCRATCH`, suivant les recommandations de stockage de Mila
    - [x] Déplacement des gros caches de paquets (`uv`, `openpi`, `huggingface`, `pip`, `jax`) de `$HOME` vers `$SCRATCH`, libérant le quota `$HOME`
    - [x] Revue ligne par ligne complète de l'étape d'entraînement RL, trouvant et corrigeant un ensemble de bugs réels : un bug d'écrasement des normalisation stats identique à celui trouvé en évaluation SFT ; ~15 flags CLI passés au script d'entraînement mais jamais lus silencieusement par celui-ci ; un flag d'écrasement de checkpoint codé en dur faisant planter chaque entraînement neuf ; un désalignement entre deux copies différentes d'un flag d'entraînement clé causant un plantage dès qu'il était réglé à une valeur non par défaut
    - [x] Lancement des premiers vrais entraînements RL sur les trois tâches depuis leurs checkpoints SFT validés
    - [x] Diagnostic et correction de deux causes distinctes de plantages OOM répétés durant l'entraînement RL : un problème de recompilation JAX déclenché par une forme d'entrée incohérente en cours d'entraînement, et l'allocateur mémoire GPU par défaut de JAX ne réservant que 75% de la carte par défaut
    - [x] Découverte que le script d'évaluation ne pouvait en réalité évaluer aucun checkpoint RL — il supposait silencieusement que chaque checkpoint était de style SFT et n'aurait toujours évalué que la policy VLA figée ; construction d'un support adéquat pour charger et évaluer l'agent RL complet (residual policy et critic inclus)
    - [x] Découverte et correction d'un second ensemble de bugs de configuration : plusieurs hyperparamètres RL définis dans les configs de tâche n'étaient jamais réellement lus par le script d'entraînement, retombant silencieusement sur des valeurs par défaut différentes ailleurs dans la codebase ; la valeur d'un hyperparamètre était silencieusement interprétée comme du texte plutôt qu'un nombre par le lecteur de fichier de config
    - [x] Lancement de courbes d'évaluation complètes (200 seeds fixes retenus par checkpoint) sur les premiers entraînements RL à travers les trois tâches
    - [x] Lancement d'une série d'expériences de comparaison faisant varier le ratio update-to-data et la proportion de chaque batch d'entraînement tirée des données de démonstration, incluant un entraînement reproduisant les hyperparamètres exacts du papier original dans sa propre durée d'entraînement validée
    - [x] Reproduction de la recette exacte d'augmentation de données d'entraînement du papier à partir de sa description, et confirmation que la codebase l'implémente déjà correctement
    - [x] Correction du logging TensorBoard en cours d'entraînement, qui manquait silencieusement la plupart des métriques d'entraînement (seuls le taux de succès et le temps de boucle étaient visibles) à cause d'un problème de filtrage de type

### Décisions et ajustements

!!! info "Décisions"
    - `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95` adopté comme réglage GPU standard pour tous les scripts de job RL, résolvant les plantages OOM récurrents
    - Tous les hyperparamètres RL proviennent désormais exclusivement des configs YAML de tâche, avec des overrides explicites câblés dans le script d'entraînement, plutôt que d'être répartis silencieusement entre le YAML et un fichier de config de modèle séparé
    - Adoption d'un ratio update-to-data réduit (2, contre 20 dans le papier) comme valeur par défaut de travail, après avoir observé que le ratio du papier déstabilise l'entraînement encore plus vite que le ratio réduit, compte tenu des contraintes de cette configuration simulée
    - Les entraînements catastrophiques antérieurs aux corrections de mémoire et de configuration ont été supprimés de `$SCRATCH` pour libérer de l'espace disque, en ne gardant que les checkpoints SFT validés et les entraînements RL post-correction

### Difficultés rencontrées

!!! warning "Difficultés"
    - Chaque entraînement RL essayé jusqu'ici — à travers les trois tâches, à travers chaque ratio update-to-data et chaque réglage de proportion de démonstration testés, y compris un entraînement court reproduisant les hyperparamètres exacts du papier dans sa propre fenêtre d'entraînement validée — finit par dégrader le taux de succès de la policy pré-entraînée par SFT plutôt que de l'améliorer
    - Cette dégradation est systématiquement précédée et accompagnée d'une instabilité croissante de la loss du critic, suggérant que la cause racine est une fonction de valeur se déstabilisant progressivement plutôt qu'un bug du pipeline (vérifié : l'évaluation utilise désormais bien la residual policy et le critic entraînés, pas seulement la policy VLA figée)
    - Réduire le ratio update-to-data retarde le déclenchement de cette instabilité mais ne l'a empêchée dans aucune configuration testée jusqu'ici — soulevant la question ouverte de savoir s'il s'agit d'une limitation fondamentale de l'entraînement de critic off-policy à haut ratio update-to-data que les entraînements courts sur robot réel du papier n'ont simplement pas le temps de révéler
    - Ce constat a été partagé avec le groupe de recherche élargi pour discussion, car il touche directement à la question de recherche centrale du projet


## Semaine 11 (13–17 juillet 2026) — semaine en cours

### Objectifs de la période

- Déterminer si la dégradation des performances du RL observée à la fin de la semaine 10 est due à une instabilité de l'entraînement du critic, à un problème d'intégrité des données, ou à un problème structurel lié à l'algorithme lui-même
- Reconstruire l'architecture du critic afin qu'elle corresponde le plus fidèlement possible aux articles XQC/XQCfD, et tester chacun des mécanismes proposés individuellement plutôt que de les activer tous simultanément
- Appliquer la même rigueur d'analyse aux baselines PPO/GRPO qui avaient été ajoutées mais qui n'avaient jamais réellement été exécutées
- Mettre à jour la documentation du projet (README, changelog) avec l'ensemble des découvertes effectuées

### Travaux réalisés

!!! abstract "Progrès"
    - [x] Implémentation du pré-entraînement du critic (warm-start BC/TD de type XQCfD sur les démonstrations avant le début du RL, activable via YAML) et test isolé — cela n'a pas permis de résoudre la dégradation observée à lui seul
    - [x] Reconstruction complète de l'architecture du critic afin de correspondre à XQC/XQCfD : remplacement du critic original basé sur une régression scalaire et un ensemble de critics par un critic catégoriel (support borné, style C51) utilisant batch normalization et weight normalization — l'architecture originale a été conservée inchangée dans un fichier séparé afin de rester directement exécutable pour les comparaisons
    - [x] Ajout d'une normalisation adaptative des récompenses (estimation continue de l'échelle des récompenses) afin que les bornes fixes du critic restent pertinentes entre différentes tâches sans nécessiter un réglage manuel — vérifié que cette approche reste valide même lorsqu'un changement important et soudain de l'échelle des récompenses survient
    - [x] Réalisation de comparaisons contrôlées entre la nouvelle architecture du critic et l'ancienne, ainsi qu'entre la boucle de collecte de données avec le bug de synchronisation des récompenses corrigé et la version contenant le bug — dans les deux cas, les estimations de valeur du critic sont devenues significativement plus stables et mieux comportées, mais le taux de succès de la policy s'est effondré de manière identique, ce qui constitue lui-même une découverte importante (voir Décisions)
    - [x] Identification et correction d'un véritable bug dans la boucle de collecte de données en ligne : le signal de récompense/terminaison de l'environnement était lu un pas trop tôt, ce qui associait chaque transition stockée à la récompense de l'action précédente plutôt qu'à celle ayant réellement produit cette transition — confirmé par une reproduction contrôlée et non uniquement par une lecture du code ; le problème est limité à la collecte RL en ligne (le pré-entraînement supervisé et la génération des démonstrations n'ont jamais été affectés)
    - [x] Analyse de la raison pour laquelle les expériences sur robot réel de l'article original ne semblent pas rencontrer cette même dégradation malgré l'utilisation d'un mécanisme identique — l'explication la plus probable est leur correction humaine en temps réel pendant l'entraînement ainsi que leur horizon d'entraînement beaucoup plus court, plutôt qu'une différence fondamentale dans l'algorithme lui-même
    - [x] Implémentation d'une régularisation KL vers la policy pré-entraînée (remplaçant le bonus d'exploration générique, conformément à XQCfD) pour la policy résiduelle entraînable, et réalisation d'un premier test — bien qu'un écart de configuration ait empêché ce premier essai d'être totalement isolé, les résultats sont les plus encourageants obtenus durant cette investigation jusqu'à présent (amélioration constante du taux de succès par rapport à une exécution comparable sans cette régularisation) ; un test isolé de suivi est en cours
    - [x] Extension de la même rigueur de débogage aux baselines PPO et GRPO ajoutées précédemment mais jamais réellement testées : identification et correction de plusieurs bugs numériques indépendants (incompatibilité lors de la construction du réseau, erreur de format d'action, fallback incorrect pour une valeur n'ayant pas de forme fermée pour notre type de policy, valeur non bornée pouvant atteindre silencieusement l'infini en cas de dérive de la policy, et instabilité numérique à la frontière de l'espace d'action) — chaque problème a été confirmé par un test ciblé et reproductible plutôt que par simple inspection du code, avec ajout de protections défensives afin qu'une occurrence similaire future soit détectée et ignorée plutôt que de corrompre silencieusement l'entraînement
    - [x] Découverte que la policy entraînable de PPO/GRPO n'était en réalité jamais connectée au checkpoint pré-entraîné supervisé — elle correspondait à un réseau séparé initialisé aléatoirement, tandis que le checkpoint pré-entraîné était uniquement utilisé pour le pré/post-traitement des entrées et sorties. Cela explique pourquoi ces baselines ne recevaient aucun signal d'apprentissage utile dès le départ, indépendamment des bugs précédemment identifiés. Ajout d'une option équivalente de warm-start par behavior cloning pour corriger ce problème
    - [x] Mise à jour complète du README du projet : résultats actuels, nouvelle architecture du critic et sa justification, bug de synchronisation temporelle et son périmètre, ainsi qu'un changelog complet des corrections réalisées durant cette phase
    - [x] Rendre l'architecture originale du critic directement sélectionnable et exécutable (et pas seulement conservée comme référence statique) afin de permettre des comparaisons A/B avec la nouvelle architecture dans des conditions identiques

### Décisions et ajustements

!!! info "Décisions"
    - Traiter séparément la reconstruction de l'architecture du critic et la correction du bug de synchronisation des récompenses, avec des tests indépendants plutôt que de regrouper les modifications, afin que leurs effets (ou leur absence d'effet) puissent être attribués précisément à chaque changement
    - Puisque ces deux corrections réelles et confirmées ont amélioré le comportement propre du critic sans modifier les performances de la policy, l'hypothèse principale s'est déplacée vers le mécanisme de sélection d'action lui-même (le même critic propose et évalue sa propre action préférée, ce qui introduit un biais connu dans ce type d'algorithme), plutôt que vers l'entraînement du critic
    - Décision d'épuiser d'abord les mécanismes proposés par l'article lui-même (pré-entraînement du critic, nouvelle architecture du critic, régularisation KL, et — à venir — une modification architecturale visant à rendre la policy consciente lorsqu'elle sort de son domaine familier) avant de tester une solution alternative issue d'une autre ligne de recherche, afin que les résultats obtenus ou non restent attribuables à une approche unique et clairement référencée
    - Confirmation que les tâches utilisées reposent sur une récompense dense (shaped reward) et non une récompense sparse — élément important pour évaluer l'importance réelle du bug de synchronisation dans notre contexte, comparativement au cadre original de l'article basé sur des récompenses sparse et un robot réel

### Difficultés rencontrées

!!! warning "Difficultés"
    - Deux corrections indépendantes et confirmées (l'architecture du critic et le bug de synchronisation des récompenses) ont chacune amélioré de manière mesurable le comportement du critic sans améliorer les performances réelles de la policy — résultat utile pour réduire l'espace des causes possibles, mais qui rappelle que « un critic plus sain » et « une meilleure policy » ne sont pas nécessairement équivalents
    - Le premier test de régularisation KL était biaisé par la présence résiduelle d'un terme de bonus d'exploration qui aurait dû être désactivé pour un test isolé — l'erreur a été détectée avant de tirer des conclusions, mais souligne l'importance de vérifier toutes les interactions possibles lors de l'ajout d'une nouvelle fonctionnalité
    - Les baselines PPO/GRPO ne réalisaient pas ce qui était supposé depuis leur intégration (continuer depuis la policy pré-entraînée) — une différence importante et difficile à détecter qui n'a été révélée qu'après une analyse critique de leurs résultats plutôt qu'une simple interprétation directe des performances obtenues