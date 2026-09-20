---
name: ds24-products
language: fr
description: À utiliser pour raccorder une app à un compte Digistore24 la première fois — mettre en place la clé d'API, créer les produits à vendre, enregistrer la connexion IPN (le webhook) et demander l'approbation de la marketplace. À utiliser aussi dès que l'utilisateur mentionne une clé d'API Digistore24, dit « connecter Digistore », veut créer des produits ou des plans, enregistrer une URL IPN, ou demande pourquoi Digistore24 n'appelle jamais son webhook.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Connecter l'app à Digistore24

Rien, dans une intégration Digistore24, ne fonctionne tant que trois choses
n'existent pas côté Digistore24 : une clé d'API utilisable par votre app, un
produit à vendre et une connexion IPN qui pointe vers votre endpoint.
**Commencez par là** — un handler IPN que personne n'appelle ne peut pas être
testé, et un lien de checkout vers un produit inexistant renvoie un 404.

## Étape 0 — qu'est-ce qui existe déjà ?

Regardez avant de poser des questions :

- Y a-t-il une `DIGISTORE_API_KEY` dans l'environnement ou dans le coffre à
  secrets ?
- Le projet contient-il un registre de produits (un fichier JSON ou de
  configuration qui liste les plans et leurs prix) ?
- `DIGISTORE_IPN_PASSPHRASE` est-elle définie ?

Ne demandez ensuite à l'utilisateur que ce qui manque vraiment. Si les trois
sont là, passez directement à l'Étape 4 et vérifiez la connexion au lieu de la
reconstruire.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez-le au fichier `VERSION` de ce pack. En cas d'écart, signalez-le en
une phrase, puis poursuivez.

## Étape 1 — l'API

```
POST https://www.digistore24.com/api/call/<FUNCTION>/format/json
Header: X-DS-API-KEY: <la clé>
Body:   application/x-www-form-urlencoded
```

**La clé passe dans l'en-tête, jamais dans un paramètre de formulaire.** C'est
un secret : une variable d'environnement ou le coffre à secrets de la
plateforme — jamais dans le code, jamais dans quoi que ce soit qui parvient au
navigateur.

L'utilisateur la crée lui-même dans son compte Digistore24, sous
*Settings → API keys*. Demandez-la-lui, dites-lui où la placer, et n'essayez
pas de la récupérer dans une session de navigateur.

⚠️ **Dites-lui d'accorder à la clé le droit d'ÉCRITURE** (*writable*, dans les
termes de Digistore24). Les droits d'une clé sont fixés à sa création ; une clé
en lecture seule lit les produits sans difficulté, puis échoue sur les deux
appels dont l'app ne peut pas se passer : la création des produits et la
création d'une URL de checkout. Dites-le pendant que l'utilisateur a cet écran
sous les yeux — élargir une clé après coup, c'est en créer une nouvelle et la
remplacer partout.

## Étape 2 — une seule liste de prix, dans votre app

**Gardez les plans dans un seul fichier de votre projet** et faites-y lire tout
le reste : la page de tarifs, le checkout, la vérification du droit d'accès.
Une entrée par **offre**, et sous elle une entrée par **façon de payer**.

🚨 **Mensuel et annuel sont deux façons de payer UNE offre — pas deux offres.**
C'est la décision de forme la plus lourde de conséquences de toute cette skill.
Un registre qui en fait deux entrées en fait deux clés de produit, et dès lors
chaque vérification d'accès de l'app doit les nommer toutes les deux :
`hasAccess(m, "pro_monthly") || hasAccess(m, "pro_yearly")`, à chaque porte,
pour toujours. Celui qui s'écrit avec une seule clé refuse silencieusement la
moitié des acheteurs, derrière une page qui s'affiche parfaitement. Gardez-les
sous une seule clé et la question reste une seule question.

Chez Digistore24, les deux axes ne se comportent pas de la même façon, et voici
pourquoi :

