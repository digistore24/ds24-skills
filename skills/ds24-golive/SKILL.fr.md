---
name: ds24-golive
language: fr
description: À utiliser quand une intégration Digistore24 est construite et qu'il faut la prouver avant que de vrais clients n'y arrivent — la vérification préalable, l'achat de test avec le cookie de test Digistore24, l'approbation de la marketplace et les contrôles de mise en production sur le domaine de production. À utiliser dès que l'utilisateur dit vouloir passer en production, lancer, vendre pour de vrai, faire un achat de test, ou demande si l'intégration de paiement est réellement prête.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# La mise en production

L'intégration est construite. Reste à prouver qu'elle fait circuler l'argent et
qu'elle débloque le produit — avant que quelqu'un d'autre que vous ne découvre
que ce n'est pas le cas.

**Ne passez pas directement à l'approbation.** Un produit approuvé et public
dont l'IPN est cassée vend un accès que personne ne reçoit, et chacune de ces
ventes finit en remboursement, plus une conversation avec le support.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez-le au `VERSION` de ce pack. En cas d'écart, signalez-le en une
phrase, puis poursuivez.

## Étape 1 — la vérification préalable

Passez ces points en revue et **rendez compte de chacun avec ce que vous avez
réellement observé**, pas d'une simple coche :

| Contrôle | Réussi quand |
|---|---|
| L'app répond sur son domaine **public en https** | une requête venue de l'extérieur obtient une réponse |
| `GET <domain>/api/ipn` (ou votre chemin) | répond **200**, sans redirection |
| `DIGISTORE_IPN_PASSPHRASE` est définie **dans l'environnement déployé** | pas seulement dans un fichier local |
| `DIGISTORE_API_KEY` est définie dans l'environnement déployé | — |
| La connexion IPN côté Digistore24 pointe vers le domaine **de production** | pas vers un tunnel ni vers une URL de prévisualisation héritée du développement |
| Le produit existe et son prix correspond à votre liste de prix | — |
| Les secrets sont dans le coffre à secrets de la plateforme | pas dans le dépôt |

La cinquième ligne est celle qui fait mal après un redéploiement : l'URL de
prévisualisation utilisée pendant la construction est restée enregistrée, et
chaque achat réel part vers une adresse qui ne répond plus.

Faites-la pointer sur la production en rappelant `ipnSetup` avec le **même
`domain_id`** et l'URL de production — c'est cela qui met la connexion à jour.
Un nouveau `domain_id` en crée une deuxième et laisse la première en place,
morte (**`ds24-products`**, Étape 4). Tant que vous y êtes, vérifiez que les
`product_ids` de la connexion couvrent bien les produits que vous êtes sur le
point de vendre.

## Étape 2 — la signature, encore une fois, sur le domaine de production

Ce qui a servi à prouver l'endpoint pendant le développement, relancez-le
**sur le domaine de production**. La skill **`ds24-ipn`** explique comment
faire : sa référence de vérification décrit deux formes, et l'étape 2 demande
celle qui passe par HTTP depuis l'extérieur. Un test à l'intérieur de l'app
exerce le handler, pas le déploiement — or les défaillances qui n'apparaissent
qu'à ce stade sont des défaillances de déploiement : un proxy qui réécrit le
corps, une passphrase qui n'est jamais arrivée dans l'environnement déployé.

Si `ds24-ipn` n'est pas installée, installez-la — sans ce qu'elle dit, cette
étape ne peut pas être faite correctement.

Au vert, **sans aucun saut**, sinon ce n'est pas prêt. Une exécution qui a sauté
les contrôles d'accès prouve la signature et pas la sémantique — dites-le
clairement plutôt que d'annoncer du vert.

Supprimez l'endpoint de sonde une fois ce contrôle passé.

## Étape 3 — l'achat de test

C'est l'étape que rien d'autre ne peut remplacer : c'est la seule qui exerce
aussi le côté Digistore24.

1. Le vendeur pose le **cookie d'achat de test Digistore24** dans son
   navigateur. (Le centre d'aide de Digistore24 fournit le lien qui le pose ;
   il vaut pour un seul navigateur et il expire.) Sur le domaine de production,
   le cookie est le bon outil — le paramètre testpay du développement
   (**`ds24-checkout`**, Étape 4a) n'a sa place que sur des URL que les clients
   ne peuvent jamais atteindre, donc ne le reprenez pas ici.
2. Achetez le produit par le lien d'achat de l'app elle-même — pas par un lien
   construit à la main pour le test.
