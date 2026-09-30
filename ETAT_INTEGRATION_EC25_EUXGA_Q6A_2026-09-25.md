# EC25-EUXGA sur Q6a — état vérifié le 25 septembre 2026

## Conclusion

Le modem **GA** communique avec Android 15 jusqu'au constat de **SIM absente**. Le EC25 répond aux commandes AT ; le RIL Quectel AIDL1 publie sept services Radio ; le framework Android reçoit `CARDSTATE_ABSENT` et expose `gsm.sim.state=ABSENT`. La chaîne complète a été observée sur le Q6a réel le 24 septembre. Le 25 septembre, la présence des deux processus, des sept services et de l'état `ABSENT` a été recontrôlée ; les échanges AT et les événements `RILJ` n'ont pas été rejoués ce jour-là.

L'installation est encore un **prototype**. Les changements de fichiers vivent dans des overlays `vendor`, `system_ext` et `product` ; le pont USB AT et le RIL sont lancés manuellement dans `/data/local/tmp` sous ADB root. Ils ne démarrent pas seuls. Les partitions sous-jacentes n'ont pas été réécrites et aucun `boot.img`, `dtbo.img` ni firmware QSPI n'a été flashé pour le modem. Une intégration durable exigera les pilotes noyau, un service `init` et une politique SELinux, puis des images reconstruites et un flash ciblé avec AVB cohérent.

## Ce qui est prouvé, et sa limite

| Point | Preuve sur la carte | Limite actuelle |
|---|---|---|
| Identité du modem | `AT+QGMR` → `EC25EUXGAR08A05M1G_01.001.01.001` ; USB `2c7c:0125`. | Les documents ou réglages GR ne sont pas présumés applicables au GA. |
| Accès AT | Interface USB 2, endpoints bulk OUT `0x03` / IN `0x84` ; `AT+CPIN?` → `+CME ERROR: 10`. | Accès assuré par un pont `usbfs`/PTY de développement, faute de pilote série `option`. |
| Radio Android | `service list` donne `IRadioConfig/default` et les six services `IRadio{Data,Messaging,Modem,Network,Sim,Voice}/slot1`. Les logs antérieurs montrent `RILJ`, `GET_SIM_STATUS` → `CARDSTATE_ABSENT`, puis `SIM_STATE_CHANGED ABSENT`. | Les services sont lancés par ADB root ; SELinux est `Enforcing` mais les processus tournent dans le domaine `su`, pas dans un domaine de production. |
| Slot et état SIM | `ro.telephony.sim_slots.count=1`, `telephony.active_modems.max_count=1`, `gsm.sim.state=ABSENT`. | Aucun test avec SIM insérée. `gsm.sim.state` seul peut rester mémorisé après arrêt du RIL : vérifier aussi les processus et services. |
| Données cellulaires | Aucun `cdc-wdm*` ni `wwan*` du EC25 sur le noyau stock. | QMI/data, SMS, appels, audio, IMS/VoLTE et enregistrement réseau non validés. |
| Suspend-to-RAM | SHA-256 de `dtbo_a` toujours `25c95174e46f253ece0d5a5c9d0033d6d56a42fd9563d0edb5d2ed046baf6c42`. | Coexistence STR avec pilotes et modem intégrés à tester. |

## Relevé de contrôle du 25 septembre 2026

Depuis le PC, `adb devices` indiquait `8ba07537 device`. Sur le Q6a : `sys.boot_completed=1`, `getenforce=Enforcing`, `ro.boot.veritymode=disabled`, sept services Radio enregistrés, `gsm.sim.state=ABSENT`, et les processus `ec25_usb_pty` et `ec25_rild_probe` présents. `init.svc.vendor.qcrild=stopped`. Les montages `vendor`, `system_ext` et `product` montrent un système de fichiers ext4 en lecture seule sous un overlay utilisant `/mnt/scratch/overlay/...`. Le DTBO porte le SHA-256 ci-dessus.

Le contrôle a aussi renvoyé `persist.radio.multisim.config=ssss`, alors que le journal de l'essai précédent relevait `ss` et que le slot effectif reste `1`. **Cette divergence de propriété est à expliquer avant de figer l'image** ; elle ne remet pas en cause le résultat Radio/SIM observé. L'horloge du Q6a indiquait le 11 juin 2026 lors du relevé : les dates Android de fichiers ou de logs ne doivent pas servir seules à dater les essais ; la date du titre est celle du PC.

## Changements du prototype par partition

