---
name: ds24-products
language: fr
description: À utiliser lors de la première connexion d'une app à un compte Digistore24 — mettre en place la clé d'API, créer les produits à vendre, enregistrer la connexion du webhook IPN et demander l'approbation de la marketplace. À utiliser dès que l'utilisateur mentionne une clé d'API Digistore24, « connecter Digistore », la création de produits ou de plans, l'enregistrement d'une URL IPN, ou demande pourquoi Digistore24 n'appelle jamais son webhook.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Connecter l'app à Digistore24

Rien dans une intégration Digistore24 ne fonctionne tant que trois choses
n'existent pas du côté de Digistore24 : une clé d'API que votre app peut
utiliser, un produit à vendre et une connexion IPN pointant vers votre endpoint.
**Faites cela avant tout le reste** — un handler d'IPN que personne
n'appelle est intestable, et un lien de checkout vers un produit qui n'existe pas
est un 404.

## Étape 0 — qu'est-ce qui existe déjà ?

Regardez avant de demander :

- Y a-t-il un `DIGISTORE_API_KEY` dans l'environnement ou le coffre à secrets ?
- Y a-t-il un registre de produits dans le projet (un fichier JSON/config listant
  les plans avec leurs prix) ?
- `DIGISTORE_IPN_PASSPHRASE` est-il défini ?

Ensuite, ne demandez à l'utilisateur que ce qui manque réellement. Si les trois
sont là, allez à l'Étape 4 et vérifiez la connexion au lieu de la reconstruire.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez avec le fichier `VERSION` de ce pack. Signalez tout écart en une
phrase, puis continuez.

## Étape 1 — l'API

```
POST https://www.digistore24.com/api/call/<FUNCTION>/format/json
Header: X-DS-API-KEY: <la clé>
Body:   application/x-www-form-urlencoded
```

**La clé voyage dans l'en-tête, jamais comme paramètre de formulaire.** C'est un
secret : variable d'environnement ou coffre à secrets de la plateforme, jamais
dans le code, jamais dans quoi que ce soit que le navigateur reçoit.

L'utilisateur la crée lui-même dans son compte Digistore24, sous *Paramètres →
Clés d'API* (*Settings → API keys*). Demandez-la, dites-lui où la mettre, et
n'essayez pas de l'extraire d'une session de navigateur.

⚠️ **Dites-lui de lui donner le droit d'ÉCRITURE** (Digistore24 appelle cela
*writable*). Une clé est cadrée au moment de sa création, et une clé en lecture
seule lit les produits parfaitement puis échoue sur les deux appels dont l'app ne
peut pas se passer : créer les produits et créer une URL de checkout. Dites-le
pendant qu'il est sur cet écran — revenir ensuite pour élargir une clé, cela veut
dire en créer une nouvelle et la remplacer partout.

## Étape 2 — une seule liste de prix, dans votre app

**Gardez les plans dans un seul fichier de votre projet** — clé, nom affiché,
prix en centimes, devise, intervalle de facturation — et faites que tout y
lise : la page de tarifs, le checkout, la vérification du droit d'accès.

Le prix ne vit **pas** sur le produit Digistore24. L'API de Digistore24 jette
`data[amount]` sur `createProduct`/`updateProduct` (« obsolète — créez plutôt un
plan de paiement »), et un plan de paiement stocké chez Digistore24 est figé :
les essais gratuits, les montées en gamme, les descentes en gamme, les bons de
réduction et les commissions d'affiliation par lien ne fonctionnent que si le
plan voyage avec l'appel de checkout. Le prix part donc à `createBuyUrl` au
moment de l'achat — voir la skill **`ds24-checkout`**.

Un prix, un seul endroit. Une deuxième liste dans le code est une liste qui
dérive.

