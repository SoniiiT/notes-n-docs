> **Wichtige Hinweise**
> - Mach das nur in einem Wartungsfenster.
> - Erstelle vor dem Start **Backups** von:
>   - Nextcloud-Datenverzeichnis (z. B. `nextcloud/html/data` oder Volume)
>   - `config/config.php`
>   - Bestehender Datenbank (zusätzlicher Dump schadet nie)
> - Teste den Ablauf zuerst in einer **Testumgebung**, bevor du es in Produktion tust.

---

## 1. Vorbereitungen

### 1.1 In Projektverzeichnis wechseln

```bash prompt:bash
cd ~/docker-runner/nextcloud
```
Bei ITcares: /var/lib/docker/volumes/portainer_data/_data/compose/37
NFS Volume: 
mkdir -p /tmp/cleanup_mount
mount -t nfs msp-storage:NextCloud-Data/ /tmp/cleanup_mount
umount /tmp/cleanup_mount
rmdir /tmp/cleanup_mount
### 1.2 Nextcloud in den Wartungsmodus versetzen

```bash prompt:bash
docker exec -u www-data nextcloud-app php occ maintenance:mode --on
```

> `nextcloud-app` ist der Name deines Nextcloud-PHP/Apache-FPM-Containers. Falls er anders heißt, anpassen.

---

## 2. Aktuelle DB-Zugangsdaten ermitteln

Öffne `config/config.php` im Nextcloud-Projekt:

```bash prompt:bash
grep -E "db(host|name|user|password)" config/config.php
```

Du solltest ähnliche Zeilen sehen wie:

```php file:config.php
'dbhost' => 'db',
'dbname' => 'nextcloud',
'dbuser' => 'nextcloud',
'dbpassword' => 'PASSWORD',
```

Notiere dir:

- `DB_HOST`: `db`
- `DB_NAME`: `nextcloud`
- `DB_USER`: `nextcloud`
- `DB_PASSWORD`: `PASSWORD` (bzw. dein tatsächliches Passwort)

---

## 3. Docker-Netzwerk prüfen

Docker Compose erstellt standardmäßig ein Netzwerk nach dem Muster `<ordnername>_default`.

### 3.1 Verfügbare Netzwerke anzeigen

```bash prompt:bash
docker network ls
```

Suche nach einem Eintrag wie:

- `nextcloud_default`

Falls dein Projektverzeichnis anders heißt, kann das Netzwerk entsprechend abweichen.

In diesem Beispiel verwenden wir:

```bash prompt:bash
NETWORK=nextcloud_default
DB_HOST=db
DB_NAME=nextcloud
DB_USER=nextcloud
DB_PASSWORD='PASSWORD'
```

> Passe `DB_PASSWORD` an dein reales Passwort aus `config.php` an.

---

## 4. Dump aus MariaDB 12 erstellen (Docker-Variante)

Wir verwenden einen separaten **mariadb:10.6** Client-Container, der sich ins gleiche Docker-Netzwerk hängt und `mysqldump` ausführt.  
Der Dump wird als Datei **auf dem Host** gespeichert.
### 4.1 Dump erstellen

```bash prompt:bash
docker run --rm --network "$NETWORK" mariadb:10.11 sh -c "mysqldump -h \"$DB_HOST\" -u \"$DB_USER\" -p\"$DB_PASSWORD\" --databases \"$DB_NAME\" --default-character-set=utf8mb4 --add-drop-table --no-tablespaces --skip-lock-tables" > nextcloud-db-from-12.sql
```

**Erläuterungen zu den Parametern:**

- `--databases "$DB_NAME"`  
  Exportiert das komplette Schema inkl. `CREATE DATABASE`.
- `--default-character-set=utf8mb4`  
  Empfohlener Zeichensatz für Nextcloud.
- `--add-drop-table`  
  Fügt `DROP TABLE` vor `CREATE TABLE` ein (wichtig für Restore).
- `--no-tablespaces`  
  Vermeidet Probleme mit neueren MariaDB/MySQL-Versionen.
- `--skip-lock-tables`  
  Verhindert Sperren, nützlich in produktiven Setups (leicht weniger konsistent, aber meist OK).

Am Ende solltest du eine Datei `nextcloud-db-from-12.sql` im Projektordner haben.

---

## 5. Neue MariaDB-10.6-Instanz starten

Wir starten jetzt eine **separate MariaDB 10.6** Instanz in einem neuen Container.

### 5.1 Container starten

```bash prompt:bash
docker run -d --name nextcloud-db-1011 -e MARIADB_ROOT_PASSWORD=RootPass123! -e MARIADB_DATABASE="$DB_NAME" -e MARIADB_USER="$DB_USER" -e MARIADB_PASSWORD="$DB_PASSWORD" -v nextcloud-db-106-data:/var/lib/mysql mariadb:10.11
```

**Wichtige Punkte:**

- `--name nextcloud-db-106`  
  Containername, später in `config.php` als Host verwendet.
- `MARIADB_ROOT_PASSWORD`  
  Starkes Root-Passwort setzen (Beispielwert unbedingt anpassen).
- `MARIADB_DATABASE`, `MARIADB_USER`, `MARIADB_PASSWORD`  
  Müssen zu deinen Nextcloud-Einstellungen passen (`dbname`, `dbuser`, `dbpassword`).
