---
name: ds24-checkout
language: fr
description: À utiliser pour construire le bouton d'achat, la page de tarifs ou le lien de checkout d'un produit Digistore24 — créer une URL d'achat signée avec createBuyUrl, attacher le prix sous forme de plan de paiement, transporter l'identité de l'acheteur jusqu'à l'IPN, et la page de remerciement. À utiliser dès que l'utilisateur mentionne un lien d'achat, un checkout, une page de tarifs, « comment le client paie-t-il ? » ou un achat qui arrive sans que personne puisse dire à qui il appartenait.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Le lien de checkout

Un checkout Digistore24 est une **URL signée et de courte durée de vie** que vous
créez via l'API et vers laquelle vous envoyez l'acheteur. Ce n'est pas un lien
statique contenant un id de produit.

## Étape 0 — est-ce déjà là ?

Cherchez dans le projet `createBuyUrl`, `payment_plan` ou une page de tarifs qui
pointe déjà vers Digistore24. Si elle existe, ne la reconstruisez pas —
confrontez-la à l'Étape 3 et à l'Étape 4 et ne corrigez que ce qui est faux.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez avec le `VERSION` de ce pack. Signalez tout écart en une phrase, puis
continuez.

## Étape 1 — l'appel

```
POST https://www.digistore24.com/api/call/createBuyUrl/format/json
Header: X-DS-API-KEY: <la clé>
```

Corps (form-encoded), les parties qui comptent :

```
product_id                              = 512345
valid_until                             = 24h
payment_plan[first_amount]              = 47.00
payment_plan[other_amounts]             = 47.00
payment_plan[currency]                  = EUR
payment_plan[number_of_installments]    = 0        # 0 = abonnement ouvert, 1 = paiement unique
payment_plan[first_billing_interval]    = 1_month  # omettre entièrement pour un paiement unique
payment_plan[other_billing_intervals]   = 1_month
```

**Le prix est envoyé ici, au moment de l'achat — il n'est pas stocké sur le
produit.** Digistore24 jette `data[amount]` sur le produit lui-même, et un plan
de paiement stocké ne peut porter ni bon de réduction, ni essai gratuit, ni
montée en gamme, ni commission d'affiliation par lien. Lisez les nombres depuis
l'unique liste de prix de votre projet (voir **`ds24-products`**).

**`product_id` décide aussi de la LANGUE du formulaire de commande — choisissez-le
selon la langue de l'acheteur.** Un produit Digistore24 porte exactement une
langue, et aucun paramètre de cet appel ne la surcharge (regardez le corps
ci-dessus : `buyer`, `payment_plan`, `tracking`, `urls`, `placeholders`,
`settings`, `addons` — aucune langue nulle part). Une app multilingue garde donc
**un id de produit par langue** et par offre, et résout la langue du visiteur
vers l'un d'eux ici même, avant l'appel. Si vous envoyez tout le monde vers le
même id, la moitié remplit un formulaire dans la mauvaise langue au moment précis
où on leur demande de payer. La skill **`ds24-products`** contient la forme de la
liste de prix et la règle en entier.

La réponse est une URL. **Mettez-la en cache par offre** — elle est valable
pendant la fenêtre de `valid_until`, et en créer une nouvelle à chaque affichage
de page, c'est un aller-retour vers Digistore24 sur le chemin de votre page de
tarifs.

⚠️ **La clé de cache doit alors inclure la langue**, pas seulement la clé de
l'offre. Une ligne par clé signifie que l'URL allemande et l'URL anglaise
s'évincent mutuellement à chaque affichage de page et, entre-temps, le cache sert
la page de checkout d'une langue à l'acheteur de l'autre langue.
`"<offerKey>:<language>"` suffit.

🚨 **Et ne mettez jamais en cache une URL qui porte l'identité d'un acheteur.**
L'Étape 2 place l'id du membre connecté dans `tracking[custom]`, et un cache
indexé sur l'offre n'a aucune dimension membre — l'identité du premier acheteur
connecté est donc servie à tous ceux qui ouvrent cette page ensuite, et chacun de
*leurs* paiements arrive attribué à ce premier membre. Rien n'échoue pendant que
cela se produit : la page s'affiche, le checkout s'ouvre, l'argent circule.

