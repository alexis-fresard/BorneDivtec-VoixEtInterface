<div align="center">

# 🧭 Borne Divtec — Panneau d'orientation interactif

**Borne d'orientation physique pour l'entrée du bâtiment B de la Division technique (CEJEF – DIVTEC).**
Le visiteur demande une salle **à la voix** ou **sur l'écran tactile**, la borne lui répond oralement
et **illumine le chemin** sur le panneau mural grâce à des matrices LED placées derrière celui-ci.

![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)
![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-4%20(8%20Go)-C51A4A?logo=raspberrypi&logoColor=white)
![Kivy](https://img.shields.io/badge/UI-Kivy-1E88E5)
![Offline](https://img.shields.io/badge/100%25-hors--ligne-2E7D32)
![Langue](https://img.shields.io/badge/langue-français-blue)

</div>

---

## Sommaire

1. [Présentation](#-présentation)
2. [Fonctionnalités](#-fonctionnalités)
3. [Architecture](#-architecture)
4. [Matériel](#-matériel)
5. [Câblage des panneaux LED](#-câblage-des-panneaux-led)
6. [Installation](#-installation)
7. [Configuration](#-configuration)
8. [Lancement](#-lancement)
9. [Déploiement en production (systemd)](#-déploiement-en-production-systemd)
10. [Dépannage](#-dépannage)
11. [Feuille de route](#-feuille-de-route)
12. [Contact](#-contact)

---

## 📌 Présentation

À l'entrée du bâtiment B se trouve un panneau physique représentant les niveaux 0 à 3 (salles colorées par catégorie : départements en rouge, salles de cours en bleu, administration en magenta…). Ce projet le rend **interactif** :

```
  « Où est la salle C2-07 ? »         ┌──────────────────────────────┐
  ─────────── 🎤 ──────────────────▶ │  Raspberry Pi 4              │
                                      │  • Vosk (reconnaissance)     │ ──▶ 🔊 « Votre destination se trouve
  👆 Toucher une carte / rechercher   │  • Matching flou (aliases)  │        au 2ème étage… »
  ─────────── 📱 ──────────────────▶ │  • Piper (synthèse vocale)   │
                                      │  • rgbmatrix (LED)           │ ──▶ 💡 Chemin lumineux animé
                                      └──────────────────────────────┘        derrière le panneau
```

- **Deux bornes** sont prévues, une à chacune des deux entrées du bâtiment.
- Tout fonctionne **sans accès à Internet** (reconnaissance vocale, synthèse vocale, interface).
- La borne est pensée pour tourner **en continu et sans surveillance** : si un composant manque (micro, modèle vocal, icône, LED), l'application continue de fonctionner en mode dégradé au lieu de planter.

---

## ✨ Fonctionnalités

| Domaine | Détail |
|---|---|
| 🎤 **Reconnaissance vocale** | Français, hors-ligne, via [Vosk](https://alphacephei.com/vosk/) + `sounddevice`. Vocabulaire restreint (grammaire construite à partir des alias du CSV) pour être robuste au bruit du hall. |
| 📱 **Interface tactile** (Kivy) | Grille de cartes colorées, barre de recherche avec clavier virtuel, filtres par étage, gros bouton micro animé, horloge, écran de veille avec logo. (Python) |
| 🔍 **Matching flou partagé** | Voix et recherche texte utilisent exactement la même logique (`core.trouver_destination`) : normalisation des accents, mots porteurs ignorés (« où est la salle… »), comptage par multiensemble (`Counter`), détection des réponses ambiguës. |
| 🔊 **Réponse vocale** | Voix neuronale [Piper](https://github.com/rhasspy/piper) (`fr_FR-siwis-medium`), repli automatique sur `pyttsx3` (espeak-ng / SAPI5). |
| 💡 **Chemin LED** | Animation point par point sur matrices HUB75 via [`rpi-rgb-led-matrix`](https://github.com/hzeller/rpi-rgb-led-matrix). Trajet en orange, arrivée en vert. |
| 🧩 **Architecture découplée** | Système d'abonnés : la voix, l'écran et les LED réagissent tous au même événement « destination reconnue ». |
| 🪟 **Développement sur PC** | Sans Raspberry Pi, les LED passent en mode simulation (messages dans la console) : la voix, le matching et l'interface se testent sous Windows/Linux. |
| ⏰ **Fonctionnement autonome** | Démarrage automatique via systemd et plages horaires (marche/arrêt programmés). |

---

## 🏗 Architecture

### Arborescence

```
Borne/
├── main.py                         # Point d'entrée : écran tactile + écoute micro + LED
├── core.py                         # Cœur commun : CSV, matching, synthèse vocale, abonnés
├── interface_tactile.py            # Interface Kivy (écran tactile)
├── voix.py                         # Reconnaissance vocale (Vosk + sounddevice)
├── led.py                          # Affichage du chemin sur les matrices LED
├── Destinations.csv                # Liste des destinations
├── CEJEFDivisiontechniquenew.png   # Logo affiché sur l'écran de veille
├── icones/                         # Icônes PNG de l'interface
│   ├── classroom.png  office.png  restaurant.png  sport.png
│   ├── heal.png       wc.png      cleaning.png
│   ├── micro.png      loupe.png
│   └── cropped-divtec_favicon-32x32.png
├── voix_piper/                     # Modèle Piper (à télécharger, voir Installation)
│   ├── fr_FR-siwis-medium.onnx
│   └── fr_FR-siwis-medium.onnx.json
└── vosk-model-small-fr-0.22/       # Modèle Vosk (à télécharger, voir Installation)
```

> Les dossiers de modèles (`vosk-model-small-fr-0.22/`, `voix_piper/`) sont volumineux : il est conseillé de les ajouter au `.gitignore` et de les télécharger lors de l'installation.

### Rôle des modules

```
                ┌───────────────┐        ┌────────────────────┐
  micro ──────▶ │   voix.py     │        │ interface_tactile  │ ◀────── écran tactile
                └──────┬────────┘        └─────────┬──────────┘
                       │  core.on_destination_reconnue(...)  │
                       └──────────────┬─────────────────────┘
                                      ▼
                            ┌───────────────────┐
                            │      core.py      │  → parler(...) (Piper / pyttsx3)
                            └─────────┬─────────┘
                       notifie tous les abonnés (ajouter_abonne)
                  ┌───────────────────┴───────────────────┐
                  ▼                                       ▼
       interface_tactile                               led.py
  (surligne la carte, stoppe l'écoute)      (dessine le chemin lumineux)
```

- **`core.py`** — Aucune dépendance à l'écran ni au micro. Charge le CSV, fait le matching, parle, et diffuse l'événement aux abonnés. Tout nouveau module (ex. pathfinding) se branche avec `core.ajouter_abonne(fonction)` où `fonction(destination_id, dest, source)` et `source` vaut `"voix"` ou `"tactile"`.
- **`voix.py`** — Boucle d'écoute bloquante ; elle est toujours lancée dans un thread séparé pour ne pas figer l'interface. Possède un **mode test clavier** (`python3 voix.py test`).
- **`interface_tactile.py`** — Kivy doit tourner dans le thread principal. Les mises à jour d'écran déclenchées depuis d'autres threads passent par `Clock.schedule_once`.
- **`led.py`** — Initialise la matrice **une seule fois**, dessine le chemin dans un thread dédié protégé par un verrou. Le tracé actuel est une ligne droite (Bresenham) ; il suffira de remplacer `calculer_chemin()` pour brancher le vrai pathfinding.

---

## 🔧 Matériel

| Composant | Remarques |
|---|---|
| **Raspberry Pi 4 (8 Go)** + alimentation officielle + boîtier | Le Pi 4 est **préféré au Pi 5** : `rpi-rgb-led-matrix` a des incompatibilités GPIO avec la puce RP1 du Pi 5. |
| **Carte de pilotage HUB75** | Carte branchée sur le GPIO (mapping `regular` dans le code). Une Adafruit RGB Matrix Bonnet/HAT est aussi possible (mapping `adafruit-hat`). |
| **Rallonge / extension GPIO** | Recommandée : sans elle, la carte peut se poser de travers et la transmission ne passe pas. |
| **Matrices LED RVB HUB75** | Configuration actuelle : 2 panneaux 32×32 chaînés (canvas 64×32). |
| **Alimentation 5 V dédiée aux panneaux** | Prévoir **~3,5 A par panneau 32×32** (cf. [documentation hzeller](https://github.com/hzeller/rpi-rgb-led-matrix/blob/master/wiring.md)). |
| **Microphone USB** | Un micro à réseau (ex. ReSpeaker USB Mic Array) est recommandé pour un hall bruyant. |
| **Haut-parleur** | Pour la réponse vocale. |
| **Écran tactile** | Interface Kivy. |
| **Prise secteur** | La borne est branchée en continu. |

---

## 🔌 Câblage des panneaux LED

> ⚠️ **Attention aux polarités !** Une inversion du **+** et du **–** sur l'alimentation peut détruire les panneaux et le Raspberry Pi. Vérifiez deux fois avant de mettre sous tension.

### 1. Carte de pilotage → Raspberry Pi
La carte se branche sur le connecteur GPIO 40 broches. **Le sens des broches dépend du modèle** (sur un Raspberry Pi 400, le connecteur est à l'arrière et l'ordre des broches est inversé par rapport à une carte Pi classique) : repérez la **broche 1** sur le Pi et sur la carte avant d'enficher. Utilisez une rallonge GPIO si la carte ne tient pas droite.

### 2. Carte de pilotage → premier panneau
Branchez la nappe de la sortie **TOP-P0** (chaîne 1) de la carte sur l'**entrée** du premier panneau.

### 3. Panneau → panneau (chaînage)
Chaque panneau possède un connecteur **d'entrée** et un **de sortie**. La flèche imprimée au dos indique le sens : la **queue** de la flèche est côté entrée, la **pointe** côté sortie. Reliez la **sortie** d'un panneau à l'**entrée** du suivant.

### 4. Alimentation des panneaux
- Reliez les câbles d'alimentation des panneaux (connecteurs **+5 V / GND**) à l'alimentation 5 V.
- Si vous utilisez une **alimentation de type ATX**, elle ne démarre que si la broche *PS_ON* (fil généralement **vert**, situé au milieu de fils noirs) est reliée à une masse (fil **noir**) — par exemple avec un cavalier.
- Reliez la masse de l'alimentation des panneaux à celle du Raspberry Pi (via la carte de pilotage) pour une référence commune.

---

## 📦 Installation

Les commandes ci-dessous supposent **Raspberry Pi OS (avec bureau)**, l'utilisateur `admin`, et le projet dans `/home/admin/Borne`. Adaptez les chemins si besoin.

### 1. Paquets système

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git python3-dev python3-venv python3-pip python3-pillow \
                    libportaudio2 espeak-ng \
                    libgraphicsmagick++-dev libwebp-dev
```

| Paquet | Pourquoi |
|---|---|
| `libportaudio2` | Requis par `sounddevice` (micro et lecture audio) |
| `espeak-ng` | Voix de repli pour `pyttsx3` si Piper est indisponible |
| `python3-dev`, `libgraphicsmagick++-dev`, `libwebp-dev` | Compilation de `rpi-rgb-led-matrix` et de ses utilitaires |

### 2. Récupérer le projet

```bash
cd /home/admin
git clone https://github.com/alexis-fresard/V2---Scripts-Python---interface--voix-et--coute.git Borne
cd Borne
```

### 3. Bibliothèque des panneaux LED (`rpi-rgb-led-matrix`)

```bash
cd /home/admin
git clone https://github.com/hzeller/rpi-rgb-led-matrix.git
cd rpi-rgb-led-matrix

# Bibliothèque C + exemples
make
cd examples-api-use && make && cd ..
cd utils && make led-image-viewer && cd ..

# Module Python « rgbmatrix » pour Python 3
cd bindings/python
make build-python PYTHON=$(which python3)
sudo make install-python PYTHON=$(which python3)
```

> La procédure de compilation des bindings Python évolue avec le dépôt de hzeller : en cas d'erreur, référez-vous au [README de `bindings/python`](https://github.com/hzeller/rpi-rgb-led-matrix/tree/master/bindings/python).

Vérifiez ensuite que le module est importable :

```bash
python3 -c "from rgbmatrix import RGBMatrix; print('rgbmatrix OK')"
```

### 4. Environnement virtuel Python

Le module `rgbmatrix` étant installé au niveau du système, l'environnement virtuel doit y avoir accès (`--system-site-packages`) :

```bash
python3 -m venv --system-site-packages /home/admin/venv
source /home/admin/venv/bin/activate
pip install --upgrade pip
pip install kivy vosk sounddevice numpy piper-tts pyttsx3
```

### 5. Modèle de reconnaissance vocale (Vosk)

```bash
cd /home/admin/Borne
wget https://alphacephei.com/vosk/models/vosk-model-small-fr-0.22.zip
unzip vosk-model-small-fr-0.22.zip && rm vosk-model-small-fr-0.22.zip
```

Le dossier `vosk-model-small-fr-0.22/` doit se trouver **à côté des scripts**.

### 6. Voix de synthèse (Piper)

```bash
cd /home/admin/Borne
source /home/admin/venv/bin/activate
python -m piper.download_voices fr_FR-siwis-medium --download-dir voix_piper
```

Les fichiers `voix_piper/fr_FR-siwis-medium.onnx` et `.onnx.json` doivent être présents. Sans eux, la borne bascule automatiquement sur `pyttsx3`.

### 7. Audio du Raspberry Pi

`rpi-rgb-led-matrix` entre en conflit avec l'audio intégré du Pi lorsqu'il utilise le « hardware pulsing ». Le projet **désactive volontairement le hardware pulsing** (`disable_hardware_pulsing = True` / `--led-no-hardware-pulse`) afin de garder le son pour la réponse vocale.

Vérifiez que le micro USB est bien détecté :

```bash
arecord -l                                    # liste les périphériques d'enregistrement
python3 -c "import sounddevice as sd; print(sd.query_devices())"
```

### 8. Tester les panneaux LED

```bash
# Démo en C
cd ~/rpi-rgb-led-matrix/examples-api-use
sudo ./demo -D 0 --led-chain=2 --led-no-hardware-pulse \
            --led-gpio-mapping=regular --led-slowdown-gpio=4

# Démo en Python
cd ~/rpi-rgb-led-matrix/bindings/python/samples
sudo python3 runtext.py --led-chain=2 --led-no-hardware-pulse --led-slowdown-gpio=4
```

> `--led-slowdown-gpio=4` est nécessaire sur un Pi 4, trop rapide pour la plupart des panneaux. Si l'image scintille ou est corrompue, essayez des valeurs entre 2 et 5.

---

## ⚙️ Configuration

### Liste des destinations — `Destinations.csv`

Fichier **UTF-8**, séparateur **`;`**, en-tête obligatoire.

| Colonne | Obligatoire | Description |
|---|:---:|---|
| `id` | ✅ | Identifiant affiché et prononcé (ex. `C2-07`) |
| `aliases` | ✅ | Formulations reconnues, séparées par `\|` (écrire les nombres **en toutes lettres** pour la voix) |
| `etage` | ➖ | `0` = rez-de-chaussée, `1`, `2`, `3`… **Convention : `4` = Bâtiment A** |
| `couleur` | ➖ | Couleur de la carte à l'écran (`#RRGGBB`), sinon bleu par défaut |
| `ligne` | ➖ | Ligne du pixel d'arrivée sur la matrice LED (0 = en haut) |
| `colonne` | ➖ | Colonne du pixel d'arrivée sur la matrice LED (0 = à gauche) |

Exemple :

```csv
id;aliases;etage;couleur;ligne;colonne
C2-07;salle c2 07|c deux zero sept|c deux sept;2;#1F4FE0;8;40
SECRETARIAT;secretariat|le secretariat|accueil;0;#D63CC8;28;50
CHIMIE;chimie|labo de chimie|laboratoire de chimie;0;#D62828;30;36
SALLE DE GYMNASTIQUE;gym|salle de gym|salle de sport|gymnastique;4;#1F4FE0;31;20
```

**Conseils pour les alias**
- Penser à la façon dont un visiteur **parle** : « c deux zéro sept », « c deux sept », « la salle de chimie »…
- Les accents et majuscules sont ignorés.
- Les mots comme « où », « est », « salle », « je cherche » sont ignorés lors du matching : inutile de les ajouter.
- Si deux destinations se ressemblent trop (ex. plusieurs « EST … »), donner à chacune un mot distinctif, sinon la borne répondra qu'elle n'a pas compris (réponse ambiguë).
- L'icône de la carte est déduite automatiquement de l'id/des alias (`wc`, `cafeteria`, `sport`, `infirmerie`, `secretariat`, `concierge`…).

### Paramètres principaux

| Fichier | Constante | Rôle |
|---|---|---|
| `core.py` | `MODEL_PATH`, `DESTINATIONS_CSV_PATH` | Chemins du modèle Vosk et du CSV |
| `core.py` | `VOIX_ACTIVEE` | Active/désactive la réponse vocale |
| `core.py` | `PIPER_LENTEUR`, `PIPER_VOLUME` | Débit (> 1 = plus lent) et volume de la voix Piper |
| `core.py` | `VOIX_VITESSE` | Débit de la voix de repli `pyttsx3` |
| `interface_tactile.py` | `DUREE_MAX_ECOUTE` | Durée max d'écoute après appui sur le micro (12 s) |
| `interface_tactile.py` | `DUREE_AVANT_VEILLE` | Inactivité avant l'écran de veille (60 s) |
| `led.py` | `DEPART` | Pixel de départ du chemin (position de la borne sur le plan) |
| `led.py` | `COULEUR_CHEMIN`, `COULEUR_DESTINATION`, `DELAI_ENTRE_POINTS` | Apparence et vitesse de l'animation |
| `led.py` | `_creer_options()` | Taille, chaînage, mapping, luminosité, `gpio_slowdown` des panneaux |

---

## ▶️ Lancement

Toujours lancer depuis le dossier du projet (les chemins du CSV et des modèles sont relatifs) :

```bash
cd /home/admin/Borne
source /home/admin/venv/bin/activate
```

| Commande | Usage |
|---|---|
| `python3 main.py` | **Mode complet** : écran tactile + écoute micro + LED |
| `python3 interface_tactile.py` | Écran tactile seul (le bouton micro reste utilisable) |
| `python3 voix.py` | Reconnaissance vocale seule, avec le micro |
| `python3 voix.py test` | **Mode test clavier** : tapez une phrase pour tester le CSV, le matching et la voix sans micro |

**Raccourci utile :** la touche **F2** affiche immédiatement l'écran de veille.

> 💡 **Sur un PC (Windows/Linux sans panneaux)** : `rgbmatrix` est absent, `led.py` passe automatiquement en mode simulation et affiche dans la console le chemin qui aurait été dessiné.

> Selon les droits d'accès au GPIO de votre système, l'affichage LED peut nécessiter d'être lancé avec `sudo` (voir [Dépannage](#-dépannage)).

---

## 🚀 Déploiement en production (systemd)

### Service principal — `/etc/systemd/system/divtec.service`

```ini
[Unit]
Description=Borne d'orientation Divtec
After=graphical.target sound.target
Wants=graphical.target
StartLimitIntervalSec=120
StartLimitBurst=5

[Service]
Type=simple
User=admin
WorkingDirectory=/home/admin/Borne
Environment=DISPLAY=:0
Environment=XAUTHORITY=/home/admin/.Xauthority
Environment=PYTHONUNBUFFERED=1
ExecStartPre=/home/admin/Borne/wait-for-x.sh
ExecStart=/home/admin/venv/bin/python /home/admin/Borne/main.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=graphical.target
```

Points importants :
- `ExecStart` pointe **directement** sur le Python du venv.
- `WantedBy=graphical.target` (et non `multi-user.target`), car l'interface a besoin de l'affichage.
- `WorkingDirectory` est indispensable pour que `Destinations.csv` et les modèles soient trouvés.

### Attente du serveur graphique — `wait-for-x.sh`

Au démarrage, systemd peut atteindre `graphical.target` avant que le serveur X soit prêt. Ce script attend le socket X (max. 60 s) :

```bash
#!/bin/bash
for i in $(seq 1 60); do
    [ -S /tmp/.X11-unix/X0 ] && exit 0
    sleep 1
done
echo "Serveur X introuvable après 60 s" >&2
exit 1
```

```bash
chmod +x /home/admin/Borne/wait-for-x.sh
sudo systemctl daemon-reload
sudo systemctl enable --now divtec.service
```

### Marche / arrêt programmés

Pour ne pas faire tourner la borne la nuit. **Adaptez les horaires à ceux du bâtiment.**

`/etc/systemd/system/divtec-start.timer`
```ini
[Unit]
Description=Démarrage quotidien de la borne Divtec

[Timer]
OnCalendar=Mon..Fri 07:00
Persistent=true
Unit=divtec.service

[Install]
WantedBy=timers.target
```

`/etc/systemd/system/divtec-stop.service`
```ini
[Unit]
Description=Arrêt de la borne Divtec

[Service]
Type=oneshot
ExecStart=/bin/systemctl stop divtec.service
```

`/etc/systemd/system/divtec-stop.timer`
```ini
[Unit]
Description=Arrêt quotidien de la borne Divtec

[Timer]
OnCalendar=Mon..Fri 20:00
Persistent=true
Unit=divtec-stop.service

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now divtec-start.timer divtec-stop.timer
systemctl list-timers | grep divtec
```

### Commandes utiles

```bash
systemctl status divtec.service          # état du service
journalctl -u divtec.service -f          # logs en direct
journalctl -u divtec.service -b          # logs depuis le dernier démarrage
sudo systemctl restart divtec.service    # redémarrer après une modification
```

---

## 🩺 Dépannage

| Symptôme | Cause probable / solution |
|---|---|
| Le service est `Active: dead` au démarrage mais fonctionne lancé à la main | Le serveur X n'était pas prêt. Vérifier `wait-for-x.sh`, `DISPLAY`, `XAUTHORITY` et `WantedBy=graphical.target`. |
| `Fichier de destinations introuvable` / modèle Vosk introuvable | L'application n'est pas lancée depuis le dossier du projet : vérifier `WorkingDirectory` ou faire `cd /home/admin/Borne`. |
| `Error querying device -1` | Micro USB non détecté. Vérifier `arecord -l`, rebrancher le micro. L'écran tactile reste utilisable en attendant. |
| Le bouton micro devient rouge foncé | Micro ou modèle Vosk indisponible (voir le message dans les logs). |
| La borne ne parle pas | Vérifier la sortie audio (`aplay -l`, `alsamixer`) et la présence du modèle Piper ; les logs indiquent si le repli `pyttsx3` est utilisé. |
| Destination souvent « non comprise » | Enrichir les `aliases` du CSV, tester avec `python3 voix.py test`. Le micro doit être proche de l'utilisateur. |
| `[led] Bibliothèque 'rgbmatrix' indisponible` | Module non installé, ou venv créé sans `--system-site-packages`. |
| Panneaux noirs, rien ne s'affiche | Alimentation des panneaux, sens de la nappe (entrée/sortie), carte bien enfichée (rallonge GPIO), droits d'accès au GPIO (essayer avec `sudo`). |
| Image scintillante ou corrompue | Ajuster `gpio_slowdown` (2 à 5) dans `led.py`. |
| Chemin LED absent pour une salle | Colonnes `ligne`/`colonne` vides ou invalides pour cette destination dans le CSV. |

---

## 🗺 Feuille de route

- [x] Reconnaissance vocale hors-ligne (Vosk)
- [x] Interface tactile (recherche, filtres, veille, clavier virtuel)
- [x] Voix neuronale Piper avec repli `pyttsx3`
- [x] Logo / icône de conciergerie
- [x] Premier affichage LED (ligne droite départ → salle)
- [ ] Compléter `Destinations.csv` (coordonnées LED de toutes les salles)
- [ ] **Pathfinding** sur un graphe des couloirs (BFS / Dijkstra / A\*) depuis chacune des deux entrées, avec cache au démarrage
- [ ] Fiabiliser le démarrage automatique au boot
- [ ] Installation définitive du micro USB et tests vocaux en conditions réelles
- [ ] Durcissement du Raspberry Pi (performances, sécurité)
- [ ] Extinction de l'écran (`xset dpms`) synchronisée avec les horaires d'ouverture

---

## 📬 Contact

Projet réalisé pour la **Division technique du CEJEF (DIVTEC)**.
Pour toute information supplémentaire : **Alexis Frésard**.
