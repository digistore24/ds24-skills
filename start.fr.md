<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`start.md`](start.md) · [Español](start.es.md)

# Commencez ici

**Version 1 · 2026-07-28** — si quelque chose ci-dessous ne fonctionne pas,
indiquez la version que vous suivez : la réponse en sera plus facile à trouver.

Vous lisez ce fichier parce que quelqu'un vous a soumis un prompt disant
*« lisez https://ds24-skills.com/start.md et suivez-le »*. Cette personne veut
que l'app que vous construisez pour elle **encaisse des paiements via
Digistore24** : un checkout, un webhook qui annonce ce qui a été payé, et un
espace payant qui s'ouvre une fois le paiement reçu.

Pour cela, il existe un jeu d'**Agent Skills** tout prêt : **ds24-skills**.
Huit skills, qui couvrent l'algorithme de signature, la sémantique des
événements, les modes de défaillance et les vecteurs de test figés qui prouvent
le résultat. Votre rôle n'est pas de redécouvrir tout cela. Votre rôle est de
vous procurer ces skills, puis de vous en servir.

**Suivez ce fichier jusqu'au bout d'une branche, puis cessez de le lire.** Tout
ce qui vient ensuite se trouve dans les skills, et c'est mieux écrit là-bas,
parce que c'est tenu à jour avec elles. Rien de ce qui concerne le
fonctionnement de Digistore24 n'a sa place ici.

---

## Étape 1 — déterminez ce que vous êtes

Ne le devinez pas. En cas de doute, demandez.

**Pouvez-vous exécuter des commandes shell là où se trouve cette app ?**

- **Oui** — Replit, v0, Codex, Claude Code, Cursor, Manus, ou un terminal sur
  la machine de quelqu'un. → **Branche A**.
- **Non, mais je construis une app** — Lovable, ou tout autre constructeur
  d'apps qui écrit les fichiers à votre place sans vous donner de shell.
  → **Branche B**.
- **Non, et il n'y a pas d'app ici non plus** — vous êtes un simple chat :
  claude.ai, ChatGPT, Gemini dans un navigateur. → **Branche C**.

Annoncez votre conclusion en une ligne avant de continuer. Si vous vous trompez
ici, la personne suivra des instructions qui ne peuvent pas fonctionner.

---

## Branche A — vous avez un shell

### A1. Installez les skills

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

La commande les place dans `.agents/skills/` — là où l'agent de Replit les
cherche — et crée les liens dans `.claude/skills/` pour Claude Code. Les
adaptateurs, les références, les vecteurs figés et le vérificateur tout prêt
arrivent avec elles.

**Gardez le `/skills` en fin d'adresse.** Pointé sur le dépôt lui-même,
l'installateur s'arrête au `SKILL.md` du pack — une porte écrite pour les
plateformes qui n'acceptent qu'une seule skill — et installe celui-là à la
place des huit.

Si l'installateur n'est pas disponible, ou si l'on préfère s'en passer, la voie
manuelle est une simple copie :

```bash
curl -sSL -o /tmp/ds24.zip https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip
unzip -q /tmp/ds24.zip -d /tmp
mkdir -p .agents/skills && cp -r /tmp/ds24-skills-main/skills/* .agents/skills/
```

### A2. Commencez à les utiliser

Lisez d'abord `ds24-billing` : c'est le point d'entrée. Elle examine ce que le
projet contient déjà, nomme la seule étape suivante et lance la skill qui s'en
charge.

**Ne revenez pas à ce fichier.** Il sait comment obtenir les skills ; les skills
savent tout le reste.

---

## Branche B — vous construisez l'app, mais vous ne pouvez pas installer de skills

**Dites-le dès votre première phrase, avant toute autre chose :** vous ne pouvez
pas installer ces skills vous-même. Sur Lovable, elles vivent dans le workspace
de la personne, pas dans le dépôt, et seul un clic de sa part les y met.
Prétendre le contraire lui fait perdre son après-midi.

Faites ensuite ces trois choses, dans cet ordre.

