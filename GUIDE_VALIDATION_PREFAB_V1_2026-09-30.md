# GUIDE DE VALIDATION PRE-FABRICATION V1

**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Date :** 2026-09-30  
**Objet :** tests et revues a terminer avant commande de la carte Power / MCU V1  
**Statut :** document de travail de reference pour la decision GO / NO-GO fabrication

## 0. Principe de decision

Le but n'est pas de prouver que le telephone est termine avant la commande. Le but est d'eliminer les erreurs qui deviendraient couteuses ou impossibles a corriger apres fabrication : mauvais niveau electrique, mauvais pinout, mauvais footprint, chemin d'alimentation non viable, GPIO de wake inutilisable, collision mecanique ou erreur de routage critique.

Trois niveaux sont utilises :

- **B0 - Bloquant fabrication** : un FAIL impose une correction du schema, du PCB ou d'un choix physique avant commande.
- **B1 - Fortement recommande** : un FAIL doit etre compris ; la commande reste possible seulement si le contournement est clair et n'impose pas de refaire la PCB.
- **B2 - Opportuniste / non bloquant** : utile pour reduire le risque logiciel, mais peut etre traite apres commande.

La commande est autorisee seulement si tous les B0 sont PASS ou explicitement fermes par une correction de conception.

## 1. Materiel disponible

- alimentation de laboratoire reglable avec limitation de courant et affichage V/A/W ;
- multimetre ;
- oscilloscope 2 voies 40 MHz ;
- generateur de fonctions, utilise uniquement comme source de signal de test et jamais pour alimenter le Q6A ;
- deux sondes de temperature / thermocouples ;
- appareil photo / macro ;
- station de soudage permettant de retirer R7 et ponter R24 ;
- Q6A V1.21, EC25-EUXGA, SIM Orange avec PIN, second telephone pour appels/SMS ;
- trois antennes modem : MAIN, AUX/DIV et GNSS.

## 2. Regles de securite et de mesure

1. Ne pas connecter une vraie LiPo au J19 avant validation du chemin J19 avec l'alimentation de laboratoire.
2. Toute premiere alimentation apres rework se fait avec limitation de courant active.
3. Ne pas alimenter simultanement le Q6A par J19 et son chemin d'alimentation habituel tant que les chemins de back-power ne sont pas caracterises.
4. Ne jamais injecter 3,3 V directement dans les signaux EC25 du domaine VIO si celui-ci est mesure autour de 1,8 V.
5. Toute mesure d'oscilloscope doit partager une masse compatible avec le montage ; ne pas accrocher une pince de masse sur un point flottant.
6. Arreter immediatement un essai en cas d'echauffement local rapide, courant anormalement eleve, odeur, oscillation d'alimentation ou tension hors plage.

---

# A. Q6A - validation de l'alimentation native J19

## A1 - Inspection du chemin J19 avant rework

**Priorite : B0**

**Objectif**  
Confirmer physiquement la configuration du Q6A et reconstruire le chemin electrique J19 -> R24 -> VBATT avant toute soudure.

**Pourquoi**  
Une erreur d'identification de pad ou un composant DNP inattendu peut conduire a injecter la tension 1S sur le mauvais noeud.

**Methode**

- Carte totalement hors tension.
- Inspection macro de R185 a R191, R24, R7 et FB4.
- Controle de continuite : J19- vers GND ; J19+ vers un pad R24 ; autre pad R24 vers domaine VBATT/R7.
- Documenter par photo quel pad de R24 est cote J19 et lequel est cote VBATT.

**Mesures / preuves a consigner**

- etat peuple/DNP de R185, R186, R187, R188, R189, R190, R191, R24, FB4 ;
- resistance J19- vers GND ;
- continuite J19+ -> pad R24 ;
- continuite autre pad R24 -> R7/VBATT.

**Attendu**

- R7 peuplee 0 ohm ;
- R24, R190, R191, FB4 DNP ;
- R185..R189 DNP ;
- topologie conforme au schema Radxa.

**PASS**  
Le chemin J19 est identifie sans ambiguite et aucune population inattendue ne modifie la topologie.

**FAIL / impact**  
**Bloquant fabrication.** Revoir le schema Q6A reel et la strategie J19 avant tout rework ou dimensionnement de la carte Power.

## A2 - Rework minimal R7 / R24

**Priorite : B0**

**Objectif**  
Isoler le bypass batteryless et fermer le chemin J19 vers VBATT.

**Methode**

- Retirer R7.
- Pour le premier bring-up, ponter R24 par un fil tres court / 0 ohm si le shunt 2 mOhm final n'est pas disponible.
- Inspecter la soudure en macro et verifier absence de court-circuit vers les pads voisins.

