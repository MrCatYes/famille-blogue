---
title: "Jalon 2 : couper la journée en deux"
date: 2026-11-03
description: "Une décision d'architecture, une faille trouvée en écrivant le schéma, et ce que la base de données dit de la garde partagée."
draft: true
---

Au jalon 1, j'avais laissé deux questions ouvertes. Faut-il des heures dans les exceptions de garde, pour le « souper avec Papa jusqu'à 18h30 » ? Et faut-il généraliser le motif d'alternance pour couvrir un 2-2-3 ? En rédigeant les décisions d'architecture de ce jalon, j'ai constaté que ces deux questions étaient en fait une seule, et que la réponse changeait la forme de la base de données.

Les trois décisions sont documentées dans le dépôt, au format ADR : [PostgreSQL et RLS](https://github.com/MrCatYes/famille/blob/main/docs/adr/0001-postgresql-supabase-et-rls.md), [la garde en demi-journées](https://github.com/MrCatYes/famille/blob/main/docs/adr/0002-modele-de-garde-par-demi-journees.md) et [Next.js 16](https://github.com/MrCatYes/famille/blob/main/docs/adr/0003-nextjs-16-et-organisation-du-code.md). Le [schéma de données commenté](https://github.com/MrCatYes/famille/blob/main/docs/schema.md) décrit chaque table. Ce billet revient sur la décision qui m'a demandé le plus de réflexion, sur l'obstacle le plus sérieux du jalon, et sur ce que le schéma m'a appris.

## La décision : la demi-journée comme unité de garde

### Le contexte

Au jalon 1, le motif de garde était une alternance simple : un parent pendant N jours, puis l'autre. Ça couvre la semaine sur deux, et c'est à peu près tout. En regardant comment la garde se passe vraiment chez nous et chez d'autres parents, trois choses ne rentraient pas dans ce modèle.

- **Le 2-2-3.** Deux jours chez l'un, deux jours chez l'autre, trois jours chez le premier, puis on inverse la semaine suivante. Les blocs n'ont pas la même longueur : aucun « N jours » ne peut le décrire.
- **Le jour du transfert.** Notre fille change de maison à la sortie de l'école. Ce jour-là, elle est le matin avec un parent et le soir avec l'autre. Avec un parent par jour, ce jour-là est forcément faux pour l'un des deux, et c'est justement le jour où on se demande « qui va la chercher ? ».
- **Les exceptions partielles.** Le souper jusqu'à 18h30 n'est pas un changement de garde pour la journée : elle dort quand même chez Maman.

Le suivi du mois, ajouté en cours de route, ajoutait une contrainte : il doit compter les nuits passées chez chaque parent. Il faut donc savoir qui a l'enfant le soir, pas seulement « dans la journée ».

### La décision

L'unité de garde devient la **demi-journée** : le matin, et l'après-midi, qui inclut la soirée et la nuit. Le parent de l'après-midi est celui chez qui notre fille dort.

Un motif est un tableau de demi-journées qui se répète, d'une à cinq semaines, avec une lettre par case : `a` ou `b`. Deux colonnes disent qui est `a` et qui est `b`. Un 2-2-3 sur deux semaines, avec transfert à l'école, s'écrit ainsi :

```
           lun  mar  mer  jeu  ven  sam  dim
semaine 1  b a  a a  a b  b b  b a  a a  a a
semaine 2  a b  b b  b a  a a  a b  b b  b b
```

Le 2-2-3 n'est qu'un exemple. Le même tableau décrit la semaine sur deux, le 2-2-5-5, ou un horaire propre à une famille. Les modèles courants servent seulement à pré-remplir le tableau, que le parent peut ensuite modifier case par case.

Trois besoins sont apparus en discutant de ce modèle, et ils l'ont rendu plus solide :

- **Changer de motif sans réécrire le passé.** Quand l'entente change, on crée un nouveau motif à partir d'une date. L'ancien reste en place pour les jours d'avant, et le calendrier de l'an dernier reste celui qu'on a vécu.
- **Inverser en un clic.** Si les semaines de Papa deviennent celles de Maman, on n'a pas à tout recomposer : le nouveau motif échange `a` et `b`. Pour une inversion en milieu de cycle, un mercredi par exemple, le tableau est décalé pour rester dans la même phase. Sinon, le cycle repartirait comme si ce mercredi était un lundi de la semaine 1.
- **Des exceptions sur une période.** Si je pars trois semaines avec notre fille, c'est une seule exception, du matin du premier jour à l'après-midi du dernier. Le motif n'est pas touché : il reprend le lendemain exactement là où il en serait arrivé.

### Les alternatives rejetées

- **Garder l'alternance de N jours, et passer par les exceptions pour le reste.** C'était la solution la plus simple, et celle que j'avais en tête au jalon 1. Mais un 2-2-3 aurait demandé des dizaines d'exceptions par mois. Une exception doit rester exceptionnelle, sinon le calendrier n'est plus lisible et on ne sait plus ce qui est l'entente et ce qui est un imprévu.
- **Une heure de transfert précise.** C'est l'alternative que j'ai le plus hésité à rejeter, parce qu'elle est plus fidèle à la réalité : le transfert se fait à 15 h 30, pas « l'après-midi ». Mais la question que les parents se posent, c'est « qui va la chercher » et « où elle dort », pas l'heure exacte. Les heures auraient compliqué le calcul, l'affichage et le décompte des nuits pour une précision que personne ne demande. Le détail (« jusqu'à 18h30 ») reste dans la note de l'exception.
- **Une liste de blocs**, comme `[(a, 2), (b, 2), (a, 3)]`. C'est plus proche de la façon dont on décrit un 2-2-3 à l'oral, mais un transfert à midi oblige à couper un bloc en deux, et l'inversion de la deuxième semaine devient un cas particulier. Le tableau de demi-journées est plus long, mais il n'a aucun cas particulier.
- **Une règle de récurrence iCalendar** (RRULE), le format des événements récurrents dans les calendriers. Il est fait pour des événements qui se répètent, pas pour une couverture où chaque instant appartient à exactement un parent. Il aurait fallu vérifier à part qu'il n'y a ni trou ni chevauchement.
- **Enregistrer chaque jour dans la base.** Le calendrier serait trivial à lire, mais il faudrait générer des lignes pour une durée arbitraire et tout régénérer à chaque changement de motif.

Le coût de ce choix est réel : une migration de la base, des tests du jalon 1 à réécrire, et une cellule de calendrier en deux couleurs les jours de transfert, qui doit rester lisible sur téléphone.

## L'obstacle : n'importe qui pouvait entrer dans une famille

### Le problème

En rédigeant l'ADR sur les règles d'accès, je devais expliquer pourquoi les données d'une famille sont protégées. En relisant la migration du jalon 1 pour l'écrire, je suis tombé sur la fonction `join_family()`, qui permet de rejoindre une famille avec son code d'invitation.

Cette fonction est déclarée `security definer` : elle s'exécute avec les droits de son créateur, et contourne donc les règles RLS. C'est nécessaire, parce qu'une personne qui n'a pas encore de famille n'a le droit de rien lire. Mais la seule vérification qu'elle faisait, c'était que le code existe. Or ce code était permanent, réutilisable, et sans limite de membres. Quiconque le connaissait (un texto transféré, une capture d'écran, un ancien conjoint) pouvait rejoindre la famille, même avec deux parents déjà inscrits, et voir aussitôt le calendrier, les documents médicaux et le jugement de garde.

Le plus troublant, c'est que les règles RLS étaient correctes. La faille était dans la porte d'entrée, une fonction écrite justement pour contourner ces règles.

### La solution

Le code permanent disparaît, et il est remplacé par une table `invitations` :

- un parent invite quelqu'un en saisissant son **adresse courriel** et son rôle (parent ou beau-parent), et le code est envoyé à cette adresse ;
- le code ne fonctionne que pour un compte connecté **avec cette adresse**, confirmée ;
- il sert **une seule fois** et **expire au bout d'une heure** ;
- il est stocké **haché**, jamais en clair ;
- en cas d'échec, le message est toujours le même, qu'il soit faux, expiré ou déjà utilisé, pour ne rien apprendre à quelqu'un qui essaie des codes au hasard ;
- une famille compte **au plus deux parents et deux beaux-parents**.

La dernière règle a posé un problème que je n'avais pas vu venir. Une contrainte `check` ne voit qu'une ligne à la fois : elle ne peut pas compter les parents d'une famille. Il faut donc compter dans la fonction. Mais si deux invitations sont acceptées à la même seconde, chacune peut compter « un seul parent » et laisser entrer la personne, et la famille en a trois. La fonction verrouille donc la ligne de la famille avant de compter.

### Un deuxième trou, trouvé de la même façon

En ajoutant la colonne `role` au profil, je me suis rendu compte qu'une règle RLS protège des **lignes**, pas des **colonnes**. La règle du jalon 1, « chacun modifie son propre profil », aurait laissé un beau-parent écrire `role = 'parent'` dans sa propre ligne et obtenir ainsi le droit de tout modifier. L'interface n'offre aucun bouton pour ça, mais n'importe qui peut appeler l'API de Supabase directement avec sa session. La correction tient en deux lignes : un privilège de colonne, qui ne laisse modifier que le nom, la couleur et la photo.

Le même raisonnement m'a fait ajouter des clés étrangères composées : sans elles, un motif de garde aurait pu désigner le parent d'une autre famille, parce que la règle RLS vérifie la famille de la ligne écrite, pas celle des lignes vers lesquelles elle pointe.

### Comment je l'ai vérifié

Au jalon 1, je ne savais pas comment tester les règles d'accès sans exposer une vraie base. J'ai exécuté tout le SQL du schéma dans PGlite, une version de PostgreSQL qui tourne dans Node.js, avec une vingtaine de cas : un beau-parent qui écrit un événement ou se donne le rôle de parent, deux exceptions qui se chevauchent d'une demi-journée, un motif qui désigne quelqu'un d'une autre famille… Tous sont refusés par la base, et les cas valides passent. Ces tests deviendront ceux de la migration.

## Ce que le schéma révèle du domaine

Écrire le schéma table par table m'a obligé à trancher des questions que l'interface laisse dans le flou.

**La garde est une couverture, pas une suite d'événements.** Chaque demi-journée appartient à exactement un parent : il n'y a ni trou ni double garde. C'est pour ça que la base ne stocke jamais le calendrier : elle stocke les règles (les motifs) et les décisions (les exceptions). Les événements sont l'inverse : des faits ponctuels, qui peuvent se chevaucher ou laisser des jours vides. Deux notions que j'appelais toutes les deux « le calendrier » ont en fait des formes opposées.

**Le passé ne se réécrit pas.** Les motifs sont datés et jamais modifiés après coup, et chaque exception, dépense ou modification de la fiche garde qui l'a faite. Dans une garde partagée, l'historique sert : on le consulte en réunion de coparentalité, et il pourrait un jour compter devant un médiateur. « Qui a changé quoi » compte autant que le changement.

**Les deux parents sont égaux.** Il n'y a pas de propriétaire de la famille. Celui qui l'a créée n'a aucun pouvoir de plus, et les motifs parlent de `a` et de `b`, pas d'un parent principal. Donner à un parent le droit de gérer la famille de l'autre recréerait dans l'application le déséquilibre qu'on essaie justement d'éviter. La seule asymétrie est volontaire : chaque parent répond du beau-parent qu'il a invité, et lui seul peut le retirer.

**Une personne, une famille.** C'est ce qui garde les règles d'accès simples, mais c'est une vraie limite pour les familles recomposées : un beau-parent chez nous est souvent parent ailleurs, et il ne peut pas être membre des deux. De la même façon, la garde est rattachée à la famille et non à l'enfant : deux enfants avec des horaires différents ne sont pas représentables. Ce sont des limites que j'accepte parce qu'elles ne touchent pas notre famille, mais je sais exactement quelles tables changeraient.

**La base ne sait pas négocier.** Aujourd'hui, un parent ajoute une exception seul, et l'autre la voit apparaître. Dans la vraie vie, un échange de fin de semaine se demande, puis s'accepte ou se refuse. Un statut sur les exceptions suffirait techniquement, mais il transformerait un calendrier en outil de négociation. Je préfère voir à l'usage si on en a besoin.

## Ce qui reste incertain

- **La Loi 25 et l'historique.** La loi donne à chacun le droit de faire supprimer ses données, mais le schéma empêche aujourd'hui de supprimer un parent qui figure dans un motif de garde. Il faut décider ce qui reste de l'historique quand un parent quitte l'application.
- **La suppression d'une exception** efface toute trace qu'elle a existé, ce qui contredit un peu le principe « le passé ne se réécrit pas ». Une suppression douce serait peut-être plus juste.
- **La migration n'est pas faite.** Tout ce billet décrit le schéma visé. La migration, les nouveaux tests et la mise à jour du calcul de garde sont la prochaine étape ([#24](https://github.com/MrCatYes/famille/issues/24)).

## L'IA à ce jalon

J'ai continué à travailler avec Claude Code, mais autrement qu'au jalon 1. Les décisions de produit sont venues de moi et de notre expérience de la garde : le motif modifiable plutôt que fixe, l'inversion en un clic, les absences de plusieurs semaines, les beaux-parents en lecture seule, les invitations par courriel. Je les ai souvent décrites à partir de maquettes de l'écran. L'assistant a rédigé les ADR et le schéma à partir de ces décisions, et je les ai relus et corrigés.

Les deux failles décrites plus haut sont apparues pendant cette rédaction : c'est en devant expliquer pourquoi les données sont protégées qu'on voit où elles ne le sont pas. Et cette fois, le travail est arrivé en petits commits, un par décision ou par révision, sur des branches avec une PR chacune.
