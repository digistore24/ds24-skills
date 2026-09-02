---
name: ds24-ipn
language: fr
description: À utiliser pour construire ou corriger le webhook IPN de Digistore24 — l'endpoint qui reçoit les événements de paiement et les transforme en accès. Couvre la vérification de la signature SHA512, la correspondance entre événement et accès, l'idempotence, et un script de vérification qui prouve que l'endpoint est correct. À utiliser dès que l'utilisateur mentionne l'IPN de Digistore24, un webhook de paiement, « signature invalide », des achats qui ne débloquent rien, des remboursements qui ne révoquent pas l'accès, ou un abonnement résilié qui a perdu l'accès trop tôt.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# L'endpoint IPN de Digistore24

C'est la pièce qui décide qui a payé et ce que cette personne a le droit
d'utiliser. Tout le
reste d'une intégration Digistore24 peut se refaire plus tard ; celle-ci doit
être juste du premier coup, car une erreur ici a deux issues : des inconnus
déverrouillent votre produit, ou des gens qui ont payé perdent leur accès.

Vous la construisez dans la stack que l'utilisateur a déjà, quelle qu'elle
soit. Cette skill ne présuppose ni Next.js, ni Supabase, ni Python, ni rien
d'autre : elle vous donne le protocole, un adaptateur pour votre runtime et un
script qui prouve le résultat.

## Étape 0 — est-ce déjà là ?

Regardez avant de demander. Cherchez dans le projet `sha_sign`, `on_payment`,
`DIGISTORE_IPN_PASSPHRASE` ou une route contenant `ipn`.

- **Rien trouvé** → passez à l'Étape 1.
- **Quelque chose trouvé** → ne le reconstruisez pas. Allez à l'Étape 5 et
  lancez le vérificateur dessus. Corrigez ce qu'il signale, et rien d'autre.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Une skill importée dans un workspace ne se met pas à jour toute seule.
Récupérez
`https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION` et
comparez-le au fichier `VERSION` qui accompagne ce pack. Si le vôtre est plus
ancien, dites-le en une phrase et proposez de réimporter — puis continuez, dans
un cas comme dans l'autre. Une copie périmée est en général encore correcte ;
une copie périmée passée sous silence, c'est ainsi qu'un bug déjà corrigé
revient.

## Étape 1 — lisez le protocole

Lisez **`references/ipn-protocol.fr.md`** maintenant, en entier. Deux pages, et
le seul détail qui coûte une journée à tout le monde s'y trouve : Digistore24
signe avec la casse **d'origine** des noms de champ (`order_id=…`), pas en
majuscules, même si son propre exemple PHP laisse croire le contraire. Une
implémentation qui se trompe là-dessus réussit tous ses propres tests et refuse
chaque paiement réel.

Lisez ensuite **`references/events.fr.md`**. On y trouve le tableau qui fait
correspondre les événements et l'accès, dont les deux lignes contre-intuitives,
celles qui coûtent de l'argent quand on les devine :

- `on_rebill_cancelled` ne change **rien** à l'accès.
- `on_payment_missed` **suspend de façon réversible** — c'est une carte
  expirée, pas un départ.

Ne les survolez pas pour écrire ensuite de mémoire. Chaque phrase y est parce
que quelqu'un s'est trompé en production.

La troisième référence, **`references/verification.fr.md`**, sert à l'Étape 5 :
ce qu'il faut prouver une fois que l'endpoint existe. Lisez-la à ce moment-là,
pas maintenant.

## Étape 2 — copiez le module de signature, n'en écrivez pas un

`adapters/` contient trois implémentations de la signature. **Copiez celle qui
correspond au runtime, telle quelle, et ne la modifiez jamais :**

| Runtime | Fichier |
|---|---|
| Node (runtime Node de Next.js, Express, Nest, Node seul) | `adapters/signature-node.mjs` |
| **Deno / Supabase Edge Functions / Lovable Cloud** / Cloudflare Workers / edge de Next.js | `adapters/signature-web.mjs` |
| Python (FastAPI, Django, Flask, sans framework) | `adapters/signature.py` |

Les trois sont vérifiées contre des vecteurs de test figés, partagés avec le
Digistore SAAS App Template : il est donc prouvé qu'elles concordent entre
elles et avec un compte Digistore24 réel. En réécrire une « pour respecter le
style du code » sacrifie cette garantie pour rien.

C'est du JavaScript pur, typé par JSDoc (ou du Python pur) ; un projet
TypeScript les importe en gardant un contrôle de types complet.

## Étape 3 — construisez l'endpoint à partir de l'adaptateur correspondant

