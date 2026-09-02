---
name: ds24-skills
language: fr
description: À utiliser dès qu'une app doit encaisser via Digistore24 — un checkout ou un bouton d'achat, un paywall, un abonnement, des crédits prépayés, le webhook IPN qui annonce ce qui a été payé, la mise en production, ou une intégration Digistore24 qui se comporte mal (signature invalide, un achat qui n'a rien débloqué, un client remboursé qui a toujours accès). C'est le Skill Pack Digistore24 : il contient huit skills et lance celle que la tâche demande. À utiliser chaque fois que Digistore24 est nommé, et chaque fois qu'il est question de facturation, de paiements ou d'un espace payant avec Digistore24 comme prestataire.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Le Skill Pack Digistore24

Digistore24 est le marchand officiel (merchant of record) : c'est lui qui tient
le checkout, encaisse l'argent, gère la TVA et les remboursements, puis informe
votre app de ce qui s'est passé par un webhook signé. Ce pack rassemble la partie
de tout cela qu'un agent ne peut pas deviner — l'algorithme de signature, la
sémantique des événements, les modes de défaillance, et des vecteurs de test
figés qui prouvent le résultat.

**Ce fichier est la porte, pas le manuel.** Il dit ce que contient le pack et
comment le lire. Le fonctionnement de Digistore24 n'a rien à faire sur cette
page : il est décrit dans les huit skills, et il l'est mieux là-bas, parce qu'on
l'y tient à jour en même temps qu'elles.

## Étape 1 — cette copie est-elle à jour ?

Récupérez
`https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION` et
comparez-le au fichier `VERSION` qui se trouve à côté de celui-ci. Une skill
importée dans un workspace ne se met pas à jour toute seule ; une copie peut donc
dater de plusieurs mois. En cas d'écart, dites-le en une phrase, proposez de
réimporter, puis poursuivez dans les deux cas.

## Étape 2 — lisez le point d'entrée et suivez-le

```
skills/ds24-billing/SKILL.fr.md
```

`ds24-billing` examine ce que le projet contient déjà, nomme la seule étape
suivante et lance la skill qui s'en charge. Lisez-la maintenant — ne planifiez
pas l'intégration depuis cette page, et ne posez pas à l'utilisateur de questions
auxquelles cette skill répond en regardant le projet.

**Vous lisez la version française.** Chaque fichier de ce pack existe aussi
en `.es.md` et en `.fr.md` à côté de l'original anglais — le point d'entrée
ci-dessus est déjà le fichier `.fr.md`. Les trois disent la même chose ; seule la
prose est traduite : les commandes et les chemins de scripts sont les mêmes, et
les renvois d'un fichier à l'autre mènent au fichier de la même langue.
Choisissez selon la langue dans laquelle l'utilisateur écrit, gardez-la pour tout
le pack et répondez dans cette langue.

## Étape 3 — comment lire l'une des huit

Ce sont des fichiers de cette skill. Essayez dans cet ordre et, si la première
voie n'a pas marché, dites laquelle a fonctionné :

1. **`skills/<name>/SKILL.fr.md`** — la copie arrivée avec cette skill.
2. **`https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/<name>/SKILL.fr.md`**
   — le même fichier sur GitHub, si votre plateforme n'a pas conservé les
   dossiers embarqués. Les fichiers qu'une skill embarque se trouvent à la même
   adresse, complétée par leur entrée `references/…`, `scripts/…` ou
   `adapters/…`.

**Si aucune des deux voies ne marche, arrêtez-vous et dites-le.** Ne
reconstruisez pas de mémoire une intégration Digistore24. Ce que ces skills
contiennent, c'est justement la partie qui paraît évidente et qui est fausse : à
la deviner, on obtient une intégration dont les propres tests passent et qui
refuse chaque paiement réel pour « signature invalide ».

## Les huit

| Skill | À lire quand |
|---|---|
| **`ds24-billing`** | **toujours en premier** — elle établit ce qui existe et lance la bonne |
| `ds24-products` | la clé d'API, la création des produits, l'enregistrement de la connexion IPN |
| `ds24-ipn` | le webhook : signature, événements, idempotence — et comment le prouver |
| `ds24-entitlements` | faire d'une commande payée un « peut utiliser le produit » |
| `ds24-checkout` | le lien d'achat, le prix sous forme de plan de paiement, la page de remerciement |
| `ds24-tokens` | les crédits prépayés, leur consommation, la recharge automatique |
| `ds24-golive` | la vérification préalable, le vrai achat de test, et le remboursement qui prouve le reste |
| `ds24-compliance` | les mentions légales, la politique de confidentialité, la divulgation IA qu'impose le règlement européen sur l'IA (EU AI Act), la suppression |

## Deux choses à emporter dans chacune d'elles

**L'accès vient d'un événement signé, jamais d'un navigateur.** Une page de
remerciement n'est qu'une URL que n'importe qui peut ouvrir. Seule l'IPN dont
vous avez vérifié la signature peut accorder quoi que ce soit.

**Prouvez-le, ne l'annoncez pas.** Chaque skill se termine par quelque chose qui
s'exécute. Les huit vecteurs figés de `skills/ds24-ipn/scripts/vectors.json` ne
doivent jamais être recalculés avec votre propre code : le bug qu'ils détectent
produit une implémentation en accord total avec elle-même, qui rejette chaque
paiement réel — un test écrit à partir du même malentendu ne fait donc que
confirmer le bug. Reproduisez-les à l'identique.

## Là où il y a un shell

Ce fichier existe pour les plateformes qui importent une skill à la fois —
Lovable et Manus — afin qu'un seul import apporte les huit. Là où une ligne de
commande est disponible, mieux vaut les installer comme huit skills distinctes,
car chacune se charge alors sur son propre déclencheur :

```
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

Cette adresse se termine volontairement par `/skills` : pointé sur le dépôt
lui-même, l'installateur s'arrête à ce fichier et installe cette seule skill à la
place des huit.
