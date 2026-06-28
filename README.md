# crypto-manage – OpenWrt LUKS Krypto-Manager

Ein schlankes, POSIX-konformes Shell-Skript für **OpenWrt** (BusyBox), das das automatische Öffnen, Einhängen (Mounten), Aushängen (Unmounten) und Schließen von LUKS-verschlüsselten Festplatten und Containern orchestriert. 

Das Skript liest die Konfiguration nativ aus dem OpenWrt-System (`UCI`) und verfügt über einen integrierten Schutz vor Datenverlust durch Blockade-Prüfung (`fuser`) sowie einen simulierten Testlauf-Modus (`Dry-Run`).

---

## 1) Einführung & Konzept

### Woher kommt die Idee?
OpenWrt bietet über Pakete wie `cryptsetup` eine hervorragende Krypto-Unterstützung, besitzt jedoch standardmäßig kein automatisiertes, dynamisches System, um verschlüsselte Festplatten im laufenden Betrieb sauber per Skript zu verwalten. Das manuelle Abfeuern von `cryptsetup open`, das Erstellen von Mountpoints, das Einhängen über das richtige Device-Mapping und das spätere saubere Schließen (ohne offene Dateihandles zu zerstören) ist fehleranfällig und unhandlich.

### Was soll das Ganze?
`crypt-manage` schließt diese Lücke. Es fungiert als intelligentes Bindeglied zwischen OpenWrts Konfigurations-Schnittstelle (UCI) und den Linux-Systemwerkzeugen. 

**Die Kernvorteile:**
* **Zentralisiert:** Es nutzt die nativen OpenWrt-Konfigurationsdateien (`/etc/config/cryptsetup` und `/etc/config/fstab`). Keine redundanten Pfade im Skript.
* **Sicher (Safety First):** Vor dem Schließen einer verschlüsselten Partition wird geprüft, ob noch Prozesse auf die Platte zugreifen. Ein harter Datenverlust wird so verhindert.
* **Trockenübung (Dry-Run):** Jeder Befehl wird standardmäßig nur *simuliert*. Erst durch das explizite Anhängen des Parameters `go` wird die Aktion scharf geschaltet.

---

## 2) Voraussetzungen

### System-Abhängigkeiten
Damit das Skript auf OpenWrt funktioniert, müssen die folgenden Pakete installiert sein:

```bash
opkg update
opkg install cryptsetup fuser kmod-crypto-xts
```

(Hinweis: uci, mount, umount und awk sind standardmäßig Teil der BusyBox und bereits auf dem System vorhanden).

### Das grundlegende Verständnis (Wichtig!)
Dieses Skript dient nicht dazu, eine verschlüsselte Festplatte neu zu erstellen. Um die Sicherheit deiner Daten zu gewährleisten, musst du das Krypto-Dateisystem vorab einmalig von Hand einrichten. Dies stellt sicher, dass du den Verschlüsselungs- und Schlüsselprozess (Passphrase) verstanden hast.

### Kurzanleitung: LUKS-Platte manuell vorbereiten
1. **Festplatte verschlüsseln (Formatieren):**
   
```bash
cryptsetup luksFormat /dev/sda1
```
2. **Testweise öffnen:**

```bash
cryptsetup open /dev/sda1 mein_safe
```

3. **Dateisystem erstellen (z. B. ext4):**

```bash
mkfs.ext4 /dev/mapper/mein_safe
```
4. **Wieder schließen:**

```bash
cryptsetup close mein_safe
```

## 3) Dokumentation der Implementierung
#### Ablauf & Funktionsweise
UCI auslesen: Das Skript prüft die OpenWrt-Konfiguration. Wird kein spezifischer Name übergeben, ermittelt es automatisch alle konfigurierten Krypto-Geräte.

Modus-Weiche:

open: Holt sich das physische Device aus /etc/config/cryptsetup und den Ziel-Pfad aus /etc/config/fstab. Öffnet den LUKS-Container und bindet das Dateisystem ein.

close: Ermittelt den aktuellen Mountpoint. Prüft via fuser, ob noch offene Dateien oder Prozesse die Platte blockieren. Wenn alles frei ist, wird sauber ausgehängt (umount) und der LUKS-Container geschlossen.

#### Ressourcen und Konfiguration
Das Skript greift ausschließlich auf die Standard-Konfigurationen von OpenWrt zu:

##### 1. Krypto-Zuordnung: /etc/config/cryptsetup
Hier wird definiert, welche UUID oder welches physische Device zu welchem Krypto-Namen gehört.

```bash
config cryptsetup 'backup_safe'
option device '/dev/sda1'
```

##### 2. Mount-Zuordnung: /etc/config/fstab
Hier wird der Mountpoint für das entschlüsselte Gerät festgelegt. Wichtig ist hierbei der Verweis auf /dev/mapper/<name>.

```bash
config mount
option target '/mnt/secure_storage'
option device '/dev/mapper/backup_safe'
option enabled '1'
```

## 4) Installation
Kopiere das Skript auf deinen OpenWrt-Router, idealerweise nach /usr/bin/ oder /usr/sbin/:

```bash
vi /usr/bin/crypt-manage
```

Füge den Skriptinhalt ein und speichere die Datei.

Mache das Skript ausführbar:

```bash
chmod +x /usr/bin/crypt-manage
```

## 5) Beispiele für Aufrufe & Use Cases
### Verwendung
```bash
crypt-manage [ open | close ] [luks_name]* [go]
```
### Usecase 1: Der Sicherheits-Testlauf (Dry-Run)
Standardmäßig tut das Skript nichts, sondern zeigt dir nur, was es tun würde. Perfekt, um deine UCI-Konfiguration vorab risikofrei zu prüfen.

```bash
crypt-manage open
```
Ausgabe:

```bash
Entschlüssele: backup_safe
[Dry-Run] cryptsetup open /dev/sda1 backup_safe
[Dry-Run] mount /dev/mapper/backup_safe /mnt/secure_storage
```
### Usecase 2: Scharfes Öffnen und Einbinden
Füge das Wort go ans Ende an, um den Befehl tatsächlich auszuführen. Du wirst nun interaktiv nach deinem LUKS-Passwort gefragt.

```bash
crypt-manage open go
```
### Usecase 3: Gezieltes Öffnen einer bestimmten Platte
Hast du mehrere Platten in der UCI definiert, möchtest aber nur eine ganz bestimmte öffnen:

```bash
crypt-manage open backup_safe go
```
### Usecase 4: Sicheres Schließen (Mit fuser-Schutz)
Wenn du die Platte schließen möchtest, prüft das Skript im Hintergrund, ob noch Schreib- oder Lesezugriffe stattfinden:

```bash
crypt-manage close go
```
Falls noch ein Prozess (z.B. SSH oder ein Samba-Share) im Verzeichnis aktiv ist, bricht das Skript ab und listet die Übeltäter auf:

```bash
Sperre: backup_safe
✗ Dateien auf /mnt/secure_storage blockiert! Unmount abgebrochen.
PID USER       COMMAND
1234 root       /bin/sh

## 6) Lizenz
Dieses Projekt ist unter der MIT-Lizenz lizenziert – siehe unten für Details.

```bash
MIT License

Copyright (c) 2026 DEIN_NAME_ODER_NICKNAME

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

Mit KI-Unterstützung entwickelt und krisenfest geschliffen.
