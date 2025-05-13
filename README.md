Absolut, Ben! Hier ist der Entwurf für das README zur "Lookup"-Maschine.

# Lookup - HackMyVM (Medium)

![Lookup.png](Lookup.png)

## Übersicht

*   **VM:** Lookup
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Lookup)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 27. Januar 2025
*   **Original-Writeup:** https://alientec1908.github.io/Lookup_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Lookup" zu erlangen. Der Weg begann mit dem Brute-Forcen von Login-Credentials für eine Webanwendung. Nach dem Login wurde eine Subdomain entdeckt, die eine verwundbare elFinder-Instanz hostete. Durch Ausnutzung einer bekannten Remote Code Execution (RCE)-Schwachstelle in elFinder wurde initialer Zugriff als Benutzer `www-data` erlangt. Die erste Rechteausweitung zum Benutzer `think` gelang durch Path Hijacking einer SUID-Binary (`/usr/sbin/pwm`), die den `id`-Befehl unsicher aufrief und so das Auslesen einer Passwortdatei ermöglichte. Die finale Eskalation zu Root erfolgte durch Ausnutzung einer unsicheren `sudo`-Regel, die dem Benutzer `think` erlaubte, `/usr/bin/look` als Root auszuführen, was zum Lesen des privaten SSH-Schlüssels des Root-Benutzers missbraucht wurde.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `ping6`
*   `ip neigh`
*   `grep`
*   `awk`
*   `sort`
*   `nmap`
*   `vi` / `nano`
*   `curl`
*   `nikto`
*   `gobuster`
*   `wapiti`
*   `ffuf`
*   `searchsploit`
*   `python2` (für elFinder Exploit)
*   `python3` (für Reverse Shell)
*   `nc` (netcat)
*   `stty`
*   `find`
*   `file`
*   `strings`
*   `mktemp`
*   `echo`
*   `chmod`
*   `export`
*   `ssh`
*   `sudo`
*   `passwd`
*   `look`
*   `ssh2john`
*   Standard Linux-Befehle (`id`, `ls`, `cat`, `pwd`, `bash`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Lookup" gliederte sich in folgende Phasen:

1.  **Reconnaissance (IPv4 & IPv6):**
    *   IPv4-Adresse des Ziels (192.168.2.164) mit `arp-scan` identifiziert.
    *   Hostname `lookup.hmv` in `/etc/hosts` eingetragen.
    *   IPv6-Nachbarerkennung fand die Link-Local-Adresse (`fe80::a00:27ff:fe5a:36a7`).
    *   `nmap`-Scans (TCP und UDP) bestätigten offene Ports 22/tcp (OpenSSH 8.2p1) und 80/tcp (Apache 2.4.41) auf IPv4 und IPv6. Die Webseite auf Port 80 zeigte "Login Page" und leitete auf `http://lookup.hmv` weiter.

2.  **Web Enumeration & Credential Access (Brute-Force):**
    *   `nikto` und `gobuster` auf `http://lookup.hmv/` zeigten keine direkten kritischen Schwachstellen, aber eine Login-Seite (`login.php`).
    *   Mittels `ffuf` wurde ein Brute-Force-Angriff auf `login.php` durchgeführt. Zuerst wurde das Passwort `password123` für den Benutzer `admin` gefunden. Anschließend wurde mit diesem Passwort weiter nach Benutzernamen gesucht, was zum Benutzer `jose` führte (Passwort ebenfalls `password123`).

3.  **Subdomain Enumeration & elFinder Exploit (Initial Access als `www-data`):**
    *   Nach dem (impliziten) Login als `jose` oder durch weitere Enumeration wurde die Subdomain `files.lookup.hmv` entdeckt und in `/etc/hosts` eingetragen.
    *   Auf `http://files.lookup.hmv/` wurde eine elFinder-Instanz unter `/elFinder/elfinder.html` gefunden.
    *   `searchsploit elFinder` fand den Exploit `php/webapps/46481.py` (elFinder < 2.1.48 Command Injection).
    *   Der Exploit wurde mit `python2 46481.py "http://files.lookup.hmv/elFinder/"` ausgeführt, was zu einer interaktiven Shell führte.
    *   Eine stabilere Reverse Shell wurde mittels Python zu einem Netcat-Listener als Benutzer `www-data` aufgebaut.

4.  **Privilege Escalation (von `www-data` zu `think` via pwm Path Hijacking):**
    *   Als `www-data` wurde bei der Suche nach SUID-Dateien `/usr/sbin/pwm` (SUID/SGID Root) entdeckt.
    *   `strings /usr/sbin/pwm` zeigte, dass das Programm den `id`-Befehl ohne vollständigen Pfad aufruft und dann versucht, `/home/%s/.passwords` zu lesen (wobei `%s` der Benutzername aus der `id`-Ausgabe ist).
    *   Ein bösartiges `id`-Skript wurde in `/tmp` erstellt (`echo 'echo "uid=1000(think)..."' > /tmp/id`), das den Benutzer `think` ausgibt.
    *   Die `PATH`-Umgebungsvariable wurde manipuliert (`export PATH=/tmp:$PATH`), sodass `/tmp/id` vor `/usr/bin/id` gefunden wird.
    *   Ausführen von `/usr/sbin/pwm` führte dazu, dass das manipulierte `id`-Skript ausgeführt wurde, `pwm` versuchte, `/home/think/.passwords` zu lesen (als Root), und dessen Inhalt ausgab.
    *   In `/home/think/.passwords` wurde das Passwort `thepassword` gefunden.

5.  **Lateral Movement (SSH als `think`) & Privilege Escalation (von `think` zu `root` via `sudo look`):**
    *   Erfolgreicher SSH-Login als `think` mit dem Passwort `thepassword`.
    *   `sudo -l` als `think` zeigte die Regel `(ALL) /usr/bin/look`. `think` durfte `look` als jeder Benutzer (einschließlich Root) ausführen.
    *   Mittels `LFILE=/root/.ssh/id_rsa; sudo look '' "$LFILE"` wurde der private SSH-Schlüssel des Root-Benutzers ausgelesen (GTFOBins-Technik).
    *   Der Schlüssel war nicht passwortgeschützt.
    *   Erfolgreicher SSH-Login als `root` mit dem extrahierten privaten Schlüssel.
    *   Die User-Flag (`38375fb4dd8baa2b2039ac03d92b820e`) und Root-Flag (`5a285a9f257e45c68bb6c9f9f57d18e8`) wurden gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Schwache Web-Login-Credentials:** Das Passwort `password123` wurde für mehrere Benutzer verwendet und konnte durch Brute-Force erraten werden.
*   **elFinder Remote Code Execution (RCE):** Eine bekannte Schwachstelle in einer älteren elFinder-Version ermöglichte die Ausführung von Code auf dem Server.
*   **Path Hijacking (SUID Binary):** Eine SUID-Root-Binary (`/usr/sbin/pwm`) rief den `id`-Befehl ohne Angabe des vollständigen Pfades auf. Durch Manipulation der `PATH`-Umgebungsvariable konnte ein bösartiges `id`-Skript ausgeführt werden, was zum Auslesen einer Passwortdatei führte.
*   **Unsichere `sudo`-Regel (look):** Ein Benutzer durfte `/usr/bin/look` als Root ausführen. Dies wurde missbraucht, um beliebige Dateien zu lesen (hier: den privaten SSH-Schlüssel von Root).
*   **Klartextpasswörter in Datei:** Der Benutzer `think` speicherte Passwörter im Klartext in `~/.passwords`.
*   **Virtuelle Host Enumeration:** Entdeckung zusätzlicher Angriffsflächen durch Identifizierung von Subdomains.

## Flags

*   **User Flag (`/home/think/user.txt`):** `38375fb4dd8baa2b2039ac03d92b820e`
*   **Root Flag (`/root/root.txt`):** `5a285a9f257e45c68bb6c9f9f57d18e8`

## Tags

`HackMyVM`, `Lookup`, `Medium`, `Web Brute-Force`, `elFinder RCE`, `Path Hijacking`, `SUID Exploit`, `sudo Exploit`, `look`, `SSH`, `Linux`, `Privilege Escalation`, `Virtual Host Enumeration`
