# Endpoints `/device/*`

Tous ces endpoints demandent l'en-tête `Authorization: <token>`.
`{sn}` est le numéro de série de l'appareil, de la forme `dwlg-XXXXXXXX`.

---

## `GET /device/me` ✅

Renvoie l'appareil associé au compte connecté. **Un seul appareil** par utilisateur : l'API ne
propose pas de liste. Le compte est lié via `user.deviceId`.

Exemple réel (anonymisé) :

```json
{
  "id": 10000,
  "serialNumber": "dwlg-00000000",
  "firmwareVersion": "04.01.04",
  "nipperVersion": "05.01.02",
  "isConnected": true,
  "registered": null,
  "inUse": true,

  "phTarget": 7.4,
  "orpTarget": 650,
  "tempTarget": 25,
  "cellOutput": 100,
  "vsdPumpSpeed": 8,
  "isVsdPumpConnected": true,
  "boostDuration": 24,
  "backwashDuration": 120,

  "modes": ["WINTER"],
  "schedules": [
    { "on": "21600", "off": "32400", "days": "1111111", "speed": "10" },
    { "on": "61200", "off": "79200", "days": "1111111", "speed": "10" }
  ],

  "poolSize": 23000,
  "nipperCell": 25,
  "brand": 0,
  "region": 1,
  "saltStatus": 0,
  "io": "00001111",

  "filtrationPump": null,
  "acidPump": null,
  "heatPump": null,

  "token": "<token appareil>",
  "certArn": "arn:aws:iot:ap-southeast-2:<aws-account>:cert/<id>",
  "certData": null,
  "createdAt": "2025-12-26T10:44:01.877Z",
  "updatedAt": "2026-10-24T22:23:03.905Z"
}
```

