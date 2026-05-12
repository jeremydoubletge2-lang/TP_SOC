## Introduction

L’objectif de ce TP est de concevoir un composant matériel spécifique : un registre 16 bits nommé `reg16_avalon_interface`, accessible en lecture et en écriture via le bus **Avalon Memory-Mapped** en mode maître/esclave. La valeur stockée dans ce registre est ensuite exportée vers l’extérieur grâce à une interface **Conduit**, permettant par exemple de piloter des LEDs ou des afficheurs 7 segments.

Le travail se déroule en plusieurs étapes principales :

1. Développement du composant en HDL (VHDL ou Verilog), incluant le registre et son interface Avalon MM.
2. Utilisation de **Qsys / Platform Designer** et du **Component Editor** afin de :
   - définir les interfaces (`clock`, `reset`, `Avalon-MM slave`, `conduit`) ;
   - configurer correctement le composant ;
   - corriger les éventuelles erreurs de validation.
3. Intégration du composant dans un système embarqué comprenant :
   - un processeur **Nios II** ;
   - une mémoire **On-Chip Memory**.
4. Génération automatique du code HDL du système complet.
5. Importation du système dans un projet **Quartus II**, compilation et programmation de la carte FPGA.

Ce TP permet ainsi de découvrir la création de périphériques personnalisés pour un système Nios II et leur intégration dans une architecture FPGA basée sur le bus Avalon.

Voici une photo de l'architecture global du système : 

<img width="704" height="512" alt="image" src="https://github.com/user-attachments/assets/fc2a982c-8bbd-431a-ba4e-fcd47b4fc983" />

# Projet VHDL — Tutoriel Composants & Interface Avalon

Ce projet regroupe plusieurs fichiers VHDL formant un système embarqué simple. Il illustre l'utilisation de composants réutilisables, d'un affichage 7 segments et d'une interface Avalon pour communiquer avec un processeur (ex. Nios II sur FPGA Altera/Intel).

---

## Structure du projet

```
.
├── component_tutorial.vhd       # Entité top-level : assemble tous les composants
├── hex7seg.vhd                  # Décodeur hexadécimal vers afficheur 7 segments
├── reg16.vhd                    # Registre 16 bits avec activation par octet
└── reg16_avalon_interface.vhd   # Interface Avalon MM autour du registre 16 bits
```

---

## Fichiers

### `hex7seg.vhd` — Décodeur 7 segments

Ce composant convertit une valeur hexadécimale 4 bits en signal de commande pour un afficheur 7 segments.

**Ports :**

| Nom       | Direction | Taille  | Description                          |
|-----------|-----------|---------|--------------------------------------|
| `hex`     | IN        | 4 bits  | Valeur hexadécimale à afficher (0–F) |
| `display` | OUT       | 7 bits  | Segments à allumer sur l'afficheur   |

**Fonctionnement :**

Le processus combinatoire écoute `hex` et, via une instruction `CASE`, sélectionne le vecteur de 7 bits correspondant à chacun des 16 caractères hexadécimaux (`0` à `F`). Chaque bit du vecteur de sortie contrôle un segment de l'afficheur (0 = allumé, 1 = éteint, selon la convention active-bas).

---

### `reg16.vhd` — Registre 16 bits avec byte enable

Registre synchrone 16 bits avec reset actif bas et contrôle d'écriture par octet (*byte enable*).

**Ports :**

| Nom          | Direction | Taille  | Description                                  |
|--------------|-----------|---------|----------------------------------------------|
| `clock`      | IN        | 1 bit   | Horloge système                              |
| `resetn`     | IN        | 1 bit   | Reset synchrone, actif bas                   |
| `D`          | IN        | 16 bits | Donnée à écrire                              |
| `byteenable` | IN        | 2 bits  | Activation par octet (`byteenable(0)` = LSB, `byteenable(1)` = MSB) |
| `Q`          | OUT       | 16 bits | Valeur stockée dans le registre              |

**Fonctionnement :**

