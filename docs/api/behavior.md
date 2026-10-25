# Fonctionnement de bout en bout

Ce document décrit **comment le système se comporte**, du point de vue d'un client de l'API.
Il sert de cahier des charges pour l'intégration Home Assistant. Les détails des routes sont
dans les fichiers `endpoints-*.md`, les combinaisons valides dans
[business-rules.md](business-rules.md).

Légende : ⚠️ point de vigilance · 🏠 recommandation pour Home Assistant.

---

## 1. Architecture

```
Client (mobile, Home Assistant…) ── HTTPS (REST, JSON) ──► API cloud Davey (AWS API Gateway, Sydney)
                                                                 │
                                                                 ▼ AWS IoT Core (certificat par appareil)
                                                            Lifeguard (contrôleur) ── électrolyseur EcoSalt2 / Nipper
                                                                 ▲
Téléphone ── Bluetooth LE (mise en service Wi-Fi uniquement) ────┘
```

- **Pas d'API locale.** Le Bluetooth sert seulement à transmettre le Wi-Fi. La « commande
  locale » dont parle Davey, c'est l'écran LCD et les boutons de l'appareil.
- Le serveur **mémorise les consignes et les modes**. L'appareil les reçoit ensuite par AWS IoT.
  Une écriture renvoie aussitôt l'objet `Device` à jour côté serveur. L'effet physique arrive
  plus tard, au bout de quelques secondes ou plus, et seulement si l'appareil est en ligne.
- Les **mesures** (`/device/status`) remontent de l'appareil vers le serveur. Le serveur les
  renvoie telles quelles, avec les états `*Error`.
- Les **alertes** sont produites côté serveur ou appareil, stockées (`/notification/notifications`)
  et envoyées en push (Firebase) aux téléphones enregistrés.
- 🏠 Home Assistant doit **interroger** l'API : il n'existe ni webhook, ni flux temps réel
  accessible à un tiers.

## 2. Session

### Connexion
1. Retirer les espaces autour de l'email et du mot de passe.
2. `POST /user/signin` → `token`, `refreshToken`, `user`.
3. Un échec renvoie un code 4xx. Les causes (email inconnu, mauvais mot de passe…) ne se
   distinguent pas de façon fiable : afficher un seul message « identifiants invalides ».

🏠 Ne stocker que les tokens et l'identifiant, jamais le mot de passe.

### Démarrage d'un client
- Rafraîchir avec `POST /user/auth {userId, refreshToken}` (refresh token stocké), puis
  sauvegarder le nouveau couple de tokens.
- La réponse contient `user`. Si `user.deviceId == 0`, le compte n'a pas d'appareil : il faut
  d'abord en lier un (§7). Sinon, lire `GET /device/me`.
- Si `/device/me` ne contient ni `id` ni `serialNumber`, les données de l'appareil sont
  incomplètes : traiter le compte comme sans appareil.
- Si le rafraîchissement échoue, redemander la connexion.

### Expiration du token
- Sur un `401`, rafraîchir avec le refresh token, puis rejouer la requête **une fois**, en
  **remplaçant** l'en-tête `Authorization`.
- Sauvegarder le nouveau couple de tokens. L'ancien refresh token semble rester valable après
  usage (supposé, non vérifié) : ne pas compter dessus.

🏠 Rafraîchir au plus une fois par requête, une seule fois pour des requêtes parallèles,
sauvegarder le nouveau couple, puis demander une nouvelle authentification si ça échoue encore
(déjà fait sur la branche `fix/security-hardening`).

### Requêtes et en-têtes
- `Authorization: <token>` sans préfixe, et `Content-Type: application/json`.
- Délai conseillé : 30 s (connexion, lecture et écriture).
- Requêtes **sans** `Authorization` : rafraîchissement (`/user/auth`), inscription, appels à
  l'API Nirvana. `/user/signin` ignore un en-tête vide ou périmé.
- Les erreurs se lisent au **code HTTP** : le corps des réponses d'erreur n'est pas documenté.

## 3. Lecture de l'état

| Source | Contenu | Quand la lire |
|---|---|---|
| `GET /device/me` | configuration : consignes, modes, plannings, durées, VSD, région, `isConnected`, firmwares | au démarrage, puis périodiquement. Chaque écriture renvoie déjà cet objet |
| `GET /device/status/{sn}` | mesures, état des sondes (`*Connected`), `*Error`, `flow`, production réelle `cellOutput` | périodiquement, et après chaque changement de consigne |
| `GET /notification/notifications/{sn}` | alertes | à un rythme lent |
| `GET /notification/logs/{sn}` | moyennes journalières pH / ORP / sel | rarement : les valeurs sont journalières |

Pour un suivi en direct, il suffit de lire `status` puis `me` en alternance, environ toutes
les 20 s.

