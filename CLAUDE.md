# CLAUDE.md

Contexte projet pour Claude Code. Lis ce fichier avant de faire des changements — il documente des choix et des pièges qui ne sont pas évidents en lisant juste le code.

## C'est quoi

App web "Petit Bac" multijoueur en temps réel. **Un seul fichier** `index.html` autonome (HTML + CSS + JS inline), pas de build, pas de framework, pas de `package.json`. Déployé sur Vercel, connecté à ce repo GitHub — un push sur `main` redéploie automatiquement en ~20-30s. Les autres fichiers à la racine (`manifest.json`, `favicon*.png/ico`, `apple-touch-icon.png`, `icon-*.png`) sont les assets PWA, référencés en chemins absolus (`/favicon.ico`, etc.) depuis `index.html`.

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
- Police **Baloo 2** (Google Fonts) pour les titres/gros éléments, police système pour le reste.
- Projet Firebase : `petit-bac-c24bf`. Config déjà dans `index.html` (l'apiKey Firebase web n'est pas un secret, normal qu'il soit visible côté client — la sécurité passe par les Rules).
- **Règles Firebase en mode test** (lecture/écriture ouverte). Ça expire ~30 jours après activation — à surveiller, sinon l'app cesse de fonctionner sans prévenir côté code.

## Modèle de données Firebase

```
games/{pin}/                    pin = code à 4 chiffres, sert aussi de clé Firebase
  hostName, hostKey             hostKey = clé du créateur, seul à pouvoir lancer une manche (handleStartRound)
  categories: [string, ...]
  roundDurationMs, maxRounds    maxRounds: 0 = illimité, sinon 3/5/10
  createdAt, lastActivity       lastActivity sert au nettoyage auto (INACTIVITY_MS = 3 min)
  players/{key}: { name, emoji, color }
  currentRound: {
    number, letter, revealAt, endTime,
    finishedAt, finishedBy,     posé par le 1er joueur qui clique "J'ai fini" -> 5s de grâce pour les autres
    answers/{playerKey}: { "0": "mot", "1": "mot", ... },   // clé = INDEX de la catégorie, pas son nom
    votes/{catIndex}/{targetPlayerKey}/{voterKey}: true | false
  }
  history/{roundNumber}: <ancien currentRound archivé tel quel>
```

**Piège important, ne pas régresser dessus** : les réponses et les votes sont indexés par **position dans le tableau `categories`** (`"0"`, `"1"`, `"2"`…), jamais par le nom de la catégorie. C'est volontaire : Firebase interdit `.` `#` `$` `[` `]` `/` dans les clés d'un objet, et une catégorie comme "Pays / Villes" cassait silencieusement l'écriture (`.set()` échouait, d'où un bug déjà vécu où aucune réponse n'apparaissait). Si tu ajoutes une nouvelle donnée indexée par catégorie, indexe-la par position (`idx` dans les boucles `categories.forEach(function(cat, idx){...})`), jamais par `cat` directement.

## Machine à états (`getPhase`)

`lobby → countdown (3s, revealAt) → playing → reveal (review + vote) → countdown suivant`

- `reveal` recouvre deux vues côté client : la review catégorie par catégorie, et l'écran de classement/podium. La bascule entre les deux est un booléen **local, non synchronisé entre joueurs** (`viewingLeaderboard`) — chaque joueur navigue à son rythme, seule l'action "Manche suivante" (déclenchée par n'importe qui) fait réellement avancer l'état Firebase partagé.
- Le "5s de grâce" après le premier "J'ai fini" est géré par `finishedAt` + `FINISH_GRACE_MS`, voir `roundEndsAt()` — la fin de manche réelle est `min(endTime, finishedAt + 5000)`.
- `reviewCatIndex` (position dans la review) est aussi local, remis à 0 à chaque nouvelle entrée en phase `reveal`.

## Points d'attention connus

- **Onglets en arrière-plan** : les navigateurs mobiles peuvent throttle `setInterval` quand l'onglet n'est pas au premier plan, ce qui peut retarder ou empêcher l'auto-soumission des réponses. Double filet de sécurité déjà en place : un listener `visibilitychange` relance `tick()` au retour au premier plan, ET `submitMyAnswers()` est aussi appelé directement depuis le callback du listener Firebase temps réel (`roomSnapshotHandler`), pas uniquement depuis le timer local. Si tu touches à cette logique, garde les deux déclencheurs.
- **Identité joueur** = `localStorage` par appareil/navigateur (`pb_identity_<pin>`), pas d'authentification réelle. Changer de navigateur/appareil = nouveau joueur dans la même partie.
- Pas de tests automatisés — teste manuellement avec plusieurs onglets/navigateurs/profils pour simuler plusieurs joueurs.
- Pas de gestion explicite du cas où l'hôte quitte définitivement une partie en cours (plus personne ne peut lancer la manche suivante depuis le lobby si l'hôte n'est pas revenu).

## Historique (ordre de construction, pour comprendre le "pourquoi" de certains choix)

1. V1 : jeu mono-partie basé sur l'API artifact de claude.ai (abandonnée, remplacée par Firebase pour un hébergement indépendant sur Vercel)
2. Migration Firebase Realtime Database + déploiement Vercel/GitHub
3. Refonte UI "vraie app mobile" (app bar, bottom sheet de réglages, anneau de minuteur SVG, avatars)
4. Favicon + PWA (`manifest.json`, add-to-homescreen)
5. Vrai système de lobby multi-parties (code PIN + QR, avatars emoji+couleur), puis simplifié : plus de code PIN saisi à la main ni de distinction publique/privée — uniquement QR + liste "Parties en cours" sur l'accueil
6. Manche "rapide" (fin déclenchée par le premier "fini" + 5s de grâce), écran de vote catégorie par catégorie, scoring 2 pts / 1 pt
7. Nettoyage auto des parties inactives (3 min)
8. Lancement réservé à l'hôte, nombre de manches configurable, écran de classement/podium entre les manches et en fin de partie

## Pistes non traitées

- Resserrer les règles Firebase avant l'expiration du mode test
- Pas de reconnexion réseau explicite au-delà du comportement natif du SDK Firebase
- Pas de gestion du départ définitif de l'hôte
