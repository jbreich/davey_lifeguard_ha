# Erreurs, alertes et seuils

## Codes `errorId`

Codes connus, recoupés avec les notifications réelles.

| errorId | `type` | Sévérité | Sens | Sujet |
|---|---|---|---|---|
| 1001 | `ORP_ERROR` (supposé) | alarme | haut | ORP trop élevé (trop de chlore) |
| 1002 | `ORP_ERROR` (supposé) | alarme | bas | ORP trop bas (pas assez de chlore) |
| 1003 | `PH_ERROR` ✅ | alarme | bas | pH trop bas (observé : 5.5, 5.8) |
| 1004 | `PH_ERROR` ✅ | alarme | haut | pH trop haut (observé : 8.2) : bidon d'acide vide ou sonde HS |
| 1005 | `PH_WARN` ✅ | avertissement | haut | pH au-dessus de la consigne (observé : 7.8) |
| 1006 | `ORP_WARN` ✅ | avertissement | haut | ORP plus de 15 mV au-dessus de la consigne |
| 1007 | `ORP_WARN` (supposé) | avertissement | bas | ORP plus de 15 mV en dessous de la consigne |
| 1008 | `PH_WARN` ✅ | avertissement | bas | pH sous la consigne |
| 1009 | `SALT_WARN` (supposé) | avertissement | bas | Sel bas : « Add {XX} kgs of salt » |
| 1010 | `SALT_ERROR` ✅ | alarme | bas | **Low Salt Cut-Out** : arrêt de l'électrolyse |
| 1011 | `SALT_WARN` (supposé) | avertissement | bas | Sel bas (variante) |
| 1012 | `FLOW_ERROR` ✅ | alarme | bas | Débit < 60 L/min pendant 3 min : pompe de filtration coupée |
| 1013–1017 | `ERROR_CALIBRATE` ✅ (1015, 1016) | entretien | — | Nettoyage / calibration de la cellule ou des sondes |
| 1018–1019 | `WIFI_ERROR` (supposé) | connexion | — | Appareil déconnecté du Wi-Fi |

✅ = `type` observé dans les données réelles.

Familles de codes :
- `1001` à `1012` : mesures (pH, ORP, sel) et débit ;
- `1013` et `1014` : nettoyage de la cellule ;
- `1015` et `1017` : nettoyage / calibration des sondes ;
- `1016` : entretien, le plus fréquent sur l'appareil testé ;
- `1018` et `1019` : perte du Wi-Fi.

### Valeurs de `type`

| `type` | Famille |
|---|---|
| `PH_ERROR`, `PH_WARN` | alarme / avertissement pH |
| `ORP_ERROR`, `ORP_WARN` | alarme / avertissement ORP |
| `SALT_ERROR`, `SALT_WARN` | alarme / avertissement sel |
| `FLOW_ERROR` | débit faible |
| `WIFI_ERROR` | Wi-Fi |
| `ERROR_CALIBRATE`, `ERROR_CLEAN`, `CLEAN_ERROR` | entretien (calibration / nettoyage) |

Exemples de textes d'aide (en anglais) :
- **1012 / Low Flow** : « Your Lifeguard has registered a flow rate below 60L/min for 3 minutes and has switched off your filtration pump ».
- **1010 / Low Salt Cut-Out** : « The salt level in your pool has fallen below the required level. Your Lifeguard is no longer producing chlorine ».
- **pH haut** : « No reduction in pH after 1 hour acid pump operation ».
- **ERROR_CALIBRATE** : « It is time to clean and calibrate your pH probe / ORP probe / Salt/Temp probe », « It is time to clean your salt cell ».

Les textes localisés complets s'obtiennent avec `POST /notification/errors/{lang}`.

Le manuel Davey précise que l'alarme **Low Salt Cut-Out** doit être acquittée sur l'appareil
(bouton SYSTEM ON/OFF) une fois le sel revenu dans la plage : l'électrolyse ne reprend pas
seule.

## Champs `*Error` de `/device/status`

`phError`, `orpError`, `saltError` et `flowError` valent `"NORMAL"` en fonctionnement normal.
Les valeurs possibles sont `NORMAL`, `WARN`, `ERROR` et `""` (sans objet). Seul `NORMAL` a été
observé. Il faut traiter `WARN` et `ERROR` comme un problème, mais `""` ou **une clé absente**
comme une valeur inconnue, et non comme une erreur.

## Seuils de référence

Ces seuils servent à juger une mesure. Les notifications, elles, sont produites par l'appareil
ou le serveur : pour l'état officiel d'une sonde, se fier aux champs `*Error`.

### Écart à la consigne

| Grandeur | Avertissement si l'écart dépasse | Erreur si l'écart dépasse |
|---|---|---|
| pH | 0.2 | 0.3 |
| ORP | 15 mV | 300 mV |
| Température | 6 °C | 15 °C |

### Plages de référence

| Grandeur | Plage normale | Idéal / défaut |
|---|---|---|
| pH | 6.8 – 8.0 | 7.4 |
| ORP | 535 – 800 mV | 650 mV |
| Sel | 3000 – 6000 ppm (normal), 1500 – 6000 ppm (*low salt*), d'après la fiche technique Davey | **4500** (NORMAL) / **2250** (LOW_SALT) |
| Température | 10 – 40 °C | 25 °C |
| Débit | au moins 60 L/min (alerte 1012 en dessous pendant 3 min) | — |

Échelles d'affichage conseillées pour des graphiques : pH 6.5–8.2 · ORP 350–980 mV ·
sel 1000–8000 ppm · température 10–40 °C.

## Plages de consignes valides

| Consigne | Valeurs |
|---|---|
| pH (`ph`) | 7.0 → 7.8, par pas de 0.1 |
| ORP (`orp`) | 350 → 980 mV, par pas de 10 |
| Production cellule (`cell`) | 0 → 100 %, par pas de 5 |
| Température (`temp`) | 15 → 39 °C, par pas de 1 |
| Vitesse VSD (`vsdPumpSpeed`, `schedules[].speed`) | 2, 4, 6, 8, 10 |
| Durée boost (`boostDuration`) | 6, 12, 18, 24 h |
| Durée backwash (`backwashDuration`) | 2, 3, 4, 5 min, envoyées en **secondes** (120–300) |

## Calcul du sel à ajouter

```
kg = ceil( (idéal(saltStatus) - salinité_actuelle) * poolSize / 1 000 000 )
idéal = 4500 ppm si saltStatus == 0 (NORMAL), 2250 ppm si saltStatus == 1 (LOW_SALT)
```

Exemple : bassin de 23 000 L à 4300 ppm → `ceil(200 × 23000 / 1e6)` = **5 kg**.
