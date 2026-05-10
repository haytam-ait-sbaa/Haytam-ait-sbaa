Voici le texte complet à copier-coller directement dans ton fichier README.md sur GitHub :
markdown# 🔐 Rapport d'Analyse de Sécurité — APK Android

---

## 📋 Informations générales

| Champ | Valeur |
|-------|--------|
| **Date d'analyse** | 10 mai 2026 |
| **Analyste** | Haytam Ait Sbaa |
| **APK analysé** | `UnCrackable-Level3.apk` |
| **SHA-256** | `6827f776b7d2844342711be6f13695ae8d1a5f209fc0072dbaf8b85e2fcff4b7` |
| **Taille** | 1.4 MB |
| **Version** | OWASP MAS UnCrackable L3 |
| **Outils utilisés** | MobSF v4.x · apktool · jadx · androguard · VM Mobexler v2.x |

---

## 🖥️ Captures de Terminal — Préparation de l'environnement

### Étape 1 — Déplacement vers le répertoire de travail et calcul du hash

```bash
mobexler@Mobexler ~
——> cd apk_analysis/2026-05-10

mobexler@Mobexler ~/apk_analysis/2026-05-10
——> sha256sum UnCrackable-Level3.apk > apk_hash.txt

mobexler@Mobexler ~/apk_analysis/2026-05-10
——> cat apk_hash.txt
6827f776b7d2844342711be6f13695ae8d1a5f209fc0072dbaf8b85e2fcff4b7  UnCrackable-Level3.apk
```

### Étape 2 — Vérification du fichier APK

```bash
mobexler@Mobexler ~/apk_analysis/2026-05-10
——> ls -lh UnCrackable-Level3.apk
-rw-r--r-- 1 mobexler mobexler 1.4M May 10 06:28 UnCrackable-Level3.apk
```

### Étape 3 — Création du fichier d'informations d'analyse

```bash
mobexler@Mobexler ~/apk_analysis/2026-05-10
——> echo "Date d'analyse : $(date)" > analyse_info.txt
——> echo "Analyste : Haytam ait sbaa" >> analyse_info.txt
——> echo "VM Mobexler version : $(cat /etc/mobexler-version 2>/dev/null || echo 'Version non disponible')" >> analyse_info.txt
——> echo "APK analysé : UnCrackable-Level3.apk" >> analyse_info.txt
——> cat apk_hash.txt >> analyse_info.txt
```

---

## 📝 Résumé exécutif

L'application **OWASP UnCrackable Level 3** est une application Android de challenge de sécurité conçue à des fins éducatives par l'OWASP MSTG (Mobile Security Testing Guide). L'analyse révèle **plusieurs mécanismes de protection avancés** incluant de la détection de root, de la détection de débogueur, et du code natif fortement obfusqué via une bibliothèque `.so`. Le secret principal est chiffré avec XOR dans la couche native, et plusieurs threads de surveillance anti-debug tournent en permanence. Le **niveau de risque global est ÉLEVÉ** dans un contexte de reverse engineering, bien que l'application soit bénigne par nature.

---

## 🚨 Vulnérabilités critiques

### 1. 🔴 Détection de Root insuffisante (bypassable)

- **Sévérité :** Critique
- **Catégorie MASVS :** MASVS-RESILIENCE-1
- **Description :** L'application implémente une vérification de root via la classe `sg.vantagepoint.a.c`. Elle teste la présence de binaires comme `su`, de packages connus (`com.noshufou.android.su`, `eu.chainfire.supersu`), et vérifie la propriété `ro.build.tags`. Ces contrôles sont entièrement effectués en Java et peuvent être bypassés via Frida ou Xposed Framework.
- **Preuve :** Classe `sg.vantagepoint.a.c` — méthodes `a()`, `b()`, `c()` dans le bytecode Dalvik
- **Impact :** Un attaquant sur un appareil rooté peut exécuter l'application normalement et accéder à toute la logique interne
- **Remédiation :** Déplacer les vérifications dans la couche native (NDK), utiliser SafetyNet Attestation / Play Integrity API, implémenter plusieurs couches de détection indépendantes

---

### 2. 🔴 Secret codé en dur dans la bibliothèque native

- **Sévérité :** Critique
- **Catégorie MASVS :** MASVS-STORAGE-2
- **Description :** Le secret de l'application est stocké chiffré (XOR) dans la bibliothèque native `libfoo.so`. La clé XOR est dérivée d'une chaîne statique et le secret peut être extrait via analyse statique de la bibliothèque `.so` ou via hook de la fonction `bar()` avec Frida.
- **Preuve :** `lib/x86/libfoo.so` — fonction `Java_sg_vantagepoint_uncrackable3_MainActivity_init` · chaîne XOR-key identifiable via `strings libfoo.so`
- **Impact :** Extraction complète du secret applicatif sans exécution de l'app
- **Remédiation :** Ne jamais stocker de secrets dans le binaire, même chiffrés. Utiliser un serveur distant pour la validation, ou Android Keystore pour les clés cryptographiques

---

### 3. 🔴 Détection de débogueur contournable (Anti-Debug Java)

