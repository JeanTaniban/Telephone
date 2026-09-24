# Q6A — cause du blocage STR confirmée et premier cycle deep réussi

Date hôte : 2026-09-24. Investigation sur le Q6A `8ba07537`, connecté au PC par OTG.

Ce document est la synthèse de la reprise d'enquête. Pour l'état de STR testé ici, il corrige les affirmations devenues obsolètes ou trop catégoriques de `RAPPORT_SUSPEND_TO_RAM_Q6A.md`, `MISSION_SUSPEND_TO_RAM_Q6A.md` et du diagnostic initial. Les journaux historiques restent conservés ; leurs résultats ne sont pas effacés.

## Conclusion

**Avec le DTBO expérimental déjà installé au début de cette enquête, le verrou restant est le contrôleur USB OTG `a600000.ssusb`, maintenu en mode périphérique actif.** Il conserve une source de réveil active qui fait attendre Android dans `pm_get_wakeup_count()`. Une tentative manuelle contourne cette attente, mais rencontre ensuite le refus du callback de suspension de ce contrôleur : `USB is outside LPM`, erreur `-16` (`EBUSY`).

Le passage temporaire du **seul contrôleur OTG** de `peripheral` à `none` supprime ce verrou. Le contrôleur entre en LPM, les tests préparatoires passent, puis **un vrai cycle suspend-to-RAM `deep` réussit**, avec réveil par le bouton Power et retour ADB.

Preuves du cycle réel, TEST07 :

- `pm_test = none` : aucune simulation de veille active ;
- `mem_sleep = deep` ;
- `echo mem` retourne **0** ;
- `suspend_stats.success` passe de **3 à 4**, `fail` reste à **2** ;
- durée de sommeil rapportée par `last_suspend_time` : **28,647334573 s**, hors transition ;
- réveil : **IRQ 31, `pon_kpdpwr_status`**, correspondant à l'appui bref effectué par l'utilisateur ;
- CPU secondaires arrêtés puis redémarrés, reprise DWC3/AIC, ADB reconnecté.

Les trois succès précédents étaient des tests `pm_test`. **Le total de quatre succès ne représente donc pas quatre veilles profondes : une seule veille réelle a été validée.** La consommation électrique et les états détaillés des rails Qualcomm n'ont pas été mesurés.

## Baseline réellement utilisée

| Élément | État vérifié |
|---|---|
| Image de référence | `Q6A-Android15-spi-emmc-boot-20260630-b1` |
| Android | 15, build userdebug |
| Kernel | `5.4.295-qgki-debug-g646f05065a7e-dirty` |
| Slot | `_a` |
| SELinux | **Enforcing pendant toute l'enquête** |
| `lpm_levels.sleep_disabled` | `N` |
| DTBO actif | Patch `qcom,ignore-wakeup-src-in-hostmode` déjà présent sur `/soc/hsusb@8c00000` |
| SHA-256 de `dtbo_a` | `25c95174e46f253ece0d5a5c9d0033d6d56a42fd9563d0edb5d2ed046baf6c42` |
| AIC8800 | `a69c:8d81`, `power/control=on` au départ et après le cycle réel |
| Radios | Wi-Fi et Bluetooth activés ; aucun `svc ... disable` appliqué dans cette enquête |

Le SHA du DTBO actif correspond au fichier local `Backups_Q6A/dtbo_2026-09-24/dtbo_a_patched_ignore-wakeup-src-in-hostmode.img`. Le backup stock porte le SHA `07cf1fa07e797cf4905c47a6265e51d4bac72be250b746cc3d3b30da228179d3`.

La comparaison des images stock et patchée confirme **85 entrées, seule l'entrée 40 modifiée**. Le diff DTS de cette entrée ajoute uniquement un fragment visant `/soc/hsusb@8c00000` avec cette propriété. Le contrôleur OTG `a600000.ssusb` n'est pas modifié par ce patch.

**Aucune partition n'a été flashée pendant la présente enquête.** Le rapport antérieur qui disait « DTBO non appliqué » était dépassé par l'état matériel constaté.

## Mécanisme établi

### 1. Deux contrôleurs, deux verrous successifs

`8c00000.hsusb` porte le hub interne et l'AIC8800 Wi-Fi/Bluetooth. C'était le premier périphérique en échec dans les tests historiques sur le DTBO stock.

`a600000.ssusb` porte le gadget `diag,adb` sur le câble vers le PC. Avec le patch interne déjà chargé, la trace TEST01 montre :

```text
msm-dwc3 8c00000.hsusb ... bus [suspend]
msm-dwc3 8c00000.hsusb, err=0
msm-dwc3 a600000.ssusb ... bus [suspend]
msm-dwc3 a600000.ssusb, err=-16
```

