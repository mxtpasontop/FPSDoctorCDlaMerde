## Write up rédigé avec Claude Opus 4.8

# FPSDoctor — Analyse technique complète

> **Enquête technique indépendante sur l'application FPSDoctor**
> Ce que le logiciel fait réellement à ton PC · ce qu'il te facture · et comment tout obtenir gratuitement.
>
> *Document factuel basé sur une analyse statique des binaires. Toutes les commandes, clés et dépendances citées sont extraites directement des exécutables — rien n'est inventé. Chacun est libre d'en tirer ses propres conclusions.*

---

## Sommaire

1. [TL;DR — le résumé en 30 secondes](#1-tldr)
2. [Avertissement et cadre de l'analyse](#2-avertissement)
3. [Qu'est-ce que FPSDoctor, techniquement ?](#3-architecture)
4. [Les composants détaillés](#4-composants)
5. [Ce que l'application fait VRAIMENT à ton PC](#5-ce-que-ca-fait)
6. [Liste complète des tweaks (avec impact FPS réel)](#6-liste-tweaks)
7. [Gratuit vs Premium — ce que le paywall verrouille](#7-gratuit-vs-premium)
8. [Les risques de sécurité](#8-securite)
9. [Le système de licence (abonnement signé)](#9-licence)
10. [Le logiciel communique-t-il sur Internet ?](#10-reseau)
11. [Verdict](#11-verdict)
12. [FAQ](#12-faq)
13. [Méthodologie et reproductibilité](#13-methodologie)
14. [Annexe A — Pile technique (dépendances)](#14-annexe-stack)
15. [Annexe B — Glossaire](#15-glossaire)
16. [🧰 ALTERNATIVES GRATUITES — commandes + logiciels open source](#16-alternatives)

---

<a name="1-tldr"></a>
## 1. TL;DR — le résumé en 30 secondes

FPSDoctor est une **interface graphique payante** qui exécute, en coulisses, des **commandes Windows gratuites et bien documentées** (`bcdedit`, `powercfg`, `reg`, `schtasks`, suppression d'apps, changement de DNS, tweaks réseau).

- Les fonctions « **Premium** » (optimisation DNS, panneau NVIDIA, Game Boost, Game Mode, diagnostic) sont des réglages que **tu peux faire toi-même gratuitement en quelques clics**.
- Certaines modifications **affaiblissent la sécurité de Windows** (désactivation de la virtualisation / VBS).
- L'application **installe un pilote noyau** (PawnIO, niveau ring 0).
- Le gain FPS réel va de « **faible** » à « **placebo** » selon le tweak.

> **Tu ne paies pas pour une technologie. Tu paies un abonnement pour un bouton qui lance des commandes gratuites.**

---

<a name="2-avertissement"></a>
## 2. Avertissement et cadre de l'analyse

- Cette analyse est **purement statique** : les binaires n'ont **pas été exécutés**. On a seulement lu leur contenu (extraction de chaînes de caractères, identification des frameworks et des dépendances).
- Aucune donnée personnelle, aucun compte, aucun jeton de licence n'a été utilisé.
- Le but de ce document est **l'information du consommateur** : permettre à chacun de comprendre ce qu'il achète et de décider en connaissance de cause.
- Ce document **ne contient aucun contournement de licence** et n'encourage pas le piratage. La bonne réponse à un produit au rapport qualité/prix discutable, c'est l'**information**, pas la copie illégale.

---

<a name="3-architecture"></a>
## 3. Qu'est-ce que FPSDoctor, techniquement ?

FPSDoctor est une application **Tauri 2** :

- **Cœur applicatif en Rust** (le « backend » qui exécute les actions système).
- **Interface en SvelteKit** (HTML/CSS/JS), affichée dans **WebView2** — le moteur de rendu d'Edge intégré à Windows.
- Les fichiers de l'interface sont **compressés (brotli) et embarqués directement dans l'exécutable** (via une table `phf`), puis servis en interne via le protocole `tauri://localhost`.
- Plusieurs **exécutables auxiliaires (« sidecars »)** se chargent du monitoring et des tweaks.

**Marqueurs qui le confirment (extraits du binaire) :** `__TAURI_INVOKE_KEY__`, `tauri://localhost`, `src\main.rs`, `EBWebView` (dossier de données WebView2), `%LOCALAPPDATA%\FPSDoctor\EBWebView`, `Global\FPSDoctor_SingleInstance`, `/_app/immutable/...` (structure de build SvelteKit).

> **À noter :** tous les messages internes du logiciel sont rédigés en **français** — l'éditeur développe en français.

---

<a name="4-composants"></a>
## 4. Les composants détaillés

| Fichier | Taille | Rôle |
|---|---:|---|
| `FPSDoctor.exe` | ~66 Mo | Application principale (interface SvelteKit + cœur Rust/Tauri) |
| `fps-doctor-sidecar.exe` | ~1,6 Mo | **Le moteur de tweaks** — exécute toutes les modifications système |
| `fps-doctor-hardware.exe` | ~80 Mo | Monitoring matériel (taille compatible avec une bibliothèque de type **LibreHardwareMonitor** embarquée) |
| `fps-doctor-gpuload.exe` | ~0,7 Mo | Test de charge / stress GPU |
| `fps-doctor-capture.exe` | ~0,5 Mo | Capture des FPS et des temps d'image (approche type PresentMon) |
| `resources\PawnIO_setup.exe` | ~3,4 Mo | Installe **PawnIO**, un **pilote noyau** (ring 0) pour l'accès matériel bas niveau |
| `resources\THIRD-PARTY-NOTICES.txt` | ~150 Ko | Licences des composants tiers utilisés |
| `uninstall.exe` | ~0,45 Mo | Désinstalleur |

**Champs de télémétrie matérielle exposés** (extraits du binaire) : `coreClockMhz`, `memClockMhz`, `voltage`, `tempC`, `fanRpm`, `powerW` → fréquences cœur/mémoire, tension, température, ventilateur, consommation. C'est du monitoring GPU/CPU classique — exactement ce que font des outils gratuits comme HWiNFO ou MSI Afterburner.

---

<a name="5-ce-que-ca-fait"></a>
## 5. Ce que l'application fait VRAIMENT à ton PC

Le moteur (`fps-doctor-sidecar.exe`) lance des commandes Windows standard via `cmd.exe /C`, `powershell -NoProfile` et des écritures directes dans le Registre. Voici les commandes **vérifiées, extraites telles quelles du binaire** :

### Démarrage et noyau
```bat
bcdedit /set {current} hypervisorlaunchtype off     :: désactive Hyper-V / la virtualisation (VBS)
bcdedit /set {current} quietboot on                 :: démarrage "silencieux"
bcdedit /timeout 3                                   :: réduit le délai du menu de boot (défaut 30)
powercfg /hibernate off                              :: désactive l'hibernation
```
+ création d'un **plan d'alimentation personnalisé** nommé **« Plan d'alimentation FPSDoctor »** (sauvegarde de l'ancien plan via `power_plan_backup`).

### Suppression d'applications intégrées (débloat)
```powershell
# Widgets
Get-AppxPackage Microsoft.WidgetsPlatformRuntime -AllUsers | Remove-AppxPackage -AllUsers
# Web Experience (widgets/actus)
Get-AppxPackage MicrosoftWindows.Client.WebExperience -AllUsers | Remove-AppxPackage -AllUsers
```
+ désactivation de **tâches planifiées** via `schtasks /Change /TN "..."`.

### Modifications du Registre (`reg` + `RegFlushKey`)
Désactive :
- l'**historique d'activité** (`disable_activity_history`)
- les **applications en arrière-plan**
- **Windows Recall** (captures d'écran continues)
- **Storage Sense** (nettoyage auto des fichiers temporaires)
- les **effets visuels et animations**
- la **reconnaissance vocale en ligne** / dictée
- les **suggestions et contenus sponsorisés** Windows
- **Windows Insider & rapports d'erreurs**
- l'**algorithme de Nagle** (tweaks réseau `TcpAckFrequency` / `TCPNoDelay`)
- les recommandations du **menu Démarrer**, les **publicités** et suggestions d'apps du Store

### Diagnostic matériel
```powershell
Get-CimInstance Win32_PhysicalMemory | Select-Object -ExpandProperty SMBIOSMemoryType   # type de RAM (DDR4/DDR5)
```
+ détection de **Docker** et de l'état de **virtualisation** (hyperviseur présent, VT dans le firmware, drapeau CPU).

👉 **Toutes ces commandes sont publiques, documentées et gratuites.** Ce sont exactement les tweaks que l'on trouve dans les scripts de « debloat/optimisation gaming » communautaires open source depuis des années.

---

<a name="6-liste-tweaks"></a>
## 6. Liste complète des tweaks (avec impact FPS réel)

Évaluation honnête de l'impact sur les FPS en jeu. *(Placebo = effet nul ou non mesurable sur les FPS ; peut avoir un intérêt « confort » mais pas « performance ».)*

| Tweak (nom interne traduit) | Ce que ça fait | Impact FPS réel |
|---|---|---|
| Désactiver la virtualisation (VBS/Hyper-V) | `bcdedit hypervisorlaunchtype off` | ⚠️ **Faible à modéré** (réel) — **mais réduit la sécurité** |
| Optimisation latence réseau (Nagle off) | `TcpAckFrequency`/`TCPNoDelay` | Faible (ping, pas FPS) |
| Optimisation de la carte réseau | réglages NIC | Faible (stabilité ping) |
| Désactiver le P2P des mises à jour | Delivery Optimization off | Faible (bande passante) |
| Désactiver l'hibernation | `powercfg /hibernate off` | **Placebo FPS** (libère de l'espace disque) |
| Désactiver les effets visuels | Registre | **Placebo** (anciens PC uniquement) |
| Désactiver l'historique d'activité | Registre | **Placebo** (vie privée, pas FPS) |
| Désactiver Windows Recall | Registre | **Placebo FPS** (utile pour la vie privée) |
| Désactiver Storage Sense | Registre | **Placebo** |
| Désactiver les apps en arrière-plan | Registre | Négligeable à faible |
| Désactiver les suggestions/pubs Windows | Registre | **Placebo** (confort) |
| Supprimer les Widgets | `Remove-AppxPackage` | **Placebo FPS** (confort) |
| Supprimer OneDrive | désinstallation | Négligeable |
| Optimiser la souris pour le jeu | désactive l'accélération souris | **0 FPS** (précision de visée, pas FPS) |
| Optimiser l'Explorateur Windows | Registre | **Placebo** |
| Optimisation Edge | Registre | **Placebo** |
| Plan d'alimentation « FPSDoctor » | `powercfg` | Faible (surtout sur portable) |
| quietboot / timeout boot | `bcdedit` | **0 FPS** (démarrage uniquement) |

**Conclusion de la section :** un seul tweak (désactivation VBS) a un impact FPS réel notable — et c'est précisément celui qui **réduit la sécurité**. Le reste relève du confort, de la vie privée, ou du placebo pur côté performances.

---

<a name="7-gratuit-vs-premium"></a>
## 7. Gratuit vs Premium — ce que le paywall verrouille

Les fonctions suivantes sont **verrouillées derrière l'abonnement** (clés internes `*_premium_required` / `premium_only`, extraites du binaire) :

| Fonction Premium | Clé interne | Ce que c'est **vraiment** |
|---|---|---|
| Optimisation DNS | `DNS_PREMIUM_REQUIRED`, `network.dns` | Changer ton serveur DNS |
| Optimisation panneau NVIDIA | `opti.drs_premium_only` | Régler le Panneau de configuration NVIDIA |
| Game Boost | `games.boost_premium_required` | Fermer des processus + tweaks déjà listés |
| Game Mode | `game_mode.premium_required` | Activer le Mode Jeu (déjà dans Windows) |
| Check-up / diagnostic complet | `checkup.premium_required` | Scan CPU/GPU/VRAM/RAM/disque/crash |
| Page vidéo tutoriel Premium | `PremiumVideoPage`, `/badges/premium.png` | Contenu vidéo |

**Message interne affiché à l'utilisateur non-Premium (verbatim) :**
> *« The NVIDIA panel optimization is reserved for Premium members. Go Premium to unlock this performance boost. »*

**Modules scannés par le « Check-up » :** Background, CpuInventory, CrashHistory, DiskHealth, CpuSingle, CpuMulti, Gpu, Vram, Ram, Combined, DiskTest.

> **En clair : l'abonnement déverrouille des réglages que Windows et NVIDIA te donnent déjà gratuitement.** Voir la [section Alternatives](#16-alternatives) pour tout refaire gratuitement.

---

<a name="8-securite"></a>
## 8. Les risques de sécurité

### 8.1 Désactivation de la virtualisation / VBS
`bcdedit /set {current} hypervisorlaunchtype off` désactive **VBS (Virtualization-Based Security)**.
- ✅ Peut donner **quelques % de FPS** sur certaines machines (tweak réel).
- ❌ **Réduit la sécurité** : VBS protège notamment contre certaines attaques mémoire (vol d'identifiants, etc.).
- ❌ Peut **casser certains jeux** avec anti-cheat exigeant VBS/HVCI activé.

### 8.2 Installation d'un pilote noyau (PawnIO)
PawnIO tourne en **ring 0** (niveau noyau) pour lire les capteurs matériels.
- C'est un pilote **légitime** (utilisé par LibreHardwareMonitor récent).
- **Mais** tout pilote noyau **augmente la surface d'attaque** : un bug dans un pilote ring 0 = compromission potentielle de tout le système.
- Tu installes du **code noyau permanent** pour, essentiellement, un **confort de monitoring**.

### 8.3 Décharge de responsabilité intégrée
Le logiciel gère un **jeton de « waiver » / renonciation** (`FPSDOCTOR_WAIVER_TOKEN`) : une **décharge de responsabilité** demandée à l'utilisateur avant les tweaks risqués. C'est le signe que l'éditeur **sait** que certaines actions peuvent causer des problèmes.

> **Bilan sécurité :** un outil qui **désactive une protection** et **installe un pilote noyau** pour lancer des commandes gratuites mérite qu'on pèse sérieusement le rapport bénéfice/risque.

---

<a name="9-licence"></a>
## 9. Le système de licence (abonnement signé)

La licence repose sur des **jetons signés cryptographiquement** :

- **Bibliothèque :** crate Rust **`minisign-verify`** (v0.2.4) → signatures **Ed25519** (primitives `curve25519`, `blake2b`, `sha512`, `base64`).
- **Format :** `payload.signature`, vérifié contre une **clé publique embarquée** dans le logiciel.
- **Deux jetons** (variables d'environnement) :
  - `FPSDOCTOR_PLAN_TOKEN` → la licence / le plan d'abonnement
  - `FPSDOCTOR_WAIVER_TOKEN` → la décharge de responsabilité
- **États gérés :** `absent`, `invalide`, `expiré` → les jetons **peuvent expirer** (modèle par abonnement).

**Messages d'erreur internes (verbatim) :**
> *« signature invalide (jeton altéré, ou serveur et logiciel signés avec des clés différentes) »*
> *« format inattendu (pas de séparateur entre contenu et signature) »*
> *« version de jeton inconnue »*
> *« Jeton de plan refusé : … »*

**Ce que ça implique :** c'est un système d'abonnement verrouillé côté serveur, cryptographiquement solide. *(C'est aussi pourquoi « cracker » l'app n'a pas de sens : la bonne réponse est d'informer les gens — ce que fait ce document — pas de redistribuer une copie illégale et potentiellement modifiée d'un logiciel qui touche au noyau.)*

---

<a name="10-reseau"></a>
## 10. Le logiciel communique-t-il sur Internet ?

Oui. Les dépendances réseau présentes dans le binaire le confirment : **`reqwest`** (client HTTP), **`hyper`**, **`rustls`** (TLS), avec une contrainte **HTTPS obligatoire** (chaîne interne *« url non https »* → rejet des URL non chiffrées). Le logiciel effectue donc des appels réseau — typiquement : validation/récupération de licence, vérification de version, et potentiellement de la télémétrie. *(L'analyse statique montre la capacité réseau ; déterminer le détail exact des requêtes demanderait une analyse dynamique — non réalisée ici.)*

---

<a name="11-verdict"></a>
## 11. Verdict

FPSDoctor est une **surcouche graphique soignée** autour de :
- commandes Windows **gratuites et publiques**,
- réglages NVIDIA / DNS **gratuits**,
- quelques tweaks au **gain FPS réel faible à nul**,
- le tout **payant par abonnement** (jetons signés qui expirent),
- avec des modifications qui **réduisent la sécurité** et un **pilote noyau** à la clé.

**Ce n'est pas une arnaque magique « +200 FPS » — c'est un paywall posé sur des réglages gratuits.** Pour l'immense majorité des utilisateurs, le rapport prix/valeur est mauvais, et **tout est reproductible gratuitement** (voir ci-dessous).

---

<a name="12-faq"></a>
## 12. FAQ

**« Est-ce que ça va vraiment booster mes FPS ? »**
Marginalement, et surtout via la désactivation de VBS (qui réduit la sécurité). Le reste est du confort ou du placebo côté FPS.

**« C'est un virus / malware ? »**
Rien dans l'analyse statique n'indique un malware classique. Le risque est ailleurs : désactivation de protections + pilote noyau + abonnement pour des réglages gratuits.

**« Puis-je annuler les changements ? »**
L'app sauvegarde l'ancien plan d'alimentation (`power_plan_backup`) et propose une restauration. Mais la suppression d'apps (Widgets, OneDrive) et les modifications de registre se **ré-appliquent manuellement** si besoin. Fais un **point de restauration système** avant toute manipulation.

**« Pourquoi payer alors que c'est gratuit ailleurs ? »**
C'est précisément la question que pose ce document. On paie la **commodité d'un bouton**, pas une technologie exclusive.

---

<a name="13-methodologie"></a>
## 13. Méthodologie et reproductibilité

Analyse **statique** uniquement :
1. Identification des types de fichiers (tous **PE/MZ** Windows).
2. Extraction des chaînes imprimables (ASCII + ISO-8859-1 pour le français) de `FPSDoctor.exe` et `fps-doctor-sidecar.exe`.
3. Identification des frameworks via marqueurs : `__TAURI`, `src\main.rs`, `EBWebView`, structure `/_app/immutable/` (SvelteKit), `minisign-verify` (licence).
4. Extraction des commandes système, des clés i18n (`*_premium_required`), et des versions de crates Rust (chemins `crates.io`).

**Toutes les commandes et chaînes citées sont extraites directement des exécutables.** L'application n'a **jamais été exécutée** (aucun dossier `%LOCALAPPDATA%\FPSDoctor` présent sur la machine d'analyse). Les fichiers d'interface SvelteKit sont compressés en **brotli** dans l'exe ; leur contenu fonctionnel a été reconstruit via les chaînes (le verrouillage Premium étant de toute façon appliqué dans le cœur **Rust**, pas dans le JavaScript).

---

<a name="14-annexe-stack"></a>
## 14. Annexe A — Pile technique (dépendances extraites)

| Domaine | Crate / version |
|---|---|
| Framework desktop | `tauri 2.10.2`, `wry 0.54.3` |
| Runtime async | `tokio 1.49.0` |
| HTTP / réseau | `reqwest 0.12.x`, `hyper 1.8.1`, `rustls 0.23.x` |
| Sérialisation | `serde 1.0.228`, `serde_json 1.0.149` |
| Licence | `minisign-verify 0.2.4` (Ed25519) |
| Assets embarqués | `phf 0.13.1` (table parfaite) + compression **brotli** |
| Divers | `regex 1.12.3`, `url 2.5.8`, `uuid 1.21.0`, `chrono 0.4.43`, `zip 4.6.1`, `anyhow`, `windows 0.61.3` |

---

<a name="15-glossaire"></a>
## 15. Annexe B — Glossaire

- **Tauri** : framework pour créer des apps desktop avec un cœur Rust et une interface web.
- **WebView2 / EBWebView** : moteur de rendu web d'Edge, utilisé pour afficher l'interface.
- **VBS (Virtualization-Based Security)** : isolation matérielle de Windows qui protège certaines données sensibles.
- **Ring 0 / pilote noyau** : code s'exécutant au niveau le plus privilégié du système.
- **minisign / Ed25519** : système de signature cryptographique ; sans la clé privée, impossible de forger un jeton valide.
- **brotli** : algorithme de compression ; ici, utilisé pour empaqueter l'interface dans l'exe.
- **sidecar** : exécutable auxiliaire lancé par l'app principale.

---

<a name="16-alternatives"></a>
## 16. 🧰 ALTERNATIVES GRATUITES — commandes + logiciels open source

> Tout ce que FPSDoctor fait (gratuit **et** Premium), reproduit gratuitement.
> ⚠️ **Avant toute manipulation : crée un point de restauration système** (`Créer un point de restauration` dans le menu Démarrer). Lis chaque commande avant de l'exécuter. PowerShell/CMD à ouvrir **en administrateur**.

### 16.1 Équivalents des fonctions « Premium »

| Fonction Premium FPSDoctor | Alternative gratuite |
|---|---|
| **Optimisation DNS** | Paramètres → Réseau et Internet → (ta connexion) → Modifier l'attribution DNS → Manuel → `1.1.1.1` (Cloudflare) ou `8.8.8.8` (Google). **1 minute.** |
| **Panneau NVIDIA** | Clic droit bureau → *Panneau de configuration NVIDIA* → Gérer les paramètres 3D (mode faible latence : **Ultra** ; mode gestion alim : **Performances maximales** ; VSync selon besoin). |
| **Game Mode** | Paramètres → **Jeux** → **Mode Jeu** → Activé. (Déjà intégré à Windows.) |
| **Game Boost** | Fermer les apps en arrière-plan (Gestionnaire des tâches) + les tweaks ci-dessous. |
| **Check-up / diagnostic** | **HWiNFO** (capteurs), **CrystalDiskInfo** (santé disque), Observateur d'événements (crashs), Gestionnaire des tâches. |

### 16.2 Commandes équivalentes (CMD/PowerShell en admin)

**Démarrage / alimentation**
```bat
:: Désactiver l'hibernation
powercfg /hibernate off

:: Réduire le délai du menu de boot
bcdedit /timeout 3

:: Plan d'alimentation "Performances élevées"
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
:: (ou "Performances ultimes" :)
powercfg -duplicatescheme e9a42b02-d5df-448d-aa00-03f14749eb61
```

> **VBS / Hyper-V** (`bcdedit ... hypervisorlaunchtype off`) : **non recommandé** sauf besoin précis — ça réduit la sécurité et peut casser des anti-cheat. Si tu y tiens malgré tout, la commande existe, mais pèse le risque.

**Réseau — latence (algorithme de Nagle)**
Se règle dans le Registre sous `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\...` et par interface. ➡️ Plus simple et réversible via l'outil open source **TCP Optimizer** (voir ci-dessous) que de taper les clés à la main.

**Débloat / confidentialité (exemples réversibles)**
```powershell
# Désactiver l'historique d'activité
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" /v PublishUserActivities /t REG_DWORD /d 0 /f

# Désactiver les suggestions/pubs du menu Démarrer
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\ContentDeliveryManager" /v SubscribedContent-338388Enabled /t REG_DWORD /d 0 /f

# Supprimer les Widgets
winget uninstall "Windows web experience Pack"

# Effets visuels : privilégier les performances
SystemPropertiesPerformance   # (ouvre l'interface Windows, onglet Effets visuels)
```

**Accélération de la souris (visée)**
Paramètres → Souris → Options supplémentaires → Options du pointeur → décocher **« Améliorer la précision du pointeur »**. (Gratuit, intégré.)

### 16.3 Logiciels gratuits / open source recommandés

**🟢 Open source (à privilégier)**

| Logiciel | Usage | Lien |
|---|---|---|
| **LibreHardwareMonitor** | Monitoring complet (temp, fréquences, ventilos, conso) — *c'est très probablement ce que FPSDoctor embarque déjà* | github.com/LibreHardwareMonitor/LibreHardwareMonitor |
| **PresentMon** (Intel) | Capture FPS / frametimes / latence — référence | github.com/GameTechDev/PresentMon |
| **CapFrameX** | Benchmark FPS/frametimes avec interface (basé sur PresentMon) | github.com/CXWorld/CapFrameX |
| **Win11Debloat** | Script de débloat Windows 10/11, simple et transparent | github.com/Raphire/Win11Debloat |
| **Optimizer** | Tweaks confidentialité/perf, open source | github.com/hellzerg/optimizer |
| **ChrisTitusTech WinUtil** | Boîte à outils tweaks/débloat (`irm christitus.com/win \| iex`) | github.com/ChrisTitusTech/winutil |
| **BloatyNosy / AutoDebloat** | Débloat moderne Windows 11 | github.com/builtbybel |

**🔵 Gratuits (fermés mais fiables et connus)**

| Logiciel | Usage |
|---|---|
| **HWiNFO** | Monitoring capteurs (référence) |
| **MSI Afterburner + RivaTuner (RTSS)** | Overlay FPS en jeu, courbe ventilo, OC |
| **CrystalDiskInfo** | Santé des disques (SMART) |
| **TCP Optimizer** (SG) | Tweaks réseau/latence en un clic, réversible |
| **Xbox Game Bar** (Win+G) | Overlay FPS intégré à Windows |
| **NVIDIA App / Control Panel** | Réglages pilote, filtres, faible latence |
| **Process Lasso** (version gratuite) | Gestion d'affinité/priorité CPU |

### 16.4 La méthode « zéro logiciel » (recommandée)

Pour 90 % des utilisateurs, le vrai gain FPS ne vient **pas** de ces tweaks mais de :
1. **Mettre à jour les pilotes GPU** (NVIDIA/AMD/Intel).
2. **Activer le profil XMP/EXPO** de la RAM dans le BIOS (souvent **+5 à +15 % de FPS** réels, gratuit).
3. **Activer Resizable BAR / Smart Access Memory** dans le BIOS si supporté.
4. **Mode Jeu** + **plan d'alimentation Performances** + **fermer les apps inutiles**.
5. **Baisser les réglages graphiques** du jeu (ombres, post-traitement) — de loin le plus efficace.
6. Garder le PC **propre thermiquement** (dépoussiérer, bonne ventilation) — un CPU/GPU qui throttle perd bien plus de FPS que n'importe quel tweak registre.

> Ces 6 points apportent plus de FPS, gratuitement et sans risque, que l'ensemble des tweaks de FPSDoctor réunis.

---

*Document d'analyse technique à visée informative. Les faits présentés sont vérifiables par toute personne disposant des binaires et d'un extracteur de chaînes. Les noms de produits tiers appartiennent à leurs éditeurs respectifs.*
