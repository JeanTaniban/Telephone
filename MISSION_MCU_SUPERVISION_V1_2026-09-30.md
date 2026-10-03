# HISTORIQUE — mission supersédée pour la carte finale

> **Ne pas utiliser ce document comme câblage final V0.5.**  
> La mission courante est `MISSION_MCU_SUPERVISION_V2_2026-10-03.md`.  
> `GPIO59` n'est plus le WAKE final : il devient `HEARTBEAT`. Le wake/cold boot final utilise le vrai `Q6A PWR_ON_KEY` commandé par le RP2040 via S8050. `GPIO58` reste `SLEEP_REQ`.

Ce document reste utile uniquement comme historique de validation expérimentale GPIO58/GPIO59 et STR.

---

# MISSION — MCU superviseur V1 / suspend-wake Q6A

**Date :** 2026-09-30  
**Projet :** Maker Phone / Radxa Dragon Q6A V1.21  
**Document d'état associé :** `PROJECT_STATE_MCU_DSI_PROTO_2026-09-30.md`  
**CDC Power/MCU courant à l'époque :** `CDC_CARTE_POWER_MCU_V1_2026-09-30.md`  
**État projet courant à l'époque :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-30_CURRENT.md`

---

# 1. Objectif concret

Valider un premier lien matériel minimal entre le RP2040-Tiny de développement et le Q6A permettant au MCU de :

```text
1. demander proprement l'entrée en Suspend-to-RAM du Q6A ;
2. réveiller le Q6A depuis un vrai mem_sleep=deep ;
3. répéter le cycle sans régression Android/USB/radio.
```

Cette mission ne cherche pas encore à implémenter l'extinction physique complète du Q6A.

---

# 2. Périmètre inclus

```text
RP2040-Tiny / MicroPython
USB PC temporaire pour debug MCU
GND commun Pico/Q6A
Q6A header pin 36 / GPIO59 = MCU_WAKE
Q6A header pin 37 / GPIO58 = MCU_SLEEP_REQ
DTBO/pinctrl nécessaire à GPIO58/GPIO59
gpio-keys ou handler équivalent pour wake
handler/service de demande de suspend
logs et validation multi-cycle
```

---

# 3. Hors périmètre

```text
hard power-off Q6A
hard power-cycle
MAIN_PWR de la carte power finale
entrée matérielle PWR via 4N35/MOSFET
fuel gauge
chargeur
boutons définitifs du téléphone
Storage OFF TPS610995
architecture audio finale
pilotage RI/DTR/UART EC25
```

Le modem reste branché en USB au Q6A selon l'état courant, mais son intégration fonctionnelle n'est pas modifiée par cette mission.

La notion d'« audio Minimal autonome Q6A OFF » est obsolète pour la V1 : l'état courant prévoit que les appels/audio passent par le Q6A. Cette mission suspend/wake n'a donc aucun codec/audio EC25 à valider.

---

# 4. Câblage figé de la mission

```text
RP2040-Tiny                Radxa Dragon Q6A
------------------------------------------------
GND ---------------------> header pin 34 GND
GPIO MCU_WAKE -----------> header pin 36 / GPIO59
GPIO MCU_SLEEP_REQ ------> header pin 37 / GPIO58
```

Les deux cartes partagent uniquement la masse et les signaux nécessaires. Ne pas relier arbitrairement les rails d'alimentation du Pico alimenté par le PC aux rails du Q6A.

Le 4N35 n'est pas utilisé dans cette mission.

---

# 5. Contraintes techniques

1. Le STR de référence doit rester un vrai `mem_sleep=deep`, pas un `pm_test`.
2. Le GPIO59 est choisi comme candidat wake car le pinctrl SC7280 le mappe au PDC wake IRQ 110 ; cette capacité doit être prouvée sur le BSP/Q6A exact.
3. GPIO58 sert seulement de demande de suspend.
4. Le MCU ne doit pas couper physiquement le Q6A.
5. Une demande MCU ne doit jamais remplacer la séquence de suspend Android/Linux.
6. Le DTBO STR expérimental existant doit être conservé et son SHA vérifié avant/après les essais.
7. Ne pas perdre la possibilité d'accès ADB/power-cycle pendant le développement.
8. Tout flash doit avoir une procédure de rollback.

