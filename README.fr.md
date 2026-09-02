<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Français** · Original en anglais — [`README.md`](README.md) · [Español](README.es.md)

# Digistore24 Skills

**Des Agent Skills qui apprennent à n'importe quel agent de codage IA à facturer
via Digistore24 — dans l'app qu'il construit, quelle que soit la stack.**

Fonctionne avec **Lovable**, **Manus**, **Replit**, **v0**, **Claude Code**,
**Codex** et tout outil qui lit la convention `SKILL.md`.

Ce n'est ni un modèle ni une bibliothèque. C'est la part d'une intégration de
paiement qu'un agent ne peut pas deviner — l'algorithme de signature, la
sémantique des événements, les modes de défaillance —, complétée par **des
vecteurs de test figés et une spécification de ce qui doit être prouvé**, pour
que le résultat soit démontré et non simplement affirmé.

---

## Installation

### La voie courte — laissez votre agent lire les instructions

Collez ceci dans l'outil avec lequel vous construisez :

```
Lisez https://ds24-skills.com/start.md et suivez-le.
Ajoutez la facturation Digistore24 à cette app.
```

L'agent reconnaît la plateforme sur laquelle il tourne et se charge de la
suite : il installe les skills lui-même là où il dispose d'un shell, et vous
indique les deux clics à faire là où il n'en a pas. Tout ce qui suit est la
même chose, faite à la main.

**Avec Lovable comme avec Manus, vous n'avez besoin ni de git ni d'un
terminal.** Cherchez votre plateforme ci-dessous.

### Lovable — collez une seule adresse

*Skills → Add → Import from GitHub*, puis collez :

```
https://github.com/digistore24/ds24-skills
```

L'installation s'arrête là, et elle apporte **les huit skills** — rien à
télécharger, rien à décompresser, rien d'installé sur votre machine.

Lovable importe une skill par adresse ; ce qui arrive est donc le
[`SKILL.md`](SKILL.fr.md) du pack lui-même : la porte. Elle transporte les huit
autres sous forme de fichiers embarqués, vérifie si votre copie est à jour et
lance `ds24-billing`, qui établit ce que votre projet contient déjà. Demandez la
facturation Digistore24, et elle s'occupe de la suite.

Vous voulez en plus certaines skills comme commandes `/` à part entière —
`/ds24-ipn` pendant que vous déboguez un webhook, par exemple ? Importez-les de
la même manière, une adresse par skill :

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

Manus propose lui aussi *Skills → **+ Add → Import from GitHub***, et la même
adresse y fonctionne.

Si vous préférez éviter GitHub :

1. **[Téléchargez le pack en ZIP](https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip)**
   *(github.com/digistore24/ds24-skills/archive/refs/heads/main.zip)*
2. Décompressez-le. Vous y trouvez `ds24-skills-main/` — déposez ce dossier
   entier pour obtenir tout le pack d'un coup, ou un seul des dossiers de
   `skills/` pour n'avoir que cette skill-là.
3. Dans Manus : *Skills* dans la barre latérale de gauche → **+ Add → Upload a
   skill**.

Invoquez-le directement avec `/ds24-skills`, ou demandez simplement la
facturation Digistore24.

### Replit, v0, Claude Code, Codex — une seule commande

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

**L'adresse se termine par `/skills`, et ce détail compte.** Pointé sur le
dépôt lui-même, l'installateur s'arrête au `SKILL.md` du pack — la porte écrite
pour Lovable — et installe cette seule skill à la place des huit. Avec
`/skills`, vous obtenez les huit, chacune se chargeant sur son propre
déclencheur : c'est ce qu'il faut partout où une ligne de commande existe.

Les skills s'installent dans `.agents/skills/` — exactement là où l'Agent de
Replit va les chercher — et un lien vers elles est posé dans `.claude/skills/`
pour Claude Code.
Les adaptateurs, les références, les vecteurs de test et le vérificateur tout
prêt, embarqués avec elles, arrivent en même temps.

<details>
<summary>Vous préférez ne pas exécuter un paquet npx ?</summary>

C'est légitime — l'installateur le dit lui-même : les skills s'exécutent avec
tous les droits de l'agent, alors lisez-les avant. La voie manuelle est une
simple copie :

