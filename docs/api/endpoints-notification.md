# Endpoints `/notification/*`

## `GET /notification/notifications/{sn}` ✅

Historique des alertes de l'appareil, des plus récentes aux plus anciennes. L'appareil testé en
renvoyait 69, sur environ 5 mois : **pas de pagination observée**.

```json
[
  {
    "id": 1836226,
    "type": "PH_WARN",
    "errorId": 1005,
    "action": "fix",
    "isRead": true,
    "value": 7.8,
    "createdAt": "2026-10-18T11:35:32.155Z",
    "updatedAt": "2026-10-24T09:28:27.773Z",
    "DeviceId": 10000,
    "UserId": null
  }
]
```

| Champ | Type | Description |
|---|---|---|
| `id` | int | Identifiant de la notification |
| `type` | string | Catégorie (voir ci-dessous) |
| `errorId` | int | Code précis (voir [errors-and-thresholds.md](errors-and-thresholds.md#codes-errorid)) |
| `action` | string | `"fix"` est la seule valeur observée |
| `isRead` | bool | Lue ou non. Passe à `true` via `read-notification` |
| `value` | number | Valeur mesurée au moment de l'alerte |
| `setpoint` | string | Consigne au moment de l'alerte. Champ possible, jamais observé |
| `DeviceId`, `UserId` | int / null | Liens |

Valeurs de `type` observées : `PH_ERROR`, `PH_WARN`, `ORP_WARN`, `SALT_ERROR`, `FLOW_ERROR`,
`ERROR_CALIBRATE`. Valeurs connues mais non observées : `ORP_ERROR`, `SALT_WARN`, `WIFI_ERROR`,
`ERROR_CLEAN`, `CLEAN_ERROR`.

Exemple de répartition sur le compte testé :

| type | errorId | nombre |
|---|---|---|
| ERROR_CALIBRATE | 1016 | 27 |
| PH_WARN | 1008 | 14 |
| ERROR_CALIBRATE | 1015 | 6 |
| PH_ERROR | 1003 | 5 |
| ORP_WARN | 1006 | 5 |
| PH_WARN | 1005 | 4 |
| FLOW_ERROR | 1012 | 4 |
| SALT_ERROR | 1010 | 3 |
| PH_ERROR | 1004 | 1 |

> Pour Home Assistant, cette route permet de créer un **capteur « dernière alerte »** et un
> **compteur d'alertes non lues**, sans rien modifier côté serveur.

Usage conseillé : lire toute la liste, filtrer sur les **7 derniers jours** pour un affichage
« récent », puis compter les non-lues (`isRead == false`). Les textes s'obtiennent via
`/notification/errors/{lang}`. Voir [behavior.md](behavior.md#5-alertes).

## `GET /notification/logs/{sn}` ✅

Moyennes **journalières** des 10 derniers jours, plus récentes en premier.

```json
[
  { "ph": 7.6, "orp": 664, "salinity": 4333, "date": "2026-10-24T00:00:00.000Z" },
  { "ph": 7.6, "orp": 678, "salinity": 4336, "date": "2026-10-23T00:00:00.000Z" }
]
```

Pas de température dans cet historique. Home Assistant garde de toute façon son propre
historique : cette route n'a d'intérêt que pour reconstituer les jours où Home Assistant était
arrêté.

## `POST /notification/errors/{language}` (non testé)

Renvoie les textes d'aide (titre, sous-titre, étapes de dépannage) pour une liste d'`errorId`.
`language` est un code de langue : `en`, `de`, `es`, `fr`, `it`, `nl` ou `pt` (`en` par
défaut). C'est un POST, mais en pratique une simple **lecture** de référentiel.

Requête :

```json
{ "ids": [1003, 1012] }
```

Réponse :

```json
[
  {
    "id": 1003,
    "type": "PH_ERROR",
    "title": "…",
    "subtitle": "…",
    "message": "…",
    "imageName": "…",
    "troubleshooting": ["…", "…"]
  }
]
```

## `GET /notification/read-notification/{id}` ⚠️

Marque la notification `id` comme lue et renvoie l'objet `Notification`. C'est un **GET avec
effet de bord**. L'état « lu » est stocké côté serveur et vaut pour tous les utilisateurs : ne
l'appeler que sur une action explicite de l'utilisateur.
