# Firmware ESPHome de la serre

| Fichier | Rôle |
|---|---|
| `serre.yaml` | Config principale : capteur, écran, réglages, régulation |
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

## Fonctionnement de la régulation (sur l'ESP32)

Toutes les 10 secondes, la vitesse est recalculée :

1. **Base** : le ventilateur tourne en continu à la vitesse de base (25 %) pour brasser l'air contre la moisissure.
2. **Seuils** : au-dessus de 28 °C ou de 85 % d'humidité, il accélère progressivement et atteint 100 % à +3 °C ou +10 % d'humidité.
3. **Renouvellement** : 15 minutes à 100 % toutes les 3 heures, entre 8 h et 20 h.
4. **Capteur muet** : au moins 50 % par précaution.
5. **Mode manuel** : vitesse imposée, sauf forte chaleur où il repasse à 100 %.

Tous ces réglages apparaissent dans Home Assistant (section Configuration de l'appareil) et sont mémorisés dans l'ESP32 : ils survivent à un redémarrage et s'appliquent sans Home Assistant ni Wi-Fi.
Sans Wi-Fi, l'ESP32 ne connaît pas l'heure : le renouvellement continue alors toutes les 3 heures, jour et nuit.

## Entités pour les alertes

- `Trop chaud` : au-dessus du seuil + 3 °C pendant 10 minutes, la ventilation ne suffit plus.
- `Trop froid` : sous 22 °C pendant 10 minutes.
- `Trop sec` : sous 60 % pendant 30 minutes.
- `Capteur en défaut` et `Connectée` : panne du BME280 ou de l'ESP32.

Il suffit de créer dans Home Assistant une automatisation « quand l'entité passe à Activé, envoyer une notification au téléphone ».
