# KLOR polydactyl — ZMK mit Azoteq TPS65 Trackpad

ZMK-Konfiguration für einen KLOR im polydactyl-Layout, kabellos über BLE,
mit einem Azoteq TPS65 Trackpad in der rechten Hälfte.

Bewusst weggelassen: **kein TRRS-Jack, kein OLED, kein Buzzer, kein Speaker.**
Die Hälften reden ausschließlich über Bluetooth miteinander.

---

## Die Tastatur

| | |
|---|---|
| Layout | KLOR polydactyl, 3×5 + 3 Daumentasten pro Hälfte = **36 Tasten** |
| Controller | 2 × nice!nano v2 (oder kompatibel) |
| Verbindung | BLE split, **rechts ist central** |
| Zeigegerät | Azoteq TPS65, 65 × 49 mm, kapazitiv, rechte Hälfte |
| Encoder | einer, linke Hälfte |
| Akku | 3,7 V LiPo, rechts unter dem Trackpad, links flach unter der Blende |

Das polydactyl-Layout bietet pro Hälfte 22 Positionen. Sieben davon bleiben
absichtlich unbestückt, damit die Matrix der gewohnten Skeletyl-Belegung entspricht:

- die **äußere Zusatzspalte** (2 Tasten pro Hälfte)
- die **äußerste Daumentaste**, die unter der Handfläche liegt
- rechts die **Encoder-Position** — dort sitzt das Trackpad

Verifiziert an der Switchplate-Geometrie: 21 MX-Ausschnitte plus eine
Encoder-Öffnung pro Hälfte, die äußere Spalte trägt nur 2 statt 3 Tasten.

### Warum rechts central ist

Der Trackpad-Treiber meldet seine Events über das ZMK-Input-Subsystem. Sitzt das
Pad auf der peripheren Hälfte, muss man den Umweg über `zmk,input-split` gehen.
Da das Pad ohnehin rechts sitzt, ist die rechte Hälfte hier central — das spart
diese Indirektion komplett.

---

## Trackpad anschließen

Das TPS65 hat einen 6-poligen ZIF-Anschluss (0,5 mm Raster):

| ZIF-Pin | Signal | anschließen an | ZMK |
|---|---|---|---|
| 1 | RDY | **Buzzer-Pad** auf dem KLOR-PCB | `&pro_micro 9` = P1.06 |
| 2 | NRST | *offen lassen* | — |
| 3 | GND | OLED-Header GND | — |
| 4 | VDDHI | OLED-Header VCC (3,3 V) | — |
| 5 | SCL | OLED-Header SCL | `&pro_micro 3` = P0.20 |
| 6 | SDA | OLED-Header SDA | `&pro_micro 2` = P0.17 |

**Der OLED-Header liegt auf dem Hardware-I²C.** SDA/SCL des KLOR gehen auf
`pro_micro_i2c` des nice!nano — siehe `zmk/app/boards/nicekeyboards/nice_nano/nice_nano_nrf52840_zmk.dts`.
Da kein Display verbaut wird, ist der Header frei und liefert alle vier
Versorgungs- und Busleitungen an einem Punkt.

**RDY braucht einen eigenen GPIO.** Der Header hat keinen freien Pin übrig.
`&pro_micro 9` führt beim KLOR zum Buzzer-Footprint — der bleibt unbestückt.
RGB, Encoder und TRRS bleiben unangetastet.
*Alternative*, falls der Buzzer doch gewünscht ist: eines der Extra-Pads des
nice!nano (P1.01, P1.02, P1.07) anzapfen; die sind auf dem KLOR-PCB unbeschaltet.

#### Zwei Stellen für RDY — dasselbe Netz

Pin 9 des Controllers und das Buzzer-Pad hängen am selben Netz (`AUDIO`).
Elektrisch macht es keinen Unterschied, wo abgegriffen wird.

| | direkt an Pin 9 | Buzzer-Pad BZ1 |
|---|---|---|
| Weg zum Trackpad | kurz | länger |
| Controller bleibt steckbar | nein | ja |

**Direkt an Pin 9** ist der kürzere Weg und die erste Wahl, wenn der Controller
ohnehin fest verlötet ist. Sitzt er in Buchsenleisten, geht die Steckbarkeit
verloren — dann entweder an die Durchkontaktierung des Pro-Micro-Footprints
löten (das PCB-Pad direkt neben dem Stiftende) oder das Buzzer-Pad nehmen.

Pin 9 ist der **letzte Pin der Reihe, die mit TX0/RXI beginnt**, also am Ende
gegenüber vom USB-Anschluss. Nicht nach links/rechts orientieren: das KLOR-PCB
ist reversibel, der Controller wird auf einer Hälfte gespiegelt bestückt.
Sichere Identifikation mit dem Durchgangsprüfer — **Pin 9 ist der Pin mit
Durchgang zum AUDIO-Pad von BZ1**.

