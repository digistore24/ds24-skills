---
name: ds24-compliance
language: fr
description: À utiliser quand une app qui encaisse de l'argent auprès de clients de l'UE doit mettre son volet juridique en ordre — les mentions légales, la politique de confidentialité, les conditions, la divulgation du règlement européen sur l'IA (EU AI Act) pour tout ce qui parle aux personnes en tant que machine, la question de savoir si un bandeau cookies est seulement nécessaire, les registres de consentement et le droit du client à ses données et à la suppression. À utiliser dès que l'utilisateur pose une question sur le RGPD (GDPR), un bandeau cookies, les mentions légales, l'AI Act, la suppression de compte ou ce qu'il doit afficher avant de vendre à de vrais clients.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Ce que l'UE demande à une app payante

Ceci est une préparation, pas un conseil juridique. Cela règle correctement les
choses évidentes et nomme celles qu'un avocat devrait regarder. Dites-le une
fois, au début, puis soyez utile.

Le déclencheur n'est pas la taille. C'est **le fait d'encaisser de l'argent
auprès de personnes dans l'UE**, ce à quoi sert précisément une intégration
Digistore24.

## Étape 0 — qu'y a-t-il déjà ?

Regardez avant d'écrire quoi que ce soit. Cherchez dans le projet des mentions
légales, une politique de confidentialité, des conditions, un bandeau cookies,
une table de consentements.

**Quoi qu'il existe, ne le remplacez pas.** Les pages juridiques sont
fréquemment la seule partie d'une app qu'un avocat a déjà vue, et une politique
de confidentialité réécrite qui se lit mieux et dit autre chose est pire qu'une
maladroite mais exacte. Lisez ce qui est là, confrontez-le aux étapes ci-dessous
et signalez les lacunes — changer la formulation est la décision de
l'utilisateur, pas la vôtre.

Un **bandeau cookies déjà installé** est la seule chose qui mérite d'être
remise en question à voix haute : l'étape 3 explique pourquoi une app comme
celle-ci n'en a généralement besoin d'aucun.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez avec le `VERSION` de ce pack. Signalez tout écart en une phrase,
puis continuez. Le texte juridique vieillit plus vite que le code — c'est la
skill où une copie périmée compte le plus.

## Étape 1 — l'inventaire d'abord

Vous ne pouvez pas écrire une politique de confidentialité véridique à partir
de votre imagination. **Listez ce que l'app stocke réellement sur les
personnes**, table par table, avant d'écrire un mot de politique :

- comptes : e-mail, nom, horodatages de connexion
- commandes : nom de l'acheteur, adresse, montant — venant de Digistore24
- payloads IPN bruts : tout ce que Digistore24 a envoyé, y compris les
  coordonnées de l'acheteur
- octrois d'accès et toute note de l'opérateur à leur sujet
- lignes du grand livre et leurs notes
- logs, et combien de temps ils sont conservés

Pour chacun : **pourquoi** vous le détenez, **combien de temps** et **qui
d'autre le voit** — Digistore24, le fournisseur de messagerie, l'hébergeur,
tout fournisseur d'IA. Cette liste est le document à partir duquel tout le
reste est écrit, et elle doit être mise à jour chaque fois qu'une table est
ajoutée.

**Les notes de l'opérateur sont des données personnelles.** Une note que le
support a écrite *au sujet* d'un client est couverte par une demande d'accès
même si l'app ne la lui montre jamais. La cacher dans l'interface est une
décision de ton, pas une exemption.

**Si les membres peuvent se voir ou s'atteindre les uns les autres, deux choses
de plus entrent dans la liste**, et aucune des deux n'est évidente sur un
diagramme de tables :

- **Le contenu que les membres ont écrit eux-mêmes** — un profil, une
  publication dans un espace partagé, un message privé. L'app détient désormais
  du texte qu'une personne a écrit pour qu'une autre le lise. Cela entre dans
  la demande d'accès et doit être atteignable par une demande d'effacement, et
  la manière honnête de faire la seconde est : vider les mots, garder la ligne
  pour que la conversation autour reste lisible, et le dire dans la politique.
  Une réponse à un message supprimé ne doit pas devenir la réponse à rien.