Les fichiers d'endpoint qui les accompagnent sont des **exemples à adapter**,
pas des fichiers à copier aveuglément :

| Stack | Fichier |
|---|---|
| Next.js App Router | `adapters/next-node.ts` |
| **Supabase Edge Function / Lovable Cloud** | `adapters/deno-edge.ts` |
| Express | `adapters/express-node.js` |
| FastAPI | `adapters/python-fastapi.py` |

Quelle que soit la stack, ces cinq propriétés ne se négocient pas :

1. **Lisez le corps brut et parsez-le vous-même.** Digistore24 envoie du
   `application/x-www-form-urlencoded`. Un framework qui parse puis
   resérialise peut casser la signature.
2. **Refusez par défaut** (fail closed). Pas de signature → refus. Pas de
   passphrase configurée → refus. « On saute la vérification quand la
   passphrase manque » transforme l'endpoint en endpoint d'écriture public dès
   la première fois qu'une variable d'environnement se perd dans un
   redéploiement.
3. **Répondez `200` à un GET**, et à `connection_test`. C'est ainsi que
   Digistore24 valide l'endpoint, et il refuse d'en enregistrer un qui
   redirige.
4. **Ne laissez jamais une exception remonter hors du handler.** Digistore24
   réessaie jusqu'à obtenir un 200 : une exception devient donc une boucle de
   renvois sans fin. Journalisez-la, répondez 200, puis rejouez à partir du
   payload brut stocké.
5. **Stockez le payload brut avant d'agir dessus.** C'est la seule trace qui
   survit à un bug dans tout ce qui vient après.

**Sur Lovable Cloud / Supabase, il y en a une sixième**, et l'oublier ne fait
aucun bruit : la fonction doit être déployée avec **`verify_jwt = false`**.
Digistore24 n'envoie pas de JWT Supabase ; avec le réglage par défaut, chaque
IPN reçoit donc un 401 avant même que votre code s'exécute, et aucun achat ne
débloque quoi que ce soit — sans la moindre erreur visible dans l'app. Mettez
ceci dans `supabase/config.toml` :

```toml
[functions.ds24-ipn]
verify_jwt = false
```

## Étape 4 — les cinq invariants qui n'apparaissent pas dans le switch

Notez-les dans les notes de l'app elle-même, car ils sont invisibles à la
relecture :

- **Chaque écriture est idempotente**, avec `(order_id, event)` pour clé — par
  une contrainte UNIQUE, pas par un `SELECT` suivi d'un `INSERT`, que deux
  renvois simultanés traversent sans encombre. Digistore24 réessaie après un
  timeout, même quand le travail avait réussi.
- **Terminé est définitif.** Une fois l'accès terminé (remboursement,
  rétrofacturation, dernier jour payé), aucun événement ultérieur ne doit le
  rouvrir. Les livraisons n'arrivent pas dans l'ordre : un `on_payment` renvoyé
  peut arriver *après* le remboursement. Vérifiez l'état stocké avant même de
  regarder le nom de l'événement.
- **Un produit que vous ne connaissez pas n'accorde rien.** La connexion IPN
  est enregistrée avec une liste `product_ids`, et `all` — tout le compte
  vendeur — est un réglage courant (voir **`ds24-products`**). Des événements
  d'un ancien tunnel de vente, d'une deuxième app ou du lancement de quelqu'un
  d'autre peuvent donc légitimement atterrir sur votre endpoint.
  Stockez le payload, répondez `200`, n'accordez rien. N'associez jamais un id
  de produit inconnu à un plan par défaut : c'est distribuer de l'accès pour un
  achat qui n'a jamais été le vôtre, et personne ne remarque cette erreur avant
  que la mauvaise personne soit à l'intérieur.
- **Une même offre peut avoir PLUSIEURS ids de produit — associez-les tous.**
  Un produit Digistore24 porte exactement une langue ; une boutique multilingue
  vend donc chaque offre par un produit par langue (**`ds24-products`**), et le
  payload nomme celui que l'acheteur a réellement utilisé. Si vous n'associez
  que l'id allemand, chaque achat en anglais tombe sous la règle juste
  au-dessus : un client qui a payé, correctement enregistré, et à qui rien n'est
  accordé. Cherchez le `product_id` du payload parmi **tous** les ids de toutes
  les offres, et résolvez-les vers la même clé de produit.