| axe | ce qu'il coûte |
|---|---|
| **langue** | un PRODUIT pour chacune — la langue du formulaire de commande est une propriété du produit (Étape 3) |
| **façon de payer** | un PLAN DE PAIEMENT pour chacune, sur ce même produit |

```
pro:
  name: "Pro"
  paymentOptions:                     # un plan de paiement pour chacune
    monthly: { priceCents: 3900,  billingInterval: 1_month }
    yearly:  { priceCents: 39000, billingInterval: 12_month }
  productIds:                         # un produit Digistore24 par langue
    de: null
    en: null
  payplanIds:                         # écrits par votre synchronisation
    de: { monthly: null, yearly: null }
    en: { monthly: null, yearly: null }
```

⚠️ **Ce qui VÉRIFIE ce fichier doit lire le prix sur la façon de payer, pas sur
l'offre.** Une offre qui déclare `paymentOptions` n'a pas de `priceCents` à
elle, si bien qu'un validateur écrit contre l'ancienne forme annonce à toute
liste de prix correcte qu'elle n'a pas de prix. Mesuré, dès la première
exécution réelle : un avertissement qui se déclenche sur une entrée correcte
apprend au lecteur à sauter les avertissements.

### Où vit le prix

**C'est votre fichier qui fait foi. Digistore24 en reçoit une copie, sous forme
de plans de paiement.**

`data[amount]` sur le produit est obsolète et ignoré : aucun prix n'est donc
jamais fixé sur le produit lui-même — l'API le dit dans son propre
avertissement : *« créez plutôt un plan de paiement »*. C'est à cela que servent
`createPaymentplan` / `updatePaymentplan`, et votre synchronisation en écrit un
par façon de payer (Étape 3).

⚠️ **Vous avez peut-être lu le contraire dans une copie plus ancienne de ce
pack**, qui disait de tenir tous les prix hors de Digistore24 et de les envoyer
avec chaque appel `createBuyUrl`. Le raisonnement était : un plan stocké est un
deuxième endroit pour le prix, et il ne sait faire ni essais gratuits, ni
montées en gamme, ni bons de réduction. La moitié était juste. Ce qui lui
échappait, c'est que **votre lien de checkout n'est pas la seule entrée vers le
produit** :

- le produit a un **formulaire de commande propre**, qu'aucun lien de vous ne
  touche ;
- un **affilié** peut y envoyer du trafic directement ;
- l'acheteur peut **changer son intervalle de facturation** depuis son propre
  achat (`switch_pay_interval_url`, présent sur chaque IPN) — et cela bascule
  vers un autre plan *stocké* : sans aucun, il n'y a rien vers quoi basculer.

Tous les trois facturent les plans accrochés au produit. Un produit sans aucun
plan de vous n'est pas pour autant sans plan : **Digistore24 lui en attribue un
par défaut** — environ 27 €, paiement unique, tel qu'observé sur un compte réel
en septembre 2026 ; regardez le produit du vendeur plutôt que de vous fier à ce
chiffre. Sur une offre en **abonnement**, une telle commande n'envoie qu'un
seul événement de paiement, jamais de renouvellement ni de résiliation :
l'acheteur paie une fois et garde l'accès **pour toujours**.

Écrire les plans, c'est ce qui referme cela. C'est la deuxième raison de cette
forme ; la première est qu'une offre a désormais besoin d'un produit au lieu de
deux.

Deux conséquences en découlent, et toutes deux passent facilement inaperçues :

- **Dites au vendeur ce qu'il verra dans son backoffice** : ses propres prix, un
  plan par façon de payer — **en copie**. Changez un prix dans l'app, puis
  synchronisez ; changez-le là-bas et la synchronisation suivante le remettra
  comme avant. Un vendeur à qui on ne le dit pas modifie une fois la mauvaise
  copie et en conclut que l'app oublie les prix.
- **Un achat passé sur le formulaire de commande propre au produit n'a toujours
  pas de `tracking[custom]`** : votre handler IPN doit donc l'attribuer
  autrement (l'Étape 2 de **`ds24-checkout`** explique pourquoi l'absence de
  `custom`, à elle seule, ne permet pas de savoir que c'est ce qui s'est passé).
  Ce qui a changé, c'est qu'il n'est plus facturé à un autre prix que le vôtre,
  et qu'il n'est plus un abonnement qui ne se renouvelle jamais.