Une page de tarifs a donc **deux chemins**, et ce ne sont pas deux versions d'un
seul :

- **Déconnecté → l'URL partagée mise en cache.** Aucune identité dedans, sûre
  pour tout le monde, aucun aller-retour vers Digistore24 pendant que la page
  s'affiche.
- **Connecté → une URL construite au moment du clic**, avec l'identité de ce
  membre dedans, utilisée une fois et **jamais écrite dans le cache**.

Décidez lequel d'après le **contenu** du champ tracking, pas d'après le fait
qu'il soit rempli ou non : un marqueur qui nomme le *forfait* est partageable, un
marqueur qui nomme une *personne* ne l'est pas. Demander simplement « le tracking
est-il rempli ? » transforme chaque carte en appel API en direct à chaque
affichage de page, ce que le cache existait précisément pour empêcher.

**Et quand l'appel échoue, la page doit quand même s'afficher.** Un Digistore24
lent ou une clé erronée doit produire un bouton désactivé avec une raison
(« checkout indisponible »), jamais une erreur levée sur la page de tarifs et
jamais un lien mort. Retournez l'échec à l'appelant au lieu de le lever.

**Cette URL n'est pas encore terminée dans un environnement de développement.**
Tant que le produit n'a pas l'approbation de la marketplace, personne ne peut
acheter au travers, et la façon de débloquer un achat de test sans toucher à
votre navigateur est d'ajouter le paramètre testpay — **ajouté à la valeur de
retour, après le cache, et uniquement là où un client ne peut jamais arriver**.
Ne construisez pas le checkout en remettant ceci à plus tard : c'est l'étape qui
décide si vous pouvez prouver que tout le reste fonctionne. L'**Étape 4a** donne
la recette et les garde-fous.

## Étape 2 — transportez l'identité de l'acheteur jusqu'au bout

L'échec de loin le plus fréquent dans une intégration Digistore24 est un paiement
qui arrive et qu'on ne peut rattacher à aucun compte. Quelqu'un a payé, l'app
n'a aucune idée de qui, et le support doit le faire à la main.

Envoyez un identifiant dans le champ tracking. Digistore24 le stocke sur l'achat
et vous le rend à **chaque** événement ultérieur de cette commande — le
renouvellement un an plus tard, le remboursement, la rétrofacturation. Il arrive
dans l'IPN sous `custom` :

```
tracking[custom] = m:<id du membre>;t:<un court token aléatoire stocké sur ce membre>
```

**Ce champ est une seule chaîne opaque qui vous appartient entièrement**,
donnez-lui donc une structure que vous pourrez étendre : des paires `key:value`
séparées par `;`, et un lecteur qui **ignore les clés qu'il ne connaît pas**
plutôt que d'échouer dessus. Vous voudrez plus tard transporter un second id
(quel forfait, quel type d'achat, une intention exprimée par l'acheteur au
checkout), et il y aura alors des achats en production qui portent l'ancienne
valeur. Un nouvel id est alors une nouvelle paire ; un second *format* est une
migration que vous ne pouvez pas faire, car les valeurs déjà posées chez
Digistore24 ne peuvent pas être réécrites.

**Deux choses à propos de ce token.** Il corrobore l'id du membre, de sorte qu'un
id deviné ou modifié ne réclame jamais à lui seul l'achat de quelqu'un d'autre —
et **ce n'est pas un identifiant de connexion** : il n'authentifie jamais une
session, il dit seulement « cet id n'a pas été inventé par la personne qui tape
l'URL ». Les deux moitiés doivent être présentes et bien formées, sinon la valeur
ne nomme personne : une demi-identité n'est pas une identité plus faible.

À l'autre bout, dans le handler d'IPN, attribuez dans cet ordre — et l'ordre
est une règle de sécurité, pas une préférence :

