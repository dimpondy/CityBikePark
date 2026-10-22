# Garages à vélo — mise en ligne (≈ 15 min)

L'application est un seul fichier, `index.html`. Pour qu'elle soit **partagée**, **sécurisée** (comptes + modération) et **géolocalisée**, il faut :
1. un projet Firebase gratuit (base de données + comptes) ;
2. un hébergement **HTTPS** (obligatoire pour la géolocalisation sur iPhone).

## Fonctionnement

- Tout le monde peut **consulter** la carte sans compte.
- Pour **ajouter**, **confirmer** ou **signaler** un point, il faut un compte (e-mail + mot de passe).
- Un ajout d'un utilisateur ordinaire part dans une file **« en attente »** : il n'apparaît sur la carte qu'après approbation d'un modérateur.
- Les **signalements** (⚑) sont visibles uniquement des modérateurs, qui peuvent ignorer le signalement ou supprimer le point.
- Les **modérateurs** (🛡️) valident ou refusent les ajouts, suppriment des points ; leurs propres ajouts sont publiés directement.
- Une confirmation « Toujours là » compte une seule fois par utilisateur.

## 1. Créer le projet Firebase

1. https://console.firebase.google.com → **Ajouter un projet**.
2. **Build › Authentication › Commencer › Méthode de connexion** → activer **E-mail/Mot de passe**.
3. **Build › Realtime Database › Créer une base de données** (région Europe), mode **verrouillé**.
4. Onglet **Règles** → coller ceci puis **Publier** :

```json
{
  "rules": {
    ".read": false,
    ".write": false,

    "garages": {
      ".read": true,
      "$id": {
        ".write": "auth != null && root.child('admins').child(auth.uid).val() === true",
        ".validate": "newData.hasChildren(['lat','lng','type','places','created'])",
        "lat":    { ".validate": "newData.isNumber() && newData.val() >= -90 && newData.val() <= 90" },
        "lng":    { ".validate": "newData.isNumber() && newData.val() >= -180 && newData.val() <= 180" },
        "type":   { ".validate": "newData.isString() && newData.val().matches(/^(arceau|abri|securise|autre)$/)" },
        "places": { ".validate": "newData.isNumber() && newData.val() >= 1 && newData.val() <= 500" },
        "note":   { ".validate": "newData.isString() && newData.val().length <= 200" },
        "created":{ ".validate": "newData.isNumber()" },
        "uid":    { ".validate": "newData.isString()" }
      }
    },

    "pending": {
      ".read": "auth != null && root.child('admins').child(auth.uid).val() === true",
      "$id": {
        ".write": "auth != null && ((!data.exists() && newData.child('uid').val() === auth.uid) || root.child('admins').child(auth.uid).val() === true)",
        ".validate": "newData.hasChildren(['lat','lng','type','places','created','uid'])",
        "lat":    { ".validate": "newData.isNumber() && newData.val() >= -90 && newData.val() <= 90" },
        "lng":    { ".validate": "newData.isNumber() && newData.val() >= -180 && newData.val() <= 180" },
        "type":   { ".validate": "newData.isString() && newData.val().matches(/^(arceau|abri|securise|autre)$/)" },
        "places": { ".validate": "newData.isNumber() && newData.val() >= 1 && newData.val() <= 500" },
        "note":   { ".validate": "newData.isString() && newData.val().length <= 200" },
        "created":{ ".validate": "newData.isNumber()" },
        "uid":    { ".validate": "newData.isString()" }
      }
    },

    "confirmations": {
      ".read": true,
      "$gid": { "$uid": {
        ".write": "auth != null && auth.uid === $uid",
        ".validate": "newData.val() === true"
      } }
    },

    "reports": {
      ".read": "auth != null && root.child('admins').child(auth.uid).val() === true",
      "$gid": {
        ".write": "auth != null && root.child('admins').child(auth.uid).val() === true",
        "$uid": {
          ".write": "auth != null && auth.uid === $uid",
          ".validate": "newData.isNumber()"
        }
      }
    },

    "admins": {
      "$uid": { ".read": "auth != null && auth.uid === $uid" }
    }
  }
}
```

5. Copier l'URL de la base (en haut de l'onglet **Données**) dans `FIREBASE_DB_URL` dans `index.html`.
6. **Paramètres du projet (⚙︎) › Général** → **Clé API Web** → la copier dans `FIREBASE_API_KEY`.
   (Cette clé n'est pas secrète ; la sécurité repose sur les règles ci-dessus. Vous pouvez en plus la restreindre à votre domaine dans Google Cloud › Identifiants.)

## 2. Créer le premier modérateur

1. Ouvrir l'application (voir étapes 3 et 4) et **créer votre compte**.
2. Firebase › **Authentication › Utilisateurs** : copier votre **UID**.
3. **Realtime Database › Données** : ajouter manuellement le nœud `admins` → enfant `VOTRE_UID` → valeur `true`.
4. Rouvrir l'application : le bouton **🛡️ Modération** apparaît.

Pour ajouter d'autres modérateurs, répéter l'étape 3 avec leur UID.

## 3. Héberger le fichier en HTTPS

Au choix (gratuit) :
- **Netlify Drop** : https://app.netlify.com/drop → glisser le dossier contenant `index.html`.
- **GitHub Pages** : déposer `index.html` dans un dépôt, puis Settings › Pages.

Si Firebase refuse la connexion, ajouter votre domaine dans **Authentication › Paramètres › Domaines autorisés**.

## 4. Installer sur l'iPhone

Ouvrir l'adresse dans **Safari** → Partager → **Sur l'écran d'accueil**. Autoriser la localisation (sinon : Réglages › Confidentialité › Service de localisation › Safari).

## Limites à connaître

- Aucune vérification d'adresse e-mail : un utilisateur peut s'inscrire avec une adresse qu'il ne possède pas. La validation des ajouts par un modérateur limite l'impact.
- Un utilisateur peut envoyer beaucoup d'ajouts en attente : surveillez la file de modération (Firebase peut aussi limiter les inscriptions via les paramètres d'Authentication).
- Fond de carte OpenStreetMap : usage raisonnable uniquement ; pour un fort trafic, utiliser un fournisseur de tuiles dédié.
