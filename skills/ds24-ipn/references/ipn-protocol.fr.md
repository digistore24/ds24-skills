<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`ipn-protocol.md`](ipn-protocol.md) · [Español](ipn-protocol.es.md)

# La signature IPN de Digistore24

Digistore24 raconte à votre app que de l'argent a bougé en postant un payload
encodé en formulaire vers un endpoint que vous avez enregistré. Cet endpoint est
sur l'internet public, donc n'importe qui peut y faire un POST. **La signature
est la seule chose qui sépare un vrai paiement de quelqu'un qui tape une URL
dans `curl`.** Tout le reste de ce document découle de cette seule phrase.

Implémentation de référence publiée par Digistore24 :
<https://www.digistore24.com/download/ipn/examples/ipn/sha_sign.php>

## L'algorithme — six étapes

Étant donné les paramètres reçus en POST et votre **passphrase IPN** :

1. **Retirez `sha_sign` et `SHASIGN`** de l'ensemble des paramètres (comparez
   les noms sans tenir compte de la casse). Ils portent la signature elle-même
   et ne faisaient pas partie de ce qui a été signé.
2. **Triez les clés restantes comme des chaînes d'octets.** C'est le
   `ksort($params, SORT_STRING)` de PHP — une simple comparaison dans l'ordre
   des octets, *pas* une comparaison sensible à la locale ni insensible à la
   casse. En JavaScript, c'est `a < b ? -1 : a > b ? 1 : 0`, pas
   `a.localeCompare(b)`.
3. **Ignorez les valeurs vides.** `undefined`, `null` et `""` n'apportent rien —
   pas même leur clé. Un champ arrivé vide doit être traité comme s'il n'était
   pas arrivé du tout.
4. **Concaténez**, pour chaque paramètre restant dans l'ordre trié :
   `KEY` + `=` + `VALUE` + `PASSPHRASE`. La passphrase vient après *chaque*
   paire, pas une seule fois à la fin.
5. **Hachez le résultat avec SHA512**, encodé en UTF-8, et rendez-le en
   **hexadécimal majuscule**.
6. **Comparez** avec le `sha_sign` reçu **sans tenir compte de la casse** et en
   **temps constant**.

Exemple concret — paramètres `{order_id: "ORD-1", product_id: "42"}` avec la
passphrase `s3cret-passphrase`. Une fois triés, cela donne `order_id` avant
`product_id`, donc la chaîne qui est hachée est :

```
order_id=ORD-1s3cret-passphraseproduct_id=42s3cret-passphrase
```

Il n'y a aucun séparateur entre les paires. C'est la passphrase qui termine
chacune d'elles.

## Le piège qui coûte une journée à tout le monde

**Digistore24 signe avec les noms de champ D'ORIGINE — `order_id=…`, pas
`ORDER_ID=…`.**

L'exemple PHP officiel comporte un commutateur `convert_keys_to_uppercase`, et
le lire de haut en bas suggère que la mise en majuscules est la norme. Observé
sur des comptes Digistore24 réels, ce n'est pas le cas : les noms de champ sont
signés exactement tels qu'ils ont été envoyés. Une implémentation qui met en
majuscules sans condition calcule une signature parfaitement valide sur la
mauvaise entrée et rejette **chaque IPN réelle** avec « signature invalide » —
alors que tous vos propres tests passent, parce qu'ils signent de la même
mauvaise manière dont ils vérifient.

**Vérifiez donc contre les deux conventions.** Calculez la signature avec la
casse d'origine, et si cela ne correspond pas, recalculez-la avec les clés en
majuscules. Accepter l'une ou l'autre ne coûte rien en sécurité — les deux
variantes exigent la passphrase secrète — et cela épargne à l'exploitant d'avoir
à faire correspondre un réglage de son compte Digistore24 qu'il ne peut pas
voir.

## Échouer en mode fermé, toujours

Ce ne sont pas des cas limites. C'est la forme d'une attaque :

| Situation | Bonne réponse |
|---|---|
| Pas de `sha_sign` dans le payload | **rejeter** |
| Aucune passphrase configurée de votre côté | **rejeter** |
| Signature présente mais qui ne correspond pas | **rejeter** |
| Un événement inconnu avec une signature valide | acceptez la requête, ne changez rien |

La deuxième ligne est celle qui est mal construite. « Si aucune passphrase n'est
configurée, saute la vérification » est une ligne de code qui a l'air
raisonnable et qui transforme votre webhook de paiement en un endpoint
d'écriture public dès qu'une variable d'environnement disparaît lors d'un
redéploiement. **Une passphrase absente est un rejet, pas un contournement.**

## Lire le corps

La signature couvre les octets qui ont été envoyés. Tout ce qui les réécrit la
casse :