1. **L'identifiant venant de `custom`, avec le token qui correspond →
   authentifié.** Votre app a écrit cette valeur, Digistore24 l'a stockée côté
   serveur, et l'acheteur n'en a jamais eu une copie modifiable.
2. **Sinon, l'e-mail de l'acheteur confronté à vos comptes → non authentifié.**
   Cette adresse a été tapée dans un formulaire Digistore24 par celui qui payait,
   et **Digistore24 ne vérifie pas qu'il la contrôle**. Elle est généralement
   juste et elle n'est jamais une preuve.
3. **Sinon, stockez la commande non attribuée** et rattachez-la quand cette
   adresse se connecte pour la première fois.

🚨 **« Pas de `custom` » n'est pas un diagnostic, et le lire ainsi est l'erreur.**
Cela a au moins deux causes que le log ne distingue pas : un de vos acheteurs qui
n'était pas connecté au moment du clic (vous n'aviez aucun id de membre à
écrire), et quelqu'un qui n'est jamais passé par votre app — le produit
Digistore24 a un formulaire de commande qui lui est propre, sur une marketplace
une fois approuvé, et un achat fait là ne porte rien que vous ayez écrit. Le
second est en outre facturé au plan stocké du **produit**, pas au vôtre
(**`ds24-products`**, Étape 2). Si vous voulez les distinguer, c'est le montant
qui les sépare, pas le champ de tracking.

Deux refus sont ce qui rend l'étape 2 acceptable :

- 🚨 **Une adresse qui correspond à plus d'un compte est refusée, pas résolue
  vers la première ligne.** Demandez au plus deux correspondances et traitez
  « deux » comme *impossible à trancher*. La requête qui renvoie une liste et
  prend `[0]` est la forme exacte de ce bug, et ce qu'elle fait, c'est remettre à
  un client l'achat d'un autre client. Non attribuée est le résultat correct ;
  deviner n'est pas une solution de repli.
- **L'attribution ne fait qu'accorder — elle ne déplace jamais et ne révoque
  jamais.** Une correspondance d'e-mail peut rattacher une commande qui
  n'appartient encore à personne. Elle ne peut pas réorienter une commande déjà
  attribuée, et aucun échec d'attribution ne peut mettre fin à un accès qui
  existe. Cette unidirectionnalité est toute la raison pour laquelle un chemin
  non authentifié est tolérable.

Et tout ce qui autorise plus tard un acte **non supervisé** — débiter un moyen de
paiement enregistré, armer une recharge automatique (**`ds24-tokens`**) —
n'accepte que le chemin 1. Une correspondance du chemin 2 est une bonne
supposition sur qui a acheté quelque chose ; ce n'est pas la permission de
débiter une carte.

Une commande non attribuée est un ticket de support. Une commande mal attribuée
est un client qui regarde l'achat de quelqu'un d'autre, et c'est la plus coûteuse
des deux.

## Étape 3 — un achat sans compte doit fonctionner quand même

Laissez les gens acheter depuis la page de tarifs publique sans se connecter
d'abord. C'est ainsi que la plupart arrivent, et imposer un compte avant le
paiement coûte des ventes. Le chemin 3 ci-dessus est ce qui rend cela sûr : la
commande attend, et la première connexion depuis cette adresse la réclame.

## Étape 4 — la page de remerciement

Digistore24 envoie l'acheteur vers une URL à vous après le paiement, avec l'id de
la commande dedans. Deux règles :

- **Elle est publique.** L'acheteur n'a pas encore de session. N'y mettez rien
  derrière qui en suppose une.
- **N'accordez pas l'accès depuis elle.** C'est un navigateur qui appelle une URL
  — n'importe qui peut l'appeler. L'accès vient de l'IPN, qui est signée. La page
  de remerciement dit « merci, c'est en route / voici comment vous connecter »,
  rien de plus.

