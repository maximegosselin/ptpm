> Imagine une application web où plusieurs personnes peuvent collaborer en même temps sur la conception d'un diagramme
> basique. L'application permet de tracer des formes, les déplacer, les relier entre elles, écrire du texte. Quels sont
> les problèmes potentiels et donne moi des pistes de solutions pour les régler.

C'est un problème classique de collaboration temps-réel avec plusieurs dimensions intéressantes. Voici les principaux
défis et des pistes de solutions.

---

## Cohérence de l'état partagé

**Le problème central:** deux utilisateurs modifient le même élément en même temps. Qui gagne?

**Pistes:**

- **Operational Transformation (OT)** - l'approche historique (Google Docs). Complexe à implémenter correctement.
- **CRDTs (Conflict-free Replicated Data Types)** - structures de données qui fusionnent automatiquement sans conflits.
  Mieux adapté aux structures arborescentes comme un diagramme. Bibliothèques: Yjs, Automerge.
- **Verrouillage optimiste** - chaque opération porte un numéro de version. Le serveur rejette les opérations en conflit
  et le client rebase.
- **Ownership temporaire** - dès qu'un utilisateur sélectionne une forme, elle est "lockée" pour les autres (approche la
  plus simple, mais bloquante).

---

## Latence et perception de fluidité

**Le problème:** les opérations réseau introduisent un délai visible.

**Pistes:**

- **Optimistic UI** - appliquer l'opération localement immédiatement, puis réconcilier avec la réponse serveur.
- **Prédiction locale** - afficher le mouvement en cours sans attendre la confirmation.
- **Throttling/debouncing** - ne pas envoyer chaque pixel de déplacement, mais par intervalles (ex: 16ms pour 60fps).

---

## Modèle de données et granularité des opérations

**Le problème:** comment représenter les changements de façon à ce qu'ils soient applicables dans n'importe quel ordre?

**Pistes:**

- **Event log immuable** - stocker des opérations (`MoveShape`, `ResizeShape`, `AddConnector`) plutôt que l'état final.
  Chaque opération est idempotente et rejouable. (Tu connais bien ça.)
- **Granularité fine** - séparer `position`, `style`, `texte` comme propriétés indépendantes pour réduire les conflits.
- **Vector clocks / Lamport timestamps** - ordonner les événements sans horloge centralisée.

---

## Transport réseau

**Le problème:** quel mécanisme utiliser pour la communication temps-réel?

**Pistes:**

- **WebSockets** - connexion persistante bidirectionnelle. Standard pour ce type d'app.
- **WebRTC (data channels)** - peer-to-peer, réduit la latence et la charge serveur, mais complexifie la gestion des
  connexions.
- **Server-Sent Events** - unidirectionnel (serveur vers client), suffisant si les writes passent par HTTP classique.

---

## Reconnexion et état hors-ligne

**Le problème:** un utilisateur perd sa connexion pendant 30 secondes.

**Pistes:**

- **Delta sync** - à la reconnexion, le client envoie son dernier `sequence_id` connu et reçoit seulement les opérations
  manquantes.
- **Snapshot + log** - garder un snapshot périodique + le log d'opérations depuis le snapshot pour reconstituer l'état
  rapidement.
- **Queue locale** - les opérations faites hors-ligne sont mises en queue et soumises à la reconnexion (avec risque de
  conflit à gérer).

---

## Présence et conscience des autres

**Le problème:** savoir qui est là et ce qu'ils font, sans que ça soit intrusif.

**Pistes:**

- **Curseurs des autres utilisateurs** - afficher la position en temps réel avec un délai de tolérance.
- **Awareness channels** - séparer le canal "présence" (léger, ephémère) du canal "données" (persisté).
- **Locks visuels** - colorier les formes sélectionnées par quelqu'un d'autre.

---

## Scalabilité serveur

**Le problème:** plusieurs salles actives en même temps, sur plusieurs instances serveur.

**Pistes:**