| Emplacement | Changement actuellement essayé | À reporter dans l'intégration durable |
|---|---|---|
| `vendor` | `ro.radio.noril=false`, `ro.vendor.radio.noril=false` ; exclusions téléphonie retirées ; manifeste VINTF Yupik converti de Radio HIDL à sept instances AIDL1 ; RIL Quectel et bibliothèques Radio V1 copiés sous `/vendor/lib64/ec25`. | Images vendor, service `init`, chemins des bibliothèques, règles `ueventd` et SELinux ; vérifier le comportement des autres paquets Qualcomm. |
| `system_ext` | Deux propriétés de slots/modems passées de `2` à `1`. | Image `system_ext_a`. |
| `product` | Deux overlays Qualcomm masqués : `TelephonyResCommon_Sys.apk` (injection QtiRIL) et `FrameworksResNonModem_Sys.apk` (voix/SMS/data désactivés). | Image `product_a` avec retrait ciblé et contrôle des ressources résultantes. |
| `boot` | Aucun changement. Le noyau stock a `CONFIG_USB_SERIAL_OPTION`, `CONFIG_USB_WDM` et `CONFIG_USB_NET_QMI_WWAN` désactivés. | Source Radxa exacte et toolchain compatible, puis activation des pilotes et vérification du mapping réel des interfaces USB GA. |
| `dtbo` | Aucun changement pendant les essais modem. Le patch STR préexistant est conservé. | Préserver l'image patchée ; vérifier son SHA avant/après tout flash. |

Les huit bibliothèques Radio V1 du paquet AIDL1 proviennent de l'APEX VNDK v33 présent sur cette image ; la copie isolée a résolu le chargement du RIL. Le paquet AIDL3 demanderait huit bibliothèques V3 absentes : **AIDL1 est la base du prototype validé**, pas une preuve que toutes les fonctions cellulaires sont opérationnelles.

## Prochaines étapes et critères de validation

1. **Stabiliser le prototype** : expliquer la valeur `ssss`, inventorier exactement les fichiers et les domaines SELinux, observer les effets sur les paquets Qualcomm. Garder les changements traçables par partition et conserver les sauvegardes.
2. **Pilotes EC25** : obtenir le noyau Radxa exact `5.4.295-qgki-debug-g646f05065a7e-dirty` et son environnement de compilation ; intégrer `USB_SERIAL_WWAN`, `USB_SERIAL_OPTION`, `USB_WDM`, `USB_NET_QMI_WWAN`. Vérifier sur la carte les pilotes liés à chaque interface, les nœuds `ttyUSB*`/`cdc-wdm*`, l'interface réseau, puis les réponses AT. Le clone CodeLinaro proche n'est pas une base de flash validée.
3. **Démarrage autonome** : installer un `rild` compatible avec le RIL Quectel AIDL1, son service `init`, les droits de périphériques et la politique SELinux minimale. Après un reboot sans ADB root ni script manuel, les sept services doivent revenir et Android doit retrouver `ABSENT`.
4. **Images et flash ciblé** : reconstruire `vendor_a`, `system_ext_a`, `product_a` et, pour les pilotes, `boot_a` ; traiter les tailles des partitions et AVB/verity. Ne pas reflasher une image eMMC stock complète, qui écraserait le DTBO STR patché. Conserver et contrôler les sauvegardes `boot_a`, `vendor_boot_a`, `dtbo_a`, `vbmeta_a`, `vbmeta_system_a` et les métadonnées `lpdump`.
5. **Qualification fonctionnelle** : avec une SIM, tester PIN, réseau, QMI/data, SMS, appels et audio ; puis STR/réveil avec le modem branché. Sans SIM, le contrôle de l'état absent et du démarrage autonome reste possible, mais pas la validation réseau.

Le [journal d'essai](notes/ec25_integration_2026-09-24.md) contient les observations chronologiques et le retour arrière ; le [plan d'intégration](PLAN_INTEGRATION_EC25_EUXGA_ANDROID15.md) détaille les dépendances par couche ; la [baseline GA](notes/ec25_euxga_baseline_2026-09-24.md) conserve les réponses AT et l'état initial. Le script [`scripts/start_ec25_probe.ps1`](scripts/start_ec25_probe.ps1) relance le prototype après reboot si ses binaires sont encore présents. Une disparition momentanée d'ADB/LED s'est produite pendant les essais ; sa cause reste indéterminée et aucun `rild` de production n'avait été lancé à ce moment.