- **Lisez le corps brut**, puis analysez-le vous-même. Un framework qui analyse,
  resérialise et vous rend un objet a pu réordonner ou réencoder quelque chose.
- **Digistore24 poste du `application/x-www-form-urlencoded`**, pas du JSON.
- **Ne rognez pas, ne mettez pas en minuscules et ne normalisez pas les
  valeurs** avant de signer. Décodez les pourcents exactement une fois, comme le
  fait l'analyse de formulaire, et laissez-les ainsi.
- **UTF-8.** Les noms d'acheteurs portent des trémas, des accents et des
  écritures non latines. Hachez les octets UTF-8 ; un langage qui retombe par
  défaut sur Latin-1 (anciennes installations Python, certaines configurations
  PHP) produira silencieusement un hachage différent pour `Jörg Müller` de celui
  du même nom signé par Digistore24.

## Répondre

- **Répondez `200` avec un corps court** une fois l'événement traité.
  Digistore24 **réessaie jusqu'à obtenir un 200**, donc une exception non gérée
  se transforme en boucle de relivraison sans fin.
- **Répondez au test de connexion.** Digistore24 valide l'endpoint quand vous
  l'enregistrez, et il le fait avec un `GET`. Renvoyez `200 OK` pour un GET, et
  pour un POST dont l'événement est `connection_test`.
- **Ne redirigez jamais.** Un `301`/`302` depuis votre endpoint IPN échoue à la
  validation — Digistore24 veut l'endpoint lui-même, pas un saut.
- **L'URL doit être en `https` public.** Digistore24 refuse `http` et refuse
  `localhost` catégoriquement. Sur une plateforme hébergée de création par IA,
  votre URL de prévisualisation est déjà en https public, ce qui est la seule
  chose qui y est *plus facile* que sur un portable.

## La livraison est sans ordre et sans limite

Deux propriétés du transport auxquelles votre handler doit survivre, parce
qu'aucune des deux n'apparaît pendant les tests :

- **Les événements arrivent dans le désordre.** Un `on_payment` relivré peut
  atterrir *après* le `on_refund` qui a mis fin à l'accès. La décision « cette
  personne peut-elle utiliser le produit ? » doit donc être prise à partir de
  l'**état**, pas d'un horodatage ni de l'ordre d'arrivée. Une fois que l'accès
  a pris fin, aucun événement ultérieur ne peut le rouvrir.
- **Le même événement arrive plus d'une fois.** Digistore24 réessaie jusqu'à
  obtenir un 200, et un timeout de votre côté après que le travail a été fait
  compte quand même comme un échec. **Chaque écriture que fait votre
  handler doit être idempotente**, avec une clé tirée du payload —
  `order_id` plus le nom de l'événement. `order_id` est l'identifiant que
  Digistore24 garantit, et il est documenté comme *« ID unique de la commande.
  Plusieurs transactions de la même commande ont le même order-ID »* : le
  paiement, son remboursement, une rétrofacturation et chaque renouvellement
  automatique d'un abonnement arrivent tous en portant cette même valeur. C'est
  ce qui en fait à la fois la clé d'idempotence et la clé sous laquelle l'accès
  lui-même est stocké — un remboursement ne peut révoquer que ce qu'un paiement
  a accordé si les deux s'accordent sur l'identifiant. Créditer un solde de
  tokens sans une telle clé distribue les crédits deux fois.

  ⚠️ **Une IPN ne porte aucun `purchase_id`.** Il n'apparaît dans aucune table
  publiée de paramètres IPN, et le message réel dans `../scripts/vectors.json`
  (`captured-on-payment`, 173 paramètres) ne le contient pas. Le nom appartient
  à l'**API** Digistore24, où `getPurchase` documente son `purchase_id` comme
  « l'order id Digistore24 » — la même valeur sous un autre nom. Indexez vos
  écritures dessus et vous les indexez sur `undefined` dans chaque message qui
  arrivera jamais : soit toutes les commandes s'effondrent sur une seule clé,
  soit rien ne correspond jamais et la nouvelle tentative refait le travail une
  seconde fois. Les deux échecs ressemblent à un endpoint qui fonctionne jusqu'à
  ce que de l'argent réel bouge.

## Quoi stocker

Stockez le **payload brut** de chaque IPN que vous acceptez, mot pour mot, avant
d'agir dessus. Cela ne coûte presque rien et c'est la seule façon de répondre
« Digistore24 a-t-il vraiment envoyé cela ? » des semaines plus tard, quand un
client conteste son accès et que vos tables dérivées sont toutes d'accord entre
elles.

Stockez le **nom de l'événement** tel qu'il est arrivé. Ne réduisez pas les
événements à un statut pour ensuite décider à partir de ce statut — voyez
`events.fr.md`, où deux événements qui disent le contraire au sujet de l'accès
se réduisent au même mot.
