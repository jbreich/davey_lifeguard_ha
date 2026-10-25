# Règles métier, exclusivités et conditions d'activation

Ces règles décrivent les combinaisons valides. On ne sait pas si le serveur ou l'appareil les
impose : aucune écriture n'a été testée. Un client doit donc les appliquer lui-même, sinon il
risque de produire des combinaisons au comportement inconnu.

Légende :
- 🔒 **Règle** : action à refuser côté client.
- 👁 **Disponibilité** : fonction sans objet ou indisponible dans ce cas.

## 1. Modes

### 1.1 Modes pilotables

Huit valeurs de mode existent, mais `PUMP` et `HEAT` n'ont aucun fonctionnement connu.
**Seuls 6 modes sont pilotables** :

| Mode | Pilotable par | Notes |
|---|---|---|
| `MANUAL` | tous les utilisateurs | Marche / arrêt du système (« System On/Off ») |
| `SCHEDULE` | tous les utilisateurs | Non modifiable à distance en Europe (voir 1.4) |
| `BOOST` | administrateurs | Électrolyse à 100 % pendant `boostDuration` heures |
| `SPA` | administrateurs | Production −80 %, seulement **sans sonde ORP** (voir 1.3) |
| `COVER` | administrateurs | Production −80 %, seulement **sans sonde ORP** (voir 1.3) |
| `WINTER` | administrateurs | Production ramenée à 85 % |
| `PUMP`, `HEAT` | personne | Aucun format ni effet connu : **à ne pas exposer** |

Administrateurs = rôles `ADMIN` et `SUPER` (voir 2.3).

### 1.2 🔒 `SCHEDULE` et `BOOST` s'excluent

Une activation n'est acceptable que si :

```
(mode == SCHEDULE et BOOST inactif) ou (mode == BOOST et SCHEDULE inactif)
ou (mode n'est ni SCHEDULE ni BOOST)
```

La désactivation est toujours permise.

### 1.3 🔒 `SPA` et `COVER` exigent l'absence de sonde ORP

```
activable = !status.orpConnected && device.isConnected && utilisateur administrateur
```

Avec une sonde ORP, l'appareil régule le chlore sur la mesure ORP, et la réduction de 80 % de
SPA / COVER n'a pas de sens : ces modes ne réduisent la production que lorsqu'aucune sonde ORP
n'est connectée.

### 1.4 👁 Région Europe (`region == 1`)

| Élément | En Europe |
|---|---|
| Plannings et mode `SCHEDULE` | non prévus pour être modifiés à distance |
| Association d'une pompe à chaleur | non prévue |

L'appareil européen testé renvoie pourtant 2 créneaux dans `schedules` (06:00–09:00 et
17:00–22:00), sans doute réglés sur l'écran de l'appareil. Recommandation : **lecture seule** en
Europe.

### 1.5 👁 Appareil hors ligne

Si `device.isConnected == false`, aucune commande de mode ne peut atteindre l'appareil :
désactiver tous les interrupteurs de mode.

### 1.6 Construction de la liste et réponse

- La liste envoyée ne contient **que les 6 modes pilotables** actifs (`MANUAL`, `WINTER`,
  `SPA`, `BOOST`, `COVER`, `SCHEDULE`). `BACKWASH` et les autres états renvoyés par l'appareil
  sont **écartés** avant l'envoi.
- Activer un mode = l'**ajouter en tête**. Désactiver = le retirer. La liste complète est
  envoyée à `POST /device/v2/change-mode`.
- ⚠️ Repartir d'une liste **vide** (envoyer `["MANUAL"]` ou `[]`) efface les autres modes
  actifs.
- La réponse est un objet `Device` complet. **`response.modes` fait foi**, pas la liste
  envoyée. Si le serveur ou l'appareil arbitre, c'est là que ça se voit.
- Les changements passent par AWS IoT et peuvent mettre quelques secondes à arriver : relire
  l'état environ 20 s plus tard.

### 1.7 États qui ne sont pas des modes

| État | Source réelle |
|---|---|
| `FILTERING` | **calculé** : `status.flow == 1` |
| `CHLORINATING` | **calculé** : `status.cellOutput > 0` |
| `BACKWASH` | présent dans `modes`, renvoyé par l'appareil |
| `MAINTENANCE`, `SERVICE` | valeurs connues, jamais observées |

`FILTERING` et `CHLORINATING` ne doivent **jamais** être envoyés dans `change-mode`.

### 1.8 Pompe et chauffage

Il n'existe aucun format connu pour une programmation de vitesse de pompe heure par heure, ni
pour un mode chauffage (`PUMP`, `HEAT`).

## 2. Consignes (`POST /device/change-target`)

### 2.1 🔒 Consigne ORP et consigne de cellule s'excluent

| Situation | Consigne ORP (`orp`) | Consigne cellule (`cell`, %) |
|---|---|---|
| Sonde ORP présente (`orp != -9999` ou `orpConnected`) | **éditable** | sans effet : la production est régulée automatiquement |
| Pas de sonde ORP | sans objet | **éditable**, valeur actuelle dans `device.cellOutput` |

Avec une sonde ORP, la production est régulée automatiquement sur la consigne ORP. Sans sonde,
on fixe directement un pourcentage de production.

### 2.2 👁 Consigne de température

Elle n'a de sens que si une **pompe à chaleur** est associée (`heatPump` de type Nirvana ou
Aquark). En Europe, l'association d'une pompe à chaleur n'est pas prévue (voir 1.4).

### 2.3 🔒 Droits

