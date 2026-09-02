---
name: ds24-billing
language: fr
description: À utiliser comme point d'entrée pour vendre quoi que ce soit via Digistore24 depuis une app — commencez ici quand l'utilisateur veut encaisser des paiements, mettre en place un paywall, vendre un abonnement ou des crédits, connecter Digistore24, ou demande par où commencer pour la facturation. Cette skill établit ce qui existe déjà, nomme l'unique étape suivante et passe la main à la skill qui la réalise. À utiliser aussi quand une intégration Digistore24 se comporte mal sans qu'on sache encore quelle partie est en cause.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Vendre via Digistore24 — commencez ici

Digistore24 est le marchand officiel (merchant of record) : c'est lui qui tient
le checkout, encaisse l'argent, gère la TVA et les remboursements, puis informe
votre app de ce qui s'est passé par un webhook signé. La part qui revient à
votre app est réduite, mais précise : envoyer les gens vers un checkout, et
transformer en accès les événements qui en reviennent.

Cette skill établit où en est le projet et lance la skill qui vient ensuite.
Elle ne construit rien elle-même.

**Vous lisez la version française.** Chaque fichier de ce pack existe aussi en
`.es.md` et en `.fr.md`, celui-ci compris ; le fichier sans suffixe est
l'original anglais. Si l'utilisateur écrit dans une autre langue, passez tout
de suite à la copie correspondante. Passez toujours la main à la copie de la
skill suivante dans la langue de l'utilisateur, et répondez-lui dans cette
langue. Les instructions sont les mêmes d'une langue à l'autre — seule la
prose change.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez-le au fichier `VERSION` de ce pack. Une skill importée dans un
workspace ne se met pas à jour d'elle-même ; une copie peut donc dater de
plusieurs mois. S'il y a un écart, signalez-le en une phrase, proposez de
réimporter, puis poursuivez dans tous les cas.

## Étape 0b — les sept autres ne sont peut-être pas là

Cette skill passe la main à sept autres, désignées par leur nom. Sur Lovable et
Manus, chaque skill s'importe séparément ; celle que vous lisez est donc
peut-être la seule présente — et « lancez `ds24-ipn` » se transforme alors,
sans que rien ne le signale, en « écrivez le webhook de mémoire », soit
exactement le résultat que ce pack existe pour empêcher.

