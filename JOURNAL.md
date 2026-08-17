# Journal de bord — La Roue de la Fortâche

## Phase 1 — Squelette minimal fonctionnel (2026-08-17)

### Demande

Construire un prototype tenant dans un seul `index.html` (JS vanilla + SVG, sans
framework, sans build, sans backend, sans compte utilisateur), déployable tel
quel sur GitHub Pages. Pour cette phase 1, sans style particulier :

- un champ pour saisir les noms des membres de l'équipe (ajout / suppression) ;
- une roue en SVG avec une part égale par membre ;
- un bouton « Tourner » qui fait tourner la roue et sélectionne un membre au
  hasard (tirage uniforme, la pondération viendra en phase 2) ;
- l'affichage du membre sélectionné.

### Ce qui a été construit

Un unique fichier `index.html` à la racine, contenant :

- Un formulaire d'ajout de membre (`<input>` + bouton « Ajouter ») et une
  liste (`<ul>`) avec un bouton « Supprimer » par membre.
- La liste des membres est persistée dans `localStorage` (clé
  `roue-fortache-members`) pour ne pas la perdre au rechargement — trois noms
  par défaut (Alice, Bob, Charlie) si rien n'est stocké.
- Une roue SVG générée dynamiquement : une part (`<path>`) par membre, calculée
  par calcul trigonométrique classique (coordonnées polaires → cartésiennes),
  avec une couleur par index piochée dans une palette fixe de 12 couleurs
  (répétée si plus de 12 membres) et le nom du membre affiché au centre de sa
  part.
- Un bouton « Tourner » qui : tire un index gagnant au hasard
  (`Math.random()`, tirage uniforme), calcule l'angle de rotation nécessaire
  pour amener le centre de la part gagnante sous le pointeur (en haut), ajoute
  5 tours complets pour l'effet visuel, applique une transition CSS de 4s sur
  `transform: rotate(...)`, puis affiche le résultat dans une zone dédiée une
  fois l'animation terminée.
- Un pointeur (triangle ▼) sous la roue indiquant la part gagnante.

### Choix et compromis

- **Aucun style/CSS pour l'instant** : conformément à la consigne de la phase
  1, l'app est fonctionnelle mais brute (HTML par défaut du navigateur). La
  mise en page, le positionnement du pointeur au-dessus de la roue et
  l'habillage visuel sont laissés aux phases suivantes.
- **Persistance en `localStorage`** : non demandée explicitement, mais ajoutée
  pour éviter de perdre la liste des membres à chaque rechargement pendant les
  tests manuels. Reste un fichier unique, aucun backend impliqué — cohérent
  avec les contraintes.
- **Rotation calculée pour arriver précisément sur le membre tiré** : plutôt
  qu'une rotation purement aléatoire suivie d'une lecture de la part sous le
  pointeur, l'angle final est calculé à partir de l'index gagnant. Le tirage
  reste bien aléatoire (`Math.random()`), seule la rotation visuelle est
  déterministe une fois l'index connu — plus simple à fiabiliser qu'une
  détection géométrique après coup.
- **Cas à un seul membre** : la roue devient un simple cercle plein (un seul
  `<path>` en arc de 360° poserait des problèmes de rendu SVG), avec le nom du
  membre affiché dessus.

### Bugs rencontrés et corrections

- **Rotation non centrée** : par défaut, un élément SVG tourne autour du point
  (0,0) du viewport et non de son centre. Corrigé en ajoutant
  `transform-origin: 50% 50%` en style inline sur le `<svg>` — nécessaire au
  bon fonctionnement de l'animation, donc conservé malgré la consigne « sans
  style » (fonctionnel, pas esthétique).
- Aucun autre bug bloquant : la syntaxe JS a été validée avec Node.js, et un
  test automatisé via Playwright (navigation directe sur le fichier
  `index.html` en local, sans serveur) a confirmé : rendu initial de la roue à
  3 parts, ajout d'un 4ᵉ membre (la roue passe à 4 parts), suppression d'un
  membre (retour à 3 parts), clic sur « Tourner » suivi de l'affichage correct
  du membre sélectionné après l'animation, et absence d'erreur console.

### État à la fin de la phase 1

Prototype fonctionnel dans `index.html`, testé manuellement via un navigateur
headless (Playwright). Prêt pour la phase 2 (pondération du tirage).

## Phase 2 — Roue « équitable » : poids, tâches et journal (2026-08-17)

### Demande

Rendre la roue équitable :

- chaque membre a un poids, et la taille de sa part sur la roue est
  proportionnelle à ce poids ;
- ajouter la notion de tâches : à chaque tâche attribuée, le poids du membre
  qui la reçoit diminue, ce qui réduit ses chances au tour suivant et
  rééquilibre visuellement la roue ;
- garder à l'écran un journal de qui a reçu quelle tâche ;
- exigence d'implémentation : le gagnant doit être déterminé D'ABORD par un
  tirage aléatoire pondéré, PUIS la roue est animée pour s'arrêter dessus —
  jamais l'inverse (faire tourner « au hasard » et lire le résultat sous le
  pointeur, ce qui ignorerait les poids).

### Ce qui a été construit