Un utilisateur est administrateur si `privledgelevel` vaut **`ADMIN` ou `SUPER`**. Seul un
administrateur peut modifier pH, ORP, température, cellule, VSD, durée de boost ou de backwash,
la pompe à chaleur, et mettre un appareil en service. Un `USER` ne pilote que `MANUAL` et
`SCHEDULE`. L'application de ces droits par le serveur n'a pas été testée.

### 2.4 Validation des valeurs

| Consigne | Valeurs valides |
|---|---|
| ORP | 350 à 980 mV, par pas de 10 |
| pH | 7.0 à 7.8, par pas de 0.1 |
| Cellule | 0 à 100 %, par pas de 5 |
| Durée boost | 6 / 12 / 18 / 24 h |
| Durée backwash | 120 / 180 / 240 / 300 s |
| VSD | 2 / 4 / 6 / 8 / 10 |
| Envoi | inutile si la valeur et l'état de la sonde n'ont pas changé |

La validation côté serveur n'a pas été testée.

### 2.5 Mise à jour partielle et pièges

- Les champs de consigne (`ph`, `orp`, `salinity`, `temp`, `cell`) peuvent être omis : pour ne
  changer que la température, il suffit d'envoyer `temp` avec les drapeaux et réglages
  (non testé).
- ⚠️ Les drapeaux `phConnected`, `orpConnected`, `salinityConnected`, `tempConnected` et
  `isVsdPumpConnected` doivent **toujours être envoyés**. Ce sont eux qui **activent ou
  désactivent les sondes**. Un client qui les oublie ou les met à `false` **désactive les
  sondes**. Il faut toujours renvoyer l'état actuel, lu dans `/device/status`.
- ⚠️ `vsdPumpSpeed`, `boostDuration` et `backwashDuration` doivent aussi toujours être envoyés.
  Des valeurs par défaut (`10`, `24`, `120`) à la place des valeurs réelles risquent de
  réinitialiser ces réglages. **Toujours renvoyer les valeurs actuelles** de `/device/me`.
- Pour ne changer que l'état de la sonde de température, renvoyer la **consigne** `tempTarget`
  dans `temp`, jamais la mesure (qui peut valoir `-9999`).
- `salinity` vaut toujours `0` ou est absent : il n'existe pas de consigne de sel.

## 3. Mesures (`GET /device/status`)

| Règle | Détail |
|---|---|
| Sonde absente | valeur sentinelle **`-9999`** (pH : `-9999.0`) |
| Mesure valable | seulement si la valeur est différente de `-9999` **et** `flow == 1` **et** l'appareil est connecté |
| Pas de débit (`flow == 0`) | **toutes les mesures sont invalides** : ORP, pH, sel, température et production |
| `status.cellOutput` | production **réelle** de la cellule (%). À ne pas confondre avec `device.cellOutput`, la **consigne** |
| `phError`, `orpError`, `saltError` | **`NORMAL`**, **`WARN`**, **`ERROR`**, **`""`** (vide : sonde absente ou état inconnu). `flowError` prend probablement les mêmes valeurs (supposé) |
| Idéal de sel | 4500 ppm si `saltStatus == 0`, 2250 ppm si `saltStatus == 1` (LOW_SALT) |

### Champ `io` de `/device/me`

Chaîne de 8 caractères `0`/`1`. Les bits 4 à 7 indiquent les sondes :

| Index | 0–3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|
| Sens | inconnu (sorties ?) | sonde pH | sonde ORP | sonde sel | sonde température |

Exemple observé : `00001111`, soit les 4 sondes présentes. Pour l'état des sondes, préférer les
champs `*Connected` de `/device/status`.

## 4. Plannings (`POST /device/edit-schedules`)

| Règle | Détail |
|---|---|
| 🔒 Début avant fin | `on < off`. **Pas de créneau qui passe minuit** |
| 🔒 Pas de chevauchement | deux créneaux ne doivent pas se recouvrir un même jour |
| 🔒 Nombre maximal | **2** créneaux |
| `days` | seule la valeur `1111111` a été observée. Le choix des jours n'est pas confirmé (peut-être à partir du firmware 4.x) |
| `speed` | `10` observé |
| Région | lecture seule en Europe (voir 1.4) |

## 5. Synthèse pour un client tiers (Home Assistant)

À appliquer dans le code :

1. Ne pas exposer `PUMP` ni `HEAT`.
2. Refuser `SCHEDULE` si `BOOST` est actif, et inversement.
3. Rendre `SPA` et `COVER` indisponibles quand `status.orpConnected` est vrai.
4. Exposer **soit** la consigne ORP (sonde ORP présente), **soit** la consigne cellule `cell`
   (pas de sonde ORP), jamais les deux en écriture.
5. Consigne de température seulement si `heatPump` est renseigné.
6. Rendre tous les contrôles indisponibles quand `isConnected` est faux.
7. Réserver aux administrateurs (`ADMIN` ou `SUPER`) tout ce qui n'est ni `MANUAL` ni
   `SCHEDULE`. Le rôle est `user.privledgelevel` dans la réponse de `/user/signin`.
8. En Europe, plannings en **lecture seule**.
9. Dans `change-target`, toujours renvoyer **toutes** les valeurs actuelles (`*Connected`, VSD,
   durées), jamais de valeur par défaut.
10. Après `change-mode`, reprendre `response.modes`.
11. Mesures avec `flow == 0` ou valeur `-9999` : état « inconnu », pas une vraie valeur.
12. Erreurs de sonde : « problème » si `WARN` ou `ERROR`, OK si `NORMAL`, « inconnu » si `""`
    ou absent.
13. Plannings : `on < off`, aucun chevauchement, 2 au maximum.
