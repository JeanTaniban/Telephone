# Maker Phone Q6A — références courantes

**Dernière mise à jour : 2026-10-04**

Pour reprendre le projet sans réintroduire une ancienne architecture, lire dans cet ordre :

```text
1. CDC_POWER_MCU_V0_8_DELTA_2026-10-04.md
2. CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md
3. MIGRATION_PCB_POWER_MCU_V0_7_2026-10-04.md
4. PROJECT_STATE_MAKER_PHONE_Q6A_2026-10-03_CURRENT.md
5. DECISIONS_ARCHITECTURE_POWER_MCU_2026-10-03.md
6. GUIDE_VALIDATION_PREFAB_V1_2026-10-03.md
7. MISSION_MCU_SUPERVISION_V2_2026-10-03.md
8. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-10-03_POWER_UPDATE.md
9. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md
10. documents spécialisés J19 / STR / écran
```

Le `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md` reste la base électrique complète gelée V0.7. `CDC_POWER_MCU_V0_8_DELTA_2026-10-04.md` est le **delta normatif courant** et a priorité sur V0.7 pour les points qu'il modifie. Il sera fusionné dans le prochain CDC complet. `MIGRATION_PCB_POWER_MCU_V0_7_2026-10-04.md` reste utile comme traduction PCB de la base V0.7 mais doit être corrigé selon le delta V0.8 avant implémentation.

Décisions courantes à ne pas inverser sans nouvelle revue :

```text
BATTERIE             = Motorola JK50, ~4850 mAh rated / ~5000 mAh typical
J_BAT PCB            = Molex 5050060812 / LCSC C779875
J_BAT pinout         = 1/8 GND ; 4/5 VBAT ; 2 ID2 ; 3 ID1 ; 6 NTC1 ; 7 NTC2
BAT_ARM              = coupure mécanique BAT+ entre BAT_RAW et BAT via 2 pads THT
protection batterie  = PCM du pack JK50 ; pas de 2e BMS série
fusible BAT          = non obligatoire baseline V1
anti-inversion BAT   = non monté baseline
charge JK50          = VREG initial ~4.15 V tant que l'alimentation EC25 haute batterie n'est pas revalidée
USB-C externe        = sink/device, charge + ADB/EDL/MTP uniquement
EC25 puissance       = SYS -> MODEM_PWR -> carrier BAT, architecture haute batterie encore en revue
EC25 USB             = Q6A USB2 host ; back-power/hard-off à valider
Q6A power utilisateur= appui court MCU -> vrai PWR_ON_KEY ; Android gère écran et autosuspend
GPIO58               = SHUTDOWN_REQ actif bas depuis GP9, plus de SLEEP_REQ forcé
GPIO59               = Q6A_STATE / HEARTBEAT vers GP27
CC1+CC2              = fusionnés sur GP26 ADC
BQ_INT               = supprimé du MCU ; polling I2C
BQ boot              = configuration + readback obligatoires avant cold boot MAIN/MODEM
GP6                  = réserve
GP28                 = réserve / RESET_N open-drain DNP
GP29                 = MODEM_PWR_EN
MAIN/MODEM           = OFF par défaut + maintien RC ~1 s
RC                   = AO3400A + 1 MOhm + 1 uF
VOL+                 = Q6A J20 pin 29 / GPIO31 direct
VOL-                 = Q6A J20 pin 32 / GPIO30 direct
ANNEXE1              = ampli HP ; AO3401A commandé via AO3400A, OFF par défaut, sans hold
ANNEXE2              = SYS commuté générique ; AO3401A commandé via AO3400A, OFF par défaut, sans hold
SBS                   = GP14/GP15 vers I2C6 Q6A ; pull-up Q6A_3V3 DNP si nécessaires
connectique interne  = J_BAT Molex uniquement ; autres faisceaux sur pads THT
```

Points explicitement à **mesurer** avant de considérer la carte physiquement validée :

```text
JK50 réelle : dimensions, flex, mating, pinout au multimètre
NTC1/NTC2 JK50 : caractérisation R/T et choix réseau BQ_TS
JK50 : tenue au pire cas Q6A + EC25 LTE + annexes sans déclenchement pack
PWR_ON_KEY avec alimentation J19 finale
SHUTDOWN_REQ + statut/ACK Q6A
hold RC pendant vrai reset MCU et Storage OFF
back-power complet : MAIN/MODEM/USB/SBS/UART/GPIO/MCU
EC25 BAT <4.30 V tant que le chemin BAT direct reste utilisé
USB host EC25 en deep et comportement VBUS avec MODEM_PWR OFF
absence de back-power Q6A via USB-C lorsque MAIN_PWR OFF
pull-up SBS réels sur le Q6A
TPS610995 shutdown/isolation et courant de stockage
```

Les documents du 30/09 et le `CDC_CARTE_POWER_MCU_V1_2026-10-02_REVIEW_FINAL.md` sont historiques/audit. Toute contradiction avec le delta V0.8 courant doit être traitée comme historique jusqu'à consolidation complète.