Le premier verrou franchi rend donc visible le suivant. Les anciennes expériences qui échouaient d'abord sur `8c00000.hsusb` ne démontraient pas que le contrôleur OTG pourrait ensuite suspendre.

```text
Contrôleur interne 8c00000.hsusb
  hub USB → AIC8800 Wi-Fi/BT
  problème historique : chaîne maintenue active / sortie de LPM avant le contrôle
  patch déjà installé : chemin host de suspension active
  résultat actuel     : passe la suspension, même avec AIC power/control=on

Contrôleur externe a600000.ssusb
  gadget diag,adb → câble OTG → PC
  état courant        : peripheral / active / source de réveil active
  effet Android       : lecture de wakeup_count en attente
  effet test manuel   : callback de suspension retourne -EBUSY
  test causal         : mode none → LPM → veille deep réussie
```

Le chemin host alternatif ne se contente pas d'ignorer un code d'erreur : lorsque `ignore_wakeup_src_in_hostmode` et `in_host_mode` sont vrais, `dwc3_msm_pm_suspend()` appelle `dwc3_msm_suspend()`. Le chemin par défaut vérifie seulement que `in_lpm` est déjà vrai. La propriété modifie aussi l'initialisation du réveil : le code saute `device_init_wakeup(mdwc->dev, 1)` lorsqu'elle est présente. **Son effet sur les capacités de réveil USB est donc réel dans le code, et doit être qualifié sur le matériel.**

### 2. Pourquoi Android ne tentait pas la veille

La baseline montre simultanément :

- `mHalAutoSuspendModeEnabled=true` ;
- la source noyau `a600000.ssusb` active, avec une durée active croissante ;
- un thread de `android.system.suspend-service` dans cette pile :

```text
pm_get_wakeup_count
wakeup_count_show
sysfs_kf_seq_show
seq_read
vfs_read
```

Cette lecture attend la fin des événements de réveil en cours. Le code AOSP de SystemSuspend lit `/sys/power/wakeup_count` avant d'écrire dans `/sys/power/state`. L'attente observée explique donc l'absence de tentative, sans supposer que le service est arrêté ou que SELinux lui interdit cette lecture.

**Validation causale TEST06 :** en maintenant SELinux Enforcing, sans écriture manuelle de `mem`, la mise au repos de l'OTG permet au service Android de lancer une tentative. `suspend attempts` passe de **0 à 1** ; la trace identifie le thread `binder:697_5`, TID **3465**, comme auteur de `suspend_enter`. Cette tentative utilise volontairement `pm_test=devices`, pour revenir automatiquement.

Les refus SELinux historiques restent des défauts possibles à examiner, mais **ils ne constituent pas un obstacle absolu à l'autosuspend dans cette configuration**. Les snapshots comportent aussi des suspend blockers Android transitoires : l'OTG est un verrou démontré, pas une promesse que tout état du framework est toujours prêt à suspendre.

### 3. Pourquoi le rôle logique USB est déterminant

Le Device Tree réellement chargé expose :

```text
/soc/ssusb@a600000/dwc3@a600000/dr_mode = "peripheral"
```

Le wrapper n'a ni propriété `extcon`, ni `usb-role-switch`. La reconstruction hors ligne du DT stock, base FDT 18 + overlay 40, retrouve cette configuration.

Dans le code Qualcomm de référence conservé :

- `dwc3_msm_default_peripheral()` initialise `vbus_active=true` en mode périphérique sans extcon/role-switch ;
- `mode_show()` affiche `peripheral` lorsque ce booléen est vrai ;
- `mode_store("none")` le met à faux, laisse l'ID flottant, puis notifie la machine d'états USB ;
- `dwc3_msm_resume()` prend une référence de réveil via `pm_stay_awake()` ; le chemin de mise au repos peut la libérer via `pm_relax()` ;
- `dwc3_msm_pm_suspend()` refuse la suspension si `in_lpm` est faux, hors du chemin host alternatif activé par le patch.

L'expérience confirme cette transition sur la carte :

```text
mode peripheral : UDC configured, runtime_status active, -EBUSY
mode none       : UDC not attached, runtime_status suspended, suspension possible
```

**Un câble débranché, un gadget délié et un rôle logique `none` ne sont pas équivalents.** Le TEST02 démontre notamment qu'un simple effacement de `g1/UDC` est annulé par les règles Android : le gadget se reconfigure avant la tentative. Aucun nouveau test de débranchement physique isolé n'a été effectué dans cette enquête ; le comportement exact câble absent reste à qualifier avant de choisir une correction permanente.

Le fichier Qualcomm provient du tag `LA.UM.9.14.7.r1-02400-QCM6490.QISI15.0`. Il explique les chemins observés, mais le kernel embarqué est marqué `dirty` : cela ne constitue pas une preuve que tout le binaire Radxa est identique à ce tag.

