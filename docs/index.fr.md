---
title: Vue d'ensemble du projet
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Vue d'ensemble du projet

!!! info "Informations générales"
    **Session**: Été 2026  

    **Auteur(s)**: [Josué Saint-martin Junior Mongan](https://mila.quebec/en/directory/josue-mongan) (Matricule 20290870)

    **Thème(s)**: Stage de recherche en - **Apprentissage par renforcement pour le fine-tuning de modèles vision-langage (fine-tuning, RL, VLAs)**

    **Superviseur(s)**: [Glen Berseth](https://mila.quebec/fr/annuaire/glen-berseth)

    **Collaborateur(s):** [Florence Cloutier](https://mila.quebec/fr/annuaire/florence-cloutier), [Albert Zhan](https://mila.quebec/fr/annuaire/albert-zhan), [Jesse Silverberg](https://mila.quebec/fr/annuaire/jesse-silverberg)

## **Description du projet**

Ce projet s'inscrit dans le domaine émergent du fine-tuning par apprentissage par renforcement (RL) appliqué 
aux modèles de vision-langage-action (VLA). Les VLAs sont des modèles de fondation multimodaux qui combinent 
compréhension visuelle, raisonnement linguistique et génération d'actions motrices pour des applications en robotique. 
<br />

Bien que ces modèles atteignent de bonnes performances sur des tâches simples via l'apprentissage par imitation (SFT), 
leurs capacités de généralisation et de raisonnement sur des tâches complexes restent limitées. Le RL post-SFT 
représente une piste prometteuse pour dépasser ces limitations, mais son application aux VLAs est encore peu explorée et 
soulève des défis fondamentaux d'optimisation, de stabilité d'entraînement et de conception de récompenses.

### Contexte

<!-- > Présentez le domaine de recherche dans lequel s’inscrit votre projet.  
> Expliquez brièvement pourquoi ce sujet est pertinent en informatique et quels enjeux généraux il soulève. -->

L'apprentissage par imitation a permis des avancées significatives en robotique, mais requiert des démonstrations 
expertes en grande quantité, ce qui est coûteux et peu scalable. Le RL offre une alternative en permettant au robot 
d'apprendre directement par interaction avec son environnement, guidé par un signal de récompense. 
Appliqué aux VLAs, le RL pourrait permettre à ces modèles d'apprendre au-delà de leur distribution 
d'entraînement initiale, en développant des comportements que l'imitation seule ne peut pas capturer. 
Cependant, l'optimisation RL sur des modèles de grande taille reste un défi ouvert, notamment en termes de 
stabilité algorithmique, d'efficacité computationnelle, et de capacité à généraliser.

### Problématique

<!-- > Décrivez la problématique, la question de recherche ou l’hypothèse que votre projet cherche à explorer.  
> La question doit être suffisamment claire pour orienter la revue de littérature, la méthodologie et l’analyse. -->

- **Le fine-tuning par RL peut-il permettre aux VLAs d'apprendre au-delà de leur distribution de données d'entraînement SFT ?**
- **L'exploration ou l'optimisation constitue-t-elle le principal obstacle au fine-tuning RL des VLAs ?** 
- **Quels algorithmes, architectures et types de tâches sont les plus propices à ce type d'amélioration ?**
- **Comment le RL pourrait améliorer la capacité du modèle à développer de véritables capacités de raisonnement plutôt que de simplement mémoriser des comportements ?**

### État de l’art préliminaire

<!-- > Présentez brièvement les approches, travaux ou concepts importants déjà identifiés.  
> L’objectif n’est pas encore de faire une revue complète, mais de situer votre projet par rapport aux travaux existants. -->

Le fine-tuning par apprentissage par renforcement (RL) des grands modèles de langage (LLM) a connu un succès notable avec des algorithmes tels que PPO et GRPO, notamment dans des travaux comme DeepSeek-R1 et InstructGPT.
L'extension de ces approches aux modèles vision-langage-action (VLA) est plus récente et introduit des défis supplémentaires en raison de leur nature multimodale et orientée action.
<br />
 
Des frameworks récents tels que [RLinf](https://github.com/RLinf/RLinf) fournissent une infrastructure unifiée pour appliquer le RL aux modèles VLA, supportant plusieurs architectures (OpenVLA, π0, GR00T) et environnements (LIBERO, ManiSkill, RoboCasa).
<br />
 
Les premiers résultats indiquent que le RL peut significativement améliorer la performance par rapport au fine-tuning supervisé (SFT), particulièrement sur des tâches à long horizon et multi-étapes. Cependant, les conditions optimales pour cette amélioration restent mal comprises. Des travaux récents ([LIBERO-OOD, ICLR 2026](https://openreview.net/forum?id=jtrhwfgseW) ; [LIBERO-Plus, arXiv 2510.13626](https://arxiv.org/abs/2510.13626)) montrent également que les modèles entraînés par SFT souffrent de problèmes de généralisation hors distribution, mémorisant souvent les configurations d'entraînement plutôt que de développer une compréhension robuste de la tâche. Cela suggère que les méthodes actuelles de fine-tuning et d'évaluation ne favorisent pas toujours l'apprentissage de représentations robustes et généralisables.
<br />
 
Sur des tâches où le modèle de base atteint un taux de succès proche de zéro, les algorithmes RL on-policy purs (par exemple PPO, GRPO) peinent à produire un quelconque signal d'apprentissage, puisque la politique n'observe jamais de trajectoire réussie à renforcer. Un nombre croissant de travaux aborde ce goulot d'étranglement lié à l'exploration en amorçant le RL avec un petit nombre de démonstrations via des méthodes off-policy — incluant le RL avec données préalables (RLPD), l'apprentissage par imitation, et des approches d'apprentissage de politique résiduelle comme ExpoFT, qui entraîne une politique de correction légère par-dessus un VLA figé en utilisant une fonction Q apprise plutôt que des gradients de politique. Ces méthodes évitent le besoin d'une log-probabilité valide sur les actions, les rendant applicables même aux VLA qui prédisent des actions continues via flow matching, où les vraisemblances exactes sont intractables.

### Objectifs de recherche

<!-- > Présentez les objectifs du projet.  
> Les objectifs peuvent viser à comprendre, comparer, tester, reproduire, améliorer ou explorer une approche.  
> Assurez-vous qu’ils sont réalistes pour un trimestre. -->

1. **Réfuter** l'hypothèse selon laquelle le RL ne fait que redistribuer la masse de probabilité du modèle pré-entraîné, en testant sur des tâches aussi différentes que possible des données d'entraînement SFT.
2. **Identifier** si l'exploration ou l'optimisation constitue le principal goulot d'étranglement dans le fine-tuning par RL des VLA.
3. **Comparer** différentes architectures (OpenVLA-OFT, π0,...) et algorithmes (PPO, GRPO,...) en termes d'efficacité et de stabilité sur un seul GPU A100.
4. **Contribuer** à la compréhension des conditions minimales requises pour que le RL puisse réellement étendre les capacités d'un VLA au-delà de son entraînement supervisé initial.

### Méthodologie envisagée

<!-- > Expliquez comment vous comptez aborder le projet : revue de littérature, analyse théorique, expérimentation, reproduction de résultats, prototype exploratoire, comparaison d’approches, etc.  
> Précisez les données, outils, modèles, articles ou méthodes qui pourraient être utilisés. -->

Le projet adopte une approche expérimentale comparative. Des expériences de fine-tuning par RL sont menées sur plusieurs architectures de VLA et suites de tâches de manipulation robotique en simulation, en utilisant des frameworks tels que RLinf. Différents algorithmes de RL sont comparés, ainsi que différentes architectures de modèles.
 
Lorsque le RL on-policy pur ne parvient pas à générer de signal d'apprentissage sur des tâches où le taux de succès de base est proche de zéro, la méthodologie part d'une configuration plus supervisée — RL off-policy amorcé avec un petit nombre de démonstrations via l'apprentissage de politique résiduelle (ExpoFT) — puis retire progressivement des composantes (démonstrations, préchauffage SFT) afin de déterminer la supervision minimale requise et de distinguer le goulot d'étranglement lié à l'exploration de celui lié à l'optimisation.
 
Une attention particulière est portée à la sélection des tâches — en priorisant celles où le modèle de base SFT performe mal et dont les données d'entraînement sont précisément documentées, afin d'évaluer rigoureusement si le RL permet une généralisation hors distribution.

### Évaluation des résultats

<!-- > Expliquez comment vous évaluerez ou analyserez les résultats : mesures quantitatives, comparaison qualitative, discussion critique, validation expérimentale, limites observées, etc. -->
Les résultats sont évalués via le taux de succès épisodique (`success_once`), comparé entre le modèle SFT de base et le modèle après fine-tuning RL. 
Les courbes d'apprentissage permettent d'évaluer la stabilité et l'efficacité de l'entraînement. 
Une analyse qualitative des comportements appris (via vidéos d'épisodes) complète l'évaluation quantitative. 
Les limites observées — en termes de stabilité algorithmique, de contraintes computationnelles et de généralisation — 
font partie intégrante de l'analyse et contribuent à la compréhension des défis ouverts dans le domaine.

## Équipe

<!-- > Présentez les membres de l’équipe et le rôle principal de chacun dans le projet. -->
**Josué Mongan** est étudiant au **baccalauréat en informatique à l'Université de Montréal**. Il est le seul membre étudiant 
de l'équipe et est responsable de la conception et de l'exécution des expériences, de la revue de littérature et de la documentation du projet.

## Échéancier

!!! info
    Le suivi complet est disponible dans la page [Suivi de projet](suivi.md).

| Activités                          | Début   |   Fin   | Livrable                              | Statut      |
|------------------------------------|---------|---------|---------------------------------------|-------------|
| Définition du sujet                | 4 mai   | 15 mai  | Proposition de projet                 | ✅ Terminé  |
| Revue de littérature préliminaire  | 4 mai   | 29 mai  | Bibliographie commentée / synthèse    | 🔄 En cours |
| Méthodologie et plan d’analyse     | 18 mai  | 5 juin  | Plan méthodologique                   | 🔄 En cours |
| Expérimentation ou analyse         | 1 juin  | 24 juil.| Résultats préliminaires               | ⏳ À venir  |
| Synthèse et interprétation         | 20 juil.| 7 août  | Discussion des résultats et limites   | ⏳ À venir  |
| Présentation + rapport final       | 7 août  | 14 août | Présentation + rapport de recherche   | ⏳ À venir  |
