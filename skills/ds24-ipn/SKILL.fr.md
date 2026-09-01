---
name: ds24-ipn
language: fr
description: À utiliser pour construire ou corriger le webhook IPN de Digistore24 — l'endpoint qui reçoit les événements de paiement et les transforme en accès. Couvre la vérification de la signature SHA512, la correspondance entre événement et accès, l'idempotence et un script de vérification qui prouve que l'endpoint est correct. À utiliser dès que l'utilisateur mentionne l'IPN de Digistore24, un webhook de paiement, « signature invalide », des achats qui ne débloquent rien, des remboursements qui ne révoquent pas l'accès ou un abonnement annulé qui a perdu l'accès trop tôt.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# L'endpoint IPN de Digistore24

C'est la pièce qui décide qui a payé et ce qu'il a le droit d'utiliser. Tout le
reste d'une intégration Digistore24 peut être refait plus tard ; celle-ci doit
être juste du premier coup, car se tromper ici laisse soit des inconnus
déverrouiller votre produit, soit retire l'accès à des gens qui ont payé.

Vous la construisez dans la stack que l'utilisateur possède déjà. Cette skill ne
présuppose ni Next.js, ni Supabase, ni Python, ni quoi que ce soit d'autre —
elle vous donne le protocole, un adaptateur pour votre runtime et un script qui
prouve le résultat.

## Étape 0 — est-ce déjà là ?

Regardez avant de demander. Cherchez dans le projet `sha_sign`, `on_payment`,
`DIGISTORE_IPN_PASSPHRASE` ou une route contenant `ipn`.

- **Rien trouvé** → continuez avec l'Étape 1.
- **Quelque chose trouvé** → ne le reconstruisez pas. Allez à l'Étape 5 et
  lancez le vérificateur dessus. Corrigez ce qu'il signale, et rien d'autre.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Les skills que vous avez importées dans un workspace ne se mettent pas à jour
toutes seules. Récupérez
`https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION` et
comparez-le au fichier `VERSION` situé à côté de ce pack. Si le vôtre est plus
ancien, dites-le en une phrase et proposez de réimporter — puis continuez dans
tous les cas. Une copie périmée reste généralement correcte ; une copie périmée
passée sous silence, c'est ainsi qu'un bug corrigé revient.

## Étape 1 — lisez le protocole

Lisez **`references/ipn-protocol.fr.md`** maintenant, en entier. Cela fait deux
pages et cela contient le seul détail qui coûte une journée à tout le monde :
Digistore24 signe avec la casse **d'origine** des noms de champs (`order_id=…`),
pas en majuscules, même si son propre exemple PHP suggère le contraire. Une
implémentation qui se trompe là-dessus passe tous ses propres tests et refuse
chaque paiement réel.

Lisez ensuite **`references/events.fr.md`**. Il contient le tableau qui associe
les événements à l'accès, y compris les deux lignes contre-intuitives et qui
coûtent de l'argent quand on les devine :

- `on_rebill_cancelled` ne fait **rien** à l'accès.
- `on_payment_missed` **suspend de façon réversible** — c'est une carte expirée,
  pas un départ.

Ne les survolez pas et n'écrivez pas de mémoire. Chaque phrase qui s'y trouve y
est parce que quelqu'un s'est trompé en production.

La troisième référence, **`references/verification.fr.md`**, est pour l'Étape 5
— ce qu'il faut prouver une fois que l'endpoint existe. Lisez-la quand vous y
serez, pas maintenant.

## Étape 2 — copiez le module de signature, n'en écrivez pas un

`adapters/` contient trois implémentations de la signature. **Copiez celle qui
correspond au runtime, telle quelle, et ne la modifiez jamais :**

| Runtime | Fichier |
|---|---|
| Node (runtime Node de Next.js, Express, Nest, Node nu) | `adapters/signature-node.mjs` |
| **Deno / Supabase Edge Functions / Lovable Cloud** / Cloudflare Workers / edge de Next.js | `adapters/signature-web.mjs` |
| Python (FastAPI, Django, Flask, nu) | `adapters/signature.py` |

Les trois sont vérifiées contre des vecteurs de test figés partagés avec la
Digistore SAAS App Template, elles concordent donc de façon prouvée entre elles
et avec un compte Digistore24 réel. En réécrire une « pour coller au style du
code » jette cette garantie pour rien.

Ce sont du JavaScript simple avec des types JSDoc (ou du Python simple), donc un
projet TypeScript les importe et conserve un typage complet.

## Étape 3 — construisez l'endpoint à partir de l'adaptateur correspondant

