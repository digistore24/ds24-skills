<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`README.md`](README.md) · [Español](README.es.md)

# Digistore24 Skills

**Des Agent Skills qui apprennent à n'importe quel agent de codage IA à facturer
via Digistore24 — dans quelque app qu'il construise, sur quelque stack que ce
soit.**

Fonctionne avec **Lovable**, **Manus**, **Replit**, **v0**, **Claude Code**,
**Codex** et tout ce qui lit la convention `SKILL.md`.

Ce n'est ni un modèle ni une bibliothèque. C'est la partie d'une intégration de
paiement qu'un agent ne peut pas deviner — l'algorithme de signature, la
sémantique des événements, les modes de défaillance — plus **des vecteurs de
test figés et une spécification de ce qui doit être prouvé**, pour que le
résultat soit démontré plutôt qu'affirmé.

---

## Installation

### La voie courte — laissez votre agent lire les instructions

Collez ceci dans ce avec quoi vous construisez :

```
Lisez https://ds24-skills.com/start.md et suivez-le.
Ajoutez la facturation Digistore24 à cette app.
```

Il détermine ce dans quoi il s'exécute et prend le relais — installant les
skills lui-même là où il a un shell, et vous indiquant les deux clics là où il
n'en a pas. Tout ce qui suit, c'est la même chose faite à la main.

**Ni Lovable ni Manus ne vous demandent git ou un terminal.** Choisissez votre
ligne.

### Lovable — collez une seule adresse

*Skills → Add → Import from GitHub*, puis collez :

```
https://github.com/digistore24/ds24-skills
```

C'est toute l'installation, et elle apporte **les huit** — pas de
téléchargement, pas de décompression, rien d'installé sur votre machine.

Lovable importe une skill par adresse, donc ce qui arrive est le
[`SKILL.fr.md`](SKILL.fr.md) propre au Skill Pack : la porte. Il porte les huit
autres sous forme de fichiers embarqués, vérifie si votre copie est à jour et lance
`ds24-billing`, qui détermine ce que votre projet possède déjà. Demandez la
facturation Digistore24 et il prend le relais.

Vous voulez aussi les skills individuelles comme commandes `/` à part entière —
`/ds24-ipn` pendant que vous déboguez un webhook, par exemple ? Importez-les de
la même façon, une adresse chacune :

```
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-billing
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-products
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-ipn
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-entitlements
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-checkout
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-tokens
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-golive
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-compliance
```

### Manus — une adresse, ou le ZIP

Manus a lui aussi *Skills → **+ Add → Import from GitHub***, et la même adresse
y fonctionne.

Vous préférez ne pas passer par GitHub :

1. **[Téléchargez le Skill Pack en ZIP](https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip)**
   *(github.com/digistore24/ds24-skills/archive/refs/heads/main.zip)*
2. Décompressez-le. À l'intérieur vous obtenez `ds24-skills-main/` — déposez ce
   dossier entier pour avoir le Skill Pack d'un coup, ou un seul dossier issu de
   `skills/` pour n'avoir que celui-là.
3. Dans Manus : *Skills* dans la barre latérale gauche → **+ Add → Upload a
   skill**.

Invoquez-le directement avec `/ds24-skills`, ou demandez simplement la
facturation Digistore24.

### Replit, v0, Claude Code, Codex — une seule commande

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

**L'adresse se termine par `/skills`, et cela compte.** Pointé sur le dépôt
lui-même, l'installateur s'arrête au `SKILL.md` propre au Skill Pack — la porte
écrite pour Lovable — et installe cette seule skill au lieu des huit. Avec
`/skills` vous obtenez les huit, chacune se chargeant sur son propre
déclencheur, ce qui est ce que vous voulez là où une ligne de commande existe.

Il installe dans `.agents/skills/` — exactement là où l'Agent de Replit regarde
— et les lie dans `.claude/skills/` pour Claude Code. Les adaptateurs joints,
les références, les vecteurs de test et le vérificateur tout prêt viennent avec.

<details>
<summary>Vous préférez ne pas exécuter un paquet npx ?</summary>

Juste — l'installateur le dit lui-même : les skills s'exécutent avec tous les
droits de l'agent, alors lisez-les d'abord. La voie manuelle est une copie :