```bash
curl -sSL -o /tmp/ds24.zip https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip
unzip -q /tmp/ds24.zip -d /tmp
mkdir -p .agents/skills && cp -r /tmp/ds24-skills-main/skills/* .agents/skills/
```

Pour Claude Code, remplacez `.agents/skills/` par `.claude/skills/`.
</details>

---

Dites ensuite **« ajoutez la facturation Digistore24 à cette app »** — ou
invoquez le point d'entrée par son nom, `ds24-billing`.

---

## Ce qu'il contient

| Skill | Ce qu'elle fait |
|---|---|
| **`ds24-billing`** | le point d'entrée : établit ce qui existe déjà et lance la skill qui convient ensuite |
| **`ds24-products`** | la clé d'API, la création des produits, l'enregistrement de la connexion IPN, l'approbation |
| **`ds24-ipn`** | le webhook : signature, événements, idempotence — **et comment le prouver** |
| **`ds24-checkout`** | le lien d'achat, le prix sous forme de plan de paiement, l'identité de l'acheteur transportée jusqu'au bout |
| **`ds24-entitlements`** | l'enregistrement d'accès et l'unique fonction que l'app interroge |
| **`ds24-tokens`** | les crédits prépayés, leur consommation, la recharge automatique |
| **`ds24-golive`** | la vérification préalable, le véritable achat de test, et le remboursement qui prouve l'autre moitié |
| **`ds24-compliance`** | les mentions légales, la politique de confidentialité, la divulgation exigée par le règlement européen sur l'IA (EU AI Act), l'accès et la suppression |

## Les trois choses qu'un agent ne peut pas deviner

1. **Digistore24 signe avec la casse D'ORIGINE des noms de champ**
   (`order_id=…`), pas en majuscules — même si son propre exemple PHP laisse
   entendre le contraire. Une erreur ici fait passer chacun de vos propres tests
   pendant que chaque paiement réel est rejeté pour « signature invalide ».
2. **`on_rebill_cancelled` ne touche pas à l'accès.** La facturation s'arrête ;
   la période payée continue de courir. Couper l'accès à ce moment-là retire au
   client des mois qu'il a payés. L'accès prend fin à `last_paid_day`.
3. **Un paiement manqué suspend de façon réversible.** Une carte expirée n'est
   pas un départ, et le paiement qui répare la situation doit *lever* la
   suspension — une écriture du type « insérer si absent » ne le fera pas.

## Prouver que ça marche

Un texte ne peut pas garantir qu'un agent a construit correctement la
vérification de signature, et « probablement juste » ne vaut rien pour un
circuit de paiement. Le pack livre donc une **spécification de ce qui doit être
prouvé** — [`verification.fr.md`](skills/ds24-ipn/references/verification.fr.md)
— et l'agent construit la vérification avec ce qui s'exécute sur sa plateforme.

**La partie qui ne s'improvise pas**, ce sont les huit vecteurs figés de
[`vectors.json`](skills/ds24-ipn/scripts/vectors.json). Toute implémentation
doit les reproduire à l'identique, et personne n'a le droit de les recalculer
avec son propre code :

> Le bug qu'ils attrapent — signer avec des noms de champ en majuscules —
> produit une implémentation en accord total avec elle-même, qui rejette
> **chaque paiement réel**. Un test écrit par le même auteur, à partir du même
> malentendu, confirme le bug. Les valeurs attendues viennent donc de
> l'extérieur.

