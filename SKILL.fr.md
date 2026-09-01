---
name: ds24-skills
language: fr
description: À utiliser quand une app doit encaisser de l'argent via Digistore24 — un checkout ou un bouton d'achat, un paywall, un abonnement, des crédits prépayés, le webhook IPN qui dit ce qui a été payé, la mise en production, ou une intégration Digistore24 qui se comporte mal (signature invalide, un achat qui n'a rien déverrouillé, un client remboursé qui a toujours accès). C'est le Skill Pack Digistore24, qui porte huit skills et démarre celle dont la tâche a besoin. À utiliser dès que Digistore24 est nommé, et dès qu'il est question de facturation, de paiements ou d'un espace payant et que Digistore24 est le fournisseur.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Le Skill Pack Digistore24

Digistore24 est le marchand de référence (merchant of record) — il gère le
checkout, encaisse l'argent, s'occupe de la TVA et des remboursements, et raconte
à votre app ce qui s'est passé au moyen d'un webhook signé. Ce pack est la partie
de tout cela qu'un agent ne peut pas deviner — l'algorithme de signature, la
sémantique des événements, les modes de défaillance et les vecteurs de test figés
qui prouvent le résultat.

**Ce fichier est la porte, pas le manuel.** Il dit ce qu'il y a ici et comment le
lire. Rien sur le fonctionnement de Digistore24 n'a sa place sur cette page —
cela est dans les huit skills, et c'est mieux écrit là-bas parce que c'est tenu à
jour avec elles.

## Étape 1 — cette copie est-elle à jour ?

Récupérez
`https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION` et
comparez-le au fichier `VERSION` placé à côté de celui-ci. Les skills importées
dans un workspace ne se mettent pas à jour toutes seules, une copie peut
donc avoir des mois. Signalez tout écart en une phrase, proposez de réimporter,
puis continuez dans tous les cas.

## Étape 2 — lisez le point d'entrée et suivez-le

```
skills/ds24-billing/SKILL.fr.md
```

`ds24-billing` regarde ce que le projet a déjà, nomme l'unique étape suivante et
démarre la skill qui la réalise. Lisez-la maintenant — ne planifiez pas
l'intégration depuis cette page, et ne posez pas à l'utilisateur des questions
auxquelles elle répond en regardant.

## Étape 3 — comment lire n'importe laquelle des huit

Ce sont des fichiers de cette skill. Essayez ceci dans l'ordre, et dites laquelle
a marché si ce n'est pas la première :

1. **`skills/<name>/SKILL.fr.md`** — la copie qui a voyagé avec cette skill.
2. **`https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/<name>/SKILL.fr.md`**
   — le même fichier depuis GitHub, si votre plateforme n'a pas conservé les
   dossiers embarqués. Les fichiers que porte une skill pendent de la même
   adresse, avec son entrée `references/…`, `scripts/…` ou `adapters/…` ajoutée.

**Si aucune des deux ne marche, arrêtez-vous et dites-le.** Ne reconstruisez pas
de mémoire une intégration Digistore24. Ce que portent ces skills, c'est
précisément la partie qui a l'air évidente et qui est fausse — devinée, le
résultat est une intégration dont les propres tests passent et dont chaque
paiement réel est refusé pour « signature invalide ».

## Les huit

| Skill | À lire quand |
|---|---|
| **`ds24-billing`** | **toujours en premier** — elle détermine ce qui existe et démarre la bonne |
| `ds24-products` | la clé d'API, la création de produits, l'enregistrement de la connexion IPN |
| `ds24-ipn` | le webhook : la signature, les événements, l'idempotence — et comment le prouver |
| `ds24-entitlements` | transformer une commande payée en « peut utiliser le produit » |
| `ds24-checkout` | le lien d'achat, le prix comme plan de paiement, la page de remerciement |
| `ds24-tokens` | les crédits prépayés, les consommer, la recharge automatique |
| `ds24-golive` | la vérification préalable, l'achat de test réel et le remboursement qui prouve le reste |
| `ds24-compliance` | les mentions légales, la politique de confidentialité, la divulgation du règlement européen sur l'IA (EU AI Act), la suppression |

## Deux choses à emporter dans chacune d'elles

**L'accès naît d'un événement signé, jamais d'un navigateur.** Une page de
remerciement est une URL que n'importe qui peut ouvrir. Seule l'IPN, dont vous
avez vérifié la signature, peut accorder quoi que ce soit.

**Prouvez-le, ne l'annoncez pas.** Chaque skill se termine par quelque chose qui
peut être exécuté. Les huit vecteurs figés de
`skills/ds24-ipn/scripts/vectors.json` ne doivent jamais être recalculés avec
votre propre code — le bug qu'ils attrapent produit une implémentation qui
s'accorde parfaitement avec elle-même et refuse chaque paiement réel, si bien
qu'un test écrit à partir du même malentendu confirme le bug. Reproduisez-les
exactement.

## Là où il y a un shell

Ce fichier existe pour les plateformes qui importent une skill à la fois —
Lovable et Manus — afin qu'un seul import apporte les huit. Là où une ligne de
commande est disponible, les installer comme huit skills distinctes vaut mieux,
car chacune se charge alors sur son propre déclencheur :

```
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

Cette adresse se termine par `/skills` exprès : pointé sur le dépôt lui-même,
l'installateur s'arrête à ce fichier et installe cette seule skill au lieu des
huit.
