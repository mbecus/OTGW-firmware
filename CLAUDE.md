# CLAUDE.md – OTGW-firmware (saját fork)

> Ez a fájl a saját forkom munkaszabályait tartalmazza, és felülírja az eredeti
> repó karbantartói utasításait (azok a `CLAUDE.upstream.md` fájlban vannak).
> Ütközés esetén **ez a fájl az irányadó**.

## 1. Kommunikáció

- Mindig **magyarul** válaszolj. Kódban, commit üzenetekben és kódkommentekben angolul írj (az eredeti projekt nyelve).
- Hobbi szinten programozok, és tanulni szeretnék. Röviden magyarázd el, **mit és miért** változtatsz; a szakkifejezéseket első előfordulásnál magyarázd meg egy félmondatban.
- Nagyobb (több fájlt érintő, vagy a fűtésvezérlést befolyásoló) változtatás előtt adj tervet, és várd meg a jóváhagyásomat.
- Ha valami nem egyértelmű, kérdezz – egyszerre egy kérdést, választási lehetőségekkel és a javaslatoddal.
- Ne állíts olyat a hardverről vagy a kódról, amit nem ellenőriztél: olvasd el a forrást, vagy mondd, hogy feltételezés.

## 2. A projekt röviden

Az OTGW-firmware a NodoShop OpenTherm Gateway (OTGW) ESP lapkáján fut. Figyeli a termosztát és a kazán közötti OpenTherm buszt, és az adatokat MQTT-n (Home Assistant auto-discovery), webes felületen és REST API-n (`/api/v2/`) teszi elérhetővé. Az OTGW-n egy PIC mikrokontroller beszél a busszal; az ESP soros porton kommunikál vele.

| Ág | Cél hardver | Megjegyzés |
| --- | --- | --- |
| `dev` (2.x) | ESP32-S3 (LOLIN S3 Mini) | alpha, async + FreeRTOS |
| `otgw-1.x.x` (1.x) | ESP8266 (NodeMCU / Wemos D1 mini) | stabil karbantartási ág |
| `sajat` | – | **az én munkaágam**, ide kerül minden módosításom |

Fő helyek a repóban:

- `src/` – a firmware forrása (`src/libraries/` alatt idegen, beemelt könyvtárak: OpenTherm, OTGWSerial – ezekhez csak indokolt esetben nyúlj)
- `include/` – fejlécfájlok
- `platformio.ini` – build definíció (PlatformIO)
- `tests/`, `test/host/`, `scripts/run_tests.py` – tesztek
- `docs/` – dokumentáció: `BUILD.md`, `FLASH_GUIDE.md`, `api/MQTT.md`, `api/openapi.yaml`, `adr/` (architektúra döntések)
- `.claude/rules/` – az eredeti projekt részletes kódolási szabályai (lásd 5. pont)

## 3. Saját eszközöm

- Hardver: NodoShop OTGW panel (PIC-kel) + **LOLIN S3 Mini (ESP32-S3)**. Panelverzió: **v2.13** (Wemos-foglalatos panel, a LOLIN S3 Mini a Wemos D1 mini helyére kerül).
- Ág: a `sajat` ág a **`dev` (2.x)** ágból indul. A 2.x alpha állapotú; ha valami hibás, előbb nézd meg, hogy az eredeti `dev` ágon is hibás-e.
- PlatformIO environment: **`esp32-combo`** (a sima `build.bat` ezt fordítja; boot közben felismeri a PIC-et). **Nem** `esp32`: az az OTGW32 panelhez készül, PIC nélkül (`platformio.ini`). Az `esp32-classic` a fix PIC-es alternatíva.
- Flash eszköz: **`flash_otgw.bat`** (ESP32-S3, 2.x). A `flash_esp.py` az 1.x / ESP8266 eszköz, ne használd.
- PIC firmware: **6.8** (OTmonitorral, USB-n feltöltve).
- Állapot: az ESP lapkán **még nincs firmware**, az eszköz még nem volt hálózaton.
- Hálózat: Wi-Fi-n a **Huawei routerhez** csatlakozik (`192.168.1.x`, ugyanaz a háló, ahol a Home Assistant van).
- IP-cím: _az első indítás után kitöltendő_ (utána DHCP-foglalással fixálni a routerben): `192.168.1.___`
- Flash módja: az **első flash USB-n** (COM port: COM5), utána OTA a webes felületen.
- Célok, sorrendben:
  1. **Először az eredeti firmware működjön**: flash, Wi-Fi, MQTT, és a Home Assistantban megjelennek az adatok. Amíg ez nincs meg, ne módosítsd a firmware-t.
  2. Utána **tanulás a kódon**: magyarázatok és kisebb módosítások.
  3. **Saját HA funkciók**: extra MQTT értékek, entitások automatizáláshoz.
  4. **Fűtésvezérlés** (pl. SAT, időjárás-követés): csak óvatosan, felügyelet mellett tesztelve.