**Si votre app parle plusieurs langues, l'entrée porte un id de produit par
langue** — pas un id unique. La raison est à l'Étape 3 ; fixez la bonne forme
dès maintenant, parce que la changer après la première vente signifie de
nouveaux produits et de nouvelles approbations.

**Et si l'app a plusieurs environnements, gardez un JEU de produits par
environnement** (dev / prod — staging seulement s'il existe vraiment). Les
produits créés contre une URL de prévisualisation ou de développement sont des
articles de test : donnez-leur leurs propres ids dans le registre, marquez-les
visiblement dans le nom du produit (`"Pro [DEV]"` — l'API de Digistore24 n'a
pas de champ d'étiquette ; le nom est ce qu'un humain voit dans le backoffice)
et laissez propres les noms des produits de production. Un jeu ne doit jamais
s'approprier les produits de l'autre — voir la note sur l'idempotence plus bas.
Un vendeur qui ne synchronise jamais que contre le domaine de production n'a
qu'un seul jeu, et c'est très bien ainsi.

## Étape 3 — créer les produits

Deux appels par produit, dans cet ordre :

1. `createProduct` / `updateProduct` avec le nom, la description et
   **`language`**. Reportez l'id de produit renvoyé dans votre liste de prix :
   la correspondance est ainsi enregistrée, pas recalculée à chaque fois.
2. `createPaymentplan` / `updatePaymentplan` **une fois par façon de payer**,
   avec `product_id`, `first_amount`, `currency`, `first_billing_interval`,
   `other_amounts`, `other_billing_intervals`, `number_of_installments`
   (`0` = abonnement à durée indéterminée, `1` = paiement unique) et
   `position`. Reportez également le `paymentplan_id` renvoyé.

   Mettez `is_switching_allowed = Y` là où l'offre a plus d'une façon de payer :
   c'est ce qui fait que le `switch_pay_interval_url` de Digistore24 mène
   quelque part, et cela vous épargne de construire un parcours de montée en
   gamme pour le changement le plus courant que fait un abonné.

⚠️ **Faites les deux ensemble, pas en deux passages.** Un produit qui existe
sans ses plans a le plan par défaut de Digistore24, à ~27 €, et un formulaire
de commande qui le facture ; la fenêtre pendant laquelle c'est vrai ne devrait
durer qu'un seul appel d'API.

🚨 **Et inscrivez au passage une MARQUE DE PROPRIÉTÉ.** `createProduct` et
`updateProduct` acceptent tous deux `data[note]`, une note interne libre
qu'aucun acheteur ne voit. Mettez-y une ligne lisible par une machine — l'id
propre à votre app et l'environnement. Sans elle, il n'y a pas de réponse
honnête à « ce produit, est-ce NOUS qui l'avons créé ? », et l'Étape 3b en a
besoin. Ne dérivez pas cet id du nom de l'app (les vendeurs renomment) ni du nom
interne du produit (deux apps bâties de la même façon s'y télescopent) :
générez-le une fois, rangez-le à côté de la liste de prix, et ne le régénérez
jamais.

🚨 **`note` conserve 47 caractères et jette le reste — en silence.** Mesuré sur
un compte réel le 2026-09-09 : une valeur de 120 caractères est revenue coupée
au milieu d'un mot, sans erreur, sans avertissement, et sans un mot à ce sujet
dans la documentation de l'API. Gardez donc la marque courte — `monapp:1:<appId>:<env>`
fait une trentaine de caractères — et **n'y mettez ni la clé de produit ni la
langue** : elles sont déjà dans `name_intern`, que le listing renvoie à côté de
la note.

Une marque trop longue d'un caractère n'est pas une marque plus courte. Elle ne
se parse pas, tous les produits créés par votre app se lisent comme ceux de
quelqu'un d'autre, et votre étape de nettoyage annonce « rien à retirer » à
partir d'une comparaison qui n'a rien trouvé parce qu'elle ne le pouvait pas.
Fixez la longueur dans un test.

### La deuxième marque : `data[tag]`

À côté de la note se trouve un champ **tag**, et il répond à une question plus
grossière : non pas « quelle app a fait ceci » mais « est-ce seulement une app
qui l'a fait ». Mettez-y un nom qui identifie votre outillage et le vendeur
pourra filtrer son backoffice dessus. **Et si vous tenez un jeu de produits par
environnement (ci-dessus), donnez à chaque jeu son propre tag** : un filtre ne
vaut la peine d'être tapé que s'il sépare les produits en production de ceux
qu'une synchronisation a créés pendant que quelqu'un testait :

```
data[tag] = ds24-skills          # le jeu en production
data[tag] = ds24-skills-test     # staging
data[tag] = ds24-skills-dev      # développement
```

🚨 **Lisez cette valeur DEPUIS l'environnement, une seule fois, avant le premier
appel, et ne donnez aucune valeur par défaut à la fonction.** La valeur par
défaut serait le tag de production, et le jour où un appel oublie l'argument il
marque un produit de test comme étant en production dans le filtre du vendeur —
invisible pour tout test qui passe sa propre valeur. Un environnement que vous
ne reconnaissez pas est une erreur, pas un repli.

Et **ajoutez seulement, ne retirez jamais** : un produit synchronisé sous un
environnement puis sous un autre conserve les deux tags. En retirer un ne vaut
pas le risque, puisque le tag ne prouve de toute façon aucune propriété
(ci-dessous).

🚨 **Les tags sont une LISTE séparée par des virgules et le champ s'écrit en
entier.** En ajouter un est donc toujours un lire-modifier-écrire :

1. lisez le `tag` actuel du produit (`getProduct`, ou le listing que vous avez
   déjà : il y figure aussi) ;
2. si le vôtre est déjà dans la liste, n'écrivez rien ;
3. sinon, ajoutez-le après une virgule et renvoyez la liste **entière** avec
   `updateProduct`.

N'écrire que votre propre tag efface tous ceux que le vendeur y avait mis. Sur
`createProduct` il n'y a rien à lire : votre tag est la valeur entière.

⚠️ **Et sachez ceci de `data` en général : il est validé contre une liste blanche
stricte.** Une clé que Digistore24 ne connaît pas est une ERREUR — l'appel
entier est refusé avec « ungültiger Array-Schlüssel », et non ignoré en silence.
Un champ plus récent que l'API du compte casse donc tous les appels qui le
portent. Si vous écrivez contre un champ en cours de déploiement : envoyez-le,
attrapez le refus, et répétez l'appel une fois sans lui.

🚨 **Et ne laissez PAS votre étape de nettoyage lire le tag.** Toutes les apps
bâties de la même façon portent le même — un retrait décidé dessus laisserait
une app supprimer les produits d'une autre. La propriété reste la marque fine
de la note, celle qui porte l'id propre à votre app.

Des 47 caractères découlent deux règles, et les deux portent sur la question de
savoir à qui appartient ce champ :

- **N'écrivez jamais par-dessus un texte que vous n'avez pas écrit.** Il n'y a
  pas la place de fusionner : si la note contient autre chose, laissez-la et
  passez. Ce produit reste alors sans marque et votre étape de nettoyage n'y
  touchera pas — la direction sûre.
- **Mais tenez votre propre PRÉFIXE pour vôtre, même s'il ne se parse pas.** La
  coupe peut tomber à l'intérieur d'une marque que vous avez écrite. Sans cela,
  une marque que vous avez un jour mal écrite est définitive : elle se lit comme
  le texte du vendeur pour toujours et rien ne peut la réparer. Un marqueur
  qu'on ne peut pas corriger est pire qu'un marqueur qu'on ne peut pas lire.

### Un produit par offre ET par langue — c'est ici que l'on se trompe le plus souvent

*(Et, pour le redire une fois de plus là où on l'inverse le plus facilement : un
produit par LANGUE, un plan de paiement par FAÇON DE PAYER. L'axe de la langue
multiplie les produits ; la façon de payer, non.)*

**Un produit Digistore24 porte exactement UNE langue, et c'est celle du
FORMULAIRE DE COMMANDE que remplit votre acheteur** — libellés des champs,
boutons, noms des moyens de paiement, conditions de résiliation. Sur le produit,
c'est `data[language]`.

**`createBuyUrl` n'a pas de paramètre de langue.** Ses arguments sont
`product_id`, `buyer`, `payment_plan`, `tracking`, `valid_until`, `urls`,
`placeholders`, `settings` et `addons` — rien là-dedans ne permet de remplacer
la langue du produit, et aucun paramètre d'URL ne le fait non plus. La langue
du formulaire ne se décide donc pas au moment du checkout ; elle se décide en
**choisissant le produit vers lequel vous envoyez l'acheteur**.

Une app dont l'interface existe en allemand et en anglais a donc besoin de
**deux produits Digistore24 par offre**, l'un avec `language=de`, l'autre avec
`language=en`, et le checkout choisit selon la langue du visiteur. Envoyez tout
le monde vers un seul produit, et la moitié de vos clients devront saisir leurs
coordonnées bancaires dans une langue qu'ils n'ont pas choisie — précisément le
moment où un achat est abandonné.

Trois conséquences à inscrire dans ce que vous construisez :

- **Définissez `data[language]` explicitement sur chaque produit.** Sans cela,
  Digistore24 retombe sur la langue de la session API — un choix que personne
  n'a fait, et la cause habituelle d'une boutique allemande qui affiche des
  formulaires de commande en anglais.
- **Couvrez toutes les langues de votre app.** Une langue absente doit quand
  même vendre (retombez sur un autre produit plutôt que d'afficher un bouton
  mort) — mais dites-le dans la sortie de votre synchronisation, car rien
  d'autre ne le dira jamais : l'app s'affiche correctement, le checkout s'ouvre,
  l'achat aboutit.
- **Chaque produit de langue est approuvé séparément**, sur la marketplace dont
  relève sa langue. Voir la skill **`ds24-golive`**.

Le *texte* de votre produit est une autre question. Envoyer le même nom et la
même description aux deux produits est un choix par défaut tout à fait valable
— c'est le *formulaire* qui l'entoure qui doit suivre l'acheteur.

Rendez l'opération **idempotente** : lancée deux fois, elle met à jour au second
passage au lieu de créer un doublon. Indexez-la sur votre propre clé de produit
**plus la langue** — et, si vous tenez des jeux séparés par environnement,
**plus l'environnement** (`pro__en__prod`) : chaque produit a besoin de son
propre identifiant stable. N'indexez jamais sur le nom affiché, qui est le même
dans les deux langues et change avec le texte.

**Retirer un produit de votre liste ne le dépublie pas à soi seul.** Le produit
que Digistore24 connaît déjà reste achetable — par son propre formulaire de
commande et par tout lien de checkout existant — jusqu'à ce que quelque chose
l'en retire. C'est l'objet de l'Étape 3b.

🚨 **Et c'est pourquoi le moment de demander, c'est AVANT de créer, pas
après.** `deleteProduct(product_id)` existe : ce n'est donc pas définitif comme
le disait une copie plus ancienne de ce pack. Mais ce n'est propre que tant que
le produit **n'a jamais vendu** : celui qui a encaissé de l'argent ne peut plus
qu'être désactivé, parce que les remboursements, les rétrofacturations et les
résiliations de ses acheteurs continuent d'arriver sous forme d'IPN nommant son
id, et votre handler doit continuer de les recevoir. Un produit qui a encaissé
de l'argent est une décision que le vendeur garde.

Donc, la première fois que votre synchronisation s'apprête à créer quoi que ce
soit : **affichez ce qui serait créé, nom par nom, dites ce que cela coûte, et
attendez un oui.** Puis créez. Les passages suivants
trouvent les ids déjà enregistrés et ne créent rien : c'est une question posée
une fois, à un moment précis, pas une invite que l'on apprend à valider sans
lire. Si certaines entrées sont des brouillons et non des offres, donnez à votre
liste un drapeau qui les tient à l'écart de la synchronisation, plutôt que de
demander à l'utilisateur de supprimer un texte qu'il veut garder.

## Étape 3b — retirer ce qui n'est plus proposé

Une entrée sortie de la liste de prix laisse un produit derrière elle. Faire ce
ménage en vaut la peine — un compte qui se remplit de produits abandonnés, c'est
ainsi qu'un vendeur finit par vendre quelque chose qu'il avait oublié — et c'est
la seule étape où se tromper détruit le travail de quelqu'un d'autre.

**N'agissez jamais sur autre chose que votre propre marque de propriété**
(Étape 3). Pas sur le nom du produit, pas sur le nom interne, pas sur le
dossier : le compte d'un vendeur contient des produits antérieurs à votre app,
des produits d'une deuxième app bâtie de la même façon, et des produits créés à
la main par un agent du support. « Sans doute à nous » ne suffit pas pour
supprimer.

Ensuite, pour chaque produit qui porte votre marque et ne figure plus dans la
liste de prix :

1. **Demandez s'il a des ventes** (`listPurchases` avec son `product_id`). Si
   vous ne pouvez pas le demander, traitez cela comme « il en a » : l'erreur
   bon marché est un produit inactif de trop.
2. **Aucune vente → `deleteProduct`.** En cas d'échec, quelle qu'en soit la
   raison, passez au point suivant.
3. **Des ventes, ou une suppression qui a échoué → `updateProduct` avec
   `data[is_active] = N`.** Le produit cesse d'être achetable et reste dans le
   compte. Dites au vendeur que son entrée doit rester dans la liste de prix,
   en entrée mise de côté, sinon son id sort de votre enregistrement IPN et les
   remboursements de ses acheteurs cessent d'arriver.

🚨 **Et ne faites strictement rien si vous n'avez pas pu lire les notes.** Si la
liste des produits est revenue sans elles, la propriété ne peut pas être
établie, et « zéro produit à retirer » n'est alors pas une réponse : c'est un
silence. Dites-le et arrêtez-vous. Prouver que le parcours a eu lieu n'est pas
prouver que la comparaison a eu lieu.

⚠️ **Et assurez-vous que le drapeau fonctionne encore avec une liste de prix
VIDE.** L'endroit naturel pour un refus « rien à synchroniser » est avant tout
échange avec Digistore24 — et alors, qui retire sa DERNIÈRE entrée garde ce
produit dans le compte sans moyen de l'enlever. Mesuré : c'est la plus facile de
ces erreurs, parce que tous les tests contiennent au moins un produit.
« Rien à synchroniser » et « rien à nettoyer » sont deux questions.

**Toujours derrière un drapeau explicite.** Contrairement à la création, ce
n'est pas une question du premier passage : une clé renommée ou une coquille
dans la liste de prix supprimeraient un produit vivant lors d'une
synchronisation ordinaire. Énumérez ce qui partirait, ne changez rien, et
laissez l'utilisateur le demander.

## Étape 4 — enregistrer la connexion IPN

C'est l'étape que l'on oublie, et son symptôme est « l'achat a fonctionné, mais
il ne s'est rien passé dans l'app ».

- `ipnSetup` enregistre l'endpoint. Digistore24 **le valide immédiatement** par
  un `GET` et exige un HTTP `200` — une redirection (301/302) échoue elle aussi.
- **L'URL doit être publique et en `https`.** Digistore24 refuse `http` et
  refuse `localhost`, purement et simplement.
- Digistore24 génère la **passphrase IPN** ou prend la vôtre. Dans les deux
  cas, elle doit se retrouver dans l'environnement de l'app sous
  `DIGISTORE_IPN_PASSPHRASE` : c'est le secret partagé avec lequel la signature
  est calculée, et sans lui chaque IPN est rejetée — à juste titre.

L'appel prend les paramètres suivants, et deux d'entre eux décident si des
événements arriveront un jour :

| | |
|---|---|
| `ipn_url` | votre endpoint, public et en https |
| `name` | le nom de la connexion dans le backoffice |
| `domain_id` | **l'identité de cette connexion** — voir plus bas |
| `product_ids` | les produits qu'elle couvre — des ids séparés par des virgules, ou `all` |
| `sha_passphrase` | la vôtre, ou `random` pour en faire générer une et la recevoir en retour |

### `ipnSetup` fait aussi la mise à jour — c'est le `domain_id` qui décide

Il n'existe pas de fonction de mise à jour distincte. Digistore24 retrouve une
connexion par le triplet **(marchand, clé d'API, `domain_id`)** : même id → la
connexion existante est mise à jour, id inconnu → une deuxième connexion
apparaît. C'est ce qui rend l'appel idempotent, et c'est pourquoi l'id doit
être **noté quelque part** (une variable d'environnement, une ligne de
paramètres) plutôt que recalculé à partir de quelque chose qui change.

**Et il doit être unique.** C'est la partie que l'on saute, et l'échec est
invisible. Une valeur générique — `test-local-1`, `local-app`, `myapp`,
`production` — n'est pas un nom, c'est une collision avec un **autre** projet du
même utilisateur : les deux n'obtiennent pas deux connexions, ils écrasent la
même à tour de rôle. La deuxième configuration redirige en silence l'IPN de la
première app vers sa propre URL, et à partir de là les achats de la première
app n'arrivent plus nulle part. Les deux passages annoncent un succès.

Ajoutez-lui donc un suffixe aléatoire, et stockez-le :

```
test-local-diw2hvnz73
myapp-prod-k7f2m9x1qc
```

La partie lisible dit de quelle app il s'agit ; le suffixe est ce qui le rend
unique. Ne réutilisez jamais le même pour deux apps, et ne le changez jamais au
seul motif que l'URL a changé — le changer, c'est exactement ainsi que l'on se
retrouve avec une deuxième connexion, en double.

### `product_ids` — les achats que cette connexion rapporte

Des ids de produits Digistore24 séparés par des virgules :
`product_ids=111,222,333`. Par défaut, `all` : tout le compte.

**Préférez nommer les produits concernés.** Le compte d'un vendeur contient
généralement plus que l'app que vous construisez — un ancien tunnel de vente,
une deuxième app, le lancement de quelqu'un d'autre — et c'est une connexion
limitée à ses propres produits qui permet de connecter deux apps du même
vendeur en même temps.

`all` est acceptable, à une condition qui relève de l'endpoint : **l'achat d'un
produit que votre app ne connaît pas doit être ignoré, pas deviné.**
Enregistrez-le si vous voulez, mais n'accordez rien pour lui. Un endpoint qui
rattache un produit inconnu à un plan par défaut distribue un accès pour un
achat qui n'a jamais été le vôtre.

**Sur une plateforme de création par IA hébergée, c'est la partie facile**, et
cela vaut la peine de le dire à l'utilisateur : l'URL de prévisualisation ou de
production d'une app Lovable, Replit, v0 ou Manus est déjà publique et en
https, l'endpoint peut donc être enregistré directement. Sur un ordinateur portable,
non — une adresse locale a d'abord besoin d'un tunnel.

## Étape 5 — avant l'argent réel : l'approbation

Un produit peut faire l'objet d'un **achat de test** immédiatement, par le
vendeur, avec le cookie d'achat de test de Digistore24 en place — ou, dans un
environnement de développement, avec le paramètre testpay sur l'URL d'achat
(**`ds24-checkout`**, Étape 4a). C'est ainsi que l'on vérifie toute la chaîne
sans faire circuler d'argent.

Vendre au public par l'intermédiaire d'un **revendeur (reseller)** exige en
plus l'**approbation de la marketplace** (`approval_status=pending`) — ne la
demandez qu'une fois la description et l'app réellement terminées : un produit
à moitié construit est rejeté, et la deuxième tentative est plus lente.

**Un Direct Seller n'a aucune étape d'approbation.** Seuls les siteowners 1
(Allemagne), 2 (États-Unis), 3 (Royaume-Uni) et 4 (Irlande) sont des revendeurs
et approuvent des produits ; un vendeur qui vend sur son propre compte n'a rien
à demander et rien à attendre. Vérifiez à qui vous avez affaire avant de
promettre à l'utilisateur une étape d'approbation — ou de lui construire un
rappel qu'il ne pourra jamais satisfaire.

Le résultat se lit dans l'API : les éléments renvoyés par `listProducts` /
`getProduct` portent `approval_status_list`, une entrée par marketplace. La
skill **`ds24-golive`** (Étape 4) décrit le champ, ses valeurs possibles et ses
pièges — et parcourt toute la mise en production, achat de test compris.

## Étape 6 — prouver la connexion

N'annoncez pas un succès sur la seule foi d'une réponse d'API. Vérifiez que :

1. `GET <votre URL IPN>` répond **200** depuis l'internet public.
2. Le produit apparaît dans le compte Digistore24 de l'utilisateur.
3. `DIGISTORE_IPN_PASSPHRASE` est définie dans l'environnement de l'app — pas
   seulement dans un fichier local que l'app déployée ne lit jamais.

Prouvez ensuite l'endpoint lui-même — la skill **`ds24-ipn`** dit ce qui doit
être vrai et comment le vérifier sur cette plateforme.

## Étape 7 — `getPurchase` : consultez la commande vous-même

Quand l'utilisateur dit *« j'ai acheté et il ne s'est rien passé »*, ne
l'envoyez pas lire un statut dans son backoffice Digistore24 pour vous le
dicter. Interrogez l'API :

```
POST https://www.digistore24.com/api/call/getPurchase/format/json
Header: X-DS-API-KEY: <la clé>
Body:   purchase_id=ABC12345
```

Elle renvoie la vue que Digistore24 a de cette commande — statut, produit,
acheteur, type de facturation, prochain paiement, et les liens de gestion
(facture, reçu, arrêt du renouvellement automatique, mise à jour des
coordonnées de paiement). L'appel ne modifie rien ; on peut donc le faire sans
risque en plein diagnostic. `listPurchases` fait la même chose pour plusieurs
commandes, avec un filtre (par exemple l'email de l'acheteur).

**Intégrez-le à l'app sous la forme d'un petit utilitaire admin ou CLI dès la
première fois que vous en avez besoin** — un litige devient une simple
requête. La réponse range la réclamation dans des cas qui n'ont rien à
voir les uns avec les autres :

| Ce que dit `getPurchase` | Ce qui ne va pas, en réalité |
|---|---|
| **Id inconnu / aucune donnée** | il n'y a pas eu d'achat, ou il a été fait dans un autre compte Digistore24 que celui de la clé que vous utilisez. L'app n'y est pour rien |
| **Il connaît la commande, votre app non** | elle a été payée et aucune IPN ne vous est parvenue. Regardez la connexion : l'URL enregistrée répond-elle encore, un autre projet a-t-il écrasé le `domain_id`, ce produit figure-t-il dans les `product_ids` de la connexion ? |
| **Les deux la connaissent, mais l'accès manque** | l'IPN est arrivée, et c'est dans la correspondance événement→accès que se trouve le défaut → **`ds24-entitlements`** |

Une IPN rejetée est un quatrième cas, avec son propre outil : la vérification
de signature de **`ds24-ipn`**, lancée sur le corps brut tel qu'il est arrivé.

## Étape 8 — la suite

- **`ds24-ipn`** — l'endpoint qui reçoit les événements (construisez-le
  maintenant s'il n'existe pas).
- **`ds24-checkout`** — le lien d'achat, en choisissant la façon de payer.
- **`ds24-golive`** — l'achat de test qui prouve toute la chaîne.

Dites laquelle vous commencez, et commencez-la.