Ce sont les vecteurs mêmes sur lesquels le
[Digistore SAAS App Template](https://github.com/digistore24/ds24-appkit) se
mesure.

**Là où il y a un shell** — Replit, v0, Manus, Claude Code, Codex ou votre
propre machine —, un vérificateur tout prêt est livré avec la skill. Il ne lui
faut que Node et une connexion réseau, rien d'autre : il tourne donc contre une
Supabase Edge Function sur Lovable Cloud aussi bien que contre une route Next.js
sur Replit. Après `npx skills add` :

```bash
node .agents/skills/ds24-ipn/scripts/verify-ipn.mjs \
  --url https://your-app.example.com/api/ipn \
  --passphrase "$DIGISTORE_IPN_PASSPHRASE" \
  --probe https://your-app.example.com/api/ds24-selftest --probe-token "$SECRET"
```

**Lovable n'a pas de shell.** Les fichiers embarqués y voyagent bien avec la
skill, mais la plateforme les lit sans les exécuter — la skill demande donc à
l'agent d'écrire l'équivalent sous forme de test *à l'intérieur de l'app*. C'est
même plus simple : un test qui a accès à la base de données lit l'enregistrement
d'accès directement et n'a besoin d'aucun endpoint de sonde. Vous pouvez aussi
lancer le script ci-dessus depuis votre propre machine contre l'URL déployée.

Dans les deux cas, voici ce qui est prouvé :

| Cas | Doit |
|---|---|
| **les huit vecteurs** | **être reproduits à l'identique** — vérifié en premier, avant tout le reste |
| `on_payment` correctement signé | être accepté, l'accès accordé |
| un octet inversé dans la signature | être rejeté |
| pas de signature, ou pas de passphrase | être rejeté (refus par défaut, *fail closed*) |
| signature à clés en majuscules | être acceptée |
| le même événement deux fois | ne pas créditer deux fois |
| `on_refund` | retirer l'accès |
| `on_payment_missed` → `on_payment` | suspendre, puis rétablir |
| `on_rebill_cancelled` | laisser l'accès **inchangé** |
| un paiement renvoyé après un remboursement | **ne pas** faire revivre l'accès |

Le `--probe` ci-dessus est ce dont la moitié « accès » a besoin quand la
vérification vient de l'extérieur : un petit endpoint protégé par un token, qui
répond `{"access": true|false}` pour un `order_id` et que l'on supprime une fois
l'exécution au vert. Sans lui, ces lignes sont rapportées comme `SKIP` — jamais
comptées en silence comme réussies.

Pour vérifier isolément les modules de signature livrés, là où un shell existe :

```bash
node .agents/skills/ds24-ipn/scripts/check-adapters.mjs
```

## Adaptateurs

[`skills/ds24-ipn/adapters/`](skills/ds24-ipn/adapters) contient deux sortes de
fichiers, et la différence compte :

**La signature — à copier telle quelle, sans jamais la modifier :**

| Runtime | Fichier |
|---|---|
| Node | `signature-node.mjs` |
| Deno · Supabase Edge Functions · **Lovable Cloud** · Cloudflare Workers | `signature-web.mjs` |
| Python | `signature.py` |

**L'endpoint — un exemple, que vous adaptez :** `next-node.ts`, `deno-edge.ts`,
`express-node.js`, `python-fastapi.py`.

> **Sur Lovable Cloud / Supabase, déployez la fonction avec `verify_jwt = false`.**
> Digistore24 n'envoie aucun JWT Supabase : avec le réglage par défaut, chaque
> IPN reçoit un 401 avant même que votre code s'exécute, et chaque achat ne
> débloque rien, en silence.

## Mise à jour

**Sur Lovable et Manus, les skills vivent dans votre workspace, pas dans votre
dépôt — elles ne se mettent donc pas à jour d'elles-mêmes.** Ce que vous avez
importé reste tel quel jusqu'à la prochaine importation. C'est pourquoi chaque
skill commence par comparer son propre `VERSION` à

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION
```

et le signale quand les deux diffèrent.

Partout ailleurs, la mise à jour est la commande même qui les a installées :

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

## Ce que ceci n'est pas

- **Ce n'est pas une app.** Pas d'authentification, pas de table utilisateurs,
  pas d'interface. Tout cela, votre agent le construit ; ces skills font en
  sorte que la partie argent soit juste.
- **La vérification couvre le circuit de l'argent**, pas la question de savoir
  si chaque page de votre app contrôle les droits.
- **C'est une préparation, pas un conseil juridique.** `ds24-compliance` met
  les évidences en ordre et désigne ce qu'un avocat devrait examiner.

Si vous préférez partir d'un SaaS fini et fonctionnel, où tout cela est déjà
intégré, c'est un autre produit : **[ds24-appkit.com](https://ds24-appkit.com)**
— un modèle SaaS Next.js complet, que vous étendez avec Claude Code.

## Licence

MIT — voir [`LICENSE`](LICENSE). Utilisez-le, modifiez-le, livrez des produits
avec, vendez-les. Sans frais, et personne à qui demander la permission.