🚨 **Mais le produit ne reste pas sans plan — Digistore24 lui donne le sien par
défaut.** (Environ 27 €, paiement unique, constaté sur un compte réel en
septembre 2026 ; regardez le produit du vendeur plutôt que de vous fier à ce
chiffre. Ce qui ne change pas, c'est qu'*un* plan est là.) Votre app ne le
facture jamais : un plan qui voyage avec l'appel `createBuyUrl` l'emporte
toujours sur celui qui est stocké. Ce qui le facture, en revanche, c'est le
**formulaire de commande propre** au produit, qui existe dès que le produit
existe — et après l'approbation sur la marketplace (**`ds24-golive`**), c'est
quelque chose que des inconnus trouvent.

Deux conséquences, faciles à manquer toutes les deux :

- **Prévenez le vendeur avant qu'il n'ouvre son backoffice.** Il y verra un prix
  qu'il n'a jamais fixé, à côté d'un produit que son app vend à un autre tarif.
  Annoncé, c'est une curiosité ; découvert seul, cela ressemble à une panne, et
  la réparation vers laquelle il se tourne est une deuxième liste de prix.
- **Décidez délibérément ce que votre gestionnaire d'IPN fait d'un achat passé
  là.** Il ne porte pas de `tracking[custom]`, il est facturé au plan du produit
  et non au vôtre, et si votre offre est un **abonnement**, il enverra
  exactement un événement de paiement — jamais un renouvellement, jamais une
  résiliation, donc rien de ce que vous accrochez à ces événements ne se
  déclenchera pour lui. L'Étape 2 de **`ds24-checkout`** dit pourquoi l'absence
  de `custom` ne suffit pas à savoir que c'est ce qui s'est passé.

**Si votre app parle plus d'une langue, l'entrée contient un id de produit par
langue** — pas un seul id. La raison, c'est l'Étape 3 ; donnez-lui la bonne forme
ici, car la changer après la première vente signifie de nouveaux produits et de
nouvelles approbations :

```
pro:
  name:      "Pro"
  priceCents: 3900
  productIds:            # un produit Digistore24 par langue
    de: null
    en: null
```