**Mesures**

- continuite J19+ -> VBATT apres pont ;
- absence de continuite anormale J19+ -> GND ;
- resistance du pont aussi faible que permet la mesure du multimetre.

**Attendu / PASS**  
R7 est reellement ouverte et J19+ rejoint le domaine VBATT via R24 sans court-circuit.

**FAIL / impact**  
**Bloquant.** Ne pas alimenter J19.

## A3 - Premier boot Q6A sur J19 a 3,8 V

**Priorite : B0**

**Objectif**  
Demontrer que le Q6A demarre et reste stable en etant alimente comme un systeme 1S via J19.

**Pourquoi**  
Toute la carte Power V1 repose sur MAIN_PWR -> J19. Si cette hypothese est fausse, le PCB est structurellement incorrect.

**Methode**

- Utiliser uniquement l'alimentation de laboratoire sur J19.
- Tension initiale : **3,8 V**.
- Limitation de courant active. Utiliser comme reference le courant de boot mesure avec le chemin d'alimentation Q6A deja fonctionnel ; regler une marge raisonnable au-dessus de ce pic. Si cette reference n'est pas disponible, commencer conservateur et augmenter seulement si l'alimentation est clairement en limitation sans echauffement anormal.
- Demarrer Android et conserver le systeme plusieurs minutes.
- Une sonde thermique sur le point chaud Q6A, l'autre en reference ambiante.

**Mesures**

- courant moyen et pic visible sur l'alimentation ;
- puissance ;
- temperature Q6A et temperature ambiante ;
- comportement boot / reset ;
- sorties de `dumpsys battery` et `/sys/class/power_supply` pour information.

**Attendu**

- boot Android complet ;
- pas de reset spontanne ;
- pas de chauffe anormale nouvelle ;
- courant stable et coherent avec le fonctionnement precedent.

**PASS**  
Android boote sur J19 et reste stable.

**FAIL / impact**  
**Bloquant fabrication.** Ne pas commander la carte Power tant que le chemin 1S/J19 n'est pas compris.

## A4 - Plage de tension 1S : 4,2 / 3,8 / 3,4 / 3,0 V

**Priorite : B0 pour viabilite ; B1 pour caracterisation fine**

**Objectif**  
Verifier que le Q6A couvre la plage utile de la batterie 1S cible.

**Methode**  
A chaque tension, laisser le systeme se stabiliser et mesurer trois etats :

1. Android idle ;
2. charge CPU soutenue ;
3. vrai Suspend-to-RAM `mem_sleep=deep`.

L'ecran final n'etant pas encore disponible, aucun test "ecran allume" n'est exige ici.

**Mesures a consigner pour chaque case**

- tension appliquee ;
- courant moyen ;
- puissance ;
- temperature point chaud Q6A ;
- delta temperature par rapport a l'ambiante ;
- stabilite / reset / brownout ;
- courant STR stabilise.

**Attendu**

- 4,2 V, 3,8 V et 3,4 V : fonctionnement normal sans instabilite ;
- 3,0 V : objectif de caracterisation de bas de plage. Un shutdown propre ou une limite fonctionnelle clairement identifiee est acceptable si elle est compatible avec le seuil de decharge retenu pour la batterie.

**PASS**  
La plage d'utilisation retenue pour la batterie est compatible avec le Q6A et aucune tension normale d'exploitation ne provoque de reset imprevisible.

**FAIL / impact**  
Si l'echec apparait a 3,4 V ou plus : **B0, architecture a revoir**.  
Si l'echec n'apparait qu'autour de 3,0 V : definir un seuil de coupure batterie plus haut ; **pas necessairement bloquant PCB**.

## A5 - USB data pendant alimentation J19

**Priorite : B1**

**Objectif**  
Verifier que le chemin de developpement ADB/OTG reste utilisable quand le Q6A est alimente par J19.

**Methode**

- Q6A alimente par J19 uniquement.
- Connecter le port OTG/data utilise pour ADB.
- Verifier ADB/scrcpy et surveiller simultanement la tension J19 et le courant.

**PASS**  
ADB fonctionne et aucun back-power significatif ou changement inattendu d'alimentation n'est observe.

**FAIL / impact**  
Non bloquant pour le principe J19, mais le chemin USB data devra etre isole/controle avant integration finale.

---

# B. Suspend / wake Q6A <-> RP2040

## B1 - GPIO58 recu comme SLEEP_REQ

**Priorite : B0**

