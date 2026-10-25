# Modèles de données

Schéma des objets JSON échangés avec l'API. Les types indiqués sont ceux **observés côté
serveur** pour les routes testées, et ceux attendus pour les autres.

## Device

Réponse de `GET /device/me`, et réponse des routes d'écriture (`change-target`,
`change-mode`, `edit-schedules`, `update`, `add-device`). Détail des champs :
[endpoints-device.md](endpoints-device.md#get-deviceme-).

```
id: int                    serialNumber: string        firmwareVersion: string
nipperVersion: string      isConnected: bool           registered: bool|null
inUse: bool                phTarget: double            orpTarget: int
tempTarget: number         cellOutput: int             vsdPumpSpeed: int
isVsdPumpConnected: bool   boostDuration: int (h)      backwashDuration: int (s)
modes: string[]            schedules: Schedule[]       poolSize: int (L)
nipperCell: int (g/h)      brand: int                  region: int (0 AU,1 EU,2 US)
saltStatus: int (0 normal,1 low)                        io: string (8 bits)
filtrationPump: int|null   acidPump: bool|null         heatPump: string|null
token: string 🔒           certArn: string 🔒          certData: Certificate|null 🔒
createdAt / updatedAt: ISO-8601
```

Le sens de `inUse` n'est pas connu. Le sens des champs matériels est détaillé dans
[models-and-firmware.md](models-and-firmware.md#2-champs-de-deviceme-qui-décrivent-le-matériel).

## Status

Réponse de `GET /device/status/{sn}`.

```
ph: double          orp: int          temp: int          salinity: int
cellOutput: number  version: string   flow: int
phConnected / orpConnected / salinityConnected / tempConnected: bool
phError / orpError / saltError / flowError: string ("NORMAL" | "WARN" | "ERROR" | "")
```

## Schedule

```
on: int|string   off: int|string   days: string(7)   speed: int|string
```
Ce sont des chaînes en lecture et des entiers en écriture. Voir [modes-and-schedules.md](modes-and-schedules.md#plannings-de-filtration-schedules).

## Corps de `POST /device/change-target`

```
{
  serialNumber: string,
  targets: {
    // consignes : facultatives, un champ absent n'est pas modifié
    ph: double, orp: int, salinity: int (toujours 0), temp: double, cell: int,
    // état des sondes et réglages : toujours envoyés, avec les valeurs actuelles
    phConnected: bool, orpConnected: bool, salinityConnected: bool, tempConnected: bool,
    isVsdPumpConnected: bool, vsdPumpSpeed: int, boostDuration: int, backwashDuration: int
  }
}
```

## Corps de `POST /device/v2/change-mode`

```
{ serialNumber: string, modes: string[] }
```

## Corps de `POST /device/edit-schedules`

```
{ serialNumber: string, firmwareVersion: string, schedules: Schedule[] }
```

## Corps de `POST /device/update`

```
{ serialNumber: string, parameters: { acidPump: bool, heatPump: bool, firmawareVersion: string } }
```

## User

```
id: int                 firstname: string       lastname: string        email: string
privledgelevel: "ADMIN"|"USER"|"SUPER"          isadmin: bool|null      isdisabled: bool
isInvited: bool|null    isVerified: bool        verifyCode: string 🔒  password: string 🔒
allowsnotifications: bool                        pushToken: string 🔒
marketingmaterialok / updatesok: bool|null       city / country / language: string|null
organisationid: int|null                         deviceId / DeviceId: int
createdAt / updatedAt: ISO-8601
```

## Réponses de connexion

```
signin, register : { token, refreshToken, user: User }
auth             : { token, refreshToken, user: User, statusCode: int }
```

## Notification

```
id: int   type: string   errorId: int   action: string   isRead: bool
value: number   setpoint: string (jamais observé)   DeviceId: int   UserId: int|null
createdAt / updatedAt: ISO-8601
```

## Historique (`/notification/logs`)

```
{ date: ISO-8601 (minuit UTC), ph: double, orp: int, salinity: int }
```

## Textes d'aide (`/notification/errors/{lang}`)

```
requête : { ids: int[] }
réponse : [ { id: int, type: string, title: string, subtitle: string,
              message: string, imageName: string, troubleshooting: string[] } ]
```

## Certificat (`/device/cert/`) 🔒

```
{ certificateArn: string, certificatePem: string, RootCA: string,
  keyPair: { PublicKey: string, PrivateKey: string } }
```

## Invitations et droits

```
POST /user/invite            { deviceId: int, inviteEmail, userEmail }      → { sent: bool }
GET  /user/invite-code                                                       → { code: int }
POST /user/email-invite      { code: string, email, language }               → { success: bool }
POST /user/redeem-invite     { code: string }                                → Device
POST /user/update/invite     { accepted: bool, userId: string }              → User
POST /user/update-privledges { userId: string, privledgelevel: "ADMIN"|"USER" } → User
POST /user/add-device        { serialNumber: string, userId: string }        → Device
```

## Compte

```
POST /user/register            { email, password, firstname, lastname, ismarketingok: bool }
POST /user/add-push/           { pushToken: string, deviceType: string, allow: bool } → User
POST /user/v2/reset            { email }                   → { success: bool }
POST /user/v2/update/password  { email, ofl, password }    → { message }
GET  /user/delete/{id}         → { deleted: bool }
GET  /user/detach-device/{id}  → { removed: bool }
```

## Pompes à chaleur

```
POST /device/register/heatpump         { cardId: string, uid: string }
Nirvana PUT sendCommand/{uid}          { cmdName: "SetHeating", cmdValue: "mode:POL,spaTemp:<T>,polTemp:<T>" }
Nirvana GET findAllParametersByCardId/{cardId}
  → [ { parameterId: { parameterID, parameterName, parameterDescription, parameterType, parameterValue, … },
        pieceID: { pieceID, pieceName, pieceSn, pieceAddress, cardID, … },
        piecesParameterPK: { parameterID, pieceID }, … } ]
```

🔒 = donnée sensible, à ne jamais journaliser.