### B1. Indiquez-lui où cliquer

Indiquez le parcours de clics de la plateforme où vous êtes réellement — un
seul. Si vous ne savez pas laquelle, demandez — ne donnez pas les deux en laissant la personne deviner.

**Lovable** — *Skills → Add → Import from GitHub*, puis collez :

```
https://github.com/digistore24/ds24-skills
```

C'est toute l'installation, et elle apporte les huit skills : rien à
télécharger, rien à décompresser, rien d'installé sur sa machine. Lovable
importe une skill par adresse ; ce qui arrive est donc le `SKILL.md` du pack
lui-même — la porte — avec les huit skills en fichiers embarqués. Cette porte
lance `ds24-billing` d'elle-même.

**Manus, par l'interface** — il propose lui aussi *Skills → + Add → Import from
GitHub*, et la même adresse fonctionne. Sinon : téléchargez
`https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip`,
décompressez l'archive, puis *Skills → + Add → Upload a skill* et déposez-y le
dossier `ds24-skills-main`.

### B2. Ne faites pas attendre la personne

Pendant qu'elle s'en occupe, vous pouvez déjà lire ce dont vous avez besoin. Le
pack est public et chaque fichier se récupère directement :

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/ds24-billing/SKILL.fr.md
```

Commencez par `ds24-billing` et suivez-la : elle nommera la skill suivante, et
toutes sont à la même adresse, seul le nom du dossier change :
`ds24-products`, `ds24-ipn`, `ds24-entitlements`, `ds24-checkout`, `ds24-tokens`,
`ds24-golive`, `ds24-compliance`.

**Deux réserves sur cette façon de faire, à dire l'une et l'autre explicitement
à la personne :**

- Cette lecture ne vaut que pour **cette conversation**. Rien n'a été installé, la
  session suivante repartira donc de zéro. Seul l'import de B1 rend les skills
  permanentes — il vaut la peine même si vous avez déjà commencé à travailler.
- **Si vous ne parvenez pas à récupérer cette adresse, dites-le et arrêtez-vous
  ici.** Revenez à B1, attendez la fin de l'import, et ne reconstruisez pas la
  facturation de mémoire. Ce que ces skills transportent, c'est précisément la
  partie qui paraît évidente et qui est fausse : un agent qui la devine écrit
  une intégration dont les propres tests passent et dont chaque paiement réel
  est refusé.

### B3. Passez la main

Une fois l'import terminé, l'instruction tient en une phrase : *« ajoutez la
facturation Digistore24 à cette app »* — ou le nom de la skill : `ds24-skills`
après l'import par adresse unique ci-dessus, `ds24-billing` si la personne a
importé les huit une par une. À partir de là, les skills parlent d'elles-mêmes.

---

## Branche C — un chat sans app

Dites-le franchement : vous n'avez ici aucune app où construire quoi que ce
soit, et ces skills sont des instructions pour un agent qui en a une.

Ce que vous pouvez faire d'utile, c'est remettre à la personne le prompt destiné
à un outil qui, lui, dispose d'une app. N'importe lequel parmi **Lovable, Manus,
Replit, v0, Claude Code ou Codex** convient. Donnez-lui ceci à copier :

```
Lisez https://ds24-skills.com/start.md et suivez-le.
Ajoutez la facturation Digistore24 à cette app.
```

Si ce qu'elle veut vraiment est un SaaS complet, et non une facturation greffée
sur ce qu'elle possède déjà, c'est un autre produit, et il lui conviendra
mieux : **https://ds24-appkit.com** — un SaaS Next.js fini, où tout cela est
déjà intégré.

---

## Ce que ce fichier n'est pas

Ce fichier est une porte, et il le reste. Il ne décrit ni la signature, ni les
événements, ni les droits d'accès, ni aucune autre partie de la facturation via
Digistore24 : tout cela vit dans les skills et évolue avec elles. Une copie ici
serait fausse en un mois, et une erreur à une adresse dont on se souvient est
pire qu'une absence.