**Objectif**  
Valider physiquement le GPIO58 choisi sur le PCB comme entree de demande de suspend.

**Methode**

- GND commun RP2040/Q6A.
- Q6A pin 37 = GPIO58.
- Generer des impulsions volontaires depuis le RP2040 et les compter cote Q6A avant de les relier au suspend reel.

**Mesures**

- niveau bas/haut au scope ;
- absence d'impulsion parasite au reset du RP2040 ;
- compteur d'evenements cote Q6A.

**PASS**  
Reception deterministe des impulsions et etat sur au boot.

**FAIL / impact**  
**Bloquant** si le probleme vient du pin/pinmux choisi et impose un autre GPIO de PCB.

## B2 - GPIO58 declenche un vrai `mem_sleep=deep`

**Priorite : B0**

**Objectif**  
Prouver que la demande MCU aboutit au vrai STR, pas seulement a l'extinction de l'ecran.

**Methode**

- `mem_sleep = deep`, `pm_test = none`.
- Emission SLEEP_REQ depuis le MCU.
- Confirmer l'entree effective en suspend profond par logs et chute de consommation.

**PASS**  
Le Q6A entre en vrai STR `deep` de facon reproductible.

**FAIL / impact**  
Si le GPIO fonctionne mais que le blocage est logiciel SystemSuspend/USB : **B1**, le PCB peut rester valable.  
Si le chemin GPIO choisi est inutilisable : **B0**.

## B3 - GPIO59 reveille depuis `deep`

**Priorite : B0**

**Objectif**  
Prouver que GPIO59 est une source de wake reelle sur le BSP exact.

**Methode**

- Q6A en `mem_sleep=deep`.
- Q6A pin 36 = GPIO59.
- Impulsion `MCU_WAKE` depuis le RP2040.
- Verifier reprise Android et cause de wake.

**PASS**  
Wake fiable depuis deep, Android operationnel apres resume, sans power-cycle.

**FAIL / impact**  
**Bloquant** si GPIO59 ne peut pas etre utilise comme wake source et qu'une autre broche physique doit etre routee sur la PCB.

## B4 - Repetition 10 cycles

**Priorite : B1**

**Objectif**  
Distinguer une preuve ponctuelle d'un chemin suffisamment robuste pour etre fige.

**PASS**  
10/10 cycles sleep -> deep -> wake sans intervention de recuperation.

**FAIL / impact**  
Si les lignes electriques sont bonnes, classer comme risque logiciel/driver et documenter avant commande.

---

# C. EC25 - validation minimale avant commande

## C1 - SIM Orange reelle et PIN

**Priorite : B1**

**Objectif**  
Depasser l'etat actuellement valide `SIM ABSENT` et prouver que le modem/RIL reconnait une vraie SIM.

**Methode**

- Antennes MAIN et AUX/DIV connectees.
- Inserer la SIM Orange avec PIN actif.
- Lancer le prototype modem actuel, meme manuellement via ADB/root.
- Verifier etat SIM, demande PIN puis passage en etat pret.

**Mesures / preuves**

- `AT+CPIN?` ;
- etat SIM Android ;
- logs RIL ;
- operateur si disponible.

**PASS**  
SIM detectee, PIN accepte, etat SIM pret.

**FAIL / impact**  
**Non bloquant PCB** tant que l'USB et les niveaux electriques du carrier sont valides. Le probleme sera traite dans l'integration Android/modem.

## C2 - Enregistrement reseau Orange

**Priorite : B1**

**Objectif**  
Verifier que le GA et la SIM peuvent s'enregistrer sur le reseau cible.

**Methode**

- Attendre l'enregistrement apres PIN.
- Controler via AT et Android.

**PASS**  
Enregistrement reseau obtenu de facon stable.

**FAIL / impact**  
Non bloquant PCB si le modem communique correctement et si les antennes sont connues bonnes ; documenter firmware/bandes/operateur comme risque logiciel/radio.

## C3 - Appel entrant et sortant etabli

**Priorite : B2 avant commande, mais tres utile**

**Objectif**  
Prouver la fonction voix le plus tot possible.

**Methode**

- Appeler le numero EC25 depuis le second telephone puis effectuer l'inverse.
- Pour cette etape, **appel etabli** suffit ; l'audio duplex final n'est pas un gate PCB.

**PASS**  
Signalisation d'appel entrant/sortant et etablissement de l'appel.

**FAIL / impact**  
Non bloquant PCB. Continuer ensuite avec IMS/VoLTE, firmware/MBN et integration Android.

## C4 - Data / SMS / VoLTE

**Priorite : B2**