### Règles d'interprétation
- **Une mesure n'est valable** que si les trois conditions sont réunies : valeur différente de
  `-9999` (sonde absente), `flow == 1` (débit présent) et `device.isConnected == true`. Sinon,
  il n'y a pas de valeur à afficher.
- **États par sonde** : `phError`, `orpError` et `saltError` sont calculés par le **serveur**
  (`NORMAL`, `WARN`, `ERROR`, `""` = sans objet) et font foi. La température n'a pas d'état
  d'erreur.
- **États calculés** : `FILTERING` = `flow == 1`, `CHLORINATING` = `status.cellOutput > 0`.
  `BACKWASH` vient de `device.modes`.
- **Deux `cellOutput`** : celui de `/device/me` est la **consigne** (% en production fixe),
  celui de `/device/status` est la **production réelle** à l'instant.
- **Wi-Fi de l'appareil** : `device.isConnected`. Quand il est faux, l'appareil est hors ligne
  et aucune commande ne peut lui parvenir : désactiver les contrôles.

🏠 Interroger `/device/me` et `/device/status` ensemble, toutes les 1 à 5 minutes. Rendre les
mesures `unknown` si l'une des trois conditions ci-dessus n'est pas remplie. Ne pas réutiliser
la dernière valeur connue, ou alors l'indiquer.

## 4. Commandes

Toutes les écritures renvoient l'objet `Device` complet après mise à jour : c'est lui qui fait
foi. Relire `/device/status` ensuite pour voir l'effet sur les mesures.

### 4.1 Modes — `POST /device/v2/change-mode {serialNumber, modes}`

`modes` **remplace** la liste complète. Pour activer ou désactiver un mode :

1. Partir des modes actifs de `/device/me`, en ne gardant **que les 6 pilotables** : `MANUAL`,
   `SCHEDULE`, `BOOST`, `SPA`, `COVER`, `WINTER`. `BACKWASH` et les autres états ne doivent
   **pas** être renvoyés.
2. Pour activer un mode, l'**ajouter en tête**. Pour le désactiver, le retirer.
3. Refuser l'activation de `SCHEDULE` si `BOOST` est actif, et inversement.
4. Envoyer la liste, puis reprendre `response.modes`, qui fait foi.

⚠️ Envoyer `["MANUAL"]` ou `[]` sans repartir des modes actifs **efface tous les autres modes**
(WINTER, BOOST, SCHEDULE…).

Sens des modes :

| Mode | Effet |
|---|---|
| `MANUAL` | Marche / arrêt du système (« System On/Off »). La cellule produit selon la consigne |
| `SCHEDULE` | Filtration selon les plannings. **Inactif**, seule l'heure de fin des créneaux s'applique : elle **arrête** la filtration |
| `BOOST` | Électrolyse à 100 % pendant `boostDuration` heures, puis arrêt automatique par l'appareil |
| `SPA` / `COVER` | Production −80 %, seulement **sans sonde ORP** |
| `WINTER` | Production ramenée à 85 % |

### 4.2 Consignes — `POST /device/change-target {serialNumber, targets}`

- Les champs de consigne `ph`, `orp`, `salinity`, `temp` et `cell` peuvent être omis : un champ
  absent n'est pas modifié (non testé).
- ⚠️ `phConnected`, `orpConnected`, `salinityConnected`, `tempConnected` et
  `isVsdPumpConnected` doivent **toujours être envoyés**. Ils commandent l'**activation des
  sondes** (§4.3) : les oublier ou les mettre à `false` désactive les sondes.
- ⚠️ `vsdPumpSpeed`, `boostDuration` et `backwashDuration` doivent aussi toujours être envoyés
  avec leurs valeurs actuelles. Des valeurs par défaut (10, 24, 120) risquent de réinitialiser
  ces réglages.
- Avant d'envoyer, vérifier que la valeur a changé et qu'elle est dans la plage valide (ORP
  350–980, pH 7.0–7.8, etc.).

🏠 Toujours envoyer l'état complet actuel (les 5 drapeaux `*Connected` lus dans
`/device/status`, VSD et durées lus dans `/device/me`) plus le champ modifié, avec les bons noms
(`cell` et non `cellOutput`, `salinity` à 0).

### 4.3 Activer ou désactiver une sonde

Chaque sonde s'active ou se désactive avec le drapeau `*Connected` correspondant :

| Action | Conséquence |
|---|---|
| `orpConnected = false` | l'appareil n'utilise plus l'ORP : la production se règle par la **consigne fixe** `cell` (%). `SPA` et `COVER` deviennent disponibles |
| `orpConnected = true` | régulation sur la consigne ORP. `cell` n'a plus d'effet |
| `phConnected = false` | plus de régulation du pH (pompe à acide) |
| `tempConnected = false` | plus de mesure de température |
| `isVsdPumpConnected` | indique qu'une pompe de filtration à vitesse variable est raccordée |

