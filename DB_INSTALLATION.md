**Anleitung zur Installation einer MySQL oder PostgreSQL Datenbank unter Ubuntu innerhalb von Multipass VMs**

---

Da ihr zwei Multipass VMs verwendet – eine für den Python-Code und eine für die Datenbank – ist es wichtig, die Verbindung zwischen diesen VMs korrekt einzurichten. Außerdem sollte beachtet werden, dass die Aktivierung von UFW (Uncomplicated Firewall) innerhalb der VMs zu Verbindungsproblemen führen kann, da Multipass die Netzwerkverbindungen verwaltet. Daher werden wir UFW in dieser Anleitung **nicht** aktivieren.

---

### **Voraussetzungen**

- **Multipass installiert** auf eurem Host-System.
- Zwei VMs innerhalb von Multipass:
  - **Python-VM**: Für die Ausführung des Python-Codes.
  - **Datenbank-VM**: Für die Installation von MySQL oder PostgreSQL.

---

### **Schritt 1: Erstellen der Multipass VMs**

#### **1.1 Python-VM erstellen**

```bash
multipass launch --name python-vm
```

#### **1.2 Datenbank-VM erstellen**

```bash
multipass launch --name db-vm
```

---

### **Schritt 2: IP-Adressen der VMs herausfinden**

Um die VMs miteinander kommunizieren zu lassen, benötigen wir ihre IP-Adressen.

#### **2.1 IP-Adresse der Datenbank-VM**

```bash
multipass list
```

Dies zeigt eine Liste aller VMs mit ihren IP-Adressen. Notiere dir die IP-Adresse der `db-vm`.

**Beispielausgabe:**

```
Name                    State             IPv4             Image
python-vm               Running           192.168.64.2     Ubuntu 24.04 LTS
db-vm                   Running           192.168.64.3     Ubuntu 24.04 LTS
```

---

### **Schritt 3: Installation der Datenbank auf der Datenbank-VM**

Wir installieren nun entweder **MySQL** oder **PostgreSQL** auf der `db-vm`.

#### **3.1 Verbindung zur Datenbank-VM herstellen**

```bash
multipass shell db-vm
```

#### **3.2 Systemaktualisierung**

```bash
sudo apt update
sudo apt upgrade -y
```

#### **Option A: Installation von MySQL**

##### **3.3A MySQL installieren**

```bash
sudo apt install mysql-server -y
```

##### **3.4A MySQL sichern**

```bash
sudo mysql_secure_installation
```

Während dieses Prozesses wirst du aufgefordert:

- Ein Root-Passwort festzulegen.
- Anonyme Benutzer zu entfernen.
- Root-Login remote zu deaktivieren.
- Testdatenbanken zu entfernen.
- Tabellenrechte neu zu laden.

**Hinweis:** Wähle sichere Einstellungen und merke dir dein Root-Passwort.

##### **3.5A Remote-Zugriff für MySQL einrichten**

- **MySQL-Konfiguration bearbeiten:**

  ```bash
  sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
  ```

- **Bind-Adresse anpassen:**

  Finde die Zeile:

  ```
  bind-address = 127.0.0.1
  ```

  Und ändere sie zu:

  ```
  bind-address = 0.0.0.0
  ```

  **Speichere** die Datei und **schließe** den Editor.

- **MySQL neu starten:**

  ```bash
sudo systemctl restart mysql
  ```

##### **3.6A Benutzer für Remote-Zugriff erstellen**

Melde dich an der MySQL-Konsole an:

```bash
sudo mysql
```

Erstelle einen neuen Benutzer und gewähre ihm die notwendigen Berechtigungen:

```sql
CREATE USER 'dein_benutzername'@'%' IDENTIFIED BY 'dein_passwort';
GRANT ALL PRIVILEGES ON *.* TO 'dein_benutzername'@'%';
FLUSH PRIVILEGES;
EXIT;
```

**Achtung:** Verwende ein sicheres Passwort und überlege, die Berechtigungen auf die benötigten Datenbanken einzuschränken.

#### **Option B: Installation von PostgreSQL**
  
