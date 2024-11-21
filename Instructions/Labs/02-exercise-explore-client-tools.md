---
lab:
  title: Erkunden von PostgreSQL mit Client-Tools
  module: Understand client-server communication in PostgreSQL
---

# Erkunden von PostgreSQL mit Client-Tools

In dieser Übung werden Sie psql und Azure Data Studio herunterladen und installieren. Wenn Sie Azure Data Studio bereits auf Ihrem Rechner installiert haben, können Sie zu Verbinden mit Azure Database for PostgreSQL – Flexibler Server übergehen.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Wenn Sie nicht über ein Azure-Abonnement verfügen, können Sie ein [kostenloses Azure-Konto](https://azure.microsoft.com/free) erstellen.

## Erstellen der Übungsumgebung

In dieser Übung und allen späteren Übungen werden Sie Bicep in der Azure Cloud Shell verwenden, um Ihren PostgreSQL-Server bereitzustellen.

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

> Hinweis
>
> Wenn Sie mehrere Module in diesem Lernpfad absolvieren, können Sie die Azure-Umgebung für alle gemeinsam nutzen. In diesem Fall müssen Sie diesen Schritt der Ressourcenzuteilung nur einmal ausführen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/02-portal-toolbar-cloud-shell.png)

3. Wählen Sie bei Aufforderung die erforderlichen Optionen aus, um eine *Bash*-Shell zu öffnen. Wenn Sie zuvor eine *PowerShell*-Konsole verwendet haben, wechseln Sie zu einer *Bash*-Shell.

4. Geben Sie an der Cloud Shell-Eingabeaufforderung Folgendes ein, um das GitHub-Repository mit den Übungsressourcen zu klonen:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

5. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen (`RG_NAME`), für die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden und für ein zufällig generiertes Kennwort für den PostgreSQL-Administrator-Login (`ADMIN_PASSWORD`).

    Im ersten Befehl ist die Region, die der entsprechenden Variablen zugewiesen ist, `eastus`, aber Sie können sie auch durch einen Ort Ihrer Wahl ersetzen.

    ```bash
    REGION=eastus
    ```

    Mit dem folgenden Befehl weisen Sie den Namen für die Ressourcengruppe zu, die alle in dieser Übung verwendeten Ressourcen enthalten wird. Der Name der Ressourcengruppe, der der entsprechenden Variablen zugewiesen ist, lautet `rg-learn-work-with-postgresql-$REGION`, wobei `$REGION` der Ort ist, den Sie oben angegeben haben. Sie können den Namen jedoch auch in einen anderen Namen für die Ressourcengruppe ändern, der Ihren Wünschen entspricht.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    Der letzte Befehl generiert nach dem Zufallsprinzip ein Kennwort für das PostgreSQL-Admin-Login. Kopieren Sie sie an einen sicheren Ort, damit Sie sie später verwenden können, um eine Verbindung zu Ihrem flexiblen PostgreSQL-Server herzustellen.

    ```bash
    a=()
    for i in {a..z} {A..Z} {0..9}; 
       do
       a[$RANDOM]=$i
    done
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo $ADMIN_PASSWORD
    ```

6. Wenn Sie Zugriff auf mehr als ein Azure-Abonnement haben und Ihr Standardabonnement nicht dasjenige ist, in dem Sie die Ressourcengruppe und andere Ressourcen für diese Übung erstellen möchten, führen Sie diesen Befehl aus, um das entsprechende Abonnement festzulegen. Ersetzen Sie dabei das Token `<subscriptionName|subscriptionId>` durch den Namen oder die ID des Abonnements, das Sie verwenden möchten:

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

7. Führen Sie den folgenden Azure CLI-Befehl aus, um Ihre Ressourcengruppe zu erstellen:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

8. Verwenden Sie schließlich die Azure CLI, um ein Bicep-Bereitstellungsskript auszuführen, um Azure-Ressourcen in Ihrer Ressourcengruppe bereitzustellen:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Das Bicep-Bereitstellungsskript stellt die für diese Übung erforderlichen Azure-Service in Ihrer Ressourcengruppe bereit. Die eingesetzten Ressourcen sind eine Azure-Datenbank für PostgreSQL – Flexibler Server. Das Bicep-Skript erstellt auch eine Datenbank, die in der Befehlszeile als Parameter konfiguriert werden kann.

    Die Bereitstellung dauert in der Regel mehrere Minuten. Sie können es von der Cloud Shell aus überwachen oder zur Seite **Bereitstellungen** für die oben erstellte Ressourcengruppe navigieren und dort den Bereitstellungsfortschritt beobachten.

8. Schließen Sie den Cloud Shell-Bereich, sobald Ihre Ressourcenbereitstellung abgeschlossen ist.

### Problembehandlung bei der Bereitstellung

