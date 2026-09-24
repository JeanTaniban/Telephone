# PROJECT STATE — Maker Phone / Radxa Dragon Q6A — premier STR deep validé

**Date :** 2026-09-24  
**Statut :** addendum maître — mécanisme Suspend-to-RAM désormais démontré sur matériel, un cycle `deep` réel réussi ; correction permanente OTG et qualification basse consommation encore à faire.  
**Base Android :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-22_MASTER_ANDROID_BOOT_OK.md`.  
**Base écran :** `PROJECT_STATE_MAKER_PHONE_Q6A_2026-09-23_MASTER_SCREEN_PROTO_READY.md`.  
**Rapport détaillé STR :** `CONCLUSIONS_STR_Q6A_2026-09-24_OTG_CONFIRME.md`.  

**Règle de priorité :** pour tout ce qui concerne le suspend Android, les contrôleurs USB `8c00000.hsusb` / `a600000.ssusb`, le DTBO de test STR, l'AIC8800 et l'analyse SELinux/SystemSuspend, **ce document supersède les conclusions antérieures contradictoires**. Les décisions écran/interposer du master du 2026-09-23 restent inchangées.

---

# 1. Résultat principal

[CONFIRMÉ SUR MATÉRIEL]

Le Q6A sait effectuer un vrai Suspend-to-RAM `deep`.

Un cycle réel a été validé avec :

```text
pm_test                  = none
mem_sleep                = deep
echo mem                 = retour 0
last_suspend_time        = ~28,647 s
réveil                   = bouton Power
wake IRQ                 = 31 / pon_kpdpwr_status
ADB après reprise        = OK
Wi-Fi scan après reprise = OK
Bluetooth état ON        = OK
SELinux                  = Enforcing
```

Les CPU secondaires ont été arrêtés puis relancés pendant le cycle. Le contrôleur DWC3, l'AIC8800 et ADB ont repris après réveil.

**Attention : un seul cycle deep réel est validé.** Les succès précédents issus de `pm_test` ne doivent pas être comptés comme des veilles profondes réelles.

---

# 2. Baseline exacte du test réussi

```text
image       : Q6A-Android15-spi-emmc-boot-20260630-b1
Android     : 15 userdebug
kernel      : 5.4.295-qgki-debug-g646f05065a7e-dirty
slot        : _a
SELinux     : Enforcing
sleep_disabled runtime : N
Wi-Fi       : activé
Bluetooth   : activé
AIC8800     : a69c:8d81
AIC power/control : on
```

DTBO actif pendant les tests :

```text
qcom,ignore-wakeup-src-in-hostmode
sur /soc/hsusb@8c00000
```

SHA-256 :

```text
dtbo_a patché : 25c95174e46f253ece0d5a5c9d0033d6d56a42fd9563d0edb5d2ed046baf6c42
dtbo_a stock  : 07cf1fa07e797cf4905c47a6265e51d4bac72be250b746cc3d3b30da228179d3
```

La comparaison des images confirme :

```text
85 entrées DTBO
une seule entrée modifiée : entrée 40
une seule propriété ajoutée sur le contrôleur interne
```

Le contrôleur OTG externe `a600000.ssusb` n'est pas modifié par ce patch.

---

# 3. Architecture USB pertinente

Deux contrôleurs DWC3 distincts interviennent.

## 3.1 Contrôleur interne

```text
8c00000.hsusb
└── xHCI / hub interne
    └── AIC8800D80 Wi-Fi / Bluetooth
```

Historique : ce contrôleur était le premier bloqueur observé avec le DTBO stock :

```text
Abort PM suspend!! (USB is outside LPM)
-EBUSY
```

Avec le DTBO actuellement chargé :

```text
qcom,ignore-wakeup-src-in-hostmode
```

le contrôleur franchit désormais le callback de system suspend dans la configuration testée, même lorsque :

```text
AIC power/control = on
Wi-Fi = ON
Bluetooth = ON
```

Cela prouve que l'AIC n'empêche pas à lui seul le cycle deep validé avec ce DTBO.

Sa politique runtime-PM reste néanmoins inefficace et doit être étudiée pour la consommation hors STR.

## 3.2 Contrôleur OTG externe

```text
a600000.ssusb
└── DWC3 gadget
    └── diag,adb
        └── câble USB vers le PC
```

C'est le verrou actif démontré dans l'état courant du BSP.

---

# 4. Cause actuelle du blocage STR avec OTG actif

[CONFIRMÉ PAR TEST A/B/A′]

Le contrôleur externe est configuré dans le Device Tree runtime comme :

```text
/soc/ssusb@a600000/dwc3@a600000/dr_mode = "peripheral"
```

Le wrapper n'expose actuellement ni :

```text
extcon
usb-role-switch
```

Dans cette configuration, le rôle périphérique reste logiquement actif.

État observé :

```text
mode = peripheral
UDC = configured
runtime_status = active
wakeup source = active
```

Conséquences :

```text
Android SystemSuspend
    ↓