- **Rooms isolées** - chaque diagramme est une room indépendante avec son propre état.
- **Pub/sub partagé** - Redis pub/sub ou similaire pour synchroniser les instances serveur.
- **Sticky sessions** - tous les clients d'une room vont au même serveur (plus simple, moins résilient).

---

## Résumé de la complexité

| Approche             | Complexité  | Robustesse |
|----------------------|-------------|------------|
| Lock optimiste       | Faible      | Faible     |
| Ownership temporaire | Faible      | Moyenne    |
| OT                   | Très élevée | Élevée     |
| CRDT (Yjs)           | Moyenne     | Élevée     |
| Event log + rebase   | Moyenne     | Élevée     |

Pour un premier prototype, **ownership temporaire + WebSockets + event log** est un bon point de départ. Pour une app de
production sérieuse, **Yjs avec des providers WebSocket** est probablement le chemin le plus court vers quelque chose de
robuste.

Tu vises quel niveau de complexité / cas d'usage?

> Maintenant lis cet article https://maximegosselin.com/posts/pull-the-plug-modeling/ et reconsidère la question que je
> t'ai posée. Qu'est-ce qui change dans ta façon de penser?

Bonne question. En lisant l'article, je réalise que ma réponse précédente était entièrement dans le registre technique.
J'ai sauté directement aux CRDTs, WebSockets, et vector clocks sans jamais me demander ce que le domaine essaie vraiment
de faire.

Voici ce que PTPM change dans ma façon de penser.

---

## Ce que je n'ai pas fait

J'ai traité le problème comme un problème d'ingénierie distribué. Mais la vraie question aurait dû être: **comment des
gens collaborent-ils sur un diagramme dans un monde sans électricité?**

La réponse est évidente: ils ne le font pas simultanément. Ils se passent la feuille.

---

## Ce que le monde papier révèle

**Un document appartient à quelqu'un.**
Une feuille est sur la table de quelqu'un. Pour que quelqu'un d'autre y touche, il y a une remise explicite. Ce n'est
pas un détail technique, c'est la nature du travail collaboratif sur un artefact physique.

Ce que ça implique pour le domaine: l'**ownership** n'est pas un mécanisme de verrouillage à implémenter. C'est la
réalité du domaine. Le "lock" que j'ai présenté comme la solution la plus simple est en fait la solution la plus fidèle
au domaine.

**Les corrections laissent une trace.**
Personne n'efface. On rature, on ajoute, on corrige. L'historique est visible sur le document lui-même.

Ce que ça implique: un event log immuable n'est pas juste une bonne pratique technique. C'est le comportement naturel du
domaine. Modifier une forme "en place" sans laisser de trace est une abstraction numérique qui s'éloigne du domaine
réel.

**Les gens travaillent de façon asynchrone.**
Dans un monde papier, deux personnes ne modifient pas la même feuille en même temps. La vraie question n'est pas "
comment gérer les conflits simultanés" mais "est-ce que la simultanéité est vraiment ce que le domaine requiert?"

Ce que ça implique: j'ai présumé que la collaboration en temps réel était la contrainte centrale. Peut-être que le vrai
besoin est de pouvoir voir ce que les autres font, pas nécessairement d'agir au même instant sur le même objet.

---

## Ce que ça change concrètement

Ma première réponse posait le problème comme: *"comment réconcilier des états concurrents?"*

Après PTPM, la question devient: *"comment transférer la responsabilité d'un artefact entre des personnes, de façon
explicite et traçable?"*

Ce sont deux problèmes très différents. Le premier mène vers OT, CRDTs, et une complexité considérable. Le second mène
vers un modèle de handoff explicite, des opérations nommées selon le vocabulaire métier, et un historique visible, ce
qui est beaucoup plus simple et probablement plus juste.

---

## Ce qui survit de ma première réponse

La latence et le feedback visuel (curseurs, présence) restent pertinents, parce qu'ils n'existent pas dans un monde
papier mais qu'ils sont une valeur ajoutée réelle du numérique. C'est là où on peut dépasser le domaine physique de
façon intentionnelle, pas par réflexe technique.

