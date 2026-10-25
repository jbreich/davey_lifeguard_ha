# Pompes à chaleur, Bluetooth et AWS IoT

## Pompes à chaleur

Deux familles de pompes à chaleur sont prises en charge. **Aucune ne se pilote par l'API
Lifeguard** : seule l'association est enregistrée chez Davey.

### Nirvana (et Missouri)

API tierce, dont l'URL de base est :

```
https://ws.nirvanahp.com:8010/WSNirvana/webresources/entitie.iccards/
```

| Méthode | Chemin | Corps | Rôle |
|---|---|---|---|
| GET | `findAllParametersByCardId/{cardId}` | — | Tous les paramètres de la carte Wi-Fi de la pompe (tableau : `parameterId.parameterName`, `parameterValue`, etc.) |
| PUT | `sendCommand/{uid}` | `{"cmdName": "SetHeating", "cmdValue": "mode:POL,spaTemp:<T>,polTemp:<T>"}` | Change la consigne de température, en forçant le mode piscine |

L'API Nirvana ne demande **aucune authentification**.

Paramètres utiles de `findAllParametersByCardId` (`parameterId.parameterName` → `parameterValue`) :

| Paramètre | Sens |
|---|---|
| `polTemp` / `spaTemp` | consigne piscine / spa |
| `currentTemp` | température mesurée |
| `polPower` / `spaPower` | `"1"` = en marche |
| `PS_Flag` | `"0"` = mode piscine, sinon spa |
| `FC_Flag` | unité (Celsius si vrai ou différent de `"0"`) |

Association : l'identifiant (« MAC code ») de la carte Wi-Fi sert de `cardId`. Si la pompe
répond et qu'elle est en marche, l'associer avec `POST /device/register/heatpump` et
`{cardId: id, uid: id}`. Le champ `heatPump` de `/device/me` contient ensuite cet identifiant.
Les pompes de marque **Missouri** utilisent le même protocole que Nirvana.

### Aquark

Les pompes Aquark sont des appareils **Tuya** (cloud Tuya, région Europe `m1.tuyaeu.com`). La
consigne est le point de données Tuya **106**, ou **104** sur les pompes récentes. Marche/arrêt
et mode silencieux existent aussi, mais leurs codes ne sont pas documentés. Côté Home Assistant,
ces pompes se gèrent avec une intégration Tuya (officielle ou *LocalTuya*), pas avec l'API
Lifeguard.

### Consigne de température

En plus de la commande envoyée à la pompe, la consigne est envoyée à Lifeguard avec
`POST /device/change-target` et `targets = { temp: X, … }` (sans `ph`, `orp` ni `salinity`).
Elle apparaît ensuite dans `tempTarget`.

## Bluetooth (BLE)

Le BLE sert **uniquement à configurer le Wi-Fi** du Lifeguard à l'installation. Il n'y a **pas**
de pilotage local par BLE.

| UUID court | Usage |
|---|---|
| `FF01` | numéro de série |
| `FF02` | SSID du Wi-Fi |
| `FF03` | reset du module Wi-Fi |
| `FF04` | mot de passe du Wi-Fi |
| `FF05` | état de la connexion Wi-Fi |

Format d'écriture :
- SSID et mot de passe : octets UTF-8 bruts ;
- reset Wi-Fi : un seul octet `0x03`.

L'appareil est trouvé par son **nom Bluetooth**, qui contient `dwlg-` ou `hwqh-` et vaut le
numéro de série.

Séquence complète :
1. `POST /user/add-device` (liaison au compte) ;
2. écriture FF02 (SSID), 1 s d'attente ;
3. écriture FF04 (mot de passe), 1 s d'attente ;
4. écriture FF03 (`0x03`) ;
5. attendre que l'appareil passe en ligne, et considérer un échec au bout de 60 s. Lire FF05
   n'est pas nécessaire.

Contraintes (manuel Davey) : Wi-Fi **2,4 GHz** uniquement, SSID et mot de passe de
**18 caractères** au plus, débit conseillé d'au moins 12 Mbit/s au niveau de l'appareil.

## AWS IoT

- Chaque Lifeguard a une identité **AWS IoT Core** (`certArn` dans `/device/me`, région `ap-southeast-2`).
- `GET /device/cert/?serialNumber=` génère et renvoie `certificatePem`,
  `keyPair.PrivateKey/PublicKey` et `RootCA` (usage à l'installation, supposé).
- Le pilotage passe par l'API REST, qui relaie vers l'appareil via AWS IoT. Un client n'a pas
  besoin de parler MQTT.
- Conséquence : il n'existe **aucune API locale documentée**. La « commande locale » vantée par
  Davey désigne l'écran LCD de l'appareil.
- Les mises à jour du firmware arrivent aussi par le cloud (OTA). Voir
  [models-and-firmware.md](models-and-firmware.md#mise-à-jour-du-firmware).
