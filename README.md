# ESP32-S3-AMOLED-SmartWatch

Projektplan und technische Referenz für einen Port von MicroPythonOS auf die
Waveshare ESP32-S3-Touch-AMOLED-2.06.

## Stand

04.07.2026 · Alle Hardware-Fakten unten wurden aus dem offiziellen
Waveshare-Repository (`waveshareteam/ESP32-S3-Touch-AMOLED-2.06`), dem
MicroPythonOS-Repository und dessen `lvgl_micropython`-Fork extrahiert und
verifiziert.

## Ausgangslage

MicroPythonOS läuft nicht out-of-the-box auf diesem Board und benötigt einen
Port. Positiv ist, dass MicroPythonOS bereits einen AXP2101-PMIC-Treiber und
mit `lilygo_t_watch_s3_plus.py` eine nahe Smartwatch-Referenz besitzt. Das
größte technische Risiko ist der fehlende CO5300-Displaytreiber in
`lvgl_micropython`.

## Verifizierte Hardware-Fakten

### Chip & Speicher

| Komponente | Wert | Konsequenz |
| --- | --- | --- |
| SoC | ESP32-S3R8, 2× LX7 @ 240 MHz | Build-Target `ESP32_GENERIC_S3` |
| PSRAM | 8 MB Octal | Build-Variant `SPIRAM_OCT` |
| Flash | 32 MB | `flash_size=32` setzen |
| USB | Type-C direkt am S3 | REPL via USB-CDC; Lightsleep trennt USB |
| Akku | 3,7 V / 400 mAh | Laufzeit stark von Helligkeit/Sleep abhängig |

### Pin-Map

#### Display CO5300 — QSPI

| Signal | GPIO |
| --- | --- |
| SDIO0 / SDIO1 / SDIO2 / SDIO3 | 4 / 5 / 6 / 7 |
| SCLK | 11 |
| CS | 12 |
| RESET | 8 |
| Auflösung | 410 × 502 |
| Helligkeit | Register `0x51` (`0x00`–`0xFF`) |

#### Touch FT3168 — I2C

| Signal | GPIO |
| --- | --- |
| SDA / SCL | 15 / 14 |
| INT | 38 |
| RESET | 9 |

Hinweis: GPIO38 ist kein RTC-GPIO, daher kein Deep-Sleep-Wake-up per Touch.

#### Gemeinsamer I2C-Bus

Auf GPIO14/15 hängen FT3168, AXP2101 (`0x34`), QMI8658, PCF85063 und ES8311.
Es sollte daher genau eine gemeinsam genutzte `machine.I2C`-Instanz geben.

#### Weitere Peripherie

- SD-Karte (SDMMC 1-bit): CLK=2, CMD=1, DATA=3, SPI-Fallback-CS=17
- Audio ES8311 / I2S: BCLK=41, WS=45, DOUT=40, DIN=42, MCLK=16, PA-Enable=46
- BOOT-Button: GPIO0
- PWR-Button: am AXP2101-PKEY, Events per PMIC

## Software-Status

| Baustein | Status | Hinweis |
| --- | --- | --- |
| AXP2101-Treiber | ✅ vorhanden | In MicroPythonOS verfügbar |
| Smartwatch-Referenzboard | ✅ vorhanden | `lilygo_t_watch_s3_plus.py` |
| ESP32-S3 + Octal-PSRAM Build | ✅ vorhanden | `ESP32_GENERIC_S3` + `SPIRAM_OCT` |
| CO5300-Displaytreiber | ❌ fehlt | Kernaufgabe des Ports |
| Beste Vorlage | ✅ | `rm67162` |
| CO5300-Init-Sequenz | ✅ | Aus `Arduino_CO5300.cpp` |
| FT3168-Touchtreiber | ⚠️ anpassen | FT-Familie vorhanden, Board-spezifisch prüfen |
| QMI8658 / PCF85063 | ⚠️ geringes Risiko | Community-Treiber verfügbar |
| ES8311 Audio | ❌ offen | Eher v2 als MVP |

## Umsetzungsplan

### Phase 0 — Setup

1. Linux-Umgebung und USB-Zugriff einrichten.
2. `MicroPythonOS/MicroPythonOS` und `waveshareteam/ESP32-S3-Touch-AMOLED-2.06`
   klonen.
3. `esptool`, `mpremote` und MPOS-Build-Abhängigkeiten installieren.
4. Board per `/dev/ttyACM0` verifizieren.

**Done wenn:** `esptool.py flash_id` 32 MB Flash meldet.

### Phase 1 — Baseline & Hardware-Verifikation

1. Vollständiges Flash-Backup ziehen:
   `esptool.py read_flash 0 0x2000000 backup_original.bin`
2. Werksdemo für Display, Touch und Laden prüfen.
3. Optional ein Arduino-Beispiel selbst bauen und flashen.

### Phase 2 — lvgl_micropython bauen

1. Mit `ESP32_GENERIC_S3`, `SPIRAM_OCT` und `flash_size=32` bauen.
2. Flashen, REPL via `mpremote` testen und PSRAM prüfen.

### Phase 3 — PMIC + Display-Bring-up

1. AXP2101 zuerst initialisieren.
2. CO5300-Treiber aus `rm67162` ableiten und mit der verifizierten
   Hersteller-Init-Sequenz portieren.
3. Mit einem LVGL-„Hello World” testen.
4. Framebuffer in PSRAM allozieren.

**Done wenn:** Anzeige scharf und farbrichtig läuft und `0x51` die Helligkeit
regelt.

### Phase 4 — Touch

1. FT3168 an I2C(14/15), INT=38, RST=9 anbinden.
2. Als LVGL-Indev registrieren und Koordinaten prüfen.

### Phase 5 — MicroPythonOS-Boarddatei

1. `waveshare_esp32_s3_touch_amoled_206.py` analog zur T-Watch-Referenz
   anlegen.
2. Board in Build-/Auswahllogik registrieren.
3. Komplettes OS bauen, flashen und Launcher testen.

### Phase 6 — Peripherie vervollständigen

- Batterieanzeige über AXP2101
- RTC PCF85063
- IMU QMI8658
- SD-Karte
- PWR-Button
- Audio ES8311 optional

### Phase 7 — Alltagstauglichkeit

- Display-Timeout, AMOLED-Off und `machine.lightsleep()`
- WiFi nur on-demand
- schwarzes Watchface für geringe Leistungsaufnahme
- realistische Laufzeit: 1–2 Tage mit gutem Duty-Cycle

## Risiken

| Risiko | Einschätzung | Gegenmaßnahme |
| --- | --- | --- |
| CO5300-Treiber zickt | mittel | Herstellersequenz 1:1 übernehmen, `rm67162` als Vorlage |
| Falsche PMIC-Rails | hoch | Rails aus Demo/Schaltplan verifizieren |
| Build-/IDF-Konflikte | mittel | Gepinnte MPOS-Versionen beibehalten |
| Lightsleep trennt USB-CDC | sicher | Sleep während Entwicklung deaktivierbar halten |
| MPOS API-Änderungen | mittel | Auf Release-Tag statt `main` pinnen |

## Nächster konkreter Schritt

Vor Phase 3 einmal `lvgl_micropython`-Issues nach `CO5300` durchsuchen. Falls
es dort bereits Vorarbeit gibt, reduziert das den Portierungsaufwand erheblich.