Das Buzzer-Pad selbst, aus `klor1_3.kicad_pcb`:

| | |
|---|---|
| Siebdruck | **BZ1** |
| Bauteil | Mallory AST1109MLTRQ (Piezo) |
| Pads | 2 × durchkontaktiert, quadratisch, 2,2 × 2,2 mm |
| Pad-Abstand | 13,2 mm |
| Pad 1 | Netz **AUDIO** → hier anlöten |
| Pad 2 | Netz **GND** → Finger weg |

Es liegt auf gleicher Höhe wie der Rotary-Encoder, rund 65 mm zur anderen
Boardseite hin. Die beiden Pads sehen identisch aus; das GND-Pad erkennst du am
Durchgang nach Masse.

**NRST bleibt offen.** Das IQS5xx-B000-Datenblatt: der Pin hat einen permanenten
internen Pull-up, *"thus an external component is not required"*. Der Treiber
behandelt `reset-gpios` entsprechend als optional.

### Pull-ups prüfen

Diese Konfiguration nimmt an, dass die **I²C-Pull-ups auf dem TPS65-Modul sitzen**.
Ohne OLED gibt es sonst keine mehr im System — der SSD1306 hatte sie mitgebracht,
das KLOR-PCB hat keine (im gesamten Schaltplan kommt kein einziger Widerstand vor).

Vor dem Einbau mit dem Multimeter messen:

```
ZIF-Pin 4 (VDDHI) ↔ Pin 5 (SCL)    ~4,7 kΩ  →  vorhanden
ZIF-Pin 4 (VDDHI) ↔ Pin 6 (SDA)    ~4,7 kΩ  →  vorhanden
```

Hochohmig? Dann zwei 4,7-kΩ-Widerstände nach 3,3 V nachrüsten, am einfachsten
auf dem FFC-Breakout.

