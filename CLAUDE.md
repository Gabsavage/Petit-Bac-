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
  roundDurationMs, maxRounds    maxRounds: 0 = illimité, sinon 3/5/10
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

**Tirage des lettres** : `recentLetters` est la mémoire glissante des 13 dernières lettres, gardée à la racine de la partie et **pas** déduite de `history` — celui-ci est effacé par "Rejouer" et par la remise à zéro des scores, une lettre pouvait donc revenir juste après. Sur 19 lettres au pool, il en reste toujours 6 tirables. Les parties créées avant ce champ retombent sur l'historique (`recentLetters()`).

**Tirage des catégories** : à la création d'une partie, `attemptCreatePin` appelle `randomCategories(CATEGORIES_PER_GAME)` au lieu de `DEFAULT_CATEGORIES` (qui ne sert plus que de repli pour l'état initial et les parties sans champ `categories`). `CATEGORY_BANK` est un tableau de **familles** (classiques / lieux / culture / nourriture / quotidien / nature / fun) ; le tirage mélange chaque famille puis fait un tour de table — une catégorie par famille, puis on recommence. Sans ça, un tirage à plat sortait régulièrement quatre catégories « nourriture ». Conséquence à garder en tête : sur 7 familles et 8 tirages, **deux catégories de la même famille peuvent se retrouver ensemble**, donc pas de quasi-doublons dans la banque (« Animal » et « Animal sauvage » ne doivent pas coexister), et pas deux fois le même emoji sur des catégories qui peuvent tomber ensemble.

Le tirage se fait **à la création seulement**, jamais au début d'une manche : les catégories corrigées à la main dans les réglages seraient écrasées à la manche 2. Pour en changer, l'édition du textarea (`handleSaveCategories`) et le bouton « Tirer d'autres catégories » (`handleShuffleCategories`) restent dispo dans le salon. `handleShuffleCategories` écrit dans Firebase **et** dans le textarea ouvert : la feuille de réglages n'est pas re-rendue par un snapshot, sans ça elle continuerait d'afficher l'ancienne liste.

`categoryEmoji(name)` cherche un mot-clé dans le nom **normalisé** (`CATEGORY_EMOJIS`, du plus spécifique au plus générique) plutôt qu'une table nom -> emoji : les catégories peuvent être réécrites librement, une catégorie inconnue retombe sur ✏️. L'ordre des entrées compte (« dans une ville » avant « ville », « cuisine » avant « objet ») — si tu ajoutes un mot-clé, vérifie qu'il n'est pas un sous-mot d'une entrée plus spécifique placée après lui.

**Reprise du rôle d'hôte** : il n'y a pas de système de présence Firebase (`onDisconnect`) — chaque client écrit `players/{key}/seen` toutes les 10s depuis `tick()` (`sendHeartbeat`), et un hôte qu'on n'a plus vu depuis `HOST_GONE_MS` (90s) est repris par un autre joueur (`maybeAdoptHost`). Quatre choses à ne pas casser :

- Le battement de cœur n'écrit **que** `players/{key}/seen`, jamais `lastActivity` : sinon un onglet oublié dans un coin empêcherait à jamais le nettoyage automatique des parties inactives.
- On ne promeut que si l'hôte a **déjà un `seen`**. Un onglet resté sur une version antérieure du code n'en écrit jamais : sans ce test, il se ferait destituer alors qu'il joue encore. Pas de `seen` = on ne touche à rien.
- Le successeur est la **plus petite clé** parmi les joueurs vus récemment (`pickNewHost`) : c'est arbitraire mais déterministe, donc tous les clients désignent le même et on ne se repasse pas le rôle en boucle. L'écriture passe par une **transaction** sur `hostKey` (`promoteHost`), qui abandonne si quelqu'un a déjà repris la main.
- 90s paraît long, c'est volontaire : `seen` est écrit depuis `tick()`, or les mobiles throttlent les timers d'un onglet en arrière-plan. À 45s, un hôte qui répond juste à un message se faisait destituer. Le départ volontaire, lui, passe la main immédiatement (`handleLeaveRoom`) — les 90s ne concernent que les départs brutaux (onglet fermé, réseau coupé).