**Et si l'app a plus d'un environnement, gardez un JEU de produits par
environnement** (dev / prod — staging seulement s'il existe vraiment). Les
produits que vous créez contre une URL de préversion ou de développement sont des
articles de test : donnez-leur leurs propres ids dans le registre, marquez-les
visiblement dans le nom du produit (`"Pro [DEV]"` — l'API de Digistore24 n'a pas
de champ d'étiquette, le nom est ce qu'un humain voit dans le backoffice), et
laissez propres les noms des produits en production. Un jeu ne doit jamais
revendiquer les produits de l'autre — voir la note sur l'idempotence plus bas. Un
vendeur qui ne synchronise jamais que contre le domaine de production a un seul
jeu, et c'est très bien.

## Étape 3 — créez les produits

`createProduct` / `updateProduct` avec le nom, la description et **`language`**.
Réécrivez l'id de produit renvoyé dans votre liste de prix, pour que la
correspondance soit enregistrée et non redéduite.

### Un produit par offre ET par langue — c'est celui que tout le monde rate

**Un produit Digistore24 porte exactement UNE langue, et cette langue est celle
du FORMULAIRE DE COMMANDE que votre acheteur remplit** — les libellés des champs,
les boutons, les noms des moyens de paiement, les conditions de résiliation.
C'est `data[language]` sur le produit.

**`createBuyUrl` n'a pas de paramètre de langue.** Ses arguments sont
`product_id`, `buyer`, `payment_plan`, `tracking`, `valid_until`, `urls`,
`placeholders`, `settings` et `addons` — il n'y a rien là-dedans pour écraser la
langue du produit, et aucun paramètre d'URL ne le fait non plus. Vous ne pouvez
donc pas décider de la langue du formulaire au moment du checkout. Vous en
décidez en **choisissant vers quel produit vous envoyez l'acheteur**.

Une app dont l'interface parle allemand et anglais a donc besoin de **deux
produits Digistore24 par offre**, un avec `language=de` et un avec `language=en`,
et le checkout choisit selon la langue du visiteur. Envoyez tout le monde vers un
seul produit et la moitié de vos clients se voient demander leurs coordonnées
bancaires dans une langue qu'ils n'ont pas choisie — ce qui est exactement là où
un achat est abandonné.

Trois conséquences qui méritent d'être écrites dans ce que vous construirez :

- **Définissez `data[language]` explicitement sur chaque produit.** Omis,
  Digistore24 retombe sur la langue de la session API — le choix délibéré de
  personne, et la cause habituelle d'une boutique allemande qui affiche des
  formulaires de commande en anglais.
- **Couvrez toutes les langues de votre app.** Une langue manquante devrait quand
  même vendre (retombez sur un autre produit plutôt que d'afficher un bouton
  mort) — mais dites-le dans la sortie de votre synchronisation, parce que rien
  d'autre ne le fera jamais : l'app s'affiche bien, le checkout s'ouvre, l'achat
  aboutit.
- **Chaque produit de langue est approuvé séparément**, sur la marketplace à
  laquelle sa langue appartient. Voir la skill **`ds24-golive`**.

Le *texte* de votre produit est une question distincte. Envoyer le même nom et la
même description aux deux produits est une valeur par défaut parfaitement bonne —
c'est le *formulaire* autour qui doit suivre l'acheteur.

Rendez cela **idempotent** : lancez-le deux fois et le second passage met à jour
au lieu de créer un doublon. Indexez-le sur votre propre clé de produit **plus la
langue** — et, si vous gardez des jeux séparés par environnement, **plus
l'environnement** (`pro__en__prod`) — chaque produit a besoin de son propre
identifiant stable. N'indexez jamais sur le nom affiché, qui est le même pour les
deux langues et change avec le texte.

**Supprimer un produit de votre liste ne le dépublie pas.** Un produit que
Digistore24 connaît déjà reste achetable jusqu'à ce que l'utilisateur le
désactive là-bas. Dites-le à voix haute quand vous en retirez un.

🚨 **Ce qui veut dire que le moment de demander est AVANT de créer, pas après.**
Aucun appel d'API n'annule un `createProduct`. Une fois votre synchronisation
passée, chaque entrée qu'elle a trouvée est un produit réel dans le compte de
l'utilisateur, et se débarrasser d'un produit est une main dans le backoffice
Digistore24 — pour chacun, dans chaque langue. Une liste de prix qui porte encore
les entrées que vous avez esquissées en concevant l'offre les publiera toutes.

Donc la première fois que votre synchronisation créerait quoi que ce soit :
**affichez ce qui serait créé, par nom, dites à l'utilisateur que c'est
irréversible, et attendez un oui.** Puis créez. Les passages suivants ont les ids
au dossier et ne créent rien, c'est donc une question à un moment, pas une invite
que l'on apprend à cliquer sans lire. Si certaines entrées sont des brouillons
plutôt que des offres, donnez à votre liste un drapeau qui les tient hors de la
synchronisation au lieu de demander à l'utilisateur de supprimer du texte qu'il
veut encore.

## Étape 4 — enregistrez la connexion IPN

C'est l'étape qu'on oublie, et son symptôme est « l'achat a marché mais rien ne
s'est passé dans l'app ».

- `ipnSetup` enregistre l'endpoint. Digistore24 **le valide immédiatement** avec
  un `GET` et exige un HTTP `200` — une redirection (301/302) échoue aussi.
- **L'URL doit être en `https` public.** Digistore24 refuse `http` et refuse
  `localhost` sans discuter.
- Digistore24 génère la **passphrase IPN** ou prend la vôtre. Quoi qu'il en soit,
  elle doit finir dans l'environnement de l'app sous `DIGISTORE_IPN_PASSPHRASE` —
  c'est le secret partagé avec lequel la signature est calculée, et sans lui
  chaque IPN est correctement rejetée.

L'appel prend ces paramètres, et deux d'entre eux décident si les événements
arrivent un jour :

| | |
|---|---|
| `ipn_url` | votre endpoint, https public |
| `name` | le nom de la connexion dans le backoffice |
| `domain_id` | **l'identité de cette connexion** — voir plus bas |
| `product_ids` | quels produits elle couvre — ids séparés par des virgules, ou `all` |
| `sha_passphrase` | la vôtre, ou `random` pour en faire générer une et la recevoir |

### `ipnSetup` est aussi la mise à jour — c'est le `domain_id` qui décide

Il n'y a pas de fonction de mise à jour séparée. Digistore24 cherche une
connexion par **(marchand, clé d'API, `domain_id`)** : même id → la connexion
existante est mise à jour, id inconnu → une deuxième connexion voit le jour.
C'est ce qui rend l'appel idempotent, et c'est pourquoi l'id doit être **noté
quelque part** (une variable d'environnement, une ligne de paramètres) plutôt que
redéduit de quelque chose qui change.

**Et il doit être unique.** C'est la partie qu'on saute, et elle échoue
invisiblement. Une valeur générique — `test-local-1`, `local-app`, `myapp`,
`production` — n'est pas un nom, c'est une collision avec l'**autre** projet de
l'utilisateur lui-même : les deux n'obtiennent pas deux connexions, ils écrasent
la même chacun leur tour. La deuxième configuration repointe silencieusement
l'IPN de la première app vers sa propre URL, et dès lors les achats de la
première app n'arrivent nulle part. Les deux passages annoncent un succès.

Alors mettez-lui une queue aléatoire et stockez-le :

```
test-local-diw2hvnz73
myapp-prod-k7f2m9x1qc
```

La partie lisible dit de quelle app il s'agit ; la queue est ce qui le rend
unique. Ne le réutilisez jamais entre deux apps, et ne le changez jamais
simplement parce que l'URL a changé — le changer, c'est comme cela qu'on obtient
une deuxième connexion, en double.

### `product_ids` — quels achats cette connexion rapporte

Ids de produits Digistore24 séparés par des virgules : `product_ids=111,222,333`.
La valeur par défaut est `all`, tout le compte.

**Préférez nommer les produits réels.** Le compte d'un vendeur contient
généralement plus que l'app que vous construisez — un ancien tunnel de vente, une
deuxième app, le lancement de quelqu'un d'autre — et une connexion cadrée sur ses
propres produits est ce qui permet à deux apps du même vendeur d'être connectées
en même temps.

`all` est acceptable, à une condition qui appartient à l'endpoint : **l'achat
d'un produit que votre app ne connaît pas doit être ignoré, pas deviné.**
Enregistrez-le si vous voulez, n'accordez rien pour lui. Un endpoint qui fait
correspondre un produit inconnu à un plan par défaut distribue un accès pour un
achat qui n'a jamais été le vôtre.

**Sur une plateforme hébergée de création par IA, c'est la partie facile**, et
cela vaut la peine de le dire à l'utilisateur : l'URL de préversion/production
d'une app Lovable, Replit, v0 ou Manus est déjà en https public, l'endpoint peut
donc être enregistré directement. Sur un portable, non — une adresse locale a
d'abord besoin d'un tunnel.

## Étape 5 — avant l'argent réel : l'approbation

Un produit peut faire l'objet d'un **achat de test** immédiatement, par le
vendeur, avec le cookie d'achat de test de Digistore24 posé — ou, dans un
environnement de développement, avec le paramètre testpay sur l'URL d'achat
(**`ds24-checkout`**, Étape 4a). C'est ainsi que vous vérifiez toute la chaîne
sans déplacer d'argent.

Vendre au public via un **revendeur (reseller)** demande en plus l'**approbation
de la marketplace** (`approval_status=pending`) — ne la demandez qu'une fois la
description et l'app vraiment terminées, car un produit à moitié construit se
fait refuser et la deuxième tentative est plus lente.

**Un Direct Seller n'a aucune étape d'approbation.** Seuls les siteowners 1
(Allemagne), 2 (États-Unis), 3 (Royaume-Uni) et 4 (Irlande) sont des revendeurs
et approuvent des produits ; un vendeur qui vend sur son propre compte n'a rien à
demander et rien à attendre. Vérifiez à qui vous avez affaire avant de promettre
à l'utilisateur une étape d'approbation — ou de construire un rappel qu'il ne
pourra jamais satisfaire.

Savoir si elle a été accordée est lisible : les éléments de `listProducts` /
`getProduct` portent `approval_status_list`, une entrée par marketplace. La skill
**`ds24-golive`** (Étape 4) a le champ, son jeu de valeurs et ses pièges — et
parcourt toute la mise en production, y compris l'achat de test.

## Étape 6 — prouvez la connexion

N'annoncez pas un succès à partir d'une seule réponse d'API. Vérifiez que :

1. `GET <votre URL IPN>` répond **200** depuis l'internet public.
2. Le produit apparaît dans le compte Digistore24 de l'utilisateur.
3. `DIGISTORE_IPN_PASSPHRASE` est défini dans l'environnement de l'app — pas
   seulement dans un fichier local que l'app déployée ne lit jamais.

Prouvez ensuite l'endpoint lui-même — la skill **`ds24-ipn`** dit ce qui doit
tenir et comment le vérifier sur cette plateforme.

## Étape 7 — `getPurchase` : consultez une commande vous-même

Quand l'utilisateur dit *« je l'ai acheté et rien ne s'est passé »*, ne l'envoyez
pas dans son backoffice Digistore24 vous lire un statut. Demandez à l'API :

```
POST https://www.digistore24.com/api/call/getPurchase/format/json
Header: X-DS-API-KEY: <la clé>
Body:   purchase_id=ABC12345
```

Cela renvoie la vue de Digistore24 sur cette commande — statut, produit,
acheteur, type de facturation, prochain paiement, et les liens de gestion
(facture, reçu, arrêt du renouvellement automatique, mise à jour des coordonnées
de paiement). Cela ne change rien, on peut donc l'appeler sans risque pendant un
diagnostic. `listPurchases` est la même chose pour plusieurs, filtrée (par
exemple par l'email de l'acheteur).

**Intégrez-le à l'app comme un petit utilitaire admin/CLI dès la première fois
que vous en avez besoin** — cela transforme une dispute en une recherche. La
réponse trie la plainte en cas qui n'ont rien à voir les uns avec les autres :

| Ce que dit `getPurchase` | Ce qui ne va pas en réalité |
|---|---|
| **Id inconnu / aucune donnée** | il n'y a pas eu d'achat, ou il a été fait dans un autre compte Digistore24 que la clé que vous utilisez. L'app va bien |
| **Il connaît la commande, votre app non** | elle a été payée et aucune IPN ne vous est parvenue. Regardez la connexion : l'URL enregistrée répond-elle encore, un autre projet a-t-il écrasé le `domain_id`, ce produit est-il dans les `product_ids` de la connexion ? |
| **Les deux la connaissent, mais l'accès manque** | l'IPN est arrivée et c'est la correspondance événement→accès qui est en faute → **`ds24-entitlements`** |

Une IPN rejetée est un quatrième cas et a son propre outil — la vérification de
la signature dans **`ds24-ipn`**, lancée contre le corps brut qui est arrivé.

## Étape 8 — la suite

- **`ds24-ipn`** — l'endpoint qui reçoit les événements (construisez-le
  maintenant s'il n'existe pas).
- **`ds24-checkout`** — le lien d'achat, avec le prix attaché.
- **`ds24-golive`** — l'achat de test qui prouve toute la chaîne.

Dites laquelle vous commencez, et commencez-la.
