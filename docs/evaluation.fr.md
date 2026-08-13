---
title: Évaluation et discussion
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Évaluation et discussion

Cette page documente comment le projet a été évalué, ce que les résultats ont montré, et
comment nous les interprétons. Elle résume les preuves derrière la conclusion centrale du
rapport : ExpoFT a systématiquement échoué à améliorer la baseline SFT jusqu'à ce qu'un bug
silencieux de configuration de la récompense soit trouvé et corrigé — et même après cela,
une seule des trois tâches en a clairement bénéficié.

## Démarche d'évaluation

Le projet s'est déroulé en deux phases expérimentales, chacune évaluée différemment.

- **Métrique principale tout au long :** le taux de succès de la tâche.
- **Phase 1 (RL on-policy, framework RLinf)** surveillait aussi `env/return`,
  `actor/policy_loss` et `actor/approx_kl` pour suivre la santé de l'optimisation
  indépendamment de la performance de la tâche.
- **Phase 2 (ExpoFT, off-policy)** surveillait aussi des diagnostics du critic :
  `training/target_q_max`, `training/critic_loss`, et un diagnostic maison de **misrank rate**
  comparant l'action choisie par le critic à une politique de référence validée séparément et
  très performante.
- **Rigueur :** chaque checkpoint est évalué sur les 200 mêmes états initiaux fixes, avec
  erreurs standards rapportées, afin que les différences observées reflètent le comportement
  de la politique plutôt que le bruit stochastique de l'environnement.

Les trois tâches d'évaluation proviennent de ManiSkill et partagent un cadre de manipulation
riche en contacts et exigeant en précision :

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/pickcube.png" alt="Curve">

            *PickCube-v1*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/pushcube.png" alt="Curve">

            *PushCube-v1*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/stackcube.png" alt="Curve">

            *StackCube-v1*
        </td>
    </tr>
</table>

*Les trois environnements ManiSkill utilisés pour l'évaluation. PickCube-v1 est le plus exigeant en exploration ; PushCube-v1 le plus facile ; StackCube-v1 le plus exigeant en précision.*

## Résultats obtenus

### Baseline SFT : la barre que le RL doit franchir

Avant tout fine-tuning par RL, les taux de succès SFT validés sur 200 seeds fixes étaient :

| Tâche | Meilleur taux de succès | Seeds d'évaluation |
|---|---|---|
| StackCube-v1 | 49 % | 200 |
| PushCube-v1 | 96 % | 200 |
| PickCube-v1 | 27 % | 200 |

