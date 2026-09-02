---
name: ds24-golive
language: fr
description: À utiliser quand une intégration Digistore24 est construite et doit être prouvée avant que de vrais clients ne l'atteignent — la vérification préalable, l'achat de test avec le cookie de test Digistore24, l'approbation de la marketplace et les contrôles de mise en production sur le domaine de production. À utiliser dès que l'utilisateur dit vouloir passer en production, lancer, vendre pour de vrai, faire un achat de test, ou demande si l'intégration de paiement est réellement prête.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# La mise en production

L'intégration est construite. Prouvez maintenant qu'elle fait circuler l'argent
et débloque le produit — avant que quelqu'un d'autre que vous ne découvre qu'elle
ne le fait pas.

**Ne sautez pas directement à l'approbation.** Un produit approuvé et public avec
une IPN cassée vend un accès que personne ne reçoit, et chacun de ces cas est un
remboursement plus une conversation avec le support.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez avec le `VERSION` de ce pack. Signalez tout écart en une phrase, puis
continuez.

## Étape 1 — la vérification préalable

Passez en revue ces points et **rendez compte de chacun avec ce que vous avez
réellement vu**, pas avec une coche :

| Contrôle | Passe quand |
|---|---|
| L'app est joignable sur son domaine **https public** | une requête venue de l'extérieur répond |
| `GET <domain>/api/ipn` (ou votre chemin) | répond **200**, sans redirection |
| `DIGISTORE_IPN_PASSPHRASE` est définie **dans l'environnement déployé** | pas seulement dans un fichier local |
| `DIGISTORE_API_KEY` est définie dans l'environnement déployé | — |
| La connexion IPN chez Digistore24 pointe vers le domaine **de production** | pas vers un tunnel ni vers une URL de prévisualisation issue du développement |
| Le produit existe et son prix correspond à votre liste de prix | — |
| Les secrets sont dans le coffre à secrets de la plateforme | pas dans le dépôt |

La cinquième ligne est celle qui mord après un redéploiement : une URL de
prévisualisation datant de la construction de la chose est toujours enregistrée,
et chaque achat réel part vers une adresse qui ne répond plus.

Repointez-la en appelant `ipnSetup` à nouveau avec le **même `domain_id`** et
l'URL de production — cela met la connexion à jour. Un nouveau `domain_id` en
crée une deuxième et laisse la première, morte, en place (**`ds24-products`**,
Étape 4). Pendant que vous y êtes, vérifiez que les `product_ids` de la connexion
couvrent les produits que vous vous apprêtez réellement à vendre.

## Étape 2 — la signature, une fois de plus, contre la production

Ce qui a prouvé l'endpoint pendant le développement, exécutez-le à nouveau
**contre le domaine de production**. La skill **`ds24-ipn`** détient le comment —
sa référence de vérification donne deux formes, et l'étape 2 a besoin de celle
qui passe par HTTP depuis l'extérieur : un test à l'intérieur de l'app exerce le
handler, pas le déploiement, et les défaillances qui n'apparaissent que
maintenant sont des défaillances de déploiement. Un proxy qui réécrit le corps,
une passphrase qui n'est jamais arrivée dans l'environnement déployé.

Si `ds24-ipn` n'est pas installée, installez-la — cette étape ne peut pas être
faite correctement sans ce qu'elle dit.

Au vert, **sans aucun saut**, sinon ce n'est pas prêt. Une exécution avec des
contrôles d'accès sautés signifie que la signature est prouvée et que la
sémantique ne l'est pas — dites-le clairement plutôt que d'appeler cela du vert.

Supprimez l'endpoint de sonde dès que cela passe.

## Étape 3 — l'achat de test

C'est l'étape qui ne peut être remplacée par rien d'autre, parce que c'est la
seule qui exerce aussi le côté de Digistore24.

1. Le vendeur pose le **cookie d'achat de test Digistore24** dans son navigateur.
   (Le centre d'aide de Digistore24 a le lien qui le pose ; il est par navigateur
   et il expire.) Sur le domaine de production, le cookie est le bon outil — le
   paramètre testpay du développement (**`ds24-checkout`**, Étape 4a) appartient
   à des URL que les clients ne peuvent jamais atteindre, donc ne le reprenez pas
   ici.
2. Achetez le produit par le lien d'achat de l'app elle-même — pas par un lien
   que vous auriez construit à la main pour le test.
