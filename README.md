# Petit Bac

Jeu du Petit Bac multijoueur en temps réel, jouable depuis un navigateur mobile — côte à côte ou à distance, sans papier. Une lettre tombe, chacun remplit ses catégories dans le temps imparti, puis le groupe valide les réponses avant le podium.

## Stack

- HTML/CSS/JS vanilla, **un seul fichier** (`index.html`), aucun build step, aucune dépendance npm
- **Firebase Realtime Database** pour l'état de jeu en temps réel (pas de backend custom)
- Hébergé sur **Vercel**, déployé automatiquement à chaque push sur `main`
- QR code généré côté client via [qrcodejs](https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js) (cdnjs)
- Installable en PWA (`manifest.json` + icônes)

## Direction artistique

Identité "carnet de scores" : fond papier crème avec une grille de points, encre foncée, un accent brique réservé aux actions primaires et un accent moutarde pour les moments ludiques. Motif récurrent de "tuile tamponnée" (bordure encre + ombre portée dure) sur le logo, les lettres, les avatars et les boutons principaux.

Trois polices avec un rôle chacune : **Fraunces** pour les titres et les gros chiffres, **Inter** pour l'interface, **JetBrains Mono** pour tout ce qui est chiffré (minuteur, scores). Les icônes sont des SVG inline ; les emojis restent réservés aux avatars des joueurs.

Thème clair et thème sombre, suivant le réglage du système.

## Développement local

Pas de build, pas d'installation. Ouvre `index.html` directement dans un navigateur, ou sers-le avec n'importe quel serveur statique :

```bash
python3 -m http.server 8000
```

La base Firebase utilisée est celle de prod (projet `petit-bac-c24bf`) — teste en créant tes propres parties, ça n'affecte personne d'autre. Pour simuler plusieurs joueurs, ouvre plusieurs onglets dans des profils ou des navigateurs différents (l'identité est stockée par navigateur).

## Déploiement

- Repo GitHub : `Gabsavage/Petit-Bac-`
- Connecté à Vercel : chaque push sur `main` redéploie automatiquement en ~20-30s
- Rien à configurer côté Vercel pour changer le code, juste push

## Firebase

- Projet : `petit-bac-c24bf`, base **Realtime Database** (pas Firestore)
- Règles actuellement en **mode test** (lecture/écriture ouverte à qui a la config) — Firebase désactive ça après ~30 jours ; il faudra resserrer les règles ou renouveler le mode test avant l'échéance
- La config Firebase (apiKey, etc.) est directement dans `index.html` — c'est normal pour une web app Firebase : l'apiKey web n'est pas un secret, la sécurité passe par les Rules, pas par la confidentialité de la clé

## Déroulé d'une partie

1. **Accueil** — écran de lancement animé (les tuiles PETIT BAC tombent une à une), puis deux actions : créer une partie, ou rejoindre depuis la liste des parties en cours. Un bloc de statistiques locales (manches jouées, record, dernière lettre) apparaît une fois qu'on a joué. L'avatar en haut à droite ouvre le profil.
2. **Profil** — avatar (emoji + couleur) et prénom configurés une fois pour toutes, depuis l'avatar du bandeau : ensuite, créer ou rejoindre une partie se fait en un geste, sans repasser par un formulaire. Le profil est stocké sur l'appareil uniquement, rien n'est publié.
3. **Création** — la partie obtient un code à 4 chiffres, qui sert de clé Firebase mais n'est jamais affiché : on rejoint par QR code, par lien (`?pin=XXXX`) ou depuis la liste.
4. **Salle d'attente** — QR code à scanner, joueurs présents, et les huit catégories de la partie, chacune avec son picto. Elles sont **tirées au sort** à la création, dans une banque groupée par famille (classiques, lieux, culture, nourriture, quotidien, nature, fun) pour que deux parties ne se ressemblent pas. Seul l'hôte lance la manche. Les réglages (catégories — à réécrire à la main ou à retirer au sort —, durée 60/90/120s, nombre de manches 3/5/10/∞) sont accessibles juste au-dessus du bouton de lancement.
5. **Manche** — décompte de 3s, puis la lettre s'affiche à gauche du minuteur. Le premier qui clique "J'ai fini" déclenche 5s de grâce pour les autres, après quoi la manche se termine pour tout le monde.
6. **Validation** — catégorie par catégorie, synchronisée : tout le monde voit la même au même moment, et n'importe qui fait avancer le groupe. Les réponses comptent par défaut, un bouton permet de refuser celles qui ne vont pas.
7. **Podium** — roulement de tambour, puis révélation par paliers : 3e, 2e, puis la 1re place sous un projecteur avec des confettis, et enfin le reste du classement.

## Règles de score

- **2 pts** pour une réponse que personne d'autre n'a, **1 pt** si elle est en double, **0** sinon
- Les réponses sont **valides par défaut** — on ne vote que pour refuser, et un seul refus suffit à annuler une réponse
- Une réponse d'**une seule lettre** vaut 0 automatiquement, sans passer par un vote
- Les **fautes de frappe** comptent comme des doublons : "Hongrie" et "Hongrir" donnent 1 pt chacun et non 2 réponses uniques (distance de Levenshtein, seuil proportionnel à la longueur du mot)
- Une lettre déjà tirée ne peut pas revenir avant **13 tirages**, y compris après un "Rejouer"
- Les catégories sont tirées au sort à la création de la partie, avec au plus deux catégories de la même famille — et restent modifiables dans les réglages du salon avant le lancement

## Autres comportements

- Nettoyage automatique : une partie sans activité depuis 3 min se ferme toute seule
- Les réponses sont sauvegardées en cours de frappe : recharger l'onglet en pleine manche ne les perd pas
- Auto-soumission des réponses à la fin du temps, même si l'onglet est passé en arrière-plan

## Limitations connues

- Pas de vraie authentification : l'identité d'un joueur est stockée en `localStorage` par appareil/navigateur — changer de navigateur = nouveau joueur, et le profil ne suit pas
- Pas de liste d'amis ni de présence : voir qui joue et le rejoindre demanderait un annuaire de joueurs publié et un identifiant stable
- Pas de suite de tests dans le repo (voir `CLAUDE.md` pour la façon de tester les fonctions pures et l'app en navigateur headless)
- Règles Firebase en mode test (voir plus haut)
- Pas de gestion de reconnexion réseau au-delà de ce que fait nativement le SDK Firebase
- Si l'hôte quitte définitivement une partie en cours, plus personne ne peut lancer la manche suivante

Pour le contexte de développement détaillé (conventions de code, modèle de données, pièges connus), voir [`CLAUDE.md`](./CLAUDE.md).
