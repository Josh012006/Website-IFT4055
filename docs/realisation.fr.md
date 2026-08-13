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

Cette page documente le travail technique réalisé au cours du projet — l'architecture
globale, les deux phases expérimentales, chaque expérience menée, les difficultés
rencontrées en chemin, et les décisions d'ingénierie clés qui ont façonné le pipeline final.
Pour les résultats d'évaluation eux-mêmes et leur interprétation, voir la page Évaluation et
discussion.

## Architecture et démarche générale

### Objectif de recherche

La question centrale de ce projet est de savoir si le fine-tuning par RL peut permettre une
**généralisation hors distribution (OOD)** chez les modèles Vision-Language-Action (VLA) —
c'est-à-dire permettre à un modèle de réussir des tâches maximalement distantes de sa
distribution d'entraînement SFT. Une question secondaire est de savoir si le facteur limitant
principal à l'amélioration par RL est l'**exploration** (la politique ne peut pas découvrir de
signal de récompense) ou l'**optimisation** (la politique ne peut pas s'améliorer étant donné
un signal).

### Aperçu du pipeline

Le projet s'est déroulé en deux phases majeures :

**Phase 1 — RL on-policy (semaines 1 à 7) :** fine-tuning par RL direct utilisant GRPO/PPO
par-dessus OpenVLA-OFT et π₀ dans les environnements LIBERO et RoboCasa. Cette phase a établi
l'hypothèse du goulot d'étranglement d'exploration et les limites des approches on-policy sans
SFT warmup.

**Phase 2 — Politique résiduelle ExpoFT (semaine 8 à aujourd'hui) :** reproduction et
adaptation de l'[architecture ExpoFT](https://pd-perry.github.io/expo-ft/) dans une codebase
dédiée, pilotée par la simulation ManiSkill (`expo-ft`), en remplaçant le pipeline de données
robot réel (DROID) original d'ExpoFT tout en gardant initialement l'algorithme lui-même
inchangé, avant de s'en écarter plus tard de manière ciblée une fois l'instabilité
diagnostiquée (voir plus bas). Le VLA (π₀.₅) est gardé **gelé**, et une petite politique de
correction résiduelle est entraînée par-dessus via une fonction Q, en combinant du RL
off-policy et des démonstrations de motion-planning. Cette conception évite deux limitations
de la Phase 1 : elle ne nécessite pas de log-probabilités d'action valides (que les VLA à
flow-matching comme π₀/π₀.₅ ne peuvent pas fournir, excluant le RL par policy-gradient
standard directement sur le modèle de base) et elle contourne le problème d'exploration à
froid (cold-start) en amorçant le replay buffer avec des démonstrations. Des baselines
on-policy (PPO, GRPO) et une baseline off-policy sans le schéma résiduel/argmax (SAC) ont été
ajoutées plus tard à la même codebase pour comparaison directe (voir plus bas).

La méthodologie choisie est le **retrait progressif de composantes** (progressive component
stripping) : partir de la recette ExpoFT complète (SFT warmup + démonstrations + RL
off-policy) et valider rigoureusement chaque étape avant de retirer des composantes (taille du
jeu de données, le SFT warmup lui-même) afin d'identifier le minimum de supervision réellement
nécessaire.


### Environnement de travail

- **Cluster :** Mila HPC (SLURM), GPU A100/A100L (40 Go/80 Go)
- **Modèles testés :** OpenVLA-OFT (7B), π₀ (3B) (Phase 1) ; π₀.₅ (checkpoint
  `pi05_droid_jointpos`, gelé) + politique résiduelle (Phase 2)
- **Environnements :** LIBERO (Spatial, Long), RoboCasa (CloseDrawer) — Phase 1 ; ManiSkill
  (StackCube-v1, PushCube-v1, PickCube-v1) — Phase 2
- **Algorithmes RL :** GRPO, PPO (Phase 1) ; ExpoFT (VLA gelé + politique résiduelle,
  Q-learning off-policy via un ensemble de critics de style REDQ au départ, puis un critic
  catégoriel/distributionnel — voir plus bas), plus des baselines PPO, GRPO et SAC ajoutées
  directement à la même codebase pour comparaison (Phase 2)
- **Framework :** RLinf (`montrealrobotics/RLinf`) pour la Phase 1 ; dépôt autonome `expo-ft`
  (fork propre d'`openpi` et de `ManiSkill` en tant que submodules git) pour la Phase 2
- **Infrastructure clé :** quatre venvs Python séparés pour gérer les conflits de
  dépendances ; dépôt hébergé sur `$HOME` avec `logs/` et `demos/` symlinkés vers `$SCRATCH` ;
  les gros caches de paquets déplacés vers `$SCRATCH`

---

## Expériences et analyses réalisées

### Phase 1 — Baselines RL on-policy

#### LIBERO-90 GRPO (OpenVLA-OFT)
Première expérience. 1000 pas (~31h40). `success_once` a montré des pics occasionnels
(~0,13) sans tendance stable à la hausse. Initialement attribué à une petite taille de batch ;
confirmé plus tard comme un goulot d'étranglement d'exploration.

![LIBERO-90 GRPO Training Results Success Once]({{ config.site_url }}/images/libero90_grpo_training1_success_once.png)
![LIBERO-90 GRPO Training Results Approx KL]({{ config.site_url }}/images/libero90_grpo_training1_approx_kl.png)

#### RoboCasa CloseDrawer PPO (π₀)
Run de 150 pas avec LoRA (rang 32). Métriques finales : `success_once` ~0,82,
`critic/value_loss` ~0,005, `approx_kl` ~0,036. A démontré que le RL peut améliorer la
performance quand la tâche est dans la distribution — mais CloseDrawer était trop facile
(baseline SFT déjà ~0,6–0,7), laissant peu de place à une contribution du RL.

![RoboCasa CloseDrawer PPO Training Results Success Once]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_success_once.png)
![RoboCasa CloseDrawer PPO Training Results Approx KL]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_approx_kl.png)
![RoboCasa CloseDrawer PPO Training Results Value Loss]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_value_loss.png)

<div style="text-align: center">
    <video width="480" controls poster="{{ config.site_url }}/images/robocasa_close_drawer_thumbnail.png">
        <source src="{{ config.site_url }}/images/robocasa_close_drawer_training1_ppo_step_150.mp4" type="video/mp4">
    </video>
</div>

*π₀ + RoboCasa CloseDrawer — épisode au pas 150 (après entraînement PPO)*