##### **3.3B PostgreSQL installieren**

```bash
sudo apt install postgresql postgresql-contrib -y
```

##### **3.4B PostgreSQL für Remote-Zugriff konfigurieren**

- **PostgreSQL-Konfiguration bearbeiten:**

  ```bash
  sudo nano /etc/postgresql/14/main/postgresql.conf
  ```

  **Kleine Falle:** Überprüfe die tatsächlich installierte PostgreSQL-Version (z.B. 12, 13, 14) und passe den Pfad entsprechend an.

- **Listen-Adresse anpassen:**

  Finde die Zeile:

  ```
  #listen_addresses = 'localhost'
  ```

  Ändere sie zu:

  ```
  listen_addresses = '*'
  ```

- **Authentifizierungsdatei bearbeiten:**

  ```bash
  sudo nano /etc/postgresql/14/main/pg_hba.conf
  ```

  Füge folgende Zeile hinzu:

  ```
  host    all             all             0.0.0.0/0               md5
  ```

- **PostgreSQL neu starten:**

  ```bash
  sudo systemctl restart postgresql
  ```

##### **3.5B Benutzer und Datenbank erstellen**

Wechsle zum PostgreSQL-Benutzer:

```bash
sudo -i -u postgres
```

Erstelle einen neuen Benutzer:

```bash
createuser --interactive
```

Beantworte die Fragen:

- **Gib den Namen des neuen Benutzers ein:** `dein_benutzername`
- **Soll der neue Benutzer ein Superuser sein?** `nein`

Erstelle eine neue Datenbank:

```bash
createdb deine_datenbank
```

Setze ein Passwort für den Benutzer:

```bash
psql
```

In der psql-Konsole:

```sql
ALTER USER dein_benutzername WITH ENCRYPTED PASSWORD 'dein_passwort';
\q
```

Verlasse den PostgreSQL-Benutzer:

```bash
exit
```

---

### **Schritt 4: Firewall-Einstellungen**

Da Multipass die Netzwerkkonfiguration verwaltet und UFW nicht aktiviert ist, müssen wir uns keine Sorgen um die Firewall-Einstellungen innerhalb der VMs machen.

**Wichtig:** **Aktiviere UFW nicht** innerhalb der Multipass VMs, da dies die Netzwerkverbindung zwischen den VMs stören kann.

---

### **Schritt 5: Verbindung von der Python-VM zur Datenbank herstellen**

#### **5.1 Verbindung zur Python-VM herstellen**

```bash
multipass shell python-vm
```

#### **5.2 Systemaktualisierung**

```bash
sudo apt update
sudo apt upgrade -y
```

#### **5.3 Python-Umgebung einrichten**

Installiere pip und virtuelle Umgebungen, falls nötig:

```bash
sudo apt install python3-pip python3-venv -y
```

Erstelle eine virtuelle Umgebung:

```bash
python3 -m venv venv
source venv/bin/activate
```

#### **5.4 Notwendige Python-Pakete installieren**

- Für **MySQL**:

  ```bash
  pip install mysql-connector-python
  ```

- Für **PostgreSQL**:

  ```bash
  pip install psycopg2-binary
  ```

#### **5.5 Testskript erstellen**

Erstelle ein Python-Skript, um die Verbindung zur Datenbank zu testen.

- **Für MySQL (`test_mysql_connection.py`):**

  ```python
  import mysql.connector

  def create_connection():
      connection = mysql.connector.connect(
          host='IP_ADRESSE_DB_VM',
          user='dein_benutzername',
          password='dein_passwort',
          database='deine_datenbank'  # ggf. anpassen
      )
      return connection

  try:
      conn = create_connection()
      print("Verbindung zur MySQL-Datenbank erfolgreich")
      conn.close()
  except mysql.connector.Error as err:
      print(f"Fehler: {err}")
  ```

