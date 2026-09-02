---
name: ds24-tokens
language: fr
description: À utiliser quand l'app mesure l'usage au lieu de restreindre des fonctionnalités — des crédits ou tokens prépayés achetés via Digistore24, consommés action par action, et rechargés automatiquement en débitant un moyen de paiement enregistré avec createBillingOnDemand. À utiliser dès que l'utilisateur parle de crédits, de tokens, de paiement à l'usage, d'un solde, de « facturer à la requête », de recharge automatique, ou d'une fonctionnalité d'IA qui doit coûter quelque chose au client.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Crédits prépayés

Pour certains produits, la question n'est pas « avez-vous le droit d'utiliser
ceci ? » mais « combien en avez-vous consommé ? ». C'est un **solde** — un
mécanisme distinct de l'accès, et non une variante de celui-ci.

## Étape 0 — est-ce seulement le bon modèle ?

Posez la question une seule fois, en une phrase : le client achète-t-il
**l'accès à une fonctionnalité** (un plan) ou **une quantité d'usage** (des
crédits) ? Le plan est le plus simple des deux, et la plupart des produits sont
des plans. Les crédits ne valent leur complexité que si vos propres coûts
croissent avec l'usage — le cas typique est une fonctionnalité d'IA.

Les deux modèles peuvent coexister ; l'un ne remplace pas l'autre.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez-le au `VERSION` de ce pack. En cas d'écart, signalez-le en une
phrase, puis continuez.

## Étape 1 — un solde n'est pas un droit d'accès

`hasAccess(member, creditPackage)` répond **false** — toujours, et à juste
titre. Un plan est un droit ; un solde est une quantité. Tenez-les dans deux
tables distinctes, et n'essayez pas d'exprimer l'un au moyen de l'autre.

```
token_accounts   member_id, balance
token_ledger     member_id, delta, reason, note, created_at, order_id
```

**Le grand livre fait foi ; le solde n'est qu'un cache.** Chaque mouvement est
une ligne. Un solde qu'on ne peut obtenir qu'en additionnant des lignes se
défend le jour où un client le conteste ; un solde qu'on a écrasé, non.

**`reason` et `note` sont des étiquettes, pas du contenu.** Le grand livre fait
partie de ce qu'une demande d'accès de la personne concernée restitue ; une
note dit donc *quel type d'action a été facturé* — « génération de rapport » —
et jamais ce que le client a saisi. Son prompt, son brouillon ou sa question,
consignés dans une ligne du grand livre, en font un second dépôt de données
personnelles, non maîtrisé, dans la seule table dont vous ne pourrez jamais
supprimer une ligne. Voir **`ds24-compliance`**.

## Étape 2 — acheter des crédits

Un pack de crédits est un produit Digistore24 comme les autres (voir
**`ds24-products`**). Ce qui change, c'est le traitement de l'IPN : pour un pack
de crédits, `on_payment` **crédite le solde** au lieu de créer un octroi d'accès.

Trois points à ne pas rater :

- **L'idempotence, avec la commande comme clé.** Digistore24 réessaie jusqu'à
  recevoir un 200 — y compris après un timeout survenu juste après une écriture
  réussie. Un crédit sans clé finit crédité deux fois.
- **Enregistrez le nombre de crédits au moment de l'achat.** N'allez pas le
  rechercher plus tard dans votre liste de prix : la liste change, et le client
  a acheté ce qui était proposé ce jour-là.
- **Un remboursement reprend les crédits.** Décidez en amont ce qui se passe si
  le solde a déjà été consommé : passer en négatif est honnête, refuser est
  défendable, ignorer le cas en silence n'est ni l'un ni l'autre. Consignez
  votre choix par écrit.

## Étape 3 — consommer : vérifier, travailler, facturer — dans cet ordre

```
1. CHECK    le solde est-il suffisant ?     -> si non, refusez avant de faire quoi que ce soit
2. WORK     faites le travail coûteux
3. CHARGE   déduisez, écrivez une ligne dans le grand livre
```

Facturer d'abord, c'est faire payer un travail qui échoue ensuite. Travailler
sans vérification préalable, c'est offrir le résultat : quand la déduction
échoue, la partie coûteuse a déjà tourné. **C'est celle-là, l'erreur qui se
commet vraiment.**

Cinq règles vont avec :

- **La fonction de facturation ne doit pas accepter d'id de membre.** Le compte
  facturé est toujours celui de l'appelant, tel que la session le connaît. Un
  id lu dans le corps de la requête est un moyen de vider le solde de
  quelqu'un d'autre — et un paramètre optionnel qui retombe sur la session par
  défaut ne ferme pas cette brèche, il permet seulement au mauvais appel de
  compiler à nouveau. Facturer pour le compte d'un tiers est une *autre*
  fonction, avec un contrôle d'opérateur en tête.
- **Le prix est le vôtre, calculé côté serveur.** Si vous lisez le montant dans
  la requête, c'est le client qui le fixera à zéro.
- **Tenez un verrou de ligne (ou une mise à jour conditionnelle atomique)
  pendant la déduction**, pour que deux requêtes concurrentes ne puissent pas
  faire passer un solde sous zéro.
- **Ce n'est pas idempotent.** Deux envois facturent deux fois — il n'existe
  aucune clé pour dédupliquer. Désactivez le bouton tant que la requête est en
  cours, et ne l'entourez jamais d'une relance à l'aveugle.