🏠 Exposer ces drapeaux comme `switch` de **configuration**, désactivés par défaut et
accompagnés d'un avertissement : couper une sonde change le mode de régulation.

### 4.4 Plannings — `POST /device/edit-schedules {serialNumber, firmwareVersion, schedules}`

- La liste **remplace** tous les créneaux. Ajouter, modifier ou supprimer revient toujours à
  envoyer la liste complète. Supprimer le dernier créneau envoie `[]`.
- Les heures sont en secondes depuis minuit, à la **minute** près.
- Contraintes : `on < off` (pas de passage à minuit), pas de chevauchement, **2 créneaux** au
  maximum.
- `days` : 7 caractères, du lundi au dimanche. Seule la valeur `1111111` (tous les jours) a été
  observée. `speed` : 10 observé.
- `firmwareVersion` est recopié depuis `/device/me`.
- La réponse est un `Device` contenant les `schedules` à jour (valeurs en chaînes).
- En Europe (`region == 1`), les plannings ne sont pas prévus pour être modifiés à distance.

🏠 En Europe, exposer les plannings en lecture seule (capteurs « début » et « fin »). Si
l'écriture est ajoutée, reproduire les contraintes ci-dessus.

### 4.5 Durées et pompe VSD

- `boostDuration` : 6, 12, 18 ou 24 h.
- `backwashDuration` : 120, 180, 240 ou 300 s (2 à 5 min).
- `vsdPumpSpeed` : 2, 4, 6, 8 ou 10, avec `isVsdPumpConnected`.

Ces trois réglages passent par `change-target`.

## 5. Alertes

1. L'appareil ou le serveur crée une `Notification` (`type`, `errorId`, `value`, `createdAt`).
2. Si un téléphone a enregistré son token (`POST /user/add-push/ {pushToken, deviceType,
   allow}`) avec `allow = true`, un push Firebase lui est envoyé. Ce push ne sert qu'à
   prévenir : le détail se lit dans `/notification/notifications`.
3. `GET /notification/notifications/{sn}` renvoie **toute** la liste, sans pagination
   (`createdAt` en UTC). Les non-lues ont `isRead == false`. Pour un affichage « récent »,
   filtrer sur les **7 derniers jours**.
4. Les textes des `errorId` s'obtiennent avec `POST /notification/errors/{lang}`, où `lang`
   vaut `en`, `de`, `es`, `fr`, `it`, `nl` ou `pt`.
5. `GET /notification/read-notification/{id}` **marque l'alerte comme lue côté serveur**, pour
   tous les utilisateurs.
