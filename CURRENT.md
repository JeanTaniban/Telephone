# Maker Phone Q6A — références courantes

**Dernière mise à jour : 2026-10-04**

Pour reprendre le projet sans réintroduire une ancienne architecture, lire dans cet ordre :

```text
1. CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md
2. MIGRATION_PCB_POWER_MCU_V0_7_2026-10-04.md
3. PROJECT_STATE_MAKER_PHONE_Q6A_2026-10-03_CURRENT.md
4. DECISIONS_ARCHITECTURE_POWER_MCU_2026-10-03.md
5. GUIDE_VALIDATION_PREFAB_V1_2026-10-03.md
6. MISSION_MCU_SUPERVISION_V2_2026-10-03.md
7. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-10-03_POWER_UPDATE.md
8. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md
9. documents spécialisés J19 / STR / écran
```

Le `CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md` est la **référence électrique normative gelée V0.7**. `MIGRATION_PCB_POWER_MCU_V0_7_2026-10-04.md` traduit cette référence en modifications concrètes à appliquer au projet KiCad : schéma, netlist, placement, routage, testpoints, BOM et contrôles.

Décisions à ne pas inverser sans nouvelle revue :

```text
BATTERIE             = Motorola JK50, ~4850 mAh rated / ~5000 mAh typical
J_BAT PCB            = Molex 5050060812 / LCSC C779875
J_BAT pinout         = 1/8 GND ; 4/5 VBAT ; 2 ID2 ; 3 ID1 ; 6 NTC1 ; 7 NTC2
BAT_ARM              = coupure mécanique BAT+ entre BAT_RAW et BAT via 2 pads THT
protection batterie  = PCM du pack JK50 ; pas de 2e BMS série
fusible BAT          = non obligatoire baseline V1
anti-inversion BAT   = non monté baseline
charge JK50          = VREG initial ~4.15 V, pas 4.40 V
USB-C externe        = sink/device, charge + ADB/EDL/MTP uniquement
EC25 puissance       = SYS -> MODEM_PWR -> carrier BAT
EC25 USB             = Q6A USB2 host direct, hors PCB Power
Q6A wake/cold boot   = vrai PWR_ON_KEY via S8050
GPIO58               = SLEEP_REQ
GPIO59               = HEARTBEAT
CC1+CC2              = fusionnés sur GP26 ADC
BQ_INT               = supprimé du MCU ; polling I2C
GP6                  = réserve
GP28                 = réserve / RESET_N open-drain DNP
GP29                 = MODEM_PWR_EN
MAIN/MODEM           = OFF par défaut + maintien RC ~1 s
RC                   = AO3400A + 1 MOhm + 1 uF
VOL+                 = Q6A J20 pin 29 / GPIO31 direct
VOL-                 = Q6A J20 pin 32 / GPIO30 direct
ANNEXE1              = alimentation ampli haut-parleur
ANNEXE2              = SYS commuté générique annexes
SBS                   = GP14/GP15 vers I2C6 Q6A ; pull-up Q6A_3V3 DNP si nécessaires
connectique interne  = J_BAT Molex uniquement ; autres faisceaux sur pads THT
```

Points explicitement à **mesurer** avant de considérer la carte physiquement validée :

```text
JK50 réelle : dimensions, flex, mating, pinout au multimètre
NTC1/NTC2 JK50 : caractérisation R/T et choix réseau BQ_TS
JK50 : tenue au pire cas Q6A + EC25 LTE + annexes sans déclenchement pack
PWR_ON_KEY avec alimentation J19 finale
hold RC pendant vrai reset MCU et Storage OFF
absence de back-power via C_HOLD/GPIO lorsque MCU non alimenté
EC25 BAT <4.30 V et stabilité sous pics LTE combinés
USB host EC25 réellement suspendu avec VBUS présent
absence de back-power Q6A via USB-C lorsque MAIN_PWR OFF
pull-up SBS réels sur le Q6A
TPS610995 shutdown/isolation et courant de stockage
```

Les documents du 30/09 et le `CDC_CARTE_POWER_MCU_V1_2026-10-02_REVIEW_FINAL.md` sont historiques/audit. Les passages des documents du 03/10 encore contradictoires avec le CDC V0.7 doivent eux aussi être considérés comme historiques jusqu'à leur consolidation complète.