# BELOTRO 🃏

**La coinche façon roguelike.** Un moteur complet de belote coinchée habillé d'une boucle roguelike inspirée de _Balatro_ : enchère, joue tes plis, empoche des jetons, achète des Jokers entre les blindes et franchis des Antes de plus en plus corsés. Rate l'objectif d'une blinde et tout le run s'effondre.

Le tout tient dans **un seul fichier HTML autonome** — aucune dépendance de build.

## ▶️ Jouer

Ouvre simplement `index.html` dans un navigateur, ou joue en ligne sur le déploiement Vercel.

## 🎮 Modes

- **Solo** — 1 joueur contre 3 IA.
- **Entre potes** — pass-and-play local (2 à 4 joueurs) avec rideau de confidentialité entre les mains.
- **En ligne** — salons par code via Supabase Realtime (multijoueur gratuit, sans serveur dédié).

## 🧩 Mécaniques de belote

- Contrats **atout / sans-atout / tout-atout**, de 80 à **160**, puis **Capot**, puis **Générale**.
- **Coinche** (×2) & **surcoinche** (×4).
- **Belote-Rebelote**, **dix de der**.
- IA d'enchère et de jeu (respect des obligations : fournir, couper, monter à l'atout).

## 🎲 Boucle roguelike (façon Balatro)

- **Antes** découpés en 3 **blindes** : Petite, Grande, Boss.
- Chaque blinde impose un **score à atteindre** (croissant) en un **nombre de manches limité**.
- **Blindes Boss** avec modificateurs (Vent Contraire, Der de Der, Petit Jeu, Brouillard).
- **Boutique** de **Jokers** entre les blindes (16 jokers, 5 slots).
- **Échec = permadeath** : rater une blinde efface le run.

## 🛠️ Stack

- HTML/CSS/JS vanilla, un seul fichier, zéro build.
- **Son** synthétisé via Web Audio API (aucun asset externe).
- **Multijoueur** : Supabase Realtime (broadcast + presence), hébergement statique sur **Vercel**.

## 📦 Développement

Voir [`CLAUDE.md`](./CLAUDE.md) pour l'architecture détaillée.
