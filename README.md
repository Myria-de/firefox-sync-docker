# firefox-sync-docker
Einen eigenen Firefox-Sync-Server einrichten

# Docker unter Ubuntu/Linux Mint installieren
Wir beschreiben die Installation für Ubuntu 22.04 und Linux Mint 21.

Zur Installation öffnen Sie ein Terminalfenster (Strg-Alt-T) und führen die folgenden vier Befehlszeilen aus:
```
sudo apt-get update
sudo apt-get install curl dbus-user-session uidmap
curl -fsSL https://get.docker.com/rootless | sh
sudo loginctl enable-linger $(whoami)
```
Öffnen Sie über den Dateimanager die versteckte Datei ".bashrc" (einblenden mit Strg-H) in einem Editor. Fügen Sie am Ende die Zeile
```
export DOCKER_HOST=unix:///run/user/1000/docker.sock
```
an. Speichern und schließen Sie die Datei.

Die ausführbaren Dateien werden im Home-Verzeichnis im Ordner "bin" installiert. Damit dieser sich im Suchpfad befindet, muss die Profilkonfiguration neu eingelesen werden:
```
source ~/.profile
```
Die Datei „.bashrc“ wird damit ebenfalls neu eingelesen.

Damit Docker ohne root-Privilegien Netzwerkports unterhalb von 1024 verwenden kann, muss die Konfiguration mit diesen zwei Zeilen angepasst werden:
```
sudo setcap cap_net_bind_service=ep $(which rootlesskit)
systemctl --user restart docker
```
**Hinweis:** Wer Docker traditionell mit root-Rechten verwenden will, entfernt bei der Installation in der curl-Zeile "rootless".

# Docker über den Webbrowser verwalten
Docker wird standardmäßig im Terminal konfiguriert und verwaltet. Über die grafische Oberfläche Portainer (www.portainer.io) gelingt das deutlich komfortabler. Zur Installation verwenden Sie im Terminal diese Befehlszeile: 
```
docker run -d -p 8000:8000 -p 9443:9443 --name=portainer --restart=always -v /$XDG_RUNTIME_DIR/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```
Anschließend rufen Sie die URL https://localhost:9443 im Webbrowser auf. 

# Den Sync-Server über Portainer einrichten
Laden Sie die Dateien "docker-compose.yaml" und "fxsync.env" herunter.

**Schritt 1:** In der Portainer-Oberfläche klicken Sie im Menü auf der linken Seite auf "Stacks" und dann auf "+Add Stack". Tippen Sie hinter "Name" ein aussagekräftige Bezeichnung ein, beispielsweise fsync-server.

**Schritt 2:** Aktivieren Sie die Option "Upload", klicken Sie auf "Select file" und wählen Sie die Datei "docker-compose.yaml". Klicken Sie auf "Load variables from .env file" und geben Sie die Datei "fxsync.env" an.

**Schritt 3:** Unterhalb von „Environment variables“ tragen Sie hinter „SYNC_MASTER_SECRET“ und „METRICS_HASH_SECRET“ jeweils eine beliebige, zufällige Zeichenfolge ein, die Sie im Terminal mit
```
head -c 20 /dev/urandom | sha1sum | awk '{print $1}'
```
erzeugen.

Hinter "Domain" gehört der Name des Rechners, über den er im lokalen Netzwerk erreichbar ist. Wie der Name genau lautet, hängt vom Router ab. Bei einer Fritzbox verwenden Sie "http://[Rechnername].fritz.box:5000" bei anderen Modellen beispielsweise "http://[Rechnername].local:5000". Probieren Sie im Terminal mit
```
ping [Domain]
```
aus, auf welche Bezeichnung der PC antwortet.

Zum Abschluss klicken Sie auf "Deploy the stack". Mit der angegebenen Konfiguration lädt Portainer das Mozilla-Docker-Image von https://hub.docker.com/r/mozilla/syncstorage-rs herunter. Wir verwenden die nicht ganz aktuelle Version 0.13.6, weil sich neuere Versionen bei unseren Tests nicht einrichten ließen. Zusätzlich installiert Portainer das Datenbanksystem Mariadb, über das der Sync-Server die Daten speichert.

Rufen Sie im Browser die URL
```
http://[Domain]:5000/__heartbeat__
```
auf. Sie erhalten vom Server eine Antwort im Json-Format, die mit dem Status "OK", die korrekte Installation bestätigt.

# Firefox für den neuen Sync-Server konfigurieren
Rufen Sie in Firefox die URL "about:config" auf. Klicken Sie auf „Risiko akzeptieren und fortfahren“ und geben Sie als Suchbegriff identity.sync.tokenserver.uri ein. Ändern Sie die URL auf 
```
http://[Domain]:5000/1.0/sync/1.5
```

Setzen Sie außerdem "services.sync.log.appender.file.logOnSuccess" und "services.sync.log.appender.file.logOnError" jeweils auf "true". Starten Sie Firefox neu. Gehen Sie im Menü auf "Extras -> Jetzt synchronisieren". Das Menü lässt sich unter Ubuntu (Gnome) mit der Alt-Taste einblenden.

Öffnen Sie die URL "about:sync-log". Die letzten Einträge sollten mit "sync-success" beginnen, Fehler werden in den Protokollen "error-sync" angezeigt, die sich per Mausklick öffnen lassen. Bei Problemen prüfen Sie, ob die enthaltenen URLs tatsächlich auf Ihren Server verweisen. Wenn nicht, korrigieren Sie den Wert für "identity.sync.tokenserver.uri". Sollte bei "sync-success" die Mozilla-Adresse "https://token.services.mozilla.com/1.0/sync/1.5" zu sehen sein, starten Sie Firefox neu. Es kann auch helfen, sich vom Mozilla-Konto ab- und wieder anzumelden.


