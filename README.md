# LAB18-SECURITE
# 🔥 FireStorm — Writeup

> **Plateforme :** pwnsec.xyz  
> **Niveau :** Medium  
> **Catégorie :** Mobile Security / Android Reverse Engineering  
> **Flag :** `PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}`
>
> **Réalisé par :** `El Ouarzazi Aya`

---

## 📋 Objectif

L'application Android contient une méthode `Password()` qui génère dynamiquement un mot de passe Firebase. Cette méthode n'est **jamais appelée** dans le flux normal de l'application. L'objectif est de :

1. Identifier cette méthode via analyse statique
2. Forcer son exécution avec **Frida**
3. Utiliser le mot de passe obtenu pour s'authentifier sur **Firebase**
4. Récupérer le flag depuis la **Realtime Database**

---

## 🛠️ Outils utilisés

| Outil | Version | Usage |
|-------|---------|-------|
| JADX-GUI | Latest | Décompilation de l'APK |
| Frida | 17.9.1 | Hooking dynamique Java |
| frida-tools | 17.9.1 | CLI Frida |
| Python | 3.10 | Script d'authentification Firebase |
| pyrebase4 | Latest | Bibliothèque Firebase Python |
| ADB | Latest | Communication avec l'émulateur |
| Android Studio | Latest | Émulateur Pixel 4a (API 29) |

---

## 📱 Étape 1 — Installation de l'environnement

### Démarrage de l'émulateur
Lancer un émulateur Android via Android Studio (Pixel 4a, API 29).

### Installation de l'APK
```bash
adb install FireStorm.apk
```

### Vérification de Frida
```bash
frida --version
# 17.9.1

frida-ps -U
# Liste des processus Android visibles → confirme la connexion
```

### Démarrage de frida-server sur l'émulateur
```bash
# Push du binaire frida-server (même version que le client)
adb push frida-server-17.9.1-android-x86_64 /data/local/tmp/frida-server
adb shell chmod +x /data/local/tmp/frida-server

# Lancement (dans un terminal dédié)
adb shell /data/local/tmp/frida-server
```

---

## 🔍 Étape 2 — Analyse statique avec JADX

Ouverture de l'APK dans JADX-GUI :
```bash
jadx-gui FireStorm.apk
```

### Package principal
```
com.pwnsec.firestorm
```

### Découverte de la méthode Password()

Dans `MainActivity`, on identifie la méthode clé :

```java
public String Password() {
    StringBuilder sb = new StringBuilder();
    String string  = getString(R.string.Friday_Night);
    String string2 = getString(R.string.Author);
    String string3 = getString(R.string.JustRandomString);
    String string4 = getString(R.string.URL);
    String string5 = getString(R.string.IDKMaybethepasswordpassowrd);
    String string6 = getString(R.string.Token);

    sb.append(string.substring(5, 9));
    sb.append(string4.substring(1, 6));
    sb.append(string2.substring(2, 6));
    sb.append(string5.substring(5, 8));
    sb.append(string3);
    sb.append(string6.substring(18, 26));

    return generateRandomString(String.valueOf(sb));
}

public native String generateRandomString(String str);
```

### Observations clés

- La méthode **Password()** existe mais **n'est jamais appelée** dans `onCreate()` ni dans aucun listener
- Elle combine des **strings statiques** extraites de `res/values/strings.xml`
- Elle appelle une **fonction native** `generateRandomString()` depuis `libfirestorm.so`
- Le mot de passe est donc **dynamique** — il change à chaque exécution

### Informations Firebase dans strings.xml

```xml
<string name="firebase_api_key">AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY</string>
<string name="firebase_email">TK757567@pwnsec.xyz</string>
<string name="firebase_database_url">https://firestorm-9d3db-default-rtdb.firebaseio.com</string>
```

---

## 🪝 Étape 3 — Hooking avec Frida

### Script frida_firestorm.js

```javascript
Java.perform(function() {

    function getPassword() {
        console.log("[*] Recherche d'instances de MainActivity...");

        Java.choose('com.pwnsec.firestorm.MainActivity', {

            onMatch: function(instance) {
                console.log("[+] Instance trouvée : " + instance);
                try {
                    var pass = instance.Password();
                    console.log("[+] Mot de passe Firebase : " + pass);
                } catch (e) {
                    console.log("[-] Erreur : " + e);
                }
            },

            onComplete: function() {
                console.log("[*] Recherche terminée.");
            }
        });
    }

    // Délai pour laisser charger l'app et libfirestorm.so
    setTimeout(getPassword, 4000);
});
```

### Explication du script

| Élément | Rôle |
|---------|------|
| `Java.perform()` | Initialise le runtime Java de Frida |
| `Java.choose()` | Parcourt la heap pour trouver les instances vivantes de MainActivity |
| `onMatch(instance)` | Appelé pour chaque instance trouvée — `instance` = `this` Java |
| `instance.Password()` | Appel direct de la méthode cachée, déclenche aussi la fonction native |
| `setTimeout(..., 4000)` | Attend 4s pour que l'app et la lib native soient chargées |

### Lancement

```bash
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js
```

### Résultat obtenu

```
[*] Recherche d'instances de MainActivity...
[+] Instance trouvée : com.pwnsec.firestorm.MainActivity@2965bef
[+] Mot de passe Firebase : C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC
[*] Recherche terminée.
```

> ⚠️ **Note importante :** Le mot de passe est différent à chaque lancement car `generateRandomString()` est non-déterministe. Il faut utiliser le mot de passe immédiatement après l'avoir obtenu.

---

## 🔥 Étape 4 — Authentification Firebase et récupération du flag

### Installation de pyrebase4

```bash
pip install pyrebase4
```

### Script get_flag.py

```python
import pyrebase

config = {
    "apiKey": "AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY",
    "authDomain": "firestorm-9d3db.firebaseapp.com",
    "databaseURL": "https://firestorm-9d3db-default-rtdb.firebaseio.com",
    "storageBucket": "firestorm-9d3db.appspot.com",
    "projectId": "firestorm-9d3db"
}

firebase = pyrebase.initialize_app(config)
auth = firebase.auth()

email = "TK757567@pwnsec.xyz"
password = "C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC"  # Obtenu via Frida

try:
    user = auth.sign_in_with_email_and_password(email, password)
    print("[+] Authentification réussie !")

    db = firebase.database()
    flag_data = db.get(user['idToken'])
    print("[+] FLAG :")
    print(flag_data.val())

except Exception as e:
    print(f"[-] Erreur : {e}")
```

### Exécution

```bash
python get_flag.py
```

### Résultat

```
[+] Authentification réussie !
[+] FLAG :
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

## 🏁 Flag

```
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

## 📚 Résumé des techniques

```
APK
 └─► JADX (analyse statique)
      └─► Découverte de Password() — méthode cachée jamais appelée
           └─► Frida (hooking dynamique)
                └─► Appel forcé de Password() + generateRandomString() (natif)
                     └─► Mot de passe dynamique obtenu
                          └─► Firebase Auth (pyrebase4)
                               └─► Realtime Database
                                    └─► FLAG 🏆
```

