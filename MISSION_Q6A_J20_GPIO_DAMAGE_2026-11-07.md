# MISSION — diagnostic GPIO J20 Q6A endommagé et réaffectation tactile / Volume−

**Date : 2026-11-07**  
**Projet parent :** Maker Phone Q6A V1.21 / Android 15 / EC25-EUXGA / Power-MCU V0.9.

## Objectif

Documenter la perte déclarée des contacts J20.13, 17, 26, 28, 32, 38, 40 et proposer un brochage qui préserve le téléphone complet sans nécessité immédiate de micro-réparation. Le câblage cible est provisoire jusqu'aux tests physiques et logiciels.

## Périmètre inclus

- Inventaire des 40 contacts J20, des 28 GPIO et des 7 contacts endommagés (6 GPIO + 3,3 V).
- Reprise à J20.1 de la référence 3,3 V au lieu de J20.17.
- SPI12 tactile : J20.19/21/23/24 avec CS0 ; J20.26/CS1 écarté.
- TP_INT J20.16 et TP_RESET J20.18 proposés.
- Volume− déplacé de J20.32/GPIO30 vers J20.22/GPIO57 (proposition) ; Volume+ inchangé J20.29/GPIO31.
- Maintien SBS 3/5, shutdown 37, heartbeat 36, bouton Power via vrai PWR_ON_KEY.
- Mise à jour de la référence normative CDC V0.9 **par ECO clairement identifiée** et de CURRENT.md.

## Hors périmètre

- Modification des schémas, Gerbers, PCB ou firmware Android/MCU.
- Soudure/réparation des pistes J20, requalification matérielle ou tests réels.
- Modification des fonctions modem/charge/BQ25628E et de la machine d'états Power V0.9.
- Garantir la compatibilité du tactile Moto G200 tant que contrôleur, tensions et pilote ne sont pas validés.

## Fichiers concernés

1. `Q6A_J20_GPIO_DAMAGE_PINOUT_2026-11-07.md` — état de référence détaillé.
2. `CDC_CARTE_POWER_MCU_V1_2026-10-05_CURRENT.md` — ECO ciblée de J20/Volume−.
3. `CURRENT.md` — index et avertissement.
4. Le présent CDC de mission.

## Contraintes

- Toute broche non mesurée est **présumée** et jamais marquée testée.
- Les noms GPIO proviennent du pinout Radxa V1.21 ; le Device Tree Android prime pour le pinmux effectivement utilisé.
- Éviter la réintroduction de l'ancien Volume− sur J20.32 dans les consignes actuelles ; conserver l'historique explicitement.
- Pas d'injection de 3,3 V dans un tactile 1,8 V ; pas de pont résistif directement sur GPIO numérique.
- Garder hors périmètre de cette ECO l'architecture Power V0.9 et son BOM.

## Briques et tests définis avant écriture

1. **Sources & inventaire :** compter 40 contacts, 28 GPIO, 7 endommagés dont 6 GPIO, identifier les 3 contacts 1/17/32 ; test par cohérence du pinout et des comptes.
2. **Allocation :** vérifier que la liste active des fonctions ne dépend pas des broches 13/17/26/28/32/38/40 ; test de collision des affectations J20, maintien 3/5/29/36/37, présence SPI12 CS0.
3. **CDC & index :** vérifier toutes les références Volume−, Power et J20 ; test documentaire des mentions de statut « à valider », liens vers la note, absence de modification des branches de puissance.
4. **Intégration GitHub :** comparer les quatre fichiers ciblés, créer un seul commit cohérent, pousser main en conservant le parent de branche ; relire le commit et les fichiers par l'API.

## Critères de validation

- Documentation exacte, prudente et exploitable après un futur test de continuité et un futur portage Android ; quatre fichiers seuls modifiés ; aucun résultat physique inventé.
- La mission **documentation** peut être terminée après tests documentaires et sauvegarde GitHub.
- La mission **réparation/qualification matérielle** reste ouverte jusqu'à validation physique, Android et non-régression.

## Risques de régression

- Conflit pinmux et pilotes Android, notamment SPI12 et GPIO57.
- Dommages cachés ou court-circuit autour de J20 malgré les broches jugées intactes.
- Rupture de référence 3,3 V et back-power pendant MAIN OFF.
- Mauvaise adaptation niveau 1,8/3,3 V du tactile.
- Confusion entre numérotation physique J20, GPIO Qualcomm et GPxx du RP2040.
