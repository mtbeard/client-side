# 🔒 Stealth-LSB Vault

> **Outil de stéganographie LSB avancé — chiffrement AES-256-GCM, 100% local, zéro backend.**  
> Cachez du texte ou des fichiers à l'intérieur d'images anodines. Rien ne quitte jamais votre navigateur.

[![Licence: MIT](https://img.shields.io/badge/Licence-MIT-emerald.svg)](LICENSE)
[![Sans dépendances](https://img.shields.io/badge/dépendances-zéro-blue.svg)](index.html)
[![Fonctionne hors-ligne](https://img.shields.io/badge/offline-prêt-success.svg)](index.html)

---

## ✨ Fonctionnalités

- **Stéganographie LSB** — modifie imperceptiblement le bit de poids faible des canaux R, G, B de chaque pixel (le canal Alpha n'est jamais touché).
- **Chiffrement AES-256-GCM** — le payload est chiffré *avant* l'injection. Même avec l'image, rien ne peut être lu sans le mot de passe.
- **Dérivation de clé PBKDF2-SHA256** — 120 000 itérations avec un sel aléatoire de 128 bits par opération d'encodage. Résistant aux attaques par force brute.
- **Cacher du texte ou des fichiers** — encodez un message secret ou n'importe quel fichier binaire (PDF, ZIP, image…) jusqu'à la capacité de l'image.
- **Export PNG sans perte** — l'image stégo est toujours sauvegardée en PNG. La compression JPEG détruirait les bits cachés.
- **Web Worker dédié** — la manipulation des pixels bit à bit s'exécute sur un thread d'arrière-plan. L'interface reste parfaitement réactive même sur de grandes images.
- **Zéro dépendance** — un unique fichier `index.html`. Pas de Node.js, pas d'étape de build, pas de npm. Ouvrez et utilisez.
- **Aucun appel réseau** — l'application tourne entièrement côté client. Vos secrets ne touchent jamais un serveur.

---

## 🚀 Démarrage rapide

```bash
# Option 1 — ouvrir directement le fichier
open index.html

# Option 2 — serveur local (certains navigateurs restreignent file:// pour les Web Workers)
python3 -m http.server 8080
# puis visiter http://localhost:8080
```

Pas d'installation. Pas de build. C'est tout.

---

## 🖥️ Utilisation

### Encodage (cacher des données)

1. Passez en mode **ENCODE** (mode par défaut).
2. Glissez-déposez une **image porteuse** (PNG, JPG, WEBP, BMP — tous les formats sont acceptés en entrée).
3. Le **moniteur de capacité** indique combien d'octets cette image peut dissimuler.
4. Choisissez votre payload :
   - **Onglet Texte** — saisissez ou collez votre message secret.
   - **Onglet Fichier** — sélectionnez n'importe quel fichier (≤ 10 Mo).
5. Entrez un **mot de passe maître** solide.
6. Cliquez sur **ENCODE & INJECT**.
7. Le navigateur télécharge un fichier `*_steg.png` — partagez cette image.

### Décodage (extraire des données)

1. Passez en mode **DECODE**.
2. Chargez le fichier `*_steg.png` par glisser-déposer.
3. Entrez le **même mot de passe** utilisé lors de l'encodage.
4. Cliquez sur **EXTRACT & DECODE**.
5. Si c'était du texte — il s'affiche dans le panneau de résultat avec un bouton de copie.  
   Si c'était un fichier — il se télécharge automatiquement (avec son nom de fichier et son type MIME d'origine).

---

## 🔬 Format binaire du payload

Comprendre la structure exacte des octets est essentiel pour l'interopérabilité et l'analyse forensique.

### Structure embarquée dans les LSBs

```
Offset   Taille   Champ
──────   ──────   ─────────────────────────────────────────────────────
  0        4      Magic : "SLSB" (0x53 0x4C 0x53 0x42)
  4        1      Version : 0x01
  5        1      Type : 0x00 = texte  |  0x01 = fichier
  6       16      Sel PBKDF2 (aléatoire, 128 bits)
 22       12      IV AES-GCM (aléatoire, 96 bits)
 34        4      Longueur des données chiffrées (uint32, big-endian)
 38        N      Ciphertext AES-GCM (inclut le tag d'auth de 16 octets)
──────   ──────
EN-TÊTE FIXE TOTAL : 38 octets
```

### Après déchiffrement AES-GCM

```
Offset   Taille   Champ
──────   ──────   ─────────────────────────────────────────────────────
  0        4      Longueur du JSON de métadonnées (uint32, big-endian)
  4        M      JSON de métadonnées (encodé UTF-8)
 4+M       R      Payload brut : texte UTF-8  OU  octets binaires du fichier
```

**Exemples de JSON de métadonnées :**
```json
{ "type": "text" }
{ "type": "file", "filename": "secret.pdf", "mime": "application/pdf" }
```

### Indexation des bits LSB

Pour l'index de bit `i` (base 0, MSB en premier par octet) :
```
pixel   = Math.floor(i / 3)      →  quel pixel
channel = i % 3                  →  0=R, 1=G, 2=B
bytePos = Math.floor(i / 8)      →  quel octet du payload
bitPos  = 7 - (i % 8)            →  quel bit (MSB en premier)
data[pixel * 4 + channel] = (data[pixel * 4 + channel] & 0xFE) | bit
```

**Formule de capacité :** `⌊ largeur × hauteur × 3 / 8 ⌋` octets

---

## 🔐 Notes de sécurité

| Aspect | Détail |
|---|---|
| Chiffrement | AES-256-GCM (chiffrement authentifié — détecte toute altération) |
| KDF | PBKDF2-SHA256, 120 000 itérations, sel aléatoire de 128 bits |
| IV | 96 bits aléatoires, unique à chaque opération d'encodage |
| Tag d'authentification | 128 bits (GCM par défaut) — mauvais mot de passe = rejet immédiat |
| Impact LSB | ±1 par canal au maximum — imperceptible à l'œil nu |
| Format image | **Toujours exporter en PNG**. Le JPEG est avec perte et détruira toutes les données cachées |
| Stéganalyse | Les attaques standard (Chi-carré, RS) peuvent détecter une modification LSB. Cet outil assure la **protection cryptographique**, pas la résistance à la stéganalyse. |

> **Important :** Cet outil offre une protection *cryptographique* solide du payload. Un adversaire déterminé disposant de l'image stégo peut statistiquement détecter que *quelque chose* est caché (via stéganalyse), même s'il ne peut pas le lire. Pour une dissimulation maximale, utilisez de grandes images porteuses et maintenez le payload petit par rapport à la capacité totale.

---

## 🏗️ Architecture technique

```
index.html (fichier unique)
│
├── Vue 3 (CDN)           — interface réactive, gestion d'état
├── Tailwind CSS (CDN)    — styles, thème sombre
│
├── Thread principal
│   ├── Web Crypto API    — PBKDF2, AES-GCM (crypto.subtle)
│   ├── Canvas API        — chargement image, accès pixels, export PNG
│   └── Application Vue   — logique UI, interactions utilisateur
│
└── Web Worker (Blob inline)
    └── Moteur LSB        — manipulation bit à bit des pixels (hors thread principal)
```

Flux de données — encodage :
```
Entrée utilisateur
  → TextEncoder / FileReader (ArrayBuffer)
  → buildEmbeddablePayload() → sel PBKDF2 + clé → chiffrement AES-GCM
  → Worker : encodeLSB(pixels, blobChiffré)
  → Canvas.putImageData()
  → canvas.toBlob('image/png') → téléchargement
```

Flux de données — décodage :
```
PNG stégo chargé dans Canvas
  → Worker passe 1 : extraire 38 octets d'en-tête → valider magic + lire encLen
  → Worker passe 2 : extraire en-tête + ciphertext
  → parseEmbeddablePayload() → PBKDF2 → déchiffrement AES-GCM
  → parse JSON méta → afficher texte  OU  déclencher téléchargement fichier
```

---

## 📁 Structure du projet

```
stealth-lsb-vault/
├── index.html      ← L'application entière (HTML + CSS + JS)
├── README.md       ← Ce fichier
└── .gitignore
```

---

## 🛠️ Développement

Aucune chaîne de build requise. Éditez `index.html` directement.

Pour un serveur de développement local (recommandé plutôt que `file://` pour éviter les restrictions sur les Workers) :

```bash
# Python
python3 -m http.server 8080

# Node (npx)
npx serve .

# PHP
php -S localhost:8080
```

---

## ⚠️ Limitations connues

- **Pas de résistance à la stéganalyse** — les outils statistiques standards (StegExpose, SteghideDetect) peuvent détecter une modification LSB. Les payloads sont fortement chiffrés, mais leur *présence* peut être détectable.
- **PNG uniquement pour le décodage** — si vous re-sauvegardez l'image stégo en JPEG (même une seule fois), les données LSB sont définitivement perdues.
- **Payload max ~10 Mo** — limité par le filtre de l'input fichier et la mémoire du navigateur.
- **Canal alpha non utilisé** — le LSB de l'alpha est intentionnellement ignoré pour éviter les artefacts de transparence.
- **Contexte sécurisé requis** — `crypto.subtle` nécessite un contexte sécurisé (`https://` ou `localhost`). L'ouverture via `file://` peut fonctionner dans certains navigateurs mais n'est pas garantie.

---

## 🤝 Contribuer

Les contributions sont les bienvenues. Quelques idées d'amélioration :

- [ ] Encodage adaptatif multi-bits (2 ou 3 LSBs par canal pour une capacité plus élevée)
- [ ] Encodage résistant à la compression JPEG (quantization-aware)
- [ ] Coller une image depuis le presse-papiers (`Ctrl+V`)
- [ ] Mode d'analyse stégano (test du Chi-carré visuel)
- [ ] Découpage multi-images pour les grands payloads
- [ ] Support PWA (service worker hors-ligne)

---

## 📜 Licence

MIT © 2024 — Libre d'utilisation, de modification et de distribution.

---

> *« Le meilleur message caché est celui dont on ne soupçonne pas l'existence. »*