Essayer seulement si C1-C3 progressent facilement. Data Android complete, SMS complets et VoLTE ne doivent pas retarder la commande de la carte Power/MCU.

---

# D. EC25 - niveaux electriques vers MCU

## D1 - Mesure VIO

**Priorite : B0**

**Objectif**  
Confirmer la tension de reference du domaine logique EC25 utilise par le SN74AVC4T245.

**Methode**

- EC25 alimente normalement.
- Mesurer VIO par rapport a GND carrier.

**Attendu**  
Environ 1,8 V, dans la plage acceptee par VCCA du SN74AVC4T245.

**PASS**  
VIO stable et compatible 1,2-3,6 V avec le level-shifter choisi.

**FAIL / impact**  
**Bloquant** si la tension est hors plage ou si VIO n'est pas une alimentation exploitable du transceiver.

## D2 - TXD et RI au repos / activite

**Priorite : B0**

**Objectif**  
Confirmer amplitudes et polarites des deux sorties EC25 qui entreront dans le level-shifter.

**Methode**

- Oscilloscope x10.
- Observer TXD au repos puis pendant commandes AT.
- Observer RI au repos et si possible pendant appel entrant/SMS/URC.

**Mesures**

- niveau bas ;
- niveau haut ;
- niveau de repos ;
- presence d'impulsions RI.

**PASS**  
Niveaux contenus dans le domaine VIO et comportement logique coherent.

**FAIL / impact**  
**Bloquant** uniquement si le niveau electrique impose de changer le level-shifter/topologie. Un RI non declenche faute de scenario logiciel reste B1/B2.

## D3 - Verification du SN74AVC4T245 et du back-power

**Priorite : B0 revue schema**

**Objectif**  
S'assurer que le transceiver reste sur lorsque l'un des domaines est eteint.

**Reference fabricant**  
Le SN74AVC4T245 supporte le partial power-down `Ioff` et place ses ports en haute impedance si l'une des alimentations VCC est a 0 V. Les broches de controle sont referencees a VCCA.

**Controle schema**

- VCCA = EC25 VIO ; VCCB = MCU 3V3 ;
- directions conformes : EC25->MCU pour TXD/RI, MCU->EC25 pour RXD/DTR ;
- OE/DIR ne doivent pas flotter ;
- verifier le comportement des resistances de commande quand VCCA disparait.

**PASS**  
Aucun chemin evident ne peut alimenter le MCU eteint ou le modem eteint au travers du transceiver.

**FAIL / impact**  
**Bloquant schema.** Corriger OE/DIR/pulls ou topologie avant commande.

---

# E. Architecture audio simplifiee

Architecture retenue : `Q6A HPH_L/R -> oreillette toujours connectee + entree haute impedance ampli stereo -> 2 HP`. Le MCU coupe l'ampli pendant les appels. Pas de jack utilisateur ; Bluetooth pour l'audio externe.

## E1 - HPH_L/R disponible sans jack physique

**Priorite : B1**

**Objectif**  
Verifier que la sortie casque du codec Q6A peut etre activee sans detection d'une fiche inseree.

**Methode**

- Aucun casque dans le jack existant.
- Forcer la route headphone avec les outils Android/ALSA disponibles (`tinymix`/HAL si necessaire).
- Lire un signal audio connu et mesurer HPH_L/R au scope.

**Mesures**

- signal AC L/R ;
- offset DC au repos ;
- amplitude a plusieurs volumes ;
- clipping eventuel.

**PASS**  
HPH_L/R peut etre utilise comme source analogique sans presence mecanique d'un jack.

**FAIL / impact**  
Non bloquant pour la carte Power, mais l'architecture audio devra revenir vers une autre sortie native Q6A (`EAR/AUX`) ou une carte audio separee.

## E2 - Compatibilite oreillette

**Priorite : B1**

**Objectif**  
Eviter de charger excessivement la sortie casque.

**Methode**

- Relever l'impedance nominale de l'oreillette retenue ; a defaut mesurer sa resistance DC uniquement comme indication.
- Privilegier une oreillette **32 ohms ou plus** si elle doit etre raccordee directement a HPH.
- Tester a volume faible puis nominal en observant forme d'onde et temperature.

**PASS**  
Pas de clipping/anomalie et charge compatible avec la sortie casque Q6A.

**FAIL / impact**  
Ajouter un buffer/ampli oreillette. Non bloquant pour Power/MCU si cette fonction reste sur une carte audio/harness separe.

## E3 - Entree ampli stereo en parallele

**Priorite : B1 revue design**

**Objectif**  
S'assurer que l'ampli des deux HP ne perturbe pas l'oreillette ni la sortie HPH.

