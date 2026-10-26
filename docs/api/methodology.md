# Méthode et conventions

## Sources

1. **Appels réels en lecture seule** sur un compte propriétaire (rôle ADMIN), avec un appareil
   en firmware `04.01.04`, électrolyseur `05.01.02`, région Europe et cellule 25 g/h. Le drapeau
   de pompe VSD est activé, mais la pompe de filtration est d'une autre marque et n'est pas
   pilotée par le contrôleur.
2. Une capture réseau publique de 2023 (dépôt GitHub `ggear/asystem`), limitée à `/user/auth`
   et `/user/redeem-invite`.
3. La documentation commerciale Davey (fiche technique ProMatic Lifeguard `DWP1837/SUK0524`,
   catalogue ChloroMatic Lifeguard, fiche technique de la pompe ProMaster PM400BT) et les
   manuels d'installation (ProMatic Lifeguard `DWP1738-4revA` de 2022, ChloroMatic Nipper de
   2020).
4. Le code de cette intégration Home Assistant.

Les routes d'écriture et les routes à effet de bord sont décrites avec leur format attendu,
mais leur comportement côté serveur n'a pas été observé : elles sont marquées **non testé**.

## Règles suivies pendant l'exploration

- **Aucune écriture** : seul `POST /user/signin` a été envoyé, toutes les autres requêtes
  étaient des `GET`.
- Liste noire des **GET à effet de bord**, jamais appelés : `/user/delete/`,
  `/user/detach-device/`, `/user/invite-code`, `/device/register/`, `/device/cert/`,
  `/device/delete-certs`, `/notification/read-notification/`.
- Les secrets (mot de passe, tokens, `pushToken`, `password`, `verifyCode`, `certArn`) sont
  masqués à l'écriture des réponses. Le mot de passe était lu depuis un fichier local et n'a
  jamais été affiché.
- Toutes les valeurs personnelles des exemples (numéro de série, ID, email, nom, ARN, codes)
  sont **remplacées** dans cette documentation.

## Requêtes effectuées

| Requête | Statut |
|---|---|
| `POST /user/signin` | 200 |
| `GET /device/me` | 200 |
| `GET /device/status/{sn}` | 200 |
| `GET /device/users/{sn}` | 200 |
| `GET /notification/notifications/{sn}` | 200 |
| `GET /notification/logs/{sn}` | 200 |
| Sondage de ~25 chemins devinés (`/user/me`, `/device/history/…`, `/swagger.json`…) | 403 : routes inexistantes |

## Limites / à confirmer

- Comportement réel des routes d'écriture : corps partiels acceptés, validation des plages,
  arbitrage entre modes cumulés (`WINTER` + `COVER`, etc.).
- Valeurs de `phError` / `orpError` / `saltError` / `flowError` autres que `NORMAL`.
- Sens des bits 0 à 3 de `io`, de `brand` et de `filtrationPump`.
- Effet réel de `vsdPumpSpeed` et des `speed` de plannings sur une pompe Davey compatible :
  l'appareil de test n'en a pas.
- Durée de validité du token, et validité de l'ancien refresh token après un rafraîchissement.
- Application des droits côté serveur pour un compte `USER`.
- Codes Tuya de marche/arrêt et du mode silencieux des pompes Aquark.
- Comportement du serveur de recette et du serveur Henden.

Pour lever ces doutes, il faut tester les écritures sur un appareil de test, en observant les
réponses (par exemple derrière un proxy HTTPS).