---

En résumé: PTPM m'aurait évité de me noyer dans la complexité accidentelle avant même d'avoir compris la complexité
essentielle.

> Et en appliquant PTPM, comment tu modéliserais plusieurs personnes qui collaborent ensemble sur un même diagramme?

Faisons l'exercice sérieusement.

---

## Pull the plug

On retire l'électricité. On a une grande table, du papier, des crayons, et plusieurs personnes qui veulent concevoir un
diagramme ensemble.

---

## Ce qu'on observe

**La feuille est au centre de la table.**
Tout le monde peut la voir. Mais une seule personne peut écrire dessus à la fois, sinon c'est le chaos. Naturellement,
les gens s'organisent sans qu'on le leur demande.

**Quelqu'un prend le crayon.**
C'est un acte explicite et visible. Les autres voient que la feuille est "prise". Ils attendent, ou ils parlent, ou ils
pointent du doigt pour suggérer.

**Le crayon se passe.**
"Tiens, c'est à toi." La remise est intentionnelle. Personne ne l'arrache. Et quand quelqu'un repose le crayon sans le
passer, les autres savent que c'est disponible.

**Les erreurs se raturent.**
On ne déchire pas la feuille pour recommencer. On barre, on annote, on ajoute une flèche. L'historique des décisions
reste visible sur le papier lui-même.

**Les gens commentent à voix haute.**
"Mets la boîte plus à gauche." "Non, relie-la à celle-ci plutôt." Le diagramme est un artefact partagé, mais la
conversation autour est aussi réelle que le dessin.

**Personne ne travaille en silence dans son coin.**
Si quelqu'un veut proposer une variante, il prend une autre feuille et la pose à côté. "Regarde, est-ce que ça serait
mieux comme ça?"

---

## Ce que ça révèle comme événements du domaine

- Le crayon a été pris par quelqu'un
- Une forme a été tracée
- Une forme a été déplacée (raturée et redessinée)
- Une connexion a été tracée entre deux formes
- Du texte a été ajouté
- Une correction a été faite (avec trace visible)
- Le crayon a été passé à quelqu'un d'autre
- Le crayon a été reposé (disponible)
- Une variante a été proposée sur une feuille séparée
- La variante a été adoptée (on remplace la feuille principale) ou rejetée

---

## Ce que ça révèle comme règles du domaine

1. **Une seule personne tient le crayon à la fois.** Ce n'est pas une contrainte technique, c'est la nature de l'
   artefact.
2. **Tout le monde voit la feuille en tout temps.** La visibilité est passive et universelle.
3. **Les suggestions viennent de la conversation, pas du crayon.** On peut influencer le dessin sans tenir le crayon.
4. **Les corrections ne détruisent pas l'historique.** On ajoute, on rature, mais la trace reste.
5. **Une variante est un artefact séparé**, pas une modification de l'original.
6. **Le transfert du crayon est explicite et volontaire.** Il ne se vole pas.

---

## Ce que ça suggère comme modèle

Le domaine n'est pas "édition collaborative temps réel". Le domaine est:

> **Un groupe de personnes qui construisent un artefact partagé, en se passant la responsabilité de l'écriture de façon
explicite, sous le regard de tous.**

Ça change tout. Le modèle central n'est pas un canvas concurrent. C'est un **artefact avec un détenteur actif** entouré
d'**observateurs qui peuvent suggérer**.

Les deux rôles distincts qui émergent naturellement:

- **Le détenteur du crayon** - il agit sur le diagramme
- **Les observateurs** - ils voient en temps réel, ils suggèrent, ils discutent

Et la mécanique centrale n'est pas la résolution de conflits. C'est le **transfert de responsabilité**.

---

C'est là que j'arrêterais avant de toucher à la technique. Est-ce que ce modèle correspond à ce que tu imagines comme
expérience utilisateur?