**Critere de selection ampli**

- entree audio haute impedance, cible >= 10 kOhm, idealement >= 20 kOhm ;
- broche EN/SHDN ou alimentation commutable ;
- mute/pop acceptable ;
- puissance adaptee aux deux HP ;
- compatibilite avec la tension de l'une des sorties ANNEXE.

**PASS**  
L'ampli peut rester raccorde a HPH_L/R sans charge significative lorsqu'il est ON ou OFF.

**FAIL / impact**  
Changer d'ampli ou ajouter isolation d'entree ; pas de mux audio obligatoire tant qu'un ampli convenable existe.

## E4 - Micro interne Q6A

**Priorite : B2 pour la carte Power**

**Objectif**  
Identifier l'entree micro native Q6A a utiliser et son MIC_BIAS.

**PASS**  
Une entree micro interne exploitable est identifiee sur le Q6A et peut etre routee vers une future petite carte/harness audio.

**FAIL / impact**  
Non bloquant pour la PCB Power/MCU actuelle ; bloquant uniquement pour une future carte audio finale.

---

# F. Revue du schema Power / chargeur

## F1 - BQ25628E pin par pin

**Priorite : B0**

**Objectif**  
Comparer le schema final au datasheet TI et au circuit d'application recommande.

**Points obligatoires**

- pinout du boitier RYK et orientation pin 1 ;
- VBUS, PMID, SW, BTST, SYS, BAT ;
- CE fixe au niveau voulu ;
- ILIM et valeur 5,1 kOhm ;
- TS/TS_BIAS et reseau 5,1 kOhm / 30 kOhm / NTC 10 kOhm ;
- REGN et decouplages ;
- I2C/INT pulls ;
- QON/PG/STAT conformes au choix du CDC ;
- valeurs et tension nominale de chaque condensateur ;
- inductance 1 uH, courant nominal/saturation avec marge.

**PASS**  
Aucune divergence non expliquee avec le datasheet.

**FAIL / impact**  
**Bloquant.** Corriger schema et footprint avant routage final.

## F2 - Courant d'entree au boot

**Priorite : B0 revue**

Le BQ25628E possede une limitation d'entree materielle via ILIM et une gestion IINDPM/VINDPM. Le montage retient 5,1 kOhm pour environ 0,49 A au boot avant prise en main I2C.

**PASS**  
La valeur, le net et le footprint d'ILIM sont corrects, et le firmware est explicitement charge d'augmenter IINDPM seulement apres identification de la source USB-C.

**FAIL / impact**  
**Bloquant**, car une erreur peut rendre le telephone incapable de booter sur USB seul ou surcharger une source.

## F3 - NTC / securite autonome

**Priorite : B0**

**Objectif**  
S'assurer qu'une panne MCU/Android ne supprime pas la protection temperature de charge.

**PASS**  
Le NTC batterie arrive directement sur le BQ via le reseau prevu ; le MCU n'est pas indispensable pour bloquer une charge hors plage.

**FAIL / impact**  
**Bloquant securite.**

---

# G. TPS610995 et Storage OFF

## G1 - Validation datasheet du shutdown

**Priorite : B0**

**Objectif**  
Valider le principe `SYS -> switch -> EN` avant de fabriquer.

**Reference fabricant**  
Le TPS61099x annonce une **true disconnection during shutdown** : lorsque le convertisseur est desactive, la charge est deconnectee de l'entree. Le TPS610995DRVR appartient a cette famille.

**Schema exige**

- VIN -> SYS ;
- EN -> pad STORAGE_SW_2 ;
- pad STORAGE_SW_1 -> SYS ;
- EN -> 100 kOhm -> GND ;
- switch ouvert = EN bas ; switch ferme = EN haut.

**PASS**  
Le symbole, le pinout, le pulldown et le sens du switch sont conformes. Le courant/tension EN restent dans les limites datasheet.

**FAIL / impact**  
**Bloquant.** Corriger avant commande.

## G2 - Analyse de back-power MCU OFF

**Priorite : B0 revue**

**Objectif**  
Identifier tous les chemins qui pourraient relever 3V3_MCU alors que TPS610995 est OFF.

**Chemins a auditer**

- BQ SDA/SCL/INT ;
- GPIO58/59 Q6A ;
- SN74AVC4T245 / EC25 ;
- CC1/CC2 ADC ;
- bouton Power ;
- EN des P-MOS MAIN/ANNEXE ;
- debug/header.

