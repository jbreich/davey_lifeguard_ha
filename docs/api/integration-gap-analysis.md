# Écarts entre l'API et l'intégration Home Assistant

État de l'intégration : version `1.4.1` (commit `9b36435`).

## Couverture des endpoints

| Endpoint | Utilisé | Remarque |
|---|---|---|
| `POST /user/signin` | ✅ | |
| `POST /user/auth` | ✅ | Les nouveaux tokens **ne sont pas sauvegardés** dans la config entry |
| `GET /device/me` | ✅ | |
| `GET /device/status/{sn}` | ✅ | |
| `POST /device/change-target` | ✅ | Corps non conforme (voir plus bas) |
| `POST /device/v2/change-mode` | ✅ | Seulement MANUAL et BOOST |
| `POST /device/edit-schedules` | ❌ | |
| `GET /notification/notifications/{sn}` | ❌ | Alertes : la donnée la plus utile qui manque |
| `GET /notification/logs/{sn}` | ❌ | Intérêt limité (Home Assistant a son propre historique) |
| `POST /notification/errors/{lang}` | ❌ | Textes d'aide, en option |
| `GET /device/users/{sn}` | ❌ | Inutile pour Home Assistant |
| Routes de compte / partage / enregistrement | ❌ | Hors périmètre |

## Données lues mais non exposées

