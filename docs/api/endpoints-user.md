# Endpoints `/user/*`

Hors connexion et rafraîchissement ([authentication.md](authentication.md)), aucune de ces
routes n'a été appelée. Elles servent à gérer le compte et le partage. Une intégration
domotique n'en a pas besoin, mais elles sont documentées pour être complet.

| Méthode | Route | Corps | Réponse | Rôle |
|---|---|---|---|---|
| POST | `/user/signin` | `{email, password}` | `{token, refreshToken, user}` | Connexion |
| POST | `/user/auth` | `{userId:int, refreshToken}` | `{token, refreshToken, user, statusCode}` | Rafraîchissement du token |
| POST | `/user/register` | `{email, password, firstname, lastname, ismarketingok:bool}` | `{token, refreshToken, user}` | Création de compte |
| POST | `/user/add-device` | `{serialNumber, userId:string}` | `Device` | Lier un appareil au compte |
| POST | `/user/invite` | `{deviceId:int, inviteEmail, userEmail}` | `{sent:bool}` | Inviter un utilisateur par email (ancienne méthode) |
| GET ⚠️ | `/user/invite-code` | — | `{code:int}` | **Génère** un code de partage |
| POST | `/user/email-invite` | `{code, email, language}` | `{success:bool}` | Envoyer le code par email |
| POST | `/user/redeem-invite` | `{code}` | `Device` | Rejoindre un appareil avec un code |
| POST | `/user/update/invite` | `{accepted:bool, userId:string}` | `User` | Accepter / refuser une invitation (ancienne méthode) |
| POST | `/user/update-privledges` | `{userId:string, privledgelevel:"ADMIN"\|"USER"}` | `User` | Changer le rôle d'un utilisateur |
| POST | `/user/add-push/` | `{pushToken, deviceType, allow:bool}` | `User` | Enregistrer le token FCM / APNs et le choix de notifications (`allow = false` coupe le push). `deviceType` : type de téléphone, par exemple `"android"` |
| POST | `/user/v2/reset` | `{email}` | `{success:bool}` | Demander une réinitialisation (code PIN par email) |
| POST | `/user/v2/update/password` | `{email, ofl, password}` | `{message}` | Nouveau mot de passe (`ofl` = code reçu) |
| GET ⚠️ | `/user/detach-device/{userId}` | — | `{removed:bool}` | **Retire** un utilisateur de l'appareil |
| GET ⚠️ | `/user/delete/{userId}` | — | `{deleted:bool}` | **Supprime** le compte et ses données |

À retenir :
- Les fautes d'orthographe (`privledges`, `privledgelevel`) font partie de l'API : il faut les
  garder telles quelles.
- Rôles : `ADMIN` (peut tout modifier), `SUPER` (mêmes droits qu'`ADMIN`, mais ne s'attribue
  pas via `update-privledges`) et `USER` (limité à MANUAL et SCHEDULE).
- Partage : l'administrateur génère un code (`invite-code`) et peut l'envoyer par email
  (`email-invite`). L'invité saisit le code (au moins 4 caractères) dans `redeem-invite`.
  `/user/invite` et `/user/update/invite` sont d'anciennes routes, remplacées par le partage par
  code (supposé).
- `/user/add-device` se fait pendant la mise en service Bluetooth, **avant** l'envoi du Wi-Fi à
  l'appareil. Le `serialNumber` est le nom Bluetooth de l'appareil.
- Réinitialisation du mot de passe : `v2/reset` envoie un code temporaire par email, à renvoyer
  avec le nouveau mot de passe dans `v2/update/password` (`ofl` = code).
- `detach-device` et `delete` sont des **GET destructifs** : ne jamais les appeler pour tester.