**PASS**  
Chaque chemin est soit haute impedance, soit limite par resistance, soit protege par une fonction Ioff/isolement documentee. Aucun signal externe ne doit maintenir le RP2040 partiellement alimente.

**FAIL / impact**  
**Bloquant** si le chemin impose un composant serie, une resistance ou une modification de topologie sur PCB.

**Note**  
Le courant residuel reel sera mesure apres fabrication ; avant fabrication on valide seulement l'absence de chemin evident.

## G3 - Etat sur des rails commutes quand MCU OFF

**Priorite : B0**

**Objectif**  
Garantir que MAIN_PWR et ANNEXE1/2 retombent dans un etat determine quand le MCU n'est plus alimente.

**PASS**  
Les 100 kOhm base-GND et gate-source maintiennent les P-MOS OFF par defaut.

**FAIL / impact**  
**Bloquant**, risque d'alimentation aleatoire au boot/storage.

---

# H. Etages MAIN_PWR / ANNEXE

## H1 - Orientation et contraintes des P-MOS

**Priorite : B0**

**Objectif**  
Verifier source/drain, diode de corps, VGS et pertes.

**Controles**

- source cote SYS, drain cote charge ;
- gate relevee vers source par 100 kOhm ;
- S8050 tire la gate vers GND ;
- VGS max reste dans la spec ;
- courant et pertes I2R compatibles avec le Q6A ;
- pistes/vias adaptes au courant.

**PASS**  
Le switch coupe dans le bon sens et ne cree pas de chute significative.

**FAIL / impact**  
**Bloquant.**

## H2 - Affectation ampli audio sur ANNEXE

**Priorite : B1**

Ne pas figer ANNEXE1 ou ANNEXE2 tant que l'ampli n'est pas choisi. Les deux sorties doivent rester identiques et utilisables comme alimentation commutable de l'ampli audio.

---

# I. USB-C externe et USB2 modem

## I1 - USB-C 5 V externe

**Priorite : B0**

**Objectif**  
Verifier le pinout du receptacle, les CC, protections et routage USB2 vers le chemin data Q6A prevu.

**Controles**

- A6/B6 reunis sur D+ ; A7/B7 reunis sur D- ;
- CC1 et CC2 chacun 5,1 kOhm vers GND ;
- 10 kOhm serie vers ADC MCU ;
- SRV05-4 sur D+/D-/CC1/CC2 ;
- SMF5.0A sur VBUS ;
- plan GND continu sous D+/D- ;
- pas de branche morte longue.

**PASS**  
Conforme au CDC et sans erreur de pinout/orientation connecteur.

**FAIL / impact**  
**Bloquant.**

## I2 - Q6A USB-A 2.0 host -> EC25

**Priorite : B0**

**Objectif**  
Eliminer toute confusion de port : le modem est relie a l'USB-A 2.0 host du Q6A, pas a son USB-C d'alimentation.

**Controles**

- VBUS host Q6A -> VBUS carrier EC25 ;
- D+ -> DP ; D- -> DN ; GND commun ;
- paire courte, torsadee si fils, ou routage propre si PCB/harness ;
- aucun MODEM_PWR separe dans la V1.

**PASS**  
Mapping exact et documente sur schema/harness.

**FAIL / impact**  
**Bloquant** si le PCB/harness serait fabrique avec le mauvais port ou une inversion D+/D-.

## I3 - Tenue VBUS aux pics LTE

**Priorite : B1**

**Objectif**  
Verifier si possible que le 5 V host du Q6A ne s'effondre pas avec l'EC25 actif.

**Methode**

- scope sur VBUS carrier ;
- scenario reseau/appel/data si disponible ;
- surveiller minimum de tension et reset USB/modem.

**PASS**  
Pas de reset EC25 ni chute manifeste de VBUS pendant l'activite observee.

**FAIL / impact**  
Si echec, il faudra ajouter une alimentation modem separee ou revoir le chemin VBUS. C'est un risque important, mais si le test reseau complet n'est pas disponible avant commande il est explicitement accepte.

---

# J. RP2040-Tiny - footprint et mecanique

## J1 - Footprint module reel

**Priorite : B0**

**Objectif**  
Eviter l'erreur mecanique la plus couteuse : castellations non alignees ou orientation inversee.

**Methode**

- comparer dessin Waveshare officiel et module reel ;
- mesurer largeur, longueur, entraxes et position des castellations ;
- imprimer le PCB/footprint a l'echelle 1:1 ;
- poser physiquement le module sur l'impression.

**PASS**  
Toutes les castellations tombent au centre des pads avec marge de soudure visible ; pin 1/orientation incontestable.

**FAIL / impact**  
**Bloquant fabrication.**