`seen` est aussi posé à la création et à l'arrivée dans une partie, sinon un joueur qui vient d'entrer ne serait pas éligible comme successeur pendant 10s, et une reconnexion (qui réécrit l'objet joueur en entier) l'effacerait.

**Scoring** (`scoreRound`) : une réponse est **valide par défaut**, l'écran de validation ne permet que de refuser — un seul refus suffit à annuler la réponse. La comparaison `no > yes` est conservée pour rescorer correctement les manches archivées avec l'ancien écran à deux boutons. Une réponse d'une seule lettre vaut 0 sans vote. Les doublons sont détectés à la faute de frappe près (`sameWord`, distance de Levenshtein : 2 fautes à partir de 7 lettres, 1 à partir de 4, aucune en dessous) pour que "Hongrie"/"Hongrir" donnent 1 pt chacun.

**Piège important, ne pas régresser dessus** : les réponses et les votes sont indexés par **position dans le tableau `categories`** (`"0"`, `"1"`, `"2"`…), jamais par le nom de la catégorie. C'est volontaire : Firebase interdit `.` `#` `$` `[` `]` `/` dans les clés d'un objet, et une catégorie comme "Pays / Villes" cassait silencieusement l'écriture (`.set()` échouait, d'où un bug déjà vécu où aucune réponse n'apparaissait). Si tu ajoutes une nouvelle donnée indexée par catégorie, indexe-la par position (`idx` dans les boucles `categories.forEach(function(cat, idx){...})`), jamais par `cat` directement.

## Machine à états (`getPhase`)

`lobby → countdown (3s, revealAt) → playing → reveal (review + vote) → countdown suivant`

- `reveal` recouvre deux vues : la validation catégorie par catégorie, et l'écran de classement/podium. Les deux sont **synchronisées via Firebase** (`currentRound/reviewIndex` et `showLeaderboard`, lus par `reviewIndexOf()` / `showingLeaderboard()`) : tout le monde valide la même catégorie en même temps. C'était local auparavant — si tu repasses des bouts en local, tu casses la synchro.
- **Seul l'hôte fait avancer la validation** (`handleReviewStep`, `handleShowLeaderboard`, même garde `myKey !== state.hostKey` que `handleStartRound`) : à plusieurs, n'importe qui pouvait enchaîner sur la catégorie suivante pendant que les autres lisaient encore. Le vote, lui, reste ouvert à tout le monde — c'est tout l'intérêt de l'écran. La garde est à la fois dans le rendu (les non-hôtes voient une ligne d'attente à la place du bouton) et dans le handler : ne retire pas la seconde en gardant la première.
- Le "5s de grâce" après le premier "J'ai fini" est géré par `finishedAt` + `FINISH_GRACE_MS`, voir `roundEndsAt()` — la fin de manche réelle est `min(endTime, finishedAt + 5000)`.
- Le podium est animé par paliers (3e, 2e, 1er + projecteur + confettis, puis les 4e et suivants). Seule la *lecture* de l'animation est locale (`podiumRevealed`, `podiumSettled`, `confettiFired`) ; son point de départ vient de `leaderboardAt`, partagé, pour que tout le monde voie le podium au même moment. `podiumSettled` retire les classes d'animation une fois la séquence jouée : sans lui, le moindre snapshot Firebase rejouerait toute la révélation.
- **État local propre à une manche** : `localAnswers`, `locked` et `hasAutoSubmitted` sont resynchronisés par `syncRoundLocalState()` à chaque changement de numéro de manche. Ne les remets pas à zéro uniquement à l'entrée dans la partie : c'était le cas avant, d'où des réponses de la manche précédente pré-remplies, grisées pour qui avait fini premier, et surtout plus aucun envoi de réponses dès la 2e manche.

## Points d'attention connus

- **Onglets en arrière-plan** : les navigateurs mobiles peuvent throttle `setInterval` quand l'onglet n'est pas au premier plan, ce qui peut retarder ou empêcher l'auto-soumission des réponses. Double filet de sécurité déjà en place : un listener `visibilitychange` relance `tick()` au retour au premier plan, ET `submitMyAnswers()` est aussi appelé directement depuis le callback du listener Firebase temps réel (`roomSnapshotHandler`), pas uniquement depuis le timer local. Si tu touches à cette logique, garde les deux déclencheurs.
- **Identité joueur** = `localStorage` par appareil/navigateur (`pb_identity_<pin>`), pas d'authentification réelle. Changer de navigateur/appareil = nouveau joueur dans la même partie.
- **Le profil est 100% local** (`pb_profile`), rien ne part dans Firebase. Une liste d'amis a été tentée puis retirée : sans présence ni invitation possible (il faudrait publier un annuaire de joueurs et un identifiant stable — aujourd'hui la clé d'un joueur est dérivée de son prénom via `keyFor`, donc deux "Marc" se confondent), elle n'aurait été qu'une liste décorative. Si la question revient, c'est ce couple identifiant stable + nœud `users/` qu'il faut trancher d'abord.
- **Le profil court-circuite les écrans de saisie** : avec un `pb_profile`, "Créer" et "Rejoindre" entrent directement en partie (`goCreate()` / `goJoin()` via `useProfileDrafts()`). Les écrans `create`/`join` ne servent plus qu'au premier lancement — et de repli quand le prénom est déjà pris dans la partie visée, cas où `handleConfirmJoin()` les ouvre pour laisser corriger.
- **Champs de saisie à 16px minimum** : en dessous, Safari iOS zoome automatiquement l'écran au focus. C'est la raison du `font-size:16px` en dur sur le textarea des catégories.
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

## Pistes non traitées

- Resserrer les règles Firebase avant l'expiration du mode test
- Pas de reconnexion réseau explicite au-delà du comportement natif du SDK Firebase
- Un seul refus suffit à annuler une réponse : suffisant entre amis, mais rien n'empêche un joueur d'invalider tout le monde