Beim Ausführen des Bicep-Bereitstellungsskripts können einige Fehler auftreten. Die häufigsten Meldungen und die Schritte zu ihrer Behebung sind:

- Wenn Sie zuvor das Bicep-Bereitstellungsskript für diesen Lernpfad ausgeführt und anschließend die Ressourcen gelöscht haben, erhalten Sie möglicherweise eine Fehlermeldung wie die folgende, wenn Sie versuchen, das Skript innerhalb von 48 Stunden nach dem Löschen der Ressourcen erneut auszuführen:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Wenn Sie diese Meldung erhalten, ändern Sie den obigen Befehl `azure deployment group create`, um den Parameter `restore` auf `true` zu setzen und führen Sie ihn erneut aus.

- Wenn die ausgewählte Region für die Bereitstellung bestimmter Ressourcen eingeschränkt ist, müssen Sie die Variable `REGION` auf einen anderen Speicherort setzen und die Befehle erneut ausführen, um die Ressourcengruppe zu erstellen und das Bicep-Bereitstellungsskript auszuführen.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Wenn das Skript aufgrund der Anforderung, die Vereinbarung über die verantwortungsvolle KI zu akzeptieren, keine KI-Ressource erstellen kann, kann der folgende Fehler auftreten. Verwenden Sie in diesem Fall die Benutzeroberfläche des Azure-Portals, um eine Azure-KI-Services-Ressource zu erstellen und führen Sie das Bereitstellungsskript dann erneut aus.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Client-Tools für die Verbindung zu PostgreSQL

### Herstellen einer Verbindung mit Azure Database for PostgreSQL mit psql

Sie können psql lokal installieren oder eine Verbindung über das Azure-Portal herstellen, das die Cloud Shell öffnet und Sie zur Eingabe des Passworts für das Administratorkonto auffordert.

#### Lokale Verbindung

1. Installieren Sie psql von [hier](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893).
    1. Wählen Sie im Setup-Assistenten **Befehlszeilentools** aus, wenn Sie das Dialogfeld **Komponenten auswählen** erreichen.
    > Hinweis
    >
    > Um zu überprüfen, ob **psql** bereits in Ihrer Umgebung installiert ist, öffnen Sie die Befehlszeile/Terminal und führen Sie den Befehl ***psql*** aus. Wenn Sie eine Meldung wie „*psql: Fehler: Verbindung zum Server auf Socket …*“ erhalten, bedeutet dies, dass das Tool **psql** bereits in Ihrer Umgebung installiert ist und Sie es nicht erneut installieren müssen.



1. Rufen Sie eine Befehlszeile auf.
1. Die Syntax für die Verbindung mit dem Server lautet:

    ```sql
    psql --h <servername> --p <port> -U <username> <dbname>
    ```