> Ça nous amène naturellement à se poser la question suivante: pour apporter une modification au diagramme, avons-nous
> besoin de savoir ce à quoi les autres pensent à ce moment?

Non. Et c'est une question piège intéressante.

Dans le monde papier, quand tu tiens le crayon, tu n'as pas accès aux pensées des autres. Tu vois leurs visages, tu
entends leurs voix s'ils parlent. Mais leur intention non exprimée t'est invisible, et ça ne t'empêche pas de dessiner.

Ce qui compte, c'est l'**état de la feuille**, pas l'état mental des autres participants.

---

## Ce que ça révèle

La question des curseurs en temps réel, que j'avais présentée comme une feature de "présence", est en réalité une
tentative de rendre visible quelque chose qui n'existe pas dans le domaine: l'intention non exprimée d'autrui.

Dans le monde papier, tu n'as pas besoin de savoir où les yeux des autres se posent pour faire ton travail. Tu as besoin
de savoir:

1. **Est-ce que le crayon est disponible?**
2. **À quoi ressemble la feuille maintenant?**

C'est tout. Le reste est du bruit.

---

## L'implication concrète

Les curseurs des autres utilisateurs, les indicateurs "X est en train de modifier...", les zones surlignées en temps
réel... ce sont des solutions à un problème que le domaine ne pose pas.

Le vrai besoin de visibilité est beaucoup plus simple: **tout le monde voit la même feuille, mise à jour quand quelque
chose change.**

Ce qui ramène la complexité technique à un niveau beaucoup plus raisonnable. Pas de synchronisation d'état éphémère à la
milliseconde. Juste une feuille partagée et un crayon qui appartient à quelqu'un.


> L'exemple de quelques personnes autour d'une table a du sens. Mais qu'en est-il si on parlait plutôt de centaines
> de personnes? Qu'est-ce qui changerait?


Bonne question. Revenons dans le monde papier, mais avec des centaines de personnes.

---

## Ce qu'on observe

Cent personnes autour d'une seule feuille, c'est physiquement absurde. Personne ne peut voir, personne ne peut
approcher. Le modèle de la table s'effondre.

Ce qui se passe naturellement dans un monde papier quand un groupe devient grand:

**On affiche la feuille.** On l'épingle au mur ou on la projette avec un rétroprojecteur. Elle devient une référence
visible par tous, mais que personne ne touche directement.

**Un secrétaire ou un animateur tient le crayon.** Il ne décide pas. Il transcrit ce que le groupe décide
collectivement. Le crayon n'appartient plus à un participant ordinaire, il appartient à un rôle.

**La parole remplace le crayon.** Tu ne te lèves pas pour modifier le diagramme. Tu lèves la main, tu proposes, le
groupe réagit, l'animateur dessine si la proposition est retenue.

**Les sous-groupes travaillent en parallèle.** On divise la salle. Chaque groupe travaille sur sa propre feuille, sur
une portion du problème. Ensuite on met les feuilles ensemble.

---

## Ce que ça révèle comme changement fondamental

À petite échelle, le modèle est: **un détenteur actif, des observateurs.**

À grande échelle, le modèle change complètement: **un artefact de référence affiché, une gouvernance collective pour le
modifier, et une décomposition en sous-artefacts.**

Ce ne sont pas deux variantes du même modèle. Ce sont deux domaines différents.

---

## Ce que ça suggère

Avant de concevoir quoi que ce soit, la vraie question à poser est: **combien de personnes collaborent activement sur le
même diagramme en même temps, dans un cas réel?**

Parce que si la réponse honnête est "3 à 5", le modèle du crayon qui se passe tient parfaitement. Si la réponse est "des
centaines", on n'est probablement plus dans de la collaboration active, on est dans de la **consultation** ou de la *
*diffusion**, ce qui est un domaine très différent avec des besoins très différents.

PTPM nous évite de construire une solution à grande échelle pour un problème qui se pose à petite échelle.