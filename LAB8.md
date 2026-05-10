# 📱 Rapport d'Analyse de Sécurité Mobile — FakeBank Pro v3.1.0



---

## A. Informations générales

| Champ | Valeur |
|-------|--------|
| **Date d'analyse** | 2026-05-10 |
| **Analyste** | Haytam Ait Sbaa |
| **Cible** | FakeBank Pro v3.1.0 |
| **Package** | `com.fakebank.pro` |
| **APK analysé** | `fakebank_pro_3.1.0.apk` (12.4 MB) |
| **SHA-256** | `a3f9c2e1b4d7a8f2c9e3b1d4a7f2c8e1b3d6a9f1c4e7b2d5a8f3c6e9b1d4a7f2` |
| **Outils utilisés** | BeVigil v2.1.0, Yaazhini v1.3.2 |
| **Périmètre** | Analyse statique — APK simulé à but pédagogique |

---

## B. Résumé exécutif

L'analyse de **FakeBank Pro v3.1.0** a révélé **7 vulnérabilités** :

| Sévérité | Nombre |
|----------|--------|
| 🔴 Critical | 1 |
| 🟠 High | 2 |
| 🟡 Medium | 2 |
| 🟢 Low | 2 |

Le problème le plus grave est un **token JWT Admin hardcodé** dans le code source donnant un accès administrateur permanent à l'API backend. Combiné à des communications partiellement non chiffrées et à un stockage de mots de passe en clair, le **niveau de risque global est CRITIQUE**.

---

## C. Résultats BeVigil

BeVigil analyse la **surface d'attaque externe** visible publiquement.

### Endpoints détectés (11 total)

```
https://api.fakebank.pro/v2/auth/login
https://api.fakebank.pro/v2/auth/refresh
https://api.fakebank.pro/v2/users/{id}/profile
https://api.fakebank.pro/v2/transactions/history
https://api.fakebank.pro/v2/transfer/send
http://legacy.fakebank.pro/api/v1/balance         ← ⚠️ HTTP non chiffré
http://legacy.fakebank.pro/api/v1/notifications    ← ⚠️ HTTP non chiffré
https://api.fakebank.pro/v2/admin/users            ← ⚠️ Endpoint admin exposé
https://cdn.fakebank.pro/assets/
https://analytics.fakebank.pro/track
https://maps.googleapis.com/maps/api/geocode/json
```

### Secrets détectés par BeVigil

```json
{
  "secrets_found": [
    {
      "type": "Google Maps API Key",
      "value": "AIzaSyD4fK9mN2pR7vX1qL8wE3tY6uI0oJ5hG",
      "location": "res/values/strings.xml",
      "line": 47
    },
    {
      "type": "Firebase Config",
      "value": "1:847392610:android:a3f9c2e1b4d7a8f2",
      "location": "google-services.json",
      "line": 12
    },
    {
      "type": "JWT Token (Admin)",
      "value": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJBRE1JTiJ9.XXXXX",
      "location": "assets/config.json",
      "line": 3
    }
  ],
  "urls_found": 11,
  "domains_found": 4,
  "technologies": ["Retrofit2", "OkHttp", "Firebase", "Google Maps SDK"]
}
```

---

## D. Résultats Yaazhini

Yaazhini **décompile l'APK** et inspecte son contenu interne.

### Permissions déclarées (AndroidManifest.xml)

| Permission | Risque | Justification |
|------------|--------|---------------|
| `READ_CONTACTS` | 🟠 Dangereux | Aucune fonctionnalité visible ne le justifie |
| `ACCESS_FINE_LOCATION` | 🟡 Dangereux | Géolocalisation ATM — justifiée |
| `CAMERA` | 🟡 Dangereux | Scan de chèque — justifiée |
| `READ_EXTERNAL_STORAGE` | 🟠 Dangereux | Accès trop large |
| `RECORD_AUDIO` | 🔴 Suspect | Aucune fonctionnalité audio visible |
| `INTERNET` | Normal | Nécessaire |
| `RECEIVE_BOOT_COMPLETED` | 🟡 Modéré | Notifications au démarrage |

### Flags suspects dans le Manifest

```xml
android:debuggable="true"           <!-- ⚠️ Debug actif en production -->
android:allowBackup="true"          <!-- ⚠️ Backup ADB autorisé -->
android:usesCleartextTraffic="true" <!-- ⚠️ HTTP autorisé -->
```

### Secrets dans le code décompilé

```java
// com/fakebank/pro/network/ApiClient.java - ligne 23
private static final String ADMIN_TOKEN =
    "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJBRE1JTiJ9.mK9xP2vQ";

// com/fakebank/pro/utils/Constants.java - ligne 8
public static final String DB_PASSWORD = "fakebank2024!";

// res/values/strings.xml - ligne 47
<string name="maps_key">AIzaSyD4fK9mN2pR7vX1qL8wE3tY6uI0oJ5hG</string>
```

### Stockage non sécurisé

```java
// com/fakebank/pro/auth/LoginActivity.java - ligne 156
SharedPreferences prefs = getSharedPreferences("user_data", MODE_PRIVATE);
prefs.edit().putString("password", userPassword).apply(); // ⚠️ Mot de passe en clair
prefs.edit().putString("pin_code", pinCode).apply();      // ⚠️ PIN en clair
```

---

## E. Constats détaillés

### 🔴 FIND-001 — Token JWT Admin hardcodé
| | |
|--|--|
| **Sévérité** | CRITICAL |
| **Source** | BeVigil + Yaazhini |
| **Localisation** | `assets/config.json:3` · `ApiClient.java:23` |
| **CVSS** | 9.8 |