| Champ | Type | Description |
|---|---|---|
| `id` | int | Identifiant interne de l'appareil (= `user.deviceId`) |
| `serialNumber` | string | Numéro de série, à utiliser dans toutes les autres routes |
| `firmwareVersion` | string `AA.BB.CC` | Firmware du contrôleur Lifeguard (voir [models-and-firmware.md](models-and-firmware.md#3-firmwares)) |
| `nipperVersion` | string | Firmware de l'électrolyseur (EcoSalt2 / Nipper) |
| `isConnected` | bool | L'appareil est en ligne côté cloud |
| `phTarget` | float | Consigne pH |
| `orpTarget` | int | Consigne ORP (mV) |
| `tempTarget` | float/int | Consigne de température de l'eau, en °C (utilisée avec une pompe à chaleur) |
| `cellOutput` | int (%) | **Consigne** de production de la cellule sans sonde ORP. À ne pas confondre avec `status.cellOutput`, qui est la production instantanée |
| `vsdPumpSpeed` | int | Vitesse **par défaut** de la pompe à vitesse variable (VSD). Valeurs valides : 2, 4, 6, 8, 10 |
| `isVsdPumpConnected` | bool | Une pompe VSD est déclarée |
| `boostDuration` | int (heures) | Durée du mode BOOST : 6, 12, 18 ou 24 (24 par défaut) |
| `backwashDuration` | int (**secondes**) | Durée du mode BACKWASH : 2, 3, 4 ou 5 min, soit 120 à 300 s (120 par défaut) |
| `modes` | string[] | Modes actifs (voir [modes-and-schedules.md](modes-and-schedules.md)) |
| `schedules` | Schedule[] | Plannings de filtration. **Attention** : dans la réponse, `on`, `off` et `speed` sont des **chaînes** |
| `poolSize` | int (litres) | Volume du bassin, utilisé pour calculer la quantité de sel à ajouter |
| `nipperCell` | int (g/h) | Taille de la cellule (15 / 25 / 35) |
| `brand` | int | Marque / gamme (0 sur l'appareil testé) |
| `region` | int | `0` AUSTRALIA, `1` EUROPE, `2` USA |
| `saltStatus` | int | `0` NORMAL (3000–6000 ppm), `1` LOW_SALT (1500–6000 ppm) |
| `io` | string (8 bits) | Bits 4 à 7 : présence des sondes pH, ORP, sel, température. Bits 0 à 3 : sens inconnu |
| `filtrationPump` | int / null | Type de pompe de filtration |
| `acidPump` | bool / null | Pompe doseuse d'acide présente |
| `heatPump` | string / null | Pompe à chaleur associée (identifiant) |
| `inUse` | bool | Sens inconnu |
| `token`, `certArn`, `certData` | — | Identité AWS IoT de l'appareil : **sensible** |

---

## `GET /device/status/{sn}` ✅

Mesures instantanées. C'est la route à interroger régulièrement.

```json
{
  "ph": 7.4,
  "orp": 704,
  "temp": 16,
  "salinity": 4462,
  "cellOutput": 0,
  "version": "04.01.04",
  "phConnected": true,
  "orpConnected": true,
  "salinityConnected": true,
  "tempConnected": true,
  "flow": 0,
  "phError": "NORMAL",
  "orpError": "NORMAL",
  "saltError": "NORMAL",
  "flowError": "NORMAL"
}
```

| Champ | Type | Description |
|---|---|---|
| `ph` | float | pH mesuré |
| `orp` | int (mV) | Potentiel redox mesuré |
| `temp` | int (°C) | Température de l'eau (**entier**) |
| `salinity` | int (ppm) | Salinité |
| `cellOutput` | number (%) | Production instantanée de la cellule |
| `version` | string | Version du firmware (identique à `firmwareVersion`) |
| `*Connected` | bool | Présence / activation de chaque sonde |
| `flow` | int | `0` = pas de circulation d'eau. Dans ce cas, les mesures ne sont pas fiables : l'intégration garde alors les dernières valeurs connues |
| `phError`, `orpError`, `saltError`, `flowError` | string | `"NORMAL"`, `"WARN"`, `"ERROR"` ou `""` (sans objet). Seul `NORMAL` a été observé |

> `/device/me` et `/device/status` ont tous les deux une clé `cellOutput`, avec des sens
> différents (consigne / mesure). Les fusionner avec `me | status`, comme le fait
> l'intégration, **écrase la consigne par la mesure**.

---

## `GET /device/users/{sn}` ✅

Liste des utilisateurs ayant accès à l'appareil : tableau d'objets `User`, voir
[data-models.md](data-models.md#user). Le serveur y renvoie aussi `password`, `verifyCode` et
`pushToken`.

---

## `POST /device/change-target` (écriture, non testé)

Modifie les consignes. Corps :

```json
{
  "serialNumber": "dwlg-00000000",
  "targets": {
    "ph": 7.4,
    "orp": 650,
    "salinity": 0,
    "temp": 25.0,
    "phConnected": true,
    "orpConnected": true,
    "salinityConnected": true,
    "tempConnected": true,
    "isVsdPumpConnected": true,
    "vsdPumpSpeed": 8,
    "boostDuration": 24,
    "backwashDuration": 120
  }
}
```

| Champ `targets` | Type | Valeur à envoyer |
|---|---|---|
| `ph` | Double | `phTarget` actuel, ou la nouvelle valeur (7.0 à 7.8, par pas de 0.1) |
| `orp` | Integer | `orpTarget` actuel, ou la nouvelle valeur (350 à 980 mV, par pas de 10) |
| `salinity` | Integer | **Toujours `0`** : l'API n'a pas de vraie consigne de sel |
| `temp` | Double | `tempTarget` (15 à 39 °C, par pas de 1). Pour ne changer que la température, on peut n'envoyer que `temp`, sans `ph`, `orp` ni `salinity` |
| `cell` | Integer | Consigne de production de la cellule en % (0 à 100, par pas de 5), seulement sans sonde ORP. **Le nom est `cell`, pas `cellOutput`** |
| `*Connected` | bool | Recopiés depuis `/device/status`. Changer l'un d'eux **active ou désactive la sonde** correspondante |
| `isVsdPumpConnected` | bool | Déclaration d'une pompe VSD |
| `vsdPumpSpeed` | int | Vitesse VSD par défaut : 2 / 4 / 6 / 8 / 10 |
| `boostDuration` | int (h) | 6 / 12 / 18 / 24 |
| `backwashDuration` | int (s) | 120 / 180 / 240 / 300 |

Les consignes (`ph`, `orp`, `salinity`, `temp`, `cell`) peuvent être omises. Les drapeaux
`*Connected` et les réglages VSD / durées doivent **toujours** être envoyés avec leurs valeurs
actuelles (voir [business-rules.md](business-rules.md#25-mise-à-jour-partielle-et-pièges)).

Réponse : un objet `Device` (comme `/device/me`) mis à jour.

---

## `POST /device/v2/change-mode` (écriture, non testé)

Remplace **toute** la liste des modes actifs. Corps :

```json
{ "serialNumber": "dwlg-00000000", "modes": ["SCHEDULE", "WINTER"] }
```

Le mode qu'on vient d'activer est inséré **en tête** de liste.
Valeurs et règles : voir [modes-and-schedules.md](modes-and-schedules.md). Réponse : `Device`.

---

## `POST /device/edit-schedules` (écriture, non testé)

Remplace **tous** les plannings. Corps :

```json
{
  "serialNumber": "dwlg-00000000",
  "firmwareVersion": "04.01.04",
  "schedules": [
    { "days": "1111111", "on": 21600, "off": 32400, "speed": 10 },
    { "days": "1111111", "on": 61200, "off": 79200, "speed": 10 }
  ]
}
```

- Dans l'envoi, `on`, `off` et `speed` sont des **entiers**. Dans la lecture (`/device/me`), ce sont des chaînes.
- `on` / `off` : secondes depuis minuit (`heures*3600 + minutes*60`).
- `days` : 7 caractères `0`/`1`, du **lundi au dimanche**. Seule la valeur `1111111` a été observée.
- `speed` : vitesse VSD pendant le créneau (`10` observé).
- Nombre de créneaux : 2 au maximum.

Réponse : `Device`.

---

## `POST /device/update` (écriture, non testé)

Déclare des équipements annexes. Corps :

```json
{
  "serialNumber": "dwlg-00000000",
  "parameters": { "acidPump": true, "heatPump": false, "firmawareVersion": "04.01.04" }
}
```

> La faute `firmawareVersion` fait partie du format attendu.

---

## Routes d'enregistrement (⚠️ effets de bord — ne pas appeler)

| Méthode | Route | Paramètre | Réponse | Usage |
|---|---|---|---|---|
| GET | `/device/register/` | `?serialNumber=` | `Device` | Enregistre l'appareil pendant l'installation |
| GET | `/device/cert/` | `?serialNumber=` | `Certificate` (`certificatePem`, `keyPair.PrivateKey`, `RootCA`, `certificateArn`) | Génère et renvoie le certificat AWS IoT de l'appareil |
| GET | `/device/delete-certs` | `?serialNumber=` | `{ "deleted": bool }` | **Supprime** les certificats IoT : l'appareil perd sa connexion cloud |
| POST | `/device/register/heatpump` | `{ "cardId", "uid" }` | — | Associe une pompe à chaleur Nirvana |