#### LIBERO-Spatial GRPO (OpenVLA-OFT)
Run d'environ 38 pas. `success_once` oscillait entre 0,22 et 0,875 sans convergence claire.
Identifié comme une mauvaise configuration du taux d'apprentissage suite à une réduction du
nombre d'epochs.

![LIBERO-Spatial GRPO Training Results Success Once]({{ config.site_url }}/images/libero_spatial_grpo_training1_success_once.png)
![LIBERO-Spatial GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_spatial_grpo_training1_approx_kl.png)

#### LIBERO-Long GRPO (OpenVLA-OFT)
~150 pas répartis sur 4 sessions reprises (~37h au total). `success_once` oscillait entre
0,06 et 0,19 sans tendance à la hausse. `approx_kl` est resté stable à ~0,035–0,048 tout du
long — les mises à jour de la politique étaient contrôlées, confirmant que le facteur limitant
est l'**exploration**, pas une instabilité d'optimisation.

![LIBERO-Long GRPO Training Results Success Once]({{ config.site_url }}/images/libero_long_grpo_training1_success_once.png)
![LIBERO-Long GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_long_grpo_training1_approx_kl.png)

#### StackCube-v1 PPO et GRPO (modèle OpenVLA de base, on-policy, sans SFT)
Les deux runs ont produit un taux de succès de 0 %. Sans SFT warmup, la politique de base ne
peut produire aucun comportement pertinent pour la tâche, donc l'exploration on-policy ne peut
amorcer aucun signal d'apprentissage. Ce résultat a directement motivé le pivot vers ExpoFT.
Voici les résultats des runs PPO et GRPO sur StackCube-v1 :

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


### Sélection et visualisation des jeux de données OOD

