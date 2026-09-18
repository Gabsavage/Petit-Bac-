# CLAUDE.md

Contexte projet pour Claude Code. Lis ce fichier avant de faire des changements — il documente des choix et des pièges qui ne sont pas évidents en lisant juste le code.

## C'est quoi

App web "Petit Bac" multijoueur en temps réel. **Un seul fichier** `index.html` autonome (HTML + CSS + JS inline), pas de build, pas de framework, pas de `package.json`. Déployé sur Vercel, connecté à ce repo GitHub — un push sur `main` redéploie automatiquement en ~20-30s. Les autres fichiers à la racine (`manifest.json`, `favicon*.png/ico`, `apple-touch-icon.png`, `icon-*.png`) sont les assets PWA, référencés en chemins absolus (`/favicon.ico`, etc.) depuis `index.html`.

**Icônes** : « PB » en Fraunces crème, ombre portée dure à l'encre, crayon moutarde en bas à droite qui mord sur le B, le tout sur fond brique. Elles ne sont pas dessinées à la main : la même compo est composée en HTML/SVG (la vraie Fraunces-Bold embarquée en base64, le crayon de `brandMark()` avec le corps en `--accent-2`), Chromium capture, Pillow réduit et assemble le `.ico`. Trois points à ne pas perdre si tu les régénères. Le **16px n'a pas de crayon** : à cette taille il devient une tache et mange la lisibilité du PB. **La superposition du crayon dépend de la taille** : bien remonté sur le B au-dessus de 100px (écran d'accueil, PWA), un cran plus bas à 24/32/48px, parce que le crayon referme la boucle du B une fois réduit et que l'icône se met à se lire « PP ». Et `icon-512-maskable.png` est une version **rentrée à 74 %** parce qu'Android recadre les icônes `purpose: maskable` dans un cercle de 80 % — avec l'icône pleine page, la gomme du crayon et la sérif gauche du P sautent. Les trois se vérifient à l'œil : rendre chaque taille en vrai, zoomer en `image-rendering:pixelated`, et passer la 512 sous un masque circulaire.

## Règles de code à respecter

- **Un seul fichier** `index.html` — tout le CSS et le JS restent inline dedans. N'éclate pas en fichiers séparés sauf si explicitement demandé (ça casserait le côté "un seul fichier à déployer").
- **Style JS volontairement conservateur** : `var`/`let`/`const` basiques, `function(){}` classiques (pas d'arrow functions), pas de destructuring, pas de template literals dans le rendu HTML (concaténation `+` avec guillemets simples partout). Ce n'est pas une contrainte technique dure, juste la cohérence avec l'existant — continue dans ce style pour les nouvelles fonctions plutôt que de mélanger les styles.
- **Rendu** : tout le HTML est généré par des fonctions `render*()` qui retournent des strings, injectées via `.innerHTML`. Pas de framework, pas de virtual DOM, pas de diffing. Les événements passent par délégation sur `document` avec des attributs `data-action="..."` — un seul listener `click` global qui dispatche sur `btn.dataset.action` dans un `if/else if` en bas du fichier.
- **Avant tout commit/push** : vérifie que chaque `data-action` utilisé dans un template a bien un `else if (action === '...')` correspondant dans le dispatcher, et inversement (une action orpheline ou un handler mort sont des bugs silencieux fréquents ici). Vérifie aussi la syntaxe JS extraite du `<script>` inline (commande ci-dessous) avant de pousser — une erreur de syntaxe casse toute l'app puisqu'il n'y a qu'un fichier.

```bash
# Extrait et valide la syntaxe du <script> inline (celui sans src=)
python3 -c "
import re
html = open('index.html', encoding='utf-8').read()
for m in re.finditer(r'<script([^>]*)>(.*?)</script>', html, re.S):
    if 'src=' in m.group(1): continue
    open('/tmp/extracted.js','w',encoding='utf-8').write(m.group(2))
"
node --check /tmp/extracted.js
```

## Stack

- **Firebase Realtime Database** (pas Firestore) pour l'état partagé. SDK **compat** chargé via `<script src=.../firebase-app-compat.js>` (API namespaced `firebase.database()`, PAS l'API modulaire v9+ à imports).
- **QR code** : `qrcodejs` (davidshimjs) via cdnjs, API `new QRCode(element, options)`.
- Identité visuelle "carnet de scores / plateau de jeu" : fond papier crème + grille de points en `radial-gradient`, encre foncée pour le texte, un seul accent (brique `--accent`) réservé aux actions primaires et un second (moutarde `--accent-2`) pour les highlights ludiques (badge hôte, 1ère place, bannière MVP). Trois polices Google Fonts avec un rôle chacune : **Fraunces** (serif) pour les titres/gros chiffres, **Inter** pour le texte UI, **JetBrains Mono** pour tout ce qui est chiffré (timer, PIN, scores). Pas d'emoji utilisé comme icône fonctionnelle — la fonction `icon(name)` (SVG inline, trait `currentColor`) couvre boutons/actions ; les emojis restent réservés aux avatars joueurs (choix ludique assumé), aux lettres/tuiles décoratives et au picto devant chaque catégorie dans le salon (`categoryEmoji()`) — là non plus l'emoji ne porte pas d'information seul, le nom de la catégorie est toujours écrit à côté. La marque du bandeau (`brandMark()`, le crayon) est volontairement **à part** d'`icon()` : les icônes d'interface sont en trait fin monochrome, la marque est une forme pleine cernée d'encre comme les tuiles — un crayon en trait fin détonnait à côté du reste.
- **Polices chargées sans bloquer le rendu** (`<link media="print" onload="this.media='all'">`, id `font-css`). En feuille de style classique, rien ne s'affichait tant qu'elle n'était pas arrivée : écran blanc, puis animation du splash déjà terminée puisque ses délais avaient couru pendant l'attente. Le départ de l'animation attend explicitement la police (`whenDisplayFontReady`, plafond de 600ms) — ne repasse pas ce `<link>` en feuille bloquante.
- Motif visuel récurrent : "tuile tamponnée" — bordure encre + ombre portée dure (offset, pas de flou) sur les tuiles de lettre, le logo, les digits de PIN, les avatars et les CTA (`.cta`). Les cartes de contenu (réponses, review, listes) restent plates avec une simple bordure fine, pour garder une hiérarchie claire entre "élément interactif/ludique" et "conteneur d'info".
- Projet Firebase : `petit-bac-c24bf`. Config déjà dans `index.html` (l'apiKey Firebase web n'est pas un secret, normal qu'il soit visible côté client — la sécurité passe par les Rules).
- **Règles Firebase en mode test** (lecture/écriture ouverte). Ça expire ~30 jours après activation — à surveiller, sinon l'app cesse de fonctionner sans prévenir côté code.

## Modèle de données Firebase

```
games/{pin}/                    pin = code à 4 chiffres, sert aussi de clé Firebase
  hostName, hostKey             hostKey = clé du créateur, seul à pouvoir lancer une manche (handleStartRound)
  categories: [string, ...]
  roundDurationMs, maxRounds    par défaut 120000 et 3 (DEFAULT_ROUND_MS / DEFAULT_MAX_ROUNDS)
                                maxRounds: 0 = illimité, sinon 3/5/10
  createdAt, lastActivity       lastActivity sert au nettoyage auto (INACTIVITY_MS = 3 min)
  recentLetters: [string, ...]  13 dernières lettres tirées (RECENT_LETTERS_MAX), survit à "Rejouer"
  players/{key}: { name, emoji, color, seen }   seen = horodatage du dernier battement de coeur (10s)
  currentRound: {
    number, letter, revealAt, endTime,
    finishedAt, finishedBy,     posé par le 1er joueur qui clique "J'ai fini" -> 5s de grâce pour les autres
    answers/{playerKey}: { "0": "mot", "1": "mot", ... },   // clé = INDEX de la catégorie, pas son nom
    votes/{catIndex}/{targetPlayerKey}/{voterKey}: false    // on ne vote QUE pour refuser (voir plus bas)
    reviewIndex,                catégorie affichée pendant la validation, partagée par tous
    showLeaderboard,            true = tout le monde bascule sur le classement
    leaderboardAt               départ commun du roulement de tambour avant le podium
  }
  history/{roundNumber}: <ancien currentRound archivé tel quel>
```

Les deux valeurs par défaut doivent rester parmi les choix proposés dans la feuille de réglages (60/90/120s et 3/5/10/illimité) : sinon, à l'ouverture des réglages, aucun bouton n'apparaît actif.

**Tirage des lettres** : `recentLetters` est la mémoire glissante des 13 dernières lettres, gardée à la racine de la partie et **pas** déduite de `history` — celui-ci est effacé par "Rejouer" et par la remise à zéro des scores, une lettre pouvait donc revenir juste après. Sur 19 lettres au pool, il en reste toujours 6 tirables. Les parties créées avant ce champ retombent sur l'historique (`recentLetters()`).

**Tirage des catégories** : à la création d'une partie, `attemptCreatePin` appelle `randomCategories(CATEGORIES_PER_GAME)` au lieu de `DEFAULT_CATEGORIES` (qui ne sert plus que de repli pour l'état initial et les parties sans champ `categories`). `CATEGORY_BANK` est un tableau de **familles** (classiques / lieux / culture / nourriture / quotidien / nature / pop et fun) ; le tirage mélange chaque famille puis fait un tour de table — une catégorie par famille, puis on recommence. Sans ça, un tirage à plat sortait régulièrement quatre catégories « nourriture ». Conséquence à garder en tête : sur 7 familles et 8 tirages, **deux catégories de la même famille peuvent se retrouver ensemble**, donc pas de quasi-doublons dans la banque (« Animal » et « Animal sauvage » ne doivent pas coexister), et pas deux fois le même emoji sur des catégories qui peuvent tomber ensemble.

Le tirage se fait **à la création et à chaque retour au salon** (`handleReplayGame`, `handleResetScores`) — une partie terminée puis relancée ne rejoue pas la même grille. Jamais au début d'une manche, en revanche : les catégories corrigées à la main dans les réglages seraient écrasées à la manche 2. Pour en changer, l'édition du textarea (`handleSaveCategories`) et le bouton « Tirer d'autres catégories » (`handleShuffleCategories`) restent dispo dans le salon. `handleShuffleCategories` écrit dans Firebase **et** dans le textarea ouvert : la feuille de réglages n'est pas re-rendue par un snapshot, sans ça elle continuerait d'afficher l'ancienne liste.

**Ce qui fait une bonne catégorie** (critère à appliquer avant d'en ajouter une) : elle doit être **vérifiable** (on peut trancher si une réponse est valide, sinon l'écran de validation n'a rien à juger), **bornée** (assez de réponses par lettre, mais pas « tout nom commun »), et **propice aux collisions** (c'est l'écart 2 pts / 1 pt qui rend la manche intéressante). Une première fournée de catégories ouvertes — « Ce qui coûte cher », « Ce qui fait peur », « Truc qu'on perd tout le temps », « Objet », « Nourriture » — a été retirée pour cette raison : tout y passait, personne ne tombait jamais sur le même mot, et le vote devenait arbitraire. Les « Ce qu'on trouve dans… » qui restent (salle de classe, sandwich, salle de bain, cuisine) sont gardés parce que le lieu borne vraiment la réponse. Éviter aussi les catégories qui se contiennent l'une l'autre (« Nourriture » et « Dessert », « Destination de vacances » et « Pays ») : deux catégories de familles différentes peuvent tomber dans la même partie.

`categoryEmoji(name)` cherche un mot-clé dans le nom **normalisé** (`CATEGORY_EMOJIS`, du plus spécifique au plus générique) plutôt qu'une table nom -> emoji : les catégories peuvent être réécrites librement, une catégorie inconnue retombe sur ✏️. L'ordre des entrées compte (« dans une ville » avant « ville », « cuisine » avant « objet », « poisson » avant « animal », « danse » avant « musique ») — si tu ajoutes un mot-clé, vérifie qu'il n'est pas un sous-mot d'une entrée placée avant lui. Le piège est vicieux : « Gâteau ou viennoiserie » héritait du picto de cinéma parce que *viennoi-série* contient « serie ». Un petit script qui rend toute la banque et signale les emojis partagés attrape ça en une seconde.

**Reprise du rôle d'hôte** : il n'y a pas de système de présence Firebase (`onDisconnect`) — chaque client écrit `players/{key}/seen` toutes les 10s depuis `tick()` (`sendHeartbeat`), et un hôte qu'on n'a plus vu depuis `HOST_GONE_MS` (90s) est repris par un autre joueur (`maybeAdoptHost`). Quatre choses à ne pas casser :

- Le battement de cœur n'écrit **que** `players/{key}/seen`, jamais `lastActivity` : sinon un onglet oublié dans un coin empêcherait à jamais le nettoyage automatique des parties inactives.
- On ne promeut que si l'hôte a **déjà un `seen`**. Un onglet resté sur une version antérieure du code n'en écrit jamais : sans ce test, il se ferait destituer alors qu'il joue encore. Pas de `seen` = on ne touche à rien.
- Le successeur est la **plus petite clé** parmi les joueurs vus récemment (`pickNewHost`) : c'est arbitraire mais déterministe, donc tous les clients désignent le même et on ne se repasse pas le rôle en boucle. L'écriture passe par une **transaction** sur `hostKey` (`promoteHost`), qui abandonne si quelqu'un a déjà repris la main.
- 90s paraît long, c'est volontaire : `seen` est écrit depuis `tick()`, or les mobiles throttlent les timers d'un onglet en arrière-plan. À 45s, un hôte qui répond juste à un message se faisait destituer. Le départ volontaire, lui, passe la main immédiatement (`handleLeaveRoom`) — les 90s ne concernent que les départs brutaux (onglet fermé, réseau coupé).

`seen` est aussi posé à la création et à l'arrivée dans une partie, sinon un joueur qui vient d'entrer ne serait pas éligible comme successeur pendant 10s, et une reconnexion (qui réécrit l'objet joueur en entier) l'effacerait.

**Scoring** (`scoreRound`) : une réponse est **valide par défaut**, l'écran de validation ne permet que de refuser, et il faut la **majorité des autres joueurs** pour l'exclure (`refusalsNeeded` / `isRefused`). L'auteur ne vote pas pour lui-même, donc le dénominateur est `nbJoueurs - 1` et le seuil est `floor(votants / 2) + 1` : à 4 joueurs il faut 2 refus sur 3 votants, à 3 joueurs il faut les 2 autres, à 2 joueurs le seul autre suffit. Le dénominateur, ce sont les **votants possibles** et pas ceux qui ont effectivement voté — sinon, à 10 joueurs, deux personnes pressées suffiraient. Le blocage de l'auto-vote est dans le rendu (pas de bouton sur sa propre réponse) **et** dans `handleVote`. Les anciens votes « oui » des manches archivées ne comptent plus que comme « pas un refus », ce qui donne le bon résultat. Une réponse d'une seule lettre vaut 0 sans vote. Les doublons sont détectés à la faute de frappe près (`sameWord`, distance de Levenshtein : 2 fautes à partir de 7 lettres, 1 à partir de 4, aucune en dessous) pour que "Hongrie"/"Hongrir" donnent 1 pt chacun.

**Abréviations** (`isAbbrevOf`, utilisé par `sameWord`) : « foot » et « football » sont le même sport, mais l'écart de longueur dépasse de loin la tolérance aux fautes de frappe. Un mot qui est le **début** d'un autre est donc considéré comme le même, sous trois conditions : même nombre de mots (sinon « football » avalerait « football américain »), au moins 4 lettres dans le plus court (écarte « car »/« carotte »), au moins 3 lettres d'écart (écarte « ours »/« oursin » et « chat »/« chaton »). Ne couvre que les troncatures pures : « resto »/« restaurant » ou « ciné »/« cinéma » changent une lettre et passent à travers. Faux positifs assumés : « train »/« traineau », « porc »/« porcelet » — rares parce que la comparaison ne se fait qu'à l'intérieur d'une même catégorie. Si tu desserres une de ces trois conditions, reteste ces paires-là en premier.

**Articles en tête de réponse** (`coreWord`) : « Une Ferrari » et « Ferrari », c'est le même mot — sans retirer l'article, l'écart de longueur dépassait le seuil de `sameWord` et les deux réponses valaient 2 pts chacune. `coreWord` sert au regroupement des doublons **et** à l'alerte « ne commence pas par la lettre du tour » (c'est le F de Ferrari qui compte), jamais à l'affichage : l'écran montre toujours ce que le joueur a tapé. Un **seul** article est retiré, les entrées sont testées de la plus longue à la plus courte (« de la » avant « de »), et on ne vide jamais la réponse : « L' » tout seul reste « l' », sinon il passerait de « réponse écrite » à « pas de réponse ». Attention si tu allonges la liste : un article ne matche qu'avec son espace ou son apostrophe, c'est ce qui empêche « Laurent » de devenir « urent ».

**Piège important, ne pas régresser dessus** : les réponses et les votes sont indexés par **position dans le tableau `categories`** (`"0"`, `"1"`, `"2"`…), jamais par le nom de la catégorie. C'est volontaire : Firebase interdit `.` `#` `$` `[` `]` `/` dans les clés d'un objet, et une catégorie comme "Pays / Villes" cassait silencieusement l'écriture (`.set()` échouait, d'où un bug déjà vécu où aucune réponse n'apparaissait). Si tu ajoutes une nouvelle donnée indexée par catégorie, indexe-la par position (`idx` dans les boucles `categories.forEach(function(cat, idx){...})`), jamais par `cat` directement.

## Machine à états (`getPhase`)

`lobby → countdown (3s, revealAt) → playing → reveal (review + vote) → countdown suivant`

- `reveal` recouvre deux vues : la validation catégorie par catégorie, et l'écran de classement/podium. Les deux sont **synchronisées via Firebase** (`currentRound/reviewIndex` et `showLeaderboard`, lus par `reviewIndexOf()` / `showingLeaderboard()`) : tout le monde valide la même catégorie en même temps. C'était local auparavant — si tu repasses des bouts en local, tu casses la synchro.
- **Seul l'hôte fait avancer la validation** (`handleReviewStep`, `handleShowLeaderboard`, même garde `myKey !== state.hostKey` que `handleStartRound`) : à plusieurs, n'importe qui pouvait enchaîner sur la catégorie suivante pendant que les autres lisaient encore. Le vote, lui, reste ouvert à tout le monde — c'est tout l'intérêt de l'écran. La garde est à la fois dans le rendu (les non-hôtes voient une ligne d'attente à la place du bouton) et dans le handler : ne retire pas la seconde en gardant la première.
- Le "5s de grâce" après le premier "J'ai fini" est géré par `finishedAt` + `FINISH_GRACE_MS`, voir `roundEndsAt()` — la fin de manche réelle est `min(endTime, finishedAt + 5000)`.
- Le podium est animé par paliers (3e, 2e, 1er + projecteur + confettis, puis les 4e et suivants). Seule la *lecture* de l'animation est locale (`podiumRevealed`, `podiumSettled`, `confettiFired`) ; son point de départ vient de `leaderboardAt`, partagé, pour que tout le monde voie le podium au même moment. `podiumSettled` retire les classes d'animation une fois la séquence jouée : sans lui, le moindre snapshot Firebase rejouerait toute la révélation.
- **État local propre à une manche** : `localAnswers`, `locked` et `hasAutoSubmitted` sont resynchronisés par `syncRoundLocalState()` à chaque changement de numéro de manche. Ne les remets pas à zéro uniquement à l'entrée dans la partie : c'était le cas avant, d'où des réponses de la manche précédente pré-remplies, grisées pour qui avait fini premier, et surtout plus aucun envoi de réponses dès la 2e manche.

## Écran de salle d'attente

Refait pour ressembler à un salon de jeu plutôt qu'à une fiche : les **joueurs sont en haut**, en grille alignée à gauche, avec une case « + » permanente qui ouvre l'invitation — un salon qui se remplit, pas un avatar seul au milieu. Chaque cellule réserve une ligne (`.tag-slot`) pour le badge « hôte », badge ou pas : sans ça la cellule de l'hôte est plus haute et décale toute la rangée suivante.

Le **QR a quitté `#content`** pour la feuille du bas (`openInviteSheet`), à 200px au lieu de 100 : il se scanne mieux et il ne mange plus le haut de l'écran une fois tout le monde arrivé. `renderQr()` est donc appelé depuis `openInviteSheet()` et **plus** depuis `renderTop()` — si tu remets un QR dans le salon, il faut re-brancher cet appel. La feuille propose aussi « Partager le lien » (`handleShareLink`) : partage natif si `navigator.share` existe, presse-papier sinon, toast avec l'URL en dernier recours. Les trois peuvent échouer ou être annulés, aucun n'est traité comme une erreur.

**Sortir du salon** se fait par `#leave-btn`, à droite de la barre du haut — un bouton à part, pas un `#back-btn` à deux visages (`renderTop()` ne touche que son `hidden`, comme les autres boutons statiques du shell). Il n'apparaît **qu'au salon** : en pleine manche, une sortie à portée de pouce ferait partir des gens par accident, et la feuille de réglages garde le lien pour ce cas. `handleLeaveRoom` demande confirmation, parce que partir efface le PIN de l'URL (`goHome` fait un `replaceState`) et que revenir suppose le QR, le lien ou la liste des parties en cours.

La barre du bas du salon **ne porte plus que « Lancer la manche »** : les réglages s'ouvrent par les deux pastilles de règles, qui appellent déjà `open-menu`. Si tu retires ces pastilles, remets une entrée vers la feuille — sinon la durée et le nombre de manches deviennent inatteignables au salon.

Les **règles du match** (nombre de manches, durée) sont affichées en deux pastilles cliquables qui ouvrent les réglages : c'était invisible avant, il fallait ouvrir la feuille pour savoir en combien de manches on jouait. Les **catégories** sont une liste alignée (`.cat-list`) et non plus un nuage de pastilles centrées, avec le re-tirage à portée de pouce dans l'en-tête — réservé à l'hôte, comme le lancement.

## Fonds « page de cahier »

Le fond est en deux morceaux, et c'est délibéré :

- **Le papier réglé est dessiné en CSS** (`.paper-doodles`, deux dégradés : la marge rouge et un `repeating-linear-gradient` pour les lignes). Il ne coûte rien, reste droit, se recolore avec le thème et s'adapte à n'importe quelle hauteur d'écran.
- **Les griffonnages sont 26 masques** dans `bg/d/` (83 Ko au total, dont 20 Ko pour le seul écran d'accueil). **Sept d'entre eux ne sont placés nulle part** (`ballon`, `bonhomme`, `cartable`, `chaise`, `cloche`, `de`, `horloge`, 28 Ko) : ils sont en réserve, pas oubliés — le salon n'a aucune bande libre et les autres écrans sont pleins. Ils ne sont pas préchargés non plus, `preloadOtherDoodles` ne parcourt que `PAPER_DOODLES`. Ne les supprime pas en croyant à des fichiers morts, détourés des images générées par ChatGPT. Ce sont des PNG en niveaux de gris + alpha, posés en `mask-image` sur un `<span>` dont la `background-color` est `--doodle-ink` : **un seul fichier sert aux deux thèmes**, seule la couleur change — bic bleu sur le papier clair, crème sur la page sombre (le bleu y virait au terne, la craie sur ardoise marche bien mieux).

Ils ont d'abord été intégrés comme huit images de fond pleine page (une par écran et par thème, 273 Ko). Ça ne marchait pas, pour une raison qui vaut d'être retenue : **une composition figée ne connaît pas la mise en page**. Le brief de génération disait « dessins dans les coins, centre vide », alors que l'app met son contenu au centre et son châssis dans les coins — donc les dessins tombaient sous le bandeau et les boutons pendant que le milieu restait désespérément vide. Et une image calée sur un écran de 844pt tombe à côté sur un téléphone plus haut.

`PAPER_DOODLES` place donc chaque dessin à la main, par écran. Chacun est ancré **en haut** (`t`), **en bas** (`b`) ou **au centre** (`c`, décalage en px) : le bandeau et la barre du bas ont une hauteur fixe, un dessin calé dessus reste à sa place quelle que soit la hauteur de l'écran, là où un pourcentage dériverait. `x` positionne depuis la gauche, `rx` depuis la droite.

**Les ancres passent par les encoches, et c'est indispensable.** `renderDoodles()` écrit `top:calc(var(--safe-top) + Npx)`, `bottom:calc(var(--safe-bottom) + Npx)` et, pour le centre, `calc(50% + (var(--safe-top) - var(--safe-bottom)) / 2 + Npx)`. Le bandeau descend de `--safe-top` et la barre du bas remonte de `--safe-bottom`, donc toute l'UI glisse sur un téléphone encoché — alors que le calque, lui, couvre le shell entier. En `top:140px` fixe, l'horloge de l'écran de manche chevauchait la pastille de score de 26px sur un iPhone à Dynamic Island, tout en tombant parfaitement sur un écran sans encoche. **Ce bug est invisible en headless** : `env(safe-area-inset-*)` y vaut 0. Pour le reproduire, injecter `:root{--safe-top:59px;--safe-bottom:34px;}` avec `addStyleTag` — le code lisant `var(--safe-top)`, ça simule l'encoche à l'exact. Le contrôle qui compte est l'**écart entre un dessin et l'élément d'UI voisin**, qui doit être identique avec et sans inset ; une position absolue ne prouve rien.

Les bandes réellement libres, mesurées à 390×844 :

| écran | bandes libres (px depuis le haut) |
|---|---|
| accueil | 60–180, 360–500, 700–844 |
| salon | aucune |
| manche | 660–760 |
| podium | 60–180, 300–760 |

Sur l'écran de manche, `.play-header` (lettre + minuteur) est remonté de 14px et `.answer-scroll` reprend exactement les mêmes 14px en `padding-top` : la zone de défilement grandit d'autant vers le haut, donc la carte centrée dedans ne bouge pas. **Les deux nombres vont par paire** — remonter l'en-tête sans compenser déplace la carte de la moitié de l'écart. Les griffonnages du haut (`chrono`, `sablier`) ont été remontés d'autant, sinon ils cessent de flanquer la tuile.

Sous la carte il ne reste qu'une bande de **~70px** (62px sur un écran de 667) : les quatre dessins du bas doivent y tenir **en entier**, bords compris. Le crayon a dû être rapetissé pour ça — une boîte tournée de 14° est plus haute que sa hauteur nominale, c'est l'`getBoundingClientRect()` qui fait foi, pas le `h` de la config. Le contrôle à faire après toute retouche de mise en page ici est « haut du dessin > bas de `.answer-list` », jamais la valeur de `b` seule.

**`.answer-list` et `.answer-row` portent `flex-shrink:0`, et c'est vital.** `.answer-scroll` est un conteneur flex en colonne, donc par défaut la carte se laisse **écraser** au lieu de déborder. Son `overflow:hidden` (qui sert au rayon des coins) rognait alors les dernières lignes, et comme rien ne débordait, `scrollHeight == clientHeight` : aucun défilement possible, la catégorie devenait tout simplement inatteignable — impossible de répondre. Mesuré avec 8 catégories aux libellés longs : une catégorie perdue sur un 393×852, **cinq** sur un 375×667. Le symptôme trompe, on croit à un problème de hauteur alors que c'est le flex. Le test qui l'attrape : comparer la hauteur affichée de `.answer-list` à sa hauteur naturelle (la remesurer après avoir forcé `flexShrink = '0'`) — si l'écart n'est pas nul, des lignes sont en train de disparaître.

Sur cet écran, la carte de réponses est centrée verticalement mais avec une **bande de 44px réservée en bas** (`padding-bottom` sur `.answer-scroll`) : un centrage plein ne laisse que 40px sous elle sur un écran de 844 et les griffonnages n'ont plus où aller. Et `justify-content:safe center`, pas `center` — quand les catégories débordent, un centrage normal rogne le haut de la liste et le rend inatteignable au défilement.

Le **salon n'a aucune bande libre** : sa liste `PAPER_DOODLES.salon` est vide exprès, il ne garde que le papier réglé. Pour remesurer après un changement de mise en page, **mesurer en 2D et pas par bandes horizontales** : découper l'écran en cellules de 20px et marquer celles que recouvre un rectangle d'élément d'UI, puis imprimer la grille en ASCII. Une mesure par bandes déclare « occupée » une ligne traversée par un seul élément central et fait rater les bords libres — c'est comme ça que les flancs de la lettre, sur l'écran de manche, étaient passés inaperçus.

Un piège dans cette mesure : exclure les **conteneurs** du sélecteur. `.answer-scroll` occupe toute la hauteur restante même quand la carte de réponses est plus courte, et fait passer l'écran entier pour plein. Ne viser que ce qui est réellement peint.

Deux pièges rencontrés au détourage, si tu dois refaire l'extraction : la luminance doit être calculée **en flottant** (`0.299*r + ...` déborde en `int16` et rend des valeurs négatives), et le seuil doit être mesuré sur une bande de page **sans dessin** — `lum < 170` et `bleu - rouge > 25` attrape l'encre et 0 % des lignes du cahier. Trop haut, les lignes passent pour de l'encre, se connectent d'un bord à l'autre et fusionnent tous les dessins en un seul bloc pleine largeur.

Le réglage est le **fond du calque** et les griffonnages en sont les **enfants** : c'est ce qui met l'encre par-dessus les lignes. En les mettant tous les deux à plat (`::after` pour le réglage), les lignes se dessinaient sur les dessins.

Le texte posé à même le fond est protégé par un **halo de papier** (`text-shadow` en `var(--paper)` sur `.app-bar-title`, `.tagline`, `.section-heading`, `.lobby-name`, `.stat-value`…) plutôt que par un masque sur l'image : on protège le texte au lieu de cacher les dessins.

`bgForScreen(screen, phase)` choisit le jeu de griffonnages et `renderTop()` le pose en `data-bg` sur le shell. Le calque n'est reconstruit **que si l'écran change** : `renderTop()` tourne à chaque snapshot Firebase, inutile de réécrire onze `<span>` à chaque battement de cœur.

**Le calque est monté pendant le splash, jamais après.** Les masques sont des fichiers à part : sans rien, ils arrivent un par un et les griffonnages se posaient en ordre dispersé une fois l'écran de lancement parti. `preloadDoodles('accueil', markDoodlesReady)` les décode donc au boot, en parallèle de la police, et `maybeHideSplash()` exige `doodlesReady` en plus d'`appReady` — l'écran de lancement s'allonge le temps qu'il faut et le rideau se lève sur une page complète. Deux plafonds pour ne jamais bloquer : `DOODLE_WAIT_MS` (1,8 s) libère l'attente si un masque ne répond pas, et `SPLASH_MAX_MS` (3,5 s) reste le filet final — c'est pour ce cas-là que `hideSplash()` appelle quand même `markDoodlesReady()`, sinon le fond resterait invisible à vie. Les autres écrans sont préchargés 600 ms après (`preloadOtherDoodles`), ce qui rend le changement de fond en cours de partie instantané.

Ne remets pas de fondu échelonné (papier puis encre) : **le splash est opaque** (`background:var(--paper)`), donc un fondu joué dessous ne se voit pas, et ce qui dépasse après son effacement se lit comme un défaut de chargement. Le `transition:opacity .4s` qui reste sur `.paper-doodles` ne sert que de filet pour la sortie par plafond ; il est coupé en `prefers-reduced-motion`.

## Points d'attention connus

- **Pas de liseré en haut du shell.** `#app-shell::before` portait un dégradé brique→moutarde de 4px ; il a été retiré (avec sa règle d'arrondi dans le `@media (min-width:560px)`) pour laisser la page de cahier monter jusqu'au bord. Si tu remets un `::before` sur `#app-shell`, pense à cet arrondi sur grand écran, sinon il déborde des coins.

- **Onglets en arrière-plan** : les navigateurs mobiles peuvent throttle `setInterval` quand l'onglet n'est pas au premier plan, ce qui peut retarder ou empêcher l'auto-soumission des réponses. Double filet de sécurité déjà en place : un listener `visibilitychange` relance `tick()` au retour au premier plan, ET `submitMyAnswers()` est aussi appelé directement depuis le callback du listener Firebase temps réel (`roomSnapshotHandler`), pas uniquement depuis le timer local. Si tu touches à cette logique, garde les deux déclencheurs.
- **Identité joueur** = `localStorage` par appareil/navigateur (`pb_identity_<pin>`), pas d'authentification réelle. Changer de navigateur/appareil = nouveau joueur dans la même partie.
- **Le profil est 100% local** (`pb_profile`), rien ne part dans Firebase. Une liste d'amis a été tentée puis retirée : sans présence ni invitation possible (il faudrait publier un annuaire de joueurs et un identifiant stable — aujourd'hui la clé d'un joueur est dérivée de son prénom via `keyFor`, donc deux "Marc" se confondent), elle n'aurait été qu'une liste décorative. Si la question revient, c'est ce couple identifiant stable + nœud `users/` qu'il faut trancher d'abord.
- **Le profil court-circuite les écrans de saisie** : avec un `pb_profile`, "Créer" et "Rejoindre" entrent directement en partie (`goCreate()` / `goJoin()` via `useProfileDrafts()`). Les écrans `create`/`join` ne servent plus qu'au premier lancement — et de repli quand le prénom est déjà pris dans la partie visée, cas où `handleConfirmJoin()` les ouvre pour laisser corriger.
- **Champs de saisie à 16px minimum** : en dessous, Safari iOS zoome automatiquement l'écran au focus. C'est la raison du `font-size:16px` en dur sur le textarea des catégories.
- **Les champs de réponse coupent la correction automatique** (`autocorrect="off" spellcheck="false"`, en plus d'`autocomplete="off"`) : sur un jeu de mots, iOS réécrivait les noms propres et les mots rares entre la frappe et l'envoi — le joueur voyait partir autre chose que ce qu'il avait tapé.
- **`setBusy()` et les boutons statiques du shell** : `#menu-btn` et `#back-btn` ne sont jamais recréés (`renderTop()` ne touche que leur `hidden` et leur `innerHTML`), contrairement aux boutons de `#content` et `#bottom-bar`. `setBusy` ne faisait que désactiver sans jamais réactiver : les seconds s'en sortaient par le re-rendu, les premiers restaient désactivés à vie dès la première action réseau — le bouton réglages ne répondait plus de toute la partie. Il mémorise désormais ce qu'il a désactivé pour le réactiver ensuite ; ne le remplace pas par un `b.disabled = isBusy` aveugle, ça réactiverait les boutons volontairement désactivés par le rendu ("Envoyé, en attente…").
- **Le départ de l'hôte est rattrapé** par la reprise de rôle décrite plus haut (immédiate s'il clique « Quitter », sous 90s s'il ferme son onglet). Il reste un cas non couvert : si **tous** les clients sont en arrière-plan ou hors ligne, personne n'est éligible et la partie attend — elle repart dès que quelqu'un revient au premier plan.

## Comment tester

Pas de suite de tests dans le repo, mais deux techniques qui marchent bien et qui ont servi à valider le scoring, la synchro et les animations :

- **Fonctions pures** (`scoreRound`, `sameWord`, `levenshtein`, `pickLetter`, `recentLetters`…) : extraire le `<script>` inline (commande plus haut), découper la fonction voulue par comptage d'accolades, `eval` dans un script Node jetable, et vérifier les cas limites. Ça évite d'ouvrir un navigateur pour un changement de règle de score.
- **App complète** : navigateur headless (Playwright/Chromium) avec `firebase` et `QRCode` remplacés par des bouchons injectés avant le chargement de la page, puis appel direct de `roomSnapshotHandler({ val: … })` pour pousser des états de partie arbitraires. Deux onglets avec le même objet de partie partagé permettent de rejouer une vraie partie à plusieurs et de vérifier la synchronisation. Pour les animations, échantillonner `getComputedStyle(el).opacity` à des instants précis est plus fiable qu'une capture d'écran.
- Et dans tous les cas, un passage manuel sur le déploiement réel avec plusieurs onglets/navigateurs/profils : rien ne remplace le vrai Firebase et le vrai Safari iOS.

## Historique (ordre de construction, pour comprendre le "pourquoi" de certains choix)

1. V1 : jeu mono-partie basé sur l'API artifact de claude.ai (abandonnée, remplacée par Firebase pour un hébergement indépendant sur Vercel)
2. Migration Firebase Realtime Database + déploiement Vercel/GitHub
3. Refonte UI "vraie app mobile" (app bar, bottom sheet de réglages, anneau de minuteur SVG, avatars)
4. Favicon + PWA (`manifest.json`, add-to-homescreen)
5. Vrai système de lobby multi-parties (code PIN + QR, avatars emoji+couleur), puis simplifié : plus de code PIN saisi à la main ni de distinction publique/privée — uniquement QR + liste "Parties en cours" sur l'accueil
6. Manche "rapide" (fin déclenchée par le premier "fini" + 5s de grâce), écran de vote catégorie par catégorie, scoring 2 pts / 1 pt
7. Nettoyage auto des parties inactives (3 min)
8. Lancement réservé à l'hôte, nombre de manches configurable, écran de classement/podium entre les manches et en fin de partie
9. Refonte complète de l'identité visuelle ("carnet de scores" papier + tuiles tamponnées, palette brique/moutarde, typographie Fraunces/Inter/JetBrains Mono, icônes SVG à la place des emojis fonctionnels) — comportement JS et modèle de données Firebase inchangés, seuls les `render*()` et le `<style>` ont été réécrits
10. Icônes PWA/favicon régénérées dans la nouvelle DA, correction du zoom iOS sur les champs, et écran de manche réorganisé (lettre à gauche du minuteur, en-tête fixe et liste de réponses seule à défiler)
11. Podium animé : roulement de tambour, révélation par paliers (3e, 2e, puis 1er sous un projecteur avec confettis, enfin le reste du classement), rangs en or/argent/bronze
12. Écran de lancement animé (tuiles PETIT BAC qui tombent) servant aussi d'écran de chargement, recalé au pixel près sur le logo de l'accueil ; chargeur maison en mini-tuiles pour les autres attentes
13. Refonte de l'accueil : liste des parties déplacée dans une popup derrière un bouton "Rejoindre", hero aéré, bloc de stats locales, marque redessinée en crayon plein
14. Règles de jeu : validation par défaut (on ne vote que pour refuser), une lettre = 0, doublons détectés à la faute de frappe près, écrans de validation synchronisés entre joueurs, et mémoire des 13 dernières lettres tirées

15. Banque de catégories tirées au sort à la création d'une partie (étalées sur 7 familles), bouton de re-tirage dans le salon, et picto devant chaque catégorie dans la salle d'attente

16. Icônes PWA/favicon refaites en « PB + crayon » (crayon retiré au 16px, variante maskable rentrée pour Android)

17. Reprise automatique du rôle d'hôte quand il s'en va (battement de cœur `seen` + transaction sur `hostKey`), et validation réservée à l'hôte

18. Vote à la majorité pour exclure une réponse (et interdiction de voter contre son propre mot), et nouvelles catégories à chaque retour au salon

19. Articles en tête de réponse ignorés, et abréviations traitées comme des doublons (« foot » = « football »), pour la détection des doublons et l'alerte sur l'initiale

20. Refonte de la salle d'attente : joueurs en tête avec case « + », QR déplacé dans une feuille avec partage de lien, règles du match visibles, catégories en liste alignée

21. Revue de la banque : catégories trop ouvertes retirées (« Ce qui coûte cher », « Objet », « Nourriture »…), remplacées par des catégories bornées et vérifiables

22. Fonds « page de cahier » griffonnée au bic — d'abord huit images pleine page (une par écran et par thème), approche **abandonnée** : une compo figée ignore la mise en page, les dessins tombaient sous le châssis et le centre restait vide

23. Fonds refaits en deux morceaux : réglage tracé en CSS, et 26 griffonnages détourés des images (`bg/d/`, masques recolorés par thème) placés à la main écran par écran, dans les bandes que l'UI laisse libres

24. Calque de fond monté **pendant** l'écran de lancement (préchargement des masques, le splash attend) au lieu d'apparaître après en ordre dispersé

25. Salon : bouton « Quitter » dans la barre du haut avec confirmation, bouton « Paramètres » retiré (les pastilles de règles ouvrent la même feuille), pastilles de règles sur fond plein

26. Griffonnages ancrés sur les safe areas (`calc(var(--safe-top) + …)`), correction automatique coupée dans les champs de réponse, et liseré dégradé du bandeau retiré

## Pistes non traitées

- Resserrer les règles Firebase avant l'expiration du mode test
- Pas de reconnexion réseau explicite au-delà du comportement natif du SDK Firebase