### 4. Ce qui reste inexpliqué dans les anciens essais

La cause exacte de la sortie du LPM du **contrôleur interne avant patch** n'a pas été reproduite et tracée jusqu'à son appelant pendant cette reprise. Il serait incorrect de la déclarer définitivement attribuée à Bluetooth ou à une IRQ externe.

Une piste précise apparaît dans `drivers/usb/core/driver.c`, récupéré au tag Qualcomm de référence : `choose_wakeup()` appelle `pm_runtime_resume()` sur un périphérique déjà suspendu si son réglage de réveil distant diffère de celui requis pour le system suspend. Une sortie de LPM pendant la préparation de la veille peut donc être demandée par le noyau lui-même, sans incrément d'un compteur de réveil externe. **Cette piste n'est pas démontrée pour les anciens échecs du Q6A** ; elle demanderait une pile d'appel ou une trace ciblée dans les conditions concernées.

La série historique de dix tentatives avait sept échecs USB et trois échecs de gel des tâches. Elle prouve l'échec de ces dix essais dans leurs conditions, mais ni l'impossibilité générale de suspendre sans DTBO, ni la nécessité absolue du patch, ni l'identité de la source de sortie du LPM. Dans les tests de la présente enquête après redémarrage, `failed_freeze` reste à zéro.

## Expériences et résultats

Numérotation **locale à cette enquête**, distincte des TEST 1–10 du journal historique.

| Test | Modification / méthode | Résultat |
|---|---|---|
| TEST01 | Baseline patchée, OTG connecté, `pm_test=devices` | Interne `8c00000` passe ; OTG `a600000` échoue `-16` |
| TEST02 | Gadget délié via `g1/UDC` | Android le relie immédiatement ; même échec. Pas un test valide d'OTG durablement détaché |
| TEST03 | `sys.usb.config=none` | Arrêt d'adbd et interruption du script avant la tentative ; **aucun résultat de suspend**. Power-cycle utilisateur nécessaire |
| TEST04 | OTG `mode=none`, `pm_test=devices` | OTG en LPM, retour 0 ; restauration ADB automatique |
| TEST05 A | `deep`, `pm_test=core`, OTG peripheral | Échec `a600000`, `-16` |
| TEST05 B | Même test, OTG none | Retour 0, arrêt/reprise CPU secondaires ; simulation, pas de sommeil réel |
| TEST05 A′ | Même test, OTG peripheral rétabli | Retour du même échec `-16` |
| TEST06 | Autosuspend Android, `pm_test=devices`, OTG none, Enforcing | Une tentative automatique réussie, auteur confirmé par trace |
| TEST07 | `pm_test=none`, `mem_sleep=deep`, OTG none | **Vraie veille ~28,65 s, retour 0, réveil Power, ADB rétabli** |

La séquence A/B/A′ isole le rôle de l'OTG : ni le DTBO, ni SELinux, ni les radios, ni la politique runtime-PM AIC ne changent entre ces trois tentatives.

Pour éviter de répéter l'incident TEST03, les scripts suivants ont quitté le cgroup d'adbd avant la coupure de liaison. Ils utilisent l'attribut `mode` du wrapper et restaurent `peripheral` à leur sortie. Les tests manuels prennent un wakelock temporaire pour éviter une tentative Android concurrente. TEST06 le relâche pendant les fenêtres d'observation.

## État après réveil et portée de la validation

Vérifications finales :

- ADB `device`, USB `diag,adb`, UDC `configured`, rôle `peripheral` restauré ;
- SELinux Enforcing, SHA du DTBO inchangé ;
- `pm_test=none`, `mem_sleep=s2idle` restauré à la valeur initiale ;
- wakelock temporaire supprimé, événements des instances de trace désactivés ;
- threads AIC présents en état `S`, aucun nouveau thread AIC bloqué en `D` ;
- Wi-Fi activé, commande de scan envoyée, **21 lignes de résultats** récupérées après reprise ;
- Bluetooth `enabled: true`, état `ON`.

Le scan ne prouve pas une connexion Wi-Fi avec trafic. Aucun échange Bluetooth avec un périphérique n'a été testé. Trois autres threads noyau en `D`, déjà présents dans la baseline, ne doivent pas être présentés comme un nouveau deadlock AIC.

**L'état final conserve l'OTG opérationnel pour travailler : le verrou associé peut donc être de nouveau présent. Aucun correctif permanent du port OTG n'a été installé.**

## Conséquences pour la suite

