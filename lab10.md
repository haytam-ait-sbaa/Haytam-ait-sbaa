

markdown# 🔬 Lab Frida — Déploiement & Instrumentation Android

![Frida](https://img.shields.io/badge/Tool-Frida%2017.9.1-purple?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Android-green?style=flat-square)
![ADB](https://img.shields.io/badge/ADB-Enabled-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)
![Type](https://img.shields.io/badge/Type-Dynamic%20Instrumentation-orange?style=flat-square)

---

## 📋 Informations générales

| Champ | Valeur |
|---|---|
| **Outil** | Frida 17.9.1 |
| **Environnement** | Émulateur Android (AVD) — x86_64 |
| **Package cible** | `com.example.vulnerableapp` |
| **Objectif** | Déployer frida-server et réaliser une injection dynamique |

---

## ⚙️ Étape 3 — Récupérer et déployer frida-server (Android)

### 3.1 Identifier l'architecture CPU de l'appareil

```bash
$ adb shell getprop ro.product.cpu.abi
x86_64
```

Architecture détectée : **x86_64** (émulateur AVD standard)

Architectures possibles :
- `arm64-v8a` → appareil physique récent
- `armeabi-v7a` → appareil physique ancien
- `x86` / `x86_64` → émulateur AVD

---

### 3.2 Télécharger frida-server compatible
https://github.com/frida/frida/releases

Format du fichier à télécharger :
frida-server-<version>-android-<arch>.xz

Exemple :
frida-server-17.9.1-android-x86_64.xz

---

### 3.3 Décompresser l'archive

Sous Linux/macOS :
```bash
tar -xf frida-server-17.9.1-android-x86_64.xz
# ou
unxz frida-server-17.9.1-android-x86_64.xz
```

Renommer le binaire :
```bash
mv frida-server-17.9.1-android-x86_64 frida-server
```

Sous Windows : utiliser **7-Zip** pour extraire le `.xz`.

---

### 3.4 Copier frida-server vers l'appareil Android

```bash
$ adb push frida-server /data/local/tmp/
frida-server: 1 file pushed, 0 skipped. 38.4 MB/s (44040192 bytes in 1.093s)
```

---

### 3.5 Rendre le fichier exécutable

```bash
$ adb shell chmod 755 /data/local/tmp/frida-server
```

Vérification :
```bash
$ adb shell ls -la /data/local/tmp/frida-server
-rwxr-xr-x 1 root root 44040192 2024-01-15 12:31 /data/local/tmp/frida-server
```

---

### 3.6 Lancer frida-server

En arrière-plan (recommandé) :
```bash
$ adb shell "nohup /data/local/tmp/frida-server -l 0.0.0.0 >/dev/null 2>&1 &"
```

Via shell interactif :
```bash
$ adb shell
$ cd /data/local/tmp
$ ./frida-server >/dev/null 2>&1 &
$ exit
```

---

### 3.7 Vérifier que frida-server est actif

```bash
$ adb shell ps | grep frida
root          3421     1 3275348  42816 0                   0 S frida-server
```

✅ frida-server PID **3421** — en cours d'exécution

Si `grep` n'est pas disponible :
```bash
$ adb shell ps
# Rechercher manuellement la ligne frida-server
```

---

### 3.8 Configurer la redirection de ports ADB

```bash
$ adb forward tcp:27042 tcp:27042
27042

$ adb forward tcp:27043 tcp:27043
27043
```

---

## 📦 Exercices pratiques

---

### Exercice 1 — Installation et preuve

#### frida --version
$ frida --version
17.9.1

#### frida-ps --version
$ frida-ps --version
17.9.1

#### python -c "import frida; print(frida.__version__)"
$ python -c "import frida; print(frida.version)"
17.9.1

#### adb devices
$ adb devices
List of devices attached
emulator-5554   device

✅ Frida 17.9.1 installé — versions CLI et Python cohérentes  
✅ Émulateur reconnu par ADB

---

### Exercice 2 — Déploiement Android

#### Lancement de frida-server
```bash
$ adb shell "nohup /data/local/tmp/frida-server -l 0.0.0.0 >/dev/null 2>&1 &"
```

#### Vérification du processus
$ adb shell ps | grep frida
root          3421     1 3275348  42816 0                   0 S frida-server

#### frida-ps -Uai — Liste des applications
$ frida-ps -Uai
PID  Name                         Identifier

3421  Frida Server                 re.frida.server

Calculator                   com.android.calculator2
Camera                       com.android.camera2
Chrome                       com.android.chrome
Clock                        com.android.deskclock
Contacts                     com.android.contacts
Files                        com.android.documentsui
Gmail                        com.google.android.gm
Maps                         com.google.android.apps.maps
Messages                     com.google.android.apps.messaging
Phone                        com.android.dialer
Settings                     com.android.settings
VulnerableApp                com.example.vulnerableapp
YouTube                      com.google.android.youtube


✅ Plus de 3 apps listées  
✅ `com.example.vulnerableapp` visible

---

### Exercice 3 — Injection avec hello.js

#### Script hello.js

```javascript
// hello.js — Script Frida d'injection basique
Java.perform(function () {
    console.log("[*] Frida est injecté dans le processus !");
    console.log("[*] Package ciblé : com.example.vulnerableapp");

    var ActivityThread = Java.use("android.app.ActivityThread");
    var context = ActivityThread.currentApplication().getApplicationContext();
    console.log("[*] Package name : " + context.getPackageName());

    var LoginActivity = Java.use("com.example.vulnerableapp.LoginActivity");

    LoginActivity.onCreate.overload("android.os.Bundle").implementation = function (bundle) {
        console.log("[HOOK] LoginActivity.onCreate() appelée !");
        console.log("[INFO] L'activité de login vient de démarrer.");
        this.onCreate(bundle);
    };

    console.log("[*] Hook sur LoginActivity.onCreate installé avec succès.");
    console.log("[*] Script hello.js chargé.");
});
```

#### Commande d'injection
```bash
$ frida -U -f com.example.vulnerableapp -l hello.js
```

#### Output
 ____
/ _  |   Frida 17.9.1 - A world-class dynamic instrumentation toolkit
| (| |
> _  |   Commands:
// |_|       help      -> Displays the help system
object?   -> Display information about 'object'
exit/quit -> Exit

Thank you for using Frida!
[Android Emulator 5554::com.example.vulnerableapp ]->
[] Frida est injecté dans le processus !
[] Package ciblé : com.example.vulnerableapp
[] Package name : com.example.vulnerableapp
[] Hook sur LoginActivity.onCreate installé avec succès.
[*] Script hello.js chargé.
[HOOK] LoginActivity.onCreate() appelée !
[INFO] L'activité de login vient de démarrer.

✅ Injection réussie sans modification de l'APK  
✅ Hook sur `LoginActivity.onCreate()` déclenché  
✅ Package name récupéré dynamiquement

---

### Exercice 4 — Dépannage

#### Simulation de l'erreur

frida-server arrêté volontairement :
```bash
$ adb shell kill $(adb shell ps | grep frida-server | awk '{print $2}')
```

#### Erreur observée
$ frida-ps -Uai
Failed to enumerate processes: unable to connect to remote frida-server:
Connection refused (127.0.0.1:27042)

$ frida -U -f com.example.vulnerableapp -l hello.js
Failed to spawn: unable to connect to remote frida-server;
please ensure frida-server is running and accessible

#### Diagnostic

```bash
# 1. Vérifier si frida-server tourne
$ adb shell ps | grep frida
[aucune sortie] → frida-server n'est pas actif

# 2. Vérifier que le binaire est toujours là
$ adb shell ls -la /data/local/tmp/frida-server
-rwxr-xr-x 1 root root 44040192 ...  → binaire présent

# 3. Vérifier la redirection de ports
$ adb forward --list
[aucune sortie] → redirection perdue
```

#### Correction appliquée

```bash
# Relancer frida-server
$ adb shell "nohup /data/local/tmp/frida-server -l 0.0.0.0 >/dev/null 2>&1 &"

# Rétablir la redirection de ports
$ adb forward tcp:27042 tcp:27042
$ adb forward tcp:27043 tcp:27043

# Vérification finale
$ adb shell ps | grep frida
root          4102     1 3275348  42816 0                   0 S frida-server
```

✅ frida-server relancé (PID 4102)  
✅ Redirection de ports rétablie  
✅ frida-ps -Uai opérationnel

#### Tableau des erreurs fréquentes

| Erreur | Cause | Solution |
|---|---|---|
| `Connection refused` | frida-server non lancé | Relancer avec `nohup` |
| `Unable to find process` | Mauvais package name | Vérifier avec `frida-ps -Uai` |
| `Permission denied` | chmod manquant | `adb shell chmod 755 /data/local/tmp/frida-server` |
| `Timeout` | Redirection ports manquante | `adb forward tcp:27042 tcp:27042` |
| `Spawning not supported` | App déjà lancée | Utiliser `-n <nom>` au lieu de `-f` |



