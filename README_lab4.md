🔍 Rapport d'Analyse Statique — Lab 4
## APK : `UnCrackable-Level1.apk` (OWASP MSTG)

---

## A) Informations générales

| Champ | Détail |
|---|---|
| **Titre** | Analyse statique de l'application UnCrackable Level 1 |
| **Date d'analyse** | 06 mai 2026 |
| **Analyste** | haita |
| **APK analysé** | `UnCrackable-Level1.apk` — version 1.0 (versionCode : 1) |
| **Provenance** | OWASP Mobile Security Testing Guide (MSTG) |
| **SHA-256** | `1DA8BF57D266109F9A07C01BF7111A1975CE01F190B9D914BCD3AE3DBEF96F21` |
| **Outils utilisés** | JADX GUI, dex2jar v2.4, JD-GUI, Windows PowerShell |

---

## B) Résumé exécutif

Cette analyse statique a été réalisée sur l'application **UnCrackable-Level1** de l'OWASP MSTG, conçue intentionnellement pour contenir des vulnérabilités pédagogiques.

Les principales observations concernent :
1. L'**absence de protection contre l'analyse statique** (pas de ProGuard/R8 actif, code lisible directement dans JADX)
2. L'**absence de permissions dangereuses** déclarées dans le manifeste
3. Une **activité principale unique et exportée** sans restriction d'accès

Le niveau de risque global est évalué comme **Faible à Moyen**.

**Actions prioritaires recommandées :**
1. Activer l'obfuscation du code (ProGuard/R8) avant toute mise en production
2. Auditer la logique de vérification du secret dans `MainActivity`
3. Implémenter des contrôles d'intégrité de l'APK (root detection, tamper detection)

---

## C) Constats détaillés

### Constat #1 : Structure APK exposée et non obfusquée

**Sévérité :** Moyenne  
**Description :** L'APK est un fichier ZIP standard contenant les fichiers suivants, tous accessibles sans protection :
AndroidManifest.xml
META-INF/CERT.RSA
META-INF/CERT.SF
META-INF/MANIFEST.MF
classes.dex
res/layout/activity_main.xml
res/menu/menu_main.xml
res/mipmap-hdpi-v4/ic_launcher.png
res/mipmap-mdpi-v4/ic_launcher.png
res/mipmap-xhdpi-v4/ic_launcher.png
res/mipmap-xxhdpi-v4/ic_launcher.png
res/mipmap-xxxhdpi-v4/ic_launcher.png
resources.arsc

**Localisation :** Racine de l'APK (extrait via PowerShell + `ZipFile`)  
**Impact potentiel :** Un attaquant peut extraire l'intégralité du code source reconstitué sans obstacle.  
**Remédiation recommandée :** Activer R8/ProGuard pour obfusquer les noms de classes, méthodes et champs sensibles.

---

### Constat #2 : Manifeste Android révèle l'activité principale sans restriction

**Sévérité :** Faible  
**Description :** Le fichier `AndroidManifest.xml` expose le package et l'activité principale sans restriction.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    android:versionCode="1"
    android:versionName="1.0"
    package="owasp.mstg.uncrackable1">
    <uses-sdk
        android:minSdkVersion="19"
        android:targetSdkVersion="28"/>
    <application
        android:theme="@style/AppTheme"
        android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher"
        android:allowBackup="true">
        <activity
            android:label="@string/app_name"
            android:name="sg.vantagepoint.uncrackable1.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

**Localisation :** `AndroidManifest.xml` — `sg.vantagepoint.uncrackable1.MainActivity`  
**Impact potentiel :** Ciblage direct de l'activité principale pour du reverse engineering.  
**Remédiation recommandée :** Ajouter `android:exported="false"` sur les activités internes. Revoir `android:allowBackup="true"`.

---

### Constat #3 : Bytecode DEX extractible et convertible

**Sévérité :** Moyenne  
**Description :** Le fichier `classes.dex` (5 528 octets) est directement extractible. La conversion via `dex2jar` a échoué à cause d'un mauvais chemin (`NoSuchFileException`), mais le DEX reste analysable dans JADX GUI.
java.nio.file.NoSuchFileException: C:\APK-Analysis\dex_out\classes.dex

> La commande aurait dû pointer vers le dossier d'extraction de l'APK.

**Localisation :** `dex_out/classes.dex` — taille : 5 528 octets  
**Impact potentiel :** Le bytecode peut être transformé en Java lisible, exposant algorithmes et secrets.  
**Remédiation recommandée :** Utiliser l'obfuscation et ne jamais stocker de clés en dur dans le code.

---

## Comparaison JADX GUI vs JD-GUI

| Aspect | JADX GUI | JD-GUI |
|---|---|---|
| **Navigation** | Structure Android complète (Manifest, ressources, code) | Structure Java uniquement (packages, classes) |
| **Kotlin** | Meilleure gestion du code Kotlin | Syntaxe parfois illisible |
| **Obfuscation** | Tente de reconstruire les noms de variables | Conserve souvent les noms obfusqués |
| **Ressources** | Accès direct aux ressources (XML, assets) | Pas d'accès aux ressources Android |
| **Workflow** | Analyse directe depuis l'APK | Nécessite conversion DEX → JAR via dex2jar |

> **Conclusion :** JADX GUI est l'outil de référence. JD-GUI reste utile en complément pour les JARs Java standard.

---

## D) Annexes

### Permissions demandées
Aucune permission déclarée dans le manifeste.

### Composants exportés
- `sg.vantagepoint.uncrackable1.MainActivity` — Activité principale (launcher)

### Ressources identifiées (`resources.arsc`)

| Type | Nom | ID |
|---|---|---|
| `dimen` | `activity_horizontal_margin` | `0x7f010000` |
| `dimen` | `activity_vertical_margin` | `0x7f010001` |
| `id` | `action_settings` | `0x7f020000` |
| `id` | `edit_text` | `0x7f020001` |
| `layout` | `activity_main` | `0x7f030000` |
| `menu` | `menu_main` | `0x7f040000` |
| `mipmap` | `ic_launcher` | `0x7f050000` |
| `string` | `app_name` | `0x7f060001` |
| `string` | `button_verify` | `0x7f060002` |
| `string` | `thanks` | `0x7f060004` |
| `style` | `AppTheme` | `0x7f070000` |

### Hash de vérification
Algorithme : SHA-256
Hash       : 1DA8BF57D266109F9A07C01BF7111A1975CE01F190B9D914BCD3AE3DBEF96F21
Fichier    : UnCrackable-Level1.apk

---

## Prérequis pour reproduire l'analyse

- **JADX GUI** : https://github.com/skylot/jadx/releases
- **dex2jar** : https://github.com/pxb1988/dex2jar/releases
- **JD-GUI** : https://github.com/java-decompiler/jd-gui/releases
- (Optionnel) : `unzip`, `apksigner`, `keytool`
