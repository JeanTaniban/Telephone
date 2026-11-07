# Q6A J20 — diagnostic des contacts endommagés et nouvelle allocation GPIO

**Date : 2026-11-07**  
**Plateforme : Radxa Dragon Q6A V1.21**  
**Statut : constat utilisateur + analyse documentaire ; aucun contrôle électrique ou test Android réalisé sur la carte endommagée.**  
**Priorité : cette note complète le CDC Power/MCU V0.9 du 2026-10-05 sans modifier son schéma d'alimentation.**

## 1. Constat, portée et précautions

Le connecteur d'extension **J20 (2 × 20, 40 contacts)** a perdu ou endommagé, selon le constat utilisateur, les contacts **13, 17, 26, 28, 32, 38, 40**. Certaines connexions peuvent être récupérables ; leur état électrique n'a **pas** été mesuré. Un contact « intact présumé » n'est pas un résultat de continuité.

- Six GPIO concernées : **GPIO0, GPIO55, GPIO9, GPIO30, GPIO98, GPIO99**.
- Une alimentation concernée : **J20.17 = 3,3 V** ; utiliser J20.1 après mesures.
- Les fonctions prioritaires **I²C6 J20.3/5**, **SPI12 J20.19/21/23/24**, **GPIO31 J20.29**, **GPIO59 J20.36** et **GPIO58 J20.37** ne figurent pas dans la liste endommagée.
- **Ce constat ne prouve pas qu'elles fonctionnent.** Vérifier l'absence de court-circuit et la continuité des pistes voisines avant toute mise sous tension.
- Le connecteur DSI **J10** et le USB hôte EC25 sont séparés de J20 ; aucun dommage n'y a été rapporté, mais ceci ne vaut pas test de fonctionnement.

### Sources du brochage

