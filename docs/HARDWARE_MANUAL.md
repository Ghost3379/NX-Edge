# NX-Edge Hardware Reference Manual

> **Dokumentversion:** 1.0 (Draft)  
> **Hardware-Revision:** M.2 2260 Key B (6-Lagen High-Density)  
> **Zielgruppe:** Entwickler, Maker & Prop-Builder

---

## Inhaltsverzeichnis

1. [Sicherheits- & Anschlusshinweise (Read First!)](#1-sicherheits---anschlusshinweise-read-first)
2. [Board-Übersicht & Komponenten-Map](#2-board-übersicht--komponenten-map)
3. [Stromversorgung & Power-Path (`VIN`, LiPo, Buck)](#3-stromversorgung--power-path-vin-lipo-buck)
4. [Batterie-Management & Fuel Gauge (`MAX17048`)](#4-batterie-management--fuel-gauge-max17048)
5. [Smart NeoPixel Power-Switch (`VN`)](#5-smart-neopixel-power-switch-vn)
6. [RF-Frontend & Antennen-Umschaltung](#6-rf-frontend--antennen-umschaltung)
7. [Onboard-Sensoren & Echtzeituhr (RTC & IMU)](#7-onboard-sensoren--echtzeituhr-rtc--imu)
8. [Audio-Verstärker (`MAX98357A`)](#8-audio-verstärker-max98357a)
9. [Massenspeicher (QSPI Flash & 4-Bit MicroSD)](#9-massenspeicher-qspi-flash--4-bit-microsd)
10. [Pinout-Tabellen (Castellated Pads & M.2-Fingers)](#10-pinout-tabellen-castellated-pads--m2-fingers)
11. [Software-Cheatsheet (GPIOs, I2C-Adressen, Snippets)](#11-software-cheatsheet-gpios-i2c-adressen-snippets)

---

## 1. Sicherheits- & Anschlusshinweise (Read First!)

> [!CAUTION]
> **KEIN PC-MAINBOARD-STECKER!**  
> Obwohl der NX-Edge mechanisch in einen standardmäßigen **M.2 Key-B-Sockel** passt, verwendet er ein **proprietäres Maker-Pinout** (ähnlich SparkFun MicroMod). 
> Auf den M.2-Fingern liegen direkte Batteriespannung (`VBAT`), ungeglättete $5\text{V}$, Lautsprecherausgänge und $3,3\text{V}$-GPIOs.  
> **Das Board darf NIEMALS in einen Desktop-PC, Laptop oder Standard-NVMe/SATA-Adapter gesteckt werden!**

* **Maximale Eingangsspannung an `VBUS` (USB-C):** $5,5\text{V}$ (Standard USB 5V).
* **Maximale Eingangsspannung an `VBAT` (JST-PH):** $4,2\text{V}$ (1S LiPo / Li-Ion Zelle).
* **Ausgangsspannung der Logik:** Strikte $3,3\text{V}$. Alle ESP32-S3 GPIOs sind **NICHT 5V-tolerant**!

---

## 2. Board-Übersicht & Komponenten-Map

<!-- BILD-PLATZHALTER: BOARD-MAP -->
> 📷 **BILD 1 BENÖTIGT:** *Draufsicht (Top-Render) mit Beschriftungs-Pfeilen auf die Hauptblöcke (USB, ESP32, Flash, IMU, Audio, SD, JST, etc.)*

```text
[USB-C] ─────────────────── [Ignion Ceramic Antenna + U.FL]
  │                                      │
[Power-Path / Buck / Charger]      [ESP32-S3 Core @ 45°]
  │                                      │
[JST-PH Battery Connector]         [16MB QSPI Flash]
  │                                      │
[MAX98357A Audio Amp]              [RV-3028 RTC + Supercap]
  │                                      │
[MicroSD 4-Bit Slot] ───────────── [BNO085 9-DOF IMU]
  └───────────────────┬──────────────────┘
                 [M.2 KEY B FINGERS]
```

---

## 3. Stromversorgung & Power-Path (`VIN`, LiPo, Buck)

Der NX-Edge regelt die Umschaltung zwischen USB und Akku völlig autonom hardwareseitig:

```text
  USB-C (VBUS 5V) ──► [Schottky D2] ─────────┐
                                              │
  LiPo (VBAT) ────► [Hard-Cut JP2] ──► [Q1] ──┴─► [ VIN ] (System Rail ~4.2 - 5.0V)
                                        ▲           │
                                        │ Gate      ├─► [TLV62569 Buck] ──► +3V3 (2.0A)
                                  (VBUS Controlled) │
                                                    └─► [NeoPixel / Audio]
```

### Der Hard-Cut Jumper `JP2` (Echter 0µA Standby)
Zwischen dem Pluspol des JST-Akkusteckers und dem Systemnetz liegt das Lötjumper-Feld **`JP2`**:
* **Standard:** Mit einer Lötbrücke geschlossen. Der Akku versorgt das System.
* **Für externe Schalter:** Die Lötbrücke wird durchtrennt und zwei Litzen zu einem physischen Ein/Aus-Kippschalter am Gehäuse/Hilt gelegt. Ergebnis: **Absolut $0,0\,\mu\text{A}$ Entladung** bei Nichtbenutzung.

<!-- BILD-PLATZHALTER: JP2 DETAIL -->
> 📷 **BILD 2 BENÖTIGT:** *Nahaufnahme von JP2 und dem JST-PH Stecker.*

---

## 4. Batterie-Management & Fuel Gauge (`MAX17048`)

* **Lade-IC (`BQ25170`):** Lädt die 1S LiPo-Zelle autonom mit $600\text{–}800\,\text{mA}$. Eine rote Status-LED zeigt den Ladevorgang an.
* **Fuel Gauge (`MAX17048G`):** 
  * I2C-Adresse: **`0x36`**
  * Misst die Zellenspannung ($V_{CELL}$) direkt am Akkuanschluss (ohne Spannungsabfall durch Leiterbahnen).
  * Berechnet den Ladezustand (**SoC in %**) über Maxim's ModelGauge-Algorithmus (kein Shunt-Widerstand nötig!).
  * Alarm-Pin an **`IO7`** (kann den ESP32-S3 bei kritischem Akkustand wecken).

---

## 5. Smart NeoPixel Power-Switch (`VN`)

Externe WS2812B/SK6812 LED-Streifen ziehen auch im ausgeschalteten Zustand ca. $1\,\text{mA}$ Ruhestrom pro Pixel. Der NX-Edge besitzt dafür eine hardwareseitige Stromabschaltung:

* **Schalttransistor:** P-Kanal MOSFET `Q2` (DMP3130L-7), angesteuert über NPN `Q3`.
* **Steuer-Pin:** **`GPIO12`** (`CTRL_NP`).
  * `digitalWrite(12, HIGH)` -> Power-Rail `VN` ist **EIN** ($5\text{V}$ bei USB / $V_{BAT}$ bei Akku).
  * `digitalWrite(12, LOW)` -> Power-Rail `VN` ist **AUS** ($0,0\text{V}$, echter Nullverbrauch).
* **Onboard-NeoPixel:** Hängt ebenfalls an der geschalteten `VN`-Schiene (Datenleitung auf **`IO4`**).
* **Castellated Pads:** Die 3 Pads an der oberen Kante führen `VN`, `NPD_IN` (Daten) und `GND` direkt nach außen.

> [!WARNING]
> **USB-Strombegrenzung beachten:**  
> Wenn viele NeoPixel (z. B. 30–60 LEDs) auf voller Helligkeit weiß leuchten, können Ströme von $>2\text{A}$ fließen. Bei reinem USB-Betrieb an schwachen Ports muss die Helligkeit in Software begrenzt werden (`FastLED.setMaxPowerInVoltsAndMilliamps(5, 1000)`), um Brownouts zu verhindern. Am Akku fängt die LiPo-Zelle diese Spitzen ab.

---

## 6. RF-Frontend & Antennen-Umschaltung

Der NX-Edge verfügt über zwei Antennenpfade, umschaltbar im Betrieb:

* **HF-Schalter:** Infineon `BGS12WN6`
* **Steuer-Pin:** **`GPIO10`** (`RF_CTRL`)
  * **`LOW` (Standard):** Interne **Ignion NN02-201** Keramik-Chipantenne aktiv.
  * **`HIGH`:** Externe **Hirose U.FL / IPEX** Buchse aktiv (z. B. für Antennenkabel aus Metall- oder Carbongehäusen).

---

## 7. Onboard-Sensoren & Echtzeituhr (RTC & IMU)

### 9-DOF IMU (`BNO085`)
* **I2C-Adresse:** **`0x4A`** (Default)
* **Takt:** Eigener $32,768\,\text{kHz}$ Quarz für maximale Sensor-Fusion-Präzision.
* **Interrupt:** `!INT_DOF` an **`IO5`** (RTC-wake-fähig).

### Ultra-Low-Power RTC (`RV-3028-C7`)
* **I2C-Adresse:** **`0x52`**
* **Stromaufnahme:** Typisch nur $45\,\text{nA}$.
* **Interrupt / Alarm:** `!INT_RTC` an **`IO6`** (weckt ESP32 aus Deep Sleep).
* **Backup-Stromversorgung (Zwei Optionen):**
  1. **Option A (SMD-Supercap Onboard):** Wird direkt auf die Lötflächen gesetzt. Das Lade-Pad `R-CHARGE` wird mit einer $0\,\Omega$-Brücke kurzgeschlossen; der interne Trickle-Charger der RV-3028 lädt den Kondensator automatisch über I2C.
  2. **Option B (Externe Pufferbatterie):** Litzen zu einer externen Knopfzelle (CR1220 etc.) anlöten. Bei Primärzellen bleibt die Ladebrücke unbestückt!

<!-- BILD-PLATZHALTER: RTC BACKUP DETAIL -->
> 📷 **BILD 3 BENÖTIGT:** *Nahaufnahme der RTC, des Supercap-Pads und der R-CHARGE Brücke.*

---

## 8. Audio-Verstärker (`MAX98357A`)

* **Typ:** I2S Mono Class-D Verstärker, bis zu **$3,2\text{W}$ an $4\,\Omega$**.
* **Versorgung:** Direkt aus `VIN` für maximale Dynamik und Lautstärke.
* **I2S-Signalbelegung:**
  * Bit Clock (`BCLK`): **`IO33`**
  * Data In (`DIN`): **`IO34`**
  * Word Select (`LRCLK`): **`IO35`**
* **Lautsprecher-Anschluss:** Zwei Castellated Pads an der Kante (`SPK+` / `SPK-`).
* *(Hinweis: Gain-Widerstands-Option wird in Revision 1.1 ergänzt).*

---

## 9. Massenspeicher (QSPI Flash & 4-Bit MicroSD)

* **16 MB QSPI Flash:** Winbond `W25Q128JVPIQ` (WSON-8). Läuft auf internen Striplines auf Layer 3 mit bis zu $80\text{ MHz}$.
* **MicroSD-Kartenhalter:** Molex 105162 Push-Pull. Läuft im nativen **Hardware 4-Bit SDMMC Slot 1 Modus** (nicht lahmes SPI!):
  * `CLK`: **`IO39`** (`MTCK`)
  * `CMD`: **`IO38`**
  * `DAT0`: **`IO40`** (`MTDO`)
  * `DAT1`: **`IO41`** (`MTDI`)
  * `DAT2`: **`IO42`** (`MTMS`)
  * `DAT3`: **`IO2`**
  * *Durchsatz:* Bis zu $30\text{–}40\,\text{MB/s}$ per Hardware-DMA für Audio-WAV-Streaming und Grafik-Assets.

---

## 10. Pinout-Tabellen (Castellated Pads & M.2-Fingers)

### Die 10 Castellated Edge-Pads (Übersicht)

<!-- BILD-PLATZHALTER: EDGE PADS GRAFIK -->
> 📷 **BILD 4 BENÖTIGT:** *Grafik oder beschrifteter Screenshot der beiden Platinenkanten mit Pin-Namen.*

| Kante | Gruppe | Pin-Nr. | Signal-Name | Funktion |
| :--- | :--- | :---: | :--- | :--- |
| **Rechts (Oben)** | **NeoPixel Bus** | 1 | `GND` | Masse für LED-Strip |
| | | 2 | `NPD_IN` | 3.3V Datenleitung (GPIO4) |
| | | 3 | `VN` | Geschaltete Power-Schiene |
| **Rechts (Unten)**| **Audio Output** | 1 | `SPK+` | Lautsprecher Ausgang (+) |
| | | 2 | `SPK-` | Lautsprecher Ausgang (-) |
| **Links (Unten)** | **I2C & Serial** | 1 | `IO9` | I2C Bus: **SCL** |
| | | 2 | `IO8` | I2C Bus: **SDA** |
| | | 3 | `RX` | UART0: **RX** (ESP32-S3 Empfang) |
| | | 4 | `TX` | UART0: **TX** (ESP32-S3 Senden) |
| **Links (Oben)** | **System / GPIO**| *(J9 TOP)* | TBD | *(In Belegung: 3V3, GND, VBAT, Reset, GPIOs)* |

---

## 11. Software-Cheatsheet (GPIOs, I2C-Adressen, Snippets)

### I2C-Adress-Tabelle (Bus auf `IO8` / `IO9`)

| Bauteil | Typ | 7-Bit I2C-Adresse (Hex) |
| :--- | :--- | :---: |
| **`MAX17048`** | LiPo Fuel Gauge | **`0x36`** |
| **`BNO085`** | 9-DOF IMU Co-Processor | **`0x4A`** *(Default)* |
| **`RV-3028-C7`** | Ultra-Low-Power RTC | **`0x52`** |

---

### Schnelle Initialisierung in Arduino / ESP-IDF

```cpp
#include <Arduino.h>
#include <Wire.h>

#define PIN_LED_BUILTIN   13
#define PIN_NEO_POWER     12
#define PIN_NEO_DATA       4
#define PIN_RF_CTRL       10
#define PIN_I2C_SDA        8
#define PIN_I2C_SCL        9

void setup() {
  // 1. Serielle Schnittstelle starten
  Serial.begin(115200);

  // 2. Status-LED
  pinMode(PIN_LED_BUILTIN, OUTPUT);
  digitalWrite(PIN_LED_BUILTIN, HIGH);

  // 3. NeoPixel Power-Rail einschalten
  pinMode(PIN_NEO_POWER, OUTPUT);
  digitalWrite(PIN_NEO_POWER, HIGH); // 5V / VBAT an NeoPixel anlegen

  // 4. RF-Antenne auf interne Keramik-Antenne stellen (LOW)
  pinMode(PIN_RF_CTRL, OUTPUT);
  digitalWrite(PIN_RF_CTRL, LOW);    // LOW = Keramik, HIGH = U.FL

  // 5. I2C Bus initialisieren
  Wire.begin(PIN_I2C_SDA, PIN_I2C_SCL, 400000); // 400 kHz Fast Mode
}

void loop() {
  // Heartbeat Blink
  digitalWrite(PIN_LED_BUILTIN, HIGH);
  delay(500);
  digitalWrite(PIN_LED_BUILTIN, LOW);
  delay(500);
}
```
