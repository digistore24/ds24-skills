---
name: ds24-tokens
language: fr
description: À utiliser quand l'app mesure la consommation au lieu de restreindre des fonctions — crédits prépayés ou tokens achetés via Digistore24, consommés par action, et rechargés automatiquement en débitant un moyen de paiement enregistré avec createBillingOnDemand. À utiliser dès que l'utilisateur mentionne des crédits, des tokens, le paiement à l'usage, un solde, « facturer par requête », la recharge automatique ou une fonction d'IA qui doit coûter quelque chose au client.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Crédits prépayés

Certains produits ne posent pas la question « avez-vous le droit d'utiliser
ceci » mais « combien en avez-vous utilisé ». C'est un **solde**, et c'est un
mécanisme différent de l'accès — pas une variante de celui-ci.

## Étape 0 — est-ce seulement le bon modèle ?

Demandez une fois, en une phrase : le client achète-t-il **l'accès à une
fonction** (un plan) ou **une quantité de consommation** (des crédits) ? Les
plans sont plus simples et la plupart des produits sont des plans. Les crédits
méritent leur complexité quand vos propres coûts augmentent avec l'usage — une
fonction d'IA est le cas habituel.

Les deux peuvent coexister. Ils ne se remplacent pas l'un l'autre.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez avec le `VERSION` de ce pack. Signalez tout écart en une phrase, puis
continuez.

## Étape 1 — un solde n'est pas un droit d'accès

`hasAccess(member, creditPackage)` répond **false**, pour toujours, et à raison.
Un plan est un droit ; un solde est une quantité. Gardez-les dans des tables
séparées et n'essayez pas d'exprimer l'un comme l'autre.

```
token_accounts   member_id, balance
token_ledger     member_id, delta, reason, note, created_at, order_id
```

**Le grand livre est la vérité, le solde est le cache.** Chaque changement est
une ligne. Un solde auquel vous ne pouvez arriver qu'en additionnant des lignes
est un solde que vous pouvez défendre quand un client le conteste ; un solde que
vous avez écrasé, non.

**`reason` et `note` sont des étiquettes, pas du contenu.** Le grand livre fait
partie de ce que restitue une demande d'accès de la personne concernée, donc une
note dit *quel type d'action a été facturé* — « génération de rapport » — et
jamais ce que le client y a tapé. Son prompt, son brouillon, sa question dans une
ligne du grand livre en fait un second stock de données personnelles non géré,
dans la seule table dont vous ne pourrez jamais supprimer les lignes. Voir
**`ds24-compliance`**.

## Étape 2 — acheter des crédits

Un pack de crédits est un produit Digistore24 comme un autre (voir
**`ds24-products`**). Ce qui diffère, c'est le traitement de l'IPN : `on_payment`
pour un pack de crédits **crédite le solde** au lieu de créer un octroi d'accès.

Trois choses à faire correctement :

- **L'idempotence, avec une clé sur la commande.** Digistore24 réessaie jusqu'à
  obtenir un 200, y compris après un timeout qui a suivi une écriture réussie. Un
  crédit sans clé est crédité deux fois.
- **Enregistrez le nombre de crédits au moment de l'achat.** N'allez pas chercher
  le montant plus tard dans votre liste de prix — la liste change, et le client a
  acheté ce qui était proposé ce jour-là.
- **Un remboursement retire les crédits.** Décidez d'avance ce qui se passe quand
  le solde a déjà été consommé : passer en négatif est honnête, refuser est
  défendable, l'ignorer en silence n'est ni l'un ni l'autre. Écrivez le choix
  quelque part.

## Étape 3 — consommer : vérifier, travailler, facturer — dans cet ordre

```
1. CHECK    le solde est-il suffisant ?     -> si non, refusez avant toute chose
2. WORK     faites la chose coûteuse
3. CHARGE   déduisez, écrivez une ligne de grand livre
```

Facturer d'abord facture un travail qui échoue ensuite. Faire le travail sans
vérification devant donne le résultat gratuitement, parce qu'au moment où la
déduction échoue la partie coûteuse a déjà tourné. **C'est l'erreur qui est
réellement commise.**

Cinq règles autour :

- **La fonction de facturation ne doit pas prendre d'id de membre.** Le compte
  débité est toujours celui de l'appelant, pris dans la session. Un id lu dans le
  corps d'une requête est un moyen de vider le solde de quelqu'un d'autre — et un
  paramètre optionnel qui bascule par défaut sur la session ne referme rien, il
  fait seulement recompiler le mauvais appel. Facturer pour le compte de
  quelqu'un d'autre est une *autre* fonction, avec un contrôle d'opérateur en
  tête.
- **Le prix est le vôtre, calculé côté serveur.** Lisez le montant dans la
  requête et le client le met à zéro.
- **Tenez un verrou de ligne (ou une mise à jour conditionnelle atomique)
  pendant la déduction**, pour que deux requêtes concurrentes ne puissent pas
  faire passer un solde sous zéro.
- **Ce n'est pas idempotent.** Deux envois facturent deux fois — il n'y a aucune
  clé sur laquelle dédupliquer. Désactivez le bouton pendant que la requête est
  en vol, et ne construisez jamais de nouvelle tentative aveugle autour.
