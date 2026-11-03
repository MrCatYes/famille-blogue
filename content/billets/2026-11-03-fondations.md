---
title: "Jalon 1 : les fondations"
date: 2026-11-03
description: "Pour qui, pourquoi, avec quoi, et ce qui reste flou."
draft: false
---

## Qui sont mes utilisateurs et quel problème je résous

Les utilisateurs, ce sont deux parents séparés qui partagent la garde de leur fille. Pour commencer, ce sera ma propre famille : l'application sert d'abord à un vrai besoin, chez nous.

Aujourd'hui, l'information sur la garde est éparpillée. L'alternance régulière, on la connaît, mais dès qu'une exception arrive, elle se règle par texto. Sur notre calendrier actuel, une semaine chez Maman contient une ligne « Souper avec Papa jusqu'à 18h30 ». C'est exactement le genre de détail qui se perd : ce n'est pas un changement de garde, mais il change l'organisation de la soirée pour les deux parents. Ajoutez à ça les réunions d'école, les rendez-vous médicaux et les journées pédagogiques, notés dans deux calendriers différents, puis les documents (carte d'assurance maladie, carnet de vaccination, bulletins) qui sont toujours chez l'autre parent au moment où on en a besoin.

L'application vise donc trois choses :

1. **Savoir d'un coup d'œil chez qui est notre fille**, n'importe quel jour. Le calendrier affiche une bande bleue pour Papa et rose pour Maman, calculée à partir d'un motif d'alternance et corrigée par les exceptions ponctuelles.
2. **Partager les événements** classés par catégorie (école, médical, anniversaire, jour férié, important), visibles par les deux parents dès qu'ils sont ajoutés.
3. **Garder les documents au même endroit**, accessibles aux deux parents et à personne d'autre.

Le découpage en user stories est dans les [issues du dépôt](https://github.com/MrCatYes/famille/issues). Les six stories marquées `priorité: haute` couvrent les comptes, la famille partagée, le calendrier de garde et les événements : c'est le cœur sans lequel l'application ne sert à rien.

## Mes premiers choix techniques

**Next.js avec TypeScript.** Je voulais une seule base de code pour l'interface et le serveur, et une application qui fonctionne bien sur téléphone, parce que c'est là qu'on regarde un calendrier. Next.js permet de rendre les pages côté serveur (le calendrier arrive déjà rempli) et d'avoir des composants interactifs là où c'est utile, comme le formulaire de connexion. TypeScript m'oblige à décrire la forme des données, ce qui compte dès que la même structure (un motif de garde, une exception) circule entre la base de données, le calcul et l'affichage.

Les alternatives considérées :

- **Django avec PostgreSQL.** C'est un choix solide, avec une interface d'administration gratuite. Mais j'aurais eu deux mondes à faire cohabiter, Python côté serveur et JavaScript pour l'interactivité du calendrier, et je voulais approfondir l'écosystème React.
- **Une application mobile native** (React Native ou Flutter). C'est tentant pour les notifications, mais il aurait fallu passer par l'App Store et Google Play : comptes développeur payants, révision de chaque mise à jour, délais de publication. Avec une application web et Supabase comme serveur, je publie quand je veux, et les deux parents y accèdent par un simple lien, sans rien installer. Le cours demande d'ailleurs une application web, et une application web bien faite s'ouvre déjà très bien sur téléphone.

**Supabase pour la base de données, l'authentification et les fichiers.** Supabase, c'est du PostgreSQL hébergé avec l'authentification et le stockage de fichiers intégrés. Le point décisif, c'est la sécurité au niveau des lignes (Row Level Security) : la règle « un parent ne voit que les données de sa famille » est écrite dans la base elle-même, dans [la migration SQL](https://github.com/MrCatYes/famille/blob/main/supabase/migrations/0001_init.sql), et non dans chaque page de l'application. Même si j'oublie un filtre dans une requête, la base refuse de renvoyer les données d'une autre famille.

Les alternatives considérées :

- **Firebase.** Il offre les mêmes services, mais sa base est orientée documents (NoSQL), alors que le cours exige une base relationnelle. Et les relations sont au cœur du domaine : une exception de garde pointe vers un parent, qui appartient à une famille.
- **PostgreSQL auto-hébergé avec une authentification maison.** C'est plus formateur sur certains points, mais écrire soi-même la gestion des mots de passe et des sessions est exactement le genre de chose qu'il vaut mieux ne pas improviser.

En chemin, le modèle de données a déjà changé. Mon idée de départ était deux comptes, Papa et Maman, point. En écrivant les règles d'accès, je me suis rendu compte qu'il manquait une notion : **la famille**. Sans elle, « les deux comptes » n'existe nulle part dans la base, et n'importe quel compte inscrit verrait tout. J'ai donc ajouté une table `families` : un parent crée la famille, l'autre la rejoint avec un code d'invitation, et toutes les autres tables portent un `family_id` sur lequel s'appuient les règles d'accès.

**L'hébergement : Vercel, a priori.** Il est conçu pour Next.js, gratuit à cette échelle, et se déploie à chaque push. Rien n'est encore déployé : c'est attendu pour le jalon 4.

**Tests et intégration continue.** Les tests utilisent Vitest, et une [pipeline GitHub Actions](https://github.com/MrCatYes/famille/blob/main/.github/workflows/ci.yml) lance l'analyse statique, les tests et la compilation à chaque push. Les tests couvrent pour l'instant la partie la plus délicate, le calcul de la garde dans [`lib/custody.ts`](https://github.com/MrCatYes/famille/blob/main/lib/custody.ts). Un de ces tests vérifie le passage à l'heure normale du 1er novembre. Si on calcule l'écart entre deux dates avec des objets `Date` à l'heure locale, une journée peut durer 25 heures, et la division par 24 heures fausse le résultat. C'est pour ça que le calcul manipule les dates en chaînes `AAAA-MM-JJ` converties en UTC.

## Ce qui reste incertain

- **Les exceptions sur une partie de la journée.** Le « souper jusqu'à 18h30 » est pour l'instant une exception sur toute la journée, avec une note. Ce n'est pas tout à fait vrai : notre fille dort quand même chez Maman ce soir-là. Faut-il des heures de début et de fin dans les exceptions ? Ça complique le calcul et l'affichage, et je n'ai pas encore tranché.
- **Les motifs de garde plus complexes.** Le motif actuel alterne entre deux parents selon un cycle de N jours. Ça couvre la semaine sur deux, mais pas un motif 2-2-3, qui n'est pas une simple alternance de blocs égaux. Je ne sais pas encore si je généraliserai le modèle ou si je m'en tiendrai aux exceptions.
- **La Loi 25.** L'application contient des renseignements personnels sur une enfant mineure : dates, lieux, documents médicaux. Je dois déterminer ce que ça implique concrètement : consentement, durée de conservation, et ce qui se passe quand un parent supprime son compte.
- **La dépendance à Supabase.** Si le service change ses conditions ou que le projet dépasse l'offre gratuite, la migration ne sera pas triviale. La base reste du PostgreSQL standard, ce qui me rassure en partie, mais l'authentification et le stockage, eux, ne le sont pas.
- **Comment tester les règles d'accès.** Les tests actuels portent sur la logique en TypeScript. Les règles RLS vivent dans la base, et je ne sais pas encore comment les tester automatiquement dans la pipeline sans exposer une vraie base.
- **Les notifications.** Être averti quand l'autre parent ajoute une exception serait probablement la fonctionnalité la plus utile au quotidien. Par courriel, par SMS ou par notification web ? Je ne sais pas encore.