Les fichiers d'endpoint qui les accompagnent sont des **exemples que vous
adaptez**, pas des fichiers à copier aveuglément :

| Stack | Fichier |
|---|---|
| Next.js App Router | `adapters/next-node.ts` |
| **Supabase Edge Function / Lovable Cloud** | `adapters/deno-edge.ts` |
| Express | `adapters/express-node.js` |
| FastAPI | `adapters/python-fastapi.py` |

Quelle que soit la stack, ces cinq propriétés ne sont pas négociables :

1. **Lisez le corps brut et parsez-le vous-même.** Digistore24 poste de
   l'`application/x-www-form-urlencoded`. Un framework qui parse puis
   re-sérialise peut casser la signature.
2. **Échouez en mode fermé.** Pas de signature → refus. Pas de passphrase
   configurée → refus. « Sauter la vérification quand la passphrase manque »
   transforme l'endpoint en endpoint d'écriture public dès la première fois
   qu'une variable d'environnement disparaît lors d'un redéploiement.
3. **Répondez `200` à un GET**, et à `connection_test`. Digistore24 valide
   l'endpoint de cette manière et refuse d'en enregistrer un qui redirige.
4. **Ne laissez jamais une exception sortir du handler.** Digistore24 réessaie
   jusqu'à obtenir un 200, une exception devient donc une boucle de relivraison
   sans fin. Journalisez-la, répondez 200, rejouez depuis le payload brut
   stocké.
5. **Stockez le payload brut avant d'agir dessus.** C'est le seul enregistrement
   qui survit à un bug dans tout ce qui vient ensuite.

**Sur Lovable Cloud / Supabase il y en a une sixième**, et la sauter est
silencieux : la fonction doit être déployée avec **`verify_jwt = false`**.
Digistore24 n'envoie aucun JWT Supabase, donc avec la valeur par défaut activée,
chaque IPN reçoit un 401 avant que votre code ne s'exécute et aucun achat ne
déverrouille quoi que ce soit — sans aucune erreur visible où que ce soit dans
l'app. Mettez-le dans `supabase/config.toml` :

```toml
[functions.ds24-ipn]
verify_jwt = false
```

## Étape 4 — les cinq invariants qui ne sont pas dans le switch

Notez-les dans les notes de l'app elle-même, car ils sont invisibles en revue :

- **Chaque écriture est idempotente**, avec pour clé `(order_id, event)` — avec
  une contrainte UNIQUE, pas un `SELECT` suivi d'un `INSERT`, que deux
  relivraisons concurrentes traversent sans s'arrêter. Digistore24 réessaie
  après un timeout même quand le travail a réussi.
- **Terminé est définitif.** Une fois l'accès terminé (remboursement,
  rétrofacturation, dernier jour payé), aucun événement ultérieur ne peut le
  rouvrir. La livraison n'est pas ordonnée, un `on_payment` relivré peut donc
  arriver *après* le remboursement. Protégez-vous sur l'état stocké, avant de
  regarder le nom de l'événement.
- **Un produit que vous ne connaissez pas n'accorde rien.** La connexion IPN est
  enregistrée avec une liste `product_ids`, et `all` — tout le compte du vendeur
  — est un réglage normal (voir **`ds24-products`**). Des événements d'un ancien
  tunnel de vente, d'une deuxième app ou du lancement de quelqu'un d'autre
  peuvent donc légitimement atterrir sur votre endpoint. Stockez le payload,
  répondez `200`, n'accordez rien. N'associez jamais un id de produit inconnu à
  un plan par défaut : cela distribue de l'accès pour un achat qui n'a jamais
  été le vôtre, et c'est une erreur que personne ne remarque tant que la
  mauvaise personne n'est pas à l'intérieur.
- **Une même offre peut avoir PLUSIEURS ids de produit — associez-les tous.** Un
  produit Digistore24 porte exactement une langue, une boutique multilingue vend
  donc chaque offre via un produit par langue (**`ds24-products`**), et le
  payload nomme celui que l'acheteur a réellement utilisé. N'associez que l'id
  allemand et chaque achat anglais tombe dans la règle juste au-dessus : un
  client qui paie, correctement enregistré, à qui rien n'est accordé. Cherchez
  le `product_id` du payload parmi **tous** les ids de toutes les offres, et
  résolvez-les vers la même clé de produit.