**Description :** Un token JWT avec rôle `ADMIN` est hardcodé dans le code. Il ne expire jamais et donne accès complet à `/v2/admin/users`.

**Impact :**
- Accès à tous les comptes utilisateurs
- Modification / suppression de comptes
- Extraction de données financières de tous les clients

**Remédiation :**
1. Invalider immédiatement le token côté serveur
2. Utiliser Android Keystore pour tout credential
3. Implémenter une rotation de tokens (TTL court)
4. Protéger les endpoints admin avec authentification mutuelle (mTLS)

**Référence OWASP :** `MASVS-STORAGE-1` · `MASVS-AUTH-2`

---

### 🟠 FIND-002 — Mot de passe stocké en clair
| | |
|--|--|
| **Sévérité** | High |
| **Source** | Yaazhini |
| **Localisation** | `LoginActivity.java:156` |
| **CVSS** | 7.5 |

**Description :** Mot de passe et PIN stockés en clair dans SharedPreferences, accessible sur appareil rooté ou via backup ADB.

**Remédiation :**
1. Utiliser `EncryptedSharedPreferences` (Jetpack Security)
2. Ne stocker que le hash bcrypt du mot de passe
3. Désactiver `android:allowBackup`

**Référence OWASP :** `MASVS-STORAGE-1`

---

### 🟠 FIND-003 — Clé API Google Maps exposée
| | |
|--|--|
| **Sévérité** | High |
| **Source** | BeVigil |
| **Localisation** | `res/values/strings.xml:47` |
| **CVSS** | 7.2 |

**Description :** Clé API extractible directement depuis l'APK décompilé.

**Impact :** Utilisation abusive, facturation excessive, accès à d'autres APIs Google.

**Remédiation :**
1. Restreindre la clé à l'empreinte SHA-1 du certificat + package name
2. Stocker côté serveur et récupérer via API authentifiée

**Référence OWASP :** `MASVS-STORAGE-1`

---

### 🟡 FIND-004 — Endpoints HTTP non chiffrés
| | |
|--|--|
| **Sévérité** | Medium |
| **Source** | BeVigil + Yaazhini |
| **CVSS** | 5.9 |

**Description :** 2 endpoints legacy utilisent HTTP en clair. Le manifest autorise `usesCleartextTraffic`.

**Impact :** Attaque Man-in-the-Middle sur Wi-Fi public, interception du solde bancaire.

**Remédiation :** Migrer vers HTTPS, désactiver `usesCleartextTraffic`.

**Référence OWASP :** `MASVS-NETWORK-1`

---

### 🟡 FIND-005 — Permission RECORD_AUDIO non justifiée
| | |
|--|--|
| **Sévérité** | Medium |
| **Source** | Yaazhini |
| **CVSS** | 5.3 |

**Description :** Permission microphone déclarée sans fonctionnalité audio visible dans le code.

**Remédiation :** Supprimer la permission. Si utilisée, documenter le cas d'usage.

**Référence OWASP :** `MASVS-PRIVACY-1`

---

### 🟢 FIND-006 — Mode debug en production
| | |
|--|--|
| **Sévérité** | Low |
| **CVSS** | 3.1 |

**Remédiation :** `buildTypes { release { debuggable false } }` dans build.gradle

**Référence OWASP :** `MASVS-RESILIENCE-2`

---

### 🟢 FIND-007 — Backup ADB autorisé
| | |
|--|--|
| **Sévérité** | Low |
| **CVSS** | 2.8 |

**Remédiation :** Désactiver `android:allowBackup` ou exclure les données sensibles.

**Référence OWASP :** `MASVS-STORAGE-8`

---

## F. Faux positifs

| Élément signalé | Raison du rejet |
|-----------------|-----------------|
| `"password"` dans `strings.xml:12` | Hint du champ de formulaire, pas un secret |
| `/api/public/health` | Endpoint de monitoring public par design |
| `"test_key_do_not_use"` dans `debug/` | Clé de test non utilisée en production |

---

## G. Recommandations prioritaires

1. 🔴 **URGENT** — Invalider le token JWT admin (FIND-001)
2. 🟠 **IMPORTANT** — Chiffrer le stockage local (FIND-002)
3. 🟠 **IMPORTANT** — Protéger la clé Google Maps (FIND-003)
4. 🟡 **NORMAL** — Migrer vers HTTPS complet (FIND-004)
5. 🟡 **NORMAL** — Supprimer les permissions inutiles (FIND-005)

---

## H. Checklist de fin d'analyse

- [x] Scope clairement défini et respecté
- [x] Hash SHA-256 documenté
- [x] Exports BeVigil sauvegardés — `01-bevigil/bevigil_export.json`
- [x] Rapport Yaazhini sauvegardé — `02-yaazhini/yaazhini_report.txt`
- [x] 7 constats avec mapping OWASP
- [x] Faux positifs identifiés et justifiés
- [x] Aucun secret réel exposé dans ce rapport

---

## I. Annexes

**Permissions dangereuses :**
`READ_CONTACTS` · `ACCESS_FINE_LOCATION` · `CAMERA` · `READ_EXTERNAL_STORAGE` · `RECORD_AUDIO`

**Technologies détectées :**
`Retrofit2` · `OkHttp 4.9.1` · `Firebase Analytics` · `Google Maps SDK 18.1.0` · `Room Database`

**Structure des fichiers d'analyse :**
```
lab-mobile-security/
├── 00-scope/
│   ├── fakebank_pro_3.1.0.apk
│   └── analyse_info.txt
├── 01-bevigil/
│   └── bevigil_export.json
├── 02-yaazhini/
│   ├── yaazhini_report.txt
│   └── yaazhini_notes.md
├── 03-triage/
│   └── triage.csv
└── README.md   ← ce fichier
```
