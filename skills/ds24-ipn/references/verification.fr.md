<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`verification.md`](verification.md) · [Español](verification.es.md)

# Prouver que l'endpoint IPN est correct

Un handler d'IPN sur lequel on a seulement raisonné n'a pas été testé. Ce
document dit **ce qu'il faut prouver** et **ce qu'il ne faut pas improviser** —
pas quel outil utiliser. Construisez la vérification avec ce qui tourne sur
votre plateforme.

## La seule chose que vous n'avez pas le droit de générer vous-même

`../scripts/vectors.json` contient des paires entrée/sortie figées pour la
signature : paramètres, passphrase et le SHA512 exact, en hexadécimal
majuscule, que Digistore24 produit pour elles.

**Votre implémentation doit tous les reproduire, octet pour octet.**

C'est la règle porteuse de tout le document, et la raison n'est pas de la
pédanterie :

> Si vous calculez les signatures attendues avec votre propre code, vous avez
> prouvé que votre code est d'accord avec lui-même. La défaillance contre
> laquelle cela protège — signer avec les noms de champ en majuscules — produit
> une implémentation parfaitement cohérente avec elle-même, dont les propres
> tests passent tous et qui rejette **chaque paiement réel**. Un test écrit par
> le même auteur, à partir du même malentendu, est d'accord avec le bug.

Donc : les vecteurs viennent de l'extérieur. Ne les recalculez pas, ne
« corrigez » pas celui qui échoue, ne régénérez pas le fichier. Un vecteur qui
échoue signifie que votre signature est fausse.

Ce sont les mêmes vecteurs contre lesquels le [Digistore SAAS App
Template](https://github.com/digistore24/ds24-appkit) mesure sa propre
implémentation, donc les reproduire signifie être d'accord avec du code qui est
en production.

Vérifiez-les **d'abord**. Un vérificateur dont la signature est fausse ne dit
rien d'un endpoint.

## Deux formes, choisissez celle que votre plateforme peut exécuter

### A — de l'extérieur, en HTTP

Un programme distinct signe des payloads et les poste sur l'endpoint déployé. Il
lui faut un runtime avec accès réseau et rien d'autre, et il teste l'endpoint
exactement comme Digistore24 l'atteint — à travers le vrai proxy, le vrai
routage, le vrai framework.

**Tout prêt :** `../scripts/verify-ipn.mjs` fait tout cela. Il lui faut Node et
une connexion réseau, rien de plus, et il fonctionne contre n'importe quelle
stack. Utilisez-le partout où vous avez un shell — Replit, v0, Manus, Claude
Code, Codex ou votre propre machine.

**Le hic :** de l'extérieur, le vérificateur ne peut pas voir la base de
données. Savoir si un remboursement a réellement retiré l'accès lui est
invisible. C'est à cela que sert la sonde ci-dessous.

### B — depuis l'intérieur de l'app

Un test dans la stack de l'app elle-même — un test Deno sur Supabase et Lovable
Cloud, un test vitest/jest dans une app Node, pytest en Python. Il construit le
payload signé, appelle le handler (ou poste sur l'URL locale), puis
**regarde directement la base de données**.

**Utilisez ceci là où il n'y a pas de shell** — Lovable est le cas qui compte :
les skills y voyagent avec leurs fichiers embarqués, mais la plateforme les lit
comme du matériel de référence au lieu de les exécuter. Tout ce que l'agent
construit *à l'intérieur de l'app* tourne très bien.

**L'avantage que personne n'attend :** aucun endpoint de sonde. Le test a déjà
accès à la base de données, il lit donc directement l'enregistrement d'accès. La
forme A a besoin d'un endpoint temporaire dont la forme B se passe.

**Le hic :** cela exerce le handler, pas le déploiement. Une signature qui
marche dans un test et échoue en production — parce qu'un proxy réécrit le
corps, ou que la passphrase manque dans l'environnement déployé — est exactement
ce que la forme A aurait attrapé. Quand l'app est en production, exécutez aussi
la forme A une fois, même s'il faut la lancer depuis votre propre machine.

## Ce qu'il faut prouver

Les deux formes vérifient les mêmes choses. Tout ce qui n'est pas sur cette
liste est un bonus ; tout ce qui y manque est un trou.

**La signature**

| Cas | Doit |
|---|---|
| chaque vecteur de `vectors.json` | être reproduit exactement |
| payload correctement signé | être accepté |
| un octet inversé dans la signature | être rejeté |
| une valeur modifiée après la signature | être rejeté |
| pas de `sha_sign` dans le payload | être rejeté |
| aucune passphrase configurée | être rejeté — **jamais un contournement** |
| signature sur des noms de champ en majuscules | être acceptée |
| `GET` sur l'endpoint | répondre `200` — Digistore24 le valide ainsi |

**Le cycle de vie de l'accès** — un order id neuf par cas, pour qu'ils ne
puissent pas interférer

| Cas | Doit |
|---|---|
| `on_payment` | accorder l'accès |
| le même événement livré deux fois | ne pas accorder deux fois, ne pas créditer deux fois |
| `on_refund` | retirer l'accès |
| `on_payment` relivré *après* `on_refund` | **ne pas** faire revivre l'accès |
| `on_payment_missed` | suspendre — accès retiré, enregistrement pas terminé |
| `on_payment` après `on_payment_missed` | restaurer — la suspension est **levée** |
| `on_rebill_cancelled` | laisser l'accès **inchangé** |
| `last_paid_day` | mettre fin à l'accès |

Les deux dernières sont la paire qui coûte de l'argent quand on la devine.
Voyez `events.fr.md`.

## La sonde — forme A uniquement

Pour vérifier la moitié « accès » depuis l'extérieur, l'app doit répondre à une
question. Construisez un petit endpoint qui prend un `order_id` et renvoie

```json
{ "access": true, "suspended": false }
```

Trois règles :

- **Protégez-le avec un bearer token.** Il rend compte des achats d'autres
  personnes.
- **Supprimez-le quand l'exécution est verte.** C'est un fixture de test, pas
  une fonctionnalité. Un endpoint qui survit au test est un endpoint que
  personne ne pense à sécuriser.
- **Il lit, il n'écrit jamais.**

La forme B n'a besoin de rien de tout cela.

## Le compte rendu

**Dites ce qui n'a pas été vérifié.** Une exécution qui a sauté la moitié
« accès » parce qu'aucune sonde n'existait n'est pas une exécution verte — c'est
une signature prouvée et une sémantique non prouvée. `verify-ipn.mjs` affiche
`SKIP` exactement pour cette raison et ne le compte jamais comme une réussite ;
ce que vous construisez devrait faire de même.

Et rapportez ce que l'exécution a réellement dit, pas le fait que vous l'avez
lancée.
