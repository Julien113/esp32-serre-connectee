# Firmware ESPHome de la serre

| Fichier | Rôle |
|---|---|
| `serre.yaml` | Config principale : capteur, écran, réglages, régulation |
| `plages.yaml` | **Tes plages** de température et d'humidité, souhaitées et critiques |
| `materiel/ventilateur_4fils.yaml` | Pilotage d'un Noctua PWM 4 fils (par défaut) |
| `materiel/ventilateur_3fils.yaml` | Pilotage d'un Noctua 3 fils via MOSFET |
| `secrets.yaml` | Exemple de secrets Wi-Fi et clé API, à remplir |
| `CABLAGE.md` | Schémas de câblage |

## Installation

1. Dans Home Assistant, installer l'add-on **ESPHome Device Builder**.
2. Copier ces fichiers dans le dossier `esphome/` de Home Assistant (en gardant le sous-dossier `materiel/`).
3. Remplir `secrets.yaml` avec ton Wi-Fi et une clé API neuve (`openssl rand -base64 32`).
4. Premier flash par câble USB depuis le Device Builder ou https://web.esphome.io ; les suivants passent par le Wi-Fi.
5. Home Assistant détecte l'appareil « Serre » et propose de l'ajouter.

## Plages (fichier `plages.yaml`)

- **Plage souhaitée** (22-28 °C, 60-80 %) : l'écran affiche OK dedans, et une étiquette FROID, CHAUD, SEC ou HUMIDE en dehors. Au-dessus du maximum, le ventilateur accélère.
- **Plage critique** (15-32 °C, 40-95 %) : au-delà pendant 2 minutes, l'écran affiche un panneau attention clignotant et l'entité `Alerte critique` passe à Activé.

Après une modification de `plages.yaml`, relancer `esphome run serre.yaml`.

## Fonctionnement de la régulation (sur l'ESP32)

Toutes les 10 secondes, la vitesse est recalculée :

1. **Base** : le ventilateur tourne en continu à la vitesse de base (25 %) pour brasser l'air contre la moisissure.
2. **Plage souhaitée** : au-dessus du maximum de température ou d'humidité, il accélère progressivement et atteint 100 % à +3 °C ou +10 % d'humidité.
   Il accélère aussi quand le VPD passe sous 0,5 kPa (air saturé) et atteint 100 % à 0,2 kPa. Un VPD trop haut ne change rien : ventiler assècherait encore plus l'air.
3. **Renouvellement** : 15 minutes à 100 % toutes les 3 heures, entre 8 h et 20 h.
4. **Capteur muet** : au moins 50 % par précaution.
5. **Mode manuel** : vitesse imposée, sauf forte chaleur où il repasse à 100 %.

Les réglages du ventilateur (vitesse de base, renouvellement, horaires, mode manuel) apparaissent dans Home Assistant et sont mémorisés dans l'ESP32 : ils survivent à un redémarrage et s'appliquent sans Home Assistant ni Wi-Fi.
Sans Wi-Fi, l'ESP32 ne connaît pas l'heure : le renouvellement continue alors toutes les 3 heures, jour et nuit.

## Écran

Les écrans défilent dans l'ordre de la liste `ecrans` de `plages.yaml`, chacun pendant `ecran_duree` secondes (15 par défaut). Pour masquer un écran, retire son nom de la liste ; pour changer l'ordre, déplace-le.

- **temps_reel** : à gauche la température, l'humidité, l'icône du ventilateur qui tourne plus ou moins vite et sa vitesse en %. À droite l'état de la température, de l'humidité et le mode du ventilateur (AUTO, RENOUV., MANUEL).
- **moyennes** : moyennes sur 24 h de la température, de l'humidité et du VPD, avec la tendance sur 30 min (hausse, baisse ou stable). Les seuils de « stable » sont dans `plages.yaml`.
- **graphe_temp**, **graphe_hum**, **graphe_vpd**, **graphe_ventil** : courbe sur les dernières `graphe_duree` heures (6 par défaut, 96 points). En haut la valeur actuelle, à gauche le maximum (en haut) et le minimum (en bas) de la période. L'historique est en mémoire : il repart de zéro après un redémarrage, le premier tracé apparaît au bout de 2 points (environ 8 min pour 6 h).
- **Alerte critique** : l'écran reste sur temps_reel et la colonne droite clignote avec un panneau attention.
- **Veille** : écran éteint de minuit à 7 h 30 (réglable dans `plages.yaml`), sauf en cas d'alerte critique.

## VPD

Le VPD (déficit de pression de vapeur, en kPa) mesure à quel point l'air « tire » l'eau des feuilles. Il combine température et humidité.
- Sous 0,4 kPa : air saturé, les plantes transpirent peu et la moisissure guette.
- 0,6 à 1,0 kPa : zone confortable pour la plupart des plantes tropicales.
- Au-dessus de 1,2 kPa : air trop sec, les plantes ferment leurs stomates et souffrent.

## Entités pour Home Assistant

- `État température` et `État humidité` : OK, Trop froid, Trop chaud, Trop sec, Trop humide.
- `Alerte critique` (avec le détail `Température critique` et `Humidité critique`) : à utiliser pour les notifications sur le téléphone.
- `VPD`, `VPD moyen 24h`, et `Tendance température`, `Tendance humidité`, `Tendance VPD`.
- Moyenne, minimum et maximum sur 24 h glissantes, pour la température et l'humidité. Ils repartent de zéro au redémarrage de l'ESP32 ; l'historique long reste dans Home Assistant.
- `Capteur en défaut` et `Connectée` : panne du BME280 ou de l'ESP32.
