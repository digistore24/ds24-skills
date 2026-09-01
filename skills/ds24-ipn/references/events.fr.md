<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`events.md`](events.md) · [Español](events.es.md)

# Ce que chaque événement Digistore24 fait à l'accès

C'est la table à laquelle toute l'intégration est suspendue. Trompez-vous sur
une ligne et soit vous verrouillez dehors un client qui paie, soit vous
continuez à servir un client remboursé.

| Événement | Ce que cela signifie | Ce que cela fait à l'accès |
|---|---|---|
| `on_payment` | l'argent est arrivé | **accorde** l'accès — et **lève une suspension** s'il y en a une |
| `on_payment_subscription_signup` | le premier paiement d'un abonnement | **accorde** l'accès |
| `on_refund` | l'argent est reparti | **met fin** à l'accès, définitivement |
| `on_chargeback` | la banque l'a repris | **met fin** à l'accès, définitivement |
| `on_payment_missed` | un renouvellement a échoué (carte expirée…) | **suspend** l'accès — **réversible** |
| `on_rebill_resumed` | le support a redémarré le renouvellement automatique | **lève une suspension** — et rien d'autre |
| `on_rebill_cancelled` | l'acheteur ou le support a arrêté le renouvellement automatique | **rien du tout** |
| `last_paid_day` | la période payée est terminée | **met fin** à l'accès. C'est ainsi que l'accès acheté expire normalement |
| `connection_test` | Digistore24 valide votre endpoint | rien — répondez `200` |

## Les deux lignes sur lesquelles on se trompe

**`on_rebill_cancelled` ne fait rien.** Il est envoyé au moment où quelqu'un
résilie, ce qui, pour un plan annuel résilié au premier mois, se situe onze mois
avant la fin prévue de l'accès. La facturation s'arrête ; l'accès continue
jusqu'à épuisement de ce qui a été payé. Mettre fin à l'accès ici retire au
client du temps qu'il a déjà payé — et c'est de loin la manière la plus courante
dont une intégration Digistore24 produit des demandes de remboursement.

**L'accès prend fin à `last_paid_day`, pas à la résiliation.** Cet événement
arrive quand la période payée est réellement terminée, en général tôt le matin.
C'est le pendant de la ligne ci-dessus, et les deux n'ont de sens qu'en paire —
gardez-les côte à côte dans votre code pour que personne n'en « simplifie » une
jusqu'à la faire disparaître.

## La suspension n'est pas la résiliation

`on_payment_missed`, c'est un client dont la carte a expiré, pas un client qui
est parti. Retirez l'accès de façon **réversible** : marquez-le comme suspendu,
ne le marquez pas comme terminé. Quand le paiement passe, `on_payment` arrive et
doit **lever** cette suspension.

Deux conséquences faciles à manquer :

- **La levée n'est pas la même opération que l'octroi.** Si votre chemin
  « accorder l'accès » est un insert-si-absent, il n'écrit rien pour une ligne
  qui existe déjà — la suspension survit donc au paiement qui y répondait, et un
  client qui vient de payer reste verrouillé dehors. Traitez explicitement le
  cas « existe déjà et est suspendu ».
- **`on_rebill_resumed` ne doit jamais créer d'accès.** C'est un clic du support
  sans aucun paiement derrière. Il lève une suspension s'il y en a une, et sinon
  il ne fait rien. Le traiter comme un paiement distribue un accès gratuit à
  quiconque a eu un jour un abonnement.

## Terminé, c'est pour toujours

Une fois que l'accès a pris fin — remboursement, rétrofacturation ou dernier
jour payé — **aucun événement ultérieur ne peut le rouvrir.** Comme la livraison
est sans ordre (voyez `ipn-protocol.fr.md`), un `on_payment` relivré peut
arriver après le `on_refund`, et un « redémarrer le renouvellement
automatique » du support peut arriver des mois après l'expiration.
Protégez-vous sur l'*état* de l'enregistrement, avant même de regarder le nom de
l'événement.

Enregistrez **pourquoi** il a pris fin (remboursement / rétrofacturation /
expiration). « Terminé » tout seul ne distingue pas un remboursement d'une
expiration normale, et ces deux-là appellent des réponses opposées quand un
client écrit.

## Ne décidez pas à partir d'un statut

Il est tentant de projeter chaque événement sur un petit ensemble de mots —
`paid`, `cancelled`, `refunded` — puis de décider l'accès à partir de ce mot.
**Ne le faites pas.** `on_rebill_cancelled` et `last_paid_day` signifient tous
les deux « annulé » pour un enregistrement de commande, et ils signifient le
contraire pour l'accès. La projection perd de l'information exactement là où la
perte coûte de l'argent.

Conservez le **nom brut de l'événement** jusqu'à la décision elle-même. Si vous
gardez aussi un statut de commande pour vos propres rapports, dérivez-le
séparément — ne faites jamais passer la décision d'accès par lui.

## L'accès est un enregistrement à part entière

Trois choses existent et elles ne sont pas la même :

| Enregistrement | Répond à | Ne sert jamais à |
|---|---|---|
| **commande** | de l'argent a-t-il bougé, combien, quand | décider de l'accès — c'est un enregistrement financier |
| **abonnement** | ce que Digistore24 croit au sujet de la facturation | décider de l'accès — derrière un abonnement résilié il y a encore un client qui paie jusqu'à `last_paid_day` |
| **accès / droit d'accès** | cette personne peut-elle utiliser ce produit | la comptabilité |

Interrogez toujours l'enregistrement d'accès. `deleted`/`cancelled` sur un
abonnement est une affirmation sur la *facturation*, et le client qui a résilié
hier a encore droit à l'accès aujourd'hui.

## Une même personne peut détenir deux plans à la fois

Un changement de plan chez Digistore24 arrête l'ancien renouvellement
automatique et démarre un nouvel achat. Les deux événements arrivent **à des
jours d'intervalle, dans un ordre ou dans l'autre**. Pendant une montée en
gamme, un client détient donc les deux plans — ou, brièvement, aucun des deux.

Donc : demandez « cette personne a-t-elle le plan X ? » pour chaque
fonctionnalité. Ne prenez jamais « son plan » pour la première entrée d'une
liste ; une app qui l'affiche ainsi montre le mauvais plan à chaque client qui
monte en gamme.

## Un solde n'est pas un droit d'accès

Si vous vendez des crédits prépayés, un achat de crédits est une **quantité**,
pas un droit. La question de l'accès y répond `false`, pour toujours, et à juste
titre. Mesurer l'usage est un mécanisme distinct — voyez la skill
`ds24-tokens`.
