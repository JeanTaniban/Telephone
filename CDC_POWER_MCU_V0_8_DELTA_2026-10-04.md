# CDC Power/MCU V0.8 — delta normatif du 2026-10-04

**Base :** `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md` V0.7  
**Statut :** décisions validées à intégrer lors de la prochaine consolidation complète du CDC.  
**Priorité :** ce document supersède V0.7 uniquement sur les points explicitement décrits ci-dessous.

---

# 1. ANNEXE1 / ANNEXE2 — commande high-side corrigée

Les rails ANNEXE1/ANNEXE2 restent des sorties `SYS` commutées, OFF par défaut, sans maintien RC.

Le P-MOS ne doit **pas** être commandé directement par une GPIO 3.3 V du RP2040.

Topologie retenue par annexe :

```text
SYS -> source AO3401A
AO3401A drain -> ANNEXE_OUT
AO3401A gate -> 100 kOhm -> SYS
AO3401A gate -> drain AO3400A
AO3400A source -> GND
GPIO GP11/GP12 -> 10 kOhm -> gate AO3400A
AO3400A gate -> 100 kOhm ou 1 MOhm -> GND
aucun condensateur de hold sur ANNEXE1/2
```

Composants : références déjà présentes dans la BOM V0.7 (`AO3401A C15127`, `AO3400A C20917`, résistances Basic). Aucun nouvel Extended.

Comportement :

```text
GPIO HIGH -> annexe ON
GPIO LOW  -> annexe OFF
GPIO Hi-Z / MCU reset -> annexe OFF
```

ANNEXE1 reste affectée à l'ampli haut-parleur. ANNEXE2 reste un SYS commuté générique.

---

# 2. Bouton Power, Android et rôle de GP9

Le bouton utilisateur reste connecté uniquement au RP2040 sur GP7.

## 2.1 Appui court

Un appui court lorsque le Q6A est alimenté doit être transmis au **vrai PWR_ON_KEY** du Q6A via GP8/S8050.

```text
bouton utilisateur -> GP7 MCU
MCU -> GP8 -> S8050 -> Q6A PWR_ON_KEY
```

Android reste maître de l'extinction écran, des wake-locks et de l'entrée effective en suspend-to-RAM. Le MCU ne force plus `mem_sleep=deep` lors d'un appui court.

Le réveil depuis deep utilise également `PWR_ON_KEY`; aucune GPIO WAKE supplémentaire n'est requise.

## 2.2 GP9 devient SHUTDOWN_REQ

L'ancien `SLEEP_REQ` est renommé et redéfini :

```text
GP9 -> Q6A GPIO58 = SHUTDOWN_REQ
```

Électriquement :

```text
Q6A_3V3 -> 100 kOhm -> SHUTDOWN_REQ
SHUTDOWN_REQ -> 10 kOhm série -> GP9
MCU actif : tire LOW
repos : GP9 input / Hi-Z
```

Cette ligne demande une **extinction Android complète et propre**, pas un suspend.

Android/Linux doit surveiller GPIO58 et déclencher sa procédure de shutdown propre lorsque `SHUTDOWN_REQ` est actif.

## 2.3 Appui long utilisateur / recovery

Politique cible :

```text
appui court : transmettre PWR_ON_KEY
appui long : demander d'abord un shutdown propre par SHUTDOWN_REQ
si Android confirme la fin : arrêter EC25 -> MODEM_PWR OFF -> MAIN_PWR OFF
si aucune réponse et bouton toujours maintenu au-delà d'un timeout long : hard cut en dernier recours
```

Les temporisations précises sont firmware et seront caractérisées; aucune coupure brutale ne doit se produire sur une simple perte de heartbeat.

## 2.4 Auto-off longue durée

Le MCU peut demander un shutdown complet après une longue période d'inutilisation uniquement selon une politique explicitement autorisée. Il ne doit pas déduire l'inactivité uniquement de l'absence d'appui utilisateur : musique, appel, navigation ou tâches Android peuvent être actives écran éteint.