## J2 - Montage face BOTTOM

**Priorite : B0**

**Controles**

- module pres d'une extremite si possible ;
- aucun via/testpoint expose sous des metallisations ;
- keepout conforme au module reel ;
- acces aux filets de soudure ;
- aucune collision batterie/coque/entretoise ;
- pas d'obligation d'assemblage JLC double face pour le module, puisqu'il sera soude manuellement.

**PASS**  
Placement physiquement realisable et inspectable.

**FAIL / impact**  
**Bloquant.**

## J3 - Encombrement global

**Priorite : B0**

**PASS**  
Contour fini **<= 70 x 25 mm**, connecteurs, fils, pads STORAGE_SW et zones de soudure accessibles.

**FAIL / impact**  
**Bloquant mecanique.**

---

# K. Revue PCB 4 couches

## K1 - Stackup et plan de masse

**Priorite : B0**

**Attendu**

- L1 : composants et signaux critiques ;
- L2 : GND continu ;
- L3 : puissance / signaux lents ;
- L4 : signaux / GND ;
- aucune coupure de plan sous USB2 ou boucles critiques.

## K2 - BQ25628E - boucle de commutation

**Priorite : B0**

**Controles**

- inductance et condensateurs PMID/SYS au plus pres ;
- boucle SW minimale ;
- noeud SW compact et eloigne de TS, CC, ADC, I2C ;
- vias thermiques sous pad expose selon recommandations ;
- cuivre puissance suffisant sur SYS/BAT/VBUS.

**FAIL / impact**  
**Bloquant** : risque EMI, instabilite ou surchauffe difficilement corrigeable apres fabrication.

## K3 - USB2

**Priorite : B0**

- D+/D- routes ensemble ;
- longueur courte ;
- references a GND continues ;
- vias limites ;
- ESD proche du connecteur externe ;
- aucune permutation P/N.

## K4 - Retours de courant et puissance

**Priorite : B0**

Verifier que les courants chargeur, batterie, Q6A et annexes ne traversent pas des cols de cuivre ou des retours de masse de signaux sensibles.

---

# L. Revue CAD / fabrication JLCPCB

## L1 - ERC / DRC

**Priorite : B0**

**PASS**  
Zero erreur non expliquee. Toute exception restante doit etre ecrite et justifiee.

## L2 - Revue footprint / polarite / pin 1

**Priorite : B0**

Revoir independamment au minimum : BQ25628E, TPS610995, SN74AVC4T245, P-MOS, S8050, USB-C, TVS/ESD, inductances, RP2040 footprint, Micro-Fit.

## L3 - Testpoints

**Priorite : B1**

Verifier presence et accessibilite :

`VBUS_USB_C`, `BAT`, `SYS`, `3V6_MCU`, `3V3_MCU`, `MCU_EN`, `GND`, `BQ_SDA`, `BQ_SCL`, `BQ_INT`, `BQ_TS`, `BQ_ILIM`, `MAIN_PWR_GATE`, `MAIN_PWR_OUT`, `ANNEXE1_OUT`, `ANNEXE2_OUT`, `CC1`, `CC2`, `EC25_VIO`, `TXD`, `RI`, `PWK`, `RST`, `Q6A_GPIO58`, `Q6A_GPIO59`.

## L4 - BOM / stock / classe JLC

**Priorite : B1 juste avant commande**

- revalider chaque LCSC ;
- confirmer boitier exact ;
- confirmer Basic/Promotional/Extended ;
- ne pas substituer un composant fonctionnel critique uniquement pour gagner le cout Extended.

## L5 - Gerber / Pick&Place / BOM

**Priorite : B0**

Inspection visuelle finale :

- dimensions ;
- orientation ;
- composants sur bonne face ;
- trous THT presents ;
- pas de cuivre hors contour ;
- pas de designator ou texte sur pads ;
- centroides et rotations plausibles ;
- composants DNP correctement exclus ;
- RP2040-Tiny non commande en assemblage JLC s'il reste montage manuel.

---

# M. Risques explicitement acceptes apres commande

Les points suivants **ne bloquent pas** la commande de la carte Power/MCU si les B0 ci-dessus sont valides :

- integration RIL autonome au boot sans ADB/root ;
- reconstruction definitive des images Android/vendor/boot ;
- QMI/data Android complet ;
- SMS complet ;
- VoLTE / IMS Orange ;
- audio duplex telephonique final ;
- choix final de l'ampli stereo et des deux HP ;
- choix final du micro interne ;
- comportement audio complet de l'oreillette pendant media ;
- mesure exhaustive des transitoires LTE ;
- charge reelle 0,5 / 1 / 1,5 / 2 A et thermique BQ sur la PCB fabriquee ;
- mesure du courant de stockage reel avec `STORAGE_SW` ouvert ;
- validation complete USB plug/unplug avec batterie et power-path reels.

