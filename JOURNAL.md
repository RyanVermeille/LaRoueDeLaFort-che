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

## Phase 3 — Habillage « fête foraine » et correction du curseur (2026-08-17)

### Demande

Habiller visuellement le prototype d'après une capture de référence fournie
en image (décor de théâtre/cirque sombre, rideaux de velours rouge, titre en
dégradé arc-en-ciel façon cirque, mise en page à trois colonnes sous le
titre — roue à gauche, résultat/journal au centre, équipe/nouvelle tâche à
droite), avec une exigence explicite : ne rien changer à la logique de
tirage validée en phase 2 (tirage pondéré D'ABORD, animation ENSUITE), et
uniquement réagencer/habiller l'existant — plus corriger le curseur, rendu
jusqu'ici sous la roue sans rien pointer.

### Ce qui a été construit

- **Décor de scène** : fond dégradé sombre avec lueur radiale chaude façon
  projecteur (`.spotlight`), deux rideaux de velours rouge en CSS pur
  (`repeating-linear-gradient` pour les plis + liseré doré), aucune image
  externe utilisée (uniquement des dégradés CSS), conformément à la
  contrainte « un seul fichier ».
- **Titre et sous-titre** : « LA ROUE DE LA FORTÂCHE » en police Google
  Fonts « Baloo 2 » (display, ronde, chargée via `<link>`), dégradé
  arc-en-ciel via `background-clip: text`, contour épais (`-webkit-text-stroke`)
  et halo lumineux (`text-shadow` multiple). Sous-titre « QUI S'Y COLLE ? »
  en doré avec lueur similaire.
- **Mise en page trois colonnes** en CSS Grid sur une seule hauteur d'écran
  (`.stage` en `100vh`, colonnes en `fr`, tailles en `clamp()`/`vw`/`vh`) :
  roue à gauche ; panneaux Résultat (haut) et Journal (bas) au centre ;
  panneaux Équipe (haut) et Nouvelle tâche (bas) à droite. Les listes
  (membres, journal) défilent **à l'intérieur de leur carte** si elles
  dépassent, pour ne jamais faire défiler la page entière sur un écran de
  laptop standard.
- **Roue** : anneau de 32 ampoules dorées généré en SVG autour de la jante
  (`renderBulbRing`), clignotant en décalé via une animation CSS
  (`@keyframes bulb-flicker` avec un `animation-delay` étalé par ampoule).
  Séparateurs blancs entre les parts, étiquettes en gras blanc avec contour
  sombre (`stroke` + `paint-order: stroke`) pour rester lisibles sur
  n'importe quelle couleur, y compris les parts fines (taille de police
  réduite automatiquement au-delà de 6 membres).
- **Couleur par membre cohérente partout** : une fonction `colorForName(name)`
  calcule une couleur déterministe (petit hash de chaîne) dans une palette
  vive de 12 couleurs. La même fonction est appelée pour la pastille dans
  « Équipe », la part de la roue et le nom dans le journal : un même nom a
  donc toujours la même couleur, y compris pour une entrée de journal dont
  le membre a depuis été retiré de l'équipe.
- **Panneau Résultat** : carte sombre arrondie, en-tête doré « RÉSULTAT DU
  TIRAGE », texte d'attente neutre avant tout tirage, nom du gagnant affiché
  en très grand avec halo lumineux. Une pluie de confettis (bibliothèque
  `canvas-confetti` via CDN jsDelivr) est déclenchée au moment où le
  résultat est révélé, centrée sur la carte Résultat.
- **Panneau Journal** : en-tête « JOURNAL DES TÂCHES ATTRIBUÉES », chaque
  entrée affiche le nom du membre dans sa couleur, suivi de la tâche et de
  l'heure (« Nom : Tâche [HH:MM] »), la plus récente en haut.
- **Section Équipe** : champ « Ajouter un membre... » + bouton rond « + »,
  chaque membre en pastille colorée (sa couleur) avec badge de poids et
  croix rouge « × » pour le retirer.
- **Section Nouvelle tâche** : champ « Nouvelle tâche » + gros bouton en
  dégradé rouge→rose « LANCER LA ROUE DE LA FORTÂCHE ! », qui déclenche le
  même tirage qu'avant (le bouton est désactivé pendant l'animation pour
  éviter un double clic).

### Correction du curseur (bug identifié dans la demande)