**Digistore24 ne stocke que des URLs https publiques.** Une URL de remerciement
en `localhost` est rejetée d'emblée (« Please only use secure URLs with
https:// »). Sur une plateforme hébergée, l'URL de votre app est déjà publique,
donc la question ne se pose pas ; sur un portable, il faut un redirecteur public
ou un tunnel.

## Étape 4a — paiements de test tant que le produit n'est pas approuvé (la clé testpay)

Un produit qui n'a pas encore l'approbation de la marketplace ne peut être acheté
qu'en **achat de test**. Il y a deux façons d'en débloquer un, et elles
conviennent à des endroits différents :

- **Le cookie d'achat de test** — posé une fois dans le navigateur du vendeur (le
  centre d'aide de Digistore24 a le lien). Par navigateur, il expire. Le bon
  outil sur tout domaine qu'un client pourrait atteindre lui aussi.
- **Le paramètre testpay** — récupéré via l'API et ajouté à l'URL d'achat, de
  sorte que le déblocage voyage avec le lien au lieu de vivre dans un
  navigateur :

  ```
  POST https://www.digistore24.com/api/call/getTestpayKey/format/json
  Header: X-DS-API-KEY: <la clé>
  ```

  Non documenté, mais réel. La réponse porte `testpay_key`,
  `get_param_name` et `expires_at`. Ajoutez
  `?<get_param_name>=<testpay_key>` à l'URL d'achat (le NOM vient de la
  réponse — ne le codez jamais en dur) et le checkout s'ouvre en mode paiement de
  test, approuvé ou non. Envoyer `do_recreate=1` fait tourner la clé : une
  nouvelle est émise et toutes les anciennes copies cessent de fonctionner.

Quatre garde-fous, tous porteurs :

- **Développement/preview uniquement — jamais sur une URL qu'un client peut
  atteindre.** Un checkout portant ce paramètre accepte des « paiements » de
  test : quiconque clique dessus obtient le produit gratuitement. Restreignez-le
  à votre environnement avec une liste d'autorisation (tout ce qui n'est pas
  clairement du développement compte comme de la production et refuse), et
  ajoutez-le au moment du rendu ou du clic.
- **Jamais dans une URL d'achat mise en cache ou partagée.** Si les URLs d'achat
  sont mises en cache (Étape 1), mettez en cache l'URL propre et ajoutez le
  paramètre après le cache — une URL décorée dans un cache partagé est servie à
  tout le monde.
- **La clé est au niveau du compte — traitez-la comme un secret.** Elle
  fonctionne sur TOUTES les URLs de checkout de ce compte vendeur, y compris
  celles en production. Gardez-la hors du repo et hors de la configuration
  déployée.
- **Faites-la tourner avant la mise en production** (`do_recreate=1`) — voir
  **`ds24-golive`**.

## Étape 5 — prouvez-le

1. Créez une URL d'achat et ouvrez-la. La page de checkout doit afficher **votre**
   prix, votre devise et votre intervalle — si elle affiche autre chose, le plan
   de paiement n'a pas voyagé.
2. Faites un **achat de test** — avec le cookie d'achat de test Digistore24 posé,
   ou dans un environnement de développement avec le paramètre testpay ajouté
   (Étape 4a).
3. Vérifiez que l'IPN est arrivée et que la commande est ressortie **attribuée au
   bon compte**. L'attribution est la partie qui a l'air correcte jusqu'à ce
   qu'elle ne le soit plus. Les achats de test arrivent avec `api_mode=test` dans
   le payload de l'IPN — traitez-les comme ceux en production (ce chemin
   identique est ce que le test prouve).

## Étape 6 — la suite

- **`ds24-ipn`** — l'endpoint qui reçoit ce que ce checkout produit.
- **`ds24-entitlements`** — transformer une commande payée en « peut utiliser le
  produit ».
- **`ds24-tokens`** — si vous vendez des crédits prépayés plutôt que des plans.
- **`ds24-golive`** — le vrai achat de test, de bout en bout.

Dites laquelle vous commencez, et commencez-la.
