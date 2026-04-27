#  FireStorm 

 **Techniques utilisées :** Reverse Android (Jadx) · Hooking dynamique (Frida) · Authentification Firebase

---

##  Objectif

L'application Android `FireStorm.apk` contient une méthode `Password()` qui génère un mot de passe pour se connecter à une base de données **Firebase**. Cette méthode n'est **jamais appelée** dans le flux normal de l'application.

L'objectif est de :
1. Analyser l'APK avec **Jadx** pour comprendre le code
2. Forcer l'exécution de `Password()` avec **Frida**
3. S'authentifier sur **Firebase** pour récupérer le flag

---

## 🗺️ Flow du challenge

```
FireStorm.apk
      │
      ▼
[Étape 2] Jadx → Analyse statique → Password() trouvée (jamais appelée)
      │
      ▼
[Étape 3] Frida → Hooking → instance.Password() forcée → Mot de passe obtenu
      │
      ▼
[Étape 5] Python/pyrebase → Authentification Firebase → FLAG 
```

---

## ⚙️ Étape 1 — Préparation de l'environnement

### Outils utilisés

- **ADB (Platform Tools)** : Communiquer avec l'émulateur
- **Frida** : Hooking dynamique
- **Jadx-GUI** : Décompilation de l'APK
- **Python + pyrebase** : Authentification Firebase

### Installation de l'APK

```powershell
.\adb install FireStorm.apk

```
**Résultat :**

<img width="735" height="61" alt="image" src="https://github.com/user-attachments/assets/7e07c203-b99e-4887-9237-a46c9ec62854" />



### Vérification de l'architecture

```powershell
.\adb shell getprop ro.product.cpu.abi
```
**Résultat :** 
<img width="689" height="59" alt="image" src="https://github.com/user-attachments/assets/672115bf-9381-42c4-9a45-10e688800f0c" />


---
### installation de frida 
<img width="1540" height="529" alt="image" src="https://github.com/user-attachments/assets/822aed5b-541a-4090-bf51-b9ef5bf6e539" />

### Lancement de frida-server

<img width="735" height="209" alt="image" src="https://github.com/user-attachments/assets/c3e6417c-f928-43bb-a8c4-b29118aa3b09" />

```powershell
.\adb push frida-server /data/local/tmp/
.\adb shell chmod 755 /data/local/tmp/frida-server
.\adb root
.\adb shell "/data/local/tmp/frida-server &"
```
<img width="737" height="88" alt="image" src="https://github.com/user-attachments/assets/7cdd026e-0840-4bae-9876-008ccf992352" />


---

### Vérification Frida + app installée

```powershell
frida-ps -U
.\adb shell pm list packages | findstr firestorm

```
**Résultat :**

<img width="1600" height="199" alt="image" src="https://github.com/user-attachments/assets/9668313f-b966-4a3c-a1eb-5074519c5e8e" />

<img width="901" height="670" alt="image" src="https://github.com/user-attachments/assets/e2751220-9ef3-4d2a-b322-fe19032a886e" />

### Installation de l'APK

```powershell
L'APK est ouvert dans **Jadx-GUI**. Navigation dans l'arborescence :

```
Source code
  └── com.pwnsec.firestorm
        └── MainActivity
Resources
  └── res
        └── values
              └── strings.xml


- `URL` = `https://pwnsec.xyz/flag?auth=`
- <img width="1465" height="280" alt="image" src="https://github.com/user-attachments/assets/63a360d4-cb27-4a53-9722-bf5759ee1f69" />
- `IDKMaybethepasswordpassowrd` = `v1n4of.5EY?%0z`
- `Token` = `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`
- <img width="845" height="185" alt="image" src="https://github.com/user-attachments/assets/c82ee2cc-6372-4b1a-94bf-fbb067eaa967" />

- `firebase_email` = `TK757567@pwnsec.xyz`
- <img width="1465" height="280" alt="image" src="https://github.com/user-attachments/assets/36c1ad41-f47d-46c9-ac92-1c56b0ea7d6c" />

- `firebase_database_url` = `https://firestorm-9d3db-default-rtdb.firebaseio.com`
- <img width="1555" height="607" alt="image" src="https://github.com/user-attachments/assets/f410c3a7-38df-46c6-8153-7ef41ffd81c9" />

- `google_api_key` = `AIzaSyAXsK0q********************HuJY`
- <img width="1432" height="246" alt="image" src="https://github.com/user-attachments/assets/de5d7344-2aee-415a-9316-5da6633fe247" />

>  Les credentials Firebase sont stockés en clair dans l'APK. N'importe qui peut les extraire avec Jadx.

### 📄 MainActivity.java — Analyse

#### 1. Chargement de la librairie native

<img width="566" height="160" alt="image" src="https://github.com/user-attachments/assets/0b9071ac-5f38-4176-93d8-4d19cfbee4b0" />

>  L'app charge `libfirestorm.so` au démarrage. C'est là que se trouve `generateRandomString()`.

---

#### 2. La méthode Password() — jamais appelée 

<img width="430" height="286" alt="image" src="https://github.com/user-attachments/assets/274266f6-f087-419f-a372-17d74c7deb43" />


>  Cette méthode **n'est appelée nulle part** dans `onCreate()`. C'est le **cœur du challenge**.

---

#### 3. La fonction native