3. Surveillez : le checkout affiche **votre** prix et votre intervalle ; la
   page de remerciement se charge ; l'IPN arrive ; la commande est
   enregistrée ; **l'accès apparaît dans l'app**. Si l'IPN n'arrive pas,
   `getPurchase` (**`ds24-products`**, Étape 7) dit si Digistore24 a bel et
   bien enregistré l'achat — c'est ce qui distingue un checkout raté d'une
   connexion cassée, et l'app seule ne permet pas de les départager.
4. Connectez-vous en tant que ce client et confirmez que ce qui a été payé est
   réellement utilisable.

Puis l'autre moitié, celle qu'on saute et qu'il ne faut pas sauter :

5. **Remboursez l'achat de test** depuis le compte Digistore24.
6. Confirmez que l'accès a **disparu** dans l'app.

Un achat qui accorde l'accès prouve la moitié de l'intégration. Le
remboursement prouve celle qui vous protège.

## Étape 4 — l'approbation

L'approbation concerne les produits **de production**. Si vous avez gardé un
jeu de développement à part (`ds24-products`, Étape 2), ne le soumettez
jamais : un produit « [DEV] » sur une marketplace, c'est un rejet que vous
aurez cherché, et les achats de test n'ont besoin d'aucune approbation.

**D'abord : ce vendeur est-il même soumis à l'approbation ?** Seuls les
quatre **revendeurs** (resellers) de Digistore24 approuvent des produits —
Allemagne (`siteowner_id` 1), USA (2), Royaume-Uni (3), Irlande (4). Tout
autre siteowner est un **Direct Seller** : le vendeur vend pour son propre
compte, et il n'y a ni étape d'approbation, ni demande à faire, ni verdict à
attendre. **Sautez toute cette étape pour eux**, et ne leur construisez pas non
plus de rappel à ce sujet — une relance pour une approbation qui ne peut pas
exister ne s'éteint jamais, et le vendeur n'y peut rien.

Deux façons de le savoir : un siteowner configuré hors de 1–4, ou un produit
dont l'`approval_status_list` ne contient aucune entrée de revendeur *active*
(`is_siteowner_active`). Notez qu'un `approval_status` sur une entrée qui n'est
pas celle d'un revendeur ne veut rien dire — le lire comme un verdict, c'est
inventer une approbation que personne n'a accordée.

Tout ce qui suit vaut pour un vendeur qui passe par un revendeur.

Demandez l'approbation de la marketplace (`approval_status=pending`) une fois
que la description du produit et l'app sont réellement terminées — un produit
à moitié construit se fait rejeter, et la deuxième tentative est plus lente que
la première.

Jusqu'à l'approbation, les achats de test du vendeur sont les seuls achats
possibles. C'est l'état normal tant que vous construisez.

⚠️ **L'approbation est un RÉFÉRENCEMENT sur la marketplace : c'est donc aussi
le moment où le formulaire de commande propre au produit devient trouvable par
des inconnus.** Ils paient alors le plan de paiement enregistré sur le produit —
la valeur par défaut de Digistore24, que personne n'a choisie, pas votre prix
(**`ds24-products`**, Étape 2). Cet achat est bien réel : il arrive sur votre
IPN, et si votre handler accorde l'accès sur `on_payment`, il l'accorde.
Demandez l'approbation parce que vous voulez la marketplace, en sachant que
cela va avec.

**La marketplace à laquelle vous soumettez dépend de la langue du PRODUIT**,
pas de celle de l'app : un produit en allemand va chez Digistore24 GmbH,
Allemagne (`siteowner_id` 1), tout le reste chez Digistore24 Inc., USA (2).
Déduire une marketplace d'un réglage global à l'app est l'erreur à éviter
ici : elle soumet en silence votre offre anglaise au revendeur allemand.

**Et une app multilingue a plus de produits que d'offres.** Un produit
Digistore24 porte exactement une langue — celle du formulaire de commande de
l'acheteur, voir **`ds24-products`** — si bien qu'une offre vendue en allemand
et en anglais fait *deux* produits, soumis à *deux* marketplaces, chacun avec
son propre verdict.

C'est là le piège de cette étape : **approuvé en Allemagne ne dit rien du
jumeau anglais.** Un affichage d'état qui rend compte par offre et non par
produit montre un feu vert alors que la moitié de la boutique ne peut pas être
vendue, et le produit anglais est celui que personne ne pense à soumettre.
Itérez sur les produits, pas sur les offres, et vérifiez que chacun d'eux
atteint `approved`.