Atteindre ces chiffres a d'abord nécessité de corriger plusieurs bugs silencieux du pipeline
(une incohérence de l'espace d'action, un override des statistiques de normalisation, un
artefact de rescaling d'action oublié), chacun confirmé via un outil dédié qui rejoue les
démonstrations de bout en bout et vérifie une fidélité de replay de 100 %. La trajectoire
complète du SFT à travers les checkpoints ressemblait à ceci :

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/curve-sft-pickcube.png" alt="Curve">

            *Balayage des checkpoints SFT : taux de succès en fonction du pas d'entraînement RL pour PickCube*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/curve-sft-pushcube.png" alt="Curve">

            *Balayage des checkpoints SFT : taux de succès en fonction du pas d'entraînement RL pour PushCube*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/curve-sft-stackcube.png" alt="Curve">

            *Balayage des checkpoints SFT : taux de succès en fonction du pas d'entraînement RL pour StackCube*
        </td>
    </tr>
</table>

*Notez les échelles d'axe y différentes selon la tâche — PushCube grimpe au-delà de 90 %, StackCube vers ~45 %, PickCube vers ~25 %.*

### Le résultat central : ExpoFT dégrade toutes les configurations

Une fois la baseline SFT validée, ExpoFT a été exécuté à partir de ces checkpoints sur
plusieurs ratios update-to-data (UTD). Chaque configuration **dégrade** le taux de succès
plutôt que de l'améliorer :

| Configuration | Ratio UTD | Pas | Succès initial | Succès final |
|---|---|---|---|---|
| Configuration originale (papier) | 20 | 20k | 62 % | 1 % |
| Mises à jour réduites | 2 | 120k | 62 % | 0 % |

Un ratio UTD plus élevé s'effondre plus rapidement ; un ratio plus faible ne fait que retarder
l'effondrement — il ne l'empêche jamais. La durée totale d'entraînement n'était pas le
facteur déterminant ; le ratio UTD l'était.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-eval-success-pushcube.png" alt="Curve">

            *Effondrement du taux de succès sous ratio UTD = 2 (120K pas)*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/config3-eval-success-pushcube.png" alt="Curve">

            *Effondrement du taux de succès sous ratio UTD = 20 (20K pas, configuration du papier)*
        </td>
    </tr>
</table>

*Les deux configurations s'effondrent d'un taux de succès de départ de ~60–100 % vers des valeurs à un chiffre — le ratio UTD le plus élevé (à droite) s'effondre plus rapidement et plus sévèrement.*

### Diagnostiquer le mécanisme d'échec

La dégradation s'accompagnait systématiquement d'une instabilité croissante de la perte du
critic, ce qui pointait d'abord vers l'instabilité d'entraînement du critic comme cause
probable. Cinq expériences de diagnostic contrôlées ont été menées pour tester cela et cerner
la vraie cause :

| Expérience | Observation | Interprétation |
|---|---|---|
| Encodeur du critic gelé | Dérive similaire de la valeur Q | Non causé par une dérive de représentation |
| Budget résiduel quasi nul | Dégradation similaire | Non causé par un mauvais apprentissage de la politique résiduelle |
| Balayage de candidats (N=2 vs. N=8) | Dérive de target-Q quasi identique | Non déterminé par la taille du pool de candidats |
| Reconstruction du critic catégoriel | Valeurs mieux comportées, même effondrement | Pas seulement un problème d'architecture du critic |
| **Diagnostic misrank** | **0,94–1,00 dès le tout premier checkpoint** | **Confirme directement un biais de sélection d'action** |

Le diagnostic misrank a été déterminant. Pour des états d'entraînement réels, nous avons
comparé le score du critic pour l'action qu'il sélectionne lui-même à son score pour l'action
qu'aurait prise à la place une politique de référence validée séparément, à 96 % de succès :

![misrank_rate par checkpoint RL, saturé entre 0,94 et 1,00 dès le premier checkpoint]({{ config.site_url }}/images/fig_misrank_rate.png)
*Le critic préférait son propre choix, pourtant pire, 94 à 100 % du temps, saturé dès le tout premier checkpoint RL évalué — un biais classique de maximisation / autoréférence : le même critic qui note les candidats amorce aussi sa propre cible d'entraînement à partir du candidat sélectionné.*

### Tentatives de correction

Guidées par ce diagnostic, six mécanismes de stabilisation issus de la littérature ont été
testés un à la fois par-dessus un critic catégoriel :

| Mécanisme | Problème ciblé | Résultat |
|---|---|---|
| Critic pretraining (warm-start sur démos) | Améliorer l'initialisation des valeurs | Amélioration partielle |
| Régularisation KL vers la politique de base | Limiter la dérive résiduelle | Amélioration partielle |
| Découplage de sélection Double-Q | Casser l'autoréférence du critic | Pas d'amélioration significative |
| Clipped double-Q (style TD3) | Réduire la surestimation de Q | Pas d'amélioration significative |
| Politique conservatrice HetStat | Revenir à des actions conservatrices hors distribution | Aucune amélioration |
| Budget résiduel / ratio de démos plus grand | Améliorer l'exploration et la qualité des données | Aucune amélioration |

!!! warning "Aucune de ces méthodes n'a résolu l'effondrement à elle seule"
    C'est un résultat négatif important en soi — il pointe vers quelque chose de plus
    fondamental qu'une simple astuce de stabilisation.



![eval/success_rate pour toutes les tentatives de correction du critic catégoriel]({{ config.site_url }}/images/categorical_critic_corrections.png)
*Taux de succès à travers toutes les tentatives de correction du critic catégoriel.*

### Le tournant : un bug silencieux de configuration de la récompense

Une vérification de cohérence sur les valeurs rapportées par le critic (habituellement entre
13 et 19, invraisemblables sous une récompense binaire sparse) a révélé deux bugs
indépendants qui se combinaient :

1. L'environnement n'avait **jamais été explicitement configuré en récompense sparse** — il
   utilisait silencieusement le mode dense et façonné par défaut de ManiSkill (confirmé en
   rejouant une démonstration et en trouvant 0 % de pas à récompense exactement nulle).
2. La normalisation de la récompense du critic catégoriel divisait par une estimation courante de sa
   propre échelle — à mesure que le succès chutait, cette estimation rétrécissait, gonflant la
   récompense effective des rares succès restants (une boucle autorenforçante).

Les deux ont été corrigés. Cela n'invalide pas la preuve du misrank rate (une comparaison
relative, à critic constant) — mais affaiblit substantiellement le cadrage antérieur selon
lequel les magnitudes de valeurs brutes étaient absurdes en elles-mêmes.

### Évaluation finale après correction

Répéter la comparaison des ratios UTD sous la récompense sparse corrigée a **inversé** le
résultat précédent : UTD = 20 est devenu la configuration la plus stable et la plus
performante, tandis qu'UTD = 1 se dégradait régulièrement.