- **⚠️ La participation elle-même peut être une information d'achat.** Une
  liste de membres d'un espace payant est une liste de ceux qui l'ont acheté —
  et pour un produit de santé, de finance ou de coaching, cela s'approche des
  données de catégories particulières (art. 9 du RGPD [GDPR]). Le design sûr
  est de n'avoir aucun listage : pas de liste de membres, pas de compteur de
  membres, pas de « qui est ici ». Quelqu'un devient visible en publiant, ce
  qu'il a choisi de faire. Si vous construisez malgré tout une liste de
  membres, dites-le dans la politique de confidentialité et lisez d'abord
  l'art. 9.

**Et si vous construisez des messages privés, décidez qui peut les lire avant
de les construire, pas après.** « Seulement les deux participants » est une
promesse que le code doit tenir : chaque requête qui lit un message porte l'id
du lecteur lui-même, il n'y a pas de vue administrateur, et une session de
support capable de se connecter en tant que membre n'en obtient pas non plus —
lire le courrier de quelqu'un ne change rien et ne laisse aucune trace, on ne
peut donc pas en rendre compte en le journalisant. Les exceptions qui valent la
peine d'être autorisées sont une demande d'accès de la personne concernée
(traitée à la main, pour une demande qui a été faite) et le signalement d'un
participant lui-même, borné à ce qu'il a choisi d'y joindre.

## Étape 2 — les mentions légales

Sous le § 5 de la DDG allemande (et ses équivalents ailleurs), un site
commercial a besoin de mentions légales atteignables : nom, adresse — **une
vraie, pas une boîte postale** — e-mail, téléphone ou une voie de contact
rapide équivalente, et le cas échéant le numéro de TVA (VAT ID) et
l'inscription au registre du commerce.

Construisez la page et **échouez bruyamment tant qu'elle est vide**. Des
mentions légales à l'état de gabarit qui partent en production sont pires que
rien : ce sont des informations visiblement fausses sur qui vend.

**Les mentions légales voyagent sous deux formes, et aucune ne se transfère à
l'autre :**

- **Sur les pages, un LIEN en pied de page est la réponse complète** — nommé
  « Impressum » / « Imprint », à un clic, sur chaque page. Ne recopiez pas le
  texte des mentions légales dans les pieds de page : « facilement
  reconnaissable, directement atteignable » demande le lien, et une deuxième
  copie intégrée est celle qui dérive.
- **Dans les mails que l'app envoie** — liens de connexion, confirmations,
  avis — **le CONTENU des mentions légales va au bas du mail lui-même.** Un
  mail envoyé dans le cadre de l'activité commerciale est une lettre
  commerciale, et son destinataire ne tient aucun pied de page à cliquer ; pour
  les sociétés immatriculées, les règles sur les lettres (Allemagne : § 35a
  GmbHG, § 125a HGB) demandent les coordonnées du prestataire *dans* la lettre.
  Rendez-le en lignes de texte brut sous le pied du mail — et n'envoyez jamais
  un gabarit par mail : aucun bloc de mentions légales dans le mail tant que
  les vraies mentions légales n'existent pas. Une exception est permise et
  délibérée : un avis purement de sécurité construit pour ne rien porter de
  cliquable reste nu, parce que des mentions légales contiennent des adresses
  web et mail et que les clients les transforment automatiquement en liens.

## Étape 3 — probablement aucun bandeau cookies, et ce n'est pas de la paresse

**Un achat n'a pas besoin de consentement.** Il repose sur l'art. 6(1)(b) du
RGPD — l'exécution d'un contrat — pas sur une permission. Et les choses qu'une
app payante pose normalement sur un appareil — la session, la langue, le thème,
un « ne plus me montrer ceci » que le client a cliqué lui-même — sont couvertes
sans demander : soit strictement nécessaires, soit le résultat direct de
quelqu'un actionnant un interrupteur.

Donc : **n'ajoutez pas de bandeau cookies à une app qui ne pose aucun cookie
non essentiel.** Il demande une permission dont vous n'avez ni besoin ni usage,
et il entraîne les gens à cliquer sans lire celui qui comptera plus tard.

