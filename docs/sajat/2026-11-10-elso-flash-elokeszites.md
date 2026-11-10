# Első USB flash előkészítése – 2026-11-10

Napló az első ESP flash előtti vizsgálatról és javításokról. Eszköz: NodoShop OTGW v2.13 panel
(PIC 6.8) + LOLIN S3 Mini (ESP32-S3), `sajat` ág (a `dev` 2.x ágból indul), USB: COM5.

## Folyamatábra

```mermaid
flowchart TD
    A([Kérés: terv az első USB flashre]) --> B[Fájlok átnézése:<br/>FLASH_GUIDE, BUILD, platformio.ini,<br/>flash_esp.py, flash_otgw.bat, build.py,<br/>partitions_otgw_esp32.csv]
    B --> C{Melyik flash eszköz?}
    C -->|flash_esp.py| C1[1.x / ESP8266 eszköz:<br/>/releases/latest → 1.x kiadás,<br/>'esp32' szűrő rossz képet választhat]
    C1 --> C2[ELVETVE]
    C -->|flash_otgw.bat| C3[ESP32-S3 / 2.x eszköz<br/>factory mód: erase + merged-full @0x0]
    C3 --> D{Melyik PlatformIO env?}
    D -->|esp32| D1[OTGW32, PIC nélkül → nem jó<br/>CLAUDE.md hibásan ezt írta]
    D -->|esp32-classic| D2[fix PIC – alternatíva]
    D -->|esp32-combo| D3[VÁLASZTVA: alap build,<br/>PIC-et boot közben felismeri]
    D3 --> E[A lépés: git állapot ellenőrzése]
    E --> F{version.h / version.hash}
    F --> F1[csak automatikus verzióléptetés<br/>→ nem kell vele foglalkozni]
    E --> G{Almodulok}
    G --> G1[OpenTherm és SimpleTelnet mappa ÜRES<br/>+ rossz commiton álltak]
    G1 --> G2[git submodule update --init --force<br/>→ f7d8254 / a1067d4]
    G2 --> G3[forrásfájlok megvannak, status tiszta]
    E --> H[CLAUDE.md javítása:<br/>env = esp32-combo, flash eszköz,<br/>9. pont állapot]
    H --> I[Commit 2574626b4 a sajat ágon]
    G3 --> J
    I --> J([Következő: B lépés – build.bat])
```

## Mit és miért

### 1. Vizsgálat (csak olvasás)

| Kérdés | Eredmény | Forrás |
| --- | --- | --- |
| Hol vannak a doksik? | `docs/guides/FLASH_GUIDE.md`, `docs/guides/BUILD.md` (nem `docs/` alatt) | – |
| Jó-e a `flash_esp.py`? | **Nem.** 1.x / ESP8266 eszköz. A `/releases/latest` végpont kihagyja a prerelease-eket, ezért az 1.x kiadást adja (kódból következtetve, élőben nem próbáltuk). Az `esp32` fájlnév-szűrő az OTGW32-es, a classic és a combo képre is illik. | `flash_esp.py:69`, `:470`; `FLASH_GUIDE.md:3` |
| Mivel flasheljünk? | `flash_otgw.bat` factory mód: teljes törlés, majd egy `*-merged-full.bin` írása 0x0-ra | `flash_otgw.bat:233` |
| Kell külön fájlrendszer-flash? | **Nem.** A merged-full kép tartalmazza: bootloader, partíciós tábla, safeboot, firmware, LittleFS | `build.py:2120-2144` |
| Melyik env? | **`esp32-combo`**. Az `esp32` az OTGW32 (PIC nélkül), a te paneleden nem működne. | `platformio.ini:3-5`, `build.py:13` |
| Partíciók | safeboot 0x10000, app0 0x100000, LittleFS 0x330000, 4 MB flash | `partitions_otgw_esp32.csv` |

### 2. Talált hibák és javításuk

**a) Üres almodulok.** A `src/libraries/OpenTherm` és a `src/libraries/SimpleTelnet` mappában csak
a `.git` hivatkozás volt, forrásfájl nem. A két almodul ráadásul újabb commiton állt, mint amit a repó vár.
Ezek nélkül a build nem fordult volna le. Valószínű ok: félbemaradt klónozás.

```
git submodule update --init --force src/libraries/OpenTherm src/libraries/SimpleTelnet
```

Eredmény: OpenTherm `f7d8254`, SimpleTelnet `a1067d4`, pontosan a `dev` ág hivatkozása szerint.
Az `other-projects` almodult szándékosan nem töltöttük le: SAT referenciakód, a build nem használja.

> Ha egy másik gépen újra klónozol: `git clone --recurse-submodules ...`, vagy klónozás után
> `git submodule update --init src/libraries/OpenTherm src/libraries/SimpleTelnet`.

**b) Hibás env a CLAUDE.md-ben.** Átírva `esp32`-ről `esp32-combo`-ra, és bekerült, hogy a flash
eszköz a `flash_otgw.bat`.

### 3. Commit

- `2574626b4` – *Add fork CLAUDE.md, keep upstream instructions as CLAUDE.upstream.md* (`sajat` ág, nincs pusholva)
- Kimaradt a commitból: `src/OTGW-firmware/version.h`, `data/version.hash`. Ezeket a build automatikusan írja.

## A flash terve (még hátravan)

1. **B – build:** `.\build.bat` → `build\OTGW-firmware-esp32-combo-<verzió>-merged-full.bin`
2. **C – flash (USB):** `.\flash_otgw.bat --port COM5 --bin build\OTGW-firmware-esp32-combo-<verzió>-merged-full.bin`
   - Ha nem kapcsolódik: tartsd nyomva a 0 (BOOT) gombot, nyomd meg az RST-t, engedd el a 0-t; vagy `--baud 115200`.
3. **D – első indítás:** a lapkát feszültségmentes panelbe tedd, a Wi-Fi-t az `OTGW-<MAC>` AP-n add meg, az IP-t DHCP-ben foglald le.
4. **E – ellenőrzés:** webes felület IP-címmel (látszik-e a PIC 6.8), telnet napló a 23-as porton.