1. Geben Sie an der Eingabeaufforderung **`--host=<servername>.postgres.database.azure.com`** ein, wobei `<servername>` der Name der oben erstellten Azure-Datenbank für PostgreSQL ist.
    1. Sie finden den Servernamen unter **Übersicht** im Azure-Portal oder als Ausgabe des Bicep-Skripts.

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin postgres
    ```

    1. Sie werden aufgefordert, das Kennwort für das oben kopierte Administratorkonto einzugeben.

1. Geben Sie Folgendes ein, um über die Eingabeaufforderung eine leere Datenbank zu erstellen:

    ```sql
    CREATE DATABASE mypgsqldb;
    ```

1. Führen Sie an der Eingabeaufforderung den folgenden Befehl zum Wechseln der Verbindung zur neu erstellten Datenbank **mypgsqldb** aus:

    ```sql
    \c mypgsqldb
    ```

1. Nachdem Sie eine Verbindung mit dem Server hergestellt und eine Datenbank erstellt haben, können Sie vertraute SQL-Abfragen ausführen, z. B. „create tables“ in der Datenbank:

    ```sql
    CREATE TABLE inventory (
        id serial PRIMARY KEY,
        name VARCHAR(50),
        quantity INTEGER
        );
    ```

1. Laden von Daten in die Tabellen

    ```sql
    INSERT INTO inventory (id, name, quantity) VALUES (1, 'banana', 150);
    INSERT INTO inventory (id, name, quantity) VALUES (2, 'orange', 154);
    ```

1. Abfragen und Aktualisieren der Daten in den Tabellen

    ```sql
    SELECT * FROM inventory;
    ```

1. Aktualisieren Sie die Daten in den Tabellen.

    ```sql
    UPDATE inventory SET quantity = 200 WHERE name = 'banana';
    ```

## Azure Data Studio installieren

> Hinweis
>
> Wenn Azure Data Studio bereits installiert ist, gehen Sie zum Schritt *Installieren der PostgreSQL-Erweiterung*.

So installieren Sie Azure Data Studio für Azure Database for PostgreSQL:

1. Navigieren Sie in einem Browser zum [Azure Data Studio herunterladen und installieren](https://go.microsoft.com/fwlink/?linkid=2282284), und klicken Sie unter der Windows-Plattform auf **User installer (recommended)** (Benutzerinstallationsprogramm (empfohlen)). Die ausführbare Datei wird in Ihren Ordner „Downloads“ heruntergeladen.
1. Klicken Sie auf **Datei öffnen**.
1. Nun wird die Lizenzvereinbarung angezeigt. Lesen und **akzeptieren Sie die Vereinbarung**, und klicken sie auf **Weiter**.
1. Wählen Sie unter **Weitere Aufgaben auswählen** die Option **Zu PATH hinzufügen** sowie und alle weiteren erforderlichen Ergänzungen aus. Wählen Sie **Weiter** aus.
1. Das Dialogfeld **Bereit für die Installation** wird angezeigt. Überprüfen Sie Ihre Einstellungen. Klicken Sie auf **Zurück**, um Änderungen vorzunehmen, oder klicken Sie auf **Installieren**.
1. Das Dialogfeld **Completing the Azure Data Studio Setup Wizard** (Der Setup-Assistent für Azure Data Studio wird abgeschlossen) wird angezeigt. Wählen Sie **Fertig stellen** aus. Azure Data Studio wird gestartet.

## Installieren der PostgreSQL-Erweiterung

1. Öffnen Sie Azure Data Studio, falls das Programm noch nicht geöffnet ist.
2. Klicken Sie im linken Menü auf **Erweiterungen**, um den Bereich „Erweiterungen“ anzuzeigen.
3. Geben Sie **Subnetze** in die Suchleiste ein. Das Symbol der PostgreSQL-Erweiterung für Azure Data Studio wird angezeigt.
   
![Screenshot der PostgreSQL-Erweiterung für Azure Data Studio](media/02-postgresql-extension.png)
   
4. Wählen Sie **Installieren** aus. Die Erweiterung wird installiert.

## Verbinden mit Azure Database for PostgreSQL – Flexibler Server

1. Öffnen Sie Azure Data Studio, falls das Programm noch nicht geöffnet ist.
2. Klicken Sie im linken Menü auf **Verbindungen**.
   
![Screenshot zeigt die Verbindungen in Azure Data Studio](media/02-connections.png)

3. Wählen Sie **Neue Verbindung** aus.
   
![Screenshot der Erstellung einer neuen Verbindung in Azure Data Studio](media/02-create-connection.png)

4. Wählen Sie unter **Details zur Verbindung** unter **Verbindungstyp** die Option **PostgreSQL** aus der Dropdownliste aus.
5. Geben Sie unter **Servername** den vollständigen Servernamen ein, so wie er im Azure-Portal angezeigt wird.
6. Behalten Sie unter **Authentifizierungstyp** „Kennwort“ bei.
7. Geben Sie unter Benutzername und Kennwort den Benutzernamen **pgAdmin** und das Kennwort **das zufällige Admin-Kennwort** ein, das Sie oben erstellt haben
8. Klicken Sie auf [ x ] Kennwort merken.
9. Die verbleibenden Felder sind optional.
10. Wählen Sie **Verbinden**. Sie sind mit dem Azure Database for PostgreSQL-Server verbunden.
11. Eine Liste der Serverdatenbanken wird angezeigt. Dazu gehören Systemdatenbanken und Benutzerdatenbanken.

## Erstellen der Zoodatenbank

1. Navigieren Sie entweder zu dem Ordner mit Ihren Übungsskriptdateien oder laden Sie die **Lab2_ZooDb.sql** von [MSLearn PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/02) herunter.
1. Öffnen Sie Azure Data Studio, falls das Programm noch nicht geöffnet ist.
1. Wählen Sie **Datei**, **Datei öffnen** und navigieren Sie zu dem Ordner, in dem Sie die Skripte gespeichert haben. Wählen Sie **../Allfiles/Labs/02/Lab2_ZooDb.sql** und **Öffnen**.
   1. Markieren Sie die Anweisungen **DROP** und **CREATE** und führen Sie sie aus.
   1. Zeigen Sie oben im Bildschirm mithilfe des Dropdownpfeils die Datenbanken auf dem Server an, einschließlich zoodb und Systemdatenbanken. Wählen Sie die Datenbank **zoodb**.
   1. Markieren Sie die Abschnitte **Tabellen erstellen**, **Fremdschlüssel erstellen** und **Tabellen befüllen** und führen Sie sie aus.
   1. Markieren Sie die 3 **SELECT**-Anweisungen am Ende des Skripts und führen Sie sie aus, um zu überprüfen, ob die Tabellen erstellt und aufgefüllt wurden.
