# Authentification

## Principe

- Connexion par **email + mot de passe**. L'API renvoie un `token` (courte durée) et un `refreshToken`.
- Le `token` va **tel quel** dans l'en-tête `Authorization` : pas de `Bearer `, pas de SigV4.
- Quand une requête renvoie `401`, il faut appeler `/user/auth` avec `userId` + `refreshToken`,
  récupérer le **nouveau couple** token / refreshToken, puis rejouer la requête **une seule fois**.
- `/user/auth` renvoie un **nouveau** refresh token. L'ancien semble rester utilisable après
  un rafraîchissement (supposé, non vérifié). Par prudence, il faut toujours sauvegarder le nouveau.
- Un client peut aussi rafraîchir au démarrage avec le refresh token stocké, sans redemander le
  mot de passe. Le déroulé complet de la session est dans [behavior.md](behavior.md#2-session).

## `POST /user/signin` ✅

Requête :

```json
{ "email": "user@example.com", "password": "********" }
```

Réponse `200` (anonymisée) :

```json
{
  "token": "<token>",
  "refreshToken": "<refresh-token>",
  "user": {
    "id": 12345,
    "firstname": "Jane",
    "lastname": "Doe",
    "email": "user@example.com",
    "allowsnotifications": true,
    "isadmin": null,
    "privledgelevel": "ADMIN",
    "password": "<hash — renvoyé par le serveur>",
    "isdisabled": false,
    "isInvited": null,
    "marketingmaterialok": null,
    "updatesok": null,
    "city": null,
    "country": null,
    "language": null,
    "organisationid": null,
    "pushToken": "<FCM token>",
    "isVerified": true,
    "verifyCode": "<code à 4 chiffres>",
    "createdAt": "2026-05-29T14:48:39.594Z",
    "updatedAt": "2026-06-03T20:02:24.086Z",
    "DeviceId": 10000,
    "deviceId": 10000
  }
}
```

À retenir :
- `user.id` est un **entier** : c'est le `userId` à renvoyer à `/user/auth`.
- `user.deviceId` / `user.DeviceId` est l'identifiant interne de l'appareil, pas son numéro de série.
- `user.deviceId == 0` signifie que le compte n'a pas encore d'appareil.
- En cas d'échec, la réponse est une erreur HTTP (4xx). Le corps de l'erreur n'est pas
  documenté : traiter toutes les causes comme « identifiants invalides ». L'intégration utilise
  `raise_for_status()`.
- Retirer les espaces autour de l'email et du mot de passe avant l'envoi : un espace
  copié-collé suffit à faire échouer la connexion.

### Constat de sécurité

La réponse de connexion, ainsi que `GET /device/users/{sn}`, renvoie des champs sensibles
qu'un client n'a pas à recevoir :
- `password` (vraisemblablement un hash) ;
- `verifyCode` (code de vérification email / PIN de réinitialisation) ;
- `pushToken` (token FCM) ;
- dans `/device/me` : `token` de l'appareil et `certArn` (avec l'identifiant du compte AWS).

**Un client ne doit jamais écrire ces champs dans ses logs.** Sur `master`, l'intégration écrit
`Device data: …` en debug avec le contenu complet de `/device/me` : le `token` de l'appareil
finit donc dans les logs Home Assistant (corrigé sur la branche `fix/security-hardening`).

## `POST /user/auth` — rafraîchissement ✅

Requête :

```json
{ "userId": 12345, "refreshToken": "<refresh-token>" }
```

Réponse :

```json
{ "token": "<nouveau token>", "refreshToken": "<nouveau refresh token>", "user": { … }, "statusCode": 200 }
```

> `userId` est un **entier**. L'intégration envoie la valeur stockée dans la config entry, qui
> vient de `data["user"]["id"]`, donc aussi un entier : c'est cohérent.

## En-têtes

| En-tête | Valeur |
|---|---|
| `Authorization` | `<token>` (brut) |
| `Content-Type` | `application/json` |

- `/user/auth` et `/user/register` n'exigent pas d'en-tête `Authorization`.
- `/user/signin` ignore un en-tête `Authorization` vide ou périmé.
- Après un rafraîchissement, **remplacer** l'en-tête `Authorization`, ne pas en ajouter un second.
- Aucune clé d'API supplémentaire n'est nécessaire.

## Erreurs

| Statut | Corps | Signification |
|---|---|---|
| `401` | — | Token expiré : rafraîchir puis rejouer la requête |
| `403` | `{"message":"Forbidden"}` | Chemin à la racine ou ressource inconnue sans authentification |
| `403` | `{"message":"Authorization header requires 'Credential' parameter. …"}` | **Route inexistante** (comportement par défaut d'AWS API Gateway) |
| `400` | variable | Corps invalide (vu sur `/user/redeem-invite` dans une capture réseau publique) |

Le message SigV4 sur une route inconnue ne veut **pas** dire que l'authentification est fausse.
C'est simplement la réponse d'API Gateway quand aucune route ne correspond.