- **Für PostgreSQL (`test_postgres_connection.py`):**

  ```python
  import psycopg2

  def create_connection():
      connection = psycopg2.connect(
          host='IP_ADRESSE_DB_VM',
          user='dein_benutzername',
          password='dein_passwort',
          database='deine_datenbank'  # ggf. anpassen
      )
      return connection

  try:
      conn = create_connection()
      print("Verbindung zur PostgreSQL-Datenbank erfolgreich")
      conn.close()
  except psycopg2.Error as err:
      print(f"Fehler: {err}")
  ```

**Achte darauf,** `IP_ADRESSE_DB_VM`, `dein_benutzername`, `dein_passwort` und `deine_datenbank` mit den tatsächlichen Werten zu ersetzen.

#### **5.6 Skript ausführen**

Stelle sicher, dass du dich noch in der virtuellen Umgebung befindest, und führe das Skript aus:

- Für **MySQL**:

  ```bash
  python test_mysql_connection.py
  ```

- Für **PostgreSQL**:

  ```bash
  python test_postgres_connection.py
  ```

Wenn die Verbindung erfolgreich ist, solltest du die entsprechende Erfolgsmeldung erhalten.

---

### **Schritt 6: Fehlerbehebung**

Falls die Verbindung fehlschlägt:

- **Überprüfe die IP-Adressen:** Stelle sicher, dass du die korrekte IP-Adresse der Datenbank-VM verwendest.
- **Benutzer und Passwörter:** Verifiziere, dass Benutzername und Passwort korrekt sind.
- **Datenbank-Server läuft:** Prüfe, ob der Datenbank-Server auf der Datenbank-VM aktiv ist.
  - Für **MySQL**:

    ```bash
    sudo systemctl status mysql
    ```

  - Für **PostgreSQL**:

    ```bash
    sudo systemctl status postgresql
    ```

- **Netzwerkverbindung:** Von der Python-VM aus kannst du prüfen, ob die Datenbank-VM erreichbar ist:

  ```bash
  ping IP_ADRESSE_DB_VM
  ```

  **Kleine Falle:** Wenn der Ping nicht funktioniert, könnte das an der Netzwerkkonfiguration von Multipass liegen. Standardmäßig sollten die VMs jedoch miteinander kommunizieren können.

---

### **Hinweise zu Multipass und Netzwerk**

- **Netzwerkmodus:** Multipass verwendet standardmäßig ein NAT-Netzwerk, das es den VMs ermöglicht, miteinander zu kommunizieren.
- **Keine UFW-Aktivierung innerhalb der VMs:** Das Aktivieren von UFW (Firewall) innerhalb der VMs kann die Kommunikation zwischen ihnen blockieren. Daher sollte UFW in diesem Kontext **nicht** aktiviert werden.
- **Host-Firewall:** Wenn auf deinem Host-System eine Firewall läuft, stelle sicher, dass sie den Verkehr zwischen den Multipass VMs nicht blockiert.

---

### **Abschließende Hinweise**

- **Sicherheit:** Da diese Umgebung zu Lernzwecken dient und die VMs lokal auf deinem Rechner laufen, ist es in Ordnung, UFW nicht zu aktivieren und die Datenbank für Remote-Verbindungen innerhalb der VMs zu öffnen. In Produktionsumgebungen sollten jedoch strengere Sicherheitsmaßnahmen ergriffen werden.
- **Datenbankberechtigungen einschränken:** Es ist eine gute Praxis, die Berechtigungen des Datenbankbenutzers auf das Nötigste zu beschränken.
- **Passwörter sicher aufbewahren:** Speichere keine Passwörter im Klartext im Code. Verwende stattdessen Umgebungsvariablen oder Konfigurationsdateien, die nicht in Versionskontrollsysteme eingecheckt werden.
- **Gründlich lesen:** Achte darauf, jeden Schritt genau zu lesen und zu verstehen, bevor du ihn ausführst. Manchmal verstecken sich wichtige Details in den Anleitungen, die übersehen werden können.

---

**Viel Erfolg bei der Einrichtung!** Wenn du Fragen hast oder auf Probleme stößt, zögere nicht, nachzufragen. Es ist wichtig, dass du die Schritte verstehst und nicht einfach nur kopierst und einfügst.