- **À qui appartient ce paiement suit un ORDRE, et cela se décide ici.**
  L'identifiant que votre checkout a mis en premier dans `tracking[custom]` —
  celui-là est authentifié. L'e-mail de l'acheteur seulement après lui, comme
  supposition **non authentifiée** : Digistore24 ne vérifie pas l'adresse que
  l'acheteur a tapée. Et une adresse qui correspond à **plus d'un compte est
  refusée**, jamais résolue vers la première ligne. L'attribution accorde et ne
  révoque jamais, ce qui est la seule raison pour laquelle la voie de l'e-mail
  est tolérable tout court. L'ordre complet, ses refus et ce qui ne peut pas
  être autorisé par une correspondance d'e-mail sont l'Étape 2 de
  **`ds24-checkout`** — lisez-la avant d'écrire cette partie, car chaque échec
  ici ressemble à un endpoint qui fonctionne.

## Étape 5 — prouvez-le

Ne dites pas à l'utilisateur que ça marche. **`references/verification.fr.md`
dit ce qu'il faut prouver** — lisez-le, puis construisez la vérification avec ce
qui s'exécute sur cette plateforme.

Deux choses décident du comment :

**Y a-t-il un shell ?** Replit, v0, Manus, Claude Code, Codex — oui. Alors
lancez le script livré avec cette skill ; il lui faut Node et une connexion
réseau, rien d'autre :

```bash
node scripts/verify-ipn.mjs \
  --url https://<l'app>/api/ipn \
  --passphrase "$DIGISTORE_IPN_PASSPHRASE" \
  --probe https://<l'app>/api/ds24-selftest --probe-token "$SECRET"
```

**Lovable n'en a pas.** Les skills y transportent leurs fichiers, mais la
plateforme les lit au lieu de les exécuter — sur Lovable, ce script est donc de
la documentation, pas un outil. Écrivez l'équivalent sous forme de **test à
l'intérieur de l'app** (un test Deno sur Lovable Cloud), ce qui est la forme B
dans `verification.fr.md`. Cela sort plus simple : un test avec accès à la base
de données lit l'enregistrement d'accès directement et n'a besoin d'aucun
endpoint de sonde.

**Une règle vaut dans les deux cas, et c'est tout l'enjeu :**

> Votre signature doit reproduire exactement chaque vecteur de
> `scripts/vectors.json`. **Ne calculez jamais les valeurs attendues avec votre
> propre code.** Le bug que cela attrape — signer avec des noms de champs en
> majuscules — produit une implémentation qui s'accorde parfaitement avec
> elle-même et refuse chaque paiement réel. Une vérification écrite depuis le
> même malentendu confirme le bug.

Si vous construisez la vérification vous-même, cette comparaison est la première
chose qu'elle fait.

**Rapportez ce qu'a dit l'exécution**, y compris ce qu'elle n'a pas couvert. Une
exécution qui a sauté la moitié « accès » est une signature prouvée et une
sémantique non prouvée — dites-le plutôt que de la déclarer verte.

### Quand aucune IPN n'arrive du tout, ce script ne peut pas aider

Il prouve ce que votre endpoint fait d'un payload. Un paiement qui ne l'a jamais
atteint ne produit rien à vérifier, et la question porte alors sur la
*connexion*, pas sur le code. Demandez à Digistore24 ce qu'il détient pour cette
commande :

```
POST https://www.digistore24.com/api/call/getPurchase/format/json
Header: X-DS-API-KEY: <la clé>
Body:   purchase_id=ABC12345
```

Inconnu là-bas → il n'y a pas eu d'achat (ou il était dans un autre compte
vendeur). Connu là-bas et absent de votre app → l'IPN n'est jamais arrivée : une
URL enregistrée qui ne répond plus, un `domain_id` qu'un autre projet a écrasé,
ou une liste `product_ids` dans laquelle ce produit ne figure pas. Les trois
sont **`ds24-products`**, Étape 4 — et les trois échouent sans message d'erreur
nulle part.

Pour vérifier les modules de signature livrés à part, là où un shell existe :

```bash
node scripts/check-adapters.mjs      # les trois runtimes contre les vecteurs
```

## Étape 6 — la suite

L'endpoint reçoit les événements. Trois choses doivent encore exister autour de
lui :

- **`ds24-products`** — récupérez la clé d'API, créez les produits chez
  Digistore24 et enregistrez cet endpoint comme connexion IPN. Sans cela,
  personne ne vous appelle jamais. **Commencez ici.**
- **`ds24-entitlements`** — l'enregistrement d'accès dans lequel les événements
  écrivent, et l'unique fonction que le reste de l'app interroge.
- **`ds24-checkout`** — le lien d'achat qui démarre un achat.

Dites laquelle vous commencez, et commencez-la.
