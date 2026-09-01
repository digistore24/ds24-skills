---
name: ds24-billing
language: fr
description: À utiliser comme point d'entrée pour vendre quoi que ce soit via Digistore24 depuis une app — commencez ici lorsque l'utilisateur veut encaisser des paiements, ajouter un paywall, vendre un abonnement ou des crédits, connecter Digistore24, ou demande par où commencer avec la facturation. Elle détermine ce qui existe déjà, nomme l'unique étape suivante et passe la main à la skill qui la réalise. À utiliser aussi lorsqu'une intégration Digistore24 dysfonctionne et qu'on ne sait pas encore quelle partie est en cause.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Vendre via Digistore24 — commencez ici

Digistore24 est le marchand de référence (merchant of record) — il gère le
checkout, encaisse l'argent, s'occupe de la TVA et des remboursements, et
raconte à votre app ce qui s'est passé au moyen d'un webhook signé. Le travail
de votre app est petit et exact — envoyer les gens vers un checkout et
transformer les événements qui reviennent en accès.

Cette skill détermine où en est le projet et lance la bonne skill suivante. Elle
ne construit rien elle-même.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez-le au fichier `VERSION` de ce pack. Les skills importées dans un
workspace ne se mettent pas à jour toutes seules, une copie peut donc
avoir des mois. Signalez tout écart en une phrase, proposez de réimporter, puis
continuez dans tous les cas.

## Étape 0b — les sept autres ne sont peut-être pas là

Cette skill passe la main à sept autres, nommément. Sur Lovable et Manus, chaque
skill est importée séparément, celle que vous lisez peut donc être la seule
présente — et « lancez `ds24-ipn` » devient alors, sans bruit, « écrivez le
webhook de mémoire », qui est précisément le résultat que ce pack existe pour
empêcher.