- **Savoir à qui appartient ce paiement suit un ORDRE, et cet ordre se décide ici.**
  En premier, l'identifiant que votre checkout a placé dans `tracking[custom]`
  — celui-là est authentifié. L'e-mail de l'acheteur ne vient qu'ensuite, et
  comme supposition **non authentifiée** : Digistore24 ne vérifie pas l'adresse
  que l'acheteur a tapée. Et une adresse qui correspond à **plus d'un compte
  est refusée**, jamais résolue vers la première ligne. L'attribution accorde
  et ne révoque jamais — c'est la seule raison qui rend la voie de l'e-mail
  tolérable. L'ordre complet, ses refus et ce qu'une correspondance d'e-mail
  ne doit pas autoriser sont l'Étape 2 de **`ds24-checkout`** — lisez-la avant
  d'écrire cette partie, car ici chaque échec a l'air d'un endpoint qui
  fonctionne.

## Étape 5 — prouvez-le

Ne dites pas à l'utilisateur que ça marche. **`references/verification.fr.md`
dit ce qu'il faut prouver** — lisez-le, puis construisez la vérification avec
ce que cette plateforme sait exécuter.

Deux questions décident de la manière :

**Y a-t-il un shell ?** Replit, v0, Manus, Claude Code, Codex — oui. Alors
lancez le script livré avec cette skill ; il lui faut Node et une connexion
réseau, rien d'autre :

```bash
node scripts/verify-ipn.mjs \
  --url https://<l'app>/api/ipn \
  --passphrase "$DIGISTORE_IPN_PASSPHRASE" \
  --probe https://<l'app>/api/ds24-selftest --probe-token "$SECRET"
```

**Lovable n'en a pas.** Les skills y arrivent avec leurs fichiers embarqués,
mais la plateforme les lit sans les exécuter — sur Lovable, ce script est donc
une documentation, pas un outil. Écrivez l'équivalent sous forme de **test à
l'intérieur de l'app** (un test Deno sur Lovable Cloud), ce qui correspond à la
forme B dans `verification.fr.md`. C'est même plus simple : un test qui a
accès à la base de données lit l'enregistrement d'accès directement et n'a
besoin d'aucun endpoint de sonde.

**Une règle vaut dans les deux cas, et c'est tout l'enjeu :**

> Votre signature doit reproduire exactement chaque vecteur de
> `scripts/vectors.json`. **Ne calculez jamais les valeurs attendues avec votre
> propre code.** Le bug visé ici — signer avec des noms de champ en majuscules
> — donne une implémentation qui concorde avec elle-même en tout point et
> refuse pourtant chaque paiement réel. Une vérification écrite à partir du même
> malentendu confirme le bug.

Si vous construisez la vérification vous-même, cette comparaison est la
première chose qu'elle fait.

**Rapportez ce que l'exécution a dit**, y compris ce qu'elle n'a pas couvert.
Une exécution qui a sauté la moitié « accès » a prouvé la signature et rien de
la sémantique — dites-le, au lieu de la déclarer au vert.

### Quand aucune IPN n'arrive du tout, ce script ne peut rien

Il prouve ce que votre endpoint fait d'un payload. Un paiement qui ne l'a jamais
atteint ne laisse rien à vérifier ; la question porte alors sur la *connexion*,
pas sur le code. Demandez à Digistore24 ce qu'il connaît de cette commande :

```
POST https://www.digistore24.com/api/call/getPurchase/format/json
Header: X-DS-API-KEY: <la clé>
Body:   purchase_id=ABC12345
```

Inconnu là-bas → il n'y a pas eu d'achat (ou il a eu lieu dans un autre
compte vendeur). Connu là-bas mais absent de votre app → l'IPN n'est jamais arrivée :
une URL enregistrée qui ne répond plus, un `domain_id` qu'un autre projet a
écrasé, ou une liste `product_ids` dans laquelle ce produit ne figure pas. Les
trois relèvent de **`ds24-products`**, Étape 4 — et les trois échouent sans le
moindre message d'erreur.

Pour vérifier les modules de signature livrés, isolément, là où un shell
existe :

```bash
node scripts/check-adapters.mjs      # les trois runtimes contre les vecteurs
```

## Étape 6 — la suite

L'endpoint reçoit les événements. Trois choses doivent encore exister autour de
lui :

- **`ds24-products`** — obtenir la clé d'API, créer les produits chez
  Digistore24 et enregistrer cet endpoint comme connexion IPN. Sans cela,
  personne ne vous appelle jamais. **Commencez par là.**
- **`ds24-entitlements`** — l'enregistrement d'accès dans lequel les événements
  écrivent, et l'unique fonction que le reste de l'app interroge.
- **`ds24-checkout`** — le lien d'achat qui démarre un achat.

Dites laquelle vous commencez, et commencez-la.
