<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`ipn-protocol.md`](ipn-protocol.md) · [Español](ipn-protocol.es.md)

# La signature IPN de Digistore24

Digistore24 informe votre app que de l'argent a changé de mains en envoyant un
POST — un payload encodé en formulaire — vers l'endpoint que vous avez
enregistré. Cet endpoint est exposé sur l'internet public : n'importe qui peut
donc y envoyer un POST. **Seule la signature distingue un vrai paiement de
quelqu'un qui tape l'URL dans `curl`.** Tout le reste de ce document découle de
cette seule phrase.

Implémentation de référence publiée par Digistore24 :
<https://www.digistore24.com/download/ipn/examples/ipn/sha_sign.php>

## L'algorithme — six étapes

À partir des paramètres reçus en POST et de votre **passphrase IPN** :

1. **Retirez `sha_sign` et `SHASIGN`** du jeu de paramètres (en comparant les
   noms sans tenir compte de la casse). Ils transportent la signature
   elle-même et ne faisaient pas partie de ce qui a été signé.
2. **Triez les clés restantes comme des chaînes d'octets.** C'est le
   `ksort($params, SORT_STRING)` de PHP : une comparaison brute, octet par
   octet — *ni* sensible à la locale, *ni* insensible à la casse. En
   JavaScript, cela s'écrit `a < b ? -1 : a > b ? 1 : 0`, et non
   `a.localeCompare(b)`.
3. **Ignorez les valeurs vides.** `undefined`, `null` et `""` ne contribuent à
   rien, pas même par leur clé. Un champ arrivé vide se traite comme s'il
   n'était jamais arrivé.
4. **Concaténez**, pour chaque paramètre restant et dans l'ordre du tri :
   `KEY` + `=` + `VALUE` + `PASSPHRASE`. La passphrase suit *chaque* paire, et
   non une seule fois à la fin.
5. **Hachez le résultat en SHA512**, sur ses octets UTF-8, et présentez-le en
   **hexadécimal majuscule**.
6. **Comparez** avec le `sha_sign` reçu, **sans tenir compte de la casse** et
   en **temps constant**.

Exemple — les paramètres `{order_id: "ORD-1", product_id: "42"}` et la
passphrase `s3cret-passphrase`. Le tri place `order_id` avant `product_id` ; la
chaîne à hacher est donc :

```
order_id=ORD-1s3cret-passphraseproduct_id=42s3cret-passphrase
```

Aucun séparateur entre les paires : c'est la passphrase qui clôt chacune
d'elles.

## Le piège qui coûte une journée à tout le monde

**Digistore24 signe avec les noms de champ D'ORIGINE — `order_id=…`, et non
`ORDER_ID=…`.**

L'exemple PHP officiel contient un commutateur `convert_keys_to_uppercase`, et
à le lire de haut en bas, on croit que la mise en majuscules est la norme.
Vérifié sur des comptes Digistore24 réels, ce n'est pas le cas : les noms de
champ sont signés exactement tels qu'ils ont été envoyés. Une implémentation
qui passe les clés en majuscules sans condition calcule une signature
tout à fait valide, mais sur la mauvaise entrée, et rejette **chaque IPN
réelle** avec « signature invalide » — pendant que tous vos tests passent,
puisqu'ils signent de la même mauvaise façon qu'ils vérifient.

**Vérifiez donc selon les deux conventions.** Calculez la signature avec la
casse d'origine ; si elle ne correspond pas, recalculez-la avec les clés en
majuscules. Accepter l'une ou l'autre ne coûte rien en sécurité, puisque les
deux variantes exigent la passphrase secrète, et cela évite à l'opérateur
d'avoir à reproduire un réglage de son compte Digistore24 qu'il ne peut pas
voir.

## Refuser par défaut (fail closed), toujours

Ce ne sont pas des cas limites. C'est le profil d'une attaque :

| Situation | Bonne réponse |
|---|---|
| Aucun `sha_sign` dans le payload | **rejeter** |
| Aucune passphrase configurée de votre côté | **rejeter** |
| Signature présente, mais qui ne correspond pas | **rejeter** |
| Événement inconnu, signature valide | accepter la requête, ne rien changer |

La deuxième ligne est celle qu'on construit de travers. « Pas de passphrase
configurée ? On saute la vérification » : la ligne de code paraît raisonnable,
et elle fait de votre webhook de paiement un endpoint d'écriture ouvert au
public dès qu'une variable d'environnement se perd à un redéploiement. **Une
passphrase absente est un motif de rejet, pas un contournement.**

## Lire le corps

La signature porte sur les octets tels qu'ils ont été envoyés. Tout ce qui les
réécrit l'invalide :

- **Lisez le corps brut**, puis analysez-le vous-même. Un framework qui
  analyse, resérialise et vous tend un objet a pu réordonner ou réencoder
  quelque chose au passage.
