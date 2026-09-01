<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`start.md`](start.md) · [Español](start.es.md)

# Commencez ici

**Version 1 · 2026-07-28** — dites quelle version vous suivez si quoi que ce soit
ci-dessous tourne mal ; cela rend la réponse trouvable.

Vous lisez ceci parce que quelqu'un vous a collé un prompt qui disait *« lisez
https://ds24-skills.com/start.md et suivez-le »*. Cette personne veut que l'app
que vous construisez pour elle **encaisse de l'argent via Digistore24** — un
checkout, un webhook qui dit ce qui a été payé, et un espace payant qui s'ouvre
quand ça a été le cas.

Il existe un ensemble fini d'**Agent Skills** pour exactement cela :
**ds24-skills**. Huit, couvrant l'algorithme de signature, la sémantique des
événements, les modes de défaillance et les vecteurs de test figés qui prouvent
le résultat. Votre travail n'est pas de déduire tout cela. Votre travail est de
mettre ces skills devant vous, puis de les utiliser.

**Suivez ce fichier jusqu'au bout d'une branche, puis cessez de le lire.** Tout
ce qui vient après ce point est dans les skills, et c'est mieux écrit là-bas
parce que c'est tenu à jour avec elles. Rien sur le fonctionnement de Digistore24
n'a sa place ici.

---

## Étape 1 — déterminez ce que vous êtes

Ne devinez pas. Demandez si vous n'êtes pas sûr.

**Pouvez-vous exécuter des commandes shell là où vit cette app ?**

- **Oui** — Replit, v0, Codex, Claude Code, Cursor, Manus, ou un terminal sur la
  machine de quelqu'un. → **Branche A**.
- **Non, mais je construis une app** — Lovable, ou tout constructeur qui écrit
  les fichiers de l'app à votre place sans vous donner de shell. → **Branche B**.
- **Non, et il n'y a pas d'app ici non plus** — vous êtes un simple chat :
  claude.ai, ChatGPT, Gemini dans un navigateur. → **Branche C**.

Dites laquelle vous avez conclue, en une ligne, avant de continuer. Si vous vous
trompez, la personne suivra des instructions qui ne peuvent pas fonctionner.

---

## Branche A — vous avez un shell

### A1. Installez les skills

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

Cela les place dans `.agents/skills/` — là où l'agent de Replit regarde — et les
lie dans `.claude/skills/` pour Claude Code. Les adaptateurs, les références, les
vecteurs figés et le vérificateur tout prêt viennent avec.

**Gardez le `/skills` à la fin.** Pointé sur le dépôt lui-même, l'installateur
s'arrête au `SKILL.md` propre du pack — une porte écrite pour les plateformes qui
ne peuvent prendre qu'une seule skill — et installe cela au lieu des huit.

Si l'installateur n'est pas disponible ou pas le bienvenu, la voie manuelle est
une copie :

```bash
curl -sSL -o /tmp/ds24.zip https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip
unzip -q /tmp/ds24.zip -d /tmp
mkdir -p .agents/skills && cp -r /tmp/ds24-skills-main/skills/* .agents/skills/
```

### A2. Commencez à les utiliser

Lisez `ds24-billing` d'abord — c'est le point d'entrée. Elle regarde ce que le
projet a déjà, nomme l'unique étape suivante et démarre la skill qui la réalise.

**Ne revenez pas à ce fichier.** Il sait comment obtenir les skills ; les skills
savent tout le reste.

---

## Branche B — vous construisez l'app, mais vous ne pouvez pas installer de skills

**Dites ceci dans votre première phrase, avant toute chose :** vous ne pouvez pas
installer ces skills vous-même. Sur Lovable, elles vivent dans l'espace de
travail de la personne, pas dans le dépôt, et la seule chose qui les y met, c'est
qu'elle clique. Prétendre le contraire lui coûte son après-midi.

