# PROJECT STATE — 2026-09-30 — SUPERSÉDÉ

Ce fichier n'est plus l'état courant.

Utiliser désormais :

```text
PROJECT_STATE_MAKER_PHONE_Q6A_2026-10-03_CURRENT.md
CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md
MISSION_MCU_SUPERVISION_V2_2026-10-03.md
```

Les décisions suivantes du 30/09 sont notamment devenues obsolètes :

```text
- EC25 alimenté principalement par VBUS USB du Q6A ;
- absence de MODEM_PWR ;
- GPIO59 utilisé comme MCU_WAKE final ;
- GPIO58/GPIO59 considérés comme paire sleep/wake finale ;
- EC25 RESET_N dédié sur une GPIO MCU ;
- CC1 et CC2 lus sur deux ADC séparés ;
- MAIN_PWR sans maintien RC de reboot MCU.
```

Nouvelle architecture de référence :

```text
SYS -> MAIN_PWR -> Q6A J19
SYS -> MODEM_PWR -> EC25 carrier BAT
Q6A USB2 host -> EC25 USB uniquement
GP8 -> vrai Q6A PWR_ON_KEY
GP9 -> Q6A SLEEP_REQ / GPIO58
Q6A GPIO59 -> GP27 HEARTBEAT
GP26 <- CC1+CC2 fusionnés
GP28 = réserve / RESET_N open-drain DNP
GP29 -> MODEM_PWR
MAIN_PWR et MODEM_PWR OFF par défaut + hold RC ~1 s
USB-C externe = charge + ADB/EDL/MTP uniquement
```

L'ancien contenu reste disponible dans l'historique Git du commit antérieur.