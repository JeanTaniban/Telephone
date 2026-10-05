# Maker Phone Q6A — références courantes

**Dernière mise à jour : 2026-10-05**

Pour reprendre le projet sans réintroduire une ancienne architecture, lire dans cet ordre :

```text
1. CDC_CARTE_POWER_MCU_V1_2026-10-05_CURRENT.md
2. MISSION_CDC_POWER_MCU_V0_9_BOM_2026-10-05.md
3. PROJECT_STATE_MAKER_PHONE_Q6A_2026-10-03_CURRENT.md
4. MISSION_MCU_SUPERVISION_V2_2026-10-03.md
5. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md
6. documents spécialisés J19 / STR / écran
7. CDC/DELTA/MIGRATION Power antérieurs = historique/audit
```

Le `CDC_CARTE_POWER_MCU_V1_2026-10-05_CURRENT.md` est désormais la **référence électrique normative courante V0.9**. Il intègre directement la BOM consolidée et supersède la V0.8/delta V0.8 dès qu'ils sont contradictoires.

Décisions courantes à ne pas inverser sans nouvelle revue :

```text
BATTERIE             = Motorola JK50, ~4850 mAh rated / ~5000 mAh typical
J_BAT PCB            = Molex 5050060812 / LCSC C779875
J_BAT pinout         = 1/8 GND ; 4/5 VBAT ; 2 ID2 ; 3 ID1 ; 6 NTC1 ; 7 NTC2
BAT_ARM              = coupure mécanique BAT+ entre BAT_RAW et BAT via 2 pads THT
protection batterie  = PCM du pack JK50 ; pas de 2e BMS série
charge JK50          = tension firmware finale à valider sur pack réel ; plus limitée par EC25 direct
USB-C externe        = sink/device, charge + ADB/EDL/MTP uniquement
EC25 puissance       = SYS -> MODEM_PWR -> RT6154A -> EC25 BAT ~3.825 V
RT6154A              = C250400 ; 2.2 uH C5832397 ; feedback 1.33 M / 200 k
EC25 USB             = D+/D- directs Q6A ; VBUS commutable par GP6
Q6A power utilisateur= appui court MCU -> vrai PWR_ON_KEY ; Android gère écran/autosuspend
GPIO58               = SHUTDOWN_REQ actif bas depuis GP9
GPIO59               = Q6A_STATE / HEARTBEAT vers GP27
CC1+CC2              = fusionnés sur GP26 ADC
BQ_INT               = supprimé du MCU ; polling I2C
BQ boot              = configuration + readback obligatoires avant cold boot MAIN/MODEM
GP6                  = EC25_USB_VBUS_EN
GP28                 = option NTC2_ADC / RESET_N open-drain DNP ; usages exclusifs
GP29                 = MODEM_PWR_EN
MAIN/MODEM           = OFF par défaut + maintien RC ~1 s
VOL+                 = Q6A J20 pin 29 / GPIO31 direct
VOL-                 = Q6A J20 pin 32 / GPIO30 direct
ANNEXE1              = ampli HP ; AO3401A via AO3400A, OFF par défaut, sans hold
ANNEXE2              = SYS commuté générique ; AO3401A via AO3400A, OFF par défaut, sans hold
SBS                  = GP14/GP15 vers I2C6 Q6A ; pull-up Q6A_3V3 DNP si nécessaires
BOM                   = section 14 du CDC V0.9 ; 12 références Extended uniques baseline
```

Points explicitement à mesurer avant validation physique :

```text
JK50 réelle : dimensions, flex, mating, pinout
NTC1/NTC2 : courbes R/T et choix BQ_TS
budget courant JK50 pire cas
RT6154 : VOUT/ripple/transitoires/température/derating COUT
EC25 : stabilité LTE sur rail ~3.825 V
PWR_ON_KEY / SHUTDOWN_REQ / HEARTBEAT_STATE
hold RC pendant reset MCU et Storage OFF
back-power complet MAIN/MODEM/USB/SBS/UART/GPIO
USB-C présent avec MAIN OFF
SN74AVC4T245 DIR/OE et isolation domaines OFF
TPS610995 shutdown/isolation et courant stockage
```

Les CDC du 30/09, 02/10, 03/10 ainsi que `CDC_POWER_MCU_V0_8_DELTA_2026-10-04.md` et `MIGRATION_PCB_POWER_MCU_V0_7_2026-10-04.md` sont désormais historiques/audit pour Power/MCU.
