<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`verification.md`](verification.md) · [Español](verification.es.md)

# Prouver que l'endpoint IPN est correct

Raisonner sur un handler IPN, ce n'est pas le tester. Ce document dit **ce
qu'il faut prouver** et **ce qu'il est interdit d'improviser** — il ne dit pas
avec quel outil. Construisez la vérification avec ce qui s'exécute sur votre
plateforme.

## La seule chose que vous n'avez pas le droit de générer vous-même

`../scripts/vectors.json` contient, pour la signature, des paires entrée/sortie
figées : les paramètres, la passphrase et, en hexadécimal majuscule, le SHA512
exact que Digistore24 calcule à partir d'eux.

**Votre implémentation doit reproduire chacune d'elles, octet pour octet.**

Tout le document repose sur cette règle, et ce n'est pas par pédanterie :

> Si vous calculez vous-même les signatures attendues, vous avez seulement
> prouvé que votre code est cohérent avec lui-même. La défaillance dont cette
> règle protège — signer avec des noms de champ passés en majuscules — donne
> une implémentation rigoureusement cohérente avec elle-même, dont tous les
> tests passent et qui rejette **chaque paiement réel**. Un test écrit par le
> même auteur, à partir du même malentendu, confirme le bug.

Donc : les vecteurs viennent de l'extérieur. Ne les recalculez pas, ne
« corrigez » pas celui qui échoue, ne régénérez pas le fichier. Un vecteur qui
échoue veut dire que votre signature est fausse.

Ce sont les vecteurs contre lesquels le [Digistore SAAS App
Template](https://github.com/digistore24/ds24-appkit) mesure sa propre
implémentation ; les reproduire, c'est donc concorder avec du code qui tourne
en production.

Vérifiez-les **en premier**. Un vérificateur qui signe mal ne prouve rien sur
un endpoint.

## Deux formes, choisissez celle que votre plateforme peut exécuter

### A — de l'extérieur, en HTTP

Un programme distinct signe des payloads et les poste sur l'endpoint déployé.
Il lui faut un runtime avec accès au réseau, rien d'autre, et il teste
l'endpoint exactement comme Digistore24 l'atteint — à travers le vrai proxy, le
vrai routage, le vrai framework.

**Tout prêt :** `../scripts/verify-ipn.mjs` fait tout cela. Il lui faut Node et
une connexion réseau, rien de plus, et il fonctionne quelle que soit la stack.
Utilisez-le partout où vous disposez d'un shell — Replit, v0, Manus, Claude
Code, Codex ou votre propre machine.

**La limite :** de l'extérieur, le vérificateur ne voit pas la base de
données. Qu'un remboursement ait réellement retiré l'accès lui reste
invisible. C'est à cela que sert la sonde décrite plus bas.

### B — depuis l'intérieur de l'app

Un test écrit dans la stack de l'app elle-même — un test Deno sur Supabase et
Lovable Cloud, un test vitest/jest dans une app Node, pytest en Python. Il
construit le payload signé, appelle le handler (ou poste sur l'URL locale),
puis **regarde directement dans la base de données**.

**C'est la forme à utiliser là où il n'y a pas de shell** — et Lovable est le
cas qui compte : les skills y arrivent avec leurs fichiers embarqués, mais la
plateforme les lit comme de la documentation de référence, elle ne les exécute
pas. En revanche, tout ce que l'agent construit *à l'intérieur de l'app*
s'exécute normalement.

**L'avantage auquel personne ne s'attend :** pas d'endpoint de sonde. Le test
a déjà accès à la base de données et lit donc l'enregistrement d'accès
directement. La forme A a besoin d'un endpoint temporaire ; la forme B s'en
passe.

**La limite :** elle sollicite le handler, pas le déploiement. Une signature
qui passe dans un test et échoue en production — parce qu'un proxy réécrit le
corps, ou que la passphrase manque dans l'environnement déployé — c'est
précisément ce que la forme A aurait attrapé. Une fois l'app en production,
exécutez donc aussi la forme A, une fois, même s'il faut la lancer depuis votre
propre machine.

## Ce qu'il faut prouver

Les deux formes vérifient les mêmes choses. Tout ce qui ne figure pas dans
cette liste est un bonus ; tout ce qui y manque est un trou.

**La signature**

| Cas | Doit |
|---|---|
| chaque vecteur de `vectors.json` | être reproduit exactement |
| payload correctement signé | être accepté |
| un octet inversé dans la signature | être rejeté |
| payload dont une valeur a changé après la signature | être rejeté |
| pas de `sha_sign` dans le payload | être rejeté |
| aucune passphrase configurée | être rejeté — **jamais un contournement** |
| signature calculée sur des noms de champ en majuscules | être acceptée |
| `GET` sur l'endpoint | répondre `200` — c'est ainsi que Digistore24 valide l'endpoint |

**Le cycle de vie de l'accès** — un nouvel id de commande par cas, pour qu'ils
ne puissent pas interférer

| Cas | Doit |
|---|---|
| `on_payment` | accorder l'accès |
| le même événement livré deux fois | ne pas accorder deux fois, ne pas créditer deux fois |
| `on_refund` | retirer l'accès |
| `on_payment` renvoyé *après* `on_refund` | **ne pas** faire revivre l'accès |
| `on_payment_missed` | suspendre — l'accès disparaît, l'enregistrement n'est pas terminé |
| `on_payment` après `on_payment_missed` | restaurer — la suspension est **levée** |
| `on_rebill_cancelled` | laisser l'accès **inchangé** |
| `last_paid_day` | mettre fin à l'accès |

Les deux derniers forment la paire qui coûte de l'argent quand on la devine.
Voir `events.fr.md`.

## La sonde — forme A uniquement

Pour vérifier la moitié « accès » depuis l'extérieur, l'app doit pouvoir
répondre à une seule question. Construisez un petit endpoint qui reçoit un
`order_id` et renvoie

```json
{ "access": true, "suspended": false }
```

Trois règles :

- **Protégez-le par un bearer token.** Il renseigne sur les achats d'autres
  personnes.
- **Supprimez-le dès que l'exécution est au vert.** C'est un fixture de test,
  pas une fonctionnalité. Un endpoint qui survit au test est un endpoint que
  personne ne pensera à sécuriser.
- **Il lit, il n'écrit jamais.**

La forme B n'a besoin de rien de tout cela.

## Le compte rendu

**Dites ce qui n'a pas été vérifié.** Une exécution qui a sauté la moitié
« accès » faute de sonde n'est pas une exécution au vert : c'est une signature
prouvée et une sémantique qui ne l'est pas. `verify-ipn.mjs` affiche `SKIP`
précisément pour cette raison et ne le compte jamais comme une réussite ; ce
que vous construisez devrait faire de même.

Et rapportez ce que l'exécution a réellement dit, pas le fait que vous l'avez
lancée.
