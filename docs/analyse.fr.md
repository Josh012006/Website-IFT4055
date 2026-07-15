---
title: Études préliminaires
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Études préliminaires

<!-- > :bulb: Cette page documente les analyses, recherches et explorations réalisées durant le projet.  
> Elle ne remplace pas le rapport final, mais permet de conserver une trace structurée de votre démarche et de votre progression.

## Structure suggérée

> La structure suivante est donnée à titre indicatif.  
> Vous pouvez l’adapter selon la nature de votre projet. -->

### Contexte et compréhension du domaine

Voici quelques concepts clés utilisés à travers ce projet :

#### Concepts clés

- **Modèle d'intelligence artificielle (IA)** : Système informatique conçu pour effectuer des tâches qui nécessitent normalement l'intelligence humaine, telles que la compréhension du langage naturel, la reconnaissance d'images ou la prise de décisions.
- **Algorithme d'optimisation** : Méthode utilisée pour ajuster les paramètres d'un modèle d'IA afin d'améliorer ses performances sur une tâche donnée.
- **Apprentissage par renforcement (RL)** : Paradigme d’apprentissage où un agent apprend à prendre des décisions en interagissant avec un environnement, guidé par un signal de récompense. Ce qui est appris, c'est une politique de prise de décision qui maximise la récompense cumulée à long terme.
- **Algorithme on-policy vs off-policy** : En apprentissage par renforcement, les algorithmes on-policy apprennent à partir des actions prises par la politique actuelle, tandis que les algorithmes off-policy peuvent apprendre à partir de données générées par une politique différente.
- **Fine-tuning** : Processus d’adaptation d’un modèle pré-entraîné à une tâche spécifique en continuant son entraînement sur des données pertinentes.
- **Modèles vision-langage-action (VLA)** : Grands modèles capables de comprendre plusieurs types d’informations (comme des images et du texte), puis de générer des actions motrices pour des applications en robotique.

#### Algorithmes de RL

Voici une liste non exhaustive d’algorithmes de reinforcement learning pertinents pour ce projet :

- **PPO (Proximal Policy Optimization)** : Algorithme de policy gradient on-policy qui stabilise l’apprentissage en contraignant la mise à jour de la politique via une fonction de clipping, limitant ainsi les variations trop importantes entre deux itérations.
- **SAC (Soft Actor-Critic)** : Algorithme off-policy basé sur l’actor-critic qui optimise simultanément la récompense attendue et l’entropie de la politique, favorisant un compromis entre performance et exploration.
- **GRPO (Group Relative Policy Optimization)** : Variante récente de policy optimization utilisée notamment dans le contexte des modèles de langage, qui estime les avantages de manière relative au sein d’un groupe d’échantillons, réduisant ou éliminant le besoin d’un critic explicite dans certaines formulations.

#### Modèles de VLA

Voici quelques modèles de vision-langage-action (VLA) pertinents pour ce projet :

- **OpenVLA** : Modèle open source de vision-langage-action conçu pour exécuter des tâches robotiques à partir d’instructions en langage naturel et d’entrées visuelles multimodales.
- **π0 (Pi0)** : Modèle de fondation pour la robotique développé dans le contexte des travaux récents sur les VLAs, visant à apprendre des politiques robotiques généralisables à partir de données multimodales.
- **GR00T** : Modèle de fondation pour la robotique humanoïde développé par NVIDIA, conçu pour faciliter l’apprentissage de comportements robotiques généralisés à partir d’entrées visuelles et linguistiques.

#### Environnements de test

Voici quelques environnements de simulation utilisés pour l’entraînement et l’évaluation des modèles de RL dans le contexte de la robotique :

- **LIBERO** : Benchmark de robotique conçu pour évaluer les capacités de généralisation et de transfert des politiques robotiques sur des tâches variées et compositionnelles.
- **RoboCasa** : Environnement de simulation centré sur des tâches domestiques réalistes, permettant d’évaluer les capacités de manipulation et d’interaction avec des objets du quotidien.
- **ManiSkill** : Framework de simulation robotique orienté vers les tâches de manipulation, proposant des environnements complexes nécessitant précision et planification.


<!-- ### Revue de littérature ou état de l’art

> Présentez les travaux, approches ou solutions existantes pertinents au projet.  
> Comparez les approches identifiées et discutez de leurs forces, limites ou différences.

### Analyse du problème

> Décomposez le problème étudié :
>
> - contraintes ;
> - hypothèses ;
> - limites des approches existantes ;
> - besoins ou critères importants.

### Explorations et expérimentations préliminaires

> Présentez les premiers tests, prototypes, analyses ou expérimentations réalisés afin de mieux comprendre le sujet.

### Choix retenus

> Expliquez les choix méthodologiques, techniques ou conceptuels retenus pour la suite du projet et justifiez-les. -->

### Référence

<!-- > Incluez les principales références utilisées :
>
> - articles scientifiques ;
> - documentation ;
> - ouvrages ;
> - projets similaires ;
> - ressources techniques. -->

#### Articles scientifiques

Pour mieux aborder ce projet, voici une liste des articles scientifiques que j'ai consulté : 

- ["OpenVLA: An Open-Source Vision-Language-Action Model"](https://arxiv.org/abs/2406.09246) : Lecture terminée
- ["LoRA: Low-Rank Adaptation of Large Language Models"](https://arxiv.org/abs/2106.09685) : En cours de lecture (10%)
- ["Training language models to follow instructions with human feedback"](https://arxiv.org/abs/2203.02155) : En cours de lecture (10%)
- ["Fine-Tuning Vision-Language-Action Models: Optimizing Speed and Success"](https://arxiv.org/abs/2502.19645) : En cours de lecture (10%)
- ["LIBERO-Plus: In-depth Robustness Analysis of Vision-Language-Action Models"](https://arxiv.org/abs/2510.13626) : En cours de lecture (10%)
- ["VLAs Are Confined yet Capable of Generalizing to Novel Tasks"](https://openreview.net/forum?id=jtrhwfgseW) : Lecture terminée
- ["Curiosity-driven Exploration by Self-supervised Prediction"](https://arxiv.org/abs/1705.05363) : Lecture prévue
- ["Exploration by Random Network Distillation"](https://arxiv.org/abs/1810.12894) : Lecture prévue
- ["EXPO-FT: Sample-Efficient Reinforcement Learning Finetuning for Vision-Language-Action Models"](https://arxiv.org/abs/2605.25477) : Lecture terminée
- ["Soft Actor-Critic: Off-Policy Maximum Entropy Deep Reinforcement Learning with a Stochastic Actor"](https://arxiv.org/abs/1801.01290) : En cours de lecture (10%)
- ["XQCfD: Accelerating Fast Actor-Critic Algorithms with Prior Data and Prior Policies"](https://arxiv.org/abs/2605.10734) : En cours de lecture (80%)
