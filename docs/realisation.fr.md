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

# Réalisation

## Architecture et approche générale

### Objectif de recherche

La question centrale de ce projet est de savoir si le fine-tuning RL peut permettre une **généralisation hors distribution (OOD)** dans les modèles Vision-Language-Action (VLA) — c'est-à-dire permettre à un modèle de réussir des tâches maximalement éloignées de sa distribution d'entraînement SFT. Une question secondaire est de savoir si le principal bottleneck à l'amélioration par RL est l'**exploration** (la policy ne peut pas découvrir de signal de reward) ou l'**optimisation** (la policy ne peut pas s'améliorer même avec un signal).

### Vue d'ensemble du pipeline

Le projet s'est déroulé en deux grandes phases :

**Phase 1 — RL on-policy (Semaines 1–7) :** Fine-tuning RL direct via GRPO/PPO sur OpenVLA-OFT et π₀ dans les environnements LIBERO et RoboCasa. Cette phase a établi l'hypothèse du bottleneck d'exploration et les limites des approches on-policy sans warmup SFT.

**Phase 2 — Residual policy ExpoFT (Semaine 8–présent) :** Reproduction et adaptation de l'[architecture ExpoFT](https://pd-perry.github.io/expo-ft/) dans une codebase dédiée pilotée par la simulation ManiSkill (`expo-ft`), remplaçant le pipeline de données real-robot (DROID) original d'ExpoFT tout en gardant l'algorithme lui-même intact. Le VLA (π₀.₅) est gardé **figé**, et une petite residual policy de correction est entraînée par-dessus via une Q-function, en combinant RL off-policy et démonstrations de motion-planning. Cette conception évite deux limitations de la Phase 1 : elle ne nécessite pas de log-probabilities d'action valides (que les VLA à flow-matching comme π₀/π₀.₅ ne peuvent pas fournir, excluant le policy-gradient RL standard directement sur le modèle de base) et elle contourne le problème d'exploration à froid (cold-start) en amorçant le replay buffer avec des démonstrations.

La méthodologie choisie est le **retrait progressif de composants** (progressive component stripping) : partir de la recette ExpoFT complète (warmup SFT + démonstrations + RL off-policy) et valider rigoureusement chaque étape avant de retirer des composants (taille du dataset, warmup SFT lui-même) afin d'identifier la supervision minimale réellement nécessaire.

### Pourquoi OpenVLA / π₀.₅ plutôt que des VLA à flow-matching

OpenVLA(-OFT) et π₀.₅ utilisent des **tokens d'action discrets**, ce qui donne des log-probabilities valides et permet donc un policy-gradient RL rigoureux (GRPO/PPO en Phase 1). Les modèles à flow-matching (π₀, MoMaLM) ne peuvent pas fournir de likelihoods tractables, ce qui polluerait le design expérimental de toute approche reposant sur des policy gradients exacts. C'est aussi pourquoi la formulation en residual policy d'ExpoFT — qui contourne entièrement le besoin de log-probabilities du VLA en agissant comme une correction additive apprise via une Q-function — est une façon intéressante de continuer à utiliser un VLA figé et performant sans être bloqué par cette limitation.

### Environnement de travail

