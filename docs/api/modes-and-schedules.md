# Modes et plannings

## Modes

Le champ `modes` de `/device/me` est une **liste de chaînes**. On la remplace en entier avec
`POST /device/v2/change-mode`.

### Modes pilotables

| Valeur | Nom Davey | Effet |
|---|---|---|
| `MANUAL` | Manual Mode / « System On/Off » | Marche / arrêt du système. La cellule produit à la consigne `cellOutput`. Si une pompe VSD Davey est reliée, elle tourne à `vsdPumpSpeed` |
| `SPA` | Spa Mode | Production réduite de 80 %. Activable seulement **sans sonde ORP** |
| `BOOST` | Boost Mode | Électrolyseur à 100 % pendant `boostDuration` heures (24 par défaut). L'ORP est ignoré pendant le boost |
| `COVER` | Cover Mode | Volet fermé. Production réduite de 80 %, activable seulement **sans sonde ORP**. Détection automatique avec un volet motorisé raccordé |
| `WINTER` | Winter Mode | Production ramenée à 85 % (hivernage actif) |
| `SCHEDULE` | Schedule Mode | Filtration selon les plannings (`schedules`). Non modifiable à distance en Europe |
| `PUMP`, `HEAT` | Pump Mode, Heat Mode | Valeurs connues, sans fonctionnement connu : ne pas utiliser |

### États

| Valeur | Source | Sens |
|---|---|---|
| `BACKWASH` | `modes`, renvoyé par l'appareil | Lavage du filtre en cours, pendant `backwashDuration` secondes |
| `FILTERING` | **calculé** : `status.flow == 1` | Filtration en cours |
| `CHLORINATING` | **calculé** : `status.cellOutput > 0` | Électrolyse en cours |
| `MAINTENANCE`, `SERVICE` | valeurs connues | Jamais observées |

> Le backwash se déclenche **sur l'appareil** : l'API ne permet que de régler sa durée.

### Règles à appliquer

Le détail, avec les conditions exactes, est dans [business-rules.md](business-rules.md). En résumé :

- Activer un mode = l'**ajouter en tête** de la liste, puis envoyer toute la liste. Désactiver = le retirer, puis envoyer toute la liste.
- **`SCHEDULE` et `BOOST` s'excluent.**
- **`SPA` et `COVER` sont sans effet quand une sonde ORP est connectée.**
- `BOOST`, `SPA`, `COVER` et `WINTER` sont réservés aux administrateurs (`ADMIN` ou `SUPER`). `MANUAL` et `SCHEDULE` sont ouverts à tous.
- Aucune commande quand l'appareil est hors ligne.
- En région Europe (`region == 1`), `SCHEDULE` et les plannings sont à traiter en lecture seule.
- La réponse de `change-mode` (un `Device`) fait foi : reprendre `response.modes`.

Exemple observé : `"modes": ["WINTER"]`.

---

## Plannings de filtration (`schedules`)

### Format

Lecture (`/device/me`) : les valeurs numériques sont des **chaînes**.

```json
"schedules": [
  { "on": "21600", "off": "32400", "days": "1111111", "speed": "10" },
  { "on": "61200", "off": "79200", "days": "1111111", "speed": "10" }
]
```

Écriture (`POST /device/edit-schedules`) : des **entiers**.

```json
{
  "serialNumber": "dwlg-00000000",
  "firmwareVersion": "04.01.04",
  "schedules": [ { "days": "1111111", "on": 21600, "off": 32400, "speed": 10 } ]
}
```

| Champ | Description |
|---|---|
| `on` | Heure de début, en secondes depuis minuit. `21600` = 06:00 |
| `off` | Heure de fin, en secondes depuis minuit. `32400` = 09:00 |
| `days` | 7 caractères `0`/`1` : position 0 = **lundi** … position 6 = **dimanche** |
| `speed` | Vitesse VSD pendant le créneau (2 / 4 / 6 / 8 / 10). Sans effet sans pompe Davey compatible |

Fonctionnement :
- `SCHEDULE` **actif** : la filtration démarre et s'arrête selon les créneaux. Le contrôleur
  ne commande la filtration qu'à travers une pompe Davey compatible. Sur les modèles européens,
  l'EcoSalt2 est de toute façon alimenté par une horloge externe (voir
  [models-and-firmware.md](models-and-firmware.md#pompe-vsd)).
- `SCHEDULE` **inactif** : seule l'heure de fin s'applique. Les créneaux ne servent plus qu'à
  **arrêter** la filtration.
- `edit-schedules` **remplace** la liste entière. Supprimer un créneau revient à renvoyer la
  liste sans lui, et supprimer le dernier envoie `[]`. Les heures sont à la minute près.

L'exemple observé correspond à deux créneaux par jour, tous les jours :
**06:00–09:00** et **17:00–22:00**, à la vitesse 10.

### Capacités selon le firmware

Aucune fonction connue de l'API ne dépend de la version du firmware. Repères non vérifiés :
plannings et pompe VSD à partir du firmware 1.3, choix des jours de la semaine à partir de la
4.x. Détail dans [models-and-firmware.md](models-and-firmware.md#3-firmwares).

En pratique : 2 créneaux au maximum.

Règles de saisie (voir [business-rules.md](business-rules.md#4-plannings-post-deviceedit-schedules)) :
`on < off` (pas de passage à minuit), pas de chevauchement. Seule la valeur `days = "1111111"`
a été observée. Pour la pompe VSD, se fier à `isVsdPumpConnected`.