À chaque front montant de l'horloge :
- Si `resetn = '0'` → le registre est remis à zéro.
- Sinon, chaque octet de `D` est écrit dans `Q` uniquement si le bit `byteenable` correspondant est à `'1'`. Cela permet d'écrire sélectivement l'octet bas, l'octet haut, ou les deux.

---

### `reg16_avalon_interface.vhd` — Interface Avalon Memory-Mapped

Ce composant encapsule `reg16` derrière une interface **Avalon Memory-Mapped (MM)**, compatible avec le bus Avalon utilisé par les systèmes Nios II sur FPGA Intel.

**Ports :**

| Nom          | Direction | Taille  | Description                                      |
|--------------|-----------|---------|--------------------------------------------------|
| `clock`      | IN        | 1 bit   | Horloge système                                  |
| `resetn`     | IN        | 1 bit   | Reset actif bas                                  |
| `read`       | IN        | 1 bit   | Signal de lecture Avalon                         |
| `write`      | IN        | 1 bit   | Signal d'écriture Avalon                         |
| `chipselect` | IN        | 1 bit   | Sélection du composant                           |
| `writedata`  | IN        | 16 bits | Donnée à écrire (depuis le maître Avalon)        |
| `byteenable` | IN        | 2 bits  | Activation par octet                             |
| `readdata`   | OUT       | 16 bits | Donnée lue (vers le maître Avalon)               |
| `Q_export`   | OUT       | 16 bits | Export du contenu du registre (vers le top-level)|

**Fonctionnement :**

- `local_byteenable` est forcé à `"00"` (aucune écriture) si `chipselect` ou `write` est inactif. L'écriture n'a lieu que si les deux signaux sont à `'1'`.
- Une instance de `reg16` est créée en interne ; elle reçoit `writedata` et le `byteenable` conditionnel.
- La sortie `from_reg` est exposée à la fois sur `readdata` (lecture Avalon) et `Q_export` (usage externe).

---

### `component_tutorial.vhd` — Top-level du système

Entité principale qui assemble le système complet sur la carte FPGA. Elle instancie le système embarqué (`embedded_system`, supposément généré par Platform Designer) ainsi que quatre décodeurs `hex7seg` pour afficher la valeur 16 bits sur les afficheurs HEX0 à HEX3.

**Ports :**

| Nom        | Direction | Taille  | Description                              |
|------------|-----------|---------|------------------------------------------|
| `CLOCK_50` | IN        | 1 bit   | Horloge 50 MHz de la carte               |
| `KEY`      | IN        | 1 bit   | Bouton poussoir (reset actif bas)        |
| `HEX0–3`   | OUT       | 7 bits  | Afficheurs 7 segments (chiffres 0 à 3)  |

**Architecture :**

```
CLOCK_50 ──┐
           ├──► embedded_system ──► to_HEX[15:0]
KEY(0) ────┘                           │
                         ┌─────────────┼─────────────┐─────────────┐
                    [3:0]▼        [7:4]▼        [11:8]▼      [15:12]▼
                     hex7seg       hex7seg       hex7seg       hex7seg
                        │             │             │             │
                      HEX0          HEX1          HEX2          HEX3
```

Les 16 bits exportés par le système embarqué sont découpés en quatre groupes de 4 bits, chacun envoyé à une instance de `hex7seg` pour affichage.

---

## Dépendances entre composants

```
component_tutorial
├── embedded_system     (généré par Platform Designer / Qsys)
│   └── reg16_avalon_interface
│       └── reg16
└── hex7seg             (×4)
```

---

## Utilisation

Ce projet est conçu pour un FPGA **Intel (Altera)**, typiquement une carte **DE1-SoC** ou **DE2**. La compilation se fait avec **Intel Quartus Prime**. Le composant `embedded_system` est généré via **Platform Designer (Qsys)** et doit être recréé ou importé séparément.

> Les fichiers `.bak` fournis sont des sauvegardes automatiques de Quartus et peuvent être ignorés.