1. [Radxa — Q6A 40-Pin GPIO Interface](https://docs.radxa.com/en/dragon/q6a/hardware-use/pin-gpio), numérotation physique / GPIO et multiplexage.
2. [Radxa — schéma Dragon Q6A V1.21](https://dl.radxa.com/dragon/q6a/hw/radxa_dragon_q6a_schematic_v1.21.pdf), feuille JTAG / GPIO / J20 (lire le dessin avec son orientation et non l'ordre d'extraction texte du PDF).
3. [Radxa — Product Brief Q6A](https://dl.radxa.com/dragon/q6a/docs/rad-doc-0150_radxa_dragon_q6a_product_brief__revision_1.5_gbc16bac.pdf), tableau GPIO.
4. CDC projet : `6 et `7 de `CDC_CARTE_POWER_MCU_V1_2026-10-05_CURRENT.md`.

**Attention :** les fonctions indiquées sont des fonctions **multiplexées**, pas des fonctions simultanées ni nécessairement activées par le BSP Android 15. Les noms TLMM `GPIOxx` ne sont pas les numéros physiques J20.

## 2. Inventaire intégral des 40 contacts (orientation physique J20)

| J20 | Signal/rail | Fonctions publiques (sélection) | État projet | Usage / impact |
|---:|---|---|---|---|
| 1 | 3V3 | 3,3 V | Réutilisable présumé | Référence Q6A_3V3_REF, à vérifier |
| 2 | 5V | 5 V | Réutilisable présumé | Alimentation auxiliaire; pas une entrée batterie |
| 3 | GPIO24 | I²C6 SDA / SPI6 MISO | Réservé CDC | SBS SDA Q6A ↔ RP2040 GP14 |
| 4 | 5V | 5 V | Réutilisable présumé | 5 V auxiliaire |
| 5 | GPIO25 | I²C6 SCL / SPI6 MOSI | Réservé CDC | SBS SCL Q6A ↔ RP2040 GP15 |
| 6 | GND | Masse | Réutilisable présumé | Masse |
| 7 | GPIO96 | MI2S MCLK | Libre sous réserve | Horloge audio numérique optionnelle |
| 8 | GPIO22 | UART5 TX / SPI5 SCLK | Libre sous réserve | UART5 éventuel |
| 9 | GND | Masse | Réutilisable présumé | Masse |
| 10 | GPIO23 | UART5 RX / SPI5 CS0 | Libre sous réserve | UART5 éventuel |
| 11 | GPIO29 | I²C7 SCL / SPI7 MOSI | Libre sous réserve | I²C7 SCL option expandeur GPIO |
| 12 | GPIO97 | MI2S0 SCK | Libre sous réserve | Audio numérique optionnel |
| 13 | GPIO0 | I²C0 SDA / SPI0 MISO | Endommagé déclaré | Ne pas compter sur I²C0/SPI0 via ce contact |
| 14 | GND | Masse | Réutilisable présumé | Masse |
| 15 | GPIO1 | I²C0 SCL / SPI0 MOSI | Libre sous réserve | Bus I²C0 incomplet avec pin 13 |
| 16 | GPIO26 | UART6 TX / SPI6 SCLK | Proposé — non validé | TP_INT tactile Moto G200 |
| 17 | 3V3 | 3,3 V | Endommagé déclaré | Remplacer prise 3,3 V par J20.1 |
| 18 | GPIO27 | UART6 RX / SPI6 CS0 | Proposé — non validé | TP_RESET tactile Moto G200 |
| 19 | GPIO49 | SPI12 MOSI / I²C12 SCL | Réservé projet — non testé | MOSI tactile |
| 20 | GND | Masse | Réutilisable présumé | Masse |
| 21 | GPIO48 | SPI12 MISO / I²C12 SDA | Réservé projet — non testé | MISO tactile |
| 22 | GPIO57 | UART14 RFR / I²C14 SCL / SPI14 MOSI | Proposé — non validé | Volume− remplace pin 32 |
| 23 | GPIO50 | SPI12 SCLK | Réservé projet — non testé | Horloge SPI tactile |
| 24 | GPIO51 | SPI12 CS0 | Réservé projet — non testé | Sélection SPI tactile |
| 25 | GND | Masse | Réutilisable présumé | Masse |
| 26 | GPIO55 | SPI12 CS1 / SPI13 CS0 | Endommagé déclaré | CS1 indisponible présumé; CS0 pin24 suffit |
| 27 | GPIO8 | I²C2 SDA / SPI2 MISO | Libre sous réserve | I²C2 incomplet avec pin28 |
| 28 | GPIO9 | I²C2 SCL / SPI2 MOSI | Endommagé déclaré | Bus I²C2 non complet |
| 29 | GPIO31 | SPI7 CS0 | Réservé CDC | Volume+ |
| 30 | GND | Masse | Réutilisable présumé | Masse |
| 31 | GPIO28 | I²C7 SDA / SPI7 MISO | Libre sous réserve | I²C7 SDA option expandeur GPIO |
| 32 | GPIO30 | UART7 TX / SPI7 SCLK | Endommagé déclaré | Ancien Volume−, abandon d'affectation |
| 33 | GPIO56 | I²C14 SDA / SPI14 MISO | Libre sous réserve | Disponibilité à vérifier; I²C14 SCL réaffecté Volume− |
| 34 | GND | Masse | Réutilisable présumé | Masse |
| 35 | GPIO100 | MI2S0 WS | Libre sous réserve | Audio numérique optionnel |
| 36 | GPIO59 | UART14 RX / SPI14 CS0 | Réservé CDC | Heartbeat Q6A vers RP2040 GP27 |
| 37 | GPIO58 | UART14 TX / SPI14 SCLK | Réservé CDC | SHUTDOWN_REQ depuis RP2040 GP9 |
| 38 | GPIO98 | MI2S0 DATA0 | Endommagé déclaré | Audio numérique optionnel perdu |
| 39 | GND | Masse | Réutilisable présumé | Masse |
| 40 | GPIO99 | MI2S0 DATA1 | Endommagé déclaré | Audio numérique optionnel perdu |

**Bilan des 28 GPIO du header :** 9 affectées au CDC/au SPI tactile à préserver (3, 5, 19, 21, 23, 24, 29, 36, 37), 6 contacts GPIO déclarés endommagés (13, 26, 28, 32, 38, 40), 13 autres GPIO accessibles en principe ; parmi ces 13, trois sont proposées au tactile/Volume− (16, 18, 22), laissant **10 GPIO non attribuées**. Une GPIO non attribuée n'est pas forcément disponible dans le Device Tree Android (pinctrl, domaines électriques, conflits périphériques).

## 3. Affectations proposées pour le téléphone — ECO J20

| Fonction | J20 | GPIO Q6A | État |
|---|---:|---|---|
| Batterie virtuelle / SBS SDA depuis RP2040 GP14 | 3 | GPIO24 / I²C6 SDA | CDC V0.9, à tester |
| Batterie virtuelle / SBS SCL depuis RP2040 GP15 | 5 | GPIO25 / I²C6 SCL | CDC V0.9, à tester |
| Tactile SPI12 MOSI | 19 | GPIO49 | Proposé, non testé |
| Tactile SPI12 MISO | 21 | GPIO48 | Proposé, non testé |
| Tactile SPI12 CLK | 23 | GPIO50 | Proposé, non testé |
| Tactile SPI12 CS0 | 24 | GPIO51 | Proposé, non testé |
| Tactile interruption `TP_INT` | 16 | GPIO26 | Proposé, non testé |
| Tactile reset `TP_RESET` | 18 | GPIO27 | Proposé, non testé |
| Volume+ | 29 | GPIO31 | CDC V0.9, à tester |
| **Volume−** (nouveau) | **22** | **GPIO57** | **ECO proposé, à valider matériel + Android** |
| `HEARTBEAT_STATE` | 36 | GPIO59 | CDC V0.9, à tester |
| `SHUTDOWN_REQ` | 37 | GPIO58 | CDC V0.9, à tester |
| Référence 3,3 V | 1 | 3V3 | Remplace J20.17, à mesurer |

La sortie `PWR_ON_KEY` du Q6A reste commandée par **RP2040 GP8 → S8050 → vrai bouton Q6A**, conformément au CDC. La touche Power utilisateur reste sur GP7 du RP2040.

### SPI tactile — points particuliers

- `SPI12_CS1` sur J20.26 est endommagé ; un unique périphérique tactile peut employer **CS0 J20.24**. Aucun pontage de CS1 n'est requis.
- **SPI12 (19/21/23/24)** doit être configuré dans le pinmux/Device Tree Android 15. Les directions MOSI/MISO et le routage réel doivent être vérifiés contre la documentation du contrôleur tactile et au banc.
- Le tactile Moto G200 envisagé utilise une logique **1,8 V à confirmer sur la dalle réelle** : ajouter les convertisseurs de niveaux appropriés entre le Q6A 3,3 V et le tactile ; ne jamais raccorder le SPI 3,3 V directement à une entrée certifiée 1,8 V.
- `TP_INT` et `TP_RESET` ne sont **pas** des broches matériellement réservées dans le CDC V0.9 ; l'allocation 16/18 est une proposition à valider (direction, pull-up, pinctrl, redémarrage).

### Volume− — ne pas réunir directement les deux boutons

- Affectation précédente : **J20.32 / GPIO30 — contact endommagé**.
- Affectation candidate : **J20.22 / GPIO57**, un bouton vers GND et tirage de rappel compatible avec le domaine GPIO après vérification ; configurer `gpio-keys` pour `KEY_VOLUMEDOWN` et tester un appui/relâchement.
- Volume+ reste **J20.29 / GPIO31**.
- **Ne pas** combiner Volume+ et Volume− dans un « pont résistif » sur une seule GPIO numérique : cette méthode demande une **entrée ADC** avec logique de décodage et seuils vérifiés ; ce n'est pas une simplification utile ici.
- Alternative de dernier recours : reprise par fil fin de la sortie 3,3 V du traducteur associé à J20.32, uniquement après identification du composant, continuité et inspection sous grossissement ; **pas nécessaire** à ce stade.

### Réserves et bus complémentaires

- **I²C7 : J20.11 (GPIO29 / SCL) + J20.31 (GPIO28 / SDA)**, candidat pour expandeur 16 GPIO de type MCP23017 à adresse distincte ; le bus devra être activé dans Android ; vérifier pull-up, 3,3 V et absence de back-power.
- **UART5 : J20.8 (TX) + J20.10 (RX)**, potentiellement disponible.
- **I²C0 :** J20.13 (SDA) perdu ; J20.15 seul ne rétablit pas le bus.
- **I²C2 :** J20.28 (SCL) perdu ; J20.27 seul ne rétablit pas le bus.
- **I²C14 :** attribuer J20.22 à Volume− exclut son utilisation simultanée comme SCL avec J20.33.
- **I²S/MI2S0 :** J20.38/40 endommagés ; les flux DATA0/DATA1 ne sont plus une option fiable sur ce header. La baseline audio Q6A/Android du CDC V0.9 ne s'appuie pas sur ces broches.

## 4. Tests de qualification et critères d'acceptation

Carte totalement **hors tension**, USB, batterie et toute autre source débranchés :

1. Inspection macro J20 : pastille absente vs piste arrachée, pont de soudure, dommages sur pistes/boîtiers voisins. Photographies avant toute intervention.
2. Multimètre : détecter un court-circuit franc entre **3,3 V / GND**, et entre contacts voisins, en tenant compte des charges internes ; contrôle de J20.1 et du cuivre restant de J20.17 ; documenter toutes les mesures.
3. Continuité point-à-point vers la destination du schéma pour les contacts déclarés endommagés **et** les contacts utiles (3, 5, 16, 18, 19, 21, 22, 23, 24, 29, 36, 37). Aucun pontage aveugle.

Une fois les contrôles passifs satisfaisants, alimentation limitée au banc puis essais ciblés :

4. Vérifier les niveaux 3,3 V, isolations, montée/descente, et l'absence de back-power en `MAIN_PWR` OFF.
5. Android 15 : lire l'état pinctrl ; faire fonctionner SPI12/CS0 ; tester l'INT et le RESET tactile ; **pas de verdict tactile sans essai de la dalle réelle**.
6. Tester `KEY_VOLUMEUP` et `KEY_VOLUMEDOWN` (via `getevent` et appui réel), `SHUTDOWN_REQ`, `HEARTBEAT_STATE` et SBS I²C6 avec RP2040.
7. Régression : USB EC25, écran DSI J10, bouton Power, deep suspend/réveil et audio habituel.

**Critères de validation :** absence de court-circuit, absence de coupure sur les 12 contacts prioritaires, SPI12 + CS0 effectifs, boutons indépendants reconnus, GPIO de supervision et SBS fonctionnels, aucune régression sur la séquence power et l'EC25. Tant qu'une mesure ou un test manque, le risque correspondant reste **ouvert**.

## 5. État des décisions

- **Constaté par l'utilisateur :** 7 contacts J20 potentiellement endommagés (sans mesure).
- **Documentaire confirmé :** fonctions publiques et alternatives indiquées dans le tableau Radxa.
- **Conception retenue pour essai :** SPI12 CS0, TP_INT J20.16, TP_RESET J20.18, Volume− J20.22.
- **Non validé :** continuité, reprises possibles, pinctrl Android, pilote tactile Moto G200, nouveaux boutons, non-régression.
- **Impact CDC :** ECO de câblage du téléphone ; aucune modification au circuit de charge/PMIC/EC25. `VOL- J20.32` n'est plus le câblage cible sur le prototype endommagé.

**Ce document ne vaut ni schéma de fabrication ni résultat de test.**