```java
public native String generateRandomString(String str);
```
> Déclarée en Java, **implémentée en C/C++** dans `libfirestorm.so`. Jadx ne peut pas la décompiler. Elle ajoute une partie **aléatoire** au mot de passe.

---

#### 4. onCreate() — le flux normal


<img width="468" height="253" alt="image" src="https://github.com/user-attachments/assets/528d8d3a-4d5d-41b1-b8fb-25557e0aa90b" />

---

###  Conclusion de l'analyse statique

> L'APK contient une méthode `Password()` dans `MainActivity` qui construit un mot de passe en combinant des fragments de `strings.xml`, puis appelle `generateRandomString()` dans `libfirestorm.so` pour y ajouter une partie dynamique. Cette méthode n'est **jamais invoquée** dans le flux normal. L'objectif est d'utiliser **Frida** pour forcer son exécution.

---

##  Étape 3 — Hooking dynamique avec Frida

### Script — `frida_firestorm.js`

```javascript
Java.perform(function() {

    function getPassword() {
        console.log("[*] Recherche de l'instance MainActivity...");

        Java.choose('com.pwnsec.firestorm.MainActivity', {

            onMatch: function(instance) {
                console.log("[+] Instance trouvée ! Extraction du secret...");
                try {
                    var pass = instance.Password();
                    console.log("[!] MOT DE PASSE OBTENU : " + pass);
                } catch (e) {
                    console.log("[-] Erreur : " + e);
                }
            },

            onComplete: function() {
                console.log("[*] Scan terminé.");
            }
        });
    }

    console.log("[*] Script chargé. Attente de 3 secondes...");
    setTimeout(getPassword, 3000);
});
```

| Élément | Rôle |
|---------|------|
| `Java.perform()` | Initialise l'environnement Java de Frida |
| `Java.choose()` | Cherche les instances vivantes de `MainActivity` en mémoire |
| `onMatch(instance)` | Appelé pour chaque instance trouvée |
| `instance.Password()` | Appel **forcé** de la méthode cachée |
| `setTimeout(..., 3000)` | Attend 3s que l'app et `libfirestorm.so` soient chargés |

---

### Lancement

```powershell
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js --no-pause
```

| Option | Signification |
|--------|---------------|
| `-U` | Cible l'émulateur connecté en USB/ADB |
| `-f` | Force le lancement de l'application |
| `-l` | Charge le script JavaScript |
| `--no-pause` | Ne met pas l'app en pause au démarrage |

---

### Résultat

```
Spawned 'com.pwnsec.firestorm'. Resuming main thread!
[*] Recherche de l'instance MainActivity...
[+] Instance trouvée ! Extraction du secret...
[!] MOT DE PASSE OBTENU : C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC
[*] Scan terminé.
```

<img width="799" height="238" alt="image" src="https://github.com/user-attachments/assets/c52589c3-0ae3-49ce-9d06-e6accbcfac7a" />


> **Mot de passe obtenu :** `C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC`

> Ce mot de passe est **dynamique** : il change à chaque lancement à cause de `generateRandomString()`.

---

##  Étape 4 — Configuration Firebase (strings.xml)

```xml
<string name="firebase_api_key">AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY</string>
<string name="firebase_email">TK757567@pwnsec.xyz</string>
<string name="firebase_database_url">https://firestorm-9d3db-default-rtdb.firebaseio.com</string>
```

---

##  Étape 5 — Script Python et récupération du flag

### Script — `get_flag.py`

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

email    = "TK757567@pwnsec.xyz"
password = "C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC"  # obtenu via Frida

user = auth.sign_in_with_email_and_password(email, password)
print("[+] Connexion réussie !")

db = firebase.database()
flag_data = db.get(user['idToken'])
print("[!] CONTENU DE LA BASE :")
print(flag_data.val())
```

### Exécution

```powershell
python get_flag.py
```

### Résultat

```
[+] Connexion réussie !
----------------------------------------
[!] CONTENU DE LA BASE :
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
----------------------------------------
```

<img width="743" height="113" alt="image" src="https://github.com/user-attachments/assets/d6bb39b9-0ddf-4829-8095-d8d46997898c" />


##  FLAG

```
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

##  Récapitulatif

| Étape | Outil | Action | Résultat |
|-------|-------|--------|----------|
| 1 | ADB + Frida | Installation APK + frida-server | Environnement prêt  |
| 2 | Jadx-GUI | Analyse statique APK | `Password()` + credentials Firebase trouvés  |
| 3 | Frida | Hooking → appel forcé `Password()` | Mot de passe obtenu  |
| 4 | strings.xml | Extraction config Firebase | Email, API key, database URL  |
| 5 | Python/pyrebase | Auth Firebase + lecture DB | **FLAG obtenu**  |

---

##  Vulnérabilités identifiées

| # | Vulnérabilité | Impact |
|---|---------------|--------|
| 1 | **Dead code dangereux** — `Password()` existe mais n'est jamais appelée | Exposé via hooking Frida |
| 2 | **Credentials en clair dans strings.xml** — Email, API key, URL Firebase lisibles après décompilation | Accès complet à la DB |
| 3 | **Librairie native insuffisante** — `libfirestorm.so` ajoute de l'aléatoire mais le contexte reste accessible via Frida | Contournement total |

---


```
---

