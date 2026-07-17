---
title: Travail réalisé
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Implémentation

## Architecture et approche générale

### Objectif de recherche

La question centrale de ce projet est de savoir si le RL fine-tuning peut permettre une **généralisation out-of-distribution (OOD)** chez les modèles Vision-Language-Action (VLA) — c'est-à-dire permettre à un modèle de réussir des tâches maximalement éloignées de sa distribution d'entraînement SFT. Une question secondaire est de savoir si le principal goulot d'étranglement pour l'amélioration par RL est l'**exploration** (la politique n'arrive pas à découvrir de signal de récompense) ou l'**optimisation** (la politique n'arrive pas à s'améliorer à partir d'un signal donné).

### Vue d'ensemble du pipeline

Le projet s'est déroulé en deux grandes phases :

**Phase 1 — RL on-policy (Semaines 1–7) :** Fine-tuning RL direct via GRPO/PPO sur OpenVLA-OFT et π₀ dans les environnements LIBERO et RoboCasa. Cette phase a établi l'hypothèse du goulot d'étranglement d'exploration et les limites des approches on-policy sans warmup SFT.

**Phase 2 — Politique résiduelle ExpoFT (Semaine 8–présent) :** Reproduction et adaptation de l'[architecture ExpoFT](https://pd-perry.github.io/expo-ft/) dans un codebase dédié, piloté entièrement par la simulation ManiSkill (`expo-ft`), remplaçant le pipeline de données original d'ExpoFT (robot réel, DROID) tout en gardant initialement l'algorithme lui-même intact, avant de s'en écarter de façon ciblée une fois l'instabilité diagnostiquée (voir plus bas). Le VLA (π₀.₅) est gardé **figé**, et une petite politique de correction résiduelle est entraînée par-dessus via une Q-function, en combinant RL off-policy et démonstrations par motion planning. Ce design évite deux limitations de la Phase 1 : il ne nécessite pas de log-probabilités d'action valides (que les VLA à flow-matching comme π₀/π₀.₅ ne peuvent pas fournir, excluant le policy-gradient RL standard directement sur le modèle de base) et il contourne le problème d'exploration à froid en amorçant le replay buffer avec des démonstrations. Des baselines on-policy (PPO, GRPO) et une baseline off-policy sans le schéma résidu/argmax (SAC) ont ensuite été ajoutées au même codebase pour comparaison directe (voir plus bas).

La méthodologie retenue est le **retrait progressif de composants** : partir de la recette ExpoFT complète (warmup SFT + démonstrations + RL off-policy) et valider rigoureusement chaque étage avant de retirer des composants (taille du dataset, warmup SFT lui-même) pour identifier la supervision minimale réellement nécessaire.


### Environnement de travail

- **Cluster :** Mila HPC (SLURM), GPU A100/A100L (40GB/80GB)
- **Modèles testés :** OpenVLA-OFT (7B), π₀ (3B) (Phase 1) ; π₀.₅ (checkpoint `pi05_droid_jointpos`, figé) + politique résiduelle (Phase 2)
- **Environnements :** LIBERO (Spatial, Long), RoboCasa (CloseDrawer) — Phase 1 ; ManiSkill (StackCube-v1, PushCube-v1, PickCube-v1) — Phase 2
- **Algorithmes RL :** GRPO, PPO (Phase 1) ; ExpoFT (VLA figé + politique résiduelle, Q-learning off-policy via un ensemble de critics façon REDQ au départ, puis un critic catégoriel/distributionnel — voir plus bas), plus des baselines PPO, GRPO et SAC ajoutées directement au même codebase pour comparaison (Phase 2)
- **Framework :** RLinf (`montrealrobotics/RLinf`) pour la Phase 1 ; dépôt autonome `expo-ft` (fork propre d'`openpi` et de `ManiSkill` en submodules git) pour la Phase 2
- **Infrastructure clé :** Quatre venvs Python séparés pour gérer les conflits de dépendances ; dépôt vivant sur `$HOME` avec `logs/` et `demos/` en lien symbolique vers `$SCRATCH` ; gros caches de paquets déplacés vers `$SCRATCH`

---

## Expériences et analyses réalisées

### Phase 1 — Baselines RL on-policy

#### LIBERO-90 GRPO (OpenVLA-OFT)
Première expérience. 1000 steps (~31h40). `success_once` a montré des pics occasionnels (~0.13) sans tendance stable à la hausse. D'abord attribué à une petite batch size ; confirmé ensuite comme un goulot d'étranglement d'exploration.

![LIBERO-90 GRPO Training Results Success Once]({{ config.site_url }}/images/libero90_grpo_training1_success_once.png)
![LIBERO-90 GRPO Training Results Approx KL]({{ config.site_url }}/images/libero90_grpo_training1_approx_kl.png)

#### RoboCasa CloseDrawer PPO (π₀)
Run de 150 steps avec LoRA (rank 32). Métriques finales : `success_once` ~0.82, `critic/value_loss` ~0.005, `approx_kl` ~0.036. A démontré que RL peut améliorer la performance quand la tâche est in-distribution — mais CloseDrawer était trop facile (base SFT déjà à ~0.6–0.7), laissant peu de marge de contribution pour RL.

![RoboCasa CloseDrawer PPO Training Results Success Once]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_success_once.png)
![RoboCasa CloseDrawer PPO Training Results Approx KL]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_approx_kl.png)
![RoboCasa CloseDrawer PPO Training Results Value Loss]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_value_loss.png)

<div style="text-align: center">
    <video width="480" controls poster="{{ config.site_url }}/images/robocasa_close_drawer_thumbnail.png">
        <source src="{{ config.site_url }}/images/robocasa_close_drawer_training1_ppo_step_150.mp4" type="video/mp4">
    </video>
</div>

*π₀ + RoboCasa CloseDrawer — épisode au step 150 (après entraînement PPO)*

#### LIBERO-Spatial GRPO (OpenVLA-OFT)
Run de ~38 steps. `success_once` a oscillé entre 0.22 et 0.875 sans convergence claire. Identifié comme un mauvais réglage du learning rate suite à une réduction du nombre d'epochs.

![LIBERO-Spatial GRPO Training Results Success Once]({{ config.site_url }}/images/libero_spatial_grpo_training1_success_once.png)
![LIBERO-Spatial GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_spatial_grpo_training1_approx_kl.png)

#### LIBERO-Long GRPO (OpenVLA-OFT)
~150 steps sur 4 sessions reprises (~37h au total). `success_once` a oscillé entre 0.06 et 0.19 sans tendance à la hausse. `approx_kl` est resté stable à ~0.035–0.048 tout du long — les mises à jour de la politique étaient contrôlées, confirmant que le goulot d'étranglement est l'**exploration**, pas une instabilité d'optimisation.

![LIBERO-Long GRPO Training Results Success Once]({{ config.site_url }}/images/libero_long_grpo_training1_success_once.png)
![LIBERO-Long GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_long_grpo_training1_approx_kl.png)

#### StackCube-v1 PPO et GRPO (base OpenVLA, on-policy, sans SFT)
Les deux runs ont produit 0% de succès. Sans warmup SFT, la politique de base ne peut produire aucun comportement pertinent pour la tâche, donc l'exploration on-policy ne peut pas amorcer de signal d'apprentissage. Ce résultat a directement motivé le pivot vers ExpoFT. Voici les résultats des runs PPO et GRPO sur StackCube-v1 :

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-ppo-nosft.mp4" type="video/mp4">
    </video>
</div>

*OpenVLA sur maniskill StackCube-v1 après PPO*


<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-grpo-nosft.mp4" type="video/mp4">
    </video>
</div>

*OpenVLA sur maniskill StackCube-v1 après GRPO*


### Sélection et visualisation du dataset OOD