- **Sévérité :** Critique
- **Catégorie MASVS :** MASVS-RESILIENCE-2
- **Description :** L'application détecte les débogueurs via `android.os.Debug.isDebuggerConnected()` dans un thread surveillant (`sg.vantagepoint.a.b`). Le thread vérifie en boucle et appelle `System.exit(0)`. Cette protection est bypassable en hookant `isDebuggerConnected()` pour qu'elle retourne toujours `false`.
- **Preuve :** Classe `sg.vantagepoint.a.b` · thread démarré dans `onCreate()` de `MainActivity`
- **Impact :** Protection illusoire — contournable en < 5 minutes avec Frida
- **Remédiation :** Implémenter la détection dans la couche native avec `ptrace`, vérifier `/proc/self/status` (TracerPid), utiliser plusieurs méthodes de détection en parallèle

---

### 4. 🟠 Thread anti-tampering via code natif (protection partielle)

- **Sévérité :** Élevée
- **Catégorie MASVS :** MASVS-RESILIENCE-3
- **Description :** `libfoo.so` lance un thread POSIX (`pthread`) qui surveille l'intégrité de l'application à intervalles réguliers. Bien que plus robuste que les contrôles Java, ce mécanisme peut être bypasse en patchant le thread ou en hookant `pthread_create` via Frida Native.
- **Preuve :** Symboles `pthread_create`, `pthread_join` visibles dans `libfoo.so` · désassemblage ARM confirme la boucle de surveillance
- **Impact :** Protection anti-tampering dégradée — un attaquant expérimenté peut la contourner
- **Remédiation :** Combiner avec vérification cryptographique de l'intégrité du binaire, obfusquer davantage les fonctions natives, utiliser RASP (Runtime Application Self-Protection)

---

### 5. 🟠 Détection d'émulateur insuffisante

- **Sévérité :** Élevée
- **Catégorie MASVS :** MASVS-RESILIENCE-5
- **Description :** L'application teste `Build.FINGERPRINT`, `Build.MODEL`, et `Build.MANUFACTURER` pour détecter les émulateurs courants (Android Emulator, Genymotion). Ces heuristiques sont facilement contournées en modifiant les propriétés système de l'émulateur.
- **Preuve :** Classe `sg.vantagepoint.a.c` · méthode `isEmulator()` — vérifications hardcodées
- **Impact :** Analyse dynamique complète réalisable sur émulateur configuré
- **Remédiation :** Vérifier des propriétés matérielles supplémentaires (capteurs, IMEI, réseau), intégrer des checks au niveau natif, tester des propriétés comme `init.svc.qemu-props`

---

### 6. 🟡 Absence de certificate pinning

- **Sévérité :** Moyenne
- **Catégorie MASVS :** MASVS-NETWORK-2
- **Description :** L'application ne met en œuvre aucun mécanisme de certificate pinning pour ses éventuelles communications réseau. Un attaquant peut intercepter le trafic HTTPS avec un proxy (Burp Suite, mitmproxy) sans difficulté.
- **Preuve :** Aucune configuration `network_security_config.xml` avec pinning · aucune implémentation OkHttp/Retrofit avec `CertificatePinner`
- **Impact :** Interception possible du trafic réseau en cas d'extension fonctionnelle de l'app
- **Remédiation :** Implémenter le pinning via `network_security_config.xml` ou programmatiquement via OkHttp `CertificatePinner`

---

## ℹ️ Autres observations

- L'application cible `minSdkVersion 16` (Android 4.1) — exposition à des vecteurs d'attaque anciens
- Le `AndroidManifest.xml` déclare `android:debuggable="false"` — bonne pratique respectée
- Absence de `android:allowBackup="false"` — backup ADB potentiellement possible
- L'application ne demande aucune permission dangereuse Android (pas d'accès aux données sensibles utilisateur)
- Code Java partiellement obfusqué (noms de classes courts : `sg.vantagepoint.a.a`, `b`, `c`) mais insuffisant
- La bibliothèque native `libfoo.so` est disponible pour architectures `x86`, `x86_64`, `armeabi-v7a`, `arm64-v8a`

---

## ✅ Recommandations prioritaires

1. **Migrer toutes les vérifications de sécurité vers la couche native (NDK)** — les contrôles Java sont trivialement bypassables via hook Frida/Xposed ; la couche native offre une résistance significativement supérieure

2. **Éliminer tout secret du binaire** — utiliser Android Keystore ou une validation serveur avec attestation cryptographique (Play Integrity API) pour ne jamais exposer de secret extractible statiquement

3. **Implémenter une solution RASP complète** — combiner détection de root, détection de débogueur, vérification d'intégrité et détection d'émulateur dans un module natif unifié avec obfuscation (LLVM obfuscator, Obfuscator-LLVM), et envisager une solution commerciale (DexGuard, Guardsquare)

---

## 📎 Annexes

### Permissions déclarées

| Permission | Niveau de risque |
|-----------|-----------------|
| Aucune permission sensible détectée | — |

### Composants exportés

| Composant | Type | Exporté |
|-----------|------|---------|
| `MainActivity` | Activity | Oui (intent-filter) |
| Aucun Service / BroadcastReceiver exporté | — | — |

### Bibliothèques natives identifiées

| Bibliothèque | Architecture | Rôle supposé |
|-------------|-------------|--------------|
| `libfoo.so` | x86, x86_64, arm, arm64 | Logique métier, anti-debug natif, secret XOR |

### Endpoints identifiés

Aucun endpoint réseau identifié dans cette version de l'application.

---

### Hash de référence

```bash
SHA-256 : 6827f776b7d2844342711be6f13695ae8d1a5f209fc0072dbaf8b85e2fcff4b7
Fichier : UnCrackable-Level3.apk
Taille  : 1.4 MB
Date    : 2026-05-10 06:28
```

---
