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

**Gardez les plans dans un seul fichier de votre projet** — clé, nom affiché,
prix en centimes, devise, intervalle de facturation — et faites-y lire tout le
reste : la page de tarifs, le checkout, la vérification du droit d'accès.

Le prix ne vit **pas** sur le produit Digistore24. L'API de Digistore24 ignore
`data[amount]` dans `createProduct`/`updateProduct` (« obsolète — créez plutôt
un plan de paiement »), et un plan de paiement stocké chez Digistore24 est
figé : essais gratuits, montées en gamme, descentes en gamme, bons de réduction
et commissions d'affiliation par lien ne fonctionnent que si le plan accompagne
l'appel de checkout. Le prix est donc transmis à `createBuyUrl` au moment de
l'achat — voir la skill **`ds24-checkout`**.

Un prix, un seul endroit. Une deuxième liste dans le code est une liste qui
dérive.

🚨 **Le produit n'est pas pour autant sans plan : Digistore24 lui en attribue
un par défaut.** (Environ 27 €, paiement unique, tel qu'observé sur un compte
réel en septembre 2026 ; regardez le produit du vendeur plutôt que de vous fier
à ce chiffre. Ce qui ne varie pas, c'est qu'*un* plan est là.) Votre app ne le
facture jamais : le plan qui accompagne l'appel `createBuyUrl` l'emporte sur le
plan stocké, à chaque fois. Ce qui le facture, c'est le **formulaire de
commande propre** au produit — il existe dès que le produit existe, et après
l'approbation de la marketplace (**`ds24-golive`**), des inconnus le trouvent.

Deux conséquences en découlent, et toutes deux passent facilement inaperçues :

- **Prévenez le vendeur avant qu'il n'ouvre son backoffice.** Il y verra un prix
  qu'il n'a jamais fixé, à côté d'un produit que son app vend à un autre tarif.
  Annoncé d'avance, c'est une curiosité ; découvert seul, cela ressemble à une
  anomalie — et la réparation qui vient à l'esprit est une deuxième liste de prix.
- **Décidez sciemment ce que votre handler IPN fait d'un achat passé par là.**
  Il n'a pas de `tracking[custom]`, son prix est celui du plan du produit et non
  le vôtre, et si votre offre est un **abonnement**, il n'enverra qu'un seul
  événement de paiement — jamais de renouvellement, jamais de résiliation :
  rien de ce que vous accrochez à ces événements ne se déclenchera pour lui.
  L'Étape 2 de **`ds24-checkout`** explique pourquoi l'absence de `custom`, à
  elle seule, ne permet pas de savoir que c'est ce qui s'est passé.

**Si votre app parle plusieurs langues, l'entrée porte un id de produit par
langue** — pas un id unique. La raison est à l'Étape 3 ; fixez la bonne forme
dès maintenant, parce que la changer après la première vente signifie de
nouveaux produits et de nouvelles approbations :

```
pro:
  name:      "Pro"
  priceCents: 3900
  productIds:            # un produit Digistore24 par langue
    de: null
    en: null
```

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

`createProduct` / `updateProduct` avec le nom, la description et **`language`**.
Reportez l'id de produit renvoyé dans votre liste de prix : la correspondance
est ainsi enregistrée, pas recalculée à chaque fois.

### Un produit par offre ET par langue — c'est ici que l'on se trompe le plus souvent

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

**Retirer un produit de votre liste ne le dépublie pas.** Un produit que
Digistore24 connaît déjà reste achetable jusqu'à ce que l'utilisateur le
désactive là-bas. Dites-le explicitement quand vous en retirez un.

🚨 **Autrement dit, le moment de demander, c'est AVANT de créer, pas après.**
Aucun appel d'API n'annule un `createProduct`. Une fois votre synchronisation
passée, chaque entrée qu'elle a trouvée est un produit réel dans le compte de
l'utilisateur, et s'en débarrasser est une opération à la main dans le
backoffice Digistore24 — pour chaque produit, dans chaque langue. Une liste de
prix qui contient encore les entrées esquissées pendant la conception de
l'offre les publiera toutes.

Donc, la première fois que votre synchronisation s'apprête à créer quoi que ce
soit : **affichez ce qui serait créé, nom par nom, prévenez l'utilisateur que
c'est irréversible, et attendez un oui.** Puis créez. Les passages suivants
trouvent les ids déjà enregistrés et ne créent rien : c'est une question posée
une fois, à un moment précis, pas une invite que l'on apprend à valider sans
lire. Si certaines entrées sont des brouillons et non des offres, donnez à votre
liste un drapeau qui les tient à l'écart de la synchronisation, plutôt que de
demander à l'utilisateur de supprimer un texte qu'il veut garder.

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
- **`ds24-checkout`** — le lien d'achat, avec le prix attaché.
- **`ds24-golive`** — l'achat de test qui prouve toute la chaîne.

Dites laquelle vous commencez, et commencez-la.