**Savoir si elle a été accordée, cela se lit, cela ne se devine pas.** Chaque
élément renvoyé par `listProducts` / `getProduct` porte `approval_status_list`
— une entrée par marketplace (`reseller_id`), avec `approval_status` valant
`new` (jamais demandée), `pending`, `approved` ou `rejected`, plus
`is_siteowner_active` et les champs du motif de rejet. Ce champ ne figure pas
dans la documentation officielle de l'API (vérifié empiriquement en 2026-07),
donc lisez-le avec prudence : une liste absente ou une valeur inconnue signifie
« impossible à dire », pas un état.

**Il y a deux questions distinctes, et elles ne se lisent pas de la même
façon :**

| Question | Comment lire la liste |
|---|---|
| *Peut-on vendre ce produit, oui ou non ?* — pour un affichage d'état ou un rappel | Agrégez sur toutes les marketplaces pour lesquelles le compte est **actif** : **approuvé quelque part l'emporte**, sinon pending, sinon rejected, sinon new. Un produit approuvé en Allemagne se vend en Allemagne, quoi qu'ait décidé le revendeur américain |
| *Dois-je demander l'approbation ici ?* — avant une écriture | L'entrée de **ce seul** `reseller_id`. Un produit approuvé en Allemagne peut encore avoir une demande légitime à faire aux USA |

Ignorez toute entrée dont `is_siteowner_active` vaut `"N"` : cette marketplace
ne peut pas agir, son verdict ne dit donc rien — et la traiter comme un état
réel produit un avertissement au sujet d'une marketplace que personne ne peut
utiliser.

Quatre règles pour l'écriture elle-même :

- **`pending` est le seul statut qui mérite d'être écrit.** `updateProduct`
  acceptera les autres, et c'est là le piège : écrire `approved` sur votre
  propre produit fait croire à tout affichage d'état qu'il se vend, si bien que
  le rappel que vous aviez construit se tait pour un produit qu'aucun revendeur
  n'a jamais regardé. `new` retire une demande déjà dans la file. `approved` et
  `rejected` appartiennent au revendeur.
- **Ne redemandez pas l'approbation d'un produit déjà `approved` sur la
  marketplace à laquelle vous écrivez.** Le revendeur ne statue que sur les
  produits `pending`, et rien ne documente si écrire `pending` par-dessus une
  approbation la remet à zéro — ce n'est pas une expérience à tenter sur un
  compte de production.
- **N'écrivez pas quand vous n'avez pas pu lire.** Si l'appel de statut a
  échoué, ou si le produit manque dans la réponse, vous ne pouvez pas exclure
  une approbation existante — refusez donc, et dites pourquoi, plutôt que de
  demander à l'aveugle. Laisser passer dans le doute, c'est ainsi qu'un produit
  approuvé, qui se vend, se retrouve remis en pending.
- **N'écrivez pas vers une marketplace dont `is_siteowner_active` vaut `"N"`.**
  L'appel réussit, personne là-bas ne le regardera jamais, et un affichage
  d'état qui filtre les marketplaces inactives continuera de présenter le
  produit comme jamais soumis — pour toujours, et répéter la demande n'y
  changera rien.

Un produit `rejected` a son motif dans le compte vendeur Digistore24.
Corrigez-le là **d'abord** : resoumis tel quel, il est rejeté de nouveau, et la
deuxième tentative est plus lente que la première.

## Étape 5 — le jour de la mise en production

**Commencez par faire tourner la clé d'achat de test** si l'intégration a un
jour utilisé le paramètre testpay pendant le développement
(**`ds24-checkout`**, Étape 4a) : appelez `getTestpayKey` avec `do_recreate=1`.
La clé vaut pour tout le compte — une copie qui a circulé pendant la
construction débloquerait des « achats » de test sur le checkout de production
pour quiconque la détient encore. La faire tourner invalide toutes les
anciennes copies en un seul appel.

Dites ces trois choses à l'utilisateur, en termes simples, parce qu'aucune
n'est évidente :

- **Surveillez le premier achat réel.** Pas le tableau de bord — l'app. La
  seule question qui compte est de savoir si l'accès est apparu.
- **Conservez les payloads IPN bruts.** C'est avec eux que se règle tout litige
  des prochains mois.
- **Un paiement qui arrive non attribué est normal, pas un bug.** Quelqu'un a
  acheté sans compte, ou avec une autre adresse. Prévoyez un moyen de le
  rattacher à la main (voir **`ds24-entitlements`**, octrois manuels) avant
  d'en avoir besoin dans l'urgence.

## Étape 6 — la suite

- **`ds24-compliance`** — les pages légales et les obligations que déclenche
  une app payante, en production, dans l'UE. À faire avant les vrais clients,
  pas après.

Dites s'il faut la lancer.