![eval/success_rate sous ratio UTD = 1 vs. ratio UTD = 20, récompense sparse, configuration corrigée]({{ config.site_url }}/images/utd_reversal_after_fix.png)
*Lecture : le ratio UTD amplifie le signal d'entraînement présent — une erreur autorenforçante avant la correction, un apprentissage sain après.*

Tester la configuration corrigée (UTD = 20) sur les trois tâches a donné des progrès réels
mais mitigés :

| Configuration | StackCube-v1 | PushCube-v1 | PickCube-v1 |
|---|---|---|---|
| Baseline SFT | 41 % | 62 % | 22 % |
| ExpoFT (critic MSE, récompense corrigée) | 32 % | **90 %** | 16 % |

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/final-curve-pickcube.png" alt="Curve">

            *Courbe d'entraînement finale après la correction de la récompense pour PickCube-v1*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/final-curve-pushcube.png" alt="Curve">

            *Courbe d'entraînement finale après la correction de la récompense pour PushCube-v1*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/final-curve-stackcube.png" alt="Curve">

            *Courbe d'entraînement finale après la correction de la récompense pour StackCube-v1*
        </td>
    </tr>
</table>

*PushCube-v1 dépasse clairement sa baseline SFT. StackCube-v1 et PickCube-v1 ne s'effondrent plus vers zéro, mais oscillent autour de leur performance de départ plutôt que de s'améliorer régulièrement — notez les échelles d'axe y différentes entre les trois panneaux.*

## Analyse et interprétation

- **Le RL on-policy est limité principalement par l'exploration.** Quand la politique de base
  a un succès quasi nul sur une tâche, elle ne génère aucune trajectoire réussie dont on
  pourrait apprendre — l'optimisation elle-même restait numériquement saine (`approx_kl`
  stable), mais le succès n'a jamais montré de tendance à la hausse.
- **Le RL off-policy (ExpoFT) résout ce problème de démarrage à froid via les démonstrations**,
  mais cela déplace le facteur limitant plutôt que de l'éliminer : l'entraînement devient très
  sensible à la formulation de la récompense, à l'architecture du critic et au ratio
  update-to-data.
- **La calibration de la récompense n'est pas un détail périphérique — elle a changé toute la
  dynamique d'entraînement.** Le même ratio UTD qui était le plus déstabilisant sous la
  récompense mal calibrée est devenu le plus stabilisant après correction.
- Globalement, les expériences ne donnent pas de réponse définitive quant à savoir si le RL
  peut apprendre de manière fiable un comportement réellement nouveau au-delà de la
  distribution SFT, mais elles font émerger des limitations concrètes et pratiques qui doivent
  être résolues avant que le fine-tuning par RL puisse étendre de façon fiable les capacités
  des VLA.

## Limites du projet

- **Portée étroite :** les expériences se limitent à la simulation ManiSkill sur trois tâches
  de manipulation — une tranche étroite des environnements robotiques possibles. Les résultats
  obtenus ici pourraient ne pas se transférer directement à des robots physiques.
- **Variabilité d'un run à l'autre :** l'entraînement RL a montré une variance non négligeable
  entre des répétitions identiques — de l'ordre d'une différence relative de ~35 % sur le taux
  de succès final pour une même configuration. Chaque comparaison basée sur un seul run dans ce
  projet comporte donc une réelle incertitude ; une évaluation multi-seed plus poussée serait
  nécessaire pour confirmer quelles améliorations partielles sont robustes.
- **Protocoles d'évaluation incohérents entre les phases :** la Phase 1 bénéficiait d'une
  collecte on-policy parallélisée, tandis que la Phase 2 (ExpoFT) reposait sur une interaction
  en ligne séquentielle — compliquant toute comparaison directe d'efficacité entre les deux
  approches.
- **Des contraintes de temps** ont laissé certaines pistes prometteuses inexplorées, notamment
  recalibrer le support de valeurs du critic catégoriel pour le régime de récompense sparse.
- **Pattern récurrent — des bugs silencieux, pas des crashs :** les incohérences d'espace
  d'action, les différences de configuration de récompense, et les outils d'évaluation faisant
  des suppositions spécifiques à une architecture ont tous produit des chiffres plausibles mais
  incorrects, jamais un crash. La solution : une validation indépendante — un outil dédié a
  rejoué les démonstrations de bout en bout à travers les vraies statistiques de normalisation,
  l'environnement et les seeds, confirmant une fidélité de replay de 100 % avant de faire
  confiance à tout résultat en aval. La leçon plus large : la reproductibilité en fine-tuning
  RL exige de vérifier chaque interaction entre l'environnement, la récompense, le pipeline de
  données et l'architecture du modèle — pas seulement de rapporter des hyperparamètres.