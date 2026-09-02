---
name: ds24-checkout
language: fr
description: À utiliser pour construire le bouton d'achat, la page de tarifs ou le lien de checkout d'un produit Digistore24 — créer une URL d'achat signée avec createBuyUrl, y attacher le prix sous forme de plan de paiement, faire suivre l'identité de l'acheteur jusqu'à l'IPN, et la page de remerciement. À utiliser dès que l'utilisateur parle d'un lien d'achat, d'un checkout, d'une page de tarifs, demande « comment le client paie-t-il ? », ou signale un achat arrivé sans que personne puisse dire à qui il appartient.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Le lien de checkout

Un checkout Digistore24 est une **URL signée, à courte durée de vie**, que l'on
crée par l'API et vers laquelle on envoie l'acheteur. Ce n'est pas un lien
statique qui contiendrait un id de produit.

## Étape 0 — est-ce déjà là ?

Cherchez dans le projet `createBuyUrl`, `payment_plan` ou une page de tarifs qui
renvoie déjà vers Digistore24. Si c'est déjà en place, ne le reconstruisez pas :
confrontez l'existant à l'Étape 3 et à l'Étape 4, et ne corrigez que ce qui est
faux.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez-le au `VERSION` de ce pack. En cas d'écart, signalez-le en une
phrase, puis poursuivez.

## Étape 1 — l'appel

```
POST https://www.digistore24.com/api/call/createBuyUrl/format/json
Header: X-DS-API-KEY: <la clé>
```

Le corps (form-encoded), réduit aux paramètres qui comptent :

```
product_id                              = 512345
valid_until                             = 24h
payment_plan[first_amount]              = 47.00
payment_plan[other_amounts]             = 47.00
payment_plan[currency]                  = EUR
payment_plan[number_of_installments]    = 0        # 0 = abonnement à durée indéterminée, 1 = paiement unique
payment_plan[first_billing_interval]    = 1_month  # à omettre complètement pour un paiement unique
payment_plan[other_billing_intervals]   = 1_month
```

**Le prix part ici, au moment de l'achat — il n'est pas stocké sur le produit.**
Digistore24 ignore `data[amount]` sur le produit lui-même, et un plan de paiement
stocké ne sait porter ni bon de réduction, ni essai gratuit, ni montée en gamme,
ni commission d'affiliation par lien. Prenez les montants dans l'unique liste de
prix de votre projet (voir **`ds24-products`**).

**`product_id` fixe aussi la LANGUE du formulaire de commande — choisissez-le
d'après la langue de l'acheteur.** Un produit Digistore24 n'a qu'une seule
langue, et aucun paramètre de cet appel ne permet de la remplacer (relisez le
corps ci-dessus : `buyer`, `payment_plan`, `tracking`, `urls`, `placeholders`,
`settings`, `addons` — pas de langue nulle part). Une app multilingue garde donc
**un id de produit par langue** pour chaque offre, et c'est ici, juste avant
l'appel, qu'elle fait correspondre la langue du visiteur à l'un d'eux. Si tout le
monde est envoyé vers le même id, la moitié des acheteurs se retrouvent devant un
formulaire dans la mauvaise langue, au moment précis où on leur demande de payer.
La skill **`ds24-products`** décrit la forme de la liste de prix et donne la
règle en entier.

La réponse est une URL. **Mettez-la en cache par offre** : elle reste valable
pendant la fenêtre `valid_until`, et en recréer une à chaque affichage de page,
c'est faire dépendre chaque affichage de votre page de tarifs d'un aller-retour
vers Digistore24.

⚠️ **La clé de cache doit alors inclure la langue**, et pas seulement la clé
de l'offre. Une seule ligne par clé, et l'URL allemande et l'URL anglaise se
chassent l'une l'autre à chaque affichage de page ; entre deux, le cache sert la
page de checkout d'une langue à l'acheteur de l'autre. `"<offerKey>:<language>"`
suffit.

