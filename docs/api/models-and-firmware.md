# Modèles, firmwares et conséquences sur les fonctions

## En bref

- L'API ne renvoie **aucun nom de modèle**. Le modèle se déduit de plusieurs champs :
  préfixe du numéro de série, `region`, `nipperCell`, `saltStatus` et `io`.
- Aucune fonction connue ne dépend de la version du **firmware** ni de la **taille de cellule**.
- Ce qui change réellement les fonctions disponibles : la **région**, les **sondes présentes**,
  la **pompe à chaleur associée**, la **pompe VSD**, le **rôle** de l'utilisateur et la
  **marque** (Davey ou Henden). Voir [business-rules.md](business-rules.md).

## 1. Gammes de produits

### Davey ProMatic Lifeguard — Europe (`region = 1`)

Source : fiche technique Davey `DWP1837/SUK0524` (2024) et page produit Davey EU.

| Modèle | Chlore (g/h) | Sel (ppm) | Bassin max. (m³) climat frais < 25 °C | tempéré 25–30 °C | chaud > 30 °C |
|---|---|---|---|---|---|
| DPLG-15E | 15 | 3000–6000 | 75 | 58 | 46 |
| DPLG-25E | 25 | 3000–6000 | 125 | 96 | 80 |
| DPLG-35E | 35 | 3000–6000 | 175 | 134 | 112 |
| DPLG-15EL | 15 | 1500–6000 (*low salt*) | 75 | 58 | 46 |
| DPLG-25EL | 25 | 1500–6000 (*low salt*) | 125 | 96 | 80 |

Kit fourni avec tous les modèles :
- un boîtier d'alimentation de la cellule (« EcoSalt2 ») et un boîtier de contrôle « Lifeguard » ;
- une cellule à inversion de polarité avec détecteur de débit intégré, fonctionnant jusqu'à 3,6 m³/h pour les pompes à vitesse variable ;
- les sondes pH, ORP, sel et température, sur un porte-sondes en ligne ;
- une pompe péristaltique d'acide ;
- le Wi-Fi (le pilotage à distance peut être désactivé), le Bluetooth et un écran LCD multilingue.

### Davey ChloroMatic Lifeguard — Australie / Nouvelle-Zélande (`region = 0`)

Sources : catalogue Davey et fiches revendeurs.

| Modèle | Chlore (g/h) | Sel (ppm) |
|---|---|---|
| DCLG-15C | 15 | 3000–6000 |
| DCLG-25C | 25 | 3000–6000 |
| DCLG-35C | 35 | 3000–6000 |
| DCLG-15CLS | 15 | 1500–6000 (*low salt*) |
| DCLG-25CLS | 25 | 1500–6000 (*low salt*) |

