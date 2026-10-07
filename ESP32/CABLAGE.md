# Câblage de la serre

## Alimentation

```
 Bloc secteur 12 V (1 A suffit)
   +12V ───────────────────────────────► ventilateur +12V (broche 2)
   GND  ──┬────────────────────────────► masse commune
          │
 ESP32 :  alimenté par son port USB (chargeur 5 V)
          ou par un petit convertisseur 12 V → 5 V branché sur VIN
          GND de l'ESP32 ─────────────► masse commune  (OBLIGATOIRE)
```

Les masses du 12 V et de l'ESP32 doivent être reliées, sinon la commande du ventilateur ne fonctionne pas.
Ne jamais envoyer le 12 V sur une broche de l'ESP32.

## Capteur BME280 et écran OLED (deux bus I2C séparés)

```
 ESP32            BME280            OLED 1,3"
 3V3  ─────────── VCC/VIN ───────── VCC      (le 3V3 reste partagé)
 GND  ─────────── GND ───────────── GND
 GPIO21 (SDA) ─── SDA
 GPIO22 (SCL) ─── SCL
 GPIO32 (SDA) ───────────────────── SDA
 GPIO33 (SCL) ───────────────────── SCL
```

Ne pas utiliser GPIO12 pour l'écran : c'est une broche lue au démarrage, et les résistances de rappel de l'écran empêcheraient l'ESP32 de démarrer.
L'ordre des broches de l'écran varie selon les séries (GND-VCC-SCL-SDA ou VCC-GND-SCL-SDA) : suivre ce qui est imprimé sur la carte.
Adresses attendues : BME280 en 0x76 (ou 0x77), OLED en 0x3C. Au démarrage, le journal ESPHome affiche les adresses trouvées sur chaque bus.

## Identifier le modèle de ventilateur

Compter les broches du connecteur du Noctua :
- **4 broches** : version PWM (NF-R8 redux-1800 PWM). Utiliser `materiel/ventilateur_4fils.yaml` (réglage par défaut).
- **3 broches** : version sans PWM. Dans `serre.yaml`, commenter la ligne 4 fils et décommenter la ligne 3 fils.

Broches du connecteur, en partant du côté du détrompeur : 1 masse, 2 +12 V, 3 tachymètre, 4 PWM.

## Variante A : ventilateur 4 fils (PWM)

Le 12 V alimente le ventilateur en permanence ; l'ESP32 ne fait que donner la consigne de vitesse.

```
 +12V ──────────────────────────── broche 2 (+12V)
 GND  ──────────────────────────── broche 1 (masse)

                         ┌──────── broche 4 (PWM)
                         │ C
 GPIO25 ──[ 1 kΩ ]────B─┤  NPN (BC547, 2N2222...)
                         │ E
 GND  ───────────────────┘

 3V3 ──[ 10 kΩ ]──┬────────────── broche 3 (tachymètre)
                  └── GPIO27
```

Le transistor protège l'ESP32 de la tension de rappel interne du ventilateur (jusqu'à 5 V) et inverse le signal, ce que la config compense (`inverted: true`).
Le tachymètre est optionnel : sans lui, tout fonctionne, seule l'entité « Régime ventilateur » reste à 0.

## Variante B : ventilateur 3 fils

L'ESP32 hache l'alimentation du ventilateur côté masse avec un MOSFET.

```
 +12V ────────┬─────────────────── broche 2 (+12V)
              │
            ──┴──  diode 1N4148 ou 1N5819
             ▲     (trait de la diode côté +12V)
              │
              ├─────────────────── broche 1 (masse du ventilateur)
              │ D
 GPIO25 ──[ 220 Ω ]──┬── G   MOSFET logique (IRLZ44N, AO3400...)
                     │      │ S
                [ 10 kΩ ]   │
                     │      │
 GND  ───────────────┴──────┘

 broche 3 (tachymètre) : non branchée
```

Le ventilateur consomme environ 0,1 A : n'importe quel MOSFET « logic level » convient, sans dissipateur.
La config impose une vitesse minimale de 40 % pour que le ventilateur ne cale pas, et 0 % l'arrête complètement.

## Récapitulatif des broches ESP32

| Broche ESP32 | Relié à |
|---|---|
| 3V3 | VCC du BME280 et de l'OLED, résistance de 10 kΩ du tachymètre |
| GND | masses du BME280, de l'OLED, du transistor et du bloc 12 V |
| GPIO21 | SDA du BME280 |
| GPIO22 | SCL du BME280 |
| GPIO32 | SDA de l'écran OLED |
| GPIO33 | SCL de l'écran OLED |
| GPIO25 | commande du ventilateur (transistor ou MOSFET) |
| GPIO27 | tachymètre (variante 4 fils seulement) |

## Placement dans la Milsbo

- Mettre le BME280 au milieu de la serre, à l'abri du flux direct du ventilateur et de la lumière des lampes, sinon il mesure l'air entrant et non l'air des plantes.
- Garder l'ESP32 et le bloc 12 V hors de l'humidité, idéalement à l'extérieur de la vitrine.
- Le ventilateur souffle vers l'intérieur : prévoir une ouverture de sortie de l'autre côté, en haut, pour que l'air chaud et humide s'échappe.
