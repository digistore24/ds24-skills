---
name: ds24-compliance
language: fr
description: À utiliser quand une app qui encaisse de l'argent auprès de clients de l'UE doit mettre son volet juridique en ordre — mentions légales, politique de confidentialité, conditions, la divulgation exigée par le règlement européen sur l'IA (EU AI Act) pour tout ce qui parle aux gens en tant que machine, la question de savoir si un bandeau cookies est nécessaire ou non, les registres de consentement, et le droit du client à ses données et à leur suppression. À utiliser dès que l'utilisateur parle de RGPD (GDPR), de bandeau cookies, de mentions légales, d'AI Act, de suppression de compte, ou demande ce qu'il doit afficher avant de vendre à de vrais clients.
---

> **Français** · Original en anglais — [`SKILL.md`](SKILL.md) · [Español](SKILL.es.md)

# Ce que l'UE demande à une app payante

Ceci prépare le terrain ; ce n'est pas un conseil juridique. La skill met en
ordre ce qui est évident et désigne les points qu'un avocat devrait examiner.
Dites-le une fois, au début, puis rendez-vous utile.

Le déclencheur n'est pas la taille de l'app : c'est le fait d'**encaisser de
l'argent auprès de personnes situées dans l'UE** — et c'est précisément à cela
que sert une intégration Digistore24.

## Étape 0 — qu'y a-t-il déjà ?

Regardez avant d'écrire quoi que ce soit. Cherchez dans le projet des mentions
légales, une politique de confidentialité, des conditions, un bandeau cookies,
une table de consentements.

**Ce qui existe, ne le remplacez pas.** Les pages juridiques sont souvent la
seule partie d'une app qu'un avocat a déjà relue, et une politique de
confidentialité réécrite qui se lit mieux mais dit autre chose est pire qu'une
rédaction maladroite mais exacte. Lisez ce qui est là, confrontez-le aux étapes
ci-dessous et signalez les écarts — en changer la formulation est une décision
de l'utilisateur, pas la vôtre.

Une seule chose mérite d'être remise en question ouvertement : un **bandeau
cookies déjà en place**. L'étape 3 explique pourquoi une app de ce type n'en a
généralement pas besoin du tout.

## Étape 0a — cette copie du Skill Pack est-elle à jour ?