- **Cluster :** Mila HPC (SLURM), GPU A100/A100L (40GB/80GB)
- **Modèles testés :** OpenVLA-OFT (7B), π₀ (3B) (Phase 1) ; π₀.₅ (checkpoint `pi05_droid_jointpos`, figé) + residual policy (Phase 2)
- **Environnements :** LIBERO (Spatial, Long), RoboCasa (CloseDrawer) — Phase 1 ; ManiSkill (StackCube-v1, PushCube-v1, PickCube-v1) — Phase 2
- **Algorithmes RL :** GRPO, PPO (Phase 1) ; Q-learning off-policy / ensemble de critics de style REDQ via ExpoFT (Phase 2)
- **Framework :** RLinf (`montrealrobotics/RLinf`) pour la Phase 1 ; dépôt autonome `expo-ft` (fork propre d'`openpi` et de `ManiSkill` en git submodules) pour la Phase 2
- **Infrastructure clé :** quatre venvs Python distincts pour gérer les conflits de dépendances ; dépôt hébergé sur `$HOME` avec `logs/` et `demos/` en lien symbolique vers `$SCRATCH` ; gros caches de paquets déplacés vers `$SCRATCH`

---

## Expériences et analyses réalisées

### Phase 1 — Baselines RL on-policy

#### LIBERO-90 GRPO (OpenVLA-OFT)
Première expérience. 1000 steps (~31h40). `success_once` a montré des pics occasionnels (~0.13) sans tendance stable à la hausse. Attribué initialement à un petit batch size ; confirmé plus tard comme un bottleneck d'exploration.

![LIBERO-90 GRPO Training Results Success Once]({{ config.site_url }}/images/libero90_grpo_training1_success_once.png)
![LIBERO-90 GRPO Training Results Approx KL]({{ config.site_url }}/images/libero90_grpo_training1_approx_kl.png)

#### RoboCasa CloseDrawer PPO (π₀)
Entraînement de 150 steps avec LoRA (rank 32). Métriques finales : `success_once` ~0.82, `critic/value_loss` ~0.005, `approx_kl` ~0.036. A démontré que le RL peut améliorer la performance quand la tâche est dans la distribution — mais CloseDrawer était trop facile (base SFT déjà ~0.6–0.7), laissant une marge limitée pour la contribution du RL.

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
Entraînement de ~38 steps. `success_once` a oscillé entre 0.22 et 0.875 sans convergence claire. Identifié comme une mauvaise configuration du learning rate suite à la réduction du nombre d'epochs.

![LIBERO-Spatial GRPO Training Results Success Once]({{ config.site_url }}/images/libero_spatial_grpo_training1_success_once.png)
![LIBERO-Spatial GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_spatial_grpo_training1_approx_kl.png)

#### LIBERO-Long GRPO (OpenVLA-OFT)
~150 steps répartis sur 4 sessions reprises (~37h au total). `success_once` a oscillé entre 0.06 et 0.19 sans tendance à la hausse. `approx_kl` est resté stable autour de 0.035–0.048 tout du long — les mises à jour de la policy étaient contrôlées, confirmant que le bottleneck est l'**exploration**, pas une instabilité d'optimisation.

![LIBERO-Long GRPO Training Results Success Once]({{ config.site_url }}/images/libero_long_grpo_training1_success_once.png)
![LIBERO-Long GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_long_grpo_training1_approx_kl.png)

#### StackCube-v1 PPO et GRPO (OpenVLA de base, on-policy, sans SFT)
Les deux entraînements ont produit un taux de succès de 0%. Sans warmup SFT, la policy de base ne peut produire aucun comportement pertinent pour la tâche, donc l'exploration on-policy ne peut pas amorcer de signal d'apprentissage. Ce résultat a directement motivé le pivot vers ExpoFT. Voici les résultats des entraînements PPO et GRPO sur StackCube-v1 :

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-ppo-nosft.mp4" type="video/mp4">
    </video>
</div>

*OpenVLA sur StackCube-v1 (ManiSkill) après PPO*


<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-grpo-nosft.mp4" type="video/mp4">
    </video>
</div>

*OpenVLA sur StackCube-v1 (ManiSkill) après GRPO*


### Sélection et visualisation du dataset OOD

- Création d'un notebook Jupyter pour l'analyse de distribution des endpoints à travers les datasets Open X-Embodiment ([Colab](https://colab.research.google.com/github/Josh012006/OpenX-Embodiment-Datasets-Visualization/blob/main/colabs/Open_X_Embodiment_Datasets.ipynb), [Dépôt](https://github.com/Josh012006/OpenX-Embodiment-Datasets-Visualization))
- Un fit UMAP partagé a été utilisé à travers tous les datasets (plutôt qu'un fit par dataset), car des fits séparés rendraient les coordonnées incomparables entre datasets
- ManiSkill identifié comme benchmark OOD cible : des tâches comme StackCube-v1 et PushCube-v1 sont maximalement éloignées des données de pré-entraînement/SFT du VLA (DROID/OXE)
- Note : la distance L2 spatiale entre datasets peut être trompeuse à cause des référentiels de coordonnées différents entre datasets

### Exploration d'hyperparamètres — réduction du temps d'entraînement (Phase 1)

Test de l'augmentation du replay ratio (`rollout_epoch` 4→8 pour GRPO, `update_epoch` 2→4 pour PPO) pour réduire le temps réel. Résultat : le bottleneck s'est déplacé de la collecte de rollouts vers les mises à jour de gradient, rendant l'entraînement *plus lent* (~27 min/step vs ~14 min/step). L'approche efficace est plutôt de réduire directement `max_episode_steps` et `max_epochs`.

---

### Phase 2 — ExpoFT : mise en place du dépôt

L'algorithme ExpoFT a été reproduit dans un dépôt dédié piloté par ManiSkill (`~/projects/expo-ft`) plutôt que dans RLinf, afin de garder le code de l'algorithme original aussi intact que possible en ne remplaçant que la couche environnement/données. Deux dépendances — `openpi` (l'implémentation de π₀.₅) et `ManiSkill` — ont été forkées et intégrées comme de vrais git submodules éditables (plutôt que des dépendances non trackées/installées via `pip`), permettant des modifications directes et trackées plutôt que des monkeypatches à l'exécution. `.gitmodules` utilise des URLs HTTPS pour la compatibilité de clonage public, avec des remotes SSH configurés localement pour le push.

La disposition du stockage suit les recommandations du cluster Mila : le dépôt lui-même vit sur `$HOME` (sauvegardé, 100GB), tandis que les artefacts volumineux et régénérables (`logs/`, `demos/`) sont en lien symbolique vers `$SCRATCH` (5TB, nettoyé automatiquement). Les gros caches de paquets (`uv`, `openpi`, `huggingface`, `pip`, `jax`, ~90GB au total) ont été déplacés vers `$SCRATCH/cache/` avec des liens symboliques retour vers `~/.cache`, libérant le quota `$HOME`.

### Phase 2 — Étape SFT : diagnostic et validation de l'espace d'action

Le premier entraînement de warmup SFT (Semaine 8), sur des démonstrations StackCube-v1 converties au format DROID/LeRobot, a produit un taux de succès proche de 0% — le bras se contentait d'approcher les cubes sans compléter l'empilement. Ceci a d'abord été suspecté comme un vrai plafond de difficulté de tâche, mais une investigation systématique l'a rattaché à un **mismatch fondamental d'espace d'action** : le pré-entraînement DROID de π₀.₅ attend des actions en position articulaire en unités physiques, alors que le pipeline utilisait le contrôle cartésien `pd_ee_delta_pose` de ManiSkill, causant l'écrêtage silencieux de la majorité des actions de démonstration hors de leur plage valide.

La correction — passage du pipeline complet au contrôle en espace articulaire de bout en bout (`control_mode=pd_joint_delta_pos`, checkpoint `pi05_droid_jointpos`, état `observation/joint_position`, `normalize_action=True`, `output_action_dim=8`, normalisation stats DROID depuis GCS) — a immédiatement ramené la grande majorité des actions dans leur plage valide lors de la validation par replay.

Cela a déclenché une revue systématique plus large (« MAJOR RESET ») qui a fait remonter un ensemble de bugs silencieux supplémentaires avant que les chiffres SFT ne puissent être fiables :

- Un script d'évaluation écrasant sans condition les normalisation stats officielles de DROID, y compris pour l'évaluation de la baseline
- Un reliquat de rescaling d'action en espace cartésien issu du mode de contrôle abandonné, corrompant une part importante des actions désormais en espace articulaire
- Un désalignement de longueur de troncature entre l'environnement et la boucle d'évaluation
- Un mismatch de backend de simulation empêchant la bonne conversion du mode de contrôle

Pour regagner confiance dans le pipeline de données, `validate_demos_full_pipeline.py` a été construit comme un outil de replay de bout en bout rigoureux (utilisant les vraies normalisation stats, le vrai wrapper d'environnement et les seeds exactement enregistrés). Après les corrections ci-dessus, cet outil a confirmé **100% de succès de replay des démonstrations sur StackCube**, donnant une vraie confiance dans le fait que l'espace d'action et la conversion des données étaient enfin corrects.

Un outillage d'évaluation approprié a aussi été construit à cette étape : seeds d'épisodes fixes (200 seeds retenus, générés une seule fois à partir d'une seed maîtresse et réutilisés à travers les checkpoints) pour une comparaison équitable entre checkpoints, barres d'erreur standard (`SE = sqrt(p(1-p)/n)`, une approximation normale plutôt qu'un intervalle de Wilson, à la demande explicite de l'équipe), et un script de sweep de checkpoints reprenable qui reconstruit son graphique après chaque checkpoint.

Les sweeps complets de checkpoints SFT sur ces 200 seeds fixes ont confirmé une **vraie généralisation validée** (pas de la mémorisation) sur StackCube, avec des taux de succès croissants au fil des itérations SFT ; PushCube-v1 (ajoutée comme deuxième tâche, plus facile, pour la vérification de bon fonctionnement) a performé encore mieux.

Voici les courbes de sweep de checkpoints SFT (taux de succès vs. step/itération d'entraînement, avec barres d'erreur SE, 200 seeds retenus) pour **StackCube-v1** et **PushCube-v1**.

![SFT Checkpoint Sweep Curves PushCube-v1]({{ config.site_url }}/images/curve-sft-pushcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pushcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur PushCube-v1 (ManiSkill) après warmup SFT*


![SFT Checkpoint Sweep Curves StackCube-v1]({{ config.site_url }}/images/curve-sft-stackcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur StackCube-v1 (ManiSkill) après warmup SFT*



Et voici des enregistrements d'épisodes montrant le comportement du meilleur checkpoint SFT sur StackCube-v1 et PushCube-v1, pour accompagner visuellement les courbes de sweep ci-dessus (même style que la vidéo RoboCasa CloseDrawer en Phase 1).

### Phase 2 — Suite de tâches : StackCube, PushCube et PickCube (abandonnée, puis réintégrée)

PickCube-v1 a été ajoutée comme troisième tâche pour diversifier davantage la suite OOD, mais montrait initialement un taux de succès persistant de 0%. Ceci a été rattaché à une **incompatibilité de conception de tâche** plutôt qu'à un problème de policy ou d'exploration : PickCube-v1 de ManiSkill cache par défaut le marqueur d'objectif (`goal_site`) aux caméras capteur, et bien que `obs['extra']['goal_pos']` soit techniquement accessible dans le simulateur, le schéma d'observation de style DROID utilisé tout au long de ce projet (image + proprioception uniquement, correspondant à ce qu'aurait un vrai robot) n'avait aucun canal pour cette information. La policy n'avait donc aucun accès à la position d'objectif par épisode — ni visuellement, ni numériquement — rendant la tâche apparemment insolvable de façon fondamentale.

PickCube-v1 a été abandonnée dans un premier temps plutôt que contournée avec un canal d'état privilégié, puisque cela ne correspondrait pas à ce qu'un vrai robot pourrait percevoir. Cependant, un monkeypatch démasquant `goal_site` spécifiquement pour la caméra capteur — donnant à la policy un véritable accès *visuel* à la position de l'objectif, exactement comme le ferait une vraie caméra pointée vers une cible visible — a par la suite été validé avec succès. Ceci a résolu le problème d'observabilité sans compromettre le schéma d'observation de style DROID, et **PickCube-v1 a été réintégrée dans la suite de tâches active**.

Un warmup SFT a donc aussi été lancé sur PickCube-v1. Voici les performances :

![SFT Checkpoint Sweep Curves PickCube-v1]({{ config.site_url }}/images/curve-sft-pickcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pickcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 sur PickCube-v1 (ManiSkill) après warmup SFT*


**Suite de tâches active pour la suite : StackCube-v1, PushCube-v1 et PickCube-v1.**

### Phase 2 — Étape RL : revue de code et corrections d'infrastructure

Avant de faire confiance à un quelconque résultat RL, l'étape d'entraînement RL a reçu la même revue ligne par ligne que l'étape SFT. Cela a fait remonter un autre ensemble de bugs réels :

- Un bug d'écrasement des normalisation stats identique à celui trouvé précédemment en évaluation SFT
- Environ 15 flags CLI passés au script d'entraînement mais jamais lus silencieusement par celui-ci (hyperparamètres orphelins — présents dans les configs, jamais réellement consommés par le code)
- Un flag d'écrasement de checkpoint codé en dur faisant planter chaque entraînement neuf (non repris)
- Un désalignement entre deux copies différentes d'un flag d'entraînement clé, causant un plantage dès qu'il était réglé à une valeur non par défaut
- Plusieurs hyperparamètres RL définis dans les configs YAML de tâche mais jamais réellement lus par le script d'entraînement, retombant silencieusement sur des valeurs par défaut différentes ailleurs dans la codebase
- La valeur d'un hyperparamètre silencieusement interprétée comme du texte plutôt qu'un float par le lecteur de config (PyYAML interprète la notation scientifique nue comme `3e-4` comme une chaîne de caractères, pas un float)

Séparément, `eval_policy.py` s'est révélé avoir par défaut `only_base_actions=True`, ce qui signifie qu'il avait silencieusement évalué uniquement la policy VLA de base figée — jamais la residual policy entraînée — sur chaque checkpoint RL évalué jusque-là. Un support approprié pour charger et évaluer l'agent RL complet (residual policy et critic ensemble) a été construit et utilisé pour toutes les évaluations à partir de ce point.

Deux causes distinctes de plantages OOM répétés durant l'entraînement RL ont aussi été diagnostiquées et corrigées :
- Un problème de recompilation JAX déclenché par une forme d'entrée incohérente en cours d'entraînement (un basculement de structure pytree `None`→dict pour `actor_batch` à travers une frontière `jax.jit` forçait JAX à compiler silencieusement et garder un second programme XLA résident)
- L'allocateur mémoire GPU par défaut de JAX ne réservant que 75% de la carte par défaut, plutôt que la carte complète — résolu avec `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95`

Enfin, le logging TensorBoard en cours d'entraînement s'est révélé manquer silencieusement la plupart des métriques d'entraînement (seuls le taux de succès et le temps de boucle étaient visibles, alors que wandb affichait tout correctement) à cause d'un problème de filtrage de type — les scalaires JAX `jnp.float32` échouent aux vérifications Python `isinstance(v, (int, float))`. Corrigé avec un cast explicite `float()`.

La recette exacte d'augmentation de données d'entraînement du papier a été reproduite à partir de sa description et confirmée comme déjà correctement implémentée dans la codebase.

### Phase 2 — Étape RL : le constat central d'instabilité

Une fois les bugs d'infrastructure et d'évaluation corrigés, et PickCube-v1 réintégrée après sa correction d'observabilité, les premiers vrais entraînements RL ont été lancés depuis les checkpoints SFT validés sur les trois tâches actives. Des courbes d'évaluation complètes (200 seeds fixes retenus par checkpoint) ont ensuite été lancées à travers une série d'expériences de comparaison faisant varier le **ratio update-to-data (UTD)** et la proportion de chaque batch d'entraînement tirée des données de démonstration, incluant un entraînement reproduisant les hyperparamètres exacts du papier dans sa propre durée d'entraînement validée.

**Résultat clé :** chaque configuration RL essayée jusqu'ici — à travers les trois tâches actives (StackCube, PushCube, PickCube) et chaque réglage de ratio UTD / proportion de démonstration testé — **dégrade le taux de succès de la policy pré-entraînée par SFT plutôt que de l'améliorer** :

- `utd_ratio=10` : effondrement catastrophique en ~2–5k steps
- `utd_ratio=2` + `offline_ratio=0.5` : retarde l'effondrement jusqu'à ~35–40k steps, se dégradant finalement à ~0.3–0.4 de succès vers ~85–90k steps
- `utd_ratio=20` (la valeur exacte du papier, exécuté dans la plage de steps validée du papier lui-même, ~20–24k steps) : s'est effondré **plus vite et plus sévèrement** que le long entraînement à `utd_ratio=2` — cela réfute l'hypothèse que s'entraîner simplement dans le budget plus court du papier (sur robot réel) éviterait le problème ; c'est le `utd_ratio` lui-même, pas la durée totale d'entraînement, qui semble être le principal facteur d'instabilité

Cette dégradation est systématiquement précédée et accompagnée d'une **instabilité croissante de la loss du critic** (pics de critic loss progressifs), suggérant que la cause racine est une fonction de valeur se déstabilisant progressivement plutôt qu'un bug du pipeline — ceci a été confirmé davantage en vérifiant que l'évaluation utilise bien la residual policy et le critic entraînés, sans retomber silencieusement sur la policy de base figée (voir la correction de `eval_policy.py` ci-dessus).

Correspondance des hyperparamètres confirmée par rapport au papier original : `offline_ratio=0` (les démonstrations sont placées dans le seul replay buffer en ligne, pas dans un buffer offline séparé 50/50 — cette convention appartient à RLPD, pas à ce papier), `hidden_dims=3×256`, `init_temperature=1.0`, `num_qs=10` / `num_min_qs=2` (ensemble de critics de style REDQ), `tau=0.005`. Écarts connus par rapport au papier : `batch_size=24` (le papier utilise 64 ; contraint par la mémoire GPU disponible) et un `edit_scale=0.2` fixe (le papier utilise 0.05–0.2, ajusté selon la difficulté de la tâche).

Voici une comparaison des métriques d'entraînement à travers les trois configurations de ratio UTD décrites ci-dessus, pour PushCube-v1, la plus facile des trois tâches :

*Évolution de la critic loss durant l'entraînement* :

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


*Évolution de la residual actor loss durant l'entraînement* :
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


*Évolution du taux de succès en évaluation durant l'entraînement* :
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


Et voici une vidéo montrant la dégradation du comportement de la policy sur StackCube-v1 après ~120k steps de fine-tuning RL :
<table>
    <tr>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
                </video>
            </div>

            *Avant le fine-tuning RL avec ExpoFT*
        </td>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-rl-degradation.mp4" type="video/mp4">
                </video>
            </div>

            *Après le fine-tuning RL avec ExpoFT*
        </td>
    </tr>
</table>

**Question de recherche ouverte**, soulevée au groupe pour discussion : s'agit-il d'une limitation fondamentale de l'entraînement de critic off-policy à haut UTD que les entraînements courts sur robot réel du papier n'ont simplement pas le temps de révéler, ou est-ce spécifique à cette configuration simulée (dataset plus petit, architecture VLA figé + residual policy, ou batch size contraint par la mémoire) ? Ceci touche directement à la question de recherche centrale du projet, puisque cela bloque actuellement la démonstration de toute contribution positive du RL par-dessus la baseline SFT validée.

**Pistes non testées identifiées pour les prochaines étapes :** un `edit_scale` plus petit (ex. 0.05) spécifiquement pour StackCube, puisque c'est la tâche la plus exigeante en précision et celle qui s'est effondrée le plus dramatiquement ; `replan_steps=4` pour StackCube (actuellement fixé à 8, alors que le papier utilise 4–8 selon la tâche) ; et `batch_size=32` comme prochain test à faible risque (jamais encore essayé).

---

## Difficultés majeures rencontrées

### Bottleneck d'exploration (Phase 1, fondamental)
Les algorithmes on-policy (PPO, GRPO) ne peuvent générer de signal d'apprentissage quand la policy de base a ~0% de succès. Cela s'applique à LIBERO-Long (base SFT à 11.9%) et directement à StackCube avec le modèle OpenVLA de base. Aucune stratégie d'exploration (température, bonus d'entropie) n'est suffisante sans un warmup SFT préalable ou des démonstrations — ceci a directement motivé le pivot vers ExpoFT.

### Mismatch d'espace d'action et le MAJOR RESET (Phase 2)
La plus grande source d'effort perdu en Phase 2 : un mismatch de contrôle cartésien-vs-articulaire a produit des symptômes (succès proche de 0%, bras approchant sans compléter la tâche) identiques en apparence à un vrai plafond de difficulté de tâche. Le diagnostic a nécessité de construire un outillage de validation par replay rigoureux et fondé sur des vérités terrain, plutôt que de faire confiance aux seules courbes d'entraînement. Plusieurs des bugs supplémentaires trouvés durant la revue systématique qui a suivi étaient silencieux — ils ne faisaient rien planter, ils produisaient simplement des chiffres plausibles mais faux — ce qui est justement ce qui a rendu une revue systématique complète nécessaire plutôt qu'une correction ciblée.

### Incompatibilité de conception de tâche pour PickCube (résolue)
Un mode d'échec qui ressemblait initialement à un problème de policy ou d'exploration (0% de succès, pas d'amélioration) s'est révélé être un problème fondamental d'observabilité entre la conception de la tâche (marqueur d'objectif caché) et le schéma d'observation de style DROID du projet. A nécessité des diagnostics au niveau caméra pour être correctement tracé, et PickCube-v1 a été abandonnée dans un premier temps en conséquence. Un monkeypatch ultérieur démasquant le marqueur d'objectif pour la caméra capteur a résolu le problème sans compromettre le schéma d'observation, et la tâche a été réintégrée. Leçon généralisée : vérifier l'alignement caméra/observabilité du VLA avec la conception d'une tâche ManiSkill donnée (marqueurs d'objectif cachés, angle de caméra fixe, présence de caméra au poignet) avant de supposer que de mauvais résultats reflètent la difficulté de la tâche — et avant de supposer que l'incompatibilité est impossible à corriger.

### Instabilité du critic en RL (Phase 2, difficulté centrale actuelle)
Chaque configuration RL essayée jusqu'ici dégrade plutôt qu'elle n'améliore la policy SFT validée, la dégradation étant corrélée à une instabilité croissante de la critic loss. Réduire le ratio UTD retarde le déclenchement de l'effondrement mais ne l'a empêché dans aucune configuration testée — y compris un entraînement reproduisant les hyperparamètres exacts du papier dans sa propre durée d'entraînement validée. C'est actuellement le problème ouvert central bloquant la question de recherche centrale du projet, et cela a été partagé avec le groupe de recherche élargi pour discussion.

### Infrastructure — cluster, Ray et stockage (Phase 1 et 2)
- Modèle chargé deux fois en mémoire GPU (RolloutGroup + ActorGroup) → OOM sur L40S (44GB) → passage à A100L (80GB) avec `enable_offload: True` (Phase 1)
- Clusters Ray résiduels d'autres utilisateurs bloquant l'initialisation sur plusieurs nœuds → résolu avec `unset RAY_ADDRESS` et exclusion de nœuds via `#SBATCH --exclude` (Phase 1)
- Répertoire `$HOME` trop petit pour les checkpoints (~50GB chacun) → projet migré vers `$SCRATCH`, puis re-migré vers `$HOME` avec seulement les artefacts volumineux et régénérables en lien symbolique vers `$SCRATCH`, suivant les recommandations de stockage mises à jour de Mila (Phase 2)
- Conflit de dépendance `robosuite` entre LIBERO et RoboCasa → résolu avec deux venvs séparés (Phase 1)
- Plantages OOM GPU de JAX durant l'entraînement RL, rattachés à un problème de recompilation/structure pytree et à la pré-allocation mémoire par défaut de 75% de JAX (Phase 2)

---

## Ajustements et décisions clés

| Décision | Justification |
|---|---|
| Abandon du L40S pour le A100L (80GB) | VRAM du L40S insuffisante pour des modèles chargés deux fois en mémoire |
| Abandon du multi-GPU pour mono-GPU + `enable_offload` | Problèmes de timeout NCCL en multi-GPU ; un seul GPU suffisant avec l'offloading |
| Abandon de LIBERO-90 | Trop de tâches simultanées ; signal d'apprentissage non focalisé |
| Abandon d'OpenVLA-OFT comme modèle principal | Pivot vers π₀ puis π₀.₅ pour ExpoFT (mieux adapté à la question de recherche) |
| Abandon de LIBERO / RoboCasa comme benchmarks principaux | Partiellement saturés par le SFT ; potentiel OOD limité |
| Abandon du RL on-policy depuis le modèle de base | 0% de succès sur StackCube ; l'on-policy ne peut pas s'amorcer sans SFT ou démonstrations |
| Sélection de ManiSkill (StackCube, PushCube, PickCube) | Maximalement OOD par rapport aux données de pré-entraînement DROID de π₀.₅ |
| Pivot vers l'architecture residual policy d'ExpoFT | Fige le VLA, entraîne une petite residual policy via une Q-function ; évite la limitation de log-probability des modèles à flow-matching ; contourne le problème d'exploration à froid via des démonstrations |
| Reproduction d'ExpoFT dans un dépôt ManiSkill dédié, pas dans RLinf | Garde le code de l'algorithme original proche de l'intact ; seule la couche environnement/données est remplacée |
| `openpi` et `ManiSkill` en git submodules, pas en installations `pip` | Permet des modifications directes et trackées plutôt que des monkeypatches à l'exécution |
| ExpoFT complet tenté en premier, composants retirés ensuite | Méthodologie de retrait progressif pour identifier la vraie supervision minimale nécessaire, plutôt que de deviner à l'avance |
| Passage au contrôle en espace articulaire de bout en bout | Correspond à l'espace d'action de pré-entraînement DROID de π₀.₅ ; le contrôle cartésien écrêtait silencieusement la plupart des actions de démonstration |
| Construction de `validate_demos_full_pipeline.py` | Validation par replay fondée sur des vérités terrain et exacte au niveau des seeds, nécessaire après que le bug d'espace d'action ait montré que les courbes d'entraînement seules ne suffisent pas |
| Erreur standard (pas d'intervalle de Wilson) pour toutes les barres d'erreur | À la demande explicite de l'équipe (Jesse) |
| PickCube-v1 abandonnée (premier temps) | La conception avec marqueur d'objectif caché semblait initialement incompatible avec le schéma d'observation de style DROID (image + proprioception uniquement) |
| PickCube-v1 réintégrée | Un monkeypatch démasquant le marqueur d'objectif pour la caméra capteur a restauré un véritable accès visuel à la position de l'objectif sans ajouter de canal d'état privilégié — l'incompatibilité était corrigible, pas fondamentale |
| Dépôt déplacé de `$SCRATCH` seul vers `$HOME` + artefacts en lien symbolique | Suit les recommandations de stockage mises à jour de Mila ; garde seulement les données volumineuses et régénérables sur le scratch nettoyé automatiquement |
| `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95` adopté comme défaut standard | Résout les plantages OOM récurrents de l'entraînement RL liés à la pré-allocation mémoire par défaut de 75% de JAX |
| Défaut de travail `utd_ratio=2` (contre 20 dans le papier) | Le ratio du papier déstabilise l'entraînement encore plus vite que le ratio réduit dans cette configuration simulée |
| Suppression des entraînements RL catastrophiques pré-correction de `$SCRATCH` | Libère de l'espace disque ; garde seulement les checkpoints SFT validés et les entraînements RL post-correction |

---

## Figures, visuels et démonstrations

Les courbes d'entraînement clés, vidéos d'épisodes et poids de checkpoints pour la Phase 1 sont intégrés directement dans chaque section d'expérience ci-dessus. Les poids finaux de l'entraînement RoboCasa CloseDrawer PPO sont disponibles sur HuggingFace : [`josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO`](https://huggingface.co/josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO).

Les figures et vidéos de la Phase 2 sont également intégrées dans les descriptions d'expériences ci-dessus. Les poids post-SFT de π₀.5 sur les tâches ManiSkill sont disponibles sur HuggingFace :

- [`josh11234/ExpoFT-Pi05-StackCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-StackCube-v1-SFT) (41% de succès sur 200 seeds retenus)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p) (62% de succès sur 200 seeds retenus)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p) (86% de succès sur 200 seeds retenus)
- [`josh11234/ExpoFT-Pi05-PickCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-PickCube-v1-SFT) (22% de succès sur 200 seeds retenus)