| Donnée | Source | Proposition Home Assistant |
|---|---|---|
| `modes` hors MANUAL/BOOST : `SPA`, `COVER`, `WINTER`, `SCHEDULE` (**pas** `PUMP` ni `HEAT`, sans fonctionnement connu) | `/device/me` | Un `switch` par mode, avec les conditions de [business-rules.md](business-rules.md). L'appareil testé est en `WINTER`, que l'intégration n'affiche pas |
| État `BACKWASH` (dans `modes`). `FILTERING` = `status.flow == 1`, `CHLORINATING` = `status.cellOutput > 0` (calculés) | `/device/me`, `/device/status` | `binary_sensor` |
| `tempTarget` | `/device/me` | `number` 15–39 °C (écriture : `targets.temp`) |
| `cellOutput` (consigne sans sonde ORP) | `/device/me` | `number` 0–100 %, par pas de 5 (écriture : `targets.cell`). Aujourd'hui, la consigne est **écrasée** par la mesure lors de la fusion `me \| status` |
| `boostDuration` | `/device/me` | `select` 6/12/18/24 h |
| `backwashDuration` | `/device/me` | `select` 2/3/4/5 min (envoyé ×60, en secondes) |
| `schedules` | `/device/me` | Capteurs « prochain démarrage / arrêt », ou entités `time` + `select` pour éditer via `edit-schedules` |
| `saltStatus`, `poolSize`, `nipperCell`, `region` | `/device/me` | Attributs ou capteurs de diagnostic |
| Quantité de sel à ajouter | calculée (`poolSize`, `salinity`, `saltStatus`) | `sensor` en kg (formule dans [errors-and-thresholds.md](errors-and-thresholds.md#calcul-du-sel-à-ajouter)) |
| Activation des sondes | `*Connected` dans `change-target` | `switch` de configuration par sonde |
| Dernière alerte / alertes non lues | `/notification/notifications` | `sensor` (type + valeur + date) et compteur. Éventuellement des `event` |

## Écarts avec le comportement attendu de l'API

| # | Sujet | Intégration | Attendu | Gravité |
|---|---|---|---|---|
| 1 | Consigne de production de la cellule dans `change-target` | envoie `cellOutput` | `cell` | **Haute** : la clé n'est probablement pas reconnue par le serveur |
| 2 | `salinity` dans `change-target` | `1250` en dur | `0` | Moyenne : l'API n'a pas de vraie consigne de sel |
| 3 | Plage pH du `select` | 7.0–7.9 | 7.0–7.8 | Basse |
| 4 | Plage ORP du `select` | 600–950, par pas de 50 | 350–980, par pas de 10 | **Moyenne** : une consigne réglée ailleurs (ex. 650 → OK, 720 → « inconnu ») n'est pas affichée. Passer en `number` |
| 5 | Exclusivité SCHEDULE / BOOST | non gérée | à refuser côté client | À reproduire si SCHEDULE est ajouté |
| 6 | Ordre des modes | ajout en fin de liste | ajout en tête | Probablement sans effet, mais facile à aligner |
| 7 | Fusion `me \| status` | `cellOutput` de status écrase celui de me | objets distincts | Moyenne (voir tableau précédent) |
| 8 | Refresh token | nouveau token gardé en mémoire seulement | sauvegardé | ✅ corrigé (branche `fix/security-hardening`) |
| 9 | 401 après rafraîchissement | récursion sans limite | une seule nouvelle tentative | ✅ corrigé (branche `fix/security-hardening`) |
| 10 | Log debug `Device data` | contient `token` et `certArn` de l'appareil | secrets masqués | ✅ corrigé (branche `fix/security-hardening`) |
| 11 | `DaveyErrorBinarySensor` | toute valeur ≠ `NORMAL` (y compris `""` ou clé absente) → « problème » | `WARN` / `ERROR` = problème, `""` = pas de sonde | Basse |
| 12 | `vsdPumpSpeed` | `select` 2/4/6/8/10 | 2/4/6/8/10 | ✅ conforme |

## Règles d'usage de l'API non respectées par l'intégration

Détail dans [business-rules.md](business-rules.md) et [behavior.md](behavior.md).

| # | Règle | Intégration actuelle | Risque |
|---|---|---|---|
| R1 | `SCHEDULE` et `BOOST` s'excluent | `BOOST` activable sans regarder `SCHEDULE` (mode non exposé, mais il peut être actif sur l'appareil) | Combinaison au comportement inconnu |
| R2 | `SPA` / `COVER` seulement sans sonde ORP | non exposés | À respecter si on les ajoute |
| R3 | Consigne ORP **ou** consigne cellule selon la présence de la sonde ORP | `select` ORP toujours disponible, consigne cellule envoyée sous le mauvais nom (`cellOutput`) | Consigne ORP sans effet sans sonde |
| R4 | Contrôles désactivés si l'appareil est hors ligne (`isConnected == false`) | toujours actifs | Commandes perdues ou en attente |
| R5 | Commandes autres que `MANUAL` / `SCHEDULE` réservées aux rôles `ADMIN` et `SUPER` | aucun contrôle de rôle | Erreurs API (probablement 401/403) pour un compte `USER` |
| R6 | `*Connected` = activation des sondes, toujours renvoyés | renvoyés depuis `status` ✅ | — (à préserver absolument) |
| R7 | Réponse de `change-mode` = vérité | ignorée, rafraîchissement complet | Faible (une requête de plus) |
| R8 | Mesures invalides si `flow == 0`, `-9999` = pas de sonde | garde la dernière valeur si `flow == 0` ✅, `-9999` affiché tel quel | Valeurs aberrantes (`-9999` ppm) dans l'historique si une sonde est retirée |
| R9 | `MANUAL` = marche / arrêt du système (« System On/Off ») | libellé « Manual » | Confusion : couper MANUAL revient probablement à **arrêter le système** |
| R10 | `change-mode` n'envoie que les 6 modes pilotables (sans `BACKWASH` ni autres états) | renvoie `data["modes"]` tel quel, plus ou moins le mode | `BACKWASH` ou un état futur réinjecté dans la liste : comportement inconnu |
| R11 | État des sondes selon les `*Error` du serveur. Mesure invalide si `-9999`, `flow == 0` ou `isConnected == false` | garde l'ancienne valeur si `flow == 0`, ignore `isConnected` | Valeurs figées présentées comme actuelles |
| R12 | Espaces retirés autour de l'email et du mot de passe | pas de nettoyage | Échec de connexion si un espace est copié-collé |
| R13 | Après une écriture, l'objet `Device` renvoyé fait foi | rafraîchissement complet | Faible |

## Priorités proposées

1. **Corriger** les points 1, 2, 8, 9 et 10.
2. **Ajouter les modes** SPA, COVER, WINTER, SCHEDULE (avec l'exclusivité BOOST) et des capteurs d'état pour BACKWASH / FILTERING / CHLORINATING.
3. **Ajouter** les entités `number` / `select` : `tempTarget`, `cell`, `boostDuration`, `backwashDuration`. Passer pH et ORP en `number` avec les plages valides.
4. **Alertes** : interroger `/notification/notifications` (par exemple toutes les 15 min) pour un capteur de dernière alerte et un compteur de non-lues.
5. **Plannings** : lecture d'abord (capteurs), puis écriture via `edit-schedules`.