3. Surveillez : le checkout affiche **votre** prix et votre intervalle ; la page
   de remerciement se charge ; l'IPN arrive ; la commande est enregistrée ;
   **l'accès apparaît dans l'app**. Si l'IPN n'arrive pas, `getPurchase`
   (**`ds24-products`**, Étape 7) dit si Digistore24 a seulement enregistré
   l'achat — c'est là la différence entre un checkout raté et une connexion
   cassée, et vous ne pouvez pas les distinguer depuis l'app seule.
4. Connectez-vous en tant que ce client et confirmez que la chose payée est
   réellement utilisable.

Puis l'autre moitié, celle que les gens sautent et ne devraient pas :

5. **Remboursez l'achat de test** depuis le compte Digistore24.
6. Confirmez que l'accès a **disparu** dans l'app.

Un achat qui accorde l'accès prouve la moitié de l'intégration. Le remboursement
prouve la moitié qui vous protège.

## Étape 4 — l'approbation

L'approbation concerne les produits **de production**. Si vous avez gardé un jeu
de développement séparé (`ds24-products`, Étape 2), ne le soumettez jamais — un
produit « [DEV] » sur une marketplace est un rejet que vous avez demandé, et les
achats de test n'ont besoin d'aucune approbation.

**D'abord : l'approbation s'applique-t-elle seulement à ce vendeur ?** Seuls les
quatre **revendeurs** (resellers) de Digistore24 approuvent des produits —
l'Allemagne (`siteowner_id` 1), les USA (2), le Royaume-Uni (3), l'Irlande (4).
Tout autre siteowner est un **Direct Seller** : le vendeur vend sur son propre
compte, et il n'y a pas d'étape d'approbation, rien à demander et rien à
attendre. **Sautez toute cette étape pour eux**, et ne construisez pas non plus
de rappel à ce sujet — un rappel insistant pour une approbation qui ne peut pas
exister ne s'éteint jamais, et le vendeur n'y peut rien.

Deux façons de le savoir : un siteowner configuré hors de 1–4, ou un produit dont
l'`approval_status_list` n'a aucune entrée de revendeur *active*
(`is_siteowner_active`). Notez qu'un `approval_status` sur une entrée qui n'est
pas celle d'un revendeur ne signifie rien — le lire comme un verdict invente une
approbation que personne n'a accordée.

Tout ce qui suit vaut pour un vendeur reseller.

Demandez l'approbation de la marketplace (`approval_status=pending`) une fois que
la description du produit et l'app sont véritablement terminées — un produit à
moitié construit se fait rejeter, et la deuxième tentative est plus lente que la
première.

Jusqu'à l'approbation, les achats de test du vendeur sont les seuls achats
possibles. C'est l'état correct dans lequel être pendant la construction.

⚠️ **L'approbation est un RÉFÉRENCEMENT sur la marketplace, c'est donc aussi le
moment où le formulaire de commande propre au produit devient quelque chose que
des inconnus trouvent.** Ils paient alors le plan de paiement stocké du produit —
celui de Digistore24 par défaut, que personne n'a fixé, pas votre prix
(**`ds24-products`**, Étape 2). Cet achat est bien réel : il arrive sur votre
IPN, et si votre gestionnaire accorde l'accès sur `on_payment`, il l'accorde.
Approuvez parce que la marketplace est voulue, en sachant que cela vient avec.

**La marketplace à laquelle vous soumettez suit la langue du PRODUIT**, pas celle
de l'app : un produit allemand va chez Digistore24 GmbH, Allemagne
(`siteowner_id` 1), tout le reste chez Digistore24 Inc., USA (2). Déduire une
marketplace unique d'un réglage valable pour toute l'app est l'erreur à éviter
ici, parce que cela soumet en silence votre offre anglaise au revendeur allemand.

**Et une app multilingue a plus de produits que d'offres.** Un produit
Digistore24 porte exactement une langue — c'est la langue du formulaire de
commande de l'acheteur, voir **`ds24-products`** — donc une offre vendue en
allemand et en anglais, ce sont *deux* produits, soumis à *deux* marketplaces,
chacun recevant son propre verdict.

C'est là le piège de cette étape : **approuvé en Allemagne ne dit rien du jumeau
anglais.** Un affichage d'état qui rend compte par offre au lieu de par produit
montre un feu vert alors que la moitié de la boutique ne peut pas être vendue, et
le produit anglais est celui que personne ne pense à soumettre. Itérez sur les
produits, pas sur les offres, et vérifiez que chacun d'eux atteint `approved`.

