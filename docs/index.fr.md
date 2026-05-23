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

    **Collaborateur(s):** [Florence Cloutier](https://mila.quebec/fr/annuaire/florence-cloutier)

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
Appliqué aux VLAs, le RL pourrait permettre d'améliorer les capacités de raisonnement et de planification de ces 
modèles sur des tâches nécessitant une précision ou une séquentialité élevées. Cependant, l'optimisation RL sur 
des modèles de grande taille reste un défi ouvert, notamment en termes de stabilité algorithmique, 
d'efficacité computationnelle et de conception du signal de récompense.

### Problématique

<!-- > Décrivez la problématique, la question de recherche ou l’hypothèse que votre projet cherche à explorer.  
> La question doit être suffisamment claire pour orienter la revue de littérature, la méthodologie et l’analyse. -->

- **Dans quelles conditions le fine-tuning par RL améliore-t-il les capacités des VLAs au-delà de ce que le SFT permet d'atteindre ?** 
- **Quels algorithmes, architectures et types de tâches sont les plus propices à ce type d'amélioration ?** 
- **Et comment concevoir des tâches et des signaux de récompense qui forcent le modèle à développer de véritables capacités de raisonnement plutôt que de simplement mémoriser des comportements ?**

### État de l’art préliminaire

<!-- > Présentez brièvement les approches, travaux ou concepts importants déjà identifiés.  
> L’objectif n’est pas encore de faire une revue complète, mais de situer votre projet par rapport aux travaux existants. -->

Le fine-tuning par apprentissage par renforcement des grands modèles de langage (LLMs) a connu des succès notables 
avec des algorithmes comme PPO et GRPO, notamment dans des travaux tels que DeepSeek-R1 et InstructGPT. 
L’extension de ces approches aux modèles vision-langage-action (VLAs) est plus récente et soulève des défis 
supplémentaires liés à la nature multimodale et à la dimension action de ces modèles. 
<br />

Des frameworks récents comme [RLinf](https://github.com/RLinf/RLinf) proposent une infrastructure unifiée permettant 
d’appliquer le RL à différents modèles vision-langage-action, tout en supportant plusieurs architectures (OpenVLA, π0, GR00T) 
et environnements (LIBERO, ManiSkill, RoboCasa). 
<br />

Les premiers résultats montrent que le RL peut améliorer significativement les performances obtenues par supervised fine-tuning (SFT), 
notamment sur les tâches longues et multi-étapes, mais les conditions optimales de cet apprentissage restent encore mal comprises. 
Des travaux récents ([LIBERO-OOD, ICLR 2026](https://openreview.net/forum?id=jtrhwfgseW) ; [LIBERO-Plus, arXiv 2510.13626](https://arxiv.org/abs/2510.13626)) montrent également que les modèles entraînés par SFT souffrent d’un problème de généralisation hors distribution, 
mémorisant souvent les configurations vues à l’entraînement plutôt que de développer une compréhension robuste des tâches. 
Cette limitation suggère que les méthodes actuelles de fine-tuning et d’évaluation ne favorisent pas toujours l’acquisition 
de représentations robustes et généralisables.

### Objectifs de recherche

<!-- > Présentez les objectifs du projet.  
> Les objectifs peuvent viser à comprendre, comparer, tester, reproduire, améliorer ou explorer une approche.  
> Assurez-vous qu’ils sont réalistes pour un trimestre. -->

1. **Explorer** les conditions sous lesquelles le RL améliore les performances des VLAs par rapport au SFT, en termes d'algorithmes, d'architectures et de types de tâches.
2. **Identifier** les types de tâches où le SFT seul est insuffisant et où le RL apporte une contribution claire — notamment les tâches nécessitant précision, planification séquentielle ou raisonnement.
3. **Étudier** l'impact de la conception du signal de récompense sur l'efficacité du fine-tuning RL.
4. **Contribuer** à la compréhension des défis d'optimisation propres au RL sur VLAs, en documentant les pratiques et configurations qui fonctionnent dans un contexte de ressources limitées.

### Méthodologie envisagée

<!-- > Expliquez comment vous comptez aborder le projet : revue de littérature, analyse théorique, expérimentation, reproduction de résultats, prototype exploratoire, comparaison d’approches, etc.  
> Précisez les données, outils, modèles, articles ou méthodes qui pourraient être utilisés. -->

Le projet adopte une approche expérimentale comparative. Des expériences de fine-tuning RL sont conduites sur 
plusieurs architectures VLA et plusieurs suites de tâches de manipulation robotique en simulation, 
en utilisant le framework RLinf. Les algorithmes d'apprentissage par renforcement (PPO, GRPO, etc.) sont comparés, 
ainsi que différents types d'architectures (OpenVLA-OFT, π0). 

Une attention particulière est portée à la sélection des tâches — en privilégiant des tâches où le modèle SFT de base est faible, 
afin de mieux isoler la contribution du RL. La conception du signal de récompense et son impact sur la qualité de l'apprentissage 
font également partie des axes à explorer.

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

| Activités                      | Début   |   Fin   | Livrable                            | Statut      |
|--------------------------------|---------|---------|-------------------------------------|-------------|
| Définition du sujet            | 4 mai   | 15 mai  | Proposition de projet               | ✅ Terminé  |
| Revue de littérature préliminaire  | 4 mai   | 29 mai  | Bibliographie commentée / synthèse    | 🔄 En cours |
| Méthodologie et plan d’analyse     | 18 mai  | 5 juin  | Plan méthodologique                   | 🔄 En cours  |
| Expérimentation ou analyse         | 1 juin  | 24 juil.| Résultats préliminaires               | ⏳ À venir  |
| Synthèse et interprétation         | 20 juil.| 7 août  | Discussion des résultats et limites   | ⏳ À venir  |
| Présentation + rapport final       | 7 août  | 14 août | Présentation + rapport de recherche   | ⏳ À venir  |
