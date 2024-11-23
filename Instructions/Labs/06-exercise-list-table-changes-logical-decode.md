---
lab:
  title: Auflisten von Tabellenänderungen mit logischer Dekodierung
  module: Understand write-ahead logging
---

# Auflisten von Tabellenänderungen mit logischer Dekodierung

In dieser Übung konfigurieren Sie die logische Replikation, die nativ in PostgreSQL enthalten ist. Sie erstellen zwei Server, die als Verleger und Abonnent fungieren. Daten in der Datenbank zoodb werden zwischen ihnen repliziert.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Wenn Sie nicht über ein Azure-Abonnement verfügen, können Sie ein [kostenloses Azure-Testkonto](https://azure.microsoft.com/free) erstellen.

## Ressourcengruppe erstellen

1. Melden Sie sich beim Azure-Portal an. Ihr Benutzerkonto muss die Rolle „Besitzer“ oder „Mitwirkender“ im Azure-Abonnement haben.
1. Wählen Sie **Ressourcengruppen** aus, und klicken Sie dann auf **+ Erstellen**.
1. Wählen Sie Ihr Abonnement aus.
1. Geben Sie in Ressourcengruppe **rg-PostgreSQL_Replication** ein.
1. Wählen Sie eine Region in der Nähe Ihres Standorts aus.
1. Klicken Sie auf **Überprüfen + erstellen**.
1. Klicken Sie auf **Erstellen**.

## Erstellen eines Verlegerservers

1. Wählen Sie unter „Azure-Dienste“ die Option **+ Ressource erstellen** aus. Wählen Sie unter **Kategorien** die Option **Datenbanken** aus. Wählen Sie unter **Azure Database for PostgreSQL** die Option **Erstellen** aus.
1. Füllen Sie auf der Registerkarte **Allgemeine Informationen** für „Flexibler Server“ die Felder wie folgt aus:
    1. Abonnement: Ihr Abonnement
    1. Ressourcengruppe: Wählen Sie **rg-PostgreSQL_Replikation**.
    1. Servername: **psql-postgresql-pub9999** (Der Name muss global eindeutig sein, ersetzen Sie also 9999 durch vier zufällige Zahlen).
    1. Region: Wählen Sie dieselbe Region wie für die Ressourcengruppe aus.
    1. PostgreSQL-Version: Wählen Sie 16.
    1. Workloadtyp: **Entwicklung**.
    1. Compute + Speicher: **Burstfähig**. Wählen Sie **Server konfigurieren** aus, und überprüfen Sie die Konfigurationsoptionen. Nehmen Sie keine Änderungen vor, und schließen Sie den Abschnitt.
    1. Verfügbarkeitszone: -1. Wenn Verfügbarkeitszonen nicht unterstützt werden, übernehmen Sie „Keine Präferenz“.
    1. Hohe Verfügbarkeit: Deaktiviert.
    1. Authentifizierungsmethode: Nur PostgreSQL-Authentifizierung.
    1. Geben Sie in **Admin-Benutzername** **`demo`** ein.
    1. Geben Sie unter **Kennwort** ein geeignetes komplexes Kennwort ein.
    1. Klicken Sie auf **Weiter: Netzwerk >**.
1. Füllen Sie auf der Registerkarte **Netzwerk** für „Flexibler Server“ jedes Feld wie folgt aus:
    1. Konnektivitätsmethode: (o) Öffentlicher Zugriff (zulässige IP-Adressen)
    1. **Öffentlichen Zugriff auf diesen Server über beliebigen Azure-Dienst in Azure gestatten**: aktiviert. Diese Option muss aktiviert werden, damit die Verleger- und Abonnentendatenbanken miteinander kommunizieren können.
    1. Wählen Sie unter „Firewallregeln“ die Option **+ Aktuelle Client-IP-Adresse hinzufügen** aus. Dadurch wird Ihre aktuelle IP-Adresse als Firewallregel hinzugefügt. Sie können dieser Firewallregel optional einen aussagekräftigen Namen geben.
1. Klicken Sie auf **Überprüfen + erstellen**. Klicken Sie anschließend auf **Erstellen**.
1. Da die Erstellung einer Azure-Datenbank für PostgreSQL nur wenige Minuten dauern kann, beginnen Sie mit dem nächsten Schritt, sobald diese Bereitstellung erfolgt ist. Denken Sie daran, ein neues Browserfenster oder eine neue Registerkarte zu öffnen, um fortzufahren.

## Erstellen eines Abonnentenservers

1. Wählen Sie unter „Azure-Dienste“ die Option **+ Ressource erstellen** aus. Wählen Sie unter **Kategorien** die Option **Datenbanken** aus. Wählen Sie unter **Azure Database for PostgreSQL** die Option **Erstellen** aus.
1. Füllen Sie auf der Registerkarte **Allgemeine Informationen** für „Flexibler Server“ die Felder wie folgt aus:
    1. Abonnement: Ihr Abonnement
    1. Ressourcengruppe: Wählen Sie **rg-PostgreSQL_Replikation**.
    1. Servername: **psql-postgresql-sub9999** (Der Name muss global eindeutig sein, ersetzen Sie also 9999 durch vier zufällige Zahlen).
    1. Region: Wählen Sie dieselbe Region wie für die Ressourcengruppe aus.
    1. PostgreSQL-Version: Wählen Sie 16.
    1. Workloadtyp: **Entwicklung**.
    1. Compute + Speicher: **Burstfähig**. Wählen Sie **Server konfigurieren** aus, und überprüfen Sie die Konfigurationsoptionen. Nehmen Sie keine Änderungen vor, und schließen Sie den Abschnitt.
    1. Verfügbarkeitszone: -2. Wenn Verfügbarkeitszonen nicht unterstützt werden, übernehmen Sie „Keine Präferenz“.
    1. Hohe Verfügbarkeit: Deaktiviert.
    1. Authentifizierungsmethode: Nur PostgreSQL-Authentifizierung.
    1. Geben Sie in **Admin-Benutzername** **`demo`** ein.
    1. Geben Sie unter **Kennwort** ein geeignetes komplexes Kennwort ein.
    1. Klicken Sie auf **Weiter: Netzwerk >**.
1. Füllen Sie auf der Registerkarte **Netzwerk** für „Flexibler Server“ jedes Feld wie folgt aus:
    1. Konnektivitätsmethode: (o) Öffentlicher Zugriff (zulässige IP-Adressen)
    1. **Öffentlichen Zugriff auf diesen Server über beliebigen Azure-Dienst in Azure gestatten**: aktiviert. Diese Option muss aktiviert werden, damit die Verleger- und Abonnentendatenbanken miteinander kommunizieren können.
    1. Wählen Sie unter „Firewallregeln“ die Option **+ Aktuelle Client-IP-Adresse hinzufügen** aus. Dadurch wird Ihre aktuelle IP-Adresse als Firewallregel hinzugefügt. Sie können dieser Firewallregel optional einen aussagekräftigen Namen geben.
1. Klicken Sie auf **Überprüfen + erstellen**. Klicken Sie anschließend auf **Erstellen**.
1. Warten Sie, bis beide Azure Database for PostgreSQL-Server bereitgestellt sind.

## Einrichten der Replikation

Für *beide*, den Verleger- und den Abonnentenserver:

1. Navigieren Sie im Azure-Portal zu Ihrem Server, und wählen Sie **Serverparameter** unter „Einstellungen“ aus.
1. Suchen Sie mithilfe der Suchleiste nach den einzelnen Parametern, und nehmen Sie die folgenden Änderungen vor:
    1. `wal_level` = LOGICAL
    1. `max_worker_processes`= 24
1. Wählen Sie **Speichern**. Wählen Sie dann **Speichern und neu starten** aus.
1. Warten Sie, bis beide Server neu gestartet sind.

    > Hinweis
    >
    > Nachdem die Server neu bereitgestellt wurden, müssen Sie möglicherweise Ihre Browserfenster aktualisieren, um festzustellen, dass die Server neu gestartet wurden.

## Ehe Sie fortfahren

Stellen Sie sicher, dass Folgendes zutrifft:

1. Sie haben beide Azure Database for PostgreSQL – Flexibler Server installiert und gestartet.
1. Sie haben bereits die Lab-Skripte aus [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git) geklont. Falls Sie dies noch nicht getan haben, klonen Sie das Repository lokal:
    1. Öffnen Sie eine Befehlszeile/ein Terminal.
    1. Führen Sie den folgenden Befehl aus:
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > HINWEIS
       > 
       > Wenn **git** nicht installiert ist, [laden Sie die App ***git*** herunter und installieren Sie sie](https://git-scm.com/download) und versuchen Sie, die vorherigen Befehle erneut auszuführen.
1. Sie haben Azure Data Studio installiert. Wenn nicht, dann [laden Sie ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284) herunter und installieren Sie es.
1. Installieren Sie die Erweiterung **PostgreSQL** in Azure Data Studio.

## Einrichten des Verlegers

1. Öffnen Sie Azure Data Studio, und stellen Sie eine Verbindung mit dem Verlegerserver her. (Kopieren Sie den Servernamen aus dem Abschnitt „Übersicht“.)
1. Wählen Sie **Datei**, **Datei öffnen** und navigieren Sie zu dem Ordner, in dem Sie die Skripte gespeichert haben.
1. Öffnen Sie das Skript **../Allfiles/Labs/06/Lab6_Replication.sql** und verbinden Sie sich mit dem Server.
1. Markieren und führen Sie den Abschnitt **Grant the admin user replication permission** aus.
1. Markieren und führen Sie den Abschnitt **Create zoodb database** aus.
1. Wählen Sie über die Dropdownliste auf der Symbolleiste zoodb als aktuelle Datenbank aus. Stellen Sie sicher, dass zoodb die aktuelle Datenbank ist, indem Sie die **SELECT**-Anweisung ausführen.
1. Markieren Sie den Abschnitt **Tabellen erstellen** und **Fremdschlüssel-Beschränkungen** in zoodb und führen Sie ihn aus.
1. Markieren und führen Sie den Abschnitt **Populate the tables in zoodb** aus.
1. Markieren und führen Sie den Abschnitt **Create a publication** aus. Wenn Sie die SELECT-Anweisung ausführen, wird nichts aufgelistet, da die Replikation noch nicht aktiv ist.

## Einrichten des Abonnenten

1. Öffnen Sie eine zweite Instanz von Azure Data Studio und verbinden Sie sich mit dem Abonnentenserver.
1. Wählen Sie **Datei**, **Datei öffnen** und navigieren Sie zu dem Ordner, in dem Sie die Skripte gespeichert haben.
1. Öffnen Sie das Skript **../Allfiles/Labs/06/Lab6_Replication.sql** und verbinden Sie sich mit dem Abonnentenserver. (Kopieren Sie den Servernamen aus dem Abschnitt „Übersicht“.)
1. Markieren und führen Sie den Abschnitt **Grant the admin user replication permission** aus.
1. Markieren und führen Sie den Abschnitt **Create zoodb database** aus.
1. Wählen Sie über die Dropdownliste auf der Symbolleiste zoodb als aktuelle Datenbank aus. Stellen Sie sicher, dass zoodb die aktuelle Datenbank ist, indem Sie die **SELECT**-Anweisung ausführen.
1. Markieren Sie den Abschnitt **Tabellen erstellen** und **Fremdschlüssel-Beschränkungen** in zoodb und führen Sie ihn aus.
1. Scrollen Sie nach unten zum Abschnitt **Create a subscription**.
    1. Bearbeiten Sie die Anweisung **CREATE SUBSCRIPTION** so, dass sie den korrekten Namen des Verlegerservers und das sichere Kennwort des Verlegers enthält. Markieren und führen Sie die Anweisung aus.
    1. Markieren und führen Sie die **SELECT**-Anweisung aus. Diese zeigt das Abonnement „sub“, das Sie erstellt haben.
1. Markieren Sie unter dem Abschnitt **Display the tables**, und führen Sie alle **SELECT**-Anweisungen aus. Die Tabellen wurden per Replikation vom Verleger aufgefüllt.

## Vornehmen von Änderungen an der Verlegerdatenbank

- Markieren Sie in der ersten Instanz von Azure Data Studio (*Ihre Verleger-Instanz*) unter **Weitere Tiere einfügen** die Anweisung **INSERT** und führen Sie sie aus. *Stellen Sie sicher, dass Sie **diese INSERT-Anweisung nicht** beim Abonnenten ausführen*.

## Anzeigen der Änderungen in der Abonnentendatenbank

- Markieren Sie in der zweiten Instanz von Azure Data Studio (Abonnent) unter **Display the animal tables** die **SELECT**-Anweisung, und führen Sie sie aus.

Sie haben nun zwei Instanzen von Azure Database for PostgreSQL – Flexibler Server erstellt und einen als Verleger und den anderen als Abonnenten konfiguriert. In der Verlegerdatenbank haben Sie die Datenbank „zoo“ erstellt und aufgefüllt. In der Abonnentendatenbank haben Sie eine leere Datenbank erstellt, die dann per Streamingreplikation aufgefüllt wurde.

## Bereinigung

1. Nachdem Sie die Übung beendet haben, löschen Sie die Ressourcengruppe, die beide Server enthält. Die Server werden Ihnen in Rechnung gestellt, wenn Sie nicht entweder die Anweisung STOP oder DELETE ausführen.
1. Löschen Sie bei Bedarf den Ordner .\DP3021Lab.