Anhaltspunkt für die Annahme: der offene TPS65-Nachbau
[GR-Trackpad65](https://github.com/geek-rabb1t/GR-Trackpad65) hat R1/R2 = 4,7 kΩ
von SCL bzw. SDA nach VDD. Für das Original-Modul veröffentlicht Azoteq keinen
Schaltplan.

### Kabel

6-poliges FFC, 0,5 mm Raster, **60 mm** genügt. Das Pad sitzt direkt über der
Gehäusekammer, die Switchplate ist darunter offen — das Kabel fällt senkrecht
auf das Breakout. Freie Höhe unter dem Pad: 5,57 mm.

Ob Type A oder Type D gebraucht wird, ergibt sich erst aus der Kombination der
beiden ZIF-Buchsen (Pad und Breakout). Beide Varianten bestellen.

---

## Pinbelegung KLOR 1.3

Aus `PCB/klor1_3/klor1_3.kicad_sch` des [KLOR-Repos](https://github.com/GEIGEIGEIST/KLOR) extrahiert:

| Pro Micro | ZMK `&pro_micro` | nRF52840 | Netz |
|---|---|---|---|
| TX0 | 1 | P0.06 | LED (RGB-Data) |
| RXI | 0 | P0.08 | TX (TRRS, unbenutzt) |
| 2 | 2 | P0.17 | **SDA** |
| 3 | 3 | P0.20 | **SCL** |
| 4 | 4 | P0.22 | RX (TRRS, unbenutzt) |
| 5 | 5 | P0.24 | row0 |
| 6 | 6 | P1.00 | row1 |
| 7 | 7 | P0.11 | row2 |
| 8 | 8 | P1.04 | row3 |
| 9 | 9 | P1.06 | **RDY** (früher Buzzer) |
| 10 | 10 | P0.09 | col5 |
| 16 | 16 | P0.10 | col4 |
| 14 | 14 | P1.11 | col3 |
| 15 | 15 | P1.13 | col2 |
| A0 | 18 | P1.15 | col1 |
| A1 | 19 | P0.02 | col0 |
| A2 | 20 | P0.29 | ENCB |
| A3 | 21 | P0.31 | ENCA |

---

## Keymap

Vier Layer, übernommen aus der Skeletyl-Konfiguration.

```
        ┌───────┬───────┬───────┬───────┬───────┐         ┌───────┬───────┬───────┬───────┬───────┐
        │   Q   │   W   │   E   │   R   │   T   │         │   Y   │   U   │   I   │   O   │   P   │
 ┌──────┼───────┼───────┼───────┼───────┼───────┤         ├───────┼───────┼───────┼───────┼───────┼──────┐
 │  --  │   A   │   S   │   D   │   F   │   G   │         │   H   │   J   │   K   │   L   │   ;   │  --  │
 ├──────┼───────┼───────┼───────┼───────┼───────┼────┬────┼───────┼───────┼───────┼───────┼───────┼──────┤
 │  --  │   Z   │   X   │   C   │   V   │   B   │ENC │ TP │   N   │   M   │   ,   │   .   │   /   │  --  │
 └──────┴───────┴───────┴───┬───┴───┬───┴───┬───┴────┴────┴───┬───┴───┬───┴───┬───┴───────┴───────┴──────┘
                        │--││ ESC   │ SPACE │  TAB  │ │ LCLK  │ RET   │ BSPC  ││--│
                        └──┘└───────┴───────┴───────┘ └───────┴───────┴───────┘└──┘
```

`--` markiert die unbelegten Zusatzpositionen. `ENC` ist der Encoder-Taster
(Mute), `TP` die vom Trackpad belegte Position.

| Layer | Zugriff | Inhalt |
|---|---|---|
| Base | — | Buchstaben, Homerow-Mods |
| Sym | Daumen rechts | Klammern, Operatoren, Sonderzeichen |
| Num | Daumen links | Ziffernblock rechts |
| Nav | Daumen links | Pfeile, Scroll, Bluetooth-Profile |

Homerow-Mods verwenden `hold-trigger-key-positions`, damit Mods nur bei
gegenüberliegenden Tasten auslösen. Die Positionslisten stehen als
`KEYS_L` / `KEYS_R` am Kopf der Keymap.

Encoder: drehen = Lautstärke, drücken = Mute.

---

## Bauen

```
config/
├── west.yml                 ZMK main + Azoteq-Treibermodul
├── klor.conf                Nutzer-Einstellungen
├── klor.keymap              Keymap, 4 Layer
└── boards/shields/klor/
    ├── klor_common.dtsi     kscan, Encoder, Sensoren
    ├── klor.dtsi            Matrix-Transform, Reihen-GPIOs
    ├── klor_left.overlay    Spalten links, Encoder aktiv
    ├── klor_right.overlay   Spalten rechts, Trackpad-Node
    └── boards/
        └── nice_nano_nrf52840_zmk.overlay   WS2812-Kette (nur bei aktivem RGB)
```

Push nach GitHub genügt — der Workflow in `.github/workflows/build.yml` baut
beide Hälften und legt die `.uf2`-Dateien als Artefakt ab.

Lokal mit west:

```sh
west init -l config
west update
west build -s zmk/app -b "nice_nano//zmk" -- -DSHIELD=klor_right -DZMK_CONFIG="$PWD/config"
```

---

## Abweichungen vom Original

Basis ist [GEIGEIGEIST/zmk-config-klor](https://github.com/GEIGEIGEIST/zmk-config-klor) (MIT).
Geändert wurde:

- **OLED entfernt.** Der `ssd1306`-Node und `chosen zephyr,display` sind raus,
  ebenso die Display-, LVGL- und Icon-Teile. Der I²C-Bus bleibt aktiv — er
  versorgt jetzt das Trackpad.
- **Rechte Hälfte ist central** statt der linken.
- **Kein `right_encoder`.** Die Position ist vom Trackpad belegt, deshalb steht
  in `sensors` nur noch der linke Encoder.
- **Encoder modernisiert.** `resolution` ist in aktuellem ZMK durch `steps` am
  Encoder-Node und `triggers-per-rotation` am Sensor-Node ersetzt.
- **RGB standardmäßig aus.** Auf dem PCB weiterhin möglich, kostet aber spürbar
  Laufzeit. Zum Aktivieren die beiden Zeilen in `klor.conf` einkommentieren und
  `chain-length` in `boards/nice_nano_nrf52840_zmk.overlay` ans Layout anpassen.
- **Trackpad ergänzt** in `klor_right.overlay`, Treiber über `west.yml`.

Trackpad-Treiber: [AYM1607/zmk-driver-azoteq-iqs5xx](https://github.com/AYM1607/zmk-driver-azoteq-iqs5xx),
explizit mit TPS43 und TPS65 getestet.

---

## Noch offen

- **Erster Build ist der Test.** Diese Konfiguration ist hier nicht kompiliert
  worden. ZMK main bewegt sich; falls der Build bricht, sind die wahrscheinlichsten
  Kandidaten die Encoder-Properties und `CONFIG_ZMK_POINTING`.
- **Achsenrichtung des Pads.** `switch-xy` ist gesetzt, weil das Pad quer
  eingebaut ist. Ob zusätzlich `flip-x` oder `flip-y` nötig ist, zeigt der erste
  Flash — beide Properties stehen kommentiert im Overlay.
- **Pull-ups messen** (siehe oben).
- **Kein Buzzer möglich**, solange RDY auf `&pro_micro 9` liegt.