- **Rien dans la configuration de l'app elle-même ne peut refuser une
  consommation.** Un réglage qui dit quel modèle cette app vend — un drapeau
  « credits enabled », un mode de tarification, un interrupteur de fonction — a
  sa place devant l'*achat* et dans l'interface, jamais devant la facturation.
  Arrêtez de vendre des crédits et tout client qui détient encore un solde payé a
  le droit de le consommer ; une consommation restreinte par cet interrupteur
  immobilise de l'argent que vous avez déjà encaissé.

## Étape 4 — recharge automatique

Digistore24 peut débiter un moyen de paiement enregistré sans le client présent :
**`createBillingOnDemand`** contre l'achat d'origine.

Cela ne fonctionne que si l'achat a enregistré les données de paiement, ce qui
pour un paiement unique signifie envoyer `settings[force_rebilling]=Y` sur l'URL
d'achat (voir **`ds24-checkout`**). Décidez-le au moment du checkout — cela ne
peut pas être ajouté après coup.

Cinq limites :

- **Le client doit avoir accepté** d'être débité à nouveau, avec des mots, avant
  le premier débit automatique. C'est une autorisation de paiement, pas un
  réglage.
- **Un seul débit en vol à la fois.** Marquez le compte pendant qu'une recharge
  est en attente et effacez la marque quand l'IPN confirme, sinon une réponse
  lente devient deux débits.
- **Comptez les débits que l'IPN n'a jamais confirmés, et arrêtez après deux.**
  C'est celui qui mord, et il mord *à cause* de la limite précédente.

  La marque doit expirer — un processus qui meurt en la tenant gèlerait le compte
  pour toujours. Mais pensez maintenant à l'IPN qui n'arrive jamais : la carte a
  été débitée, le solde n'a jamais été crédité, donc le solde est toujours sous
  le seuil, et à l'instant où la marque expire la consommation suivante débite la
  carte **encore une fois**. Et encore. Aussi longtemps que le solde du client
  reste bas, c'est-à-dire pour toujours, parce que le crédit qui le remonterait
  est précisément ce qui n'est jamais arrivé.

  Digistore24 autorise dix débits par jour et par achat, donc sa limite ne vous
  sauve pas : une expiration de six heures donne quatre débits par jour et reste
  confortablement en dessous.

  **Rien là-dedans ne ressemble à une panne.** Chaque débit RÉUSSIT. Aucune
  erreur n'est levée, aucune requête n'échoue, et le réglage de recharge
  automatique du client indique toujours « activé ». La seule anomalie est un
  crédit qui n'est pas arrivé, et rien ne surveille cela à moins que vous ne le
  construisiez.

  Gardez donc un compteur à côté de la marque — les débits depuis le dernier
  revenu sous forme de crédit comptabilisé — incrémentez-le dans la même écriture
  atomique qui prend la marque, et refusez de débiter dès qu'il atteint deux.
  Remettez-le à zéro quand un crédit se comptabilise réellement. Deux plutôt
  qu'un, parce qu'un seul débit non confirmé est l'état normal de toute recharge
  saine tant que l'IPN est en vol, et Digistore24 a le droit d'être plus lent que
  votre expiration.

  **Mettez en pause, ne désactivez pas son réglage.** Rien de ce que le client a
  demandé n'a changé — seulement votre confiance dans le fait que le débit lui
  parvient. Cela reprend alors tout seul dès qu'un crédit se comptabilise. Et
  dites-le à **vous-même**, pas à lui : son réglage est toujours activé et
  toujours correct, alors mettez-le là où celui qui assure le support de cette
  app le verra.
- **Une recharge échouée n'est pas une erreur à cacher.** Dites au client que son
  solde est épuisé et que la recharge n'est pas passée.
- **Ne rechargez jamais pour le compte de quelqu'un d'autre.** Si votre app a un
  mode « agir en tant que ce client » pour le support, supprimez le débit
  automatique à l'intérieur — il n'y a personne de présent pour accepter un
  paiement.

## Étape 5 — prouvez-le

1. Achetez un pack de crédits avec un achat de test (cookie, ou en développement
   le paramètre testpay — **`ds24-checkout`**, Étape 4a) → solde crédité **une
   fois**.
2. Renvoyez la même IPN à la main → solde **inchangé**. (Le vérificateur de
   **`ds24-ipn`** rejoue un événement exactement pour cette raison, mais il ne
   peut pas voir votre solde — celui-ci, vérifiez-le à la main.)
3. Consommez jusqu'à être à court → l'action est refusée **avant** le travail
   coûteux, et aucune ligne de grand livre n'est écrite.
4. Remboursez le pack → les crédits ressortent, de la façon dont vous l'avez
   décidé à l'Étape 2.
5. **Déclenchez une recharge automatique puis jetez l'IPN** — ne la livrez pas.
   Attendez que votre expiration passe, consommez à nouveau, attendez encore,
   consommez à nouveau. La carte doit être débitée **deux fois et plus jamais
   ensuite**, et quelque chose que vous pouvez lire doit dire à quel compte c'est
   arrivé. Sauter celui-ci, c'est ainsi que la boucle de l'Étape 4 part en
   production : tous les autres tests passent avec elle en place, parce que
   chaque débit individuel fonctionne.

## Étape 6 — la suite

- **`ds24-entitlements`** — si certaines choses sont restreintes plutôt que
  mesurées.
- **`ds24-compliance`** — le grand livre contient des notes sur des personnes ;
  cela a des conséquences.
- **`ds24-golive`** — le vrai achat de test.

Dites laquelle vous commencez, et commencez-la.
