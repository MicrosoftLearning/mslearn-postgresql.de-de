---
lab:
  title: Bewerten der Abfrageleistung mit Query Store
  module: Tune queries in Azure Database for PostgreSQL
---

# Bewerten der Abfrageleistung mit Query Store

In dieser Übung erfahren Sie, wie Sie Leistungsmetriken mithilfe des Abfragespeichers in Azure Database for PostgreSQL abfragen.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Wenn Sie nicht über ein Azure-Abonnement verfügen, können Sie ein [kostenloses Azure-Testkonto](https://azure.microsoft.com/free) erstellen.

Zusätzlich muss Folgendes auf Ihrem Computer installiert sein:

- Visual Studio Code.
- Postgres Visual Studio Code-Erweiterung von Microsoft
- Azure-Befehlszeilenschnittstelle.
- Git.

## Erstellen der Übungsumgebung

In diesen und späteren Übungen verwenden Sie ein Bicep-Skript, um die Azure Database for PostgreSQL – Flexibler Server und andere Ressourcen in Ihrem Azure-Abonnement bereitzustellen. Die Bicep-Skripts befinden sich im `/Allfiles/Labs/Shared`-Ordner des zuvor geklonten Github Repositorys.

### Laden Sie die Visual Studio Code- und die PostgreSQL-Erweiterung herunter, und installieren Sie sie.

Wenn Sie Visual Studio Code nicht installiert haben:

1. Navigieren Sie in einem Browser zu [Herunterladen von Visual Studio Code](https://code.visualstudio.com/download) , und wählen Sie die entsprechende Version für Ihr Betriebssystem aus.

1. Folgen Sie den Anweisungen zur Installation für Ihr Betriebssystem.

1. Öffnen Sie Visual Studio Code.

1. Klicken Sie im linken Menü auf **Erweiterungen**, um den Bereich „Erweiterungen“ anzuzeigen.

1. Geben Sie **Subnetze** in die Suchleiste ein. Das Symbol der PostgreSQL-Erweiterung für Visual Studio Code wird angezeigt. Stellen Sie sicher, dass Sie die Microsoft-Option auswählen.

1. Wählen Sie **Installieren** aus. Die Erweiterung wird installiert.

### Herunterladen und Installieren von Azure CLI und Git

Wenn Sie Azure CLI oder Git nicht installiert haben:

1. Navigieren Sie in einem Browser zu [Azure CLI installieren](https://learn.microsoft.com/cli/azure/install-azure-cli) und befolgen Sie die Anweisungen für Ihr Betriebssystem.

1. Navigieren Sie in einem Browser zu [Git herunterladen und installieren](https://git-scm.com/downloads) und folgen Sie den Anweisungen für Ihr Betriebssystem.

### Herunterladen der Übungsdateien

Wenn Sie bereits das GitHub-Repository geklont haben, das die Übungsdateien enthält, *überspringen Sie den Download der Übungsdateien*.

Um die Übungsdateien herunterzuladen, klonen Sie das Github Repository mit den Übungsdateien auf Ihrem lokalen Computer. Das Repository enthält alle Skripts und Ressourcen, die Sie zum Abschließen dieser Übung benötigen.

1. Öffnen Sie Visual Studio Code, wenn es noch nicht geöffnet ist.

1. Wählen Sie **Alle Befehle anzeigen** (STRG+UMSCHALT+P) aus, um die Befehlspalette zu öffnen.

1. Suchen Sie in der Befehlspalette den Befehl **Git: Klonen**, und wählen Sie ihn aus.

1. Geben Sie in der Befehlspalette Folgendes ein, um das Github Repository mit Übungsressourcen zu klonen, und drücken Sie die **EINGABETASTE**:

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. Folgen Sie den Prompts, um einen Ordner auszuwählen, in den das Repository geklont werden soll. Das Repository wird in einen Ordner namens `mslearn-postgresql` an dem von Ihnen ausgewählten Speicherort geklont.

1. Wenn Sie gefragt werden, ob Sie das geklonte Repository öffnen möchten, wählen Sie **Öffnen** aus. Das Repository wird in Visual Studio Code geöffnet.

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Wenn Ihre Azure-Ressourcen bereits installiert sind, *überspringen Sie die Bereitstellung von Ressourcen*.

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus Visual Studio Code, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

> &#128221; Wenn Sie mehrere Module in diesem Lernpfad absolvieren, können Sie die Azure-Umgebung für alle gemeinsam nutzen. In diesem Fall müssen Sie diesen Schritt der Ressourcenzuteilung nur einmal ausführen.

1. Öffnen Sie Visual Studio Code, falls es noch nicht geöffnet ist, und öffnen Sie den Ordner, in den Sie das Github Repository geklont haben.

1. Erweitern Sie den Ordner **mslearn-postgresql** im Explorer-Bereich.

1. Erweitern Sie den Ordner **Allfiles/Labs/Shared** .

1. Klicken Sie mit der rechten Maustaste auf den Ordner **Allfiles/Labs/Shared** und wählen Sie **In integriertem Terminal öffnen** aus. Diese Auswahl öffnet ein Terminalfenster im Visual Studio Code-Fenster.

1. Das Terminal kann standardmäßig ein **PowerShell**-Fenster öffnen. Für diesen Abschnitt des Labs verwenden Sie die **Bash Shell**. Neben dem **+**-Symbol gibt es einen Dropdownpfeil. Wählen Sie ihn aus, und wählen Sie **Git Bash** oder **Bash** aus der Liste der verfügbaren Profile aus. Diese Auswahl öffnet ein neues Terminalfenster mit der **Bash Shell**.

    > &#128221; Sie können das **PowerShell**-Terminalfenster schließen, wenn Sie möchten, aber es ist nicht erforderlich. Sie können mehrere Terminalfenster gleichzeitig geöffnet haben.

1. Führen Sie im Terminalfenster den folgenden Befehl aus, um sich bei Ihrem Azure-Konto anzumelden:

    ```bash
    az login
    ```

    Dieser Befehl öffnet ein neues Browserfenster, in dem Sie aufgefordert werden, sich bei Ihrem Azure-Konto anzumelden. Kehren Sie nach der Anmeldung zum Terminalfenster zurück.

1. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen (`RG_NAME`), für die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden und für ein zufällig generiertes Kennwort für den PostgreSQL-Administrator-Login (`ADMIN_PASSWORD`).

    Im ersten Befehl ist die Region, die der entsprechenden Variablen zugewiesen ist, `eastus`, aber Sie können sie auch durch einen Ort Ihrer Wahl ersetzen.

    ```bash
    REGION=eastus
    ```

    Der folgende Befehl weist den Namen zu, der für die Ressourcengruppe verwendet werden soll, die alle in dieser Übung verwendeten Ressourcen enthält. Der Name der Ressourcengruppe, die der entsprechenden Variablen zugewiesen ist, lautet `rg-learn-work-with-postgresql-$REGION`, wobei `$REGION` der zuvor angegebene Speicherort ist. *Sie können den Namen jedoch auch in einen anderen Namen für die Ressourcengruppe ändern, der Ihren Wünschen entspricht oder den Sie bereits verwenden*.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    Der letzte Befehl generiert nach dem Zufallsprinzip ein Kennwort für die PostgreSQL-Admin-Anmeldedaten. Kopieren Sie sie an einen sicheren Ort, damit Sie sie später verwenden können, um eine Verbindung zu Ihrem flexiblen PostgreSQL-Server herzustellen.

    ```bash
    #!/bin/bash
    
    # Define array of allowed characters explicitly
    chars=( {a..z} {A..Z} {0..9} '!' '@' '#' '$' '%' '^' '&' '*' '(' ')' '_' '+' )
    
    a=()
    for ((i = 0; i < 100; i++)); do
        rand_char=${chars[$RANDOM % ${#chars[@]}]}
        a+=("$rand_char")
    done
    
    # Join first 18 characters without delimiter
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]:0:18}")
    
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo "$ADMIN_PASSWORD"
    echo "Please copy it to a safe place, as you will need it later to connect to your PostgreSQL flexible server."
    ```

1. Wenn Sie Zugriff auf mehr als ein Azure-Abonnement haben und Ihr Standardabonnement *nicht* das Abonnement ist, in dem Sie die Ressourcengruppe und andere Ressourcen für diese Übung erstellen möchten, führen Sie diesen Befehl aus, um das entsprechende Abonnement festzulegen. Ersetzen Sie dabei das Token `<subscriptionName|subscriptionId>` durch den Namen oder die ID des Abonnements, das Sie verwenden möchten:

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. (Überspringen Sie diesen Schritt, wenn Sie bereits eine Ressourcengruppe verwenden.) Führen Sie den folgenden Azure CLI-Befehl aus, um Ihre Ressourcengruppe zu erstellen:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. Verwenden Sie schließlich die Azure CLI, um ein Bicep-Bereitstellungsskript auszuführen, um Azure-Ressourcen in Ihrer Ressourcengruppe bereitzustellen:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    Das Bicep-Bereitstellungsskript stellt die für diese Übung erforderlichen Azure-Service in Ihrer Ressourcengruppe bereit. Die eingesetzten Ressourcen sind eine Azure-Datenbank für PostgreSQL – Flexibler Server. Das Bicep-Skript erstellt auch die Adventureworks-Datenbank.

    Die Bereitstellung dauert in der Regel mehrere Minuten. Sie können den Vorgang über das Bash-Terminal überwachen oder zur Seite **Bereitstellungen** für die zuvor erstellte Ressourcengruppe navigieren und dort den Fortschritt der Bereitstellung beobachten.

1. Da das Skript einen zufälligen Namen für den PostgreSQL-Server erstellt, können Sie den Namen des Servers finden, indem Sie den folgenden Befehl ausführen:

    ```azurecli
    az postgres flexible-server list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" --output table
    ```

    Notieren Sie sich den Namen des Servers, da Sie ihn benötigen, um später in dieser Übung eine Verbindung mit dem Server herzustellen.

    > &#128221; Sie finden auch den Namen des Servers im Azure-Portal. Navigieren Sie im Azure-Portal zu **Ressourcengruppen**, und wählen Sie die zuvor erstellte Ressourcengruppe aus. Der PostgreSQL-Server ist in der Ressourcengruppe aufgeführt.

### Problembehandlung bei der Bereitstellung

Beim Ausführen des Bicep-Bereitstellungsskripts können einige Fehler auftreten. Die häufigsten Meldungen und die Schritte zu ihrer Behebung sind:

- Wenn Sie zuvor das Bicep-Bereitstellungsskript für diesen Lernpfad ausgeführt und anschließend die Ressourcen gelöscht haben, wird möglicherweise die folgende Fehlermeldung angezeigt, wenn Sie versuchen, das Skript innerhalb von 48 Stunden nach dem Löschen der Ressourcen erneut auszuführen:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Wenn Sie diese Meldung erhalten, ändern Sie den vorherigen Befehl `azure deployment group create` so, dass der Parameter `restore` auf `true` festgelegt wird, und führen Sie ihn erneut aus.

- Wenn die ausgewählte Region für die Bereitstellung bestimmter Ressourcen eingeschränkt ist, müssen Sie die Variable `REGION` auf einen anderen Speicherort setzen und die Befehle erneut ausführen, um die Ressourcengruppe zu erstellen und das Bicep-Bereitstellungsskript auszuführen.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Wenn das Lab KI-Ressourcen benötigt, wird möglicherweise die folgende Fehlermeldung angezeigt. Dieser Fehler tritt auf, wenn das Skript keine KI-Ressource erstellen kann, da die zuständige KI-Vereinbarung akzeptiert werden muss. Wenn dies der Fall ist, verwenden Sie die Azure-Portal Benutzeroberfläche, um eine Azure KI Services-Ressource zu erstellen, und führen Sie dann das Bereitstellungsskript erneut aus.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

### Installieren Sie psql.

Da wir die Daten aus der CSV-Datei in die PostgreSQL-Datenbank kopieren müssen, müssen wir auf Ihrem lokalen Computer `psql` installieren. *Wenn Sie `psql` bereits installiert haben, überspringen Sie diesen Abschnitt.*

1. Um zu überprüfen, ob **psql** bereits in Ihrer Umgebung installiert ist, öffnen Sie die Befehlszeile/Terminal und führen Sie den Befehl ***psql*** aus. Wenn Sie eine Meldung wie *psql: Fehler: Verbindung zum Server auf Socket …* erhalten, bedeutet dies, dass das Tool **psql** bereits in Ihrer Umgebung installiert ist und Sie es nicht erneut installieren müssen. Sie können diesen Abschnitt überspringen.

1. Installieren Sie [psql](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893).

1. Folgen Sie im Setup-Assistenten den Anweisungen, bis Sie das Dialogfeld **Komponenten auswählen** erreichen und wählen Sie **Befehlszeilentools** aus. Sie können die anderen Komponenten deaktivieren, wenn Sie diese nicht verwenden möchten. Befolgen Sie die Anweisungen, um die Installation abzuschließen.

1. Öffnen Sie eine neue Eingabeaufforderung oder ein Terminalfenster, und führen Sie **psql** aus, um zu überprüfen, ob die Installation erfolgreich war. Wenn eine Meldung wie *psql: Fehler: Verbindung zum Server auf Socket...*" angezeigt wird, bedeutet dies, dass das Tool **psql** erfolgreich installiert ist. Andernfalls müssen Sie möglicherweise das PostgreSQL-Lagerfach-Verzeichnis zu Ihrer Systempfadvariablen hinzufügen.

    1. Wenn Sie Windows verwenden, stellen Sie sicher, dass Sie der Systempfadvariablen das PostgreSQL-Lagerfach-Verzeichnis hinzufügen. Der Speicherort des Lagerfach-Verzeichnisses ist in der Regel das `C:\Program Files\PostgreSQL\<version>\bin`.
        1. Sie können überprüfen, ob sich das Lagerfach-Verzeichnis in Ihrer Pfadvariablen befindet, indem Sie den Befehl `echo %PATH%` in einer Eingabeaufforderung ausführen und überprüfen, ob das PostgreSQL-Lagerfach-Verzeichnis aufgeführt ist. Wenn nicht, können Sie es manuell hinzufügen.
        1. Um es manuell hinzuzufügen, klicken Sie mit der rechten Maustaste auf die Schaltfläche **Start**.
        1. Wählen Sie **System** und anschließend **Erweiterte Systemeinstellungen** aus.
        1. Wählen Sie die Registerkarte **Umgebungsvariablen** aus.
        1. Doppelklicken Sie im Abschnitt **Systemvariablen** auf die **Pfadvariable**.
        1. Wählen Sie **Neu** aus, und fügen Sie den Pfad zum PostgreSQL-Lagerfach-Verzeichnis hinzu.
        1. Schließen Sie nach dem Hinzufügen die Eingabeaufforderung, und öffnen Sie sie erneut, damit die Änderungen wirksam werden.

    1. Wenn Sie macOS oder Linux verwenden, befindet sich das PostgreSQL-`bin`-Verzeichnis in der Regel unter `/usr/local/pgsql/bin`.  
        1. Sie können überprüfen, ob sich dieses Verzeichnis in Ihrer `PATH`-Umgebungsvariable befindet, indem Sie `echo $PATH` in einem Terminal ausführen.  
        1. Andernfalls können Sie es hinzufügen, indem Sie die Shellkonfigurationsdatei bearbeiten, in der Regel `.bash_profile`, `.bashrc`oder `.zshrc`, je nach Shell.

### Auffüllen der Datenbank mit Daten

Nachdem Sie überprüft haben, ob **psql** installiert ist, können Sie über die Befehlszeile eine Verbindung mit Ihrem PostgreSQL-Server herstellen, indem Sie eine Eingabeaufforderung oder ein Terminalfenster öffnen.

> &#128221; Wenn Sie Windows verwenden, können Sie die **Windows PowerShell** oder die **Eingabeaufforderung** verwenden. Wenn Sie macOS oder Linux verwenden, können Sie die Anwendung **Terminal** verwenden.

Die Syntax für die Verbindung mit dem Server lautet:

```sql
psql -h <servername> -p <port> -U <username> <dbname>
```

1. Geben Sie an der Eingabeaufforderung oder dem Terminal **`--host=<servername>.postgres.database.azure.com`** ein, wobei `<servername>` der Name der oben erstellten Azure Database for PostgreSQL ist.

    Sie finden den Servernamen unter **Übersicht** im Azure-Portal oder als Ausgabe des Bicep-Skripts oder im Azure-Portal.

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin adventureworks
    ```

    Sie werden aufgefordert, das Kennwort für das oben kopierte Administratorkonto einzugeben.

1. Sie müssen eine Tabelle in der Datenbank erstellen und sie mit Beispieldaten füllen, damit Sie Informationen haben, mit denen Sie arbeiten können, wenn Sie in dieser Übung das Sperren überprüfen.

1. Führen Sie den folgenden Befehl aus, um die Tabelle `production.workorder` für das Laden der Daten zu erstellen:

    ```sql
    DROP SCHEMA IF EXISTS production CASCADE;
    CREATE SCHEMA production;
    
    DROP TABLE IF EXISTS production.workorder;
    CREATE TABLE production.workorder
    (
        workorderid integer NOT NULL,
        productid integer NOT NULL,
        orderqty integer NOT NULL,
        scrappedqty smallint NOT NULL,
        startdate timestamp without time zone NOT NULL,
        enddate timestamp without time zone,
        duedate timestamp without time zone NOT NULL,
        scrapreasonid smallint,
        modifieddate timestamp without time zone NOT NULL DEFAULT now()
    )
    WITH (
        OIDS = FALSE
    )
    TABLESPACE pg_default;
    ```

1. Als nächstes verwenden Sie den Befehl `COPY`, um Daten aus CSV-Dateien in die oben erstellte Tabelle zu laden. Beginnen Sie, indem Sie den folgenden Befehl ausführen, um die Tabelle `production.workorder` aufzufüllen:

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/08/Lab8_workorder.csv' CSV HEADER
    ```

    Die Befehlsausgabe sollte `COPY 72591` lauten, was bedeutet, dass 72.591 Zeilen aus der CSV-Datei in die Tabelle geschrieben wurden.

1. Schließen Sie die Eingabeaufforderung oder das Terminalfenster.

### Stellen Sie mit Visual Studio Code eine Verbindung zur Datenbank her 

1. Öffnen Sie Visual Studio Code, falls es noch nicht geöffnet ist, und öffnen Sie den Ordner, in den Sie das Github Repository geklont haben.

1. Wählen Sie im linken Menü das **PostgreSQL**-Symbol aus.

    > &#128221; Wenn das PostgreSQL-Symbol nicht angezeigt wird, wählen Sie das Symbol **Erweiterungen** aus, und suchen Sie nach **PostgreSQL**. Wählen Sie die Erweiterung **PostgreSQL** von Microsoft und dann **Installieren** aus.

1. Wenn Sie bereits eine Verbindung mit Ihrem PostgreSQL-Server erstellt haben, fahren Sie mit dem nächsten Schritt fort. So erstellen Sie eine neue Verbindung:

    1. Wählen Sie in der Erweiterung **PostgreSQL** die Option **+ Verbindung hinzufügen**, um eine neue Verbindung hinzuzufügen.

    1. Geben Sie im Dialogfeld **NEUE VERBINDUNG** die folgenden Informationen ein:

        - **Servername**: `<your-server-name>`.postgres.database.azure.com
        - **Authentifizierungstyp**: Kennwort
        - **Benutzername**: pgAdmin
        - **Kennwort**: Das zufällige Kennwort, das Sie zuvor generiert haben.
        - Aktivieren Sie das Kontrollkästchen **Kennwort speichern**.
        - **Verbindungsname**: `<your-server-name>`

    1. Überprüfen Sie die Verbindung, indem Sie **Verbindung testen** auswählen. Wenn die Verbindung erfolgreich hergestellt wurde, wählen Sie **Speichern und verbinden**, um die Verbindung zu speichern. Andernfalls überprüfen Sie die Verbindungsinformationen und versuchen Sie es erneut.

1. Falls noch keine Verbindung besteht, wählen Sie für Ihren PostgreSQL-Server die Option **Verbinden**. Sie sind nun mit dem Azure Database for PostgreSQL-Server verbunden.

1. Erweitern Sie den Serverknoten und die zugehörigen Datenbanken. Die vorhandenen Datenbanken werden aufgelistet.

1. Öffnen Sie Visual Studio Code, wenn es noch nicht geöffnet ist.

1. Öffnen Sie die Befehlspalette (Strg + Umschalt + P) und wählen Sie **PGSQL: New Query**. Wählen Sie die neue Verbindung aus, die Sie in der Liste in der Befehlspalette erstellt haben. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das Kennwort ein, das Sie für die neue Rolle erstellt haben.

1. Stellen Sie unten rechts im Fenster **Neue Abfrage** sicher, dass die Verbindung grün ist. Ist dies nicht der Fall, sollte **PGSQL Disconnected** angezeigt werden. Wählen Sie den Text **PGSQL Disconnected** aus und wählen Sie anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette aus. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

1. Kopieren Sie im Fenster **Neue Abfrage** die folgende SQL-Anweisung, markieren Sie sie und führen Sie sie aus:

    ```sql
    SELECT current_database();
    ```

1. Wenn die aktuelle Datenbank nicht auf **adventureworks** festgelegt ist, müssen Sie die Datenbank in **adventureworks** ändern. Um die Datenbank zu ändern, wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `adventureworks` aus. Überprüfen Sie, ob die Datenbank jetzt auf `adventureworks` festgelegt ist, indem Sie die Anweisung **SELECT current_database();** ausführen.

1. Kopieren Sie im Fenster **Neue Abfrage** die folgende SQL-Anweisung, markieren Sie sie und führen Sie sie aus:

    ```sql
    SELECT * FROM production.workorder;
    ```

1. Minimieren Sie das Fenster **Neue Abfrage**, und kehren Sie später in dieser Übung dorthin zurück.

## Aufgabe 1: Aktivieren des Abfrageaufzeichnungsmodus

1. Wechseln Sie zum Azure-Portal, und melden Sie sich an.

1. Wählen Sie für diese Übung Ihren Azure Database for PostgreSQL-Server aus.

1. Wählen Sie unter **Einstellungen** die Option **Serverparameter** aus.

1. Navigieren Sie zu der Einstellung **`pg_qs.query_capture_mode`**.

1. Wählen Sie **TOP** aus.

1. Navigieren Sie zu **`pgms_wait_sampling.query_capture_mode`**, wählen Sie **ALLE** und wählen Sie **Speichern**.

1. Warten Sie, bis die Serverparameter aktualisiert werden.

## Anzeigen von pg_stat-Daten

1. Kehren Sie zu Visual Studio Code zurück, und wählen Sie das Fenster **Neue Abfrage** aus, das wir zuvor geöffnet haben.

1. Ersetzen Sie die Abfrage durch die folgende, und wählen Sie dann **Ausführen** aus.

    ```sql
    SELECT 
        pid,                    -- Process ID of the server process
        datid,                  -- OID of the database
        datname,                -- Name of the database
        usename,                -- Name of the user
        application_name,       -- Name of the application connected to the database
        client_addr,            -- IP address of the client
        client_hostname,        -- Hostname of the client (if available)
        client_port,            -- TCP port number that the client is using for the connection
        backend_start,          -- Timestamp when the backend process started
        xact_start,             -- Timestamp of the current transaction start, if any
        query_start,            -- Timestamp when the current query started, if any
        state_change,           -- Timestamp when the state was last changed
        wait_event_type,        -- Type of event the backend is waiting for, if any
        wait_event,             -- Event that the backend is waiting for, if any
        state,                  -- Current state of the session (e.g., active, idle, etc.)
        backend_xid,            -- Transaction ID, if active
        backend_xmin,           -- Transaction ID that the process is working with
        query,                  -- Text of the query being executed
        encode(backend_type::bytea, 'escape') AS backend_type,           -- Type of backend (e.g., client backend, autovacuum worker). We use encode(…, 'escape') to safely display raw data with invalid characters by converting it into a readable format, doing this prevents a UTF-8 conversion error in Visual Studio Code.
        leader_pid,             -- PID of the leader process, if this is a parallel worker
        query_id               -- Query ID (added in more recent PostgreSQL versions)
    FROM pg_stat_activity;
    ```

1. Überprüfen Sie die verfügbaren Metriken.

1. Lassen Sie Visual Studio Code für den nächsten Vorgang geöffnet.

## Aufgabe 2: Untersuchen von Abfragestatistiken

> &#128221; Für eine neu erstellte Datenbank gibt es, wenn überhaupt, möglicherweise nur begrenzte Statistiken. Wenn Sie 30 Minuten lang warten, sind Statistiken aus Hintergrundprozessen verfügbar.

1. Kopieren Sie im Fenster **Neue Abfrage** die folgende SQL-Anweisung, markieren Sie sie und führen Sie sie aus:

    ```sql
    SELECT current_database();
    ```

1. Sie müssen die Datenbank in **azure_sys** ändern. Um die Datenbank zu ändern, wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `azure_sys` aus. Überprüfen Sie, ob die Datenbank jetzt auf `azure_sys` festgelegt ist, indem Sie die Anweisung **SELECT current_database();** ausführen.

1. Kopieren Sie im Fenster **Neue Abfrage** die folgende SQL-Anweisung, markieren Sie sie und führen Sie sie aus:

    ```sql
    SELECT * FROM query_store.query_texts_view;
    ```

    ```sql
    SELECT * FROM query_store.qs_view;
    ```

    ```sql
    SELECT * FROM query_store.runtime_stats_view;
    ```

    ```sql
    SELECT * FROM query_store.pgms_wait_sampling_view;
    ```

1. Überprüfen Sie die verfügbaren Metriken.

## Bereinigung

1. Wenn Sie diesen PostgreSQL-Server nicht mehr für andere Übungen benötigen, um unnötige Azure-Kosten zu vermeiden, löschen Sie die in dieser Übung erstellte Ressourcengruppe.

1. Wenn Sie möchten, dass der PostgreSQL-Server weiterläuft, können Sie ihn einfach laufen lassen. Wenn Sie ihn nicht laufen lassen wollen, können Sie den Server beenden, um unnötige Kosten im Bash-Terminal zu vermeiden. Um den Server zu beenden, führen Sie den folgenden Befehl aus:

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Ersetzen Sie `<your-server-name>` durch den Namen Ihres PostgreSQL-Servers.

    > &#128221; Sie können den Server auch über das Azure-Portal stoppen. Navigieren Sie im Azure-Portal zu **Ressourcengruppen**, und wählen Sie die zuvor erstellte Ressourcengruppe aus. Wählen Sie den PostgreSQL-Server aus und wählen Sie anschließend im Menü die Option **Beenden**.

1. Löschen Sie bei Bedarf das Git Repository, das Sie zuvor geklont haben.

In dieser Übung haben Sie erfahren, wie Sie die Abfrageleistung mithilfe des Abfragespeichers in Azure Database for PostgreSQL bewerten. Außerdem haben Sie erfahren, wie Sie pg_stat Daten anzeigen und Abfragestatistiken untersuchen.