⚠️ **La règle porte sur l'APPAREIL, pas sur les cookies.** Le § 25 de la TDDDG
et ses équivalents couvrent `localStorage`, `sessionStorage`, IndexedDB et les
empreintes d'appareil dans exactement les mêmes termes — donc « nous
n'utilisons pas de cookies, nous utilisons localStorage » n'est pas une
exemption, c'est le même acte avec une autre API. Cela vaut d'être dit parce
que c'est le raccourci qui est pris : une préférence enregistrée sans demander
est correcte quand c'est l'utilisateur qui l'a réglée, et une analytique
enregistrée sans demander est une infraction quel que soit le stockage où elle
atterrit.

**Tenez une liste écrite de tout ce que votre app pose sur un appareil**, avec
une ligne chacun disant pourquoi cela ne nécessite aucun consentement. C'est ce
à partir de quoi la politique de confidentialité est écrite, et c'est ce qui
cesse silencieusement d'être vrai — chaque nouvelle fonctionnalité qui retient
quelque chose est une nouvelle entrée, et personne ne remarque la quatrième.

**Quand quelque chose nécessite vraiment un consentement** — une balise
d'analytique, un mail marketing, un widget tiers intégré — alors :

- déclarez la **finalité**, séparément par finalité
- enregistrez **qui a consenti, à quelle formulation, quand** — et stockez une
  **version de la formulation**, car changer le texte signifie que tout le
  monde a consenti à autre chose. Puis **relisez cette version** : un
  consentement enregistré contre une formulation plus ancienne compte comme
  absent et la question est reposée. Stocker la version et l'ignorer à
  l'endroit où vous vérifiez, c'est l'échec qui a l'air conforme dans la table
  et ne l'est pas dans le produit
- rendez l'enregistrement **en ajout seul (append-only)**. Un retrait est une
  nouvelle ligne, jamais une modification : vous devez être en mesure de
  *démontrer* le consentement (art. 7(1)), et une ligne que vous avez écrasée
  ne démontre rien
- le retrait doit être aussi facile que le fait de le donner

## Étape 4 — la divulgation IA est du droit, pas du texte publicitaire

**Art. 50(1) du règlement européen sur l'IA (EU AI Act), applicable à partir du
2 août 2026 :** un système qui interagit avec des personnes doit indiquer
clairement qu'elles ont affaire à une machine, au plus tard lors de la première
interaction.

Si l'app a un chat, un assistant, une réponse générée — tout ce qui parle à une
personne en tant que machine — **elle le dit, visiblement, dans chaque langue
que l'app parle**. Pas dans les conditions. Là où la conversation a lieu.

Écrivez-le comme une règle plutôt que comme un cas isolé : *tout ce qui, ici,
parle à une personne en tant que machine le dit*. Quelle que soit la
fonctionnalité d'IA ajoutée ensuite, elle en hérite.

**Faites de cette règle une LISTE, et faites en sorte que quelque chose la
parcoure.** Un seul endroit nommant chaque surface où une machine parle à une
personne, et une vérification qui échoue quand une surface de la liste n'a pas
d'avis. Une règle que personne ne peut exécuter est une règle qui tient jusqu'à
la mise en production de la deuxième fonctionnalité d'IA. Deux choses au sujet
de la liste :

- **Ce qui ajoute une surface ajoute sa propre entrée.** Une fonctionnalité
  optionnelle, un plug-in, une deuxième zone de l'app — la liste centrale ne
  peut pas énumérer quelque chose qui n'est pas activé, donc une partie de
  l'app qui apporte sa propre IA apporte sa propre entrée, et la vérification
  parcourt toutes les listes qui existent. Une surface que personne n'a
  enregistrée est exactement celle qui part en production sans avis.
- **Et la vérification lit tous les magasins de TEXTE qui existent, pas
  seulement le central.** Une fonctionnalité qui apporte sa propre surface
  apporte généralement sa propre formulation avec elle, là où cette partie de
  l'app garde ses textes. Une vérification qui parcourt les deux listes mais ne
  lit que le fichier de textes principal signale un avis manquant pour une
  phrase que l'app affiche parfaitement — et une vérification juridique qui
  crie au loup est une vérification que les gens apprennent à ignorer, ce qui
  coûte plus cher que de ne pas l'avoir. Donc : ce que l'app FUSIONNE à
  l'exécution pour décider de ce qu'une personne lit, la vérification le
  fusionne aussi.