Mêmes fonctions de chimie (pH avec dosage d'acide, ORP, sel, température). Le système est aussi
vendu en **module** pour équiper un électrolyseur Davey **Nipper** existant. C'est sans doute
pourquoi l'API parle de `nipperVersion` et de `nipperCell` pour la partie électrolyseur.

### Henden Smart Chlorinator — Reece (Australie)

C'est une **marque blanche** du même système :

| | Davey | Henden |
|---|---|---|
| Préfixe du numéro de série / nom Bluetooth | `dwlg-` | `hwqh-` |
| Serveur API | `https://lg.dwprod.co` | `https://hwqh.henden.co` |

Une intégration qui vise aussi Henden devrait rendre l'URL de base configurable. Le serveur
Henden n'a pas été testé.

### Région USA (`region = 2`)

La valeur existe, mais aucun produit Lifeguard américain n'a été trouvé. Elle se comporte
probablement comme l'Australie : plannings et pompe à chaleur disponibles (supposé).

## 2. Champs de `/device/me` qui décrivent le matériel

| Champ | Sens | Effet sur les fonctions |
|---|---|---|
| `serialNumber` | préfixe `dwlg-` (Davey) ou `hwqh-` (Henden) | identifie aussi l'appareil en Bluetooth |
| `region` | 0 Australie, 1 Europe, 2 USA | **oui** : en Europe, plannings en lecture seule et pas de pompe à chaleur |
| `nipperCell` | production nominale de la cellule en g/h (15 / 25 / 35) | aucun connu (n'intervient pas dans le calcul du sel à ajouter) |
| `saltStatus` | 0 = normal (idéal 4500 ppm), 1 = *low salt* (idéal 2250 ppm). Correspond probablement aux variantes `EL` / `CLS` | **oui** : idéal de sel et quantité à ajouter |
| `io` | 8 bits. Bits 4 à 7 = sondes pH, ORP, sel, température. Bits 0 à 3 : sens inconnu | préférer `status.*Connected` pour l'état des sondes |
| `firmwareVersion` | firmware du **contrôleur Lifeguard**, au format `AA.BB.CC` | renvoyé dans `edit-schedules` |
| `nipperVersion` | firmware de l'**électrolyseur** (EcoSalt2 / Nipper) | aucun connu |
| `brand` | 0 observé sur un appareil Davey. Valeurs Henden inconnues | aucun connu |
| `isVsdPumpConnected`, `vsdPumpSpeed` | pompe de filtration à vitesse variable déclarée | **oui** : réglage VSD |
| `heatPump` | identifiant de pompe à chaleur. Nirvana = 17 caractères avec `-`, Aquark = plus de 5 caractères | **oui** : consigne de température |
| `acidPump`, `filtrationPump` | déclarés via `POST /device/update` (non testé) | aucun connu |
| `inUse` | sens inconnu | aucun connu |

## 3. Firmwares

### Repères de fonctions

Aucune fonction de l'API ne se conditionne au firmware. Les repères suivants circulent, sans
avoir été vérifiés :
- **1.0 à 1.2** : ni plannings, ni pompe VSD ;
- **1.3 et plus** : plannings et pompe VSD ;
- **2.x et 3.x** : pas de changement connu ;
- **4.x** : choix des jours de la semaine dans les plannings. Seule la valeur `days = "1111111"`
  a pourtant été observée.

### Mise à jour du firmware

Selon le manuel ProMatic Lifeguard (`DWP1738-4revA`, 2022, §7.1) :
- Davey pousse les mises à jour **critiques** en OTA, à condition que l'appareil soit connecté à Internet.
- Davey recommande de **forcer** une mise à jour à l'installation, puis de temps en temps, en commençant par le contrôleur :
  1. contrôleur Lifeguard : maintenir le bouton **bas** environ 5 s. L'écran s'éteint, puis affiche « software update in progress » ;
  2. électrolyseur (EcoSalt2 ou Nipper) : maintenir ensuite le bouton **retour** du contrôleur environ 5 s. L'écran s'éteint pendant la mise à jour.
- Lire les versions : menu `settings` → `about` du contrôleur. L'électrolyseur affiche sa version au démarrage (l'éteindre puis le rallumer).

Côté API :
- Aucune route ne permet de lancer une mise à jour, ni de savoir si une version plus récente existe.
- Le canal OTA passe probablement par AWS IoT (l'appareil a son propre certificat) : il est hors de portée de l'API utilisateur.

### Liste des versions

Davey ne publie **ni liste de versions, ni notes de version**. Les seuls repères publics :

| Source | Composant | Version | Ce qu'on en tire |
|---|---|---|---|
| Manuel ChloroMatic Nipper (2020), écran de démarrage | Nipper `AU 15` | `V1.3.2` (le texte parle de `v1.2.1`) | ancien format `Vx.y.z` |
| Même manuel, figure 11.1 | électrolyseur `EU 15L` | `V1.2.1` | idem |
| Manuel ProMatic Lifeguard (2022), §4.6.3 et §9.1 | contrôleur | — | liaison avec les pompes SP400BT / PM400BT et consigne VSD « disponibles dans une future révision du firmware » |
| Appareil de test (API) | contrôleur / électrolyseur | `04.01.04` / `05.01.02` | VSD déclarée : la fonction annoncée en 2022 a donc bien été livrée |

L'écran de démarrage de l'électrolyseur suit le format `DAVEY V<version> <région> <taille>[L]` :
`AU` ou `EU`, puis 15, 25 ou 35, et `L` pour les modèles *low salt*. Il permet de confirmer le
modèle sans ouvrir le boîtier. Pour l'appareil de test, on s'attend à `DAVEY V5.1.2 EU 25`
(hypothèse : `05.01.02` côté API correspond à `V5.1.2` à l'écran).

On ne peut donc pas savoir si une version est la dernière. Le plus simple : forcer une mise à
jour (voir plus haut), puis relire `firmwareVersion` et `nipperVersion`. Si rien ne change,
l'appareil est à jour.

### Versions observées

| Composant | Version |
|---|---|
| Contrôleur Lifeguard (`firmwareVersion`, et `version` dans `/device/status`) | `04.01.04` |
| Électrolyseur (`nipperVersion`) | `05.01.02` |

**Recommandation pour une intégration** : ne rien conditionner au firmware. Afficher
`firmwareVersion` et `nipperVersion` dans les informations de l'appareil
(`sw_version` / `hw_version`), ce que fait déjà l'intégration.

## 4. Matrice : condition → conséquence

| Condition | Conséquence |
|---|---|
| `region == 1` (Europe) | plannings et mode `SCHEDULE` en lecture seule. Pas d'association de pompe à chaleur |
| Sonde ORP présente (`status.orpConnected`, ou `orp != -9999`) | consigne **ORP** éditable, consigne **cellule %** sans effet. Modes `SPA` et `COVER` indisponibles |
| Pas de sonde ORP | consigne **cellule %** (`targets.cell`) éditable. `SPA` et `COVER` disponibles (−80 %) |
| Sonde pH / sel / température absente (`-9999`) | pas de mesure. Consigne pH sans objet sans sonde pH |
| `heatPump` vide | consigne de température sans objet |
| Pompe à chaleur Nirvana | pilotage via l'API Nirvana (`ws.nirvanahp.com:8010`) en plus de `targets.temp` |
| Pompe à chaleur Aquark | pilotage via le cloud Tuya |
| `isVsdPumpConnected` | vitesse VSD réglable (2–10) |
| `saltStatus == 1` | idéal de sel 2250 ppm au lieu de 4500 |
| `flow == 0` | toutes les mesures invalides |
| `isConnected == false` | aucune commande possible, mesures invalides |
| rôle `USER` | seulement `MANUAL` et `SCHEDULE`. Aucune consigne ni réglage |
| Marque Henden | autre serveur, autre préfixe de série |
| `firmwareVersion`, `nipperCell`, `brand` | **aucune** connue |

## 5. Appareil de test

| Indice | Valeur | Conclusion |
|---|---|---|
| Préfixe du numéro de série | `dwlg-` | marque Davey |
| `region` | 1 | Europe, donc gamme **ProMatic Lifeguard** |
| `nipperCell` | 25 | 25 g/h |
| `saltStatus` | 0 | sel normal (3000–6000 ppm). Mesure de 4462 ppm, cohérente |
| `io` | `00001111` | 4 sondes : pH, ORP, sel, température |
| `isVsdPumpConnected` | `true` (vitesse 8) | pompe VSD déclarée |
| `heatPump` | `null` | pas de pompe à chaleur |
| Firmwares | `04.01.04` / `05.01.02` | — |
| Rôle du compte | `ADMIN` | tous les réglages autorisés |

**Modèle : Davey ProMatic Lifeguard DPLG-25E**. La gamme et la puissance sont certaines. La
lettre `E` (et non `EL`) découle de `saltStatus = 0`. Pour lever le doute, vérifier l'étiquette
du boîtier ou l'écran de démarrage de l'électrolyseur.

Fonctions effectivement pilotables pour ce profil :

| Fonction | Disponible ? |
|---|---|
| `MANUAL` (« System On/Off »), `BOOST`, `WINTER` | ✅ |
| `SCHEDULE` et plannings | ❌ lecture seule (Europe) |
| `SPA`, `COVER` | ❌ sonde ORP présente |
| Consigne pH, consigne ORP | ✅ |
| Consigne cellule % | ❌ sonde ORP présente |
| Consigne de température | ❌ pas de pompe à chaleur |
| Vitesse VSD, durées boost / backwash | ✅ |