1. Le patch interne permet bien de franchir `8c00000.hsusb` dans cette configuration. Sa nécessité absolue et les wakeups Wi-Fi/BT qu'il peut supprimer ne sont pas démontrés par cette enquête.
2. `power/control=on` de l'AIC reste un problème d'efficacité runtime-PM à étudier, mais **n'empêche pas le cycle deep validé avec le patch déjà chargé**. L'origine exacte de cette politique dans le driver AIC reste à auditer.
3. La correction permanente doit gérer le rôle et la présence USB/VBUS du port externe, et la politique de veille lorsqu'un hôte de debug est connecté. L'écriture `mode=none` est un contournement de diagnostic qui coupe la liaison USB, pas une solution finale transparente.
4. Le nom `ignore-wakeup-src-in-hostmode` concerne le chemin host : recopier ce patch sur le contrôleur externe en mode périphérique ne constitue pas une correction justifiée.
5. Reste à qualifier les cycles répétés, le trafic Wi-Fi/BT après reprise, les réveils requis par le téléphone, l'autosuspend Android réel en `deep`, et la consommation. Un seul cycle réel ne qualifie pas un téléphone complet.

Le `Suspend Info` Android a affiché une durée aberrante après une simulation `pm_test`. Les dates RTC ont également varié entre boots. Les durées des simulations ne doivent pas être utilisées pour mesurer l'autonomie. La durée citée pour TEST07 vient du **`last_suspend_time` de ce cycle réel**, corroboré par l'entrée deep et le réveil Power.

## Preuves conservées

Dossier local : [notes/investigation_2026-09-24](notes/investigation_2026-09-24/).

- [Baseline et service Android](notes/investigation_2026-09-24/baseline.txt), [pile d'attente](notes/investigation_2026-09-24/inspect.txt).
- [DT runtime et règles init expliquant TEST03](notes/investigation_2026-09-24/inspect02.txt).
- [Trace TEST01](notes/investigation_2026-09-24/device/test01_trace.txt).
- [Aller-retour causal TEST05](notes/investigation_2026-09-24/test05_aba_core.log), [dmesg](notes/investigation_2026-09-24/test05_dmesg.txt).
- [Autosuspend TEST06](notes/investigation_2026-09-24/test06_android_auto.log), [trace de son auteur](notes/investigation_2026-09-24/test06_trace.txt).
- [Cycle réel TEST07](notes/investigation_2026-09-24/test07_real_deep.log), [dmesg](notes/investigation_2026-09-24/test07_dmesg.txt), [script utilisé](notes/investigation_2026-09-24/test07.sh).
- [État final](notes/investigation_2026-09-24/final_check.txt).

Références de code : [driver Qualcomm local](notes/dwc3-msm-src-ref/dwc3-msm_LA.UM.9.14.7.r1-02400-QCM6490.QISI15.0.c), [SystemSuspend AOSP Android 15](https://android.googlesource.com/platform/system/hardware/interfaces/+/refs/heads/android15-release/suspend/1.0/default/SystemSuspend.cpp), [wakeup.c Android common 5.4](https://android.googlesource.com/kernel/common/+/refs/heads/android12-5.4/drivers/base/power/wakeup.c).

Les scripts, captures et extractions sont des éléments locaux d'investigation ; aucun commit ni publication n'a été effectué.

## Niveau de certitude des conclusions

| Conclusion | Niveau et justification |
|---|---|
| Le patch DTBO était déjà installé à la reprise | **Confirmé** : propriété runtime + SHA partition identique au fichier patché |
| Le blocage manuel actuel est sur l'OTG externe | **Confirmé** : callback `a600000.ssusb`, retour `-16`, séquence A/B/A′ |
| Mettre ce contrôleur en `none` permet de franchir le verrou | **Confirmé** : runtime suspend, tests préparatoires et cycle réel |
| Le Q6A a effectué une vraie veille deep puis repris sur Power | **Confirmé pour un cycle** : test désactivé, succès noyau, durée et IRQ de réveil |
| Android attendait sur le protocole wakeup_count | **Confirmé** : pile du thread, source active, progression après mise au repos USB |
| SELinux doit être désactivé pour permettre l'autosuspend | **Contredit dans cette configuration** : tentative automatique en Enforcing |
| La politique de rôle USB du BSP est impliquée | **Confirmé pour la transition testée** : DT, état logique et effet de `mode=none` ; correction de détection physique encore à concevoir |
| Bluetooth est responsable de la sortie de LPM historique | **Non démontré** |
| Un patch du driver AIC est indispensable au deep avec le DTBO actuel | **Non** pour le cycle testé : deep réussi sans modifier AIC ; efficacité runtime encore à travailler |
| Le DTBO actuel est le correctif final optimal | **Non démontré** : wakeups host et robustesse à qualifier |
| Le téléphone possède maintenant une veille autonome fiable et économe | **Non validé** : une preuve de mécanisme et un cycle réussi, pas une qualification complète |