Le Q6A doit fournir au MCU un état permettant de distinguer au minimum RUN / extinction en cours / deep attendu / défaut avant qu'un auto-off soit utilisé.

---

# 3. GPIO59 / GP27 — statut Q6A

```text
Q6A GPIO59 -> 100 kOhm série -> GP27
GP27 -> 1 MOhm -> GND
```

La ligne conserve un rôle de `Q6A_STATE / HEARTBEAT` logiciel.

Elle peut signaler RUN et les transitions de shutdown/deep. Sa perte seule ne déclenche jamais immédiatement MAIN_PWR OFF.

Un protocole simple et déterministe de statut/ACK devra être validé avant le firmware final, notamment pour fournir un `SHUTDOWN_READY` exploitable par le MCU.

---

# 4. BQ25628E — séquence d'initialisation obligatoire

Deux cas sont distingués.

## 4.1 Cold boot depuis OFF

MAIN_PWR, MODEM_PWR et ANNEXE1/2 restent OFF tant que le BQ n'est pas configuré et relu.

Séquence :

```text
1. MCU boot
2. attendre ACK I2C du BQ25628E
3. première configuration I2C : traiter explicitement le watchdog
4. programmer les paramètres V1 : VREG, VSYSMIN, ICHG, IINDPM/EN_EXTILIM, TS/JEITA validés
5. port USB-C étant 5 V uniquement : utiliser le seuil VBUS OVP adapté à cette politique (6.3 V) sauf justification contraire
6. relire les registres critiques et vérifier les valeurs
7. vérifier états/faults BQ
8. seulement ensuite autoriser MAIN_PWR
9. démarrer Q6A par PWR_ON_KEY
10. autoriser MODEM_PWR lorsque SYS/VBAT sont dans la plage validée
```

Le watchdog BQ ne doit jamais rester à une configuration implicite : V1 choisit explicitement soit sa désactivation, soit une politique de rafraîchissement documentée. La baseline V1 privilégie sa **désactivation après prise de contrôle I2C** pour éviter un retour silencieux aux valeurs POR.

En cas de NACK I2C, readback incohérent ou fault critique : rester en état sûr, MAIN/MODEM OFF, diagnostic MCU possible.

## 4.2 Reboot bref du MCU pendant RUN

Le hold RC MAIN/MODEM reste utilisé pour empêcher la chute des rails pendant un reboot du RP2040.

Au tout début du boot MCU, détecter/réaffirmer les rails déjà maintenus avant expiration du RC, puis resynchroniser le BQ et relire sa configuration. Ne pas appliquer la séquence cold-boot qui couperait un Q6A déjà en fonctionnement.

---

# 5. Back-power — exigence de validation renforcée

Le risque de back-power reste une gate bloquante avant fabrication :

```text
USB-C/Q6A avec MAIN_PWR OFF
USB host EC25 avec MODEM_PWR OFF
Q6A actif avec MCU Storage OFF
MCU actif avec Q6A OFF
SBS/UART/GPIO lorsque le domaine opposé est éteint
```

Le choix d'ajouter des P-MOS back-to-back sur MAIN_PWR/MODEM_PWR est en revue; ne pas considérer un P-MOS simple comme un reverse blocker bidirectionnel.

Le SN74AVC4T245 doit être câblé de façon à exploiter son isolation/partial-power-down, avec OE/DIR définis et les groupes TXD+RI / RXD+DTR cohérents.

---

# 6. Pinout MCU modifié

Seule la sémantique de GP9 change :

```text
GP7  <- PWR_BUTTON utilisateur
GP8  -> Q6A PWR_ON_KEY
GP9  -> Q6A SHUTDOWN_REQ / GPIO58, actif LOW / Hi-Z au repos
GP27 <- Q6A_STATE / HEARTBEAT / GPIO59
```

Les autres affectations V0.7 restent inchangées.