lecture /sys/power/wakeup_count
    ↓
pm_get_wakeup_count()
    ↓
attend tant que l'événement de réveil OTG reste actif
```

Si l'on contourne Android et lance manuellement :

```bash
echo mem > /sys/power/state
```

le callback de `a600000.ssusb` refuse ensuite le suspend :

```text
USB is outside LPM
-EBUSY
```

---

# 5. Test causal déterminant

Changer uniquement le rôle logique du contrôleur externe :

```text
peripheral → none
```

produit :

```text
UDC détaché
runtime_status → suspended
contrôleur OTG → LPM
SystemSuspend peut progresser
system suspend manuel peut progresser
```

Le test aller-retour :

```text
peripheral → échec -EBUSY
none       → succès
peripheral → retour de l'échec -EBUSY
```

isole le rôle OTG comme facteur causal.

---

# 6. Autosuspend Android

[CONFIRMÉ]

L'absence historique de tentative automatique ne doit plus être attribuée à SELinux.

Baseline :

```text
mHalAutoSuspendModeEnabled = true
SELinux = Enforcing
```

Le thread `android.system.suspend-service` a été observé en attente dans :

```text
pm_get_wakeup_count
wakeup_count_show
sysfs_kf_seq_show
seq_read
vfs_read
```

Lorsque le contrôleur OTG est mis en `none`, Android SystemSuspend lance effectivement une tentative automatiquement, toujours avec SELinux Enforcing.

Conclusion :

```text
SELinux n'est pas un obstacle absolu à SystemSuspend dans cette configuration.
```

Les AVC historiques sur `/sys/class/wakeup/wakeupNN` restent des défauts éventuels à corriger ou qualifier, mais ils ne constituent plus la cause principale retenue pour l'absence d'autosuspend.

---

# 7. Ce que le DTBO interne prouve — et ne prouve pas

Le patch :

```dts
qcom,ignore-wakeup-src-in-hostmode;
```

sur `8c00000.hsusb` permet au chemin host Qualcomm de suspendre activement le contrôleur au lieu de simplement exiger qu'il soit déjà en LPM.

Dans la configuration actuelle :

```text
8c00000.hsusb passe le suspend
```

Le patch est donc fonctionnel pour franchir ce verrou.

Mais sa nécessité absolue n'est pas démontrée.

Il peut également modifier les capacités de réveil USB host : le code associé change la logique `device_init_wakeup()`.

Donc :

```text
DTBO actuel = expérimental fonctionnel
DTBO actuel = pas encore déclaré correction finale optimale
```

À qualifier avant intégration définitive :

```text
wake Wi-Fi
wake Bluetooth
wake USB host si nécessaire
robustesse multi-cycle
consommation
```

---

# 8. AIC8800 — conclusion révisée

Ancienne hypothèse :

```text
AIC power/control=on empêche nécessairement le deep suspend
```

Cette affirmation est maintenant fausse dans la configuration patchée.

Le cycle deep réel a réussi avec :

```text
AIC power/control=on
Wi-Fi ON
Bluetooth ON
```

La bonne conclusion est :

```text
AIC power/control=on n'empêche pas le STR validé avec le DTBO actuel
mais reste probablement mauvais pour l'efficacité runtime-PM lorsque le SoC est éveillé
```

Il faut donc poursuivre l'audit AIC pour l'autonomie, mais ce n'est plus le premier verrou fonctionnel du STR.

---

# 9. État après reprise

Après le cycle deep réel :

```text
ADB = reconnecté
USB = diag,adb
UDC = configured
OTG = peripheral restauré
SELinux = Enforcing
Wi-Fi = activé
scan Wi-Fi = résultats obtenus
Bluetooth = ON
threads AIC = présents, état S
aucun nouveau thread AIC bloqué en D observé
```

Limites :

```text
pas encore de trafic IP Wi-Fi validé après reprise
pas encore d'échange Bluetooth avec un périphérique
pas encore de série de cycles deep
```

---

# 10. Ce qui n'est pas encore un correctif permanent

Le test :

```text
echo none > .../mode
```

est une excellente preuve causale, mais **pas une solution produit**.

Il coupe la liaison USB de debug et dépend d'une action logicielle explicite.

La future correction permanente doit gérer correctement :

```text
présence VBUS
connexion/déconnexion câble
rôle USB peripheral / none
ADB/DIAG lorsqu'un hôte est réellement présent
possibilité d'entrer en suspend lorsqu'aucun hôte ne doit maintenir le lien
```

---

# 11. Point d'architecture à résoudre

Le BSP actuel semble traiter le port externe comme périphérique permanent :

```text
dr_mode = peripheral
pas d'extcon
pas de role-switch
```

Pour le téléphone final, il faut déterminer comment le Q6A doit connaître :

```text
VBUS réellement présent
câble réellement connecté
mode debug volontaire
mode charge uniquement
mode périphérique USB
aucun hôte connecté
```

Le correctif peut se situer dans :

```text
Device Tree
USB role detection
extcon / role-switch
PMIC / VBUS detection
init Android / gadget policy
driver Qualcomm
```

Aucune de ces options n'est encore choisie définitivement.

---

# 12. Tests prioritaires suivants

## P0 — qualification STR répétée

```text
[ ] 10 cycles deep réels minimum
[ ] réveil Power à chaque cycle
[ ] aucun failed_freeze
[ ] aucun -EBUSY USB
[ ] ADB récupéré si volontairement restauré
```

## P0 — autosuspend Android réel

Tester sans `echo mem` manuel :

```text
[ ] OTG correctement au repos
[ ] écran éteint
[ ] pm_test=none
[ ] mem_sleep=deep
[ ] Android déclenche automatiquement le suspend
[ ] réveil Power
```

## P0 — politique OTG sans câble

Qualifier séparément :

```text
câble physiquement absent
câble présent mais gadget non configuré
câble présent + ADB actif
rôle none
```

Ne pas considérer ces états comme équivalents.

## P1 — reprise radios

```text
[ ] reconnexion Wi-Fi réelle
[ ] ping / trafic réseau après resume
[ ] Bluetooth scan
[ ] connexion à un périphérique Bluetooth
[ ] second cycle après utilisation des radios
```

## P1 — consommation

Mesurer :

```text
Android idle écran ON
Android écran OFF mais awake
runtime idle USB
STR deep
```

Le critère projet est l'autonomie, pas seulement le retour 0 de `echo mem`.

## P1 — wake sources

Qualifier :

```text
Power
RTC
Wi-Fi WoWLAN si requis
Bluetooth si requis
future MCU
modem
USB externe selon politique produit
```

---

# 13. Décisions gelées à cette date

| Sujet | État 2026-09-24 |
|---|---|
| Q6A capable de STR deep | **OUI — confirmé pour un cycle réel** |
| Réveil Power | **OUI — confirmé** |
| SELinux doit être permissive | **NON — contredit** |
| OTG `a600000.ssusb` actif bloque actuellement le STR | **OUI — confirmé** |
| `mode=none` permet de franchir ce verrou | **OUI — confirmé** |
| AIC `power/control=on` empêche nécessairement deep | **NON — contredit avec DTBO actuel** |
| Patch `8c00000.hsusb` fonctionne pour le suspend | **OUI dans la configuration testée** |
| Patch DTBO = solution finale optimale | **NON démontré** |
| Gestion OTG permanente résolue | **NON** |
| Autosuspend Android deep autonome qualifié | **NON** |
| Consommation STR mesurée | **NON** |
| 10 cycles deep validés | **NON** |

---

# 14. Règles pour la suite

1. Ne plus présenter SELinux comme cause racine du STR sans nouvelle preuve.
2. Ne plus présenter l'AIC comme verrou indispensable du deep avec le DTBO actuel.
3. Ne pas recopier `qcom,ignore-wakeup-src-in-hostmode` sur `a600000.ssusb` : ce contrôleur est utilisé en mode périphérique, et aucune justification technique n'existe pour ce patch.
4. Ne pas déclarer `mode=none` solution finale ; c'est un test causal et un contournement de développement.
5. Préserver l'accès au power-cycle matériel pendant les tests USB/PM.
6. Conserver le DTBO stock et le DTBO patché avec leurs SHA-256.
7. Distinguer strictement `pm_test` d'un vrai `pm_test=none + mem_sleep=deep`.
8. Valider les fonctionnalités post-resume, pas uniquement l'entrée en suspend.

---

# 15. État global du projet après cette étape

```text
Android Q6A bootable                  : OUI
flash EDL maîtrisé                    : OUI
workflow ADB / scrcpy                 : OUI
interposer écran V1 défini            : OUI, voir master 2026-09-23
STR deep matériellement démontré      : OUI, 1 cycle
wake Power démontré                   : OUI
blocage OTG actuel compris            : OUI
correction OTG permanente             : NON
DTBO STR final qualifié               : NON
runtime-PM AIC optimisé               : NON
consommation deep mesurée             : NON
autosuspend téléphone final qualifié  : NON
```

La prochaine priorité Android/PM est désormais **la gestion propre du rôle/VBUS du contrôleur externe `a600000.ssusb`**, puis la qualification répétée du suspend et de la consommation.

Le projet n'est plus au stade « le Q6A sait-il faire du Suspend-to-RAM ? ». La réponse est désormais oui sur le matériel testé. Le travail porte maintenant sur l'intégration produit et la robustesse.