Ces points devront etre repris dans le guide de bring-up post-fabrication.

---

# N. Feuille de resultat GO / NO-GO

| ID | Test | Niveau | Resultat | Commentaire / preuve |
|---|---|---|---|---|
| A1 | Inspection J19 | B0 | [ ] PASS [ ] FAIL | |
| A2 | Rework R7/R24 | B0 | [ ] PASS [ ] FAIL | |
| A3 | Boot J19 3,8 V | B0 | [ ] PASS [ ] FAIL | |
| A4 | Plage 1S | B0/B1 | [ ] PASS [ ] FAIL | |
| A5 | USB data + J19 | B1 | [ ] PASS [ ] FAIL | |
| B1 | GPIO58 electrique | B0 | [ ] PASS [ ] FAIL | |
| B2 | GPIO58 -> deep | B0/B1 | [ ] PASS [ ] FAIL | |
| B3 | GPIO59 wake | B0 | [ ] PASS [ ] FAIL | |
| B4 | 10 cycles STR | B1 | [ ] PASS [ ] FAIL | |
| C1 | SIM + PIN | B1 | [ ] PASS [ ] FAIL | |
| C2 | Reseau Orange | B1 | [ ] PASS [ ] FAIL | |
| C3 | Appel etabli | B2 | [ ] PASS [ ] FAIL | |
| D1 | EC25 VIO | B0 | [ ] PASS [ ] FAIL | |
| D2 | EC25 TXD/RI | B0 | [ ] PASS [ ] FAIL | |
| D3 | Level-shifter/back-power | B0 | [ ] PASS [ ] FAIL | |
| E1 | HPH sans jack | B1 | [ ] PASS [ ] FAIL | |
| E2 | Oreillette | B1 | [ ] PASS [ ] FAIL | |
| E3 | Ampli haute impedance | B1 | [ ] PASS [ ] FAIL | |
| F1 | BQ pin par pin | B0 | [ ] PASS [ ] FAIL | |
| F2 | ILIM/IINDPM | B0 | [ ] PASS [ ] FAIL | |
| F3 | NTC autonome | B0 | [ ] PASS [ ] FAIL | |
| G1 | TPS shutdown | B0 | [ ] PASS [ ] FAIL | |
| G2 | Back-power MCU OFF | B0 | [ ] PASS [ ] FAIL | |
| G3 | Rails OFF par defaut | B0 | [ ] PASS [ ] FAIL | |
| H1 | P-MOS MAIN/ANNEXE | B0 | [ ] PASS [ ] FAIL | |
| I1 | USB-C externe | B0 | [ ] PASS [ ] FAIL | |
| I2 | USB-A Q6A -> EC25 | B0 | [ ] PASS [ ] FAIL | |
| I3 | VBUS LTE | B1 | [ ] PASS [ ] FAIL | |
| J1 | Footprint RP2040 | B0 | [ ] PASS [ ] FAIL | |
| J2 | RP2040 BOTTOM | B0 | [ ] PASS [ ] FAIL | |
| J3 | PCB <=70x25 | B0 | [ ] PASS [ ] FAIL | |
| K1-K4 | Revue routage | B0 | [ ] PASS [ ] FAIL | |
| L1-L5 | Revue fabrication | B0/B1 | [ ] PASS [ ] FAIL | |

## Decision finale

**GO FABRICATION** si :

1. tous les B0 sont PASS ;
2. aucun B1 en FAIL n'impose de modifier la PCB ;
3. les risques restants sont identifies comme logiciels, firmware, harness ou composants externes ;
4. Gerber, BOM et Pick&Place correspondent exactement au schema revu.

**NO-GO** si un seul point B0 reste non compris.

---

# O. Documents de reference du depot

- `CDC_CARTE_POWER_MCU_V1_2026-09-30.md`
- `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-30_CURRENT.md`
- `MISSION_Q6A_NATIVE_BATTERY_BRINGUP_2026-09-25.md`
- `MISSION_MCU_SUPERVISION_V1_2026-09-30.md`
- `PROJECT_STATE_MCU_DSI_PROTO_2026-09-30.md`
- `ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md`

References composants critiques : datasheets TI `BQ25628E`, `TPS61099x/TPS610995`, `SN74AVC4T245`, et schema officiel Radxa Dragon Q6A V1.21.