- Création d'un notebook Jupyter pour l'analyse de distribution des points d'arrivée à travers les datasets Open X-Embodiment ([Colab](https://colab.research.google.com/github/Josh012006/OpenX-Embodiment-Datasets-Visualization/blob/main/colabs/Open_X_Embodiment_Datasets.ipynb), [Repo](https://github.com/Josh012006/OpenX-Embodiment-Datasets-Visualization))
- Un fit UMAP partagé a été utilisé pour tous les datasets (plutôt qu'un fit par dataset), puisqu'un fit par dataset rendrait les coordonnées incomparables entre datasets
- ManiSkill identifié comme benchmark OOD cible : des tâches comme StackCube-v1 et PushCube-v1 sont maximalement éloignées des données SFT/pretraining du VLA (DROID/OXE)
- Remarque : la distance L2 spatiale entre datasets peut être trompeuse à cause des référentiels de coordonnées différents entre datasets

### Exploration d'hyperparamètres — réduction du temps d'entraînement (Phase 1)

Test d'une augmentation du replay ratio (`rollout_epoch` 4→8 pour GRPO, `update_epoch` 2→4 pour PPO) pour réduire le temps réel. Résultat : le goulot d'étranglement s'est déplacé de la collecte de rollouts vers les mises à jour de gradient, rendant l'entraînement *plus lent* (~27 min/step contre ~14 min/step). L'approche efficace consiste plutôt à réduire directement `max_episode_steps` et `max_epochs`.

---

### Phase 2 — ExpoFT : mise en place du dépôt

L'algorithme ExpoFT a été reproduit dans un dépôt dédié, piloté par ManiSkill (`~/projects/expo-ft`) plutôt que dans RLinf, pour garder le code de l'algorithme original aussi intact que possible tout en ne remplaçant que la couche environnement/données. Deux dépendances — `openpi` (l'implémentation de π₀.₅) et `ManiSkill` — ont été forkées et intégrées comme véritables submodules git éditables (plutôt que des dépendances non trackées ou installées via `pip`), permettant des modifications directes et trackées plutôt que des monkeypatches à l'exécution. `.gitmodules` utilise des URLs HTTPS pour la compatibilité de clonage public, avec des remotes SSH configurés localement pour les push.

L'organisation du stockage suit les recommandations du cluster Mila : le dépôt lui-même vit sur `$HOME` (sauvegardé, 100GB), tandis que les artefacts volumineux et régénérables (`logs/`, `demos/`) sont en lien symbolique vers `$SCRATCH` (5TB, nettoyé automatiquement). Les gros caches de paquets (`uv`, `openpi`, `huggingface`, `pip`, `jax`, ~90GB au total) ont été déplacés vers `$SCRATCH/cache/` avec des liens symboliques vers `~/.cache`, libérant le quota `$HOME`.

### Phase 2 — Étage SFT : diagnostic et validation de l'espace d'action

Le premier run de warmup SFT (Semaine 8), sur des démonstrations StackCube-v1 converties au format DROID/LeRobot, a produit un taux de succès proche de 0% — le bras approchait seulement les cubes sans compléter l'empilement. Ceci a d'abord été suspecté d'être un plafond de difficulté réel de la tâche, mais une investigation systématique l'a retracé à un **décalage fondamental d'espace d'action** : le pretraining DROID de π₀.₅ attend des actions en position articulaire, en unités physiques, alors que le pipeline utilisait le contrôle cartésien `pd_ee_delta_pose` de ManiSkill, causant l'écrêtage silencieux de la majorité des actions de démonstration hors de leur plage valide.

Le fix — basculer l'ensemble du pipeline vers le contrôle en espace articulaire de bout en bout (`control_mode=pd_joint_delta_pos`, checkpoint `pi05_droid_jointpos`, état `observation/joint_position`, `normalize_action=True`, `output_action_dim=8`, stats de normalisation DROID depuis GCS) — a immédiatement ramené la grande majorité des actions dans leur plage valide lors de la validation par replay.

Ceci a déclenché une revue systématique plus large ("MAJOR RESET") qui a mis au jour un ensemble de bugs silencieux supplémentaires avant que les chiffres SFT puissent être considérés fiables :

- Un script d'évaluation écrasant sans condition les stats de normalisation DROID officielles, même pour l'évaluation de baseline
- Un artefact de rescaling d'action en espace cartésien, résidu du mode de contrôle abandonné, corrompant une fraction significative des actions désormais en espace articulaire
- Un décalage de longueur de troncature entre l'environnement et la boucle d'évaluation
- Une incompatibilité de backend de simulation empêchant la conversion correcte du mode de contrôle

Pour regagner confiance dans le pipeline de données, `validate_demos_full_pipeline.py` a été construit comme un outil de replay rigoureux de bout en bout (utilisant les vraies stats de normalisation, le vrai wrapper d'environnement, et les seeds exactement enregistrées). Après les fixes ci-dessus, cet outil a confirmé **100% de succès de replay des démos sur StackCube**, donnant une réelle confiance dans le fait que l'espace d'action et la conversion des données étaient enfin corrects.

Un outillage d'évaluation approprié a aussi été construit à cette étape : des seeds d'épisode fixes (200 seeds held-out, générées une fois à partir d'une seed maîtresse et réutilisées pour tous les checkpoints) pour une comparaison équitable entre checkpoints, des barres d'erreur standard (`SE = sqrt(p(1-p)/n)`, une approximation normale plutôt qu'un intervalle de Wilson, selon la préférence explicite de l'équipe), et un script de balayage de checkpoints reprenable qui reconstruit son graphique après chaque checkpoint.

Des balayages complets de checkpoints SFT sur ces 200 seeds fixes ont confirmé une **généralisation réelle et validée** (pas de la mémorisation) sur StackCube, avec des taux de succès croissants à travers les itérations SFT ; PushCube-v1 (ajoutée comme deuxième tâche, plus facile, pour vérifier la cohérence du pipeline) a obtenu des performances encore meilleures.

Voici les courbes de balayage de checkpoints SFT (taux de succès vs. step/itération d'entraînement, avec barres d'erreur SE, 200 seeds held-out) pour **StackCube-v1** et **PushCube-v1**.

![SFT Checkpoint Sweep Curves PushCube-v1]({{ config.site_url }}/images/curve-sft-pushcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pushcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur maniskill PushCube-v1 après warmup SFT*


![SFT Checkpoint Sweep Curves StackCube-v1]({{ config.site_url }}/images/curve-sft-stackcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur maniskill StackCube-v1 après warmup SFT*



Et voici des enregistrements d'épisodes du comportement du meilleur checkpoint SFT sur StackCube-v1 et PushCube-v1, pour accompagner visuellement les courbes de balayage ci-dessus (même style que la vidéo RoboCasa CloseDrawer en Phase 1).

### Phase 2 — Suite de tâches : StackCube, PushCube, et PickCube (retirée, puis réintégrée)

PickCube-v1 a été ajoutée comme troisième tâche pour diversifier davantage la suite OOD, mais montrait initialement un taux de succès persistant de 0%. Ceci a été retracé à une **incompatibilité de design de la tâche** plutôt qu'un problème de politique ou d'exploration : PickCube-v1 de ManiSkill cache le marqueur d'objectif (`goal_site`) aux caméras capteur par défaut, et bien que `obs['extra']['goal_pos']` soit techniquement accessible dans le simulateur, le schéma d'observation imitant DROID utilisé tout au long de ce projet (image + proprioception uniquement, correspondant à ce qu'un robot réel aurait) n'avait aucun canal pour cette information. La politique n'avait donc aucun accès à la position de l'objectif par épisode — ni visuellement ni numériquement — rendant la tâche apparemment insoluble.

PickCube-v1 a été retirée dans un premier temps plutôt que contournée avec un canal d'état privilégié, puisque cela ne correspondrait pas à ce qu'un robot réel pourrait percevoir. Cependant, un monkeypatch révélant `goal_site` à la caméra capteur elle-même — donnant à la politique un accès *visuel* réel à la position de l'objectif, exactement comme une vraie caméra pointée sur une cible visible le ferait — a ensuite été validé avec succès. Ceci a résolu le manque d'observabilité sans compromettre le schéma d'observation imitant DROID, et **PickCube-v1 a été réintégrée dans la suite de tâches active**.

J'ai donc aussi lancé un warmup SFT sur PickCube-v1. Voici les performances :

![SFT Checkpoint Sweep Curves PickCube-v1]({{ config.site_url }}/images/curve-sft-pickcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pickcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur maniskill PickCube-v1 après warmup SFT*


**Suite de tâches active à partir de maintenant : StackCube-v1, PushCube-v1, et PickCube-v1.**

### Phase 2 — Étage RL : revue de code et fixes d'infrastructure

Avant de faire confiance à un quelconque résultat RL, l'étage d'entraînement RL a reçu le même niveau de revue ligne par ligne que l'étage SFT. Ceci a mis au jour un autre ensemble de vrais bugs :

- Un bug d'écrasement des stats de normalisation identique à celui trouvé précédemment dans l'évaluation SFT
- Environ 15 flags CLI passés au script d'entraînement mais jamais lus par celui-ci (hyperparamètres orphelins — présents dans les configs, jamais réellement consommés par le code)
- Un flag d'écrasement de checkpoint codé en dur qui faisait planter chaque run frais (non repris)
- Une incohérence entre deux copies différentes d'un flag d'entraînement clé, causant un crash dès que sa valeur différait du défaut
- Plusieurs hyperparamètres RL définis dans les configs YAML de tâche mais jamais réellement lus par le script d'entraînement, retombant silencieusement sur des défauts différents définis ailleurs dans le codebase
- La valeur d'un hyperparamètre silencieusement interprétée comme du texte plutôt qu'un float par le lecteur de config (PyYAML interprète la notation scientifique nue comme `3e-4` comme une chaîne de caractères, pas un float)

Séparément, `eval_policy.py` a été trouvé avec `only_base_actions=True` par défaut, ce qui signifie qu'il évaluait silencieusement uniquement la politique VLA de base figée — jamais la politique résiduelle entraînée — sur chaque checkpoint RL évalué jusque-là. Un support approprié pour charger et évaluer l'agent RL complet entraîné (politique résiduelle et critic ensemble) a été construit et utilisé pour toutes les évaluations à partir de ce point.

Deux causes distinctes de crashs répétés par manque de mémoire (OOM) pendant l'entraînement RL ont aussi été diagnostiquées et corrigées :
- Un problème de recompilation JAX déclenché par une forme d'entrée incohérente à mi-parcours de l'entraînement (un basculement de structure pytree `None`→dict pour `actor_batch` à travers une frontière `jax.jit` forçait JAX à compiler et garder résident silencieusement un second programme XLA)
- L'allocateur mémoire GPU par défaut de JAX ne réservant que 75% de la carte par défaut, plutôt que la carte entière — résolu avec `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95`

Enfin, le logging TensorBoard en cours d'entraînement s'est révélé silencieusement dépourvu de la plupart des métriques d'entraînement (seuls le taux de succès et le temps de boucle étaient visibles, alors que wandb affichait tout correctement) à cause d'un problème de filtrage par type — les scalaires JAX `jnp.float32` échouent aux vérifications `isinstance(v, (int, float))` de Python. Corrigé avec un cast explicite `float()`.

La recette exacte d'augmentation de données d'entraînement du papier a été reproduite à partir de sa description et confirmée comme déjà correctement implémentée dans le codebase.

### Phase 2 — Étage RL : la découverte centrale d'instabilité

Une fois les bugs d'infrastructure et d'évaluation corrigés, et PickCube-v1 réintégrée après son fix d'observabilité, les premiers vrais runs de fine-tuning RL ont été lancés à partir des checkpoints SFT validés sur les trois tâches actives. Des courbes d'évaluation complètes (200 seeds fixes held-out par checkpoint) ont ensuite été produites pour une batterie d'expériences comparatives faisant varier le **ratio update-to-data (UTD)** et la proportion de chaque batch d'entraînement tirée des données de démonstration, incluant un run reproduisant les hyperparamètres exacts du papier dans la durée d'entraînement validée par le papier lui-même.

**Résultat clé :** chaque configuration RL testée jusqu'ici — sur les trois tâches actives (StackCube, PushCube, PickCube) et chaque réglage de ratio UTD / proportion de démonstration testé — **dégrade la politique pré-entraînée par SFT plutôt que de l'améliorer** :

- `utd_ratio=10` : effondrement catastrophique en ~2–5k steps
- `utd_ratio=2` + `offline_ratio=0.5` : retarde l'effondrement à ~35–40k steps, se dégradant finalement à ~0.3–0.4 de succès vers 85–90k steps
- `utd_ratio=20` (la valeur exacte du papier, exécutée dans la plage de steps validée par le papier lui-même, ~20–24k steps) : s'est effondré **plus vite et plus sévèrement** que le long run à `utd_ratio=2` — ceci réfute l'hypothèse qu'entraîner simplement dans le budget plus court du papier (robot réel) éviterait le problème ; `utd_ratio` lui-même, pas la durée totale d'entraînement, semble être le facteur dominant de l'instabilité

La dégradation est systématiquement précédée et accompagnée d'une **instabilité croissante de la loss d'entraînement du critic** (spikes de critic loss progressifs), suggérant que la cause racine est une fonction de valeur se déstabilisant progressivement plutôt qu'un bug de pipeline — ceci a été confirmé davantage en vérifiant que l'évaluation utilise démontrablement la politique résiduelle et le critic entraînés, sans retomber silencieusement sur la politique de base figée (voir le fix d'`eval_policy.py` ci-dessus).

Correspondance d'hyperparamètres confirmée avec le papier original : `offline_ratio=0` (les démonstrations sont placées dans l'unique replay buffer en ligne, pas dans un buffer hors-ligne séparé à 50/50 — cette convention appartient à RLPD, pas à ce papier), `hidden_dims=3×256`, `init_temperature=1.0`, `num_qs=10` / `num_min_qs=2` (ensemble de critics façon REDQ), `tau=0.005`. Écarts connus par rapport au papier : `batch_size=24` (le papier utilise 64 ; contraint par la mémoire GPU disponible) et un `edit_scale=0.2` fixe (le papier utilise 0.05–0.2, ajusté selon la difficulté de la tâche).

Voici une comparaison des métriques d'entraînement à travers les trois configurations de ratio UTD décrites ci-dessus, pour PushCube-v1, la plus facile des trois tâches :

*Évolution de la critic loss pendant l'entraînement* :

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    <tr>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


*Évolution de la residual actor loss pendant l'entraînement* :
<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


*Évolution du taux de succès en évaluation pendant l'entraînement* :
<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


Et voici une vidéo montrant la dégradation du comportement de la politique sur StackCube-v1 après ~120k steps de fine-tuning RL :
<table>
    <tr>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
                </video>
            </div>

            *Avant fine-tuning RL avec ExpoFT*
        </td>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-rl-degradation.mp4" type="video/mp4">
                </video>
            </div>

            *Après fine-tuning RL avec ExpoFT*
        </td>
    </tr>
</table>

**Question de recherche ouverte**, soumise au groupe pour discussion : est-ce une limitation fondamentale de l'entraînement de critic off-policy à UTD élevé, que les runs courts du papier sur robot réel n'exposent tout simplement jamais faute de durée suffisante, ou est-ce spécifique à cette configuration simulée (dataset plus petit, architecture VLA-figé + politique résiduelle, ou batch size contrainte par la mémoire) ? Ceci porte directement sur la question de recherche centrale du projet, puisque cela bloque actuellement toute démonstration d'une contribution RL positive par-dessus la baseline SFT validée.

**Pistes non testées identifiées pour la suite :** un `edit_scale` plus petit (par ex. 0.05) spécifiquement pour StackCube, puisque c'est la tâche la plus exigeante en précision et celle qui s'est effondrée le plus dramatiquement ; `replan_steps=4` pour StackCube (actuellement fixé à 8, alors que le papier utilise 4–8 selon la tâche) ; et `batch_size=32` comme prochain test à faible risque (jamais encore essayé).

### Phase 2 — Étage RL : isoler l'instabilité du critic d'un biais structurel de sélection

La phase de travail suivante a repris les pistes ci-dessus et les a testées systématiquement une par une, tout en prenant du recul pour caractériser précisément *pourquoi* la loss d'entraînement du critic se déstabilisait en premier lieu, plutôt que de continuer à chercher des hyperparamètres.

**Mécanisme identifié.** ExpoFT sélectionne l'action exécutée en échantillonnant plusieurs actions candidates (la sortie du VLA de base elle-même, plus plusieurs variantes éditées par résidu), en notant toutes ces candidates avec le critic, et en prenant celle avec le meilleur score — et ce même critic sert ensuite à bootstrap sa propre cible d'entraînement pour la candidate choisie. C'est un biais classique de maximisation/auto-référence (étroitement lié à la raison d'être de Double Q-learning) : l'argmax va systématiquement préférer la candidate que le critic surestime le plus, et cette valeur surestimée devient ensuite une partie de la prochaine cible d'entraînement du critic lui-même, renforçant l'erreur. Ce mécanisme a été identifié comme la cause racine la plus probable de l'instabilité croissante de la critic loss décrite ci-dessus, et est devenu l'hypothèse organisatrice pour la suite de cette phase de travail.

**Premières mitigations testées, individuellement — aucune n'a résolu l'effondrement :**

- Une ablation à température fixe, le gel de l'encodeur d'image du critic, le gradient clipping sur le critic, et le weight decay du critic ont chacun été testés isolément comme stabilisateurs candidats. Aucun n'a changé le résultat qualitatif (`target_q_max` continuait de croître sans borne, `critic_loss` continuait de devenir de plus en plus erratique, `eval/success_rate` continuait de s'effondrer).
- Un `edit_scale` plus petit (0.05, suivant les recommandations du papier lui-même pour les tâches exigeant de la précision) a été testé spécifiquement sur StackCube. Même résultat.
- Le pretraining du critic — une phase de warmup, avant tout RL, qui entraîne uniquement le critic (jamais la politique résiduelle) sur des données de démonstration via le même mécanisme de Bellman backup utilisé ensuite — a été implémenté comme une option activable via YAML, suivant une idée similaire proposée dans le papier XQCfD (arXiv 2605.10734) pour la cohérence critic/actor. Testé isolément : n'a pas résolu l'effondrement. Notamment, `target_q_max` pendant cette phase de pretraining hors-ligne uniquement a *elle-même* recommencé à grimper à mi-parcours, sur des données de démonstration seules — une preuve que l'instabilité ne concerne pas principalement les données en ligne que voit le critic, mais le mécanisme d'argmax/bootstrap lui-même.

**Reconstruction de l'architecture du critic (XQC / XQCfD).** Pour tester si la dynamique d'entraînement du critic lui-même (par opposition au mécanisme de sélection) était le facteur principal, le critic a été reconstruit de zéro en suivant XQC (arXiv 2509.25174) et XQCfD (arXiv 2605.10734) : une tête **catégorielle (façon C51, à support borné fixe)** remplaçant la cible de régression scalaire non bornée originale, avec **batch normalization et weight normalization** sur le MLP du critic, et **aucun ensemble** (correspondant à la configuration rapportée par XQCfD elle-même). Le signal de récompense est normalisé par une estimation courante de sa propre échelle avant utilisation, pour que les bornes de valeur fixes restent pertinentes à travers les tâches sans les ajuster à la main pour chacune. L'architecture originale a été préservée, non modifiée, dans un module séparé plutôt qu'écrasée, spécifiquement pour que les deux puissent tourner côte à côte dans des conditions par ailleurs identiques.

**Résultat :** les propres estimations de valeur du critic catégoriel se comportent de façon qualitativement différente et plus saine — elles convergent et se stabilisent dans une plage bornée plutôt que de grimper indéfiniment, et la loss d'entraînement du critic reste lisse plutôt que de devenir de plus en plus erratique. Mais `eval/success_rate` s'effondre toujours essentiellement de la même façon. C'était un résultat négatif important : il écarte l'instabilité d'entraînement du critic elle-même comme facteur principal de la dégradation de la politique.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-target-q-max.png" alt="Curve">

            *target_q_max — architecture de critic originale vs. catégorielle*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-critic-loss.png" alt="Curve">

            *critic_loss — architecture de critic originale vs. catégorielle*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — architecture de critic originale vs. catégorielle*
        </td>
    </tr>
</table>

**Un second bug indépendant trouvé et corrigé : le timing de la récompense/terminaison.** Une revue plus approfondie de la boucle d'entraînement en ligne a révélé que le signal de récompense et de terminaison d'épisode de l'environnement était lu *avant* que l'action de ce step ne soit réellement exécutée, plutôt qu'après — ce qui signifie que chaque transition stockée était associée au résultat de l'action *précédente* plutôt qu'à celle qu'elle était réellement censée représenter. Ceci a été confirmé comme un décalage structurel réel (pas simplement une convention de nommage) via une petite reproduction contrôlée en dehors du codebase principal, et retracé comme n'affectant que la boucle de collecte de données RL en ligne — le fine-tuning supervisé, la génération de démonstrations, et le pretraining du critic utilisent tous des pipelines de données séparés qui n'ont jamais été affectés. Les tâches utilisées dans ce projet reposent sur un signal de récompense façonné (dense), pas sparse, ce qui limite l'ampleur de la perturbation que ce décalage temporel pouvait causer sur la partie du signal variant en douceur — mais le bonus de succès discret superposé à ce terme de façonnage est exactement le genre de valeur qu'un tel décalage attribuerait le plus mal.

Le fix a été appliqué, et une seconde comparaison contrôlée avant/après a été menée dans des conditions par ailleurs identiques. **Résultat :** le même schéma que le changement d'architecture ci-dessus — `target_q_max` se stabilise nettement par rapport à la version non corrigée, mais `eval/success_rate` s'effondre à nouveau essentiellement de la même façon quoi qu'il en soit.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/reward-timing-fix-comparison-target-q-max.png" alt="Curve">

            *target_q_max — avant vs. après le fix de timing de récompense*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/reward-timing-fix-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — avant vs. après le fix de timing de récompense*
        </td>
    </tr>
</table>

**Pourquoi le papier original ne rapporte-t-il pas ce problème ?** En revisitant la configuration robot réel du papier pour une explication : elle repose sur un opérateur humain corrigeant activement le robot en temps réel pendant l'entraînement (surtout en début d'entraînement), ce qui agit comme un coupe-circuit continu sur exactement le mode de défaillance décrit ci-dessus — une action mauvaise et surestimée n'a jamais l'occasion d'être renforcée, puisqu'elle est corrigée avant de pouvoir être enregistrée. Leurs runs d'entraînement sont aussi considérablement plus courts (de l'ordre de 8–20k steps, contre 100k+ ici), et les logs de ce projet montrent que l'instabilité ne devient sévère que bien au-delà de cette plage. Le mécanisme sous-jacent dans la formulation du papier lui-même est identique à celui utilisé ici — rien dans l'algorithme lui-même ne semble le rendre immunisé — ce qui pointe vers le filet de sécurité humain du papier et son horizon d'entraînement plus court comme explication la plus probable, plutôt qu'une différence fondamentale dans la méthode.

**Hypothèse principale actuelle et prochaines étapes.** Avec deux fixes indépendants, réels et confirmés, améliorant chacun quelque chose de mesurable dans le critic sans résoudre le symptôme réel, le mécanisme de sélection lui-même (le même critic proposant *et* jugeant sa propre candidate préférée) est l'explication restante la plus probable, par élimination. Deux directions complémentaires du papier XQCfD sont testées dans l'ordre, pour garder les résultats attribuables à une seule méthodologie citable avant de considérer un fix alternatif issu d'une autre ligne de recherche (découpler les étapes de sélection et d'évaluation, façon Double DQN, en utilisant les réseaux critic online et target déjà présents dans l'architecture) :

- **La régularisation KL** contre la politique pré-entraînée, remplaçant le bonus d'exploration générique utilisé par défaut — calculée en forme close entre la propre distribution de la politique résiduelle (avant compression tanh) et une distribution de référence fixe représentant "aucune correction". Un premier test (pas encore complètement isolé — voir plus bas) a déjà montré le résultat le plus encourageant de toute cette investigation : le taux de succès est resté systématiquement 5 à 10 points de pourcentage au-dessus d'un run comparable sans cette régularisation, pendant l'essentiel de l'entraînement, et le terme de régularisation lui-même est resté une contrainte active et non négligeable tout du long plutôt que d'être noyé par le bonus d'exploration actif en parallèle dans ce premier test. Un test isolé (avec le bonus d'exploration entièrement désactivé, correspondant exactement à la formulation du papier) est en cours pour confirmer la part de ce gain réellement attribuable au terme de régularisation.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/kl-regularization-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — avec vs. sans régularisation KL (premier test, non isolé)*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/kl-regularization-kl-penalty.png" alt="Curve">

            *Le terme de régularisation KL lui-même au cours de l'entraînement*
        </td>
    </tr>
</table>

- **Une architecture de politique "stationnaire"** (pas encore implémentée), la pièce restante de la recette XQCfD, destinée à faire en sorte que la politique adopte par défaut un comportement conservateur et incertain quand elle agit en dehors de la région réellement couverte par les démonstrations, plutôt que de continuer à agir avec assurance.

L'architecture originale du critic (scalaire) a aussi été rendue directement sélectionnable (pas seulement préservée comme référence passive) pour pouvoir être relancée dans exactement les mêmes conditions que la version catégorielle sur demande, pour comparaison supplémentaire.

### Phase 2 — Étendre la même rigueur aux baselines PPO/GRPO

Des baselines on-policy (PPO, GRPO) ont été ajoutées au même codebase pour fournir des points de comparaison supplémentaires, partageant la même boucle d'entraînement et la même infrastructure qu'ExpoFT. Exercer ce code pour la première fois a mis au jour plusieurs bugs numériques indépendants et confirmés — aucun n'avait été détecté auparavant, puisque ce chemin de code n'avait encore jamais été exécuté jusqu'au bout :

- Une incompatibilité de construction de réseau entre le réseau de fonction de valeur et une pièce d'infrastructure partagée qui supposait une forme de réseau différente (ensemble)
- Une étape de post-traitement d'action qui paddait la dimension de l'action deux fois, une fois de trop
- Une vérification de repli incorrecte pour une quantité (l'entropie de la politique) qui n'a pas de forme close pour la distribution d'action de ce projet, empruntant silencieusement la mauvaise branche de code et plantant au lieu d'utiliser l'approximation par échantillon déjà implémentée pour exactement cette situation
- Un terme de ratio non clippé dans l'objectif PPO/GRPO qui pouvait, après suffisamment de dérive de la politique, atteindre silencieusement l'infini en virgule flottante et corrompre la loss de tout un batch
- Une instabilité numérique à la frontière exacte de l'espace d'action (compressé par tanh), rencontrée lors du rejeu de certaines actions stockées, qui pouvait de façon similaire corrompre un pas de gradient

Chacun de ces éléments a été confirmé par un test ciblé et reproductible construit spécifiquement pour l'isoler, plutôt que corrigé sur simple inspection du code. Une protection défensive a aussi été ajoutée par-dessus l'ensemble : si un gradient calculé s'avère non fini pour une raison quelconque, cette mise à jour particulière est désormais entièrement sautée (laissant le modèle et l'état de l'optimiseur intacts) plutôt que de pouvoir corrompre silencieusement l'entraînement, avec un logging de diagnostic ajouté à chaque étape du calcul pour qu'une future occurrence soit immédiatement traçable plutôt que de se présenter comme un simple crash bien plus tard.

**Une découverte plus fondamentale.** L'investigation de pourquoi ces baselines ne montraient aucun signal d'apprentissage dès le tout début de l'entraînement (plutôt que le schéma "commence bien, puis se dégrade" observé avec ExpoFT) a révélé que leur réseau de politique entraînable n'avait en réalité jamais été connecté au checkpoint pré-entraîné par supervision : il s'agit d'un petit réseau séparé, initialisé aléatoirement, le checkpoint pré-entraîné n'étant utilisé que pour convertir ses entrées et sorties vers et depuis le bon format — pas pour lui donner quoi que ce soit du comportement de la politique pré-entraînée comme point de départ. C'est une conséquence raisonnable de l'utilisation d'un modèle de base à flow-matching, qui n'expose pas directement le type de vraisemblance d'action dont ces deux algorithmes ont besoin, mais cela signifiait que ces baselines devaient en réalité réapprendre la manipulation depuis zéro, reproduisant la propre découverte de la Phase 1 de ce projet (le RL on-policy ne peut pas s'amorcer sans une politique de départ compétente) alors même que l'intention était de tester le fine-tuning on-policy *à partir* du checkpoint SFT. Une option de warmup par clonage comportemental, à l'image de l'idée de pretraining du critic utilisée pour ExpoFT, a été ajoutée pour ces baselines afin de combler cet écart avant de tirer d'autres conclusions à partir d'elles.

---

## Difficultés majeures rencontrées

### Goulot d'étranglement d'exploration (Phase 1, fondamental)
Les algorithmes on-policy (PPO, GRPO) ne peuvent pas générer de signal d'apprentissage quand la politique de base a ~0% de succès. Ceci s'applique à LIBERO-Long (11.9% de base SFT) et directement à StackCube avec le modèle OpenVLA de base. Aucune stratégie d'exploration (température, bonus d'entropie) n'est suffisante sans un warmup SFT préalable ou des démonstrations — ceci a directement motivé le pivot vers ExpoFT.

### Décalage d'espace d'action et le MAJOR RESET (Phase 2)
La plus grande source d'effort perdu en Phase 2 : une incohérence de contrôle cartésien vs. articulaire produisait des symptômes (succès proche de 0%, bras approchant sans compléter la tâche) identiques en apparence à un véritable plafond de difficulté de tâche. Le diagnostiquer a nécessité de construire un outillage de validation par replay rigoureux et ancré dans la vérité terrain, plutôt que de faire confiance aux seules courbes d'entraînement. Plusieurs des bugs supplémentaires trouvés durant la revue systématique qui a suivi étaient silencieux — ils ne faisaient rien planter, ils produisaient simplement des chiffres plausibles mais faux — ce qui a rendu une revue systématique complète nécessaire plutôt qu'un fix ciblé.

### Incompatibilité de design de la tâche PickCube (résolue)
Un mode de défaillance qui ressemblait d'abord à un problème de politique ou d'exploration (0% de succès, aucune amélioration) s'est révélé être un manque fondamental d'observabilité entre le design de la tâche (marqueur d'objectif caché) et le schéma d'observation imitant DROID du projet. A nécessité des diagnostics au niveau caméra pour être correctement retracé, et PickCube-v1 a été retirée dans un premier temps en conséquence. Un monkeypatch ultérieur révélant le marqueur d'objectif à la caméra capteur a résolu le manque sans compromettre le schéma d'observation, et la tâche a été réintégrée. Leçon généralisée : vérifier l'alignement caméra/observabilité du VLA avec le design d'une tâche ManiSkill donnée (marqueurs d'objectif cachés, angle de caméra fixe, présence de caméra au poignet) avant de supposer que de mauvais résultats reflètent la difficulté de la tâche — et avant de supposer que l'incompatibilité est infixable.

### Instabilité du critic RL (Phase 2, difficulté centrale — considérablement précisée depuis son premier signalement)
Chaque configuration RL testée jusqu'ici dégrade plutôt qu'elle n'améliore la politique SFT validée, la dégradation étant initialement corrélée à une instabilité croissante de la critic loss. Deux fixes indépendants et réels — une architecture de critic reconstruite avec des estimations de valeur structurellement bornées, et un bug de timing de récompense/terminaison corrigé dans la boucle de collecte de données — ont chacun été confirmés comme stabilisant significativement la dynamique d'entraînement propre du critic, mais aucun n'a changé l'effondrement de la performance de la politique. Ceci a déplacé l'hypothèse principale de l'instabilité d'entraînement du critic elle-même vers le mécanisme de sélection d'action (le même critic proposant et jugeant sa propre action préférée), désormais testé directement plutôt qu'inféré par élimination. Une leçon utile, quoique d'abord contre-intuitive, de cette période de travail : un critic qui a l'air plus sain n'est pas la même chose qu'une meilleure politique, et les deux doivent être vérifiés séparément plutôt que supposés évoluer ensemble.

### Premier test de régularisation confondu (résolu, leçon de processus)
Le premier test de régularisation KL contre la politique pré-entraînée a été lancé avec le bonus d'exploration par défaut encore actif en parallèle, plutôt que désactivé comme le sous-entend la formulation du papier source lui-même — ce qui signifie que le résultat encourageant de ce run ne pouvait pas encore être attribué au terme de régularisation spécifiquement. Repéré avant d'en tirer une conclusion ; un test isolé de suivi est en cours. Leçon générale : lors de l'ajout d'un terme à un objectif existant, vérifier explicitement chaque autre terme avec lequel il pourrait interagir avant de rapporter un résultat, plutôt que de supposer qu'une nouvelle option est indépendante de ce qui existe déjà.

### Baselines PPO/GRPO silencieusement déconnectées de la politique pré-entraînée (résolu)
Ces baselines étaient supposées continuer à partir du checkpoint pré-entraîné par supervision, de la même façon que la méthode ExpoFT principale — ce n'était pas le cas. Leur réseau de politique entraînable était un réseau séparé, initialisé aléatoirement, depuis le début, le checkpoint pré-entraîné n'étant utilisé que pour le formatage des entrées/sorties. Ceci n'a été détecté par aucun test réussi ni aucun run plantant ; cela n'a émergé qu'une fois les *résultats* de ces baselines examinés de façon critique (performance proche de zéro persistante dès le tout premier step) plutôt que le code supposé correct parce qu'il s'exécutait sans erreur.

### Infrastructure — cluster, Ray, et stockage (Phase 1 et 2)
- Modèle chargé deux fois en mémoire GPU (RolloutGroup + ActorGroup) → OOM sur L40S (44GB) → passage à A100L (80GB) avec `enable_offload: True` (Phase 1)
- Clusters Ray résiduels d'autres utilisateurs bloquant l'initialisation sur plusieurs nœuds → résolu avec `unset RAY_ADDRESS` et exclusion de nœuds via `#SBATCH --exclude` (Phase 1)
- Répertoire `$HOME` trop petit pour les checkpoints (~50GB chacun) → projet migré vers `$SCRATCH`, puis re-migré vers `$HOME` avec seulement les artefacts volumineux régénérables en lien symbolique vers `$SCRATCH`, selon les recommandations de stockage mises à jour de Mila (Phase 2)
- Conflit de dépendance `robosuite` entre LIBERO et RoboCasa → résolu avec deux venvs séparés (Phase 1)
- Crashs OOM GPU JAX pendant l'entraînement RL, retracés à un problème de recompilation/structure pytree et à la pré-allocation mémoire par défaut de 75% de JAX (Phase 2)

---

## Ajustements et décisions clés

| Décision | Justification |
|---|---|
| L40S abandonné pour A100L (80GB) | VRAM du L40S insuffisante pour des modèles chargés deux fois en mémoire |
| Multi-GPU abandonné pour GPU unique + `enable_offload` | Problèmes de timeout NCCL en multi-GPU ; un seul GPU suffisant avec l'offloading |
| LIBERO-90 abandonné | Trop de tâches simultanées ; signal d'apprentissage peu focalisé |
| OpenVLA-OFT abandonné comme modèle principal | Pivot vers π₀ puis π₀.₅ pour ExpoFT (mieux adapté à la question de recherche) |
| LIBERO / RoboCasa abandonnés comme benchmarks principaux | Partiellement saturés par le SFT ; potentiel OOD limité |
| RL on-policy depuis le modèle de base abandonné | 0% de succès sur StackCube ; l'on-policy ne peut pas s'amorcer sans SFT ni démonstrations |
| ManiSkill sélectionné (StackCube, PushCube, PickCube) | Maximalement OOD par rapport aux données de pretraining DROID de π₀.₅ |
| Pivot vers l'architecture résiduelle ExpoFT | Fige le VLA, entraîne une petite politique résiduelle via une Q-function ; évite la limitation de log-probabilité des modèles à flow-matching ; contourne le problème d'exploration à froid via les démonstrations |
| Reproduire ExpoFT dans un dépôt ManiSkill dédié, pas dans RLinf | Garde le code de l'algorithme original proche de son état intact ; seule la couche environnement/données est remplacée |
| `openpi` et `ManiSkill` en submodules git, pas en installations `pip` | Permet des modifications directes et trackées plutôt que des monkeypatches à l'exécution |
| ExpoFT complet tenté d'abord, composants retirés ensuite | Méthodologie de retrait progressif pour identifier la supervision minimale réellement nécessaire, plutôt que de deviner à l'avance |
| Bascule vers le contrôle en espace articulaire de bout en bout | Correspond à l'espace d'action du pretraining DROID de π₀.₅ ; le contrôle cartésien écrêtait silencieusement la plupart des actions de démonstration |
| Construction de `validate_demos_full_pipeline.py` | Validation de replay ancrée dans la vérité terrain, aux seeds exactes, nécessaire après que le bug d'espace d'action ait montré que les courbes d'entraînement seules ne sont pas fiables |
| Erreur standard (pas intervalle de Wilson) pour toutes les barres d'erreur | Selon la préférence explicite de l'équipe (Jesse) |
| PickCube-v1 retirée (premier passage) | Le design de tâche à marqueur d'objectif caché semblait d'abord incompatible avec le schéma d'observation imitant DROID (image + proprioception uniquement) |
| PickCube-v1 réintégrée | Un monkeypatch révélant le marqueur d'objectif à la caméra capteur a restauré un accès visuel réel à la position de l'objectif sans ajouter de canal d'état privilégié — l'incompatibilité était corrigeable, pas fondamentale |
| Dépôt déplacé de `$SCRATCH` uniquement vers `$HOME` + artefacts en lien symbolique | Suit les recommandations de stockage mises à jour de Mila ; garde uniquement les données volumineuses et régénérables sur le scratch auto-nettoyé |
| `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95` adopté comme défaut permanent | Résout les crashs OOM récurrents de l'entraînement RL liés à la pré-allocation mémoire par défaut de 75% de JAX |
| Défaut de travail `utd_ratio=2` (vs. 20 dans le papier) | Le ratio du papier lui-même déstabilise l'entraînement encore plus vite que le ratio réduit dans cette configuration simulée |
| Runs RL catastrophiques d'avant les fixes supprimés de `$SCRATCH` | Libère de l'espace disque ; ne garde que les checkpoints SFT validés et les runs RL post-fix |
| Critic reconstruit en architecture catégorielle (à support borné), selon XQC/XQCfD | Teste directement si la croissance non bornée de la valeur du critic était la cause racine de la dégradation de la politique, plutôt que de continuer à chercher des hyperparamètres autour de l'architecture originale |
| Architecture de critic originale préservée dans un module séparé toujours exécutable plutôt qu'écrasée | Garde une comparaison A/B propre contre la refonte catégorielle disponible à tout moment, plutôt qu'une migration à sens unique |
| Normalisation adaptative (par estimation courante) de la récompense plutôt que des bornes de valeur fixes ajustées à la main | Garde le support fixe du critic catégoriel pertinent à travers les tâches sans ajustement par tâche, et sans besoin de reprojeter le support en cours d'entraînement |
| Tester les mitigations propres au papier XQCfD (pretraining du critic, critic catégoriel, régularisation KL, architecture stationnaire) dans l'ordre, avant un fix alternatif issu d'une autre ligne de recherche | Garde les résultats attribuables à une seule méthodologie citable plutôt que de mélanger des approches issues de papiers non liés |
| Régularisation KL implémentée comme un terme additionnel et activable plutôt qu'un remplacement strict du bonus d'exploration existant | Préserve la capacité de reproduire la formulation exacte du papier (en désactivant l'autre terme) tout en gardant les deux disponibles indépendamment pour l'ablation |
| Ajout d'une option de warmup par clonage comportemental pour les baselines PPO/GRPO | Répond à la découverte que leur politique entraînable n'avait aucune connexion au checkpoint pré-entraîné autrement |
| Ajout d'une garde défensive contre les gradients non finis à l'étape de mise à jour PPO/GRPO | Empêche un futur problème numérique encore inconnu de corrompre silencieusement un run d'entraînement, indépendamment du fait que sa cause racine ait été trouvée |

---

## Figures, visuels, et démonstrations

Les courbes d'entraînement clés, vidéos d'épisodes, et poids de checkpoints pour la Phase 1 sont intégrés directement dans chaque section d'expérience ci-dessus. Les poids finaux pour le run RoboCasa CloseDrawer PPO sont disponibles sur HuggingFace : [`josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO`](https://huggingface.co/josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO).

Les figures et vidéos de la Phase 2 sont aussi intégrées dans les descriptions d'expériences ci-dessus. Les poids post-SFT de Pi0.5 sur les tâches maniskill sont disponibles sur HuggingFace : 

- [`josh11234/ExpoFT-Pi05-StackCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-StackCube-v1-SFT) (41% de succès sur 200 seeds held-out)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p) (62% de succès sur 200 seeds held-out)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p) (86% de succès sur 200 seeds held-out)
- [`josh11234/ExpoFT-Pi05-PickCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-PickCube-v1-SFT) (22% de succès sur 200 seeds held-out)

Les figures de comparaison d'architecture de critic, de fix de timing de récompense, et de régularisation KL référencées dans les dernières mises à jour ci-dessus restent à ajouter — les placeholders utilisent la même convention `images/` que le reste de cette page.---
title: Travail réalisé
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Implémentation

## Architecture et approche générale

### Objectif de recherche

La question centrale de ce projet est de savoir si le RL fine-tuning peut permettre une **généralisation out-of-distribution (OOD)** chez les modèles Vision-Language-Action (VLA) — c'est-à-dire permettre à un modèle de réussir des tâches maximalement éloignées de sa distribution d'entraînement SFT. Une question secondaire est de savoir si le principal goulot d'étranglement pour l'amélioration par RL est l'**exploration** (la politique n'arrive pas à découvrir de signal de récompense) ou l'**optimisation** (la politique n'arrive pas à s'améliorer à partir d'un signal donné).

### Vue d'ensemble du pipeline

Le projet s'est déroulé en deux grandes phases :

**Phase 1 — RL on-policy (Semaines 1–7) :** Fine-tuning RL direct via GRPO/PPO sur OpenVLA-OFT et π₀ dans les environnements LIBERO et RoboCasa. Cette phase a établi l'hypothèse du goulot d'étranglement d'exploration et les limites des approches on-policy sans warmup SFT.

**Phase 2 — Politique résiduelle ExpoFT (Semaine 8–présent) :** Reproduction et adaptation de l'[architecture ExpoFT](https://pd-perry.github.io/expo-ft/) dans un codebase dédié, piloté entièrement par la simulation ManiSkill (`expo-ft`), remplaçant le pipeline de données original d'ExpoFT (robot réel, DROID) tout en gardant initialement l'algorithme lui-même intact, avant de s'en écarter de façon ciblée une fois l'instabilité diagnostiquée (voir plus bas). Le VLA (π₀.₅) est gardé **figé**, et une petite politique de correction résiduelle est entraînée par-dessus via une Q-function, en combinant RL off-policy et démonstrations par motion planning. Ce design évite deux limitations de la Phase 1 : il ne nécessite pas de log-probabilités d'action valides (que les VLA à flow-matching comme π₀/π₀.₅ ne peuvent pas fournir, excluant le policy-gradient RL standard directement sur le modèle de base) et il contourne le problème d'exploration à froid en amorçant le replay buffer avec des démonstrations. Des baselines on-policy (PPO, GRPO) et une baseline off-policy sans le schéma résidu/argmax (SAC) ont ensuite été ajoutées au même codebase pour comparaison directe (voir plus bas).

La méthodologie retenue est le **retrait progressif de composants** : partir de la recette ExpoFT complète (warmup SFT + démonstrations + RL off-policy) et valider rigoureusement chaque étage avant de retirer des composants (taille du dataset, warmup SFT lui-même) pour identifier la supervision minimale réellement nécessaire.

### Pourquoi OpenVLA / π₀.₅ plutôt que des VLA à flow-matching

OpenVLA(-OFT) et π₀.₅ utilisent des **tokens d'action discrets**, qui produisent des log-probabilités valides et permettent donc un policy-gradient RL principiel (GRPO/PPO en Phase 1). Les modèles à flow-matching (π₀, MoMaLM) ne peuvent pas fournir de vraisemblances exploitables, ce qui polluerait le design expérimental de toute approche reposant sur des gradients de politique exacts. C'est aussi pourquoi la formulation en politique résiduelle d'ExpoFT — qui contourne entièrement le besoin de log-probabilités du VLA en agissant comme une correction additive apprise via une Q-function — est un moyen attractif de continuer à utiliser un VLA figé et compétent sans être bloqué par cette limitation.

### Environnement de travail

- **Cluster :** Mila HPC (SLURM), GPU A100/A100L (40GB/80GB)
- **Modèles testés :** OpenVLA-OFT (7B), π₀ (3B) (Phase 1) ; π₀.₅ (checkpoint `pi05_droid_jointpos`, figé) + politique résiduelle (Phase 2)
- **Environnements :** LIBERO (Spatial, Long), RoboCasa (CloseDrawer) — Phase 1 ; ManiSkill (StackCube-v1, PushCube-v1, PickCube-v1) — Phase 2
- **Algorithmes RL :** GRPO, PPO (Phase 1) ; ExpoFT (VLA figé + politique résiduelle, Q-learning off-policy via un ensemble de critics façon REDQ au départ, puis un critic catégoriel/distributionnel — voir plus bas), plus des baselines PPO, GRPO et SAC ajoutées directement au même codebase pour comparaison (Phase 2)
- **Framework :** RLinf (`montrealrobotics/RLinf`) pour la Phase 1 ; dépôt autonome `expo-ft` (fork propre d'`openpi` et de `ManiSkill` en submodules git) pour la Phase 2
- **Infrastructure clé :** Quatre venvs Python séparés pour gérer les conflits de dépendances ; dépôt vivant sur `$HOME` avec `logs/` et `demos/` en lien symbolique vers `$SCRATCH` ; gros caches de paquets déplacés vers `$SCRATCH`

---

## Expériences et analyses réalisées

### Phase 1 — Baselines RL on-policy

#### LIBERO-90 GRPO (OpenVLA-OFT)
Première expérience. 1000 steps (~31h40). `success_once` a montré des pics occasionnels (~0.13) sans tendance stable à la hausse. D'abord attribué à une petite batch size ; confirmé ensuite comme un goulot d'étranglement d'exploration.

![LIBERO-90 GRPO Training Results Success Once]({{ config.site_url }}/images/libero90_grpo_training1_success_once.png)
![LIBERO-90 GRPO Training Results Approx KL]({{ config.site_url }}/images/libero90_grpo_training1_approx_kl.png)

#### RoboCasa CloseDrawer PPO (π₀)
Run de 150 steps avec LoRA (rank 32). Métriques finales : `success_once` ~0.82, `critic/value_loss` ~0.005, `approx_kl` ~0.036. A démontré que RL peut améliorer la performance quand la tâche est in-distribution — mais CloseDrawer était trop facile (base SFT déjà à ~0.6–0.7), laissant peu de marge de contribution pour RL.

![RoboCasa CloseDrawer PPO Training Results Success Once]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_success_once.png)
![RoboCasa CloseDrawer PPO Training Results Approx KL]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_approx_kl.png)
![RoboCasa CloseDrawer PPO Training Results Value Loss]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_value_loss.png)

<div style="text-align: center">
    <video width="480" controls poster="{{ config.site_url }}/images/robocasa_close_drawer_thumbnail.png">
        <source src="{{ config.site_url }}/images/robocasa_close_drawer_training1_ppo_step_150.mp4" type="video/mp4">
    </video>
</div>

*π₀ + RoboCasa CloseDrawer — épisode au step 150 (après entraînement PPO)*

#### LIBERO-Spatial GRPO (OpenVLA-OFT)
Run de ~38 steps. `success_once` a oscillé entre 0.22 et 0.875 sans convergence claire. Identifié comme un mauvais réglage du learning rate suite à une réduction du nombre d'epochs.

![LIBERO-Spatial GRPO Training Results Success Once]({{ config.site_url }}/images/libero_spatial_grpo_training1_success_once.png)
![LIBERO-Spatial GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_spatial_grpo_training1_approx_kl.png)

#### LIBERO-Long GRPO (OpenVLA-OFT)
~150 steps sur 4 sessions reprises (~37h au total). `success_once` a oscillé entre 0.06 et 0.19 sans tendance à la hausse. `approx_kl` est resté stable à ~0.035–0.048 tout du long — les mises à jour de la politique étaient contrôlées, confirmant que le goulot d'étranglement est l'**exploration**, pas une instabilité d'optimisation.

![LIBERO-Long GRPO Training Results Success Once]({{ config.site_url }}/images/libero_long_grpo_training1_success_once.png)
![LIBERO-Long GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_long_grpo_training1_approx_kl.png)

#### StackCube-v1 PPO et GRPO (base OpenVLA, on-policy, sans SFT)
Les deux runs ont produit 0% de succès. Sans warmup SFT, la politique de base ne peut produire aucun comportement pertinent pour la tâche, donc l'exploration on-policy ne peut pas amorcer de signal d'apprentissage. Ce résultat a directement motivé le pivot vers ExpoFT. Voici les résultats des runs PPO et GRPO sur StackCube-v1 :

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-ppo-nosft.mp4" type="video/mp4">
    </video>
</div>

*OpenVLA sur maniskill StackCube-v1 après PPO*


<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-grpo-nosft.mp4" type="video/mp4">
    </video>
</div>

*OpenVLA sur maniskill StackCube-v1 après GRPO*


### Sélection et visualisation du dataset OOD

- Création d'un notebook Jupyter pour l'analyse de distribution des points d'arrivée à travers les datasets Open X-Embodiment ([Colab](https://colab.research.google.com/github/Josh012006/OpenX-Embodiment-Datasets-Visualization/blob/main/colabs/Open_X_Embodiment_Datasets.ipynb), [Repo](https://github.com/Josh012006/OpenX-Embodiment-Datasets-Visualization))
- Un fit UMAP partagé a été utilisé pour tous les datasets (plutôt qu'un fit par dataset), puisqu'un fit par dataset rendrait les coordonnées incomparables entre datasets
- ManiSkill identifié comme benchmark OOD cible : des tâches comme StackCube-v1 et PushCube-v1 sont maximalement éloignées des données SFT/pretraining du VLA (DROID/OXE)
- Remarque : la distance L2 spatiale entre datasets peut être trompeuse à cause des référentiels de coordonnées différents entre datasets

### Exploration d'hyperparamètres — réduction du temps d'entraînement (Phase 1)

Test d'une augmentation du replay ratio (`rollout_epoch` 4→8 pour GRPO, `update_epoch` 2→4 pour PPO) pour réduire le temps réel. Résultat : le goulot d'étranglement s'est déplacé de la collecte de rollouts vers les mises à jour de gradient, rendant l'entraînement *plus lent* (~27 min/step contre ~14 min/step). L'approche efficace consiste plutôt à réduire directement `max_episode_steps` et `max_epochs`.

---

### Phase 2 — ExpoFT : mise en place du dépôt

L'algorithme ExpoFT a été reproduit dans un dépôt dédié, piloté par ManiSkill (`~/projects/expo-ft`) plutôt que dans RLinf, pour garder le code de l'algorithme original aussi intact que possible tout en ne remplaçant que la couche environnement/données. Deux dépendances — `openpi` (l'implémentation de π₀.₅) et `ManiSkill` — ont été forkées et intégrées comme véritables submodules git éditables (plutôt que des dépendances non trackées ou installées via `pip`), permettant des modifications directes et trackées plutôt que des monkeypatches à l'exécution. `.gitmodules` utilise des URLs HTTPS pour la compatibilité de clonage public, avec des remotes SSH configurés localement pour les push.

L'organisation du stockage suit les recommandations du cluster Mila : le dépôt lui-même vit sur `$HOME` (sauvegardé, 100GB), tandis que les artefacts volumineux et régénérables (`logs/`, `demos/`) sont en lien symbolique vers `$SCRATCH` (5TB, nettoyé automatiquement). Les gros caches de paquets (`uv`, `openpi`, `huggingface`, `pip`, `jax`, ~90GB au total) ont été déplacés vers `$SCRATCH/cache/` avec des liens symboliques vers `~/.cache`, libérant le quota `$HOME`.

### Phase 2 — Étage SFT : diagnostic et validation de l'espace d'action

Le premier run de warmup SFT (Semaine 8), sur des démonstrations StackCube-v1 converties au format DROID/LeRobot, a produit un taux de succès proche de 0% — le bras approchait seulement les cubes sans compléter l'empilement. Ceci a d'abord été suspecté d'être un plafond de difficulté réel de la tâche, mais une investigation systématique l'a retracé à un **décalage fondamental d'espace d'action** : le pretraining DROID de π₀.₅ attend des actions en position articulaire, en unités physiques, alors que le pipeline utilisait le contrôle cartésien `pd_ee_delta_pose` de ManiSkill, causant l'écrêtage silencieux de la majorité des actions de démonstration hors de leur plage valide.

Le fix — basculer l'ensemble du pipeline vers le contrôle en espace articulaire de bout en bout (`control_mode=pd_joint_delta_pos`, checkpoint `pi05_droid_jointpos`, état `observation/joint_position`, `normalize_action=True`, `output_action_dim=8`, stats de normalisation DROID depuis GCS) — a immédiatement ramené la grande majorité des actions dans leur plage valide lors de la validation par replay.

Ceci a déclenché une revue systématique plus large ("MAJOR RESET") qui a mis au jour un ensemble de bugs silencieux supplémentaires avant que les chiffres SFT puissent être considérés fiables :

- Un script d'évaluation écrasant sans condition les stats de normalisation DROID officielles, même pour l'évaluation de baseline
- Un artefact de rescaling d'action en espace cartésien, résidu du mode de contrôle abandonné, corrompant une fraction significative des actions désormais en espace articulaire
- Un décalage de longueur de troncature entre l'environnement et la boucle d'évaluation
- Une incompatibilité de backend de simulation empêchant la conversion correcte du mode de contrôle

Pour regagner confiance dans le pipeline de données, `validate_demos_full_pipeline.py` a été construit comme un outil de replay rigoureux de bout en bout (utilisant les vraies stats de normalisation, le vrai wrapper d'environnement, et les seeds exactement enregistrées). Après les fixes ci-dessus, cet outil a confirmé **100% de succès de replay des démos sur StackCube**, donnant une réelle confiance dans le fait que l'espace d'action et la conversion des données étaient enfin corrects.

Un outillage d'évaluation approprié a aussi été construit à cette étape : des seeds d'épisode fixes (200 seeds held-out, générées une fois à partir d'une seed maîtresse et réutilisées pour tous les checkpoints) pour une comparaison équitable entre checkpoints, des barres d'erreur standard (`SE = sqrt(p(1-p)/n)`, une approximation normale plutôt qu'un intervalle de Wilson, selon la préférence explicite de l'équipe), et un script de balayage de checkpoints reprenable qui reconstruit son graphique après chaque checkpoint.

Des balayages complets de checkpoints SFT sur ces 200 seeds fixes ont confirmé une **généralisation réelle et validée** (pas de la mémorisation) sur StackCube, avec des taux de succès croissants à travers les itérations SFT ; PushCube-v1 (ajoutée comme deuxième tâche, plus facile, pour vérifier la cohérence du pipeline) a obtenu des performances encore meilleures.

Voici les courbes de balayage de checkpoints SFT (taux de succès vs. step/itération d'entraînement, avec barres d'erreur SE, 200 seeds held-out) pour **StackCube-v1** et **PushCube-v1**.

![SFT Checkpoint Sweep Curves PushCube-v1]({{ config.site_url }}/images/curve-sft-pushcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pushcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur maniskill PushCube-v1 après warmup SFT*


![SFT Checkpoint Sweep Curves StackCube-v1]({{ config.site_url }}/images/curve-sft-stackcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur maniskill StackCube-v1 après warmup SFT*



Et voici des enregistrements d'épisodes du comportement du meilleur checkpoint SFT sur StackCube-v1 et PushCube-v1, pour accompagner visuellement les courbes de balayage ci-dessus (même style que la vidéo RoboCasa CloseDrawer en Phase 1).

### Phase 2 — Suite de tâches : StackCube, PushCube, et PickCube (retirée, puis réintégrée)

PickCube-v1 a été ajoutée comme troisième tâche pour diversifier davantage la suite OOD, mais montrait initialement un taux de succès persistant de 0%. Ceci a été retracé à une **incompatibilité de design de la tâche** plutôt qu'un problème de politique ou d'exploration : PickCube-v1 de ManiSkill cache le marqueur d'objectif (`goal_site`) aux caméras capteur par défaut, et bien que `obs['extra']['goal_pos']` soit techniquement accessible dans le simulateur, le schéma d'observation imitant DROID utilisé tout au long de ce projet (image + proprioception uniquement, correspondant à ce qu'un robot réel aurait) n'avait aucun canal pour cette information. La politique n'avait donc aucun accès à la position de l'objectif par épisode — ni visuellement ni numériquement — rendant la tâche apparemment insoluble.

PickCube-v1 a été retirée dans un premier temps plutôt que contournée avec un canal d'état privilégié, puisque cela ne correspondrait pas à ce qu'un robot réel pourrait percevoir. Cependant, un monkeypatch révélant `goal_site` à la caméra capteur elle-même — donnant à la politique un accès *visuel* réel à la position de l'objectif, exactement comme une vraie caméra pointée sur une cible visible le ferait — a ensuite été validé avec succès. Ceci a résolu le manque d'observabilité sans compromettre le schéma d'observation imitant DROID, et **PickCube-v1 a été réintégrée dans la suite de tâches active**.

J'ai donc aussi lancé un warmup SFT sur PickCube-v1. Voici les performances :

![SFT Checkpoint Sweep Curves PickCube-v1]({{ config.site_url }}/images/curve-sft-pickcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pickcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur maniskill PickCube-v1 après warmup SFT*


**Suite de tâches active à partir de maintenant : StackCube-v1, PushCube-v1, et PickCube-v1.**

### Phase 2 — Étage RL : revue de code et fixes d'infrastructure

Avant de faire confiance à un quelconque résultat RL, l'étage d'entraînement RL a reçu le même niveau de revue ligne par ligne que l'étage SFT. Ceci a mis au jour un autre ensemble de vrais bugs :

- Un bug d'écrasement des stats de normalisation identique à celui trouvé précédemment dans l'évaluation SFT
- Environ 15 flags CLI passés au script d'entraînement mais jamais lus par celui-ci (hyperparamètres orphelins — présents dans les configs, jamais réellement consommés par le code)
- Un flag d'écrasement de checkpoint codé en dur qui faisait planter chaque run frais (non repris)
- Une incohérence entre deux copies différentes d'un flag d'entraînement clé, causant un crash dès que sa valeur différait du défaut
- Plusieurs hyperparamètres RL définis dans les configs YAML de tâche mais jamais réellement lus par le script d'entraînement, retombant silencieusement sur des défauts différents définis ailleurs dans le codebase
- La valeur d'un hyperparamètre silencieusement interprétée comme du texte plutôt qu'un float par le lecteur de config (PyYAML interprète la notation scientifique nue comme `3e-4` comme une chaîne de caractères, pas un float)

Séparément, `eval_policy.py` a été trouvé avec `only_base_actions=True` par défaut, ce qui signifie qu'il évaluait silencieusement uniquement la politique VLA de base figée — jamais la politique résiduelle entraînée — sur chaque checkpoint RL évalué jusque-là. Un support approprié pour charger et évaluer l'agent RL complet entraîné (politique résiduelle et critic ensemble) a été construit et utilisé pour toutes les évaluations à partir de ce point.

Deux causes distinctes de crashs répétés par manque de mémoire (OOM) pendant l'entraînement RL ont aussi été diagnostiquées et corrigées :
- Un problème de recompilation JAX déclenché par une forme d'entrée incohérente à mi-parcours de l'entraînement (un basculement de structure pytree `None`→dict pour `actor_batch` à travers une frontière `jax.jit` forçait JAX à compiler et garder résident silencieusement un second programme XLA)
- L'allocateur mémoire GPU par défaut de JAX ne réservant que 75% de la carte par défaut, plutôt que la carte entière — résolu avec `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95`

Enfin, le logging TensorBoard en cours d'entraînement s'est révélé silencieusement dépourvu de la plupart des métriques d'entraînement (seuls le taux de succès et le temps de boucle étaient visibles, alors que wandb affichait tout correctement) à cause d'un problème de filtrage par type — les scalaires JAX `jnp.float32` échouent aux vérifications `isinstance(v, (int, float))` de Python. Corrigé avec un cast explicite `float()`.

La recette exacte d'augmentation de données d'entraînement du papier a été reproduite à partir de sa description et confirmée comme déjà correctement implémentée dans le codebase.

### Phase 2 — Étage RL : la découverte centrale d'instabilité

Une fois les bugs d'infrastructure et d'évaluation corrigés, et PickCube-v1 réintégrée après son fix d'observabilité, les premiers vrais runs de fine-tuning RL ont été lancés à partir des checkpoints SFT validés sur les trois tâches actives. Des courbes d'évaluation complètes (200 seeds fixes held-out par checkpoint) ont ensuite été produites pour une batterie d'expériences comparatives faisant varier le **ratio update-to-data (UTD)** et la proportion de chaque batch d'entraînement tirée des données de démonstration, incluant un run reproduisant les hyperparamètres exacts du papier dans la durée d'entraînement validée par le papier lui-même.

**Résultat clé :** chaque configuration RL testée jusqu'ici — sur les trois tâches actives (StackCube, PushCube, PickCube) et chaque réglage de ratio UTD / proportion de démonstration testé — **dégrade la politique pré-entraînée par SFT plutôt que de l'améliorer** :

- `utd_ratio=10` : effondrement catastrophique en ~2–5k steps
- `utd_ratio=2` + `offline_ratio=0.5` : retarde l'effondrement à ~35–40k steps, se dégradant finalement à ~0.3–0.4 de succès vers 85–90k steps
- `utd_ratio=20` (la valeur exacte du papier, exécutée dans la plage de steps validée par le papier lui-même, ~20–24k steps) : s'est effondré **plus vite et plus sévèrement** que le long run à `utd_ratio=2` — ceci réfute l'hypothèse qu'entraîner simplement dans le budget plus court du papier (robot réel) éviterait le problème ; `utd_ratio` lui-même, pas la durée totale d'entraînement, semble être le facteur dominant de l'instabilité

La dégradation est systématiquement précédée et accompagnée d'une **instabilité croissante de la loss d'entraînement du critic** (spikes de critic loss progressifs), suggérant que la cause racine est une fonction de valeur se déstabilisant progressivement plutôt qu'un bug de pipeline — ceci a été confirmé davantage en vérifiant que l'évaluation utilise démontrablement la politique résiduelle et le critic entraînés, sans retomber silencieusement sur la politique de base figée (voir le fix d'`eval_policy.py` ci-dessus).

Correspondance d'hyperparamètres confirmée avec le papier original : `offline_ratio=0` (les démonstrations sont placées dans l'unique replay buffer en ligne, pas dans un buffer hors-ligne séparé à 50/50 — cette convention appartient à RLPD, pas à ce papier), `hidden_dims=3×256`, `init_temperature=1.0`, `num_qs=10` / `num_min_qs=2` (ensemble de critics façon REDQ), `tau=0.005`. Écarts connus par rapport au papier : `batch_size=24` (le papier utilise 64 ; contraint par la mémoire GPU disponible) et un `edit_scale=0.2` fixe (le papier utilise 0.05–0.2, ajusté selon la difficulté de la tâche).

Voici une comparaison des métriques d'entraînement à travers les trois configurations de ratio UTD décrites ci-dessus, pour PushCube-v1, la plus facile des trois tâches :

*Évolution de la critic loss pendant l'entraînement* :

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    <tr>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


*Évolution de la residual actor loss pendant l'entraînement* :
<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


*Évolution du taux de succès en évaluation pendant l'entraînement* :
<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


Et voici une vidéo montrant la dégradation du comportement de la politique sur StackCube-v1 après ~120k steps de fine-tuning RL :
<table>
    <tr>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
                </video>
            </div>

            *Avant fine-tuning RL avec ExpoFT*
        </td>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-rl-degradation.mp4" type="video/mp4">
                </video>
            </div>

            *Après fine-tuning RL avec ExpoFT*
        </td>
    </tr>
</table>

**Question de recherche ouverte**, soumise au groupe pour discussion : est-ce une limitation fondamentale de l'entraînement de critic off-policy à UTD élevé, que les runs courts du papier sur robot réel n'exposent tout simplement jamais faute de durée suffisante, ou est-ce spécifique à cette configuration simulée (dataset plus petit, architecture VLA-figé + politique résiduelle, ou batch size contrainte par la mémoire) ? Ceci porte directement sur la question de recherche centrale du projet, puisque cela bloque actuellement toute démonstration d'une contribution RL positive par-dessus la baseline SFT validée.

**Pistes non testées identifiées pour la suite :** un `edit_scale` plus petit (par ex. 0.05) spécifiquement pour StackCube, puisque c'est la tâche la plus exigeante en précision et celle qui s'est effondrée le plus dramatiquement ; `replan_steps=4` pour StackCube (actuellement fixé à 8, alors que le papier utilise 4–8 selon la tâche) ; et `batch_size=32` comme prochain test à faible risque (jamais encore essayé).

### Phase 2 — Étage RL : isoler l'instabilité du critic d'un biais structurel de sélection

La phase de travail suivante a repris les pistes ci-dessus et les a testées systématiquement une par une, tout en prenant du recul pour caractériser précisément *pourquoi* la loss d'entraînement du critic se déstabilisait en premier lieu, plutôt que de continuer à chercher des hyperparamètres.

**Mécanisme identifié.** ExpoFT sélectionne l'action exécutée en échantillonnant plusieurs actions candidates (la sortie du VLA de base elle-même, plus plusieurs variantes éditées par résidu), en notant toutes ces candidates avec le critic, et en prenant celle avec le meilleur score — et ce même critic sert ensuite à bootstrap sa propre cible d'entraînement pour la candidate choisie. C'est un biais classique de maximisation/auto-référence (étroitement lié à la raison d'être de Double Q-learning) : l'argmax va systématiquement préférer la candidate que le critic surestime le plus, et cette valeur surestimée devient ensuite une partie de la prochaine cible d'entraînement du critic lui-même, renforçant l'erreur. Ce mécanisme a été identifié comme la cause racine la plus probable de l'instabilité croissante de la critic loss décrite ci-dessus, et est devenu l'hypothèse organisatrice pour la suite de cette phase de travail.

**Premières mitigations testées, individuellement — aucune n'a résolu l'effondrement :**

- Une ablation à température fixe, le gel de l'encodeur d'image du critic, le gradient clipping sur le critic, et le weight decay du critic ont chacun été testés isolément comme stabilisateurs candidats. Aucun n'a changé le résultat qualitatif (`target_q_max` continuait de croître sans borne, `critic_loss` continuait de devenir de plus en plus erratique, `eval/success_rate` continuait de s'effondrer).
- Un `edit_scale` plus petit (0.05, suivant les recommandations du papier lui-même pour les tâches exigeant de la précision) a été testé spécifiquement sur StackCube. Même résultat.
- Le pretraining du critic — une phase de warmup, avant tout RL, qui entraîne uniquement le critic (jamais la politique résiduelle) sur des données de démonstration via le même mécanisme de Bellman backup utilisé ensuite — a été implémenté comme une option activable via YAML, suivant une idée similaire proposée dans le papier XQCfD (arXiv 2605.10734) pour la cohérence critic/actor. Testé isolément : n'a pas résolu l'effondrement. Notamment, `target_q_max` pendant cette phase de pretraining hors-ligne uniquement a *elle-même* recommencé à grimper à mi-parcours, sur des données de démonstration seules — une preuve que l'instabilité ne concerne pas principalement les données en ligne que voit le critic, mais le mécanisme d'argmax/bootstrap lui-même.

**Reconstruction de l'architecture du critic (XQC / XQCfD).** Pour tester si la dynamique d'entraînement du critic lui-même (par opposition au mécanisme de sélection) était le facteur principal, le critic a été reconstruit de zéro en suivant XQC (arXiv 2509.25174) et XQCfD (arXiv 2605.10734) : une tête **catégorielle (façon C51, à support borné fixe)** remplaçant la cible de régression scalaire non bornée originale, avec **batch normalization et weight normalization** sur le MLP du critic, et **aucun ensemble** (correspondant à la configuration rapportée par XQCfD elle-même). Le signal de récompense est normalisé par une estimation courante de sa propre échelle avant utilisation, pour que les bornes de valeur fixes restent pertinentes à travers les tâches sans les ajuster à la main pour chacune. L'architecture originale a été préservée, non modifiée, dans un module séparé plutôt qu'écrasée, spécifiquement pour que les deux puissent tourner côte à côte dans des conditions par ailleurs identiques.

**Résultat :** les propres estimations de valeur du critic catégoriel se comportent de façon qualitativement différente et plus saine — elles convergent et se stabilisent dans une plage bornée plutôt que de grimper indéfiniment, et la loss d'entraînement du critic reste lisse plutôt que de devenir de plus en plus erratique. Mais `eval/success_rate` s'effondre toujours essentiellement de la même façon. C'était un résultat négatif important : il écarte l'instabilité d'entraînement du critic elle-même comme facteur principal de la dégradation de la politique.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-target-q-max.png" alt="Curve">

            *target_q_max — architecture de critic originale (en jaune) vs. catégorielle (en marron)*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-critic-loss.png" alt="Curve">

            *critic_loss — architecture de critic originale (en jaune) vs. catégorielle (en marron)*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — architecture de critic originale (en jaune) vs. catégorielle (en marron)*
        </td>
    </tr>
</table>

**Un second bug indépendant trouvé et corrigé : le timing de la récompense/terminaison.** Une revue plus approfondie de la boucle d'entraînement en ligne a révélé que le signal de récompense et de terminaison d'épisode de l'environnement était lu *avant* que l'action de ce step ne soit réellement exécutée, plutôt qu'après — ce qui signifie que chaque transition stockée était associée au résultat de l'action *précédente* plutôt qu'à celle qu'elle était réellement censée représenter. Ceci a été confirmé comme un décalage structurel réel (pas simplement une convention de nommage) via une petite reproduction contrôlée en dehors du codebase principal, et retracé comme n'affectant que la boucle de collecte de données RL en ligne — le fine-tuning supervisé, la génération de démonstrations, et le pretraining du critic utilisent tous des pipelines de données séparés qui n'ont jamais été affectés. Les tâches utilisées dans ce projet reposent sur un signal de récompense façonné (dense), pas sparse, ce qui limite l'ampleur de la perturbation que ce décalage temporel pouvait causer sur la partie du signal variant en douceur — mais le bonus de succès discret superposé à ce terme de façonnage est exactement le genre de valeur qu'un tel décalage attribuerait le plus mal.

Le fix a été appliqué, et une seconde comparaison contrôlée avant/après a été menée dans des conditions par ailleurs identiques. **Résultat :** le même schéma que le changement d'architecture ci-dessus — `target_q_max` se stabilise nettement par rapport à la version non corrigée, mais `eval/success_rate` s'effondre à nouveau essentiellement de la même façon quoi qu'il en soit.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/reward-timing-fix-comparison-target-q-max.png" alt="Curve">

            *target_q_max — avant (en vert) vs. après (en rouge) le fix de timing de récompense*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/reward-timing-fix-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — avant (en vert) vs. après (en rouge) le fix de timing de récompense*
        </td>
    </tr>
</table>

**Pourquoi le papier original ne rapporte-t-il pas ce problème ?** En revisitant la configuration robot réel du papier pour une explication : elle repose sur un opérateur humain corrigeant activement le robot en temps réel pendant l'entraînement (surtout en début d'entraînement), ce qui agit comme un coupe-circuit continu sur exactement le mode de défaillance décrit ci-dessus — une action mauvaise et surestimée n'a jamais l'occasion d'être renforcée, puisqu'elle est corrigée avant de pouvoir être enregistrée. Leurs runs d'entraînement sont aussi considérablement plus courts (de l'ordre de 8–20k steps, contre 100k+ ici), et les logs de ce projet montrent que l'instabilité ne devient sévère que bien au-delà de cette plage. Le mécanisme sous-jacent dans la formulation du papier lui-même est identique à celui utilisé ici — rien dans l'algorithme lui-même ne semble le rendre immunisé — ce qui pointe vers le filet de sécurité humain du papier et son horizon d'entraînement plus court comme explication la plus probable, plutôt qu'une différence fondamentale dans la méthode.

**Hypothèse principale actuelle et prochaines étapes.** Avec deux fixes indépendants, réels et confirmés, améliorant chacun quelque chose de mesurable dans le critic sans résoudre le symptôme réel, le mécanisme de sélection lui-même (le même critic proposant *et* jugeant sa propre candidate préférée) est l'explication restante la plus probable, par élimination. Deux directions complémentaires du papier XQCfD sont testées dans l'ordre, pour garder les résultats attribuables à une seule méthodologie citable avant de considérer un fix alternatif issu d'une autre ligne de recherche (découpler les étapes de sélection et d'évaluation, façon Double DQN, en utilisant les réseaux critic online et target déjà présents dans l'architecture) :

- **La régularisation KL** contre la politique pré-entraînée, remplaçant le bonus d'exploration générique utilisé par défaut — calculée en forme close entre la propre distribution de la politique résiduelle (avant compression tanh) et une distribution de référence fixe représentant "aucune correction". Un premier test (pas encore complètement isolé — voir plus bas) a déjà montré le résultat le plus encourageant de toute cette investigation : le taux de succès est resté systématiquement 5 à 10 points de pourcentage au-dessus d'un run comparable sans cette régularisation, pendant l'essentiel de l'entraînement, et le terme de régularisation lui-même est resté une contrainte active et non négligeable tout du long plutôt que d'être noyé par le bonus d'exploration actif en parallèle dans ce premier test. Un test isolé (avec le bonus d'exploration entièrement désactivé, correspondant exactement à la formulation du papier) est en cours pour confirmer la part de ce gain réellement attribuable au terme de régularisation.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/kl-regularization-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — avec (en rose) vs. sans (en rouge) régularisation KL (premier test, non isolé)*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/kl-regularization-kl-penalty.png" alt="Curve">

            *Le terme de régularisation KL lui-même au cours de l'entraînement*
        </td>
    </tr>
</table>

- **Une architecture de politique "stationnaire"** (pas encore implémentée), la pièce restante de la recette XQCfD, destinée à faire en sorte que la politique adopte par défaut un comportement conservateur et incertain quand elle agit en dehors de la région réellement couverte par les démonstrations, plutôt que de continuer à agir avec assurance.

L'architecture originale du critic (scalaire) a aussi été rendue directement sélectionnable (pas seulement préservée comme référence passive) pour pouvoir être relancée dans exactement les mêmes conditions que la version catégorielle sur demande, pour comparaison supplémentaire.

### Phase 2 — Étendre la même rigueur aux baselines PPO/GRPO

Des baselines on-policy (PPO, GRPO) ont été ajoutées au même codebase pour fournir des points de comparaison supplémentaires, partageant la même boucle d'entraînement et la même infrastructure qu'ExpoFT. Exercer ce code pour la première fois a mis au jour plusieurs bugs numériques indépendants et confirmés — aucun n'avait été détecté auparavant, puisque ce chemin de code n'avait encore jamais été exécuté jusqu'au bout :

- Une incompatibilité de construction de réseau entre le réseau de fonction de valeur et une pièce d'infrastructure partagée qui supposait une forme de réseau différente (ensemble)
- Une étape de post-traitement d'action qui paddait la dimension de l'action deux fois, une fois de trop
- Une vérification de repli incorrecte pour une quantité (l'entropie de la politique) qui n'a pas de forme close pour la distribution d'action de ce projet, empruntant silencieusement la mauvaise branche de code et plantant au lieu d'utiliser l'approximation par échantillon déjà implémentée pour exactement cette situation
- Un terme de ratio non clippé dans l'objectif PPO/GRPO qui pouvait, après suffisamment de dérive de la politique, atteindre silencieusement l'infini en virgule flottante et corrompre la loss de tout un batch
- Une instabilité numérique à la frontière exacte de l'espace d'action (compressé par tanh), rencontrée lors du rejeu de certaines actions stockées, qui pouvait de façon similaire corrompre un pas de gradient

Chacun de ces éléments a été confirmé par un test ciblé et reproductible construit spécifiquement pour l'isoler, plutôt que corrigé sur simple inspection du code. Une protection défensive a aussi été ajoutée par-dessus l'ensemble : si un gradient calculé s'avère non fini pour une raison quelconque, cette mise à jour particulière est désormais entièrement sautée (laissant le modèle et l'état de l'optimiseur intacts) plutôt que de pouvoir corrompre silencieusement l'entraînement, avec un logging de diagnostic ajouté à chaque étape du calcul pour qu'une future occurrence soit immédiatement traçable plutôt que de se présenter comme un simple crash bien plus tard.

**Une découverte plus fondamentale.** L'investigation de pourquoi ces baselines ne montraient aucun signal d'apprentissage dès le tout début de l'entraînement (plutôt que le schéma "commence bien, puis se dégrade" observé avec ExpoFT) a révélé que leur réseau de politique entraînable n'avait en réalité jamais été connecté au checkpoint pré-entraîné par supervision : il s'agit d'un petit réseau séparé, initialisé aléatoirement, le checkpoint pré-entraîné n'étant utilisé que pour convertir ses entrées et sorties vers et depuis le bon format — pas pour lui donner quoi que ce soit du comportement de la politique pré-entraînée comme point de départ. C'est une conséquence raisonnable de l'utilisation d'un modèle de base à flow-matching, qui n'expose pas directement le type de vraisemblance d'action dont ces deux algorithmes ont besoin, mais cela signifiait que ces baselines devaient en réalité réapprendre la manipulation depuis zéro, reproduisant la propre découverte de la Phase 1 de ce projet (le RL on-policy ne peut pas s'amorcer sans une politique de départ compétente) alors même que l'intention était de tester le fine-tuning on-policy *à partir* du checkpoint SFT. Une option de warmup par clonage comportemental, à l'image de l'idée de pretraining du critic utilisée pour ExpoFT, a été ajoutée pour ces baselines afin de combler cet écart avant de tirer d'autres conclusions à partir d'elles.

---

## Difficultés majeures rencontrées

### Goulot d'étranglement d'exploration (Phase 1, fondamental)
Les algorithmes on-policy (PPO, GRPO) ne peuvent pas générer de signal d'apprentissage quand la politique de base a ~0% de succès. Ceci s'applique à LIBERO-Long (11.9% de base SFT) et directement à StackCube avec le modèle OpenVLA de base. Aucune stratégie d'exploration (température, bonus d'entropie) n'est suffisante sans un warmup SFT préalable ou des démonstrations — ceci a directement motivé le pivot vers ExpoFT.

### Décalage d'espace d'action et le MAJOR RESET (Phase 2)
La plus grande source d'effort perdu en Phase 2 : une incohérence de contrôle cartésien vs. articulaire produisait des symptômes (succès proche de 0%, bras approchant sans compléter la tâche) identiques en apparence à un véritable plafond de difficulté de tâche. Le diagnostiquer a nécessité de construire un outillage de validation par replay rigoureux et ancré dans la vérité terrain, plutôt que de faire confiance aux seules courbes d'entraînement. Plusieurs des bugs supplémentaires trouvés durant la revue systématique qui a suivi étaient silencieux — ils ne faisaient rien planter, ils produisaient simplement des chiffres plausibles mais faux — ce qui a rendu une revue systématique complète nécessaire plutôt qu'un fix ciblé.

### Incompatibilité de design de la tâche PickCube (résolue)
Un mode de défaillance qui ressemblait d'abord à un problème de politique ou d'exploration (0% de succès, aucune amélioration) s'est révélé être un manque fondamental d'observabilité entre le design de la tâche (marqueur d'objectif caché) et le schéma d'observation imitant DROID du projet. A nécessité des diagnostics au niveau caméra pour être correctement retracé, et PickCube-v1 a été retirée dans un premier temps en conséquence. Un monkeypatch ultérieur révélant le marqueur d'objectif à la caméra capteur a résolu le manque sans compromettre le schéma d'observation, et la tâche a été réintégrée. Leçon généralisée : vérifier l'alignement caméra/observabilité du VLA avec le design d'une tâche ManiSkill donnée (marqueurs d'objectif cachés, angle de caméra fixe, présence de caméra au poignet) avant de supposer que de mauvais résultats reflètent la difficulté de la tâche — et avant de supposer que l'incompatibilité est infixable.

### Instabilité du critic RL (Phase 2, difficulté centrale — considérablement précisée depuis son premier signalement)
Chaque configuration RL testée jusqu'ici dégrade plutôt qu'elle n'améliore la politique SFT validée, la dégradation étant initialement corrélée à une instabilité croissante de la critic loss. Deux fixes indépendants et réels — une architecture de critic reconstruite avec des estimations de valeur structurellement bornées, et un bug de timing de récompense/terminaison corrigé dans la boucle de collecte de données — ont chacun été confirmés comme stabilisant significativement la dynamique d'entraînement propre du critic, mais aucun n'a changé l'effondrement de la performance de la politique. Ceci a déplacé l'hypothèse principale de l'instabilité d'entraînement du critic elle-même vers le mécanisme de sélection d'action (le même critic proposant et jugeant sa propre action préférée), désormais testé directement plutôt qu'inféré par élimination. Une leçon utile, quoique d'abord contre-intuitive, de cette période de travail : un critic qui a l'air plus sain n'est pas la même chose qu'une meilleure politique, et les deux doivent être vérifiés séparément plutôt que supposés évoluer ensemble.

### Premier test de régularisation confondu (résolu, leçon de processus)
Le premier test de régularisation KL contre la politique pré-entraînée a été lancé avec le bonus d'exploration par défaut encore actif en parallèle, plutôt que désactivé comme le sous-entend la formulation du papier source lui-même — ce qui signifie que le résultat encourageant de ce run ne pouvait pas encore être attribué au terme de régularisation spécifiquement. Repéré avant d'en tirer une conclusion ; un test isolé de suivi est en cours. Leçon générale : lors de l'ajout d'un terme à un objectif existant, vérifier explicitement chaque autre terme avec lequel il pourrait interagir avant de rapporter un résultat, plutôt que de supposer qu'une nouvelle option est indépendante de ce qui existe déjà.

### Baselines PPO/GRPO silencieusement déconnectées de la politique pré-entraînée (résolu)
Ces baselines étaient supposées continuer à partir du checkpoint pré-entraîné par supervision, de la même façon que la méthode ExpoFT principale — ce n'était pas le cas. Leur réseau de politique entraînable était un réseau séparé, initialisé aléatoirement, depuis le début, le checkpoint pré-entraîné n'étant utilisé que pour le formatage des entrées/sorties. Ceci n'a été détecté par aucun test réussi ni aucun run plantant ; cela n'a émergé qu'une fois les *résultats* de ces baselines examinés de façon critique (performance proche de zéro persistante dès le tout premier step) plutôt que le code supposé correct parce qu'il s'exécutait sans erreur.

### Infrastructure — cluster, Ray, et stockage (Phase 1 et 2)
- Modèle chargé deux fois en mémoire GPU (RolloutGroup + ActorGroup) → OOM sur L40S (44GB) → passage à A100L (80GB) avec `enable_offload: True` (Phase 1)
- Clusters Ray résiduels d'autres utilisateurs bloquant l'initialisation sur plusieurs nœuds → résolu avec `unset RAY_ADDRESS` et exclusion de nœuds via `#SBATCH --exclude` (Phase 1)
- Répertoire `$HOME` trop petit pour les checkpoints (~50GB chacun) → projet migré vers `$SCRATCH`, puis re-migré vers `$HOME` avec seulement les artefacts volumineux régénérables en lien symbolique vers `$SCRATCH`, selon les recommandations de stockage mises à jour de Mila (Phase 2)
- Conflit de dépendance `robosuite` entre LIBERO et RoboCasa → résolu avec deux venvs séparés (Phase 1)
- Crashs OOM GPU JAX pendant l'entraînement RL, retracés à un problème de recompilation/structure pytree et à la pré-allocation mémoire par défaut de 75% de JAX (Phase 2)

---

## Ajustements et décisions clés

| Décision | Justification |
|---|---|
| L40S abandonné pour A100L (80GB) | VRAM du L40S insuffisante pour des modèles chargés deux fois en mémoire |
| Multi-GPU abandonné pour GPU unique + `enable_offload` | Problèmes de timeout NCCL en multi-GPU ; un seul GPU suffisant avec l'offloading |
| LIBERO-90 abandonné | Trop de tâches simultanées ; signal d'apprentissage peu focalisé |
| OpenVLA-OFT abandonné comme modèle principal | Pivot vers π₀ puis π₀.₅ pour ExpoFT (mieux adapté à la question de recherche) |
| LIBERO / RoboCasa abandonnés comme benchmarks principaux | Partiellement saturés par le SFT ; potentiel OOD limité |
| RL on-policy depuis le modèle de base abandonné | 0% de succès sur StackCube ; l'on-policy ne peut pas s'amorcer sans SFT ni démonstrations |
| ManiSkill sélectionné (StackCube, PushCube, PickCube) | Maximalement OOD par rapport aux données de pretraining DROID de π₀.₅ |
| Pivot vers l'architecture résiduelle ExpoFT | Fige le VLA, entraîne une petite politique résiduelle via une Q-function ; évite la limitation de log-probabilité des modèles à flow-matching ; contourne le problème d'exploration à froid via les démonstrations |
| Reproduire ExpoFT dans un dépôt ManiSkill dédié, pas dans RLinf | Garde le code de l'algorithme original proche de son état intact ; seule la couche environnement/données est remplacée |
| `openpi` et `ManiSkill` en submodules git, pas en installations `pip` | Permet des modifications directes et trackées plutôt que des monkeypatches à l'exécution |
| ExpoFT complet tenté d'abord, composants retirés ensuite | Méthodologie de retrait progressif pour identifier la supervision minimale réellement nécessaire, plutôt que de deviner à l'avance |
| Bascule vers le contrôle en espace articulaire de bout en bout | Correspond à l'espace d'action du pretraining DROID de π₀.₅ ; le contrôle cartésien écrêtait silencieusement la plupart des actions de démonstration |
| Construction de `validate_demos_full_pipeline.py` | Validation de replay ancrée dans la vérité terrain, aux seeds exactes, nécessaire après que le bug d'espace d'action ait montré que les courbes d'entraînement seules ne sont pas fiables |
| Erreur standard (pas intervalle de Wilson) pour toutes les barres d'erreur | Selon la préférence explicite de l'équipe (Jesse) |
| PickCube-v1 retirée (premier passage) | Le design de tâche à marqueur d'objectif caché semblait d'abord incompatible avec le schéma d'observation imitant DROID (image + proprioception uniquement) |
| PickCube-v1 réintégrée | Un monkeypatch révélant le marqueur d'objectif à la caméra capteur a restauré un accès visuel réel à la position de l'objectif sans ajouter de canal d'état privilégié — l'incompatibilité était corrigeable, pas fondamentale |
| Dépôt déplacé de `$SCRATCH` uniquement vers `$HOME` + artefacts en lien symbolique | Suit les recommandations de stockage mises à jour de Mila ; garde uniquement les données volumineuses et régénérables sur le scratch auto-nettoyé |
| `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95` adopté comme défaut permanent | Résout les crashs OOM récurrents de l'entraînement RL liés à la pré-allocation mémoire par défaut de 75% de JAX |
| Défaut de travail `utd_ratio=2` (vs. 20 dans le papier) | Le ratio du papier lui-même déstabilise l'entraînement encore plus vite que le ratio réduit dans cette configuration simulée |
| Runs RL catastrophiques d'avant les fixes supprimés de `$SCRATCH` | Libère de l'espace disque ; ne garde que les checkpoints SFT validés et les runs RL post-fix |
| Critic reconstruit en architecture catégorielle (à support borné), selon XQC/XQCfD | Teste directement si la croissance non bornée de la valeur du critic était la cause racine de la dégradation de la politique, plutôt que de continuer à chercher des hyperparamètres autour de l'architecture originale |
| Architecture de critic originale préservée dans un module séparé toujours exécutable plutôt qu'écrasée | Garde une comparaison A/B propre contre la refonte catégorielle disponible à tout moment, plutôt qu'une migration à sens unique |
| Normalisation adaptative (par estimation courante) de la récompense plutôt que des bornes de valeur fixes ajustées à la main | Garde le support fixe du critic catégoriel pertinent à travers les tâches sans ajustement par tâche, et sans besoin de reprojeter le support en cours d'entraînement |
| Tester les mitigations propres au papier XQCfD (pretraining du critic, critic catégoriel, régularisation KL, architecture stationnaire) dans l'ordre, avant un fix alternatif issu d'une autre ligne de recherche | Garde les résultats attribuables à une seule méthodologie citable plutôt que de mélanger des approches issues de papiers non liés |
| Régularisation KL implémentée comme un terme additionnel et activable plutôt qu'un remplacement strict du bonus d'exploration existant | Préserve la capacité de reproduire la formulation exacte du papier (en désactivant l'autre terme) tout en gardant les deux disponibles indépendamment pour l'ablation |
| Ajout d'une option de warmup par clonage comportemental pour les baselines PPO/GRPO | Répond à la découverte que leur politique entraînable n'avait aucune connexion au checkpoint pré-entraîné autrement |
| Ajout d'une garde défensive contre les gradients non finis à l'étape de mise à jour PPO/GRPO | Empêche un futur problème numérique encore inconnu de corrompre silencieusement un run d'entraînement, indépendamment du fait que sa cause racine ait été trouvée |

---

## Figures, visuels, et démonstrations

Les courbes d'entraînement clés, vidéos d'épisodes, et poids de checkpoints pour la Phase 1 sont intégrés directement dans chaque section d'expérience ci-dessus. Les poids finaux pour le run RoboCasa CloseDrawer PPO sont disponibles sur HuggingFace : [`josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO`](https://huggingface.co/josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO).

Les figures et vidéos de la Phase 2 sont aussi intégrées dans les descriptions d'expériences ci-dessus. Les poids post-SFT de Pi0.5 sur les tâches maniskill sont disponibles sur HuggingFace : 

- [`josh11234/ExpoFT-Pi05-StackCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-StackCube-v1-SFT) (41% de succès sur 200 seeds held-out)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p) (62% de succès sur 200 seeds held-out)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p) (86% de succès sur 200 seeds held-out)
- [`josh11234/ExpoFT-Pi05-PickCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-PickCube-v1-SFT) (22% de succès sur 200 seeds held-out)

Les figures de comparaison d'architecture de critic, de fix de timing de récompense, et de régularisation KL référencées dans les dernières mises à jour ci-dessus restent à ajouter — les placeholders utilisent la même convention `images/` que le reste de cette page.