## 4. Környezetem

- Home Assistant (HA OS) a `192.168.1.110` címen; az MQTT broker is itt fut.
- Két router, két alhálózat: `192.168.1.x` (fő, itt a HA) és `192.168.2.x` (második router mögött). A két háló között **nincs mDNS**, ezért eszközt mindig **IP-címmel** érj el (OTA, telnet, böngésző).
- Több gépen dolgozom; a gépek között csak a GitHubon keresztül szinkronizálok. A beszélgetés-előzmény gépenként külön van, ezért a fontos állapotot a 9. pontba írd.
- Jelszavakat, Wi-Fi/MQTT hitelesítő adatokat **soha** ne írj a repóba – a fork nyilvános. Ezek az eszköz webes felületén állíthatók.

## 5. Firmware szabályok (kötelezőek)

Ezek az eredeti projektből jönnek, és a hardver miatt fontosak. Részletek: `.claude/rules/` és `docs/adr/`.

- **A `Serial` port a PIC-hez tartozik.** Inicializálás után soha ne írj `Serial`-ra (se debug, se `Serial.print`). Debug kimenet a telnet naplóba megy (23-as port).
- **PIC parancsot csak az `addCommandToQueue()` függvényen keresztül küldj.**
- Ne használj **ArduinoJson**-t, **HTTPS**-t vagy **WSS**-t – a projekt ezeket szándékosan kerüli.
- **Memória:** a szövegkonstansok flash-ben maradjanak (pl. `F()` / `PSTR()` / `PROGMEM`, ahogy a környező kód csinálja); gyakran futó kódban kerüld a `String` osztályt.
- Platformfüggő `#ifdef`-et ne szórj szét a kódban; használd a projekt meglévő platform-absztrakcióját.
- Kövesd a környező kód stílusát és elnevezéseit. Minimális változtatás: csak azt módosítsd, ami a feladathoz kell. KISS, YAGNI.
- MQTT vagy REST API változásnál frissítsd a vonatkozó dokumentációt is (`docs/api/MQTT.md`, `docs/api/openapi.yaml`).
- Hibajavításnál először reprodukáld a hibát, és mutasd meg, miért javítja a módosítás.

## 6. Build, teszt, flash

Fordítás **csak** a projekt szkriptjeivel (ezek állítják be a verziót és a fájlrendszert is):

- Windows: `build.bat`  
- Linux/macOS: `./build.sh`  
- (Közvetlenül, ha kell: `pio run -e <environment>` – lásd `docs/BUILD.md`)

Ellenőrzések minden firmware-módosítás után:

1. Build hiba nélkül lefut.
2. `python evaluate.py --quick` – nincs új hiba a korábbiakhoz képest.
3. `python scripts/run_tests.py` – a tesztek zöldek.

Csak akkor mondd, hogy kész, ha ez a három teljesült; ha valamelyiket nem tudtad futtatni, mondd ki.

Flash:

- Alapból **OTA a webes felületen** (IP-címmel), USB csak ha az OTA nem megy (`docs/FLASH_GUIDE.md`).
- Flash előtt szólj, hogy **mentsem a beállításokat** – főleg ha a fájlrendszer is íródik.
- A flasht és a valós próbát én végzem; te add meg a pontos lépéseket és azt, mit figyeljek a naplóban.
- **Soha ne javasold PIC firmware Wi-Fin keresztüli flashelését OTmonitorral** – téglásíthatja a PIC-et. PIC frissítés csak a webes felület beépített funkciójával.

Hasznos portok az eszközön: 80 (web + REST), 81 (WebSocket napló), 23 (telnet debug), 25238 (soros híd OTmonitorhoz / HA OpenTherm integrációhoz).

## 7. Git szabályok

- Minden munka a **`sajat`** ágon (vagy abból nyitott ágon) történik.
- Commit üzenet angolul, rövid, felszólító módban (pl. `Add MQTT topic for return water temperature`).
- Push **csak az `origin` (a saját forkom) felé** és csak ha kérem. Az `upstream` (rvdbreemen/OTGW-firmware) felé **soha** ne pusholj, ne nyiss PR-t kérés nélkül.
- Force push, ág törlése, `git reset --hard`: csak kifejezett, eseti jóváhagyással.
- Az upstream frissítéseit így húzzuk be: `git fetch upstream` → `git merge upstream/<dev|otgw-1.x.x>`. Konfliktusnál a saját módosításaimat tartsd meg, és sorold fel, mit hogyan döntöttél.
- A 2.x fejlesztés közben az eredeti projekt egy prerelease verziószámot léptet (`bin/bump-prerelease.sh`) és git hookokat használ. Ha egy hook nálam hibát dob, ne kerüld meg csendben (`--no-verify`); mondd el, mi a gond.

## 8. Ami az eredeti projektből NEM vonatkozik rám

A `CLAUDE.upstream.md`, az `AGENTS.md` és a `.claude/rules/` egyes részei a karbantartó munkafolyamatáról szólnak. Ezeket **hagyd figyelmen kívül**:

- Backlog.md / `backlog` CLI feladatkezelés és a TASK-azonosítós commit követelmény
- Discord szabályok, szerepkörök (Robert, Sergeant D, tesztelők)
- Kiadási parancsok: `/release-2x`, `/beta-release-2x`, `/release`, `/beta-prerelease`, `/update-docs*`
- Az eredeti push-engedély (`origin/dev`, `origin/otgw-1.x.x` automatikus push) – nálam a 7. pont érvényes
- OpenWolf (`.wolf/`), graphify, ADR-kit parancsok – csak ha kifejezetten kérem

A **kódolási szabályok** (5. pont és `.claude/rules/` firmware részei) viszont érvényesek.

## 9. Folyamatban

> Claude frissíti egy-egy munkamenet végén, hogy a másik gépen is folytatható legyen. Rövid maradjon.

- Aktuális cél: az eredeti 2.x firmware feltöltése és beüzemelése (1. cél)
- Utolsó állapot (2026-11-10): **az ESP-n fent van a `2.0.0-beta.171+924e662` (esp32-combo)**, USB-n flashelve (factory mód, „Flash complete”). Még nem indult a panelben, nincs Wi-Fi. Napló: `docs/sajat/2026-11-10-elso-flash-elokeszites.md`
- Következő lépés: **D lépés**: lapka a feszültségmentes panelbe → panel bekapcsolása → `OTGW-<MAC>` AP-n Wi-Fi (Huawei, 192.168.1.x) → IP kikeresése és DHCP-foglalása a routerben → IP beírása a 3. pontba. Utána **E lépés**: web UI IP-címmel (PIC 6.8 látszik-e), telnet napló a 23-as porton. Majd MQTT / Home Assistant.
- Új gépen kezdés: `docs/sajat/UJ-GEP-BEALLITAS.md`
- Nyitott kérdések: –
