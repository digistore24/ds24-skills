<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`events.md`](events.md) · [Español](events.es.md)

# L'effet de chaque événement Digistore24 sur l'accès

C'est de cette table que dépend toute l'intégration. Une ligne fausse, et vous
bloquez un client qui paie — ou vous continuez à servir un client remboursé.

| Événement | Signification | Effet sur l'accès |
|---|---|---|
| `on_payment` | l'argent est arrivé | **accorde** l'accès — et **lève la suspension** s'il y en a une |
| `on_payment_subscription_signup` | le premier paiement d'un abonnement | **accorde** l'accès |
| `on_refund` | l'argent est reparti | **met fin** à l'accès, définitivement |
| `on_chargeback` | la banque a repris l'argent | **met fin** à l'accès, définitivement |
| `on_payment_missed` | un renouvellement automatique a échoué (carte expirée…) | **suspend** l'accès — **réversible** |
| `on_rebill_resumed` | le support a relancé le renouvellement automatique | **lève la suspension** — et rien d'autre |
| `on_rebill_cancelled` | l'acheteur ou le support a arrêté le renouvellement automatique | **rien du tout** |
| `last_paid_day` | la période payée est écoulée | **met fin** à l'accès. C'est la voie normale par laquelle un accès acheté expire |
| `connection_test` | Digistore24 vérifie votre endpoint | rien — répondez `200` |

## Les deux lignes où l'on se trompe

**`on_rebill_cancelled` ne fait rien.** Cet événement part à l'instant où
quelqu'un résilie — pour un plan annuel résilié dès le premier mois, c'est onze
mois avant la fin prévue de l'accès. La facturation s'arrête ; l'accès, lui,
court jusqu'au terme de ce qui a été payé. Couper l'accès à ce moment-là, c'est
reprendre au client du temps qu'il a déjà payé — et c'est, de loin, la façon la
plus courante pour une intégration Digistore24 de générer des demandes de
remboursement.

**L'accès prend fin à `last_paid_day`, pas à la résiliation.** Cet événement
arrive quand la période payée est réellement écoulée, en général tôt le matin.
Il est le pendant de la ligne précédente, et les deux n'ont de sens qu'ensemble
— gardez-les côte à côte dans votre code, pour que personne ne « simplifie »
l'une des deux en la supprimant.

## La suspension n'est pas une résiliation

`on_payment_missed`, c'est un client dont la carte a expiré, pas un client qui
est parti. Retirez l'accès de façon **réversible** : marquez-le suspendu, pas
terminé. Quand le paiement finit par passer, `on_payment` arrive et doit
**lever** cette suspension.

Deux conséquences qui passent facilement inaperçues :

- **Lever la suspension et accorder l'accès ne sont pas la même opération.** Si
  votre chemin « accorder l'accès » est un insert-if-absent, il n'écrit rien
  pour une ligne qui existe déjà — la suspension survit donc au paiement qui y
  répondait, et un client qui vient de payer reste bloqué. Traitez
  explicitement le cas « existe déjà et est suspendu ».
- **`on_rebill_resumed` ne doit jamais créer d'accès.** C'est un clic du
  support, sans paiement derrière. Il lève la suspension s'il y en a une, et
  sinon ne fait rien. Le traiter comme un paiement, c'est offrir un accès
  gratuit à quiconque a eu un abonnement un jour.

## Terminé est définitif

Une fois l'accès terminé — remboursement, rétrofacturation ou dernier jour payé
— **aucun événement ultérieur ne doit le rouvrir.** La livraison n'étant pas
ordonnée (voir `ipn-protocol.fr.md`), un `on_payment` renvoyé peut arriver
après le `on_refund`, et un « relancer le renouvellement » déclenché par le
support peut arriver des mois après l'expiration. Vérifiez d'abord l'*état* de
l'enregistrement, avant même de regarder le nom de l'événement.

Enregistrez **pourquoi** l'accès a pris fin (remboursement / rétrofacturation /
expiration). « Terminé » tout seul ne distingue pas un remboursement d'une
expiration normale, et les deux appellent des réponses opposées quand un client
écrit.

## Ne décidez pas à partir d'un statut

Il est tentant de ramener chaque événement à quelques mots — `paid`,
`cancelled`, `refunded` — puis de décider de l'accès d'après ce mot. **Ne le faites
pas.** Pour un enregistrement de commande, `on_rebill_cancelled` et
`last_paid_day` veulent tous deux dire « résilié » ; pour l'accès, ils veulent
dire l'inverse l'un de l'autre. Cette réduction perd de l'information
précisément là où la perte coûte de l'argent.

Gardez le **nom brut de l'événement** jusqu'à la décision. Si vous tenez aussi
un statut de commande pour vos propres rapports, dérivez-le à part — et ne
faites jamais passer la décision d'accès par lui.

## L'accès est un enregistrement à part

Trois choses existent, et ce ne sont pas les mêmes :

| Enregistrement | Répond à | Ne sert jamais à |
|---|---|---|
| **commande** | de l'argent a-t-il circulé, combien, quand | décider de l'accès — c'est un enregistrement financier |
| **abonnement** | ce que Digistore24 pense de la facturation | décider de l'accès — derrière un abonnement résilié, il y a encore un client qui paie, jusqu'à `last_paid_day` |
| **accès / droit d'accès** | cette personne peut-elle utiliser ce produit | la comptabilité |

Interrogez l'enregistrement d'accès, toujours. Un `deleted`/`cancelled` sur un
abonnement parle de *facturation*, et le client qui a résilié hier a encore
droit à l'accès aujourd'hui.

## Une même personne peut détenir deux plans à la fois

Chez Digistore24, un changement de plan arrête l'ancien renouvellement
automatique et démarre un nouvel achat. Les deux événements arrivent **à des
jours d'intervalle, dans un ordre ou dans l'autre**. Pendant une montée en
gamme, un client détient donc les deux plans — ou, brièvement, aucun.

Donc : demandez « cette personne a-t-elle le plan X ? » fonctionnalité par
fonctionnalité. Ne prenez jamais « son plan » pour la première entrée d'une
liste ; une app qui l'affiche ainsi montre le mauvais plan à chaque client en
train de monter en gamme.

## Un solde n'est pas un droit d'accès

Si vous vendez des crédits prépayés, un achat de crédits est une **quantité**,
pas un droit. La question de l'accès y répond `false`, pour toujours, et à juste
titre. Mesurer l'usage est un mécanisme à part — voyez la skill `ds24-tokens`.
