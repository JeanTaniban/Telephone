# Mission — consolidation CDC Power/MCU V0.9 + BOM — 2026-10-05

## Objectif

Consolider le CDC normatif de la carte Power/MCU du Maker Phone Q6A après validation de l'alimentation régulée du modem par RT6154A, et intégrer une BOM exploitable pour schéma, PCB et préparation JLCPCB.

## Périmètre inclus

- nouvelle architecture `SYS -> MODEM_PWR -> RT6154A -> EC25 BAT ~3.8 V` ;
- conservation du switch USB_VBUS EC25 indépendant ;
- conservation du JMTQ55P02A MODEM_PWR comme coupure primaire/hard-off + hold reset MCU ;
- option NTC2 vers GP28/ADC2 en DNP, alternative au RESET_N EC25 DNP ;
- BOM complète : références fabricant/LCSC, quantités, Basic/Extended, DNP, rôle précis, prix indicatifs, coût par carte et frais Extended ;
- mise à jour de l'index `CURRENT.md`.

## Hors périmètre

- timings firmware détaillés ;
- seuils logiciels exacts de batterie faible ;
- valeurs TS/JEITA finales avant caractérisation physique de la JK50 ;
- validation thermique et RF du produit fini.

## Fichiers concernés

- `CDC_CARTE_POWER_MCU_V1_2026-10-05_CURRENT.md` : nouvelle référence normative V0.9 ;
- `CURRENT.md` : pointeur courant ;
- la V0.8 existante reste historique et n'est pas écrasée.

## Contraintes

- JLCPCB Economic PCBA ;
- priorité Basic > Promotional > Extended ;
- PCB 4 couches, <= 70 x 25 mm ;
- RP2040-Tiny monté manuellement BOTTOM ;
- aucune réintroduction USB-C host/PD/DRP ;
- EC25 BAT doit rester régulé dans sa plage sûre indépendamment de la tension haute de la JK50.

## Tests documentaires

1. aucune occurrence normative de `SYS -> MODEM_PWR -> EC25 BAT` direct dans le nouveau CDC ;
2. présence du RT6154A, de son inductance, de son feedback et de ses condensateurs dans la BOM ;
3. cohérence pinout MCU : GP28 = NTC2_ADC DNP ou RESET_N DNP, usages exclusifs ;
4. cohérence machine d'états : MODEM_PWR puis rail régulé modem, USB_VBUS séparé ;
5. BOM avec quantités, classe JLC, prix snapshot et rôle ;
6. frais Extended comptés par référence unique, pas par exemplaire ;
7. points non gelés explicitement marqués TBD/DNP.

## Critères de validation

La mission est validée si le CDC V0.9 est autonome, sans contradiction interne connue, si `CURRENT.md` le désigne comme référence normative courante et si les éléments encore ouverts sont clairement séparés des choix figés.

## Risques de régression

- conserver par erreur l'ancienne limite de charge JK50 uniquement due à l'EC25 direct ;
- oublier le hard-off modem en supprimant MODEM_PWR ;
- utiliser GP28 simultanément pour NTC2 et RESET_N ;
- figer des valeurs TS/JEITA non mesurées ;
- considérer les prix/classes JLC comme immuables alors qu'ils doivent être revalidés à la commande.