```bash
curl -sSL -o /tmp/ds24.zip https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip
unzip -q /tmp/ds24.zip -d /tmp
mkdir -p .agents/skills && cp -r /tmp/ds24-skills-main/skills/* .agents/skills/
```

Utilisez `.claude/skills/` au lieu de `.agents/skills/` pour Claude Code.
</details>

---

Dites ensuite **« ajoute la facturation Digistore24 à cette app »** — ou
invoquez le point d'entrée par son nom, `ds24-billing`.

---

## Ce qu'il contient

| Skill | Ce qu'elle fait |
|---|---|
| **`ds24-billing`** | le point d'entrée : détermine ce qui existe déjà et lance la bonne skill suivante |
| **`ds24-products`** | la clé d'API, la création de produits, l'enregistrement de la connexion IPN, l'approbation |
| **`ds24-ipn`** | le webhook : signature, événements, idempotence — **et comment le prouver** |
| **`ds24-checkout`** | le lien d'achat, le prix comme plan de paiement, le transport de l'identité de l'acheteur |
| **`ds24-entitlements`** | l'enregistrement d'accès et l'unique fonction que l'app interroge |
| **`ds24-tokens`** | les crédits prépayés, leur consommation, la recharge automatique |
| **`ds24-golive`** | la vérification préalable, l'achat de test réel, et le remboursement qui prouve l'autre moitié |
| **`ds24-compliance`** | les mentions légales, la politique de confidentialité, la divulgation du règlement européen sur l'IA (EU AI Act), l'accès et la suppression |

## Les trois choses qu'un agent ne peut pas deviner

1. **Digistore24 signe avec la casse D'ORIGINE des noms de champs**
   (`order_id=…`), pas en majuscules — même si son propre exemple PHP suggère le
   contraire. Trompez-vous là-dessus et chacun de vos propres tests passe tandis
   que chaque paiement réel est rejeté comme « signature invalide ».
2. **`on_rebill_cancelled` ne fait rien à l'accès.** La facturation s'arrête ; la
   période payée continue de courir. Mettre fin à l'accès là retire au client des
   mois qu'il a payés. L'accès prend fin à `last_paid_day`.
3. **Un paiement manqué suspend de façon réversible.** Une carte expirée n'est
   pas un départ, et le paiement qui règle cela doit *lever* la suspension — un
   insert-if-absent ne le fera pas.

## Prouver que ça marche

Un texte ne peut pas garantir qu'un agent a construit correctement la
vérification de signature, et « probablement juste » ne vaut rien pour un rail
de paiement. Le Skill Pack livre donc une **spécification de ce qui doit être
prouvé** —
[`verification.fr.md`](skills/ds24-ipn/references/verification.fr.md) — et
l'agent construit la vérification dans ce qui tourne sur sa plateforme.

**La partie qui ne peut pas être improvisée**, ce sont les huit vecteurs figés
de [`vectors.json`](skills/ds24-ipn/scripts/vectors.json). Toute implémentation
doit les reproduire exactement, et personne ne peut les recalculer avec son
propre code :

> Le bug qu'ils attrapent — signer avec des noms de champs en majuscules —
> produit une implémentation qui s'accorde parfaitement avec elle-même et
> rejette **chaque paiement réel**. Un test écrit par le même auteur, à partir
> du même malentendu, confirme le bug. Les valeurs attendues viennent donc de
> l'extérieur.