**Qu'elle ait été accordée se lit, ne se devine pas.** Chaque élément de
`listProducts` / `getProduct` porte `approval_status_list` — une entrée par
marketplace (`reseller_id`) avec `approval_status` valant `new` (jamais
demandée), `pending`, `approved` ou `rejected`, plus `is_siteowner_active` et les
champs du motif de rejet. Le champ n'est pas dans la documentation officielle de
l'API (vérifié empiriquement en 2026-07), donc lisez-le sur la défensive : une
liste absente ou une valeur inconnue signifie « impossible à dire », pas un état.

**Il y a deux questions différentes, et elles demandent des lectures
différentes :**

| Question | Comment lire la liste |
|---|---|
| *Ce produit peut-il seulement être vendu ?* — pour un affichage d'état ou un rappel | Agrégez sur toutes les marketplaces pour lesquelles le compte est **actif** : **approuvé quelque part l'emporte**, sinon pending, sinon rejected, sinon new. Un produit approuvé en Allemagne se vend en Allemagne quoi qu'ait décidé le revendeur américain |
| *Dois-je demander l'approbation ici ?* — avant une écriture | L'entrée de **ce seul** `reseller_id`. Un produit approuvé en Allemagne peut encore avoir une demande légitime à faire aux USA |

Ignorez une entrée dont `is_siteowner_active` vaut `"N"` : cette marketplace ne
peut pas agir, donc son verdict ne dit rien — et la traiter comme un état réel
produit un avertissement à propos d'une marketplace que personne ne peut
utiliser.

Quatre règles pour l'écriture elle-même :

- **`pending` est le seul statut qui vaille la peine d'être écrit.**
  `updateProduct` acceptera les autres, et c'est là le piège : écrire `approved`
  sur votre propre produit fait croire à tout affichage d'état qu'il se vend, si
  bien que le rappel que vous aviez construit se tait pour un produit qu'aucun
  revendeur n'a jamais regardé. `new` retire une demande déjà en file d'attente.
  `approved` et `rejected` appartiennent au revendeur.
- **Ne redemandez pas pour un produit déjà `approved` sur la marketplace à
  laquelle vous écrivez.** Le côté revendeur ne décide que sur des produits
  `pending`, et savoir si écrire `pending` par-dessus une approbation la
  réinitialise n'est pas documenté — ce n'est pas une expérience à mener sur un
  compte de production.
- **N'écrivez pas quand vous n'avez pas pu lire.** Si l'appel de statut a échoué,
  ou si le produit manque dans la réponse, vous ne pouvez pas exclure une
  approbation existante — donc refusez et dites pourquoi, plutôt que de demander
  à l'aveugle. Échouer en mode ouvert ici, c'est ainsi qu'un produit approuvé et
  qui se vend se retrouve remis en pending.
- **N'écrivez pas à une marketplace dont `is_siteowner_active` vaut `"N"`.**
  L'appel réussit, personne là-bas ne le regardera jamais, et un affichage d'état
  qui filtre les marketplaces inactives continuera de signaler le produit comme
  jamais soumis — pour toujours, sans que répéter la demande n'y change rien.

Un produit `rejected` nomme son motif dans le compte vendeur Digistore24.
Corrigez-le là **d'abord** : le resoumettre inchangé le fait rejeter à nouveau,
et la deuxième tentative est plus lente que la première.

## Étape 5 — le jour où c'est en production

**D'abord, faites tourner la clé d'achat de test** si l'intégration a jamais
utilisé le paramètre testpay pendant le développement (**`ds24-checkout`**,
Étape 4a) : appelez `getTestpayKey` avec `do_recreate=1`. La clé est au niveau du
compte — une copie qui aurait circulé pendant la construction débloquerait des
« achats » de test sur le checkout de production pour quiconque la détient
encore. La faire tourner invalide toutes les anciennes copies en un seul appel.

Dites ces trois choses à l'utilisateur en mots simples, parce qu'aucune n'est
évidente :

- **Surveillez le premier achat réel.** Pas le tableau de bord — l'app. Savoir si
  l'accès est apparu est la seule question qui compte.
- **Conservez les payloads IPN bruts.** C'est avec eux que tout litige des
  prochains mois se règle.
- **Un paiement qui arrive non attribué est normal, pas un bug.** Quelqu'un a
  acheté sans compte, ou avec une adresse différente. Ayez un moyen de le
  rattacher à la main (voir **`ds24-entitlements`**, octrois manuels) avant d'en
  avoir besoin dans l'urgence.

## Étape 6 — la suite

- **`ds24-compliance`** — les pages légales et les obligations que déclenche une
  app en production, payante, dans l'UE. Faites-le avant les vrais clients, pas
  après.

Dites s'il faut la commencer.