- **Digistore24 envoie du `application/x-www-form-urlencoded`**, pas du JSON.
- **Pas de trim, pas de passage en minuscules, aucune normalisation des
  valeurs** avant de signer. Décodez les séquences pour-cent exactement une
  fois — comme le fait l'analyse d'un formulaire — et n'y touchez plus.
- **UTF-8.** Les noms d'acheteurs contiennent des trémas, des accents et des
  alphabets non latins. Hachez les octets UTF-8 : un langage qui retombe par
  défaut sur Latin-1 (anciennes installations Python, certaines configurations
  PHP) produira sans un mot un hachage différent pour `Jörg Müller` de celui
  que Digistore24 a signé pour le même nom.

## Répondre

- **Répondez `200` avec un corps court** une fois l'événement traité.
  Digistore24 **réessaie jusqu'à obtenir un 200** : une exception non gérée
  devient une boucle de renvois sans fin.
- **Répondez au test de connexion.** Digistore24 valide l'endpoint au moment
  où vous l'enregistrez, et le fait par un `GET`. Renvoyez `200 OK` à un GET,
  ainsi qu'à un POST dont l'événement est `connection_test`.
- **Ne redirigez jamais.** Un `301`/`302` renvoyé par votre endpoint IPN fait
  échouer la validation : Digistore24 veut l'endpoint lui-même, pas une étape
  intermédiaire.
- **L'URL doit être publique et en `https`.** Digistore24 refuse `http`, et
  refuse `localhost` tout net. Sur une plateforme de création par IA hébergée,
  votre URL de prévisualisation est déjà publique et en https — c'est bien la seule
  chose qui y est *plus facile* que sur un ordinateur portable.

## La livraison n'est ni ordonnée ni bornée

Deux propriétés du transport auxquelles votre handler doit résister, parce
qu'aucune des deux ne se manifeste pendant les tests :

- **Les événements arrivent dans le désordre.** Un `on_payment` renvoyé peut
  atterrir *après* le `on_refund` qui a mis fin à l'accès. La question « cette
  personne peut-elle utiliser le produit ? » se tranche donc d'après
  l'**état**, jamais d'après un horodatage ni l'ordre d'arrivée. Une fois
  l'accès terminé, aucun événement ultérieur ne peut le rouvrir.
- **Le même événement arrive plusieurs fois.** Digistore24 réessaie jusqu'à
  obtenir un 200, et un timeout de votre côté, même une fois le travail fait,
  compte comme un échec. **Chaque écriture de votre handler doit être
  idempotente**, sous une clé tirée du payload : `order_id` plus le nom de
  l'événement. `order_id` est l'identifiant que Digistore24 garantit,
  documenté comme *« ID unique de la commande. Plusieurs transactions d'une
  même commande portent le même order-ID »* : le paiement, son remboursement,
  une rétrofacturation (chargeback) et chaque renouvellement automatique d'un
  même abonnement arrivent tous avec cette même valeur. C'est ce qui en fait à
  la fois la clé d'idempotence et la clé sous laquelle l'accès lui-même est
  stocké — un remboursement ne peut révoquer ce qu'un paiement a accordé que si
  les deux s'entendent sur l'identifiant. Créditer un solde de tokens sans une
  telle clé distribue les crédits deux fois.

  ⚠️ **Une IPN ne transporte aucun `purchase_id`.** Ce nom ne figure dans
  aucune table publiée de paramètres IPN, et le message réel de
  `../scripts/vectors.json` (`captured-on-payment`, 173 paramètres) ne le
  contient pas. Il appartient à l'**API** Digistore24, où `getPurchase`
  documente son `purchase_id` comme « l'id de la commande Digistore24 » — la
  même valeur sous un autre nom. Indexez vos écritures dessus, et vous les
  indexez sur `undefined` dans chaque message qui arrivera jamais : soit toutes
  les commandes s'écrasent sur une seule clé, soit rien ne correspond jamais et
  la nouvelle tentative refait le travail une seconde fois. Dans les deux cas,
  l'endpoint a l'air de fonctionner — jusqu'à ce que de l'argent réel circule.

## Quoi stocker

Stockez le **payload brut** de chaque IPN acceptée, tel quel, avant d'agir
dessus. Cela ne coûte presque rien, et c'est le seul moyen de répondre, des
semaines plus tard, à la question « Digistore24 a-t-il vraiment envoyé
cela ? » — quand un client conteste son accès et que vos tables dérivées se
donnent toutes raison entre elles.

Stockez le **nom de l'événement** tel qu'il est arrivé. Ne réduisez pas les
événements à un statut pour décider ensuite d'après ce statut : voyez
`events.fr.md`, où deux événements qui disent l'inverse l'un de l'autre au
sujet de l'accès aboutissent au même mot.
