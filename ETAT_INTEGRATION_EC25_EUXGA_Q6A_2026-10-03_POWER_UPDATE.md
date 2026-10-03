# EC25-EUXGA / Q6A — mise à jour hardware Power/USB

**Date :** 2026-10-03  
**Portée :** alimentation, USB et supervision MCU du EC25.  
**Baseline Android/RIL :** `ETAT_INTEGRATION_EC25_EUXGA_Q6A_2026-09-25.md` reste valide pour les preuves logicielles.

Ce document **supersède uniquement les paragraphes hardware** qui pouvaient laisser entendre que le modem était principalement alimenté par le VBUS USB du Q6A.

---

# 1. Architecture finale EC25

```text
BQ25628E SYS
   |
   +--> MODEM_PWR high-side
            |
            +--> carrier EC25 BAT   = alimentation principale LTE

Q6A USB2 HOST
   +--> VBUS ----------------------> carrier VBUS
   +--> D+ ------------------------> carrier DP
   +--> D- ------------------------> carrier DN
   +--> GND -----------------------> carrier GND
```

Le câble USB Q6A <-> EC25 **ne traverse pas la carte Power**.

`carrier BAT` et `USB VBUS` ont deux rôles distincts : BAT fournit l'énergie principale du modem ; VBUS sert au domaine USB/présence host.

---

# 2. MODEM_PWR

```text
P-MOS : JMTQ55P02A / C2890429
Driver : AO3400A / C20917 Basic
GPIO : GP29
```

Hold reboot MCU :

```text
GP29 -> 10 kOhm -> gate AO3400A
AO gate -> 1 MOhm -> GND
AO gate -> 1 uF -> GND
P-MOS gate -> 100 kOhm -> SYS
```

Le rail est OFF par défaut mais reste maintenu environ une seconde si le RP2040 reboot brièvement.

Firmware : ne pas activer le modem si SYS est hors fenêtre sûre. Politique de départ : environ `3.35 V <= VSYS <= 4.25 V`. Validation physique obligatoire : BAT carrier doit rester strictement sous 4.30 V, y compris batterie pleine et transitoires.

---

# 3. USB et sommeil

Aucun switch USB_VBUS n'est ajouté en V1.

Pour obtenir le sommeil modem :

```text
AT+QSCLK=1
DTR HIGH
USB host Q6A réellement suspendu
MODEM_PWR conservé
```

Deux gates sont obligatoires :

```text
[ ] EC25 atteint son état basse consommation avec VBUS USB toujours présent mais bus suspendu.
[ ] MODEM_PWR OFF avec VBUS Q6A présent ne provoque pas de back-power problématique du modem.
```

Si la première gate échoue, la liaison directe permet de modifier ultérieurement le fil VBUS ou d'ajouter une coupure externe sans rerouter immédiatement toute la carte Power.

---

# 4. Supervision MCU

```text
GP0   -> EC25 RXD via SN74AVC4T245
GP1   <- EC25 TXD via SN74AVC4T245
GP2   -> EC25 DTR via SN74AVC4T245
GP3   <- EC25 RI via SN74AVC4T245
GP13  -> EC25 PWRKEY via S8050
GP29  -> MODEM_PWR_EN
```

Le `SN74AVC4T245` est entièrement occupé par TX/RX/DTR/RI.

`RESET_N` n'est plus une fonction MCU normale. Ordre de recovery :

```text
1. commande AT propre ;
2. PWRKEY ;
3. MODEM_PWR power-cycle.
```

GP28 reste une réserve :

```text
GP28 -> TP_GP28_3V3
GP28 -> étage S8050 DNP -> TP_EC25_RST_OD -> strap DNP -> RESET_N
```

L'étage optionnel fournit une sortie open-drain utilisable dans le domaine 1.8 V ; ce n'est pas un traducteur push-pull 1.8 V.

---

# 5. Séquence modem

Démarrage :

```text
1. vérifier VSYS et faults BQ ;
2. MODEM_PWR ON ;
3. attendre stabilisation ;
4. PWRKEY ;
5. attendre VIO/UART/USB ;
6. Android/RIL prend la main via USB.
```

Extinction :

```text
1. AT+QPOWD en priorité ou PWRKEY selon état ;
2. attendre confirmation/disparition logique ;
3. MODEM_PWR OFF ;
4. power-cycle forcé uniquement en recovery.
```

---

# 6. USB disponibles Q6A

Répartition système retenue :

```text
USB2 HOST #1 -> EC25
USB2 HOST #2 -> caméra USB future
USB2 HOST #3 -> réserve / futur USB-A amovible
USB3.1 OTG   -> USB-C externe device seulement
```

Ainsi le port USB-C utilisateur reste simple : charge + ADB/EDL/MTP.

---

# 7. Baseline Android

Les preuves du document du 25/09 restent inchangées : EC25-EUXGA détecté, commandes AT accessibles, sept services Radio AIDL1 publiés et état SIM absente observé. Les prochaines validations radio restent SIM Orange/PIN/réseau/appels, puis data/SMS/IMS selon progression.