Récupérez `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
et comparez avec le `VERSION` de ce pack. En cas d'écart, signalez-le en une
phrase, puis poursuivez. Un texte juridique vieillit plus vite que du code : de
toutes les skills, c'est ici qu'une copie périmée pèse le plus.

## Étape 1 — l'inventaire d'abord

Une politique de confidentialité véridique ne s'écrit pas de tête. Avant le
premier mot de politique, **listez ce que l'app stocke réellement sur les
personnes**, table par table :

- comptes : e-mail, nom, horodatages de connexion
- commandes : nom de l'acheteur, adresse, montant — reçus de Digistore24
- payloads IPN bruts : tout ce que Digistore24 a envoyé, coordonnées de
  l'acheteur comprises
- octrois d'accès, et les notes qu'un opérateur y a laissées
- lignes du grand livre et leurs notes
- logs, et leur durée de conservation

Pour chaque entrée : **pourquoi** vous la détenez, **pendant combien de temps**,
et **qui d'autre la voit** — Digistore24, le fournisseur de messagerie,
l'hébergeur, un éventuel fournisseur d'IA. Cette liste est le document dont
tout le reste découle, et elle est à mettre à jour à chaque nouvelle table.

**Les notes d'un opérateur sont des données personnelles.** Une note que le
support a écrite *au sujet* d'un client relève de la demande d'accès, même si
l'app ne la lui montre jamais. La masquer dans l'interface est un choix de ton,
pas une exemption.

**Si les membres peuvent se voir ou se contacter entre eux, deux entrées de
plus rejoignent la liste**, et aucune des deux ne se lit sur un schéma de
tables :

- **Le contenu que les membres ont écrit eux-mêmes** — un profil, une
  publication dans un espace partagé, un message privé. L'app détient désormais
  un texte qu'une personne a écrit pour qu'une autre le lise. Il entre dans la
  demande d'accès et il doit être à portée d'une demande d'effacement ; la
  façon honnête de faire cette seconde chose est de vider les mots et de garder
  la ligne, pour que la conversation autour reste lisible, et de le dire dans
  la politique. Une réponse à un message supprimé ne doit pas devenir une
  réponse à rien.
- **⚠️ La participation elle-même peut être une information d'achat.** La liste
  des membres d'un espace payant est la liste de ceux qui l'ont acheté — et
  pour un produit de santé, de finance ou de coaching, on frôle les données de
  catégories particulières au sens de l'art. 9 du RGPD (GDPR). Le design sûr,
  c'est de ne tenir aucun registre des membres : pas de liste, pas de compteur,
  pas de « qui est là ». On devient visible en publiant, et c'est un geste que
  l'on a choisi. Si vous construisez malgré tout une liste de membres,
  dites-le dans la politique de confidentialité et lisez l'art. 9 d'abord.

**Et si vous construisez une messagerie privée, décidez qui peut lire les
messages avant de la construire, pas après.** « Seuls les deux participants »
est une promesse que le code doit tenir : chaque requête qui lit un message
porte l'id du lecteur lui-même, il n'existe pas de vue administrateur, et une
session de support capable de se connecter comme un membre n'en obtient pas
davantage — lire le courrier de quelqu'un ne modifie rien et ne laisse aucune
trace, donc aucun journal ne peut en rendre compte. Les seules exceptions qui
méritent d'être prévues sont la demande d'accès de la personne concernée
(traitée à la main, en réponse à une demande effectivement formulée) et le
signalement fait par un participant, limité à ce qu'il a choisi d'y joindre.

## Étape 2 — les mentions légales

Le § 5 de la DDG allemande (et ses équivalents ailleurs) impose à un site
commercial des mentions légales accessibles : nom, adresse (**une vraie, pas
une boîte postale**), e-mail, téléphone ou un autre moyen de contact tout aussi
rapide, et, le cas échéant, le numéro de TVA (VAT ID) et l'immatriculation au
registre du commerce.

Construisez la page et **faites-la échouer bruyamment tant qu'elle est vide**.
Des mentions légales factices qui partent en production sont pires que pas de
mentions du tout : c'est une information visiblement fausse sur l'identité du
vendeur.

**Les mentions légales circulent sous deux formes, et l'une ne remplace pas
l'autre :**

- **Sur les pages, un LIEN en pied de page est la réponse complète** —
  intitulé « Impressum » / « Imprint », à un clic, sur chaque page. Ne recopiez
  pas le texte des mentions légales dans les pieds de page : « facilement
  reconnaissable, directement accessible » appelle un lien, et c'est la
  deuxième copie, intégrée en dur, qui finit par dériver.
- **Dans les mails que l'app envoie** — liens de connexion, confirmations,
  notifications — **c'est le CONTENU des mentions légales qui va au bas du mail
  lui-même.** Un mail envoyé dans le cadre de l'activité est une lettre
  commerciale, et son destinataire n'a aucun pied de page à cliquer ; pour les
  sociétés immatriculées, les règles sur les lettres commerciales (Allemagne :
  § 35a GmbHG, § 125a HGB) exigent les coordonnées du prestataire *dans* la
  lettre. Rendez-les en lignes de texte brut sous le pied du mail — et
  n'envoyez jamais de texte provisoire : pas de bloc de mentions légales dans
  les mails tant que les vraies n'existent pas. Une seule exception, et elle
  est délibérée : une notification purement de sécurité, conçue pour ne rien
  contenir de cliquable, reste nue, parce que des mentions légales contiennent
  des adresses web et mail et que les clients de messagerie les transforment
  d'eux-mêmes en liens.

## Étape 3 — probablement pas de bandeau cookies, et ce n'est pas de la paresse

**Un achat n'a pas besoin de consentement.** Il repose sur l'art. 6(1)(b) du
RGPD — l'exécution d'un contrat — et non sur une permission. Et ce qu'une app
payante dépose normalement sur un appareil — la session, la langue, le thème,
un « ne plus me montrer ceci » que le client a cliqué lui-même — est couvert
sans rien demander : soit strictement nécessaire, soit conséquence directe d'un
réglage que quelqu'un a actionné.

Donc : **n'ajoutez pas de bandeau cookies à une app qui ne dépose aucun cookie
non essentiel.** Il demande une permission dont vous n'avez ni besoin ni usage,
et il habitue les gens à balayer d'un clic celui qui, plus tard, comptera
vraiment.

⚠️ **La règle porte sur l'APPAREIL, pas sur les cookies.** Le § 25 TDDDG et ses
équivalents couvrent `localStorage`, `sessionStorage`, IndexedDB et les
empreintes d'appareil dans exactement les mêmes termes — « nous n'utilisons pas
de cookies, nous utilisons localStorage » n'est donc pas une exemption, c'est
le même acte derrière une autre API. Cela vaut la peine d'être dit, parce que
c'est précisément le raccourci que l'on prend : une préférence enregistrée sans
demander est en règle quand c'est l'utilisateur qui l'a choisie, et des données
d'analyse d'audience enregistrées sans demander sont une infraction, quel que
soit le stockage où elles atterrissent.

**Tenez une liste écrite de tout ce que votre app dépose sur un appareil**,
avec pour chaque entrée une ligne qui dit pourquoi elle n'exige aucun
consentement. C'est de cette liste que s'écrit la politique de confidentialité,
et c'est elle qui cesse d'être vraie sans que personne ne s'en aperçoive :
chaque nouvelle fonctionnalité qui retient quelque chose est une entrée de
plus, et personne ne remarque la quatrième.

**Quand quelque chose exige réellement un consentement** — une balise d'analyse
d'audience, un mail marketing, un widget tiers intégré — alors :

- déclarez la **finalité**, séparément pour chaque finalité
- enregistrez **qui a consenti, à quelle formulation, et quand** — et stockez
  une **version de la formulation**, parce que changer le texte revient à
  faire consentir tout le monde à autre chose. Puis **relisez cette version**
  au moment de vérifier : un consentement enregistré sous une version plus
  ancienne vaut absence de consentement, et la question est reposée. Stocker
  la version et l'ignorer à l'endroit où l'on vérifie, c'est le défaut qui a
  l'air conforme dans la table et ne l'est pas dans le produit
- rendez l'enregistrement **append-only** (ajout seul). Un retrait est une
  nouvelle ligne, jamais une modification : vous devez pouvoir *démontrer* le
  consentement (art. 7(1)), et une ligne écrasée ne démontre rien
- le retrait doit être aussi simple que le consentement lui-même

## Étape 4 — la divulgation IA relève de la loi, pas du marketing

**Art. 50(1) du règlement européen sur l'IA (EU AI Act), applicable à partir du
2 août 2026 :** un système qui interagit avec des personnes doit leur faire
clairement savoir qu'elles ont affaire à une machine, au plus tard lors de la
première interaction.

Si l'app comporte un chat, un assistant, une réponse générée — tout ce qui
s'adresse à une personne en tant que machine — **elle le dit, visiblement, dans
chaque langue que l'app parle**. Pas dans les conditions : là où la
conversation a lieu.

Formulez-le comme une règle et non comme un cas particulier : *tout ce qui,
ici, parle à une personne en tant que machine le dit*. La prochaine
fonctionnalité d'IA, quelle qu'elle soit, en hérite.

**Faites de cette règle une LISTE, et chargez quelque chose de la parcourir.**
Un seul endroit qui nomme chaque surface où une machine parle à une personne,
et une vérification qui échoue dès qu'une surface de la liste n'a pas d'avis.
Une règle que personne ne peut exécuter tient jusqu'à la mise en production de
la deuxième fonctionnalité d'IA. Deux choses au sujet de cette liste :

- **Ce qui ajoute une surface ajoute sa propre entrée.** Une fonctionnalité
  optionnelle, un plug-in, une deuxième zone de l'app : la liste centrale ne
  peut pas énumérer ce qui n'est pas activé, donc une partie de l'app qui
  apporte sa propre IA apporte aussi sa propre entrée, et la vérification
  parcourt toutes les listes qui existent. La surface que personne n'a
  enregistrée est exactement celle qui part en production sans avis.
- **Et la vérification lit tous les magasins de TEXTES qui existent, pas
  seulement le magasin central.** Une fonctionnalité qui apporte sa propre
  surface apporte généralement sa propre formulation, à l'endroit où cette
  partie de l'app range ses textes. Une vérification qui parcourt les deux
  listes mais ne lit que le fichier de textes principal signale un avis
  manquant pour une phrase que l'app affiche pourtant correctement — et une
  vérification juridique qui crie au loup est une vérification que les gens
  apprennent à ignorer, ce qui coûte plus cher que de ne pas en avoir. Donc :
  tout ce que l'app FUSIONNE à l'exécution pour décider de ce qu'une personne
  lit, la vérification le fusionne aussi.
- **L'avis n'est pas conditionnel.** Il s'affiche pour tout le monde, à chaque
  fois — jamais derrière une préférence, un rôle, un plan ou un drapeau « on le
  leur a déjà dit ». L'obligation est attachée à l'interaction, et un client
  qui revient interagit de nouveau ; un avis qu'on peut désactiver est à un
  clic d'une app qui ne divulgue rien, et après coup vous ne pouvez plus
  montrer quels clients l'ont vu un jour.

**Une IA qui lit ce que l'utilisateur a PRODUIT est redevable de l'avis plus
tôt, et d'une phrase différente.** Un chat de support est le cas facile :
l'interaction est une question que quelqu'un a choisi de poser, et « ceci est
une IA » arrive à temps. Mais une app qui demande à son utilisateur de lui
confier son brouillon, sa réponse, son plan — puis les fait lire par un modèle
— a eu sa première interaction dès l'instant où il a commencé à taper. Donc :

- l'avis est lisible **avant qu'il n'écrive**, pas une fois qu'il existe une
  transcription, ni une fois que quelque chose a fini de charger ;
- il dit **ce qu'il advient de ce qu'il écrit**, pas seulement ce qu'est la
  chose. *« Une IA lit ce que vous écrivez ici et y répond »* dit à quelqu'un
  ce qu'il accepte ; *« propulsé par l'IA »* ne le dit pas.

La même exigence vaut pour la politique de confidentialité. Une phrase comme
*« rien vous concernant n'est envoyé à l'IA »* est vraie d'un chatbot qui ne
répond qu'à partir d'un manuel et **fausse** de tout ce qui lit le travail de
l'utilisateur lui-même — et fausse dans un document juridique, c'est-à-dire au
pire endroit pour se tromper. Si l'app a les deux, délimitez chacun des deux :
dites lequel ne reçoit rien et lequel reçoit ce que l'utilisateur soumet.

## Étape 5 — les propres données du client

Deux obligations, et l'une comme l'autre relève de l'ingénierie ordinaire une
fois l'étape 1 faite :

**Accès (art. 15).** Une commande ou un bouton produit tout ce qui est détenu
sur une personne. Cherchez par **adresse e-mail, pas par compte** : les
personnes les plus susceptibles de demander sont celles qui n'ont jamais eu de
compte, parce qu'un achat effectué sans se connecter laisse leur nom sur une
commande sans id de membre.

Une exception, documentée : les payloads bruts des webhooks tiers peuvent
contenir les données d'une autre personne, et personne ne se trouve entre les
deux pour les caviarder (art. 15(4)). Laissez-les hors de l'export *destiné au
client* et gardez-les dans celui de l'opérateur.

**Suppression (art. 17), et ce qu'elle ne couvre pas.** Supprimer un compte ne
supprime pas tout, et la boîte de dialogue doit le dire :

- **Les commandes restent.** Ce sont des pièces comptables soumises à une
  obligation légale de conservation. En supprimer une serait l'infraction, pas
  le remède. Rompez plutôt le lien avec le compte.
- **Tout le reste disparaît**, ou est anonymisé.
- **Un abonnement en cours déclenche un avertissement, pas un blocage.** Refuser
  l'effacement parce qu'il dérange est l'infraction. Mais une facturation qui
  continue chez Digistore24 sans plus aucun compte derrière mérite une phrase
  bien visible — et un lien pour résilier.
- L'action de suppression ne prend **aucun id dans la requête** : toujours le
  compte de l'appelant, et lui seul.

## Étape 6 — les conditions et le droit de rétractation

Vendre à des consommateurs dans l'UE implique un droit de rétractation, et pour
du contenu numérique, cela implique de demander à l'acheteur d'accepter la
livraison immédiate — sinon le délai court alors que l'accès a déjà été remis.
Digistore24, en tant que marchand officiel (merchant of record), prend en
charge une bonne partie de cela au checkout ; **confirmez ce qu'il couvre pour
ce compte plutôt que de le supposer, dans un sens ou dans l'autre**, et dites
ce que vous avez confirmé.

## Étape 6a — deux règles qui ne sont pas des lois, et qui coûtent plus cher que la plupart de celles qui en sont

La plateforme par laquelle vous vendez a ses propres critères. En enfreindre un
ne produit ni erreur, ni test en échec, ni client mécontent — cela produit un
produit refusé à l'approbation, ou un compte fermé après des mois de ventes.
Rien à l'intérieur de l'app ne peut le ressentir, et c'est pour cela que ces
deux règles relèvent d'une vérification et non de la mémoire de quelqu'un.

**1. Ne promettez pas la durée de l'accès.** Un espace membres ne peut pas être
vendu comme étant à vie, permanent, illimité ou « pour aussi longtemps
que vous voulez » ; deux ans est le maximum qui puisse être proposé. La raison est
financière, pas une question de ton : une offre qui disparaît au bout de
24 mois peut obliger le vendeur à rembourser le prix intégral. Écrivez plutôt
ce qui est VRAI — *payez une fois, sans abonnement* pour un paiement unique,
*tant que votre plan est actif* pour un abonnement. Un octroi d'accès sans date
de fin n'est pas une promesse ; c'est l'absence d'un événement qui y mette fin,
et un remboursement reste un tel événement.

**Vérifiez-le sur des RADICAUX, et seulement là où la phrase parle d'accès.**
Trois choses décident si cette vérification sert à quelque chose :

- **Des radicaux, parce que les langues fléchissent les mots.** La phrase qui a
  fait de ceci une règle était *« Einmal kaufen, dauerhaft nutzen »* — elle ne
  contient aucun des mots interdits tels que les critères les épellent
  (*dauerhafter*), et elle les contient tous dans le sens où ils sont entendus.
  Une liste de mots littérale la laisse passer.
- **Seulement si un mot d'accès figure dans la même phrase.** *« Unbegrenzt
  viele Notizen »* est une fonctionnalité, et c'est permis ; *« unbegrenzt
  nutzen »* est la promesse refusée. La différence tient au substantif. Une
  simple liste de mots s'ouvre sur un mur de constats dans toute app généreuse
  en quoi que ce soit, et une vérification qui s'ouvre sur un mur est une
  vérification que quelqu'un désactive — en emportant la règle avec elle.
- **Dites combien vous en avez écartées.** Affichez le nombre de phrases qui
  portaient un tel mot à propos d'autre chose que l'accès, pour que personne ne
  lise la coche verte comme « ces mots n'apparaissent pas ici ».

**2. Dites à l'acheteur qui l'a débité.** Le nom qui figure sur le relevé
bancaire est celui de la plateforme de paiement, pas celui du vendeur. Une
ligne que personne ne reconnaît ne devient pas un mail au support, elle devient
un appel à la banque — et une rétrofacturation (chargeback) coûte la vente, les
frais et une marque sur le compte.

Deux propriétés, et c'est la seconde qu'on oublie :

- **Chaque surface après l'achat, pas seulement la page de remerciement.** Un
  acheteur déjà connecté est généralement envoyé droit vers ce qu'il a payé et
  ne voit jamais la page de remerciement. Quel que soit l'écran où il atterrit
  RÉELLEMENT, cet écran le dit aussi.
- **La phrase et l'endroit où elle s'affiche sont deux choses à vérifier
  séparément.** Un texte que rien n'affiche est une chaîne dans un fichier ; un
  affichage sans texte montre la clé au client. Vérifiez les deux, surface par
  surface, langue par langue.

## Étape 7 — ce qu'il faut remettre

Laissez dans le dépôt :

1. l'inventaire de l'étape 1, sous forme de fichier mis à jour à chaque
   nouvelle table
2. les mentions légales, la politique de confidentialité et les conditions, en
   vraies pages
3. une note datée : ce qui a été vérifié, ce qui a été décidé, ce qui reste
   ouvert

Cette dernière fait la différence entre « nous y avons réfléchi » et pouvoir le
montrer.

## Étape 8 — la suite

Si cette skill a tourné avant le lancement, retournez à **`ds24-golive`** et
terminez l'achat de test. Si l'app est déjà en production, la prochaine étape
honnête, c'est un avocat qui relit les pages que vous venez d'écrire.
