# Maker Phone Q6A — références courantes

**Dernière mise à jour : 2026-10-03**

Pour reprendre le projet sans réintroduire une ancienne architecture, lire dans cet ordre :

```text
1. CDC_CARTE_POWER_MCU_V1_2026-10-03_CURRENT.md
2. PROJECT_STATE_MAKER_PHONE_Q6A_2026-10-03_CURRENT.md
3. GUIDE_VALIDATION_PREFAB_V1_2026-10-03.md
4. MISSION_MCU_SUPERVISION_V2_2026-10-03.md
5. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-10-03_POWER_UPDATE.md
6. ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md
7. documents spécialisés J19 / STR / écran
```

Décisions à ne pas inverser sans nouvelle revue :

```text
USB-C externe       = sink/device, charge + ADB/EDL/MTP uniquement
EC25 puissance      = SYS -> MODEM_PWR -> carrier BAT
EC25 USB            = Q6A USB2 host direct, hors PCB Power
Q6A wake/cold boot  = vrai PWR_ON_KEY via S8050
GPIO58              = SLEEP_REQ
GPIO59              = HEARTBEAT
CC1+CC2             = fusionnés sur GP26 ADC
GP28                = réserve / RESET_N open-drain DNP
GP29                = MODEM_PWR_EN
MAIN/MODEM          = OFF par défaut + maintien RC ~1 s
RC validé           = AO3400A + 1 MOhm + 1 uF
USB2 host restants  = caméra future + réserve USB-A amovible
```

Les fichiers datés 2026-09-30 et le `CDC_CARTE_POWER_MCU_V1_2026-10-02_REVIEW_FINAL.md` sont désormais des documents historiques/audit. En cas de contradiction, les documents du 03/10 ci-dessus ont priorité.