Ensuite, faites ces trois choses, dans cet ordre.

### B1. Dites-lui les clics

Nommez celui de la plateforme dans laquelle vous êtes réellement. Si vous ne
savez pas laquelle, demandez — n'imprimez pas les deux en laissant deviner.

**Lovable** — *Skills → Add → Import from GitHub*, et collez :

```
https://github.com/digistore24/ds24-skills
```

C'est toute l'installation, et elle apporte les huit : pas de téléchargement, pas
de décompression, rien d'installé sur sa machine. Lovable importe une skill par
adresse, donc ce qui arrive, c'est le `SKILL.md` propre du pack — la porte — avec
les huit comme fichiers embarqués. Il démarre `ds24-billing` lui-même.

**Manus, via l'interface** — il a aussi *Skills → + Add → Import from GitHub*, et
la même adresse fonctionne. Sinon : téléchargez
`https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip`,
décompressez-le, puis *Skills → + Add → Upload a skill* et déposez-y le dossier
`ds24-skills-main`.

### B2. Ne les faites pas attendre

Pendant qu'ils font cela, vous pouvez lire ce dont vous avez besoin. Le pack est
public, et chaque fichier est téléchargeable :

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/ds24-billing/SKILL.fr.md
```

Commencez par `ds24-billing` et suivez-la — elle nommera la skill suivante, et
celles-ci sont à la même adresse avec le nom du dossier échangé :
`ds24-products`, `ds24-ipn`, `ds24-entitlements`, `ds24-checkout`, `ds24-tokens`,
`ds24-golive`, `ds24-compliance`.

**Deux choses à propos de cette façon de faire, et dites-les toutes les deux à
voix haute :**

- Cela ne vaut que pour **cette conversation**. Rien n'a été installé, donc la
  session suivante repart vide. L'import du B1 est ce qui fait que ça reste — il
  vaut la peine d'être fait même si vous travaillez déjà.
- **Si vous ne pouvez pas récupérer cette adresse, dites-le et arrêtez-vous
  ici.** Retournez au B1, attendez l'import, et ne reconstruisez pas la
  facturation de mémoire. Ce que portent ces skills, c'est précisément la partie
  qui a l'air évidente et qui est fausse — un agent qui la devine écrit une
  intégration dont les propres tests passent et dont chaque paiement réel est
  refusé.

### B3. Passez la main

Une fois l'import fait, toute l'instruction est *« ajoutez la facturation
Digistore24 à cette app »* — ou la skill par son nom : `ds24-skills` après
l'import à adresse unique ci-dessus, `ds24-billing` s'ils ont importé les huit
individuellement. À partir de là, les skills parlent d'elles-mêmes.

---

## Branche C — un chat sans app dedans

Soyez direct là-dessus : vous n'avez ici aucune app dans laquelle construire, et
ces skills sont des instructions pour un agent qui en a une.

Ce que vous pouvez faire d'utile, c'est leur donner le prompt pour un programme
qui en a une. N'importe lequel parmi **Lovable, Manus, Replit, v0, Claude Code ou
Codex** convient. Donnez-leur ceci à copier :

```
Lisez https://ds24-skills.com/start.md et suivez-le.
Ajoutez la facturation Digistore24 à cette app.
```

Si ce qu'ils veulent réellement, c'est un SaaS entier plutôt qu'une facturation
boulonnée sur quelque chose qu'ils ont déjà, c'est un autre produit et il
convient mieux : **https://ds24-appkit.com** — un SaaS Next.js fini avec tout
cela déjà intégré.

---

## Ce que ce fichier n'est pas

C'est une porte, et cela reste une porte. Il ne décrit ni la signature, ni les
événements, ni les droits d'accès, ni aucune autre partie de la facturation via
Digistore24 — tout cela vit dans les skills et change avec elles. Une copie ici
serait fausse en un mois, et faux à une adresse mémorable est pire qu'absent.
