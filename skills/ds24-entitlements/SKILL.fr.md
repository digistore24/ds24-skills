---
name: ds24-entitlements
language: fr
description: À utiliser pour décider ce qu'un client payant a réellement le droit d'utiliser — l'enregistrement d'accès que les événements Digistore24 alimentent, et la seule fonction que le reste de l'app interroge. Explique pourquoi l'accès n'est ni la table des commandes ni celle des abonnements, comment restreindre une page ou une fonctionnalité, les montées en gamme pendant lesquelles quelqu'un détient deux plans à la fois, et le plan mis en pause après un paiement manqué. À utiliser aussi dès que l'utilisateur demande comment vérifier que quelqu'un a payé, comment réserver une fonctionnalité à un plan, ou signale qu'un client ayant résilié a perdu l'accès trop tôt.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Ce qu'un client payant peut utiliser

L'app ne pose qu'une seule question — *cette personne a-t-elle le droit
d'utiliser ceci ?* — et cette question doit avoir exactement une réponse, à un
seul endroit. Chaque fois que cela a mal tourné, c'est parce que la question
avait été posée à une autre table.

## Étape 0 — est-ce déjà là ?

Cherchez une table d'accès, de droits d'accès ou d'octrois, ou une vérification
du type `hasPlan(...)`. S'il en existe une, n'en construisez pas une seconde :
confrontez-la aux Étapes 2 et 3.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez-le au `VERSION` de ce pack. S'ils diffèrent, dites-le en une
phrase, puis poursuivez.

## Étape 1 — trois enregistrements, et un seul répond

| Enregistrement | Répond à la question | Ne doit jamais décider de l'accès |
|---|---|---|
| **commande** | de l'argent a-t-il circulé, combien, quand | c'est une pièce comptable |
| **abonnement** | l'état de la facturation tel que Digistore24 le voit | derrière un abonnement résilié, il y a encore un client qui a payé, jusqu'à la fin de la période payée |
| **accès / octroi** | **cette personne a-t-elle le droit d'utiliser ce produit** | — |

Le piège, c'est la ligne du milieu. Un client résilie au troisième jour d'un
plan annuel ; Digistore24 signale aussitôt l'abonnement comme résilié ; il
reste onze mois au client. Une app qui restreint l'accès d'après le statut de
l'abonnement le bloque l'après-midi même — et la demande de remboursement qui
suit est tout à fait justifiée.

**Tenez un enregistrement d'accès à part.** Structure minimale :

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

C'est l'IPN qui tient cette table à jour. Rien d'autre n'y écrit, à
l'exception d'un octroi manuel délibéré. Quel événement fait quoi, c'est la
table des événements de la skill **`ds24-ipn`** qui le dit — lisez-la là-bas
avant d'écrire la moindre ligne de ceci. Si cette skill n'est pas installée,
celle-ci ne peut pas être menée à bien correctement : installez-la également.

## Étape 2 — une seule fonction, interrogée par fonctionnalité

```
hasAccess(memberId, productKey) -> boolean
```

Renvoie vrai lorsqu'il existe une ligne pour cette paire avec `ended_at IS NULL`
**et** `suspended_at IS NULL` **et** (`access_until IS NULL` ou
`access_until > now`).

Chaque restriction d'accès de l'app passe par elle. Aucune page ne lit
directement la table des octrois, et aucune ne consulte les commandes ou les
abonnements pour décider quoi que ce soit.

**Interrogez-la fonctionnalité par fonctionnalité, pas une fois par
utilisateur.** Chez Digistore24, un changement de plan arrête l'ancien
renouvellement automatique et déclenche un nouvel achat, et les deux événements
arrivent **à plusieurs jours d'écart, dans un ordre ou dans l'autre** — pendant
une montée en gamme, un client détient donc les *deux* plans, ou, brièvement,
*aucun des deux*. Un code qui prend pour « son plan » la première entrée d'une
liste affiche le mauvais plan à tous les clients en train de monter en gamme.

## Étape 3 — trois règles qui ne vont pas de soi

**Un paiement manqué ne doit pas ressembler à une fermeture de compte.** Quand
`suspended_at` est renseigné, `hasAccess` répond faux, et c'est juste — mais
pour le client, le produit a tout simplement disparu. Donnez à l'interface une
seconde question, réservée à l'affichage (« est-ce en pause ? »), et dites-lui
*votre accès est en pause, le dernier paiement n'est pas passé*. Jamais rien
du tout.

**À l'affichage, `access_until` exige un fuseau horaire UTC explicite.** Stockez
la fin du jour qu'il couvre et affichez-le en le fixant à UTC — sinon, tout
lecteur dont l'heure locale est en avance sur UTC lit le lendemain. Donnez à
`null` une vraie phrase (« sans date de fin »), jamais une cellule vide.

**Un solde prépayé n'est pas un droit d'accès.** Pour un pack de crédits,
`hasAccess` répondra toujours faux, et c'est la bonne réponse : un plan est un
droit, un solde est une quantité. Voir **`ds24-tokens`**.

## Étape 4 — les octrois manuels, parce que le support en a besoin

Il arrivera qu'un client paie en dehors du système, ou qu'un achat ne puisse
pas être attribué. Permettez à un opérateur d'accorder l'accès à la main, avec
`source = 'manual'` et un motif rédigé.

Deux garde-fous à prévoir dès le départ :

- **Seuls les octrois manuels peuvent être révoqués à la main.** Un accès acheté
  prend fin sur un événement Digistore24, jamais sur un clic — sans quoi le
  support peut retirer à un client ce qu'il a payé, sans qu'aucun remboursement
  n'y soit associé.
- **Faites respecter cette règle dans l'écriture elle-même**, pas seulement dans
  l'interface qui masque le bouton. Tout handler est aussi un endpoint HTTP à
  part entière.

## Étape 5 — prouvez-le

Lancez le vérificateur de la skill **`ds24-ipn`** avec `--probe` pointé vers un
petit endpoint qui s'appuie sur `hasAccess`. C'est précisément ce que ses
vérifications d'accès mesurent : un remboursement retire l'accès, une
résiliation ne le retire pas, un paiement manqué suspend de façon réversible, et
un paiement renvoyé une seconde fois ne fait pas revivre une commande
remboursée.

Si le vérificateur répond `SKIP` sur ces points, rien de ce qui précède n'a été
testé — dites-le, au lieu de présenter le travail comme terminé.

## Étape 6 — la suite

- **`ds24-tokens`** — si l'usage est mesuré plutôt que restreint.
- **`ds24-golive`** — l'achat de test réel.
- **`ds24-compliance`** — ce que vous stockez désormais sur des personnes, et ce
  à quoi cela vous oblige.

Dites laquelle vous entamez, et entamez-la.