- `nextcloud-db-106-data`  
  Docker-Volume, in dem die neue Datenbank gespeichert wird.

### 5.2 Version prüfen

```bash prompt:bash
docker exec -it nextcloud-db-106 mysql -u root -p -e "SELECT VERSION();"
```

Passwort ist `MARIADB_ROOT_PASSWORD` (hier: `RootPass123!`).  
Die Ausgabe sollte eine **10.6.x** Version zeigen.

---

## 6. Dump in MariaDB 10.6 importieren

### 6.1 Dump in den neuen Container kopieren

```bash prompt:bash
docker cp nextcloud-db-from-12.sql nextcloud-db-106:/tmp/nextcloud-db-from-12.sql
```

### 6.2 Import als root durchführen

```bash prompt:bash
docker exec -it nextcloud-db-106 sh -c 'mysql -u root -p"$MARIADB_ROOT_PASSWORD" < /tmp/nextcloud-db-from-12.sql'
```

> Je nach Datenbankgröße kann der Import einige Zeit dauern.

---

## 7. Nextcloud auf MariaDB 10.6 umstellen

Jetzt zeigen wir Nextcloud auf die neue Datenbank-Instanz.

### 7.1 `config.php` bearbeiten

Auf dem Host im Nextcloud-Projektordner:

```bash prompt:bash
nano config/config.php
```

Passe folgende Werte an (Beispiel):

```php file:config.php
'dbtype' => 'mysql',
'dbname' => 'nextcloud',                 // dein DB_NAME
'dbhost' => 'nextcloud-db-106',          // neuer Containername
'dbport' =Y '3306'                       // Datenbank Port
'dbuser' => 'nextcloud',                 // dein DB_USER
'dbpassword' => 'SEHR_GEHEIM',           // dein DB_PASSWORD
```

**Wichtig:**

- `dbhost` muss **den Namen des neuen Containers** + Port enthalten (`nextcloud-db-106`), damit der `nextcloud-app` Container ihn im Docker-Netzwerk erreichen kann.
- `dbpassword` auf den korrekten Wert setzen (kann identisch zum alten sein, muss aber zur neuen DB-Instanz passen).

Speichern und schließen.

---

## 8. Nextcloud Maintenance / Repair / Upgrade

Falls der Wartungsmodus noch nicht aktiv ist, erneut aktivieren (zur Sicherheit):

```bash prompt:bash
docker exec -u www-data nextcloud-app php occ maintenance:mode --on
```

### 8.1 Repair laufen lassen

```bash prompt:bash
docker exec -u www-data nextcloud-app php occ maintenance:repair
```

### 8.2 Upgrade ausführen

```bash prompt:bash
docker exec -u www-data nextcloud-app php occ upgrade
```

> Hier werden ggf. DB-Migrationen ausgeführt. Prüfe die Ausgabe sorgfältig auf Fehler.

### 8.3 Wartungsmodus deaktivieren

Wenn alles ohne Fehler durchläuft:

```bash prompt:bash
docker exec -u www-data nextcloud-app php occ maintenance:mode --off
```

---

## 9. Funktionstest

### 9.1 Im Browser testen

- Aufruf der Nextcloud-Weboberfläche
- Login mit einem normalen Benutzer
- Folgendes prüfen:
  - Datei hochladen
  - Datei herunterladen
  - Datei löschen
  - Evtl. Kalender/Kontakte stichprobenartig testen

### 9.2 Logs prüfen

```bash prompt:bash
docker logs nextcloud-db-106 --tail=100
docker logs nextcloud-app --tail=100
```

Achte auf:

- Keine SQL-Fehler
- Keine DB-Verbindungsfehler
- Keine auffälligen PHP-Fatal-Errors

Wenn alles sauber läuft und Nextcloud normal arbeitet, ist der Downgrade-Pfad von MariaDB 12 → 10.6 erfolgreich validiert.

---

## 10. Aufräumen & Umstellung in Produktion

### 10.1 Empfohlener Ablauf für produktive Instanzen (Kurzform)

Für **jede** produktive Instanz:

1. **Wartungsfenster planen**
2. **Wartungsmodus aktivieren**
   ```bash prompt:bash
   docker exec -u www-data nextcloud-app php occ maintenance:mode --on
   ```
3. **Dump aus MariaDB 12** mit Client-Container erstellen  
   (wie in Abschnitt 4)
4. **Neue MariaDB-10.6-Instanz starten**  
   (wie in Abschnitt 5)
5. **Dump in MariaDB 10.6 importieren**  
   (wie in Abschnitt 6)
6. **Nextcloud `config.php` auf neuen DB-Container umstellen**  
   (Abschnitt 7)
7. **occ-Wartungsschritte ausführen**
   ```bash prompt:bash
   docker exec -u www-data nextcloud-app php occ maintenance:repair
   docker exec -u www-data nextcloud-app php occ upgrade
   ```
8. **Wartungsmodus deaktivieren**
   ```bash prompt:bash
   docker exec -u www-data nextcloud-app php occ maintenance:mode --off
   ```
9. **Funktionstests & Log-Check**  
   (wie in Abschnitt 9)
10. **Alte MariaDB-12-Instanz erst nach erfolgreichen Tests abschalten/entfernen**
    - Container stoppen
    - Daten-Volume erst nach zusätzlicher Backup-Kontrolle löschen