- **L'avis n'est pas conditionnel.** Il s'affiche pour tout le monde, à chaque
  fois — jamais derrière une préférence, un rôle, un plan ou un drapeau « on le
  leur a déjà dit une fois ». L'obligation s'attache à l'interaction, et un
  client qui revient interagit de nouveau ; un avis qui peut être désactivé est
  à un clic d'une app qui ne divulgue rien, et ensuite vous ne pouvez pas
  montrer quels clients l'ont jamais vu.

**Une IA qui lit ce que l'utilisateur a PRODUIT doit l'avis plus tôt, et doit
une phrase différente.** Un chat de support est facile : l'interaction est une
question que quelqu'un a choisi de poser, et « ceci est une IA » arrive à
temps. Mais une app qui demande à son utilisateur de livrer son brouillon, sa
réponse, son plan — puis fait lire cela par un modèle — a déjà eu sa première
interaction au moment où il commence à taper. Donc :

- l'avis est lisible **avant qu'il n'écrive**, pas une fois qu'il y a une
  transcription et pas une fois que quelque chose a chargé ;
- il dit **ce qui arrive à ce qu'il écrit**, pas seulement ce qu'est la chose.
  *« Une IA lit ce que vous écrivez ici et y répond »* dit à quelqu'un ce qu'il
  accepte ; *« propulsé par l'IA »* non.

Il en va de même pour ce qui va dans la politique de confidentialité. Une
phrase comme *« rien vous concernant n'est envoyé à l'IA »* est vraie d'un
chatbot de manuel et **fausse** de tout ce qui lit le travail propre de
l'utilisateur — et elle est fausse dans un document juridique, ce qui est le
pire endroit pour se tromper. Si l'app a les deux, délimitez les deux : dites à
laquelle rien n'est envoyé et à laquelle est envoyé ce que l'utilisateur
soumet.

## Étape 5 — les données propres du client

Deux obligations, et les deux sont de l'ingénierie ordinaire une fois que vous
avez l'étape 1 :

**Accès (art. 15).** Une commande ou un bouton produit tout ce qui est détenu
sur une personne. Cherchez par **adresse e-mail, pas par compte** — les
personnes les plus susceptibles de demander sont celles qui n'ont jamais eu de
compte, parce qu'un achat fait sans se connecter laisse leur nom sur une
commande sans id de membre.

Une exception documentée : les payloads bruts de webhooks tiers peuvent porter
les données d'une autre personne et personne n'est entre les deux pour les
caviarder (art. 15(4)). Laissez-les hors de l'export *destiné au client* et
gardez-les dans celui de l'opérateur.

**Suppression (art. 17), et ce qu'elle ne couvre pas.** Supprimer un compte ne
supprime pas tout, et la boîte de dialogue doit le dire :

- **Les commandes restent.** Ce sont des documents comptables soumis à un délai
  de conservation légal. En supprimer un serait l'infraction, pas le remède.
  Coupez plutôt le lien avec le compte.
- **Tout le reste s'en va**, ou est anonymisé.
- **Un abonnement en cours avertit et ne bloque pas.** Refuser l'effacement
  parce que c'est gênant est l'infraction. Mais une facturation qui continue
  chez Digistore24 sans aucun compte derrière mérite une phrase bien sonore —
  et un lien pour résilier.
- L'action de suppression ne prend **aucun id de la requête** : toujours le
  compte de l'appelant lui-même.

## Étape 6 — les conditions et le droit de rétractation

Vendre à des consommateurs dans l'UE implique un droit de rétractation, et pour
le contenu numérique cela implique de demander à l'acheteur d'accepter la
livraison immédiate — sinon le délai court et l'accès a déjà été remis.
Digistore24 gère une grande partie de cela au checkout en tant que marchand de
référence (merchant of record) ; **confirmez ce qu'il couvre pour ce compte
plutôt que de supposer dans un sens ou dans l'autre**, et dites ce que vous
avez confirmé.