Ce sont les mêmes vecteurs auxquels se mesure la [Digistore SAAS App
Template](https://github.com/digistore24/ds24-appkit).

**Là où il y a un shell** — Replit, v0, Manus, Claude Code, Codex, ou votre
propre machine — un vérificateur tout prêt est livré avec la skill. Il lui faut
Node et une connexion réseau, rien d'autre, donc il tourne contre une Supabase
Edge Function sur Lovable Cloud exactement comme contre une route Next.js sur
Replit. Après `npx skills add` :

```bash
node .agents/skills/ds24-ipn/scripts/verify-ipn.mjs \
  --url https://your-app.example.com/api/ipn \
  --passphrase "$DIGISTORE_IPN_PASSPHRASE" \
  --probe https://your-app.example.com/api/ds24-selftest --probe-token "$SECRET"
```

**Lovable n'a pas de shell.** Là-bas les fichiers embarqués voyagent avec une
skill, mais la plateforme les lit au lieu de les exécuter — la skill fait donc
écrire à l'agent l'équivalent sous forme de test *à l'intérieur de l'app*. Cela
se révèle plus simple : un test avec accès à la base de données lit
l'enregistrement d'accès directement et n'a besoin d'aucun endpoint de sonde. Ou
lancez le script ci-dessus depuis votre propre machine contre l'URL déployée.

Dans un cas comme dans l'autre, voici ce qui est prouvé :

| Cas | Doit |
|---|---|
| **les huit vecteurs** | **être reproduits exactement** — vérifié en premier, avant toute autre chose |
| `on_payment` correctement signé | être accepté, l'accès accordé |
| un octet retourné dans la signature | être rejeté |
| pas de signature, ou pas de passphrase | être rejeté (échouer en mode fermé) |
| signature à clés en majuscules | être acceptée |
| le même événement deux fois | ne pas créditer deux fois |
| `on_refund` | retirer l'accès |
| `on_payment_missed` → `on_payment` | suspendre, puis restaurer |
| `on_rebill_cancelled` | laisser l'accès **inchangé** |
| un paiement relivré après un remboursement | **ne pas** raviver l'accès |

Le `--probe` ci-dessus est ce dont la moitié « accès » a besoin lors d'une
vérification depuis l'extérieur : un petit endpoint protégé par token qui répond
`{"access": true|false}` pour un `order_id`, supprimé de nouveau une fois que
l'exécution est au vert. Laissez-le de côté et ces lignes sont rapportées comme
`SKIP` — jamais silencieusement comptées comme réussies.

Pour vérifier les modules de signature livrés à part, là où un shell existe :

```bash
node .agents/skills/ds24-ipn/scripts/check-adapters.mjs
```

## Adaptateurs

[`skills/ds24-ipn/adapters/`](skills/ds24-ipn/adapters) contient deux sortes de
fichiers, et la différence compte :

**La signature — copiez telle quelle, ne l'éditez jamais :**

| Runtime | Fichier |
|---|---|
| Node | `signature-node.mjs` |
| Deno · Supabase Edge Functions · **Lovable Cloud** · Cloudflare Workers | `signature-web.mjs` |
| Python | `signature.py` |

**L'endpoint — un exemple que vous adaptez :** `next-node.ts`, `deno-edge.ts`,
`express-node.js`, `python-fastapi.py`.

> **Sur Lovable Cloud / Supabase, déployez la fonction avec `verify_jwt = false`.**
> Digistore24 n'envoie aucun JWT Supabase, donc avec la valeur par défaut
> activée chaque IPN reçoit un 401 avant que votre code ne s'exécute et chaque
> achat ne débloque silencieusement rien.

## Mise à jour

**Sur Lovable et Manus, les skills vivent dans votre workspace, pas dans votre
dépôt — elles ne se mettent donc pas à jour toutes seules.** Ce que vous avez
importé reste jusqu'à ce que vous importiez de nouveau. Chaque skill commence
donc par comparer son propre `VERSION` à

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION
```

et dit quelque chose quand ils diffèrent.

Partout ailleurs, mettre à jour est la même commande que celle qui les a
installées :

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

## Ce que ceci n'est pas

- **Ce n'est pas une app.** Pas d'authentification, pas de table utilisateurs,
  pas d'UI. Votre agent construit cela ; ces skills rendent la partie argent
  correcte.
- **La vérification couvre le chemin de l'argent**, pas la question de savoir si
  chaque page de votre app contrôle les permissions.
- **C'est de la préparation, pas un conseil juridique.** `ds24-compliance` règle
  correctement les choses évidentes et nomme ce qu'un avocat devrait voir.

Si vous préférez partir d'un SaaS fini et fonctionnel avec tout cela déjà
intégré, c'est un autre produit :
**[ds24-appkit.com](https://ds24-appkit.com)** — un modèle SaaS Next.js complet
que vous étendez avec Claude Code.

## Licence

MIT — voir [`LICENSE`](LICENSE). Utilisez-le, modifiez-le, lancez des produits
avec, vendez-les. Sans frais, sans personne à qui demander.
