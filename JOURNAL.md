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