🚨 **Et ne mettez jamais en cache une URL qui porte l'identité d'un acheteur.**
L'Étape 2 place l'id du membre connecté dans `tracking[custom]`, or un cache
indexé par offre ignore le membre : l'identité du premier acheteur connecté est
alors servie à tous ceux qui ouvrent la page après lui, et chacun de *leurs*
paiements arrive attribué à ce premier membre. Rien ne tombe en panne pendant ce
temps : la page s'affiche, le checkout s'ouvre, l'argent circule.

Une page de tarifs a donc **deux chemins**, et non deux variantes d'un même
chemin :

- **Visiteur déconnecté → l'URL partagée, celle du cache.** Aucune identité
  dedans, sûre pour tout le monde, aucun aller-retour vers Digistore24 pendant
  l'affichage de la page.
- **Membre connecté → une URL construite au moment du clic**, avec l'identité de
  ce membre dedans, utilisée une seule fois et **jamais écrite dans le cache**.

Pour choisir entre les deux, regardez le **contenu** du champ tracking, pas sa
simple présence : un marqueur qui désigne un *forfait* peut être partagé, un
marqueur qui désigne une *personne* ne le peut pas. Se contenter de demander
« le tracking est-il renseigné ? » transforme chaque carte de tarif en appel API
en direct à chaque affichage de page — exactement ce que le cache devait éviter.

**Et si l'appel échoue, la page doit s'afficher quand même.** Un Digistore24 lent
ou une clé erronée doit se traduire par un bouton désactivé accompagné d'un motif
(« checkout indisponible »), jamais par une exception sur la page de tarifs ni
par un lien mort. Renvoyez l'échec à l'appelant au lieu de lever une exception.

**En environnement de développement, cette URL n'est pas encore complète.** Tant
que le produit n'est pas approuvé sur la marketplace, personne ne peut acheter
par cette URL, et le moyen de débloquer un achat de test sans toucher à votre
navigateur est d'y ajouter le paramètre testpay — **ajouté à la valeur de retour,
après le cache, et seulement là où aucun client ne peut jamais arriver**. Ne
construisez pas le checkout en remettant cela à plus tard : c'est l'étape qui
décide si vous pourrez prouver que le reste fonctionne. La recette et les
garde-fous sont à l'**Étape 4a**.

## Étape 2 — faire suivre l'identité de l'acheteur jusqu'au bout

La panne la plus fréquente, et de loin, dans une intégration Digistore24, c'est
un paiement qui arrive sans qu'on puisse le rattacher à un compte. Quelqu'un a
payé, l'app ne sait pas qui, et c'est le support qui doit faire le rapprochement
à la main.

Envoyez un identifiant dans le champ tracking. Digistore24 le stocke sur l'achat
et vous le renvoie à **chaque** événement ultérieur de cette commande — le
renouvellement un an plus tard, le remboursement, la rétrofacturation
(chargeback). Dans l'IPN, il arrive sous `custom` :

```
tracking[custom] = m:<id du membre>;t:<un court token aléatoire stocké sur ce membre>
```