---

# 6. Briques fonctionnelles et tests

## Brique A — Firmware MCU minimal

Fonctions :

```text
heartbeat GP2
commande manuelle SLEEP_REQ depuis REPL
commande manuelle WAKE depuis REPL
logs horodatés des actions
état GPIO sûr au boot
```

Test avant connexion Q6A : mesurer au multimètre et/ou à l'oscilloscope les niveaux et impulsions.

Critère : aucune sortie ne pulse involontairement pendant reset/boot du Pico.

## Brique B — GPIO58 / SLEEP_REQ vu par Q6A

Fonction : le Q6A détecte l'impulsion GPIO58 sans déclencher encore un suspend profond.

Test : compteur IRQ/log noyau ou interface de test reproductible.

Critère : 20 impulsions MCU -> 20 événements Q6A, aucun événement parasite.

## Brique C — demande de suspend propre

Fonction : un événement GPIO58 déclenche le chemin logiciel de suspend choisi.

Test :

```text
mem_sleep = deep
pm_test = none
MCU_SLEEP_REQ
vérifier entrée effective en STR
```

Critère : suspend réel, pas seulement extinction écran.

## Brique D — GPIO59 / wake source

Fonction : GPIO59 réveille le Q6A depuis le STR.

Test : lancer STR puis générer l'impulsion MCU_WAKE.

Critère : reprise Android correcte et cause de wake attribuable à GPIO59/chemin créé.

## Brique E — intégration multi-cycle

Test minimal :

```text
10 cycles consécutifs
MCU_SLEEP_REQ -> deep -> MCU_WAKE -> reprise
```

À chaque cycle vérifier :

```text
ADB si volontairement activé
Wi-Fi état/reprise
absence de failed_freeze
absence de -EBUSY USB nouveau
logs wake/suspend
processus Radio EC25 si le prototype modem est actif
```

Critère : 10/10 cycles réussis sans intervention manuelle autre que la commande MCU de test.

---

# 7. Critères de validation de mission

Mission validée seulement si :

```text
[ ] câblage 3 fils confirmé
[ ] niveaux électriques confirmés
[ ] firmware MCU ne produit aucune impulsion parasite au boot
[ ] GPIO58 reçu fiablement par le Q6A
[ ] GPIO58 peut demander un vrai STR deep
[ ] GPIO59 réveille depuis deep
[ ] wake source identifiable
[ ] reprise Android fonctionnelle
[ ] 10 cycles consécutifs réussis
[ ] pas de régression du DTBO STR existant
[ ] rollback documenté
[ ] logs et mesures archivés dans le dépôt
```

---

# 8. Risques de régression

```text
mauvais pinmux GPIO58/GPIO59
GPIO59 non propagé au PDC dans le BSP exact
pull-up/down entraînant un événement parasite au boot
Pico reset provoquant un faux wake
SystemSuspend toujours bloqué par la politique USB OTG
interaction avec le DTBO STR existant
ADB coupé pendant deep rendant le debug ambigu
modem EC25 ou autre USB host modifiant la politique PM
```

En cas d'échec, identifier d'abord si la cause est :

```text
signal électrique
pinctrl/DTBO
IRQ/wakeup
service Android
SystemSuspend
USB/driver
reprise post-suspend
```

Ne pas modifier plusieurs couches simultanément avant d'avoir isolé la cause.

---

# 9. Relation avec la carte Power finale

La mission GPIO-only est un prototype de validation. La carte finale était alors définie par `CDC_CARTE_POWER_MCU_V1_2026-09-30.md` et ajoutait notamment :

```text
MAIN_PWR physique Q6A
ANNEXE1/ANNEXE2
BQ25628E
TPS610995 + STORAGE_SW
EC25 UART/RI/DTR/PWK/RST
```

Cette section est historique. Pour la carte finale V0.5, utiliser `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md` et `MISSION_MCU_SUPERVISION_V2_2026-10-03.md`.

---

# 10. Livrables attendus à la fin

```text
firmware MicroPython du RP2040-Tiny
DTBO/source DTS ou patch exact
handler/service Q6A exact
script de test répétable
logs de 10 cycles
mesure des niveaux GPIO
SHA des images flashées
commit(s) Git
bilan PASS/FAIL factuel
```
