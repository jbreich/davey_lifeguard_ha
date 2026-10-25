# API cloud Davey Lifeguard — documentation non officielle

Documentation de l'API REST cloud qui pilote les électrolyseurs / contrôleurs de piscine Davey
*ChloroMatic Lifeguard* et *ProMatic Lifeguard*, ainsi que leur marque blanche Henden.

Davey ne publie aucune documentation de cette API. Légende utilisée dans tous les fichiers :

- ✅ **testé** : vérifié par des appels réels en lecture seule, sur un compte et un appareil
  réels (firmware `04.01.04`, électrolyseur `05.01.02`, région Europe) ;
- **non testé** : route d'écriture ou à effet de bord, jamais appelée. Le format est connu, mais
  la réaction du serveur n'a pas été observée ;
- **supposé** : déduction à confirmer.

Les exemples JSON sont anonymisés. Voir [methodology.md](methodology.md) pour les règles suivies.

## Sommaire

| Fichier | Contenu |
|---|---|
| [behavior.md](behavior.md) | **À lire en premier** : fonctionnement de bout en bout (session, interrogation, commandes, alertes, mise en service, partage, pompes à chaleur) et récapitulatif pour Home Assistant |
| [authentication.md](authentication.md) | Connexion, tokens, rafraîchissement, en-têtes, erreurs |
| [endpoints-device.md](endpoints-device.md) | `/device/*` : état, consignes, modes, plannings, enregistrement |
| [endpoints-notification.md](endpoints-notification.md) | `/notification/*` : alertes, historique |
| [endpoints-user.md](endpoints-user.md) | `/user/*` : compte, invitations, droits, push |
| [data-models.md](data-models.md) | Schéma de tous les objets JSON |
| [modes-and-schedules.md](modes-and-schedules.md) | Modes de fonctionnement et plannings de filtration |
| [models-and-firmware.md](models-and-firmware.md) | **Modèles** (ProMatic / ChloroMatic / Henden), champs matériels, firmwares, matrice condition → fonction |
| [business-rules.md](business-rules.md) | **Combinaisons valides** : modes incompatibles, consignes selon les sondes, droits, région, pièges de `change-target` |
| [errors-and-thresholds.md](errors-and-thresholds.md) | Codes `errorId`, seuils, plages de consignes |
| [heat-pumps-and-ble.md](heat-pumps-and-ble.md) | Pompes à chaleur (Nirvana, Aquark), Bluetooth, AWS IoT |
| [integration-gap-analysis.md](integration-gap-analysis.md) | Écarts entre l'API et l'intégration Home Assistant actuelle |
| [methodology.md](methodology.md) | Sources, règles de sécurité suivies, limites |

## Vue d'ensemble

| Élément | Valeur |
|---|---|
| URL de base (prod) | `https://lg.dwprod.co/` |
| URL de base (recette) | `https://uat.lg.dwprod.co` (non testée) |
| Marque blanche Henden | `https://hwqh.henden.co` (non testée) |
| Hébergement | AWS API Gateway, région `ap-southeast-2` (Sydney) |
| Format | JSON (`Content-Type: application/json`) |
| Authentification | Token opaque dans l'en-tête `Authorization` (**sans** préfixe `Bearer`) |
| Délai conseillé | 30 s |
| Temps réel | Aucun push pour les données : il faut interroger l'API. Les alertes peuvent être poussées par Firebase Cloud Messaging (voir `/user/add-push/`) |
| Services liés | API Nirvana et cloud Tuya, pour les pompes à chaleur |

### Carte des endpoints

| Méthode | Chemin | Rôle | Testé |
|---|---|---|---|
| POST | `/user/signin` | Connexion email / mot de passe | ✅ |
| POST | `/user/auth` | Rafraîchissement du token | ✅ (via l'intégration) |
| GET | `/device/me` | Configuration complète de l'appareil lié au compte | ✅ |
| GET | `/device/status/{serialNumber}` | Mesures instantanées + état des sondes | ✅ |
| GET | `/device/users/{serialNumber}` | Utilisateurs ayant accès à l'appareil | ✅ |
| POST | `/device/change-target` | Modifier les consignes (pH, ORP, temp., cellule, VSD, durées) | non testé (écriture) |
| POST | `/device/v2/change-mode` | Définir la liste des modes actifs | non testé (écriture) |
| POST | `/device/edit-schedules` | Remplacer les plannings de filtration | non testé (écriture) |
| POST | `/device/update` | Déclarer pompe à acide / pompe à chaleur / firmware | non testé (écriture) |
| GET ⚠️ | `/device/register/?serialNumber=` | Enregistrer un appareil (**effet de bord**) | interdit |
| GET ⚠️ | `/device/cert/?serialNumber=` | Obtenir le certificat AWS IoT (**effet de bord probable**) | interdit |
| GET ⚠️ | `/device/delete-certs?serialNumber=` | Supprimer les certificats (**destructif**) | interdit |
| POST | `/device/register/heatpump` | Associer une pompe à chaleur | non testé (écriture) |
| GET | `/notification/notifications/{serialNumber}` | Historique des alertes | ✅ |
| GET | `/notification/logs/{serialNumber}` | Moyennes journalières pH / ORP / sel (10 jours) | ✅ |
| POST | `/notification/errors/{language}` | Textes d'aide pour des `errorId` | non testé (POST) |
| GET ⚠️ | `/notification/read-notification/{id}` | Marquer une alerte comme lue (**effet de bord**) | interdit |
| POST | `/user/register` | Création de compte | non testé |
| POST | `/user/add-device` | Lier un appareil à un utilisateur | non testé |
| POST | `/user/invite`, `/user/email-invite` | Inviter un utilisateur | non testé |
| GET ⚠️ | `/user/invite-code` | Générer un code d'invitation (**effet de bord**) | interdit |
| POST | `/user/redeem-invite` | Utiliser un code d'invitation | non testé |
| POST | `/user/update/invite` | Accepter / refuser une invitation | non testé |
| POST | `/user/update-privledges` | Changer le rôle ADMIN / USER | non testé |
| POST | `/user/add-push/` | Enregistrer le token de notification push | non testé |
| POST | `/user/v2/reset`, `/user/v2/update/password` | Réinitialisation du mot de passe | non testé |
| GET ⚠️ | `/user/detach-device/{userId}` | Retirer un utilisateur de l'appareil (**destructif**) | interdit |
| GET ⚠️ | `/user/delete/{userId}` | Supprimer le compte et ses données (**destructif**) | interdit |

> ⚠️ **Plusieurs GET ont des effets de bord**, dont certains destructifs. Un client ne doit
> jamais les appeler « pour voir ».

Une route inconnue renvoie `403` avec un message AWS SigV4
(`Authorization header requires 'Credential' parameter…`), et non `404`.
Voir [authentication.md](authentication.md#erreurs).
