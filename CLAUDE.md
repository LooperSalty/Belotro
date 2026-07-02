# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Vue d'ensemble

**BELOTRO** est un jeu de **belote coinchée** habillé d'une boucle **roguelike** inspirée de _Balatro_. Tout le jeu tient dans **un unique fichier autonome** : `Belotro.html` (HTML + CSS + JS vanilla, ~1900 lignes). **Aucun build, aucune dépendance npm, aucun asset externe** hormis les polices Google Fonts. Le son est synthétisé à la volée (Web Audio API).

Pour travailler : ouvrir `Belotro.html` dans un navigateur. Il n'existe ni package.json, ni test runner, ni lint — les modifications se font directement dans le fichier.

## Lancer / tester

- **Jouer** : ouvrir `Belotro.html` (double-clic) OU le servir en HTTP (le navigateur peut refuser `file://` pour certaines API). Ex. serveur statique : `node -e "require('http').createServer((q,r)=>require('fs').createReadStream('Belotro.html').pipe(r)).listen(4599)"` puis http://localhost:4599.
- **Vérifier la syntaxe JS** sans navigateur : extraire le contenu de `<script>` et le passer à `new Function(code)` sous Node (`node -e "..."`).
- **Tester le moteur de façon déterministe** : dans la console du navigateur, rediriger `window.setTimeout` vers une file pompée manuellement (contourne à la fois l'asynchronisme des temporisations ET le throttling des onglets d'arrière-plan), puis piloter les points de décision humains (passer les enchères, jouer `legalMoves(seat)[0]`, cliquer les boutons d'overlay). Invariant clé à vérifier en phase `play` : `Σ hands[i].length + trickIndex*4 + trick.cardsPlayed.length === 32` (détecte toute fuite de carte).

> ⚠️ **Piège de test** : dans un onglet Chrome en arrière-plan (`document.visibilityState === 'hidden'`), `setTimeout`/`setInterval` sont bridés (~1s minimum) et coalescés. Piloter le jeu avec un `setInterval` externe qui court après les chaînes de `setTimeout` du moteur provoque des **races** (mains incohérentes) qui n'existent PAS en jeu réel. Préférer le pilotage synchrone décrit ci-dessus.

## Architecture du fichier `Belotro.html`

Trois sections : `<style>` (thème + animations), le markup (`#menu`, `#table`, `#overlayRoot`), puis un gros `<script>`.

### État global : `S`

Toute la partie vit dans un seul objet mutable `S` (et `uiState` pour les choix du menu). `S = null` hors partie. **Le code mute `S` en place** (ce n'est PAS du style immuable — cohérence interne du fichier). Champs clés : `hands[4]`, `originalHands[4]` (pour détecter la belote), `contract`, `bidding`, `trick`, `tricksWon/tricksCountWon/trickWinsByPlayer`, `teamScore[2]`, `jetons`, `jokers[]`, `ante`, `blindIndex`, `blindReq`, `blindProgress`, `handsLeft`, `bossModifier`, `phase`.

`phase` pilote toute la logique : `'bidding' → 'coinche_def' → 'play' → 'scoring' → 'shop'`.

### Moteur de belote (le cœur, à ne pas casser)

Fonctions pures/quasi-pures reliées par des tables de constantes en tête de script :
- Ordres et points : `TRUMP_ORDER`/`NORMAL_ORDER`, `TRUMP_PTS_BASE`/`NORMAL_PTS`. `trumpPts()` applique les jokers modifiant les points d'atout (Petit Prince → Valet 30, Atout Maître → 9 vaut 20).
- `isTrumpCard`, `cardStrength`, `cardPoints`, `sortHandForDisplay` : dépendent du `contract` (type `'atout' | 'sans-atout' | 'tout-atout'`).
- `legalMoves(seat)` : **règles de fournir / couper / monter à l'atout / pisser** (l'obligation de monter dépend de qui mène le pli, y compris la coopération avec le partenaire `(seat+2)%4`). C'est la fonction la plus subtile.
- `currentTrickWinner()` : gagnant du pli courant selon atout > couleur demandée.
- IA : `aiBiddingDecision`, `aiCoincheDecision` (enchères), `aiPlay` (jeu — mène petit, sinon monte pour prendre / défausse petit si le partenaire tient le pli).

**Équipes** : `TEAM_OF = [0,1,0,1]` — siège 0 = « Toi » (Nord-Sud, équipe A, celle du joueur/du run roguelike), sièges impairs = Est-Ouest (équipe B). Ordre de parole/jeu : `(dealerIndex+1)%4` puis sens horaire.

### Contrats et enchères

Paliers `BID_STEPS = [80..160]` (max **160** car le total réel est 162), puis **Capot** (`mode:'capot'`, 250 pts, l'équipe fait les 8 plis) puis **Générale** (`mode:'generale'`, 500 pts, **un seul joueur** — `contract.player` — fait les 8 plis, suivi via `trickWinsByPlayer`). Comparaison via `bidRank()` (numérique < 200 capot < 300 générale). Coinche `×2` / surcoinche `×4` via `contract.coincheLevel`. `bidCap()`/`chelemAllowed()` gèrent le boss « Petit Jeu ».

### Boucle roguelike (façon Balatro) — la couche par-dessus le moteur

- **Ante → 3 blindes** (`BLIND_NAMES`/`BLIND_MULT` : Petite ×1, Grande ×1.5, Boss ×2). `baseReq(ante)=arrondi(100·1.6^(ante-1))`, `blindReq()` applique le joker Stratège (−10%).
- `setupBlind()` fixe l'objectif (`blindReq`), les manches disponibles (`handsForBlind()` : 3, +1 boss, +1 joker Tenace, −1 boss Brouillard) et tire un `bossModifier` (uniquement en blinde Boss).
- Chaque manche, `finishHandScoring()` calcule `scoreA` (points Nord-Sud) ; `afterScoring()` l'ajoute à `blindProgress` et décrémente `handsLeft`. **`blindProgress ≥ blindReq` → `blindCleared()` → boutique → `advanceBlind()`**. **`handsLeft` épuisé sans objectif → `runFailed()` → retour menu (permadeath roguelike).** Boss du dernier Ante franchi → `runWon()`.
- **Boutique** (`openShop`/`buyJoker`) : 3 jokers proposés parmi `JOKER_POOL` (16, max `MAX_JOKERS=5` possédés), prix réduits par le joker Fine Bouche. Modificateurs boss dans `BOSS_POOL`.

Chaque joker est un `id` dans `S.jokers` ; son effet est codé en dur à l'endroit pertinent (chercher `S.jokers.includes('<id>')`). **Ajouter un joker = l'ajouter à `JOKER_POOL` ET brancher son effet.**

### Rendu

`render()` reconstruit tout le DOM depuis `S` à chaque changement (topbar via `#topGrp`, sièges, `trickZone`, `logPanel`, main du sud). C'est appelé partout après mutation. `renderSouthHand()` gère la main interactive du joueur humain courant (en solo : toujours la main du siège 0 ; en jeu : `legalMoves` marque les cartes jouables). `cardHTML()` génère une carte. Les overlays (`ov()`/`clearOv()`) couvrent l'écran pour score/boutique/fin.

### Modes & « rideau » (hotseat)

`uiState.mode` = `'solo'` (1 humain siège 0 + 3 IA) ou `'hotseat'` (pass-and-play, 2-4 humains). `showCurtainIfNeeded()` affiche un rideau de confidentialité au changement de joueur humain (jamais en solo). `playerTypes[seat]` vaut `'human'` ou `'ai'`.

### Son & animations

- **Son** : `ensureAudio()` (AudioContext paresseux, initialisé sur le 1er clic), `beep()` (oscillateur + enveloppe), `sfx(name)` (presets). `toggleMute()` / `muted`. **Tout est synthétisé — ne jamais ajouter de fichier audio externe.**
- **Animations** : CSS (`dropIn` cartes au pli, `dealIn` distribution, `floatUp` points flottants via `floatPoints()`, `shake` sur coinche via `screenShake()`, `popIn` overlays). Le layout est **mono-écran sans scroll** (`#table{overflow:hidden}`, plateau `grid-template-rows:auto 1fr auto`) — toute modif des tailles de cartes doit préserver ça.

## Conventions

- **Français** partout (UI, logs, commentaires).
- Le fichier assume la **mutation en place** de `S` : rester cohérent avec ce style ici plutôt que d'imposer l'immutabilité.
- Après avoir modifié la logique de jeu, **relancer le test d'intégrité synchrone** (invariant des 32 cartes) avant de committer.

## Déploiement & multijoueur

- **GitHub** : dépôt public `LooperSalty/Belotro` (remote `origin` en SSH).
- **Vercel** : hébergement statique. `vercel.json` réécrit `/` → `/Belotro.html`. Déploiement par intégration Git (push sur `origin`) ou `vercel deploy` (nécessite `vercel login`).
- **Multijoueur en ligne** (mode « En ligne », gratuit) : implémenté via **Supabase Realtime** (broadcast + presence), salons par code, modèle **host-authoritative** (l'hôte fait tourner le moteur et diffuse des vues expurgées ; les invités sont des terminaux qui renvoient leurs intentions `bid`/`pass`/`coinche`/`play`). Code dans le bloc `MULTIJOUEUR EN LIGNE` : `NET`/`GV` (état réseau), `joinChannel`/`onPresence`/`showLobby` (salon), `netView`/`netBroadcastSync` (hôte), `applyGuestView`/`renderGuestHand` (invité), `onHostAction` (application des intentions). Les fonctions moteur branchent le type de siège `'human'` (local) / `'remote'` (invité) / `'ai'` via `S.awaiting`. La couche roguelike (blindes/boutique/jokers) reste **solo** ; en ligne = belote « classique », 1re équipe à `NET.target` (1000). **Clés Supabase** (`window.BELOTRO_SB_URL` / `BELOTRO_SB_ANON`) injectées en tête de fichier (placeholders `__SUPABASE_URL__`/`__SUPABASE_ANON__`) ; lib `supabase-js` **vendorisée** dans `supabase.min.js` (même origine). Sans clés, le mode dégrade proprement (message « non configuré »), solo/hotseat intacts. Realtime doit être activé côté projet Supabase (broadcast/presence ne nécessitent aucune table).