- **Modèle de données** : chaque membre est désormais un objet
  `{ name, weight }` (au lieu d'une simple chaîne). Poids par défaut de 10 à
  la création. Migration automatique au chargement si d'anciennes données de
  la phase 1 (tableau de chaînes) sont présentes dans `localStorage`.
- **Taille des parts proportionnelle au poids** : une fonction
  `computeSlices()` calcule les bornes angulaires de chaque part au prorata
  du poids de chaque membre par rapport au poids total (`part = poids / somme
  des poids × 360°`). Elle est utilisée à la fois pour dessiner la roue
  (`renderWheel`) et pour localiser la part du gagnant lors de l'animation
  (`spin`), afin que le rendu et le tirage restent strictement cohérents.
- **Tirage pondéré** : une fonction `weightedRandomIndex(weights)` tire un
  index selon une méthode classique de tirage par poids cumulés (on tire un
  nombre aléatoire entre 0 et la somme des poids, puis on parcourt les poids
  en le décrémentant jusqu'à passer sous 0). Vérifié statistiquement hors
  navigateur (100 000 tirages) : poids égaux → ~33 %/33 %/33 % ; poids
  `[10, 2, 2]` → ~71 %/14 %/14 % ; poids extrêmes `[1, 1, 50]` → ~2 %/2 %/96 %.
  Conforme aux proportions attendues.
- **Ordre tirage → animation (exigence explicite)** : dans `spin()`, le
  `winnerIndex` est calculé en tout premier via `weightedRandomIndex`, avant
  tout calcul de rotation. La rotation cible est ensuite déduite de la
  position de la part du gagnant déjà connu (`computeSlices()[winnerIndex]`).
  L'animation ne fait donc qu'illustrer un résultat déjà tiré au sort — elle
  ne le détermine jamais après coup.
- **Notion de tâche** : le bouton « Tourner » est remplacé par un formulaire
  (`assign-task-form`) avec un champ obligatoire « Nom de la tâche ». La
  soumission déclenche le tirage pondéré et l'animation ; une fois
  l'animation terminée, le poids du gagnant diminue
  (`WEIGHT_DECREMENT = 2`, plancher `MIN_WEIGHT = 1`), l'affectation est
  journalisée, et la roue/liste des membres sont redessinées avec les
  nouveaux poids.
- **Journal des tâches** : liste affichée à l'écran (`journal-list`), la plus
  récente en tête, au format « « Tâche » → Membre (date/heure) ». Persisté
  dans `localStorage` (clé `roue-fortache-journal`) au même titre que les
  membres, pour survivre à un rechargement de page.
- **Affichage du poids** : chaque membre de la liste affiche désormais son
  poids courant (« Alice — poids : 10 »), pour que l'effet du rééquilibrage
  soit visible sans avoir à observer uniquement la roue.
- Un court texte explicatif a été ajouté sous le titre « Membres de
  l'équipe » pour expliciter la règle de décroissance du poids (toujours
  sans mise en forme/CSS, conformément à la sobriété demandée en phase 1 et
  non remise en cause ici).

### Choix et compromis

- **Décroissance linéaire avec plancher** (`poids -= 2`, jamais en dessous de
  1) plutôt qu'une décroissance multiplicative (ex. poids × 0,5) : plus
  simple à expliquer et à démontrer à l'oral (« le poids baisse de 2 à
  chaque tâche »), et le plancher garantit qu'un membre garde toujours une
  chance non nulle d'être retiré, même après de nombreuses tâches — cohérent
  avec l'esprit « équitable » demandé.
- **Valeurs par défaut arbitraires** (`DEFAULT_WEIGHT = 10`,
  `WEIGHT_DECREMENT = 2`, `MIN_WEIGHT = 1`) : pas de configuration utilisateur
  pour l'instant (non demandée), juste des constantes en tête de script,
  faciles à ajuster si besoin plus tard.
- **Pas de bouton de réinitialisation des poids** : aurait été pratique pour
  rejouer une démo, mais non demandé ; ajouté seulement si le besoin se
  confirme, pour ne pas déborder du périmètre de la phase.
- **Le rééquilibrage visuel de la roue est volontairement immédiat et sans
  transition** juste après l'annonce du résultat (les parts se redessinent
  aux nouvelles proportions dès que le poids du gagnant a changé) : c'est
  exactement l'effet demandé (« la roue se rééquilibre donc visuellement
  toute seule »), pas un défaut d'animation.

### Bugs rencontrés et corrections

- Aucun bug bloquant. Le tirage pondéré a été testé isolément en Node.js
  (100 000 tirages sur plusieurs distributions de poids, cf. ci-dessus) avant
  intégration, pour vérifier la logique indépendamment du rendu et de
  l'animation. Un test de bout en bout via Playwright a ensuite confirmé,
  dans le navigateur réel : parts initiales à poids égaux, diminution du
  poids du gagnant après une tâche, rééquilibrage visuel de la roue,
  écriture et persistance du journal après rechargement de page, absence
  d'erreur console.

### État à la fin de la phase 2

Roue pondérée fonctionnelle avec gestion des tâches et journal persistant,
testée manuellement (Node.js pour la distribution statistique, Playwright
pour le parcours complet dans un navigateur). Le tirage est bien déterminé
avant l'animation, conformément à l'exigence d'implémentation.