## Étape 6a — deux règles qui ne sont pas du droit, et qui coûtent plus cher que la plupart de celles qui en sont

La plateforme par laquelle vous vendez a ses propres critères. En enfreindre un
ne produit aucune erreur, aucun test en échec et aucun client mécontent — cela
produit un produit refusé à l'approbation, ou un compte fermé après des mois de
vente. Rien à l'intérieur de l'app ne peut le sentir, et c'est pourquoi les
deux relèvent d'une vérification plutôt que de la mémoire de quelqu'un.

**1. Ne promettez pas combien de temps l'accès dure.** Un espace membres ne
peut pas être vendu comme à vie, permanent, illimité ou « aussi longtemps que
vous voulez » ; deux ans est le maximum qui peut être offert. La raison est
l'argent plutôt que le ton : une offre qui disparaît au bout de 24 mois peut
obliger le vendeur à rembourser le prix entier. Écrivez plutôt ce qui est VRAI
— *payez une fois, pas d'abonnement* pour un paiement unique, *aussi longtemps
que votre plan tourne* pour un abonnement. Un octroi d'accès sans date de fin
n'est pas une promesse ; c'est l'absence d'un événement qui y met fin, et un
remboursement reste un tel événement.

**Vérifiez-le par RACINES, et seulement là où la phrase nomme l'accès.** Trois
choses qui décident si la vérification sert à quelque chose :

- **Des racines, parce que les langues fléchissent.** La phrase qui a fait de
  ceci une règle était *« Einmal kaufen, dauerhaft nutzen »* — elle ne contient
  aucun des mots interdits tels que les critères les orthographient
  (*dauerhafter*) et elle les contient tous tels qu'ils sont entendus. Une
  liste de mots littérale la laisse passer.
- **Seulement avec un mot d'accès dans la même phrase.** *« Unbegrenzt viele
  Notizen »* est une fonctionnalité et convient ; *« unbegrenzt nutzen »* est
  la promesse refusée. La différence est le substantif. Une simple liste de
  mots ouvre sur un mur de constats dans toute app généreuse sur quelque chose,
  et une vérification qui ouvre sur un mur est une vérification que quelqu'un
  désactive — emportant la règle avec elle.
- **Dites combien vous en avez écartées.** Affichez le nombre de phrases qui
  portaient un mot au sujet d'autre chose que l'accès, pour que personne ne
  lise la coche verte comme « ces mots n'apparaissent pas ici ».

**2. Dites à l'acheteur qui l'a débité.** Le nom sur le relevé bancaire est
celui de la plateforme de paiement, pas celui du vendeur. Une ligne que
personne ne reconnaît ne devient pas un mail au support, elle devient un appel
à la banque — et une rétrofacturation (chargeback) coûte la vente, les frais et
une marque sur le compte.

Deux propriétés, et la seconde est celle qu'on rate :

- **Chaque surface après l'achat, pas seulement la page de remerciement.** Un
  acheteur déjà connecté est généralement envoyé directement vers ce qu'il a
  payé et ne voit jamais la page de remerciement. Quel que soit l'écran sur
  lequel il atterrit VRAIMENT, il le dit aussi.
- **La phrase et l'endroit où elle est affichée sont deux choses distinctes à
  vérifier.** Un texte que personne ne rend est une chaîne dans un fichier ; un
  rendu sans texte montre la clé à un client. Vérifiez les deux, par surface,
  par langue.

## Étape 7 — ce qu'il faut remettre

Laissez derrière vous, dans le dépôt :

1. l'inventaire de l'étape 1, sous forme d'un fichier mis à jour à chaque
   nouvelle table
2. les mentions légales, la politique de confidentialité et les conditions sous
   forme de vraies pages
3. une note datée de ce qui a été vérifié, de ce qui a été décidé et de ce qui
   reste ouvert

Cette dernière est la différence entre « nous y avons réfléchi » et pouvoir le
montrer.

## Étape 8 — la suite

Si ceci a tourné avant le lancement, retournez à **`ds24-golive`** et terminez
l'achat de test. Si l'app est déjà en production, l'étape suivante honnête est
un avocat qui regarde les pages que vous venez d'écrire.
