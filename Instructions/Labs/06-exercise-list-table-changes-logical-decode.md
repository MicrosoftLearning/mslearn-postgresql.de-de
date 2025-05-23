---
lab:
  title: Auflisten von Tabellenänderungen mit logischer Dekodierung
  module: Understand write-ahead logging
---

# Auflisten von Tabellenänderungen mit logischer Dekodierung

In dieser Übung konfigurieren Sie die logische Replikation, die nativ in PostgreSQL enthalten ist. Sie erstellen zwei Server, die als Herausgeber und Abonnent fungieren. Daten in der Datenbank „zoodb“ werden zwischen ihnen repliziert.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Wenn Sie nicht über ein Azure-Abonnement verfügen, können Sie ein [kostenloses Azure-Testkonto](https://azure.microsoft.com/free) erstellen.

Darüber hinaus müssen Sie Folgendes auf Ihrem Computer installiert haben:

- Visual Studio Code.
- Postgres Visual Studio Code-Erweiterung von Microsoft.
- Azure-Befehlszeilenschnittstelle.
- Git.

## Erstellen der Übungsumgebung

In diesen und späteren Übungen verwenden Sie ein Bicep-Skript, um die Azure-Datenbank for PostgreSQL – Flexibler Server und andere Ressourcen in Ihrem Azure-Abonnement bereitzustellen. Die Bicep-Skripts befinden sich im `/Allfiles/Labs/Shared`-Ordner des zuvor geklonten GitHub-Repositorys.

### Herunterladen und Installieren von Visual Studio Code und der PostgreSQL-Erweiterung

Falls Sie Visual Studio Code noch nicht installiert haben:

1. Navigieren Sie in einem Browser zum [Herunterladen von Visual Studio Code](https://code.visualstudio.com/download), und wählen Sie die entsprechende Version für Ihr Betriebssystem aus.

1. Befolgen Sie die Installationsanweisungen für Ihr Betriebssystem.

1. Öffnen Sie Visual Studio Code.

1. Klicken Sie im linken Menü auf **Erweiterungen**, um den Bereich „Erweiterungen“ anzuzeigen.

1. Geben Sie **Subnetze** in die Suchleiste ein. Das Symbol für die PostgreSQL-Erweiterung für Visual Studio Code wird angezeigt. Stellen Sie sicher, dass Sie die von Microsoft ausgewählte Option auswählen.

1. Wählen Sie **Installieren** aus. Die Erweiterung wird installiert.

### Herunterladen und Installieren der Azure CLI und Git

Wenn Sie Azure CLI oder Git noch nicht installiert haben:

1. Navigieren Sie in einem Browser zu [Installieren der Azure-CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) und befolgen Sie die Anweisungen für Ihr Betriebssystem.

1. Navigieren Sie in einem Browser zu [Git herunterladen und installieren](https://git-scm.com/downloads) und befolgen Sie die Anweisungen für Ihr Betriebssystem.

### Herunterladen der Übungsdateien

Wenn Sie bereits das GitHub-Repository geklont haben, das die Übungsdateien enthält, *überspringen Sie den Download der Übungsdateien*.

Um die Übungsdateien herunterzuladen, klonen Sie das GitHub-Repository mit den Übungsdateien auf Ihrem lokalen Computer. Das Repository enthält alle Skripts und Ressourcen, die Sie zum Abschließen dieser Übung benötigen.

1. Öffnen Sie Visual Studio Code, falls es noch nicht geöffnet wurde.

1. Wählen Sie **Alle Befehle anzeigen** (Strg + Umschalt + P), um die Befehlspalette zu öffnen.

1. Suchen Sie in der Befehlspalette nach **Git: Clone** und wählen Sie ihn aus.

1. Geben Sie in der Befehlspalette Folgendes ein, um das GitHub-Repository mit Übungsressourcen zu klonen, und drücken Sie die **Eingabetaste**:

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. Folgen Sie den Aufforderungen, um einen Ordner auszuwählen, in den das Repository geklont werden soll. Das Repository wird in einen Ordner namens `mslearn-postgresql` an dem von Ihnen gewählten Ort geklont.

1. Wenn Sie gefragt werden, ob Sie das geklonte Repository öffnen möchten, wählen Sie **Öffnen** aus. Das Repository wird in Visual Studio Code geöffnet.

## Ressourcengruppe erstellen

In diesem Abschnitt erstellen Sie eine Ressourcengruppe, die die Azure Database for PostgreSQL-Server enthält. Die Ressourcengruppe ist ein logischer Container, der zu einer Azure-Lösung gehörige Ressourcen enthält.

1. Melden Sie sich beim Azure-Portal an. Ihr Benutzerkonto muss die Rolle „Besitzer“ oder „Mitwirkender“ im Azure-Abonnement haben.

1. Wählen Sie **Ressourcengruppen** aus, und klicken Sie dann auf **+ Erstellen**.

1. Wählen Sie Ihr Abonnement aus.

1. Geben Sie in Ressourcengruppe **rg-PostgreSQL_Replication** ein.

1. Wählen Sie eine Region in der Nähe Ihres Standorts aus.

1. Klicken Sie auf **Überprüfen + erstellen**.

1. Klicken Sie auf **Erstellen**.

## Erstellen eines Verlegerservers

In diesem Abschnitt erstellen Sie den Verlegerserver. Der Verlegerserver ist die Quelle der Daten, die auf den Abonnentenserver repliziert werden sollen.

1. Wählen Sie unter „Azure-Dienste“ die Option **+ Ressource erstellen** aus. Wählen Sie unter **Kategorien** die Option **Datenbanken** aus. Wählen Sie unter **Azure Database for PostgreSQL** die Option **Erstellen** aus.

1. Füllen Sie auf der Registerkarte **Allgemeine Informationen** für „Flexibler Server“ die Felder wie folgt aus:
    - **Abonnement**: Ihr Abonnement
    - **Ressourcengruppe**: Wählen Sie **rg-PostgreSQL_Replikation** aus.
    - **Servername:** - *psql-postgresql-pub9999* (Der Name muss global eindeutig sein, ersetzen Sie also 9999 durch vier zufällige Zahlen).
    - **Region**: Wählen Sie dieselbe Region wie die Ressourcengruppe aus.
    - **PostgreSQL-Version**: Wählen Sie 16.
    - **Workloadtyp** - *Entwicklung*.
    - **Compute + Speicher** - *burstfähig*. Wählen Sie **Server konfigurieren** aus, und überprüfen Sie die Konfigurationsoptionen. Nehmen Sie keine Änderungen vor, und schließen Sie den Abschnitt.
    - **Verfügbarkeitszone**: 1. Wenn Verfügbarkeitszonen nicht unterstützt werden, übernehmen Sie „Keine Präferenz“.
    - **Hohe Verfügbarkeit**: Deaktiviert.
    - **Authentifizierungsmethode**: Nur PostgreSQL-Authentifizierung.
    - **Admin-Benutzername**, geben Sie **`pgAdmin`** ein.
    - **Kennwort**, geben Sie ein geeignetes komplexes Kennwort ein.

1. Klicken Sie auf **Weiter: Netzwerk >**.

1. Füllen Sie auf der Registerkarte **Netzwerk** für „Flexibler Server“ jedes Feld wie folgt aus:
    - **Konnektivitätsmethode**: (o) Öffentlicher Zugriff (zulässige IP-Adressen).
    - **Öffentlichen Zugriff auf diesen Server über beliebigen Azure Service in Azure gestatten**: aktiviert. Diese Checkbox muss aktiviert werden, damit die Verleger- und Abonnentendatenbanken miteinander kommunizieren können.
    - **Firewallregeln**: Wählen Sie die Option **+ Aktuelle Client-IP-Adresse hinzufügen** aus. Mit dieser Option wird Ihre aktuelle IP-Adresse als Firewallregel hinzugefügt. Sie können dieser Firewallregel optional einen aussagekräftigen Namen geben.

1. Klicken Sie auf **Überprüfen + erstellen**. Klicken Sie anschließend auf **Erstellen**.

1. Da die Erstellung einer Azure-Datenbank für PostgreSQL nur wenige Minuten dauern kann, beginnen Sie mit dem nächsten Schritt, sobald diese Bereitstellung erfolgt ist. Denken Sie daran, ein neues Browserfenster oder eine neue Registerkarte zu öffnen, um fortzufahren.

## Erstellen eines Abonnentenservers

1. Wählen Sie unter „Azure-Dienste“ die Option **+ Ressource erstellen** aus. Wählen Sie unter **Kategorien** die Option **Datenbanken** aus. Wählen Sie unter **Azure Database for PostgreSQL** die Option **Erstellen** aus.

1. Füllen Sie auf der Registerkarte **Allgemeine Informationen** für „Flexibler Server“ die Felder wie folgt aus:
    - **Abonnement**: Ihr Abonnement
    - **Ressourcengruppe**: Wählen Sie **rg-PostgreSQL_Replikation** aus.
    - **Servername:** - *psql-postgresql-pub9999* (Der Name muss global eindeutig sein, ersetzen Sie also 9999 durch vier zufällige Zahlen).
    - **Region**: Wählen Sie dieselbe Region wie die Ressourcengruppe aus.
    - **PostgreSQL-Version**: Wählen Sie 16.
    - **Workloadtyp** - *Entwicklung*.
    - **Compute + Speicher** - *burstfähig*. Wählen Sie **Server konfigurieren** aus, und überprüfen Sie die Konfigurationsoptionen. Nehmen Sie keine Änderungen vor, und schließen Sie den Abschnitt.
    - **Verfügbarkeitszone**: -2. Wenn Verfügbarkeitszonen nicht unterstützt werden, übernehmen Sie „Keine Präferenz“.
    - **Hohe Verfügbarkeit**: Deaktiviert.
    - **Authentifizierungsmethode**: Nur PostgreSQL-Authentifizierung.
    - **Admin-Benutzername**, geben Sie **`pgAdmin`** ein.
    - **Kennwort**, geben Sie ein geeignetes komplexes Kennwort ein.

1. Klicken Sie auf **Weiter: Netzwerk >**.

1. Füllen Sie auf der Registerkarte **Netzwerk** für „Flexibler Server“ jedes Feld wie folgt aus:
    - **Konnektivitätsmethode**: (o) Öffentlicher Zugriff (zulässige IP-Adressen).
    - **Öffentlichen Zugriff auf diesen Server über beliebigen Azure Service in Azure gestatten**: aktiviert. Diese Checkbox muss aktiviert werden, damit die Verleger- und Abonnentendatenbanken miteinander kommunizieren können.
    - **Firewallregeln**: Wählen Sie die Option **+ Aktuelle Client-IP-Adresse hinzufügen** aus. Mit dieser Option wird Ihre aktuelle IP-Adresse als Firewallregel hinzugefügt. Sie können dieser Firewallregel optional einen aussagekräftigen Namen geben.

1. Klicken Sie auf **Überprüfen + erstellen**. Klicken Sie anschließend auf **Erstellen**.

1. Warten Sie, bis beide Azure Database for PostgreSQL-Server bereitgestellt sind.

## Einrichten der Replikation

Für *beide*, den Verleger- und den Abonnentenserver:

1. Navigieren Sie im Azure-Portal zu Ihrem Server, und wählen Sie **Serverparameter** unter „Einstellungen“ aus.

1. Suchen Sie mithilfe der Suchleiste nach den einzelnen Parametern, und nehmen Sie die folgenden Änderungen vor:
    - `wal_level` = LOGICAL
    - `max_worker_processes`= 24

1. Wählen Sie **Speichern**. Wählen Sie dann **Speichern und neu starten** aus.

1. Warten Sie, bis beide Server neu gestartet sind.

    > &#128221; Nachdem die Server neu bereitgestellt wurden, müssen Sie möglicherweise Ihre Browserfenster aktualisieren, um festzustellen, dass die Server neu gestartet wurden.

## Einrichten des Verlegers

In diesem Abschnitt richten Sie den Verlegerserver ein. Der Verlegerserver ist die Quelle der Daten, die auf den Abonnentenserver repliziert werden sollen.

1. Öffnen Sie die erste Instanz von Visual Studio Code, um eine Verbindung mit dem Verlegerserver herzustellen.

1. Öffnen Sie den Ordner, in den Sie das Github Repository geklont haben.

1. Wählen Sie im linken Menü das **PostgreSQL-Symbol** aus.

    > &#128221; Wenn Sie das PostgreSQL-Symbol nicht sehen, wählen Sie das Symbol **Erweiterungen** und suchen Sie nach **PostgreSQL**. Wählen Sie die Erweiterung **PostgreSQL** von Microsoft aus und klicken Sie auf **Installieren**.

1. Wenn Sie bereits eine Verbindung mit Ihrem PostgreSQL-*Verlegerserver* erstellt haben, fahren Sie mit dem nächsten Schritt fort. So erstellen Sie eine neue Verbindung:

    1. Wählen Sie in der Erweiterung **PostgreSQL** die Option **+ Verbindung hinzufügen**, um eine neue Verbindung hinzuzufügen.

    1. Geben Sie im Dialogfeld **NEUE VERBINDUNG** die folgenden Informationen ein:

        - **Servername**: `<your-publisher-server-name>`.postgres.database.azure.com
        - **Authentifizierungstyp**: Kennwort
        - **Benutzername**: pgAdmin
        - **Kennwort**: Das zufällige Kennwort, das Sie zuvor generiert haben.
        - Aktivieren Sie das Kontrollkästchen **Kennwort speichern**.
        - **Verbindungsname**: `<your-publisher-server-name>`

    1. Überprüfen Sie die Verbindung, indem Sie **Verbindung testen** auswählen. Wenn die Verbindung erfolgreich hergestellt wurde, wählen Sie **Speichern und verbinden**, um die Verbindung zu speichern. Andernfalls überprüfen Sie die Verbindungsinformationen und versuchen Sie es erneut.

1. Falls noch keine Verbindung besteht, wählen Sie für Ihren PostgreSQL-Server die Option **Verbinden**. Sie sind nun mit dem Azure Database for PostgreSQL-Server verbunden.

1. Erweitern Sie den Serverknoten und die zugehörigen Datenbanken. Die vorhandenen Datenbanken werden aufgelistet.

1. Wählen Sie in Visual Studio Code **Datei** und **Datei öffnen** aus und navigieren Sie zu dem Ordner, in dem Sie die Skripte gespeichert haben. Wählen Sie **../Allfiles/Labs/06/Lab6_Replication.sql** und **Öffnen** aus.

1. Vergewissern Sie sich unten rechts in Visual Studio Code, dass die Verbindung in Grün angezeigt wird. Ist dies nicht der Fall, sollte **PGSQL Disconnected** angezeigt werden. Wählen Sie den Text **PGSQL Disconnected** aus und wählen Sie anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette aus. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

1. Es ist Zeit, den *Verleger* einzurichten.

    1. Markieren und führen Sie den Abschnitt **Grant the admin user replication permission** aus.

    1. Markieren und führen Sie den Abschnitt **Create zoodb database** aus.

    1. Wenn Sie nur die Anweisung **SELECT current_database()** markieren und ausführen, werden Sie feststellen, dass die Datenbank derzeit auf `postgres` festgelegt ist. Sie müssen sie in `zoodb` ändern.

    1. Wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus.

        > &#128221; Sie können die Datenbank auch im Abfragebereich ändern. Sie können den Servernamen und den Datenbanknamen unter der Abfrageregisterkarte selbst notieren. Durch Auswahl des Datenbanknamens wird eine Liste der Datenbanken angezeigt. Wählen Sie die Datenbank `zoodb` aus der Liste aus.

    1. Markieren Sie den Abschnitt **Tabellen erstellen** und **Fremdschlüssel-Beschränkungen** in zoodb und führen Sie ihn aus.

    1. Markieren und führen Sie den Abschnitt **Populate the tables in zoodb** aus.

    1. Markieren und führen Sie den Abschnitt **Create a publication** aus. Wenn Sie die SELECT-Anweisung ausführen, wird nichts aufgelistet, da die Replikation noch nicht aktiv ist.

    1. Führen Sie den Abschnitt **CREATE SUBSCRIPTION** NICHT aus. Dieses Skript wird auf dem Abonnentenserver ausgeführt.

    1. Schließen Sie diese Visual Studio Code-Instanz NICHT, minimieren Sie sie einfach. Nachdem Sie den Abonnentenserver eingerichtet haben, kehren Sie zu ihr zurück.

Sie haben nun den Verlegerrserver und die zoodb-Datenbank erstellt. Die Datenbank enthält die Tabellen und Daten, die auf den Abonnentenserver repliziert werden.

## Einrichten des Abonnenten

In diesem Abschnitt richten Sie den Abonnentenserver ein. Der Abonnentenserver ist das Ziel der Daten, die vom Verlegerserver repliziert werden. Sie erstellen eine neue Datenbank auf dem Abonnentenserver, die mit den Daten vom Verlegerserver gefüllt wird.

1. Öffnen Sie eine *zweite Instanz* von Visual Studio Code und verbinden Sie sich mit dem Abonnentenserver.

1. Öffnen Sie den Ordner, in den Sie das Github Repository geklont haben.

1. Wählen Sie im linken Menü das **PostgreSQL-Symbol** aus.

    > &#128221; Wenn Sie das PostgreSQL-Symbol nicht sehen, wählen Sie das Symbol **Erweiterungen** und suchen Sie nach **PostgreSQL**. Wählen Sie die Erweiterung **PostgreSQL** von Microsoft aus und klicken Sie auf **Installieren**.

1. Wenn Sie bereits eine Verbindung mit Ihrem PostgreSQL-*Abonnentenserver* erstellt haben, fahren Sie mit dem nächsten Schritt fort. So erstellen Sie eine neue Verbindung:

    1. Wählen Sie in der Erweiterung **PostgreSQL** die Option **+ Verbindung hinzufügen**, um eine neue Verbindung hinzuzufügen.

    1. Geben Sie im Dialogfeld **NEUE VERBINDUNG** die folgenden Informationen ein:

        - **Servername**: `<your-subscriber-server-name>`.postgres.database.azure.com
        - **Authentifizierungstyp**: Kennwort
        - **Benutzername**: pgAdmin
        - **Kennwort**: Das zufällige Kennwort, das Sie zuvor generiert haben.
        - Aktivieren Sie das Kontrollkästchen **Kennwort speichern**.
        - **Verbindungsname**: `<your-subscriber-server-name>`

    1. Überprüfen Sie die Verbindung, indem Sie **Verbindung testen** auswählen. Wenn die Verbindung erfolgreich hergestellt wurde, wählen Sie **Speichern und verbinden**, um die Verbindung zu speichern. Andernfalls überprüfen Sie die Verbindungsinformationen und versuchen Sie es erneut.

1. Falls noch keine Verbindung besteht, wählen Sie für Ihren PostgreSQL-Server die Option **Verbinden**. Sie sind nun mit dem Azure Database for PostgreSQL-Server verbunden.

1. Erweitern Sie den Serverknoten und die zugehörigen Datenbanken. Die vorhandenen Datenbanken werden aufgelistet.

1. Wählen Sie in Visual Studio Code **Datei** und **Datei öffnen** aus und navigieren Sie zu dem Ordner, in dem Sie die Skripte gespeichert haben. Wählen Sie **../Allfiles/Labs/06/Lab6_Replication.sql** und **Öffnen** aus.

1. Vergewissern Sie sich unten rechts in Visual Studio Code, dass die Verbindung in Grün angezeigt wird. Ist dies nicht der Fall, sollte **PGSQL Disconnected** angezeigt werden. Wählen Sie den Text **PGSQL Disconnected** aus und wählen Sie anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette aus. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

1. Es ist Zeit, den *Abonnenten* einzurichten.

    1. Markieren und führen Sie den Abschnitt **Grant the admin user replication permission** aus.

    1. Markieren und führen Sie den Abschnitt **Create zoodb database** aus.

    1. Wenn Sie nur die Anweisung **SELECT current_database()** markieren und ausführen, werden Sie feststellen, dass die Datenbank derzeit auf `postgres` festgelegt ist. Sie müssen sie in `zoodb` ändern.

    1. Wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus.

        > &#128221; Sie können die Datenbank auch im Abfragebereich ändern. Sie können den Servernamen und den Datenbanknamen unter der Abfrageregisterkarte selbst notieren. Durch Auswahl des Datenbanknamens wird eine Liste der Datenbanken angezeigt. Wählen Sie die Datenbank `zoodb` aus der Liste aus.

    1. Markieren Sie den Abschnitt **Tabellen erstellen** und **Fremdschlüssel-Beschränkungen** in `zoodb` und führen Sie ihn aus.

    1. Führen Sie NICHT den Abschnitt **Publikation erstellen** aus, diese Anweisung wurde bereits auf dem Verlegerserver ausgeführt.

    1. Scrollen Sie nach unten zum Abschnitt **Create a subscription**.

        1. Bearbeiten Sie die Anweisung **CREATE SUBSCRIPTION** so, dass sie den korrekten Namen des Verlegerservers und das sichere Kennwort des Verlegers enthält. Markieren und führen Sie die Anweisung aus.

        1. Markieren und führen Sie die **SELECT**-Anweisung aus. Diese zeigt das Abonnement „sub“, das Sie bereits erstellt haben.

    1. Markieren Sie unter dem Abschnitt **Display the tables**, und führen Sie alle **SELECT**-Anweisungen aus. Der Verlegerserver hat diese Tabellen über die Replikation gefüllt.

Sie haben den Abonnentenserver und die zoodb-Datenbank erstellt. Die Datenbank enthält die Tabellen und Daten, die vom Verlegerserver repliziert wurden.

## Vornehmen von Änderungen an der Verlegerdatenbank

- Markieren Sie in der ersten Instanz von Visual Studio Code (*Ihre Verleger-Instanz*) unter **Weitere Tiere einfügen** die Anweisung **INSERT** und führen Sie sie aus. *Stellen Sie sicher, dass Sie **diese INSERT-Anweisung nicht** beim Abonnenten ausführen*.

## Anzeigen der Änderungen in der Abonnentendatenbank

- Markieren Sie in der zweiten Instanz von Visual Studio Code (Abonnent) unter **Display the animal tables** die **SELECT**-Anweisung, und führen Sie sie aus.

Sie haben nun zwei Instanzen von Azure Database for PostgreSQL – Flexibler Server erstellt und einen als Verleger und den anderen als Abonnenten konfiguriert. In der Verlegerdatenbank haben Sie die Datenbank „zoo“ erstellt und aufgefüllt. In der Abonnentendatenbank haben Sie eine leere Datenbank erstellt, die dann per Streamingreplikation aufgefüllt wurde.

## Bereinigung

1. Wenn Sie diese PostgreSQL-Server nicht mehr für andere Übungen benötigen, um unnötige Azure-Kosten zu vermeiden, löschen Sie die in dieser Übung erstellte Ressourcengruppe.

1. Wenn Sie möchten, dass die PostgreSQL-Server weiterlaufen, können Sie sie einfach laufen lassen. Wenn Sie sie nicht laufen lassen möchten, können Sie den Server beenden, um unnötige Kosten im Bash-Terminal zu vermeiden. Um die Server zu beenden, führen Sie für jeden Server den folgenden Befehl aus:

    ```bash

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Ersetzen Sie `<your-server-name>` durch den Namen Ihrer PostgreSQL-Server.

    > &#128221; Sie können den Server auch über das Azure-Portal stoppen. Navigieren Sie im Azure-Portal zu **Ressourcengruppen**, und wählen Sie die zuvor erstellte Ressourcengruppe aus. Wählen Sie den PostgreSQL-Server aus und wählen Sie anschließend im Menü die Option **Beenden**. Führen Sie diesen Schritt für beide, den Verleger- und den Abonnentenserver, aus:

1. Löschen Sie bei Bedarf das Git Repository, das Sie zuvor geklont haben.

Sie haben erfolgreich einen PostgreSQL-Server erstellt und für die logische Replikation konfiguriert. Sie haben einen Verlegerserver und einen Abonnentenserver erstellt und die Replikation zwischen ihnen eingerichtet. Außerdem haben Sie Änderungen an der Verlegerdatenbank vorgenommen und die Änderungen in der Abonnentendatenbank angezeigt. Sie haben nun ein gutes Verständnis dafür, wie Sie die logische Replikation in PostgreSQL einrichten.
