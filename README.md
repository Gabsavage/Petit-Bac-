# 🎲 Petit Bac

Jeu du Petit Bac multijoueur en temps réel, jouable entre potes depuis un navigateur mobile. Une lettre tombe, chacun tape ses mots dans un temps limité, puis le groupe vote pour valider ou refuser chaque réponse.

## Stack

- HTML/CSS/JS vanilla, **un seul fichier** (`index.html`), aucun build step, aucune dépendance npm
- **Firebase Realtime Database** pour l'état de jeu en temps réel (pas de backend custom)
- Hébergé sur **Vercel**, déployé automatiquement à chaque push sur `main`
- QR code généré côté client via [qrcodejs](https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js) (cdnjs)
- Police **Baloo 2** (Google Fonts) pour l'identité visuelle ; le reste utilise les polices système

## Développement local

Pas de build, pas d'installation. Ouvre `index.html` directement dans un navigateur, ou sers-le avec n'importe quel serveur statique :

```bash
python3 -m http.server 8000
```

La base Firebase utilisée est celle de prod (projet `petit-bac-c24bf`) — teste en créant tes propres parties, ça n'affecte personne d'autre.

## Déploiement

- Repo GitHub : `Gabsavage/Petit-Bac-`
- Connecté à Vercel : chaque push sur `main` redéploie automatiquement en ~20-30s
- Rien à configurer côté Vercel pour changer le code, juste push

## Firebase

- Projet : `petit-bac-c24bf`, base **Realtime Database** (pas Firestore)
- Règles actuellement en **mode test** (lecture/écriture ouverte à qui a la config) — Firebase désactive ça après ~30 jours ; il faudra resserrer les règles ou renouveler le mode test avant l'échéance
- La config Firebase (apiKey, etc.) est directement dans `index.html` — c'est normal pour une web app Firebase : l'apiKey web n'est pas un secret, la sécurité passe par les Rules, pas par la confidentialité de la clé

## Fonctionnalités actuelles

- Créer une partie (avatar emoji + couleur, prénom) → génère un code à 4 chiffres
- Rejoindre via QR code, lien direct (`?pin=XXXX`), ou depuis la liste "Parties en cours" sur l'accueil
- Seul l'hôte (créateur) peut lancer une manche
- Réglages avant de jouer : catégories personnalisables, durée de manche (60/90/120s), nombre de manches (3/5/10/∞)
- Manche "rapide" : le premier joueur qui clique "J'ai fini" déclenche 5s de grâce pour les autres avant fin de manche automatique
- Écran de review catégorie par catégorie avec vote ✅/❌ collectif sur chaque réponse
- Scoring : 2 pts réponse unique validée, 1 pt en double validée, 0 sinon
- Classement avec podium entre chaque manche, écran de fin de partie avec vainqueur
- Nettoyage automatique : une partie sans activité depuis 3 min se ferme toute seule

## Limitations connues

- Pas de vraie authentification : l'identité d'un joueur est stockée en `localStorage` par appareil/navigateur — changer de navigateur = nouveau joueur
- Pas de tests automatisés
- Règles Firebase en mode test (voir plus haut)
- Pas de gestion de reconnexion réseau au-delà de ce que fait nativement le SDK Firebase

Pour le contexte de développement détaillé (conventions de code, modèle de données, pièges connus), voir [`CLAUDE.md`](./CLAUDE.md).