6. Pour les alertes de sel (1009–1011), la quantité de sel à ajouter se calcule à partir du
   volume du bassin (formule dans
   [errors-and-thresholds.md](errors-and-thresholds.md#calcul-du-sel-à-ajouter)).

🏠 Interroger `/notification/notifications` à un rythme lent (10 à 15 min). Créer un capteur
« dernière alerte » (type, `errorId`, valeur, date) et un compteur de non-lues sur 7 jours, et
éventuellement un `event` par nouvelle alerte. **Ne jamais** appeler `read-notification` sans
action explicite de l'utilisateur.

## 6. Historique

- `GET /notification/logs/{sn}` renvoie une moyenne par jour (date à minuit UTC) pour le pH,
  l'ORP et le sel, sans température. Le serveur a renvoyé **10 jours**.
- Pour juger une valeur, se fier aux plages de
  [errors-and-thresholds.md](errors-and-thresholds.md) et, pour l'état actuel, aux champs
  `*Error` du serveur.

🏠 Home Assistant historise déjà ses capteurs. Cette route ne sert qu'à combler les périodes où
Home Assistant était arrêté.

## 7. Mise en service et partage

### Ajout d'un appareil (ADMIN)
1. Recherche Bluetooth d'un appareil dont le nom contient `dwlg-` (Davey) ou `hwqh-` (Henden).
   Le nom de l'appareil **est** son numéro de série.
2. `POST /user/add-device {serialNumber, userId}` : l'appareil est **lié au compte avant**
   d'être connecté au Wi-Fi.
3. Écritures Bluetooth : SSID (FF02), puis 1 s plus tard le mot de passe (FF04), puis 1 s plus
   tard l'octet `0x03` (FF03), qui redémarre le module Wi-Fi.
4. L'appareil passe en ligne (`isConnected`) en quelques secondes. Sans résultat au bout de
   60 s, considérer la mise en service comme échouée. La caractéristique FF05 (état Wi-Fi)
   n'est pas nécessaire.

Contraintes Wi-Fi (manuel Davey) : réseau **2,4 GHz**, SSID et mot de passe de **18 caractères**
au plus.

### Partage
- L'administrateur génère un code (`GET /user/invite-code`, avec effet de bord) et peut
  l'envoyer par email (`POST /user/email-invite {code, email, language}`).
- L'invité saisit le code (au moins 4 caractères) : `POST /user/redeem-invite {code}` le lie à
  l'appareil.
- Rôles : `ADMIN` et `SUPER` (mêmes droits) et `USER` (accès limité). Un `USER` ne pilote que
  MANUAL et SCHEDULE.
- L'administrateur peut changer un rôle (`POST /user/update-privledges`) ou retirer un
  utilisateur (`GET /user/detach-device/{userId}`, avec effet de bord).
- Supprimer son compte (`GET /user/delete/{id}`) dissocie l'utilisateur. Les réglages restent
  sur l'appareil (supposé).

## 8. Pompes à chaleur

L'association d'une pompe à chaleur est prévue hors Europe. Trois marques : **Aquark**,
**Nirvana** et **Missouri** (cette dernière utilise le protocole Nirvana).

### Nirvana / Missouri
- **Association** : l'identifiant (« MAC code ») de la carte Wi-Fi de la pompe permet de lire
  `GET https://ws.nirvanahp.com:8010/…/findAllParametersByCardId/{id}`, sans authentification.
  Si la pompe est allumée, l'associer avec `POST /device/register/heatpump {cardId: id, uid: id}`.
  Le champ `heatPump` de `/device/me` contient ensuite cet identifiant, qui compte 17
  caractères avec des `-`.
- **Lecture** : `polTemp` et `spaTemp` (consignes), `currentTemp`, `polPower` et `spaPower`
  (`"1"` = marche), `PS_Flag` (`"0"` = piscine, sinon spa), `FC_Flag` (Celsius ou Fahrenheit).
- **Consigne** : `PUT …/sendCommand/{id}` avec
  `{"cmdName": "SetHeating", "cmdValue": "mode:POL,spaTemp:<T>,polTemp:<T>"}`. Cette commande
  force le mode piscine et règle les deux consignes. En parallèle, envoyer `targets.temp` au
  serveur Davey.
- Consigne entre 15 et 39 °C.

### Aquark
- Les pompes Aquark sont des appareils **Tuya** : elles se pilotent par le cloud Tuya, pas par
  l'API Lifeguard.
- Points de données Tuya : consigne de température **106**, ou **104** sur les pompes récentes
  (quand 106 contient l'unité). Marche/arrêt et mode silencieux existent aussi, mais leurs codes
  ne sont pas documentés.

🏠 Pour une pompe à chaleur, passer par une intégration dédiée (Tuya / LocalTuya pour Aquark,
API Nirvana pour Nirvana), pas par l'API Lifeguard.

## 9. Services liés

| Service | Usage | Remarque |
|---|---|---|
| Firebase Cloud Messaging | push des alertes | enregistrement via `/user/add-push/` |
| Tuya (`m1.tuyaeu.com`) | pompes Aquark | |
| `ws.nirvanahp.com:8010` | pompes Nirvana | sans authentification |
| `uat.lg.dwprod.co` | serveur de recette | non testé |
| `hwqh.henden.co` | serveur de la marque Henden | non testé |

## 10. Récapitulatif pour l'intégration Home Assistant

| Sujet | À faire |
|---|---|
| Authentification | Retirer les espaces des identifiants. Rafraîchir au plus une fois, avec un verrou. Sauvegarder les nouveaux tokens. Masquer `token`, `certArn` et `certData` dans les logs |
| Interrogation | `/device/me` et `/device/status` toutes les 1 à 5 min. Alertes toutes les 10 à 15 min |
| Mesures | `unknown` si `-9999`, `flow == 0` ou `isConnected == false` |
| États de sonde | problème si `WARN` ou `ERROR`, OK si `NORMAL`, inconnu si `""` ou absent |
| Modes | 6 modes pilotables, liste reconstruite comme au §4.1. Exclusivité SCHEDULE / BOOST. SPA et COVER seulement sans ORP. Reprendre `response.modes` |
| Consignes | état complet plus le champ modifié, noms `cell` / `salinity: 0`, plages valides. ORP **ou** `cell` selon la sonde. Température seulement avec une pompe à chaleur |
| Sondes | interrupteurs `*Connected` en configuration, avec avertissement |
| Plannings | lecture seule en Europe. Sinon remplacement complet, 2 créneaux, `on < off`, pas de chevauchement |
| Disponibilité | entités indisponibles si `isConnected == false`. Actions réservées aux rôles `ADMIN` et `SUPER`, sauf MANUAL et SCHEDULE |
| À ne jamais appeler | les GET à effet de bord (`read-notification`, `invite-code`, `detach-device`, `delete`, `register`, `cert`, `delete-certs`) |