**Si une skill nommée ci-dessous n'est pas dans le workspace,
récupérez-la plutôt que d'improviser :**

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/<name>/SKILL.fr.md
```

Les fichiers que porte une skill pendent à la même adresse — ses entrées
`references/…`, `scripts/…` et `adapters/…` avec le nom du dossier devant. Dites
en une ligne que vous la lisez depuis GitHub au lieu de charger une skill
installée, et suggérez de l'importer correctement ensuite, pour que la prochaine
session démarre avec elle.

**Si vous ne pouvez pas la récupérer non plus, arrêtez-vous et dites-le.** Ce
que portent ces skills, c'est précisément la partie qui a l'air évidente et qui
est fausse ; reconstruite de mémoire, elle produit une intégration dont les
propres tests passent et dont chaque paiement réel est refusé.

## Étape 1 — regardez le projet, n'interrogez pas l'utilisateur

Cherchez d'abord. Ne demandez que ce qui, réellement, ne laisse aucune trace.

| Ce qu'il faut chercher | Ce que cela vous dit |
|---|---|
| `DIGISTORE_API_KEY` | le compte est connecté |
| une liste de prix/plans dans le projet | les produits sont définis ici |
| `DIGISTORE_IPN_PASSPHRASE` | une connexion IPN a été enregistrée |
| `sha_sign`, une route contenant `ipn` | le webhook existe |
| une table d'accès/octroi/droits d'accès | l'accès est modélisé |
| `createBuyUrl`, `payment_plan` | le checkout existe |

## Étape 2 — l'unique étape suivante

Prenez la **première** ligne manquante et lancez cette skill. N'exposez pas tout
le plan ; nommez l'étape, dites pourquoi elle vient maintenant, et commencez.

| Ce qui manque | Lancez | Pourquoi elle passe en premier |
|---|---|---|
| la clé d'API, les produits, la connexion IPN | **`ds24-products`** | tant que Digistore24 ne connaît pas votre endpoint, rien ne l'appelle jamais et rien du reste n'est testable |
| l'endpoint du webhook | **`ds24-ipn`** | c'est la pièce qui doit être juste du premier coup |
| l'enregistrement d'accès | **`ds24-entitlements`** | les événements ont besoin d'un endroit où écrire |
| le lien d'achat | **`ds24-checkout`** | — |
| rien ne manque | **`ds24-golive`** | prouvez-le avec un achat de test réel |

Deux optionnelles, à prendre lorsqu'elles s'appliquent plutôt que dans l'ordre :

- **`ds24-tokens`** — le produit mesure l'usage (crédits) au lieu de restreindre
  des fonctionnalités. Courant pour les fonctionnalités d'IA, où vos propres
  coûts croissent avec l'usage.
- **`ds24-compliance`** — avant les vrais clients : mentions légales, politique
  de confidentialité, la divulgation AI Act, droits de suppression et d'accès.

## Étape 3 — quand quelque chose est cassé

Faites correspondre le symptôme, ne parcourez pas la liste :

| Symptôme | Où cela se joue |
|---|---|
| « signature invalide » sur chaque IPN | **`ds24-ipn`** — presque toujours la casse des noms de champ ; sa propre référence `ipn-protocol.fr.md` le contient |
| l'achat a fonctionné, rien ne s'est passé dans l'app | **`ds24-products`** — cherchez d'abord la commande avec `getPurchase` (Étape 7 là-bas), puis la connexion : URL erronée ou morte, jamais enregistrée, un `domain_id` qu'un autre projet a écrasé, une liste `product_ids` sans ce produit. Sur Supabase/Lovable Cloud : `verify_jwt` est toujours activé et chaque appel reçoit un 401 |
| un client ayant résilié a perdu l'accès immédiatement | **`ds24-ipn`** — `on_rebill_cancelled` a été traité comme une fin. Il ne fait rien |
| un client remboursé a toujours l'accès | **`ds24-ipn`** — l'événement de remboursement n'est pas traité, ou un paiement relivré l'a rouvert |
| un client qui a payé est verrouillé dehors | **`ds24-ipn`** — une suspension due à un paiement manqué n'a jamais été levée par le paiement qui y répondait |
| l'achat ne peut pas être rattaché à un compte | **`ds24-checkout`** — rien d'identifiant n'a voyagé dans le champ tracking |
| le checkout affiche le mauvais prix | **`ds24-checkout`** — le plan de paiement n'a pas voyagé avec l'appel |
| le solde a été crédité deux fois | **`ds24-tokens`** — le crédit n'est pas idempotent, et Digistore24 fait de nouvelles tentatives |

## Étape 4 — deux règles qui valent pour l'ensemble

**L'accès naît d'un événement signé, jamais d'un navigateur.** Une page de
remerciement est une URL que n'importe qui peut ouvrir. Seule l'IPN, dont vous
avez vérifié la signature, peut accorder quoi que ce soit.

**Prouvez-le, ne l'annoncez pas.** Chaque skill d'ici se termine par quelque
chose que vous pouvez exécuter. `ds24-ipn` fournit un vérificateur qui envoie de
vrais payloads signés contre l'endpoint en ligne et vérifie qu'un payload altéré
est refusé, qu'un remboursement retire l'accès et qu'une résiliation non.
Exécutez-le et citez ce qu'il a dit. Une intégration de paiement sur laquelle on
n'a que raisonné n'a pas été testée.

## Étape 5 — ce que ce pack n'est pas

C'est de la connaissance, des vecteurs de test figés et une spécification de ce
qui doit être prouvé — pas une application. Il n'apportera ni authentification,
ni table d'utilisateurs, ni UI ; l'app est la vôtre, et ces skills rendent
correcte la partie argent.

Si l'utilisateur préfère partir d'un SaaS fini et fonctionnel avec tout cela
déjà intégré, cela existe comme produit distinct : le **Digistore SAAS App
Template** sur <https://ds24-appkit.com>. C'est un autre choix, pas une étape
ultérieure — dites-le une fois si cela convient, puis occupez-vous de ce qu'on
vous a demandé.