- Création d'un notebook Jupyter pour l'analyse de la distribution des points d'arrivée
  (endpoints) à travers les jeux de données Open X-Embodiment ([Colab](https://colab.research.google.com/github/Josh012006/OpenX-Embodiment-Datasets-Visualization/blob/main/colabs/Open_X_Embodiment_Datasets.ipynb), [Repo](https://github.com/Josh012006/OpenX-Embodiment-Datasets-Visualization))
- Un fit UMAP partagé a été utilisé à travers tous les jeux de données (plutôt qu'un fit par
  jeu de données), car des fits séparés rendraient les coordonnées incomparables entre jeux de
  données
- ManiSkill a été identifié comme le benchmark OOD cible : des tâches comme StackCube-v1 et
  PushCube-v1 sont maximalement distantes des données de SFT/préentraînement du VLA
  (DROID/OXE)
- Note : la distance spatiale L2 entre jeux de données peut être trompeuse en raison de
  référentiels de coordonnées différents entre eux

### Exploration d'hyperparamètres — réduction du temps d'entraînement (Phase 1)

Test d'une augmentation du ratio de replay (`rollout_epoch` 4→8 pour GRPO, `update_epoch` 2→4
pour PPO) pour réduire le temps réel d'entraînement. Résultat : le goulot d'étranglement s'est
déplacé de la collecte de rollouts vers les mises à jour de gradient, rendant l'entraînement
*plus lent* (~27 min/pas contre ~14 min/pas). L'approche efficace consiste plutôt à réduire
directement `max_episode_steps` et `max_epochs`.

---

### Phase 2 — ExpoFT : mise en place du dépôt

L'algorithme ExpoFT a été reproduit dans un dépôt dédié, piloté par ManiSkill
(`~/projects/expo-ft`), plutôt qu'à l'intérieur de RLinf, afin de garder le code de
l'algorithme original aussi inchangé que possible tout en ne remplaçant que la couche
environnement/données. Deux dépendances — `openpi` (l'implémentation de π₀.₅) et `ManiSkill`
— ont été forkées et intégrées comme de véritables submodules git éditables (plutôt que des
dépendances non trackées ou installées via `pip`), permettant des modifications directes et
traçables plutôt que des monkeypatches à l'exécution. `.gitmodules` utilise des URLs HTTPS
pour la compatibilité de clonage public, avec des remotes SSH configurés localement pour les
push.

L'organisation du stockage suit les recommandations du cluster de Mila : le dépôt lui-même
vit sur `$HOME` (sauvegardé, 100 Go), tandis que les artefacts volumineux et régénérables
(`logs/`, `demos/`) sont symlinkés vers `$SCRATCH` (5 To, nettoyé automatiquement). Les gros
caches de paquets (`uv`, `openpi`, `huggingface`, `pip`, `jax`, ~90 Go au total) ont été
déplacés vers `$SCRATCH/cache/` avec des symlinks pointant vers `~/.cache`, libérant le quota
de `$HOME`.

### Phase 2 — Étape SFT : diagnostic et validation de l'espace d'action

Le premier run de SFT warmup (semaine 8), sur des démonstrations StackCube-v1 converties au
format DROID/LeRobot, a produit un taux de succès quasi nul — le bras ne faisait que
s'approcher des cubes sans compléter l'empilement. Cela a d'abord été suspecté d'être un
véritable plafond de difficulté de la tâche, mais une investigation systématique a permis de
la retracer à une **incohérence fondamentale de l'espace d'action** : le préentraînement DROID
de π₀.₅ s'attend à des actions en position articulaire (joint-position) en unités physiques,
alors que le pipeline utilisait le contrôle cartésien `pd_ee_delta_pose` de ManiSkill, causant
un clipping silencieux de la majorité des actions de démonstration hors de leur plage valide.

La correction — faire basculer l'ensemble du pipeline vers un contrôle en espace articulaire
de bout en bout (`control_mode=pd_joint_delta_pos`, checkpoint `pi05_droid_jointpos`, état
`observation/joint_position`, `normalize_action=True`, `output_action_dim=8`, norm stats DROID
depuis GCS) — a immédiatement ramené la grande majorité des actions dans leur plage valide lors
de la validation par replay.

Cela a déclenché une revue systématique plus large (« MAJOR RESET ») qui a fait émerger tout
un ensemble de bugs silencieux supplémentaires avant que les chiffres SFT puissent être
considérés fiables :

- Un script d'évaluation qui écrasait inconditionnellement les norm stats DROID officielles,
  même pour l'évaluation de la baseline
- Un artefact de rescaling d'action en espace cartésien, oublié du mode de contrôle abandonné,
  corrompant une fraction significative des actions désormais en espace articulaire
- Une incohérence de longueur de troncature entre l'environnement et la boucle d'évaluation
- Une incohérence du backend de simulation empêchant une conversion correcte du mode de
  contrôle

Pour regagner confiance dans le pipeline de données, `validate_demos_full_pipeline.py` a été
construit comme un outil rigoureux de replay de bout en bout (utilisant les vraies norm stats,
le vrai wrapper d'environnement, et les seeds exacts enregistrés). Après les corrections
ci-dessus, cet outil a confirmé un **succès de replay des démos de 100 % sur StackCube**,
donnant une réelle confiance que l'espace d'action et la conversion des données étaient enfin
corrects.

Un outillage d'évaluation approprié a aussi été construit à cette étape : des seeds d'épisode
fixes (200 seeds tenues à l'écart, générées une fois à partir d'une seed maîtresse et
réutilisées à travers les checkpoints) pour une comparaison équitable entre checkpoints, des
barres d'erreur standard (`SE = sqrt(p(1-p)/n)`, une approximation normale plutôt qu'un
intervalle de Wilson, selon la préférence explicite de l'équipe), et un script de balayage de
checkpoints reprenable qui reconstruit son graphique après chaque checkpoint.

Des balayages complets de checkpoints SFT sur ces 200 seeds fixes ont confirmé une
**généralisation réelle et validée** (pas de la mémorisation) sur StackCube, avec des taux de
succès augmentant à travers les itérations SFT ; PushCube-v1 (ajoutée comme deuxième tâche,
plus facile, pour un test de cohérence) a même mieux performé.

Voici les courbes de balayage des checkpoints SFT (taux de succès en fonction du
pas/itération d'entraînement, avec barres d'erreur SE, 200 seeds tenues à l'écart) pour
**StackCube-v1** et **PushCube-v1**.

![SFT Checkpoint Sweep Curves PushCube-v1]({{ config.site_url }}/images/curve-sft-pushcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pushcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.₅ sur maniskill PushCube-v1 après SFT warmup*


![SFT Checkpoint Sweep Curves StackCube-v1]({{ config.site_url }}/images/curve-sft-stackcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.₅ sur maniskill StackCube-v1 après SFT warmup*



Et voici des enregistrements d'épisodes du comportement du meilleur checkpoint SFT sur
StackCube-v1 et PushCube-v1, pour accompagner visuellement les courbes de balayage ci-dessus
(même style que la vidéo RoboCasa CloseDrawer en Phase 1).

### Phase 2 — Suite de tâches : StackCube, PushCube et PickCube (abandonnée, puis réintégrée)

PickCube-v1 a été ajoutée comme troisième tâche pour diversifier davantage la suite OOD, mais
a d'abord montré un taux de succès persistant de 0 %. Cela a été retracé à une
**incompatibilité de conception de la tâche** plutôt qu'à un problème de politique ou
d'exploration : PickCube-v1 de ManiSkill cache par défaut le marqueur d'objectif (`goal_site`)
aux caméras du capteur, et bien que `obs['extra']['goal_pos']` soit techniquement accessible
dans le simulateur, le schéma d'observation imitant DROID utilisé tout au long de ce projet
(image + proprioception seulement, correspondant à ce qu'aurait un vrai robot) n'avait aucun
canal pour cela. La politique n'avait donc absolument aucun accès à la position de l'objectif
par épisode — ni visuellement, ni numériquement — rendant la tâche fondamentalement
insoluble en apparence.

PickCube-v1 a été abandonnée dans une première passe plutôt que contournée avec un canal
d'état privilégié, car cela ne correspondrait pas à ce qu'un vrai robot pourrait percevoir.
Cependant, un monkeypatch dévoilant `goal_site` à la caméra du capteur elle-même — donnant à
la politique un accès *visuel* réel à la position de l'objectif, exactement comme le ferait
une vraie caméra pointée sur une cible visible — a ensuite été validé avec succès. Cela a
résolu l'écart d'observabilité sans compromettre le schéma d'observation imitant DROID, et
**PickCube-v1 a été réintégrée dans la suite de tâches active**.

J'ai donc aussi effectué un SFT warmup sur PickCube-v1. Voici les performances :

![SFT Checkpoint Sweep Curves PickCube-v1]({{ config.site_url }}/images/curve-sft-pickcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pickcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.₅ sur maniskill PickCube-v1 après SFT warmup*


**Suite de tâches active pour la suite : StackCube-v1, PushCube-v1 et PickCube-v1.**

### Phase 2 — Étape RL : revue de code et corrections d'infrastructure

Avant de faire confiance à un quelconque résultat RL, l'étape d'entraînement RL a reçu le même
niveau de revue ligne par ligne que l'étape SFT. Cela a fait émerger un autre ensemble de
vrais bugs :

- Un bug d'écrasement des norm stats identique à celui trouvé précédemment dans l'évaluation
  SFT
- Environ 15 flags CLI passés au script d'entraînement qu'il ne lisait silencieusement jamais
  (hyperparamètres orphelins — présents dans les configs, jamais réellement consommés par le
  code)
- Un flag d'écrasement de checkpoint codé en dur qui faisait planter chaque run frais (non
  repris)
- Une incohérence entre deux copies différentes d'un flag d'entraînement clé, causant un crash
  chaque fois qu'il était mis à une valeur non par défaut
- Plusieurs hyperparamètres RL définis dans les configs YAML des tâches qui n'étaient jamais
  réellement lus par le script d'entraînement, retombant silencieusement sur des valeurs par
  défaut différentes définies ailleurs dans la codebase
- La valeur d'un hyperparamètre étant silencieusement interprétée comme du texte plutôt qu'un
  float par le lecteur de config (PyYAML interprète une notation scientifique nue comme `3e-4`
  comme une chaîne de caractères, pas un float)

Séparément, `eval_policy.py` s'est révélé avoir `only_base_actions=True` par défaut, ce qui
signifie qu'il évaluait silencieusement seulement la politique VLA de base gelée — jamais la
politique résiduelle entraînée — sur chaque checkpoint RL évalué jusque-là. Un support
approprié pour charger et évaluer l'agent RL complet entraîné (politique résiduelle et critic
ensemble) a été construit et utilisé pour toutes les évaluations à partir de ce point.

Deux causes distinctes de crashs répétés par manque de mémoire (out-of-memory) durant
l'entraînement RL ont aussi été diagnostiquées et corrigées :
- Un problème de recompilation JAX déclenché par une forme d'entrée incohérente en cours
  d'entraînement (un basculement de structure pytree `None`→dict pour `actor_batch` à travers
  une frontière `jax.jit` amenait JAX à compiler silencieusement et garder un second programme
  XLA résident)
- L'allocateur de mémoire GPU par défaut de JAX ne réservant que 75 % de la carte par défaut,
  plutôt que la carte entière — résolu avec `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95`

Enfin, le logging TensorBoard en cours d'entraînement s'est révélé abandonner
silencieusement la plupart des métriques d'entraînement (seuls le taux de succès et le temps
de boucle étaient visibles, alors que wandb montrait tout correctement) à cause d'un problème
de filtrage par type — les scalaires JAX `jnp.float32` échouent aux vérifications
`isinstance(v, (int, float))` de Python. Corrigé avec un cast explicite en `float()`.

La recette exacte d'augmentation de données d'entraînement du papier a été reproduite à partir
de sa description et confirmée comme déjà correctement implémentée dans la codebase.

### Phase 2 — Étape RL : le résultat central sur l'instabilité

Une fois les bugs d'infrastructure et d'évaluation corrigés, et PickCube-v1 réintégrée après
sa correction d'observabilité, les premiers vrais runs de fine-tuning RL ont été lancés à
partir des checkpoints SFT validés sur les trois tâches actives. Des courbes d'évaluation
complètes (200 seeds fixes tenues à l'écart par checkpoint) ont ensuite été produites à
travers toute une batterie d'expériences de comparaison faisant varier le **ratio
update-to-data (UTD)** et la proportion de chaque batch d'entraînement tirée des données de
démonstration, incluant un run reproduisant les hyperparamètres exacts du papier dans la
longueur d'entraînement validée par le papier lui-même.

**Résultat clé :** chaque configuration RL testée jusqu'ici — sur les trois tâches actives
(StackCube, PushCube, PickCube) et chaque réglage de ratio UTD / ratio de démonstration testé
— **dégrade le taux de succès de la politique préentraînée par SFT plutôt que de
l'améliorer** :

- `utd_ratio=10` : effondrement catastrophique en ~2 000 à 5 000 pas
- `utd_ratio=2` + `offline_ratio=0.5` : retarde l'effondrement à ~35 000–40 000 pas,
  se dégradant finalement vers un succès de ~0,3–0,4 vers 85 000–90 000 pas
- `utd_ratio=20` (la valeur exacte du papier, exécutée dans la plage de pas validée par le
  papier lui-même, ~20 000–24 000 pas) : s'est effondré **plus rapidement et plus sévèrement**
  que le long run à `utd_ratio=2` — ceci réfute l'hypothèse selon laquelle s'entraîner
  simplement dans le budget plus court du robot réel du papier éviterait le problème ;
  `utd_ratio` lui-même, et non la durée totale d'entraînement, semble être le facteur
  déterminant de l'instabilité

La dégradation est systématiquement précédée et accompagnée d'une **instabilité croissante de
la perte d'entraînement du critic** (pics progressifs de critic loss), suggérant que la cause
racine est une fonction de valeur qui se déstabilise progressivement plutôt qu'un bug de
pipeline — ceci a été confirmé plus avant en vérifiant que l'évaluation utilise bien de
manière démontrable la politique résiduelle et le critic entraînés, sans retomber
silencieusement sur la politique de base gelée (voir la correction d'`eval_policy.py`
ci-dessus).

Correspondance d'hyperparamètres confirmée avec le papier original : `offline_ratio=0` (les
démonstrations sont placées dans le seul replay buffer en ligne, pas dans un buffer offline
séparé 50/50 — cette convention appartient à RLPD, pas à ce papier), `hidden_dims=3×256`,
`init_temperature=1.0`, `num_qs=10` / `num_min_qs=2` (ensemble de critics de style REDQ),
`tau=0.005`. Écarts connus par rapport au papier : `batch_size=24` (le papier utilise 64 ;
contraint par la mémoire GPU disponible) et un `edit_scale=0.2` fixe (le papier utilise
0,05–0,2, mis à l'échelle selon la difficulté de la tâche).

Voici une comparaison des métriques d'entraînement à travers les trois configurations de
ratio UTD décrites ci-dessus, pour PushCube-v1, la plus facile des trois tâches :

*Évolution du critic loss durant l'entraînement* :

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=2 + offline_ratio=0.5*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


*Évolution du residual actor loss durant l'entraînement* :
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

            *utd_ratio=2 + offline_ratio=0.5*
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

            *utd_ratio=2 + offline_ratio=0.5*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


Et voici une vidéo montrant la dégradation du comportement de la politique sur StackCube-v1
après ~120 000 pas de fine-tuning RL :
<table>
    <tr>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
                </video>
            </div>

            *Avant fine-tuning RL avec expo*
        </td>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-rl-degradation.mp4" type="video/mp4">
                </video>
            </div>

            *Après fine-tuning RL avec expo*
        </td>
    </tr>
</table>

**Question de recherche ouverte**, soulevée auprès du groupe pour discussion : s'agit-il
d'une limitation fondamentale de l'entraînement de critic off-policy à haut UTD, que les runs
courts du papier sur robot réel n'exposent tout simplement pas faute de durer assez longtemps,
ou est-ce spécifique à cette configuration simulée (jeu de données plus petit, architecture
VLA gelé + politique résiduelle, ou taille de batch contrainte par la mémoire) ? Ceci touche
directement à la question de recherche centrale du projet, puisque cela bloque actuellement la
démonstration de toute contribution positive du RL par-dessus la baseline SFT validée.

**Pistes non testées identifiées pour la suite :** un `edit_scale` plus petit (p. ex. 0,05)
spécifiquement pour StackCube, puisque c'est la tâche la plus exigeante en précision et celle
qui s'est effondrée le plus dramatiquement ; `replan_steps=4` pour StackCube (actuellement à 8
fixe, alors que le papier utilise 4 à 8 selon la tâche) ; et `batch_size=32` comme prochain
test à faible risque (jamais encore essayé).

### Phase 2 — Étape RL : isoler l'instabilité du critic d'un biais structurel de sélection

La phase de travail suivante a repris les pistes ci-dessus et les a testées systématiquement
une à la fois, tout en prenant du recul pour caractériser précisément *pourquoi* la perte
d'entraînement du critic se déstabilisait en premier lieu, plutôt que de continuer à chercher
des hyperparamètres.

**Mécanisme identifié.** ExpoFT sélectionne l'action exécutée en échantillonnant plusieurs
actions candidates (la sortie du VLA de base elle-même, plus plusieurs variantes éditées par
le résiduel), en les notant toutes avec le critic, et en prenant celle avec le meilleur score
— et ce même critic est ensuite utilisé pour amorcer sa propre cible d'entraînement à partir
du candidat qui a été choisi. C'est un biais classique de maximisation/autoréférence
(étroitement lié à la raison d'être du Double Q-learning) : l'argmax va systématiquement
préférer le candidat que le critic surestime, et cette valeur surestimée devient alors part de
la prochaine cible d'entraînement du critic lui-même, renforçant l'erreur. Ce mécanisme a été
identifié comme la cause racine la plus probable de l'instabilité croissante du critic loss
décrite ci-dessus, et est devenu l'hypothèse organisatrice pour le reste de cette phase de
travail.

**Premières mitigations testées, individuellement — aucune n'a résolu l'effondrement :**

- Une ablation à température fixe, geler l'encodeur d'image du critic, du gradient clipping
  sur le critic, et du weight decay sur le critic ont chacun été testés isolément comme
  stabilisateurs candidats. Aucun n'a changé le résultat qualitatif (`target_q_max`
  continuait de croître sans limite, `critic_loss` continuait de devenir de plus en plus
  erratique, `eval/success_rate` continuait de s'effondrer).
- Un `edit_scale` plus petit (0,05, suivant les recommandations du papier lui-même pour les
  tâches exigeantes en précision) a été testé spécifiquement sur StackCube. Même résultat.
- Le critic pretraining — une phase de warm-start, avant tout RL, qui entraîne seulement le
  critic (jamais la politique résiduelle) sur des données de démonstration via le même
  mécanisme de Bellman-backup utilisé plus tard — a été implémenté comme une option
  activable via YAML, suivant une idée similaire proposée dans le papier XQCfD (arXiv
  2605.10734) pour la cohérence critic/actor. Testé isolément : n'a pas résolu
  l'effondrement. Notamment, `target_q_max` durant cette phase de pretraining hors-ligne
  seule *elle-même* recommençait à grimper en cours de route, sur des données de
  démonstration seules — une preuve que l'instabilité ne concerne pas principalement les
  données en ligne que voit le critic, mais le mécanisme d'argmax/bootstrap lui-même.

**Reconstruction de l'architecture du critic (XQC / XQCfD).** Pour tester si la dynamique
d'entraînement du critic lui-même (par opposition au mécanisme de sélection) était le facteur
principal, le critic a été reconstruit de zéro en suivant XQC (arXiv 2509.25174) et XQCfD
(arXiv 2605.10734) : une tête de critic **catégorielle (style C51, support borné fixe)**
remplaçant la cible de régression scalaire non bornée originale, avec **batch normalization
et weight normalization** sur le MLP du critic, et **sans ensemble** (correspondant à la
configuration rapportée par XQCfD elle-même). Le signal de récompense est normalisé par une
estimation courante de sa propre échelle avant d'être utilisé, afin que les bornes de valeur
fixes restent significatives à travers les tâches sans les régler manuellement une à une.
L'architecture originale a été préservée, non modifiée, dans un module séparé plutôt
qu'écrasée, spécifiquement pour que les deux puissent être exécutées côte à côte dans des
conditions par ailleurs identiques.

**Résultat :** les estimations de valeur propres au critic catégoriel se comportent de
manière qualitativement différente et plus saine — elles convergent et se stabilisent dans une
plage bornée plutôt que de grimper indéfiniment, et la perte d'entraînement du critic reste
lisse plutôt que de devenir de plus en plus erratique. Mais `eval/success_rate` s'effondre
toujours de la même manière, essentiellement. C'était un résultat négatif important : il
écarte l'instabilité d'entraînement du critic elle-même comme facteur principal de la
dégradation de la politique.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-target-q-max.png" alt="Curve">

            *target_q_max — architecture de critic originale (jaune) vs. catégorielle (marron)*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-critic-loss.png" alt="Curve">

            *critic_loss — architecture de critic originale (jaune) vs. catégorielle (marron)*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — architecture de critic originale (jaune) vs. catégorielle (marron)*
        </td>
    </tr>
</table>

**Un second bug, indépendant, trouvé et corrigé : le timing de la reward/termination.** Une
revue plus approfondie de la boucle d'entraînement en ligne a révélé que le signal de
récompense et de terminaison d'épisode de l'environnement était lu *avant* que l'action de ce
pas ne soit réellement prise, plutôt qu'après — ce qui signifie que chaque transition stockée
était associée au résultat de l'action *précédente* plutôt qu'à celle qu'elle était censée
représenter. Ceci a été confirmé comme une véritable incohérence structurelle (pas seulement
une convention d'étiquetage) via une petite reproduction contrôlée en dehors de la codebase
principale, et retracé comme n'affectant que la boucle de collecte de données RL en ligne — le
supervised fine-tuning, la génération de démonstrations et le critic pretraining utilisent
tous des pipelines de données séparés qui n'ont jamais été affectés. Les tâches utilisées dans
ce projet reposent sur un signal de récompense façonné (dense), pas sparse, ce qui limite à
quel point la partie continûment variable de la récompense aurait pu être perturbée par ce
décalage de timing en pratique — mais le bonus de succès discret superposé à ce terme de
façonnage est exactement le genre de valeur qu'un tel décalage attribuerait le plus mal.

La correction a été appliquée, et une seconde comparaison contrôlée avant/après a été menée
dans des conditions par ailleurs identiques. **Résultat :** le même schéma que le changement
d'architecture ci-dessus — `target_q_max` se stabilise nettement par rapport à la version non
corrigée, mais `eval/success_rate` s'effondre à nouveau de la même manière, essentiellement,
quoi qu'il arrive.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/reward-timing-fix-comparison-target-q-max.png" alt="Curve">

            *target_q_max — avant (vert) vs. après (rouge) la correction du timing de la récompense*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/reward-timing-fix-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — avant (vert) vs. après (rouge) la correction du timing de la récompense*
        </td>
    </tr>
</table>

**Pourquoi le papier original ne rapporte-t-il pas cela ?** En revisitant la configuration sur
robot réel du papier pour trouver une explication : elle repose sur un opérateur humain
corrigeant activement le robot en temps réel durant l'entraînement (surtout au tout début), ce
qui agit comme un disjoncteur continu contre exactement le mode d'échec décrit ci-dessus — une
mauvaise action surestimée n'a jamais la chance d'être renforcée, puisqu'elle est corrigée
avant de pouvoir être enregistrée. Leurs runs d'entraînement sont aussi considérablement plus
courts (de l'ordre de 8 000 à 20 000 pas, contre 100 000+ ici), et les logs de ce projet
montrent que l'instabilité ne devient sévère que bien au-delà de cette plage. Le mécanisme
sous-jacent dans la formulation même du papier est identique à celui utilisé ici — rien dans
l'algorithme lui-même ne semble le rendre immunisé — ce qui pointe vers le filet de sécurité
human-in-the-loop du papier et son horizon d'entraînement plus court comme l'explication la
plus probable, plutôt qu'une différence fondamentale de méthode.

**Hypothèse principale actuelle et prochaines étapes.** Avec deux corrections indépendantes,
réelles et confirmées, améliorant chacune quelque chose de mesurable sur le critic sans
résoudre le symptôme réel, le mécanisme de sélection lui-même (le même critic qui propose *et*
juge son propre candidat préféré) est l'explication restante la plus probable, par
élimination. Deux directions complémentaires issues du papier XQCfD sont testées dans l'ordre,
pour garder les résultats attribuables à une méthodologie unique et citable avant d'envisager
une correction alternative issue d'une autre ligne de recherche (découpler les étapes de
sélection et d'évaluation, dans le style de Double DQN, en utilisant les réseaux critic online
et target déjà présents dans l'architecture) :

- **Régularisation KL** contre la politique préentraînée, remplaçant le bonus d'exploration
  générique utilisé par défaut — calculée sous forme close entre la distribution propre de la
  politique résiduelle (avant squashing) et une distribution de référence fixe représentant
  « aucune correction ». Un test initial (pas encore complètement isolé — voir plus bas) a
  déjà montré le résultat le plus encourageant de toute cette investigation : le taux de
  succès s'est maintenu de manière cohérente 5 à 10 points de pourcentage au-dessus d'un run
  comparable sans cette régularisation, pour la majeure partie de l'entraînement, et le terme
  de régularisation lui-même est resté une contrainte active et non négligeable tout du long
  plutôt que d'être noyé par le bonus d'exploration s'exécutant en parallèle dans ce premier
  test. Un test isolé (avec le bonus d'exploration entièrement désactivé, correspondant
  exactement à la formulation du papier lui-même) est en cours pour confirmer quelle part de
  ce gain est attribuable spécifiquement au terme de régularisation.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/kl-regularization-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — avec (en rose) vs. sans (en rouge) régularisation KL (test initial, non isolé)*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/kl-regularization-kl-penalty.png" alt="Curve">

            *Le terme de régularisation KL lui-même au cours de l'entraînement*
        </td>
    </tr>
</table>

- **Une architecture de politique « stationnaire »** (pas encore implémentée), la pièce
  restante de la recette XQCfD, destinée à faire en sorte que la politique revienne par défaut
  à un comportement conservateur, à haute incertitude, quand elle agit en dehors de la région
  effectivement couverte par ses démonstrations, plutôt que de continuer à agir avec
  confiance.

L'architecture originale à critic scalaire a aussi été rendue directement sélectionnable (pas
seulement préservée comme référence passive) afin de pouvoir être réexécutée exactement dans
les mêmes conditions que la version catégorielle sur demande, pour comparaison ultérieure.

### Phase 2 — Étendre la même rigueur aux baselines PPO/GRPO

Des baselines on-policy (PPO, GRPO) ont été ajoutées à la même codebase pour fournir des
points de comparaison supplémentaires, partageant la même boucle d'entraînement et la même
infrastructure qu'ExpoFT. Faire tourner ce code pour la première fois a fait émerger plusieurs
bugs numériques indépendants et confirmés — aucun n'avait été détecté auparavant, puisque ce
chemin de code n'avait en réalité jamais été exécuté jusqu'au bout auparavant :

- Une incompatibilité de construction de réseau entre le réseau de fonction de valeur et une
  pièce d'infrastructure partagée qui supposait une forme de réseau différente (de type
  ensemble)
- Une étape de post-traitement de l'action qui rembourrait (padding) la dimensionnalité de
  l'action deux fois, une fois de trop
- Une vérification de repli incorrecte pour une quantité (l'entropie de la politique) qui n'a
  pas de forme close pour la distribution d'action de ce projet, empruntant silencieusement le
  mauvais chemin de code et plantant au lieu d'utiliser l'approximation par échantillonnage
  déjà implémentée exactement pour cette situation
- Un terme de ratio non clippé dans l'objectif PPO/GRPO qui pouvait, après suffisamment de
  dérive de la politique, atteindre silencieusement l'infini en virgule flottante et corrompre
  la perte de tout un batch
- Une instabilité numérique exactement à la frontière de l'espace d'action (compressé par
  tanh), rencontrée en rejouant certaines actions stockées, qui pouvait de la même manière
  corrompre un pas de gradient

Chacun de ces éléments a été confirmé par un petit test ciblé et reproductible, construit
spécifiquement pour l'isoler, plutôt que corrigé sur la seule base d'une inspection du code.
Une protection défensive a aussi été ajoutée par-dessus tout cela : si un gradient calculé se
révèle non fini pour une quelconque raison, cette mise à jour particulière est désormais
entièrement sautée (laissant le modèle et l'état de l'optimiseur intacts) plutôt que de risquer
de corrompre silencieusement l'entraînement, avec un logging de diagnostic ajouté à chaque
étape du calcul afin que toute occurrence future soit immédiatement traçable plutôt que de se
présenter comme un simple crash bien plus tard.

**Un résultat plus fondamental.** Investiguer pourquoi ces baselines ne montraient aucun
signal d'apprentissage dès le tout début de l'entraînement (plutôt que le schéma « commence
bien, puis se dégrade » observé avec ExpoFT) a révélé que leur réseau de politique entraînable
n'avait en réalité jamais été connecté au checkpoint préentraîné par supervision : c'est un
petit réseau, séparé et initialisé aléatoirement, le checkpoint préentraîné n'étant utilisé que
pour convertir ses entrées et sorties vers et depuis le bon format — pas pour lui donner quoi
que ce soit du comportement propre de la politique préentraînée comme point de départ. C'est
une conséquence raisonnable de l'utilisation d'un modèle de base à flow-matching, qui n'expose
pas le genre de vraisemblance d'action dont ces deux algorithmes ont directement besoin, mais
cela signifiait que ces baselines devaient effectivement réapprendre la manipulation depuis
zéro, reproduisant le résultat de la Phase 1 de ce projet (le RL on-policy ne peut pas
s'amorcer sans une politique de départ compétente) malgré l'intention de tester le fine-tuning
on-policy *à partir* du checkpoint SFT. Une option de warm-start par behavior cloning, reprenant
l'idée du critic pretraining utilisée pour ExpoFT, a été ajoutée pour ces baselines afin de
combler cet écart avant d'en tirer de nouvelles conclusions.

---

## Difficultés majeures rencontrées

### Le goulot d'étranglement de l'exploration (Phase 1, fondamental)
Les algorithmes on-policy (PPO, GRPO) ne peuvent générer aucun signal d'apprentissage quand
la politique de base a ~0 % de succès. Cela s'applique à LIBERO-Long (11,9 % de baseline SFT)
et directement à StackCube avec le modèle OpenVLA de base. Aucune stratégie d'exploration
(température, bonus d'entropie) ne suffit sans un SFT warmup préalable ou des démonstrations —
ceci a directement motivé le pivot vers ExpoFT.

### Incohérence de l'espace d'action et le MAJOR RESET (Phase 2)
La plus grande source d'effort perdu de la Phase 2 : une incohérence entre contrôle cartésien
et contrôle en espace articulaire a produit des symptômes (succès quasi nul, bras
s'approchant sans compléter la tâche) identiques en apparence à un véritable plafond de
difficulté de tâche. Le diagnostiquer a nécessité de construire un outillage rigoureux de
validation par replay basé sur le ground-truth plutôt que de faire confiance aux seules courbes
d'entraînement. Plusieurs des bugs supplémentaires trouvés durant la revue systématique qui a
suivi étaient silencieux — ils ne faisaient rien planter, ils produisaient simplement des
chiffres plausibles mais faux — ce qui a rendu une revue systématique complète nécessaire
plutôt qu'une correction ciblée.

### Incompatibilité de conception de la tâche PickCube (résolue)
Un mode d'échec qui ressemblait d'abord à un problème de politique ou d'exploration (0 % de
succès, aucune amélioration) s'est avéré être un écart d'observabilité fondamental entre la
conception de la tâche (marqueur d'objectif caché) et le schéma d'observation imitant DROID
du projet. A nécessité des diagnostics au niveau caméra pour être correctement retracé, et
PickCube-v1 a été abandonnée dans une première passe en conséquence. Un monkeypatch
subséquent dévoilant le marqueur d'objectif à la caméra du capteur a résolu l'écart sans
compromettre le schéma d'observation, et la tâche a été réintégrée. Leçon généralisée :
vérifier l'alignement caméra/observabilité du VLA avec la conception d'une tâche ManiSkill
donnée (marqueurs d'objectif cachés, angle de caméra fixe, présence d'une caméra au poignet)
avant de supposer que de mauvais résultats reflètent la difficulté de la tâche — et avant de
supposer que l'incompatibilité est impossible à corriger.

### Instabilité du critic RL (Phase 2, difficulté centrale — considérablement cernée depuis le premier signalement)
Chaque configuration RL testée jusqu'ici dégrade plutôt qu'elle n'améliore la politique SFT
validée, la dégradation étant initialement corrélée à une instabilité croissante du critic
loss. Deux corrections indépendantes et réelles — une architecture de critic reconstruite avec
des estimations de valeur structurellement bornées, et un bug de timing reward/termination
corrigé dans la boucle de collecte de données — ont chacune été confirmées comme stabilisant
significativement la dynamique d'entraînement propre au critic, mais ni l'une ni l'autre n'a
changé l'effondrement de la performance de la politique. Cela a déplacé l'hypothèse principale,
de l'instabilité d'entraînement du critic elle-même vers le mécanisme de sélection d'action (le
même critic proposant et jugeant sa propre action préférée), qui est désormais testé
directement plutôt qu'inféré par élimination. Une leçon utile, quoique d'abord
contre-intuitive, de cette période de travail : un critic qui a l'air plus sain n'est pas la
même chose qu'une meilleure politique, et les deux doivent être vérifiés séparément plutôt que
supposés évoluer ensemble.

### Premier test de régularisation confondu (résolu, leçon de processus)
Le premier test de régularisation KL contre la politique préentraînée a été exécuté avec le
bonus d'exploration par défaut toujours actif en parallèle, plutôt que désactivé comme le
suggère la formulation même du papier source — ce qui signifie que le résultat encourageant de
ce run ne pouvait pas encore être attribué spécifiquement au terme de régularisation. Repéré
avant d'en tirer une conclusion ; un test de suivi correctement isolé est en cours. Leçon
générale : lors de l'ajout d'un terme à un objectif existant, vérifier explicitement chaque
autre terme avec lequel il pourrait interagir avant de rapporter un résultat, plutôt que de
supposer qu'une nouvelle option est indépendante de ce qui existe déjà.

### Baselines PPO/GRPO silencieusement déconnectées de la politique préentraînée (résolu)
Ces baselines étaient supposées reprendre à partir du checkpoint préentraîné par supervision,
de la même manière que la méthode ExpoFT principale — ce n'était pas le cas. Leur réseau de
politique entraînable était un réseau séparé, initialisé aléatoirement, tout du long, le
checkpoint préentraîné n'étant utilisé que pour le formatage des entrées/sorties. Ceci n'a été
détecté par aucun test qui passe ni aucun run qui plante ; cela n'a émergé qu'une fois les
*résultats* de ces baselines examinés de manière critique (performance persistante quasi nulle
dès le tout premier pas) plutôt que le code supposé correct parce qu'il s'exécutait sans
erreur.

### Infrastructure — cluster, Ray et stockage (Phases 1 et 2)
- Modèle chargé deux fois en mémoire GPU (RolloutGroup + ActorGroup) → OOM sur L40S (44 Go) →
  bascule vers A100L (80 Go) avec `enable_offload: True` (Phase 1)
- Des clusters Ray résiduels d'autres utilisateurs bloquaient l'initialisation sur plusieurs
  nœuds → résolu avec `unset RAY_ADDRESS` et exclusion de nœuds via `#SBATCH --exclude`
  (Phase 1)
- Répertoire `$HOME` trop petit pour les checkpoints (~50 Go chacun) → projet migré vers
  `$SCRATCH`, puis re-migré vers `$HOME` avec seulement les artefacts volumineux et
  régénérables symlinkés vers `$SCRATCH`, selon les recommandations de stockage à jour de
  Mila (Phase 2)
- Conflit de dépendance `robosuite` entre LIBERO et RoboCasa → résolu avec deux venvs séparés
  (Phase 1)
- Crashs GPU JAX par manque de mémoire durant l'entraînement RL, retracés à un problème de
  recompilation/structure pytree et à la pré-allocation par défaut de 75 % de la mémoire par
  JAX (Phase 2)

---

## Ajustements et décisions clés

| Décision | Justification |
|---|---|
| Abandon du L40S au profit de l'A100L (80 Go) | VRAM du L40S insuffisante pour des modèles chargés deux fois en mémoire |
| Abandon du multi-GPU au profit d'un seul GPU + `enable_offload` | Problèmes de timeout NCCL en multi-GPU ; un seul GPU suffisant avec l'offloading |
| Abandon de LIBERO-90 | Trop de tâches simultanées ; signal d'apprentissage peu ciblé |
| Abandon d'OpenVLA-OFT comme modèle principal | Pivot vers π₀ puis π₀.₅ pour ExpoFT (mieux adapté à la question de recherche) |
| Abandon de LIBERO / RoboCasa comme benchmarks principaux | Partiellement saturés par le SFT ; potentiel OOD limité |
| Abandon du RL on-policy depuis le modèle de base | 0 % de succès sur StackCube ; le on-policy ne peut s'amorcer sans SFT ni démonstrations |
| Sélection de ManiSkill (StackCube, PushCube, PickCube) | Maximalement OOD par rapport aux données de préentraînement DROID de π₀.₅ |
| Pivot vers l'architecture résiduelle ExpoFT | Gèle le VLA, entraîne une petite politique résiduelle via une fonction Q ; évite la limitation de log-probabilité des modèles à flow-matching ; contourne le problème d'exploration à froid via des démonstrations |
| Reproduction d'ExpoFT dans un dépôt ManiSkill dédié, pas à l'intérieur de RLinf | Garde le code de l'algorithme original quasiment inchangé ; seule la couche environnement/données est remplacée |
| `openpi` et `ManiSkill` en submodules git, pas en installations `pip` | Permet des modifications directes et traçables plutôt que des monkeypatches à l'exécution |
| ExpoFT complet tenté en premier, composantes retirées ensuite | Méthodologie de retrait progressif pour identifier le véritable minimum de supervision nécessaire, plutôt que de deviner à l'avance |
| Passage à un contrôle en espace articulaire de bout en bout | Correspond à l'espace d'action du préentraînement DROID de π₀.₅ ; le contrôle cartésien clippait silencieusement la plupart des actions de démonstration |
| Construction de `validate_demos_full_pipeline.py` | Validation par replay ground-truth, seed exacte, nécessaire après que le bug d'espace d'action a montré que les courbes d'entraînement seules ne sont pas fiables |
| Erreur standard (pas d'intervalle de Wilson) pour toutes les barres d'erreur | Selon la préférence explicite de l'équipe (Jesse) |
| PickCube-v1 abandonnée (première passe) | Conception de tâche à marqueur d'objectif caché initialement incompatible en apparence avec le schéma d'observation imitant DROID (image + proprioception seulement) |
| PickCube-v1 réintégrée | Un monkeypatch dévoilant le marqueur d'objectif à la caméra du capteur a restauré un accès visuel réel à la position de l'objectif sans ajouter de canal d'état privilégié — l'incompatibilité était corrigible, pas fondamentale |
| Dépôt déplacé de `$SCRATCH` seul vers `$HOME` + artefacts symlinkés | Suit les recommandations de stockage à jour de Mila ; ne garde que les grosses données régénérables sur le scratch auto-nettoyé |
| `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95` adopté comme défaut permanent | Résout les crashs OOM récurrents de l'entraînement RL liés à la pré-allocation par défaut de 75 % de mémoire par JAX |
| Défaut de travail de `utd_ratio=2` (vs. 20 dans le papier) | Le ratio propre au papier déstabilise l'entraînement encore plus vite que le ratio réduit dans cette configuration simulée |
| Suppression des runs RL catastrophiques pré-correction depuis `$SCRATCH` | Libère de l'espace disque ; ne garde que les checkpoints SFT validés et les runs RL post-correction |
| Reconstruction du critic en architecture catégorielle (support borné), selon XQC/XQCfD | Teste directement si la croissance non bornée de la valeur du critic était la cause racine de la dégradation de la politique, plutôt que de continuer à chercher des hyperparamètres autour de l'architecture originale |
| Préservation de l'architecture de critic originale dans un module séparé et toujours exécutable plutôt que de l'écraser | Garde une comparaison A/B propre contre la reconstruction catégorielle disponible à tout moment, plutôt qu'une migration à sens unique |
| Normalisation adaptative (estimation courante) de la récompense plutôt que des bornes de valeur fixes et réglées à la main | Garde le support fixe du critic catégoriel significatif à travers les tâches sans réglage par tâche, et sans besoin de reprojeter le support en cours d'entraînement |
| Test des mitigations propres au papier XQCfD (critic pretraining, critic catégoriel, régularisation KL, architecture stationnaire) dans l'ordre, avant une correction alternative issue d'une autre ligne de recherche | Garde les résultats attribuables à une méthodologie unique et citable plutôt que de mélanger des approches issues de papiers non liés |
| Régularisation KL implémentée comme un terme additionnel et activable plutôt qu'un remplacement pur et simple du bonus d'exploration existant | Préserve la capacité de reproduire la formulation exacte du papier (en désactivant l'autre terme) tout en gardant les deux disponibles indépendamment pour l'ablation |
| Ajout d'une option de warm-start par behavior cloning pour les baselines PPO/GRPO | Répond à la découverte que leur politique entraînable n'avait autrement aucun lien avec le checkpoint préentraîné |
| Ajout d'une garde défensive contre les gradients non finis à l'étape de mise à jour PPO/GRPO | Empêche un futur problème numérique, encore inconnu, de corrompre silencieusement un run d'entraînement, indépendamment du fait que sa cause racine ait été trouvée ou non |

---

## Figures, visuels et démonstrations

Les principales courbes d'entraînement, vidéos d'épisodes et poids de checkpoints pour la
Phase 1 sont intégrés directement dans chaque section d'expérience ci-dessus. Les poids finaux
du run RoboCasa CloseDrawer PPO sont disponibles sur HuggingFace :
[`josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO`](https://huggingface.co/josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO).

Les figures et vidéos de la Phase 2 sont aussi intégrées dans les descriptions d'expériences
ci-dessus. Les poids post-SFT du Pi0.5 sur les tâches maniskill sont disponibles sur
HuggingFace :

- [`josh11234/ExpoFT-Pi05-StackCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-StackCube-v1-SFT) (41 % de succès sur 200 seeds tenues à l'écart)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p) (62 % de succès sur 200 seeds tenues à l'écart)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p) (86 % de succès sur 200 seeds tenues à l'écart)
- [`josh11234/ExpoFT-Pi05-PickCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-PickCube-v1-SFT) (22 % de succès sur 200 seeds tenues à l'écart)