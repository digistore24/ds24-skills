---
name: ds24-entitlements
language: fr
description: À utiliser pour décider ce qu'un client payant peut réellement utiliser — l'enregistrement d'accès dans lequel écrivent les événements Digistore24, et l'unique fonction que le reste de l'app interroge. Couvre pourquoi l'accès n'est ni la table des commandes ni la table des abonnements, comment restreindre une page ou une fonctionnalité, les montées en gamme où quelqu'un détient deux plans à la fois, et un plan en pause après un paiement manqué. À utiliser dès que l'utilisateur demande comment vérifier si quelqu'un a payé, comment verrouiller une fonctionnalité derrière un plan, ou signale qu'un client ayant résilié a perdu l'accès trop tôt.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Ce qu'un client payant peut utiliser

Il y a une seule question que pose l'app — *cette personne peut-elle utiliser
cette chose ?* — et elle doit avoir exactement une réponse, à un seul endroit.
Chaque version de ceci qui tourne mal a mal tourné en interrogeant une autre
table.

## Étape 0 — est-ce déjà là ?

Cherchez une table d'accès, de droits d'accès ou d'octrois, ou une vérification
comme `hasPlan(...)`. Si quelque chose existe, n'en construisez pas une
deuxième — confrontez-la à l'Étape 2 et à l'Étape 3.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez-le au `VERSION` de ce pack. Signalez tout écart en une phrase, puis
continuez.

## Étape 1 — trois enregistrements, et un seul répond

| Enregistrement | Répond | Ne doit jamais décider de l'accès |
|---|---|---|
| **commande** | si l'argent a bougé, combien, quand | c'est un enregistrement financier |
| **abonnement** | ce que Digistore24 croit à propos de la facturation | derrière un abonnement résilié il y a toujours un client payant jusqu'à la fin de la période payée |
| **accès / octroi d'accès** | **si cette personne peut utiliser ce produit** | — |

La ligne du milieu est le piège. Quelqu'un résilie au jour 3 d'un plan annuel ;
Digistore24 signale l'abonnement comme résilié immédiatement ; il reste onze
mois au client. Une app qui restreint selon le statut de l'abonnement le
verrouille dehors l'après-midi même, et la demande de remboursement est
entièrement justifiée.

**Tenez un enregistrement d'accès séparé.** Forme minimale :

```
access_grants
  member_id      qui
  product_key    quoi
  source         'purchase' | 'manual'
  order_id       de quel achat il provient (null pour manual)
  suspended_at   posé par on_payment_missed, levé par on_payment  (réversible)
  ended_at       posé par refund / chargeback / last_paid_day     (terminal)
  ended_reason   'refund' | 'chargeback' | 'lastPaidDay' | 'revoked'
  access_until   une DATE de fin, pour les octrois manuels uniquement — null pour les achats
  unique (member_id, product_key, order_id)
```

L'IPN le maintient. Rien d'autre n'y écrit, hormis un octroi d'accès manuel
délibéré. Quel événement fait quoi, c'est la table des événements de la skill
**`ds24-ipn`** — lisez-la là-bas avant d'écrire quoi que ce soit de ceci. Si
cette skill n'est pas installée, celle-ci ne peut pas être terminée
correctement : installez-la aussi.

## Étape 2 — une seule fonction, interrogée par fonctionnalité

```
hasAccess(memberId, productKey) -> boolean
```

Vrai lorsqu'une ligne existe pour cette paire avec `ended_at IS NULL` **et**
`suspended_at IS NULL` **et** (`access_until IS NULL` ou `access_until > now`).

Chaque restriction de l'app l'appelle. Aucune page ne lit la table des octrois
elle-même, et aucune page ne lit les commandes ou les abonnements pour décider
quoi que ce soit.

**Interrogez-la par fonctionnalité, pas une fois par utilisateur.** Un changement
de plan Digistore24 arrête l'ancien renouvellement automatique et démarre un
nouvel achat, et les deux événements arrivent **à des jours d'intervalle, dans
un ordre ou dans l'autre** — pendant une montée en gamme, un client détient donc
les *deux* plans, ou brièvement *aucun*. Le code qui prend « leur plan » pour la
première entrée d'une liste affiche le mauvais plan à tout client qui monte en
gamme.

## Étape 3 — trois règles qui ne sont pas évidentes

**Un paiement manqué ne doit pas se lire comme une fermeture de compte.**
Lorsque `suspended_at` est posé, `hasAccess` dit false à juste titre — et le
client voit un produit qui s'est simplement volatilisé. Donnez à l'UI une
seconde question, d'affichage uniquement (« est-ce en pause ? »), et dites
*votre accès est en pause, le dernier paiement n'est pas passé*. Jamais rien du
tout.

**`access_until` a besoin d'un fuseau horaire UTC explicite à l'affichage.**
Stockez la fin de la journée qu'il couvre et affichez-le épinglé à UTC — sinon
tout lecteur en avance sur UTC lit le jour suivant. Donnez à `null` une vraie
phrase (« pas de date de fin »), jamais une cellule vide.

**Un solde prépayé n'est pas un droit d'accès.** `hasAccess` répond false pour
un pack de crédits à jamais, et c'est correct : un plan est un droit, un solde
est une quantité. Voir **`ds24-tokens`**.

## Étape 4 — les octrois manuels, parce que le support en a besoin

Quelqu'un paiera hors du système, ou un achat ne parviendra pas à être attribué.
Permettez à un opérateur d'accorder l'accès à la main, avec `source = 'manual'`
et un motif écrit.

Deux limites qu'il vaut la peine d'intégrer dès le départ :

- **Seuls les octrois manuels peuvent être révoqués à la main.** L'accès acheté
  se termine par un événement Digistore24, jamais par un clic — sinon le support
  peut retirer quelque chose qu'un client a payé sans aucun remboursement
  associé.
- **Imposez cela dans l'écriture elle-même**, pas seulement dans l'UI qui cache
  le bouton. Tout handler est un endpoint HTTP à part entière.

## Étape 5 — prouvez-le

Exécutez le vérificateur de la skill **`ds24-ipn`** avec `--probe` pointé sur un
petit endpoint adossé à `hasAccess`. C'est exactement ce que testent ses
vérifications d'accès : un remboursement retire l'accès, une résiliation non, un
paiement manqué suspend de façon réversible, et un paiement relivré ne ressuscite
pas une commande remboursée.

Si le vérificateur dit `SKIP` sur celles-ci, rien de ce qui précède n'a été
testé — dites-le plutôt que de l'annoncer comme fait.

## Étape 6 — la suite

- **`ds24-tokens`** — si l'usage est mesuré plutôt que restreint.
- **`ds24-golive`** — l'achat de test réel.
- **`ds24-compliance`** — ce que vous stockez désormais sur les personnes, et à
  quoi cela vous oblige.

Dites laquelle vous commencez, et commencez-la.