**Ce champ est une chaîne opaque qui n'appartient qu'à vous** ; donnez-lui donc
une structure extensible : des paires `key:value` séparées par `;`, et un lecteur
qui **ignore les clés qu'il ne connaît pas** au lieu d'échouer dessus. Tôt ou
tard, vous voudrez y faire passer un second id (quel forfait, quel type d'achat,
une intention exprimée par l'acheteur au checkout) — et à ce moment-là, des
achats en production porteront déjà l'ancienne valeur. Un nouvel id n'est alors
qu'une paire de plus ; un second *format*, en revanche, est une migration
impossible, parce que les valeurs déjà déposées chez Digistore24 ne peuvent pas
être réécrites.

**Deux choses sur ce token.** Il corrobore l'id du membre : un id deviné ou
modifié ne suffit donc jamais, à lui seul, à s'approprier l'achat de quelqu'un
d'autre. Et ce n'est **pas un secret d'authentification** : il n'ouvre jamais de
session, il dit seulement « cet id n'a pas été inventé par la personne qui tape
l'URL ». Les deux moitiés doivent être présentes et bien formées, sinon la valeur
ne désigne personne : une demi-identité n'est pas une identité affaiblie.

À l'autre bout, dans le handler IPN, attribuez dans cet ordre — et cet ordre est
une règle de sécurité, pas une préférence :

1. **L'identifiant tiré de `custom`, si le token correspond → authentifié.** C'est
   votre app qui a écrit cette valeur, Digistore24 l'a stockée côté serveur, et
   l'acheteur n'en a jamais eu de copie modifiable.
2. **Sinon, l'e-mail de l'acheteur comparé à vos comptes → non authentifié.**
   Cette adresse a été saisie dans un formulaire Digistore24 par la personne qui
   payait, et **Digistore24 ne vérifie pas qu'elle en a le contrôle**. Elle est
   le plus souvent juste, et elle n'est jamais une preuve.
3. **Sinon, stockez la commande sans l'attribuer**, et rattachez-la à la
   première connexion de cette adresse.

🚨 **« Pas de `custom` » n'est pas un diagnostic, et l'erreur est justement de le
lire comme tel.** Cela a au moins deux causes, indiscernables dans le log : un de
vos acheteurs qui n'était pas connecté au moment du clic (vous n'aviez aucun id
de membre à écrire), et quelqu'un qui n'est jamais passé par votre app — le
produit Digistore24 a son propre formulaire de commande, présent sur une
marketplace une fois approuvé, et un achat fait là-bas ne porte rien de ce que
vous écrivez. Le second est en outre facturé au plan stocké sur le **produit**,
et non au vôtre (**`ds24-products`**, Étape 2). Pour distinguer les deux, c'est
le montant qui fait la différence, pas le champ tracking.

Ce sont deux refus qui rendent l'étape 2 tenable :

- 🚨 **Une adresse qui correspond à plusieurs comptes est refusée, pas ramenée à
  la première ligne.** Demandez au plus deux correspondances et traitez « deux »
  comme *impossible à trancher*. La requête qui renvoie une liste et prend `[0]`
  a exactement la forme de ce bug, et son effet est de remettre à un client
  l'achat d'un autre. « Non attribuée » est le bon résultat ; deviner n'est pas
  une solution de repli.
- **L'attribution ne fait qu'accorder — elle ne déplace jamais rien et ne
  révoque jamais rien.** Une correspondance d'e-mail peut rattacher une commande
  qui n'appartient encore à personne. Elle ne peut pas rediriger une commande
  déjà attribuée, et aucun échec d'attribution ne peut mettre fin à un accès
  existant. C'est ce sens unique, et lui seul, qui rend tolérable un chemin non
  authentifié.

Et tout ce qui autorisera plus tard un acte **sans intervention humaine** —
débiter un moyen de paiement enregistré, armer une recharge automatique
(**`ds24-tokens`**) — n'accepte que le chemin 1. Une correspondance par le
chemin 2 est une bonne hypothèse sur l'identité de l'acheteur ; ce n'est pas une
autorisation de débiter une carte.

Une commande non attribuée, c'est un ticket de support. Une commande mal
attribuée, c'est un client qui voit l'achat de quelqu'un d'autre — et des deux,
c'est la plus coûteuse.

## Étape 3 — un achat sans compte doit fonctionner aussi

Laissez les gens acheter depuis la page de tarifs publique sans avoir à se
connecter d'abord. C'est ainsi qu'arrive la majorité d'entre eux, et exiger un
compte avant le paiement fait perdre des ventes. Le chemin 3 ci-dessus rend cela
sûr : la commande attend, et la première connexion depuis cette adresse la
réclame.

## Étape 4 — la page de remerciement

Après le paiement, Digistore24 renvoie l'acheteur vers une URL à vous, avec l'id
de la commande dedans. Deux règles :

- **Elle est publique.** L'acheteur n'a pas encore de session. Ne mettez rien
  derrière elle qui en suppose une.
- **N'accordez pas d'accès depuis cette page.** C'est un navigateur qui appelle
  une URL — n'importe qui peut l'appeler. L'accès vient de l'IPN, qui est signée.
  La page de remerciement dit « merci, c'est en route / voici comment vous
  connecter », et rien de plus.