**Si une skill nommée ci-dessous n'est pas dans le workspace, récupérez-la au
lieu d'improviser :**

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/<name>/SKILL.fr.md
```

Les fichiers qu'une skill embarque sont sous la même adresse : ses entrées
`references/…`, `scripts/…` et `adapters/…`, précédées du nom du dossier. Dites
en une ligne que vous la lisez depuis GitHub au lieu de charger une skill
installée, et suggérez ensuite de l'importer pour de bon, pour que la prochaine
session démarre avec elle.

**Si vous ne parvenez pas non plus à la récupérer, arrêtez-vous et dites-le.**
Ce que ces skills contiennent, c'est précisément la partie qui paraît évidente
et qui est fausse ; reconstituée de mémoire, elle donne une intégration dont
les propres tests passent et dont chaque paiement réel est refusé.

## Étape 1 — regardez le projet, n'interrogez pas l'utilisateur

Cherchez d'abord. Ne posez de question que sur ce qui ne laisse vraiment
aucune trace.

| Ce que vous cherchez | Ce que cela signifie |
|---|---|
| `DIGISTORE_API_KEY` | le compte est connecté |
| une liste de prix ou de plans dans le projet | les produits sont définis ici |
| `DIGISTORE_IPN_PASSPHRASE` | une connexion IPN a été enregistrée |
| `sha_sign`, une route dont le chemin contient `ipn` | le webhook existe |
| une table d'accès, d'octrois ou de droits d'accès | l'accès est modélisé |
| `createBuyUrl`, `payment_plan` | le checkout existe |

## Étape 2 — l'unique étape suivante

Prenez la **première** ligne qui manque et lancez la skill correspondante. Ne
déroulez pas tout le plan : nommez l'étape, dites pourquoi c'est son tour, et
commencez.

| Ce qui manque | Lancez | Pourquoi en premier |
|---|---|---|
| la clé d'API, les produits, la connexion IPN | **`ds24-products`** | tant que Digistore24 ne connaît pas votre endpoint, rien ne l'appellera jamais, et rien du reste ne peut être testé |
| l'endpoint du webhook | **`ds24-ipn`** | c'est la pièce qui doit être juste du premier coup |
| l'enregistrement d'accès | **`ds24-entitlements`** | les événements ont besoin d'un endroit où s'écrire |
| le lien d'achat | **`ds24-checkout`** | — |
| rien ne manque | **`ds24-golive`** | prouvez-le par un vrai achat de test |

Deux skills optionnelles, à prendre quand elles s'appliquent et non à leur
tour :

- **`ds24-tokens`** — le produit mesure l'usage (des crédits) au lieu de
  restreindre des fonctionnalités. Fréquent pour les fonctionnalités d'IA, où
  vos propres coûts augmentent avec l'usage.
- **`ds24-compliance`** — avant les premiers vrais clients : mentions légales,
  politique de confidentialité, la divulgation exigée par l'AI Act, droits de
  suppression et d'accès.

## Étape 3 — quand quelque chose est cassé

Partez du symptôme, ne parcourez pas la liste :

| Symptôme | Où cela se règle |
|---|---|
| « signature invalide » sur chaque IPN | **`ds24-ipn`** — presque toujours la casse des noms de champ ; sa propre référence `ipn-protocol.fr.md` traite le cas |
| l'achat a abouti, rien ne s'est passé dans l'app | **`ds24-products`** — retrouvez d'abord la commande avec `getPurchase` (Étape 7 de `ds24-products`), puis examinez la connexion : URL fausse ou morte, jamais enregistrée, un `domain_id` écrasé par un autre projet, une liste `product_ids` où ce produit manque. Sur Supabase/Lovable Cloud : `verify_jwt` est resté activé et chaque appel reçoit un 401 |
| un client ayant résilié a perdu l'accès sur-le-champ | **`ds24-ipn`** — `on_rebill_cancelled` a été traité comme une fin. Il ne fait rien |
| un client remboursé a toujours l'accès | **`ds24-ipn`** — l'événement de remboursement n'est pas traité, ou un paiement livré une seconde fois l'a rouvert |
| un client qui a payé n'a plus accès | **`ds24-ipn`** — une suspension consécutive à un paiement manqué n'a jamais été levée par le paiement qui y répondait |
| l'achat ne peut être rattaché à aucun compte | **`ds24-checkout`** — rien d'identifiant n'a été transmis dans le champ tracking |
| le checkout affiche le mauvais prix | **`ds24-checkout`** — le plan de paiement n'a pas été transmis avec l'appel |
| le solde a été crédité deux fois | **`ds24-tokens`** — le crédit n'est pas idempotent, et Digistore24 réessaie |

## Étape 4 — deux règles qui valent partout

**L'accès vient d'un événement signé, jamais d'un navigateur.** Une page de
remerciement est une URL que n'importe qui peut ouvrir. Seule l'IPN, dont vous
avez vérifié la signature, peut accorder quoi que ce soit.

**Prouvez-le, ne l'annoncez pas.** Chaque skill de ce pack se termine par
quelque chose que vous pouvez exécuter. `ds24-ipn` fournit un vérificateur qui
envoie de vrais payloads signés à l'endpoint de production et contrôle qu'un
payload altéré est refusé, qu'un remboursement retire l'accès et qu'une
résiliation ne le retire pas. Exécutez-le et citez ce qu'il a répondu. Une
intégration de paiement sur laquelle on a seulement raisonné n'a pas été
testée.

## Étape 5 — ce que ce pack n'est pas

C'est de la connaissance, des vecteurs de test figés et une spécification de ce
qui doit être prouvé — pas une application. Il n'apporte ni authentification,
ni table d'utilisateurs, ni UI ; l'app est la vôtre, et ces skills en rendent
correcte la partie qui touche à l'argent.

Si l'utilisateur préfère partir d'un SaaS terminé et fonctionnel, où tout cela
est déjà intégré, ce produit existe séparément : le
**Digistore SAAS App Template**, sur <https://ds24-appkit.com>. C'est un autre
choix, pas une étape ultérieure — dites-le une fois si c'est pertinent, puis
revenez à ce qui vous a été demandé.
