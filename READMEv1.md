# Morpion en ligne — 2 joueurs (P2P)

Jeu de morpion pour deux joueurs, en **un seul fichier HTML** sans backend ni installation.
Les deux navigateurs se connectent **directement l'un à l'autre** en WebRTC via [PeerJS](https://peerjs.com/).

Charte graphique reprise du *Jeu des paires* (bleus `#38B6FF` / `#C2F6FF`, barre de configuration,
barre de statistiques, panneau latéral, modales).

---

## Sommaire

- [Fonctionnalités](#fonctionnalités)
- [Mise en ligne](#mise-en-ligne)
- [Utilisation](#utilisation)
- [Règles du jeu](#règles-du-jeu)
- [Architecture technique](#architecture-technique)
- [Protocole réseau](#protocole-réseau)
- [Personnalisation](#personnalisation)
- [Limites connues](#limites-connues)
- [Compatibilité](#compatibilité)
- [Licence](#licence)

---

## Fonctionnalités

| Domaine | Détail |
|---|---|
| Modes de jeu | **En ligne** (2 postes, connexion P2P) ou **local** (2 joueurs sur le même écran) |
| Grille | **3 × 3** ou **4 × 4** |
| Objectif | Aligner **3** ou **4** symboles (limité à la taille de la grille) |
| Symboles | Classique ✕ ◯, Cœurs, Animaux, Alimentation, Formes |
| Connexion | Code de salle à **5 caractères** + lien d'invitation `?room=CODE` |
| Suivi de partie | Chronomètre, joueur courant, coups joués, score cumulé des manches, matchs nuls |
| Manches | Score conservé d'une manche à l'autre, **alternance automatique** du joueur qui commence |
| Discussion | Chat texte + 5 réactions rapides, messages système de connexion/déconnexion |
| Confort | Aperçu fantôme au survol, mise en évidence du dernier coup, surlignage de l'alignement gagnant, historique des coups (`B2`), latence affichée en temps réel |

---

## Mise en ligne

Le jeu est un fichier statique : n'importe quel hébergement **HTTPS** convient.

### GitHub Pages

```bash
git init
cp 08092026_2124_MorpionEnLigne.html index.html
git add index.html
git commit -m "Morpion en ligne P2P"
git branch -M main
git remote add origin https://github.com/<utilisateur>/<depot>.git
git push -u origin main
```

Puis dans le dépôt : **Settings → Pages → Source: Deploy from a branch → main / (root)**.
Le jeu est disponible sous quelques minutes sur `https://<utilisateur>.github.io/<depot>/`.

### Netlify / Vercel / Cloudflare Pages

Renommer le fichier en `index.html` et le glisser-déposer dans l'interface de déploiement.
Aucune commande de build, aucun répertoire de sortie à configurer.

### Test en local

```bash
python3 -m http.server 8000
# puis http://localhost:8000/index.html
```

> ⚠️ Ouvrir le fichier par un double-clic (`file://`) fonctionne pour le **mode local**,
> mais WebRTC est bloqué par certains navigateurs dans ce contexte : le **mode en ligne
> exige HTTPS** (ou `http://localhost`).

---

## Utilisation

1. **Hôte** — ouvre la page, saisit son pseudo, clique sur **Créer une partie**.
   Un code de 5 caractères et un lien d'invitation s'affichent.
2. **Invité** — ouvre le lien reçu (ou clique sur **Rejoindre une partie** et saisit le code),
   saisit son pseudo, puis **Se connecter**.
3. Le voyant du bandeau réseau passe au **vert**. L'hôte choisit la grille, l'alignement et les
   symboles, puis clique sur **▶ Lancer la manche**.
4. Chacun joue à son tour ; la barre de statistiques indique en permanence à qui c'est.
5. En fin de manche, l'hôte lance la suivante ; l'invité peut envoyer une **demande de revanche**.

Rôles : l'**hôte joue le premier symbole** et pilote les réglages, l'**invité joue le second**
et reçoit automatiquement toute la configuration.

### Raccourcis

| Touche | Effet |
|---|---|
| `Échap` | Ferme la modale ouverte |
| `Entrée` (champ chat) | Envoie le message |
| `Entrée` (champ code) | Lance la connexion |

---

## Règles du jeu

- Les joueurs déposent leur symbole tour à tour sur une case libre.
- Le premier qui aligne le nombre requis de symboles en **ligne**, **colonne** ou **diagonale**
  remporte la manche ; l'alignement est surligné en vert.
- Grille remplie sans alignement : **match nul**.
- Le score des manches et le nombre de matchs nuls sont cumulés jusqu'au bouton **Quitter**
  ou à un nouveau **Lancer la manche** (qui remet les compteurs à zéro).

En 4 × 4 avec un objectif de 3, le premier joueur est fortement avantagé ;
un objectif de **4** donne des parties beaucoup plus équilibrées.

---

## Architecture technique

Fichier unique, sans dépendance de build. Trois blocs :

- **CSS** — variables de la charte graphique dans `:root`, mise en page en flexbox
  (config / bandeau réseau / stats / panneau latéral + plateau).
- **HTML** — panneau de configuration, barre de statistiques, panneau latéral (joueurs, chat,
  historique), zone de jeu, modales (connexion, aide, fin de manche).
- **JavaScript** (IIFE, mode strict) — trois objets d'état :

| Objet | Rôle |
|---|---|
| `S` | État du jeu : `board`, `current`, `scores`, `history`, `winLine`, `elapsed`… |
| `NET` | État réseau : `mode` (`local` / `host` / `guest`), `peer`, `conn`, `code`, `myRole`, `ping` |
| `SYMBOLS` | Jeux de symboles disponibles |

### Modèle autoritaire

L'**hôte détient l'unique source de vérité**. L'invité n'applique jamais un coup lui-même :
il envoie une intention, l'hôte valide (case libre, bon tour, partie en cours), applique,
puis diffuse l'état complet. Ce choix supprime toute désynchronisation possible entre les deux
postes, au prix d'un aller-retour réseau par coup — négligeable pour un morpion.

### Détection de victoire

`winningLine(r, c, p)` part de la dernière case jouée et étend l'alignement dans les quatre
directions (`→`, `↓`, `↘`, `↙`), dans les deux sens. Complexité `O(k)` par coup, indépendante
de la taille de la grille et valable pour tout couple (taille, objectif).

### Chargement de PeerJS

La bibliothèque est chargée depuis `unpkg`, avec bascule automatique sur `cdnjs` puis `jsDelivr`
si le premier CDN est injoignable (`ensurePeer()`).

---

## Protocole réseau

Messages JSON échangés sur un `DataConnection` fiable. Identifiant de salle :
`morpion-p2p-<code en minuscules>`.

| Message | Sens | Contenu | Effet |
|---|---|---|---|
| `join` | invité → hôte | `name` | Enregistre le pseudo, marque la connexion active |
| `welcome` | hôte → invité | `names`, `myRole` | Confirme le rôle 2 à l'invité |
| `state` | hôte → invité | `s` (instantané complet) | Remplace l'état local et redessine |
| `move` | invité → hôte | `r`, `c` | Intention de coup, validée par l'hôte |
| `chat` | ↔ | `text` | Message de discussion |
| `rematch` | invité → hôte | — | Demande de nouvelle manche |
| `ping` / `pong` | ↔ | `ts` | Mesure de latence toutes les 4 s |
| `bye` | ↔ | — | Départ propre (émis à la fermeture de l'onglet) |

L'instantané `state` contient : `n`, `k`, `board`, `current`, `starter`, `moves`, `history`,
`scores`, `draws`, `rounds`, `names`, `symbolKey`, `running`, `over`, `winLine`, `elapsed`.

---

## Personnalisation

**Rétablir des grilles plus grandes** — ajouter des options dans `#sel-size` (et les objectifs
correspondants dans `#sel-align`) :

```html
<option value="5">5 × 5</option>
<option value="6">6 × 6</option>
```

Aucune autre modification n'est nécessaire : dimensionnement des cases, détection de victoire et
synchronisation sont déjà génériques.

**Ajouter un jeu de symboles** — compléter l'objet `SYMBOLS` puis ajouter l'option dans
`#sel-symbols` :

```js
const SYMBOLS = {
  classic: ["✕", "◯"],
  meteo:   ["☀️", "🌧️"]
};
```

**Changer les couleurs** — tout passe par les variables CSS de `:root`
(`--brand-dark`, `--game-bg`, `--accent`, `--p1`, `--p2`…).

**Utiliser son propre serveur PeerJS** — remplacer les deux instanciations de `Peer` :

```js
const PEER_OPTS = { host: "peer.mondomaine.fr", port: 443, secure: true, path: "/myapp" };
NET.peer = new Peer(PREFIX + NET.code.toLowerCase(), PEER_OPTS); // hôte
NET.peer = new Peer(PEER_OPTS);                                   // invité
```

Serveur correspondant : `npx peer --port 9000 --path /myapp` (paquet `peer`), derrière un
reverse-proxy HTTPS.

---

## Limites connues

- **Annuaire PeerJS public** — gratuit, mais sans garantie de disponibilité ni de confidentialité
  des identifiants de salle. Pour un usage régulier, héberger son propre PeerServer.
- **Réseaux très fermés** — derrière certains pare-feux d'entreprise ou NAT symétriques, WebRTC
  nécessite un serveur **TURN** (non fourni ; à ajouter via l'option `config.iceServers` de PeerJS).
- **Une seule connexion par salle** — toute tentative supplémentaire est refusée (pas de spectateurs).
- **Pas de reprise automatique** — si l'hôte ferme son onglet, la partie est perdue ; il faut
  recréer une salle. Aucun état n'est persisté.
- **Pas d'annulation de coup en ligne** — volontairement retiré du mode réseau pour éviter les
  litiges ; disponible uniquement dans la version locale hors ligne.

---

## Compatibilité

Chrome, Edge, Firefox et Safari récents (bureau et mobile). Le plateau se redimensionne
automatiquement selon la place disponible ; le panneau latéral reste fixe à 225 px.

---

## Licence

Usage libre et modification sans restriction. PeerJS est distribué sous licence MIT.