**Digistore24 n'enregistre que des URL https publiques.** Une URL de
remerciement en `localhost` est rejetée d'emblée
(`Please only use secure URLs with https://`). Sur une plateforme hébergée,
l'URL de votre app est déjà publique et la question ne se pose pas ; sur un
ordinateur portable, il faut un redirecteur public ou un tunnel.

## Étape 4a — paiements de test avant l'approbation (la clé testpay)

Un produit pas encore approuvé sur la marketplace ne peut être acheté qu'en
**achat de test**. Il y a deux façons d'en débloquer un, et chacune a sa place :

- **Le cookie d'achat de test** — posé une fois dans le navigateur du vendeur
  (le lien est dans le centre d'aide de Digistore24). Propre à un navigateur, et
  il expire. C'est le bon outil sur tout domaine qu'un client pourrait atteindre
  lui aussi.
- **Le paramètre testpay** — obtenu par l'API et ajouté à l'URL d'achat, de
  sorte que le déblocage voyage avec le lien au lieu de résider dans un
  navigateur :

  ```
  POST https://www.digistore24.com/api/call/getTestpayKey/format/json
  Header: X-DS-API-KEY: <la clé>
  ```

  Non documenté, mais bien réel. La réponse contient `testpay_key`,
  `get_param_name` et `expires_at`. Ajoutez
  `?<get_param_name>=<testpay_key>` à l'URL d'achat (le NOM vient de la
  réponse — ne le codez jamais en dur) et le checkout s'ouvre en mode paiement
  de test, produit approuvé ou non. Envoyer `do_recreate=1` fait tourner la
  clé : une nouvelle est émise et toutes les anciennes copies cessent de
  fonctionner.

Quatre garde-fous, tous indispensables :

- **Développement et prévisualisation seulement — jamais sur une URL qu'un
  client peut atteindre.** Un checkout qui porte ce paramètre accepte des
  « paiements » de test : quiconque clique obtient le produit gratuitement.
  Conditionnez-le à votre environnement par une liste d'autorisation (tout ce
  qui n'est pas clairement du développement compte comme de la production, et
  refuse), et ajoutez-le au moment du rendu ou du clic.
- **Jamais dans une URL d'achat mise en cache ou partagée.** Si les URL d'achat
  sont mises en cache (Étape 1), mettez en cache l'URL nue et ajoutez le
  paramètre en sortie de cache — une URL décorée dans un cache partagé est
  servie à tout le monde.
- **La clé vaut pour tout le compte — traitez-la comme un secret.** Elle
  fonctionne sur TOUTES les URL de checkout de ce compte vendeur, celles de
  production comprises. Gardez-la hors du dépôt et hors de la configuration
  déployée.
- **Faites-la tourner avant la mise en production** (`do_recreate=1`) — voir
  **`ds24-golive`**.

## Étape 5 — prouvez-le

1. Créez une URL d'achat et ouvrez-la. La page de checkout doit afficher
   **votre** prix, devise et intervalle — si elle affiche autre chose, le plan de
   paiement n'est pas passé.
2. Faites un **achat de test** — avec le cookie d'achat de test de Digistore24
   posé, ou en environnement de développement avec le paramètre testpay ajouté
   (Étape 4a).
3. Vérifiez que l'IPN est bien arrivée et que la commande en est sortie
   **attribuée au bon compte**. L'attribution est la partie qui semble aller
   jusqu'au jour où elle ne va plus. Les achats de test arrivent avec
   `api_mode=test` dans le payload IPN — traitez-les exactement comme les achats
   réels (c'est ce chemin identique que le test prouve).

## Étape 6 — la suite

- **`ds24-ipn`** — l'endpoint qui reçoit ce que ce checkout produit.
- **`ds24-entitlements`** — transformer une commande payée en « a le droit
  d'utiliser le produit ».
- **`ds24-tokens`** — si vous vendez des crédits prépayés plutôt que des plans.
- **`ds24-golive`** — le vrai achat de test, de bout en bout.

Dites laquelle vous commencez, puis commencez-la.