Le triangle indicateur (`#pointer` en phase 1/2) était placé dans le flux
normal du document, **sous** la roue, sans overlay ni positionnement — il
ne pointait donc rien. Il est remplacé par `.wheel-pointer`, positionné en
`position: absolute` en haut du cadre de la roue (`.wheel-frame`), pointant
vers le bas grâce à un triangle CSS (`border`), avec halo doré.

Point important : **la correction est purement CSS/HTML**. Le calcul de
rotation dans `spin()` (`computeSlices`, `weightedRandomIndex`, et la formule
`targetRotation = extraTurns - winnerMid`) utilise depuis la phase 1 la
convention « angle 0° = haut du cercle », donc la part gagnante était déjà
mathématiquement amenée sous l'angle 0° (le haut) à chaque tirage — seul le
triangle n'était pas positionné à cet endroit précis à l'écran. Aucune ligne
de la logique de tirage/rotation n'a donc été modifiée ; seul l'habillage du
pointeur a changé. Vérifié par un test automatisé (voir ci-dessous).

### Choix et compromis

- **Une seule police Google Fonts chargée** (Baloo 2, plusieurs graisses)
  plutôt que deux, pour limiter les requêtes externes et garder le fichier
  simple ; elle sert à la fois aux titres/boutons et au texte courant.
- **Dégradé continu sur tout le titre** plutôt qu'un dégradé différent par
  lettre : `background-clip: text` avec un dégradé arc-en-ciel unique donne
  déjà une couleur différente à chaque lettre (chacune affiche une tranche
  du dégradé) sans avoir à envelopper chaque caractère dans un `<span>` —
  plus simple, même effet visuel.
- **Étiquettes de la roue non tournées radialement** (restent horizontales) :
  une rotation radiale du texte aurait pu le faire apparaître à l'envers en
  bas de la roue sans logique de correction supplémentaire ; le texte
  horizontal, plus simple et plus sûr, reste toujours lisible dans le même
  sens grâce à la contre-rotation décrite ci-dessous.
- **Couleur du nom du gagnant dans le panneau Résultat** volontairement
  laissée en blanc/doré lumineux (style du titre) plutôt que dans la couleur
  du membre : l'exigence de cohérence de couleur ne portait que sur la
  pastille équipe, la part de roue et le nom dans le journal ; garder le
  panneau Résultat toujours très lumineux évite qu'un membre à couleur
  sombre y soit peu visible.
- **Suppression du paragraphe explicatif sur la décroissance du poids**
  (présent en phase 2 dans la section Équipe) : la nouvelle maquette ne
  prévoit pas cet espace ; l'information reste accessible via le badge de
  poids affiché sur chaque pastille et l'effet visible sur la roue.
- **Repli responsive à `max-width: 900px` / `max-height: 480px`** : les trois
  colonnes passent en une seule colonne empilée et la page peut alors
  défiler verticalement (`overflow-y: auto`) — c'est le seul cas où un
  défilement est autorisé, en dégradation, une fenêtre de laptop standard
  n'en a jamais besoin.

### Bugs rencontrés et corrections