- **Rien dans la configuration de l'app elle-même ne doit pouvoir refuser une
  consommation.** Un réglage qui dit quel modèle cette app vend — un drapeau
  « credits enabled », un mode de tarification, un interrupteur de
  fonctionnalité — a sa place devant l'*achat* et dans l'interface, jamais
  devant la facturation. Si vous cessez de vendre des crédits, chaque client
  qui détient encore un solde payé garde le droit de le consommer ; une
  consommation conditionnée à cet interrupteur bloque de l'argent que vous avez
  déjà encaissé.

## Étape 4 — recharge automatique

Digistore24 peut débiter un moyen de paiement enregistré sans que le client
soit là : **`createBillingOnDemand`**, sur l'achat d'origine.

Cela ne fonctionne que si l'achat a enregistré les données de paiement — pour
un paiement unique, cela veut dire envoyer `settings[force_rebilling]=Y` dans
l'URL d'achat (voir **`ds24-checkout`**). La décision se prend au moment du
checkout ; on ne peut pas l'ajouter après coup.

Cinq limites :

- **Le client doit avoir consenti**, en toutes lettres, à être débité de
  nouveau — avant le premier débit automatique. C'est une autorisation de
  paiement, pas un réglage.
- **Un seul débit en cours à la fois.** Posez une marque sur le compte tant
  qu'une recharge est en attente, et retirez-la quand l'IPN confirme ; sinon,
  une réponse lente devient deux débits.
- **Comptez les débits que l'IPN n'a jamais confirmés, et arrêtez-vous à
  deux.** C'est celle-ci qui vous piège, et elle vous piège *à cause* de la
  précédente.

  La marque doit expirer : un processus qui meurt en la tenant gèlerait le
  compte pour toujours. Mais pensez maintenant à l'IPN qui n'arrive jamais. La
  carte a été débitée, le solde n'a jamais été crédité, il reste donc sous le
  seuil — et dès que la marque expire, la consommation suivante débite la carte
  **une nouvelle fois**. Puis encore une. Aussi longtemps que le solde du client
  reste bas, c'est-à-dire pour toujours, puisque le crédit qui le remonterait
  est justement celui qui n'est jamais arrivé.

  Digistore24 autorise dix débits par jour et par achat ; son plafond ne vous
  sauvera donc pas : avec une expiration de six heures, on est à quatre débits
  par jour, confortablement en dessous.

  **Rien de tout cela ne ressemble à une panne.** Chaque débit RÉUSSIT. Aucune
  erreur n'est levée, aucune requête n'échoue, et le réglage de recharge
  automatique du client affiche toujours « activé ». La seule anomalie, c'est
  un crédit qui n'est pas arrivé — et rien ne le surveille, sauf si vous
  construisez cette surveillance.

  Tenez donc un compteur à côté de la marque — le nombre de débits depuis le
  dernier qui est revenu sous forme de crédit comptabilisé —, incrémentez-le
  dans la même écriture atomique que celle qui pose la marque, et refusez tout
  débit dès qu'il atteint deux. Remettez-le à zéro quand un crédit est
  effectivement comptabilisé. Deux et non un, parce qu'un unique débit non
  confirmé est l'état normal de toute recharge saine tant que l'IPN est en
  chemin, et que Digistore24 a le droit d'être plus lent que votre expiration.

  **Mettez en pause ; ne désactivez pas le réglage du client.** Rien n'a changé
  dans ce qu'il a demandé — seule votre confiance que le débit lui parvient a
  changé. La recharge reprend d'elle-même dès qu'un crédit est comptabilisé. Et
  c'est **vous** qu'il faut prévenir, pas lui : son réglage est toujours activé
  et toujours juste, alors consignez la pause là où la personne qui assure le
  support de cette app la verra.
- **Une recharge qui échoue n'est pas une erreur à cacher.** Dites au client
  que son solde est épuisé et que la recharge n'est pas passée.
- **Ne rechargez jamais pour le compte d'un tiers.** Si votre app possède, pour
  le support, un mode « agir en tant que ce client » sous quelque forme que ce
  soit, coupez le débit automatique à l'intérieur de ce mode : personne n'est
  là pour consentir à un paiement.

## Étape 5 — prouvez-le

1. Achetez un pack de crédits par un achat de test (le cookie, ou en
   développement le paramètre testpay — **`ds24-checkout`**, Étape 4a) → solde
   crédité **une fois**.
2. Renvoyez la même IPN à la main → solde **inchangé**. (Le vérificateur de
   **`ds24-ipn`** rejoue un événement précisément pour cette raison, mais il ne
   voit pas votre solde : celui-ci, vérifiez-le à la main.)
3. Consommez jusqu'à ce que le solde ne suffise plus → l'action est refusée
   **avant** le travail coûteux, et aucune ligne n'est écrite dans le grand
   livre.
4. Remboursez le pack → les crédits sont repris, de la manière que vous avez
   décidée à l'Étape 2.
5. **Déclenchez une recharge automatique, puis faites disparaître l'IPN** — ne
   la livrez pas. Laissez passer votre délai d'expiration, consommez de nouveau,
   laissez-le passer encore, consommez encore. La carte doit être débitée
   **deux fois, et plus jamais ensuite**, et quelque chose que vous pouvez lire
   doit dire sur quel compte c'est arrivé. Sauter ce test, c'est exactement
   ainsi que la boucle de l'Étape 4 part en production : tous les autres tests
   passent avec elle, parce que chaque débit, pris isolément, fonctionne.

## Étape 6 — la suite

- **`ds24-entitlements`** — si certaines choses sont restreintes plutôt que
  mesurées.
- **`ds24-compliance`** — le grand livre contient des notes sur des personnes ;
  cela a des conséquences.
- **`ds24-golive`** — le vrai achat de test.

Dites laquelle vous commencez, et commencez-la.