- **Étiquettes à l'envers selon l'angle d'arrêt de la roue** : en donnant à
  `#wheel` (parts + étiquettes) une seule transformation CSS de rotation
  pour l'animation, les étiquettes tournaient rigidement avec les parts —
  et se retrouvaient donc la tête en bas dès que la rotation finale
  dépassait 90°–270°, y compris au repos entre deux tirages (pas seulement
  pendant l'animation). Repéré visuellement sur une capture de test (tirage
  forcé sur Bob avec une rotation de 180°, cf. `phase3-spin-0.5.png` avant
  correction). Corrigé en appliquant à chaque étiquette une contre-rotation
  SVG `rotate(-currentRotation, x, y)` autour de son propre point d'ancrage :
  composée avec la rotation CSS du `<svg id="wheel">` parent, elle annule
  l'inclinaison du texte (qui reste toujours droit) tout en laissant son
  ancrage suivre normalement sa part. Aucun changement à la logique de
  tirage : `currentRotation` reflète toujours exactement la rotation
  actuellement appliquée à la roue, donc la compensation est correcte à
  chaque rendu (chargement initial, ajout/suppression de membre, fin
  d'animation).
- **Panneaux écrasés à hauteur quasi nulle sur petit écran** : dans le repli
  responsive à une colonne, les cartes (`flex: 1 1 0` dans un conteneur dont
  la hauteur n'est plus contrainte par la grille à trois colonnes)
  s'effondraient à leur taille minimale, ne laissant voir que leur titre.
  Corrigé en fixant `flex: 0 0 auto` et une `min-height` sur les cartes,
  et un plafond de hauteur avec défilement interne sur les listes, dans la
  media query dédiée.
- Aucun autre bug bloquant. Vérifications automatisées via Playwright :
  - **Alignement géométrique du curseur** : en interceptant `Math.random`
    pour forcer trois gagnants différents (Alice/Bob/Charlie), lecture de la
    rotation réellement appliquée (`getComputedStyle(...).transform`) et
    recalcul indépendant, côté test, de la part qui se retrouve sous l'angle
    0° (le pointeur) — coïncide exactement (écart de 0.000°) avec le nom du
    gagnant annoncé dans les trois cas.
  - Absence de défilement de page sur trois résolutions de laptop courantes
    (1366×768, 1280×720, 1440×900), avant tirage et avec 8 membres +
    plusieurs tâches attribuées ; repli propre vérifié à 800×700.
  - 32 ampoules bien présentes dans l'anneau, canvas de confettis bien
    injecté après un tirage, aucune erreur console dans tous les scénarios.

### État à la fin de la phase 3

Prototype habillé façon fête foraine, sur un seul écran, sans toucher à la
mécanique de tirage pondéré de la phase 2 (vérifié géométriquement par
test). Le curseur pointe désormais correctement la part gagnante. Couleurs
cohérentes entre équipe, roue et journal. Testé sur plusieurs résolutions et
avec un nombre de membres variable.

## Phase 4 — Retrait de la persistance, réglage manuel des poids, limite de participants, réinitialisation spectaculaire (2026-08-17)

### Demande

Quatre adaptations, sans toucher à la logique de tirage validée en phase 2
(gagnant tiré D'ABORD par tirage pondéré, PUIS animation) ni au style « fête
foraine » de la phase 3 :

- supprimer toute la persistance `localStorage` (membres, poids, journal) —
  le prototype doit repartir de zéro à chaque rechargement ;
- ajouter un réglage manuel du poids de chaque membre (boutons « − » / « + »
  par pastille), borné entre 1 et 20, en plus de la décrémentation
  automatique existante après tirage (les deux mécanismes coexistent) ;
- limiter l'équipe à 15 participants, avec message et bouton d'ajout
  désactivé une fois la limite atteinte ;
- un gros bouton rouge « TOUT RÉINITIALISER » qui coupe l'écran au noir
  instantanément, réinitialise tout l'état pendant que l'écran est noir, puis
  fait réapparaître l'application en fondu sur environ 3 secondes.

### Ce qui a été construit

- **Suppression complète de `localStorage`** : `loadMembers()` et
  `loadJournal()` retournent désormais directement des valeurs par défaut en
  mémoire (`getDefaultMembers()` pour les membres, `[]` pour le journal),
  sans plus jamais lire ni écrire dans le navigateur. Les fonctions
  `saveMembers()` et `saveJournal()` ainsi que tous leurs appels (ajout,
  suppression de membre, fin de tirage) ont été supprimés — plus aucune trace
  de `localStorage` dans le fichier.
- **Réglage manuel du poids** : chaque pastille de membre affiche maintenant
  deux boutons ronds « − » et « + » de part et d'autre du badge de poids.
  Une nouvelle fonction `adjustWeight(index, delta)` modifie le poids d'une
  unité, borné par `Math.min(MAX_WEIGHT, Math.max(MIN_WEIGHT, ...))` (nouvelle
  constante `MAX_WEIGHT = 20`, `MIN_WEIGHT = 1` déjà existante), puis
  redessine la liste et la roue immédiatement. Les boutons se désactivent
  eux-mêmes aux bornes (`disabled` posé directement à partir du poids courant
  dans `renderMemberList()`). Ce réglage ne remplace pas la décrémentation
  automatique de `WEIGHT_DECREMENT` après un tirage dans `spin()` : les deux
  cohabitent sans conflit, un poids pouvant être ajusté manuellement entre
  deux tirages.
- **Limite de 15 participants** : nouvelle constante `MAX_MEMBERS = 15`.
  `addMember()` refuse tout ajout au-delà de cette limite, et une fonction
  `updateMemberLimitUI()` désactive à la fois le champ de saisie et le
  bouton « + » et affiche le message « Maximum 15 participants »
  (`#member-limit-message`, masqué par défaut) dès que la limite est
  atteinte. Appelée après chaque ajout/suppression de membre et au
  chargement initial.
- **Bouton « Tout réinitialiser »** : gros bouton rond rouge en
  `position: fixed` en haut à droite de l'écran (`.reset-all-btn`, dégradé
  radial rouge vif façon détonateur), avec une confirmation `confirm()`
  avant de déclencher l'effet. Un `<div id="blackout-overlay">` plein écran
  (`position: fixed; inset: 0; z-index: 9999`) sert d'overlay noir. Au clic,
  `resetEverything()` :
  1. passe l'overlay à `opacity: 1` avec `transition: none` (coupure
     instantanée), et force un reflow (`void blackoutOverlayEl.offsetHeight`)
     pour garantir que ce changement est bien appliqué avant l'étape 3 ;
  2. réinitialise tout l'état en mémoire pendant que l'écran est noir :
     `members = getDefaultMembers()`, `journal = []`, remise à zéro de la
     rotation de la roue, du texte de résultat et du champ de tâche, puis
     redessine membres/roue/journal ;
  3. sur la frame suivante (`requestAnimationFrame`), rétablit la transition
     CSS (`transition: opacity 3s ease`) et ramène l'opacité à 0, révélant
     l'application déjà réinitialisée en fondu progressif ;
  4. après 3 secondes, repasse `pointer-events` à `none` sur l'overlay pour
     ne plus bloquer les clics une fois le fondu terminé.

### Choix et compromis

- **Persistance retirée : limite assumée du prototype**, pas une régression.
  La persistance de la phase 1/2 avait été ajoutée spontanément « pour ne pas
  perdre la liste pendant les tests manuels », mais n'avait jamais été
  demandée explicitement ; la retirer maintenant recentre le prototype sur sa
  vocation de démonstration rejouable à volonté (chaque rechargement — ou
  chaque clic sur « Tout réinitialiser » — repart d'un état propre), au prix
  explicite de perdre l'équipe et le journal à chaque fermeture d'onglet.
- **Bouton de réinitialisation en position fixe, hors de la grille des trois
  colonnes** : plutôt que de l'intégrer dans un des panneaux existants, il
  est positionné en `position: fixed` au-dessus de tout le reste
  (`z-index: 50`), pour rester visible et accessible quel que soit
  l'agencement des colonnes — et ne pas devoir être déplacé lors du
  réagencement de mise en page prévu en phase 5.
- **`confirm()` natif du navigateur** pour la confirmation avant
  réinitialisation, plutôt qu'une modale personnalisée : suffisant pour un
  prototype, évite d'ajouter un nouveau composant d'UI pour une action
  optionnelle et ponctuelle.
- **Reflow forcé (`offsetHeight`) avant d'attacher la transition de fondu** :
  sans ce forçage, le navigateur pourrait fusionner le changement d'opacité
  instantané et la pose de la transition dans le même cycle de rendu, et
  animer l'apparition du noir au lieu de la rendre instantanée — casserait
  l'effet « coupure de courant » demandé.
- **Boutons « − » / « + » désactivés directement via l'attribut `disabled`
  posé au rendu**, plutôt qu'un `pointer-events: none` en CSS conditionnel :
  plus simple, cohérent avec le reste du fichier (boutons d'ajout/de tâche
  déjà gérés ainsi), et bénéficie gratuitement du style `:disabled` déjà
  défini pour les autres boutons ronds.

### Bugs rencontrés et corrections

Aucun bug bloquant. Vérifié via un script Playwright dédié (navigation
directe sur `index.html`, sans serveur) : le poids d'un membre monte
jusqu'à 20 puis le bouton « + » se bloque, redescend jusqu'à 1 puis le
bouton « − » se bloque ; l'ajout de membres s'arrête à 15 avec le champ et
le bouton désactivés et le message affiché ; l'overlay passe à une opacité
proche de 1 immédiatement après le clic sur « Tout réinitialiser » (état
remis à 3 membres par défaut pendant que l'écran est noir, journal vidé),
puis retombe à 0 après le fondu ; un rechargement de page complet ramène
bien l'équipe par défaut (Alice/Bob/Charlie) et un journal vide, confirmant
l'absence totale de persistance ; aucune erreur console dans tous les
scénarios.

### État à la fin de la phase 4

Prototype sans aucune persistance, avec réglage manuel du poids par membre
(coexistant avec la décrémentation automatique), équipe plafonnée à 15
participants, et bouton de réinitialisation complète avec effet de coupure
de courant. Logique de tirage pondéré et style visuel de la phase 3
inchangés. Mise en page encore identique à la phase 3 à ce stade — le
réagencement en trois colonnes (roue au centre) est traité séparément en
phase 5.
