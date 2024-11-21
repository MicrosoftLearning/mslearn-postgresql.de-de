---
lab:
  title: Erstellen einer gespeicherten Prozedur in Azure Database for PostgreSQL
  module: Procedures and functions in PostgreSQL
---

# Erstellen einer gespeicherten Prozedur in Azure Database for PostgreSQL

In dieser Übung werden Sie eine gespeicherte Prozedur erstellen.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Wenn Sie nicht über ein Azure-Abonnement verfügen, können Sie ein [kostenloses Azure-Konto](https://azure.microsoft.com/free) erstellen.

## Erstellen der Übungsumgebung

In dieser Übung und allen späteren Übungen werden Sie Bicep in der Azure Cloud Shell verwenden, um Ihren PostgreSQL-Server bereitzustellen.
Überspringen Sie das Bereitstellen von Ressourcen und die Installation von Azure Data Studio, wenn Sie diese bereits installiert haben.

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus der Azure Cloud Shell, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für diese Übung erforderlichen Azure-Services in Ihrem Azure-Abonnement bereitzustellen.

> Hinweis
>
> Wenn Sie mehrere Module in diesem Lernpfad absolvieren, können Sie die Azure-Umgebung für alle gemeinsam nutzen. In diesem Fall müssen Sie diesen Schritt der Ressourcenzuteilung nur einmal ausführen.

1. Öffnen Sie einen Webbrowser, und navigieren Sie zum [Azure-Portal](https://portal.azure.com/).

2. Wählen Sie das Symbol **Cloud Shell** in der Symbolleiste des Azure-Portals aus, um einen neuen [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)-Bereich am unteren Rand Ihres Browserfensters zu öffnen.

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/05-portal-toolbar-cloud-shell.png)

    Wählen Sie bei Aufforderung die erforderlichen Optionen aus, um eine *Bash*-Shell zu öffnen. Wenn Sie zuvor eine *PowerShell*-Konsole verwendet haben, wechseln Sie zu einer *Bash*-Shell.

3. Geben Sie an der Cloud Shell-Eingabeaufforderung Folgendes ein, um das GitHub-Repository mit den Übungsressourcen zu klonen:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen (`RG_NAME`), für die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden und für ein zufällig generiertes Kennwort für den PostgreSQL-Administrator-Login (`ADMIN_PASSWORD`).

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

5. Wenn Sie Zugriff auf mehr als ein Azure-Abonnement haben und Ihr Standardabonnement nicht dasjenige ist, in dem Sie die Ressourcengruppe und andere Ressourcen für diese Übung erstellen möchten, führen Sie diesen Befehl aus, um das entsprechende Abonnement festzulegen. Ersetzen Sie dabei das Token `<subscriptionName|subscriptionId>` durch den Namen oder die ID des Abonnements, das Sie verwenden möchten:

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. Führen Sie den folgenden Azure CLI-Befehl aus, um Ihre Ressourcengruppe zu erstellen:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. Verwenden Sie schließlich die Azure CLI, um ein Bicep-Bereitstellungsskript auszuführen, um Azure-Ressourcen in Ihrer Ressourcengruppe bereitzustellen:

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

## Klonen des GitHub Repositorys lokal

Stellen Sie sicher, dass Sie die Lab-Skripte aus [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git) bereits geklont haben. Falls Sie dies noch nicht getan haben, klonen Sie das Repository lokal:

1. Öffnen Sie eine Befehlszeile/ein Terminal.
1. Führen Sie den folgenden Befehl aus:
    ```bash
    md .\DP3021Lab
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
    ```
    > HINWEIS
    > 
    > Wenn **git** nicht installiert ist, [laden Sie die App ***git*** herunter und installieren Sie sie](https://git-scm.com/download) und versuchen Sie, die vorherigen Befehle erneut auszuführen.

## Azure Data Studio installieren

Wenn Sie Azure Data Studio nicht installiert haben:

1. Navigieren Sie in einem Browser zum [Azure Data Studio herunterladen und installieren](/sql/azure-data-studio/download-azure-data-studio), und klicken Sie unter der Windows-Plattform auf **User installer (recommended)** (Benutzerinstallationsprogramm (empfohlen)). Die ausführbare Datei wird in Ihren Ordner „Downloads“ heruntergeladen.
1. Klicken Sie auf **Datei öffnen**.
1. Nun wird die Lizenzvereinbarung angezeigt. Lesen und **akzeptieren Sie die Vereinbarung**, und klicken sie auf **Weiter**.
1. Wählen Sie unter **Weitere Aufgaben auswählen** die Option **Zu PATH hinzufügen** sowie und alle weiteren erforderlichen Ergänzungen aus. Wählen Sie **Weiter** aus.
1. Das Dialogfeld **Bereit für die Installation** wird angezeigt. Überprüfen Sie Ihre Einstellungen. Klicken Sie auf **Zurück**, um Änderungen vorzunehmen, oder klicken Sie auf **Installieren**.
1. Das Dialogfeld **Completing the Azure Data Studio Setup Wizard** (Der Setup-Assistent für Azure Data Studio wird abgeschlossen) wird angezeigt. Wählen Sie **Fertig stellen** aus. Azure Data Studio wird gestartet.

## Installieren der PostgreSQL-Erweiterung

Wenn Sie die PostgreSQL-Erweiterung nicht in Ihrem Azure Data Studio installiert haben:

1. Öffnen Sie Azure Data Studio, falls das Programm noch nicht geöffnet ist.
1. Klicken Sie im linken Menü auf **Erweiterungen**, um den Bereich „Erweiterungen“ anzuzeigen.
1. Geben Sie **Subnetze** in die Suchleiste ein. Das Symbol der PostgreSQL-Erweiterung für Azure Data Studio wird angezeigt.
1. Wählen Sie **Installieren** aus. Die Erweiterung wird installiert.

## Verbinden mit Azure Database for PostgreSQL – Flexibler Server

1. Öffnen Sie Azure Data Studio, falls das Programm noch nicht geöffnet ist.
1. Klicken Sie im linken Menü auf **Verbindungen**.
1. Klicken Sie auf **Neue Verbindung**.
1. Wählen Sie unter **Details zur Verbindung** unter **Verbindungstyp** die Option **PostgreSQL** aus der Dropdownliste aus.
1. Geben Sie unter **Servername** den vollständigen Servernamen ein, so wie er im Azure-Portal angezeigt wird.
1. Behalten Sie unter **Authentifizierungstyp** „Kennwort“ bei.
1. Geben Sie unter Benutzername und Kennwort den Benutzernamen **pgAdmin** und das Kennwort **das zufällige Admin-Kennwort** ein, das Sie oben erstellt haben
1. Klicken Sie auf [ x ] Kennwort merken.
1. Die verbleibenden Felder sind optional.
1. Wählen Sie **Verbinden**. Sie sind mit dem Azure Database for PostgreSQL-Server verbunden.
1. Eine Liste der Serverdatenbanken wird angezeigt. Dazu gehören Systemdatenbanken und Benutzerdatenbanken.
1. Wenn Sie die zoodb-Datenbank noch nicht erstellt haben, wählen Sie **Datei**, **Datei öffnen** und navigieren Sie zu dem Ordner, in dem Sie die Skripts gespeichert haben. Wählen Sie **../Allfiles/Labs/02/Lab2_ZooDb.sql** und **Öffnen**.
   1. Markieren Sie die Anweisungen **DROP** und **CREATE** und führen Sie sie aus.
   1. Zeigen Sie oben im Bildschirm mithilfe des Dropdownpfeils die Datenbanken auf dem Server an, einschließlich zoodb und Systemdatenbanken. Wählen Sie die Datenbank **zoodb**.
   1. Markieren Sie die Abschnitte **Tabellen erstellen**, **Fremdschlüssel erstellen** und **Tabellen befüllen** und führen Sie sie aus.
   1. Markieren Sie die 3 **SELECT**-Anweisungen am Ende des Skripts und führen Sie sie aus, um zu überprüfen, ob die Tabellen erstellt und aufgefüllt wurden.

## Erstellen Sie die gespeicherte Prozedur repopulate_zoo()

1. Legen Sie im oberen Teil des Bildschirms in der Dropdownliste zoodb als aktuelle Datenbank fest.
1. Wählen Sie in Azure Data Studio **Datei** und dann **Datei öffnen** aus, und navigieren Sie danach zu den Skripts für das Lab. Wählen Sie **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql** und wählen Sie dann **Öffnen**. Stellen Sie bei Bedarf erneut eine Verbindung mit dem Server her.
1. Markieren Sie den Abschnitt unter **Erstellen einer gespeicherten Prozedur** von **DROP PROCEDURE** bis **END $$.** Führen Sie den markierten Text aus.
1. Halten Sie Azure Data Studio mit der Datei geöffnet, um für die nächste Übung bereit zu sein.

## Erstellen der gespeicherten new_exhibit() Prozedur

1. Legen Sie im oberen Teil des Bildschirms in der Dropdownliste zoodb als aktuelle Datenbank fest.
1. Wählen Sie in Azure Data Studio **Datei** und dann **Datei öffnen** aus, und navigieren Sie danach zu den Skripts für das Lab. Wählen Sie **../Allfiles/Labs/05/Lab5_StoredProcedure.sql** und wählen Sie dann **Öffnen**. Stellen Sie bei Bedarf erneut eine Verbindung mit dem Server her.
1. Markieren Sie den Abschnitt unter **Erstellen einer gespeicherten Prozedur** von **DROP PROCEDURE** bis **END $$.** Führen Sie den markierten Text aus. Lesen Sie sich die Prozedur durch. Sie werden sehen, dass einige Eingabeparameter deklariert und verwendet werden, um Zeilen in die Tabelle der Gehege und die Tabelle der Tiere einzufügen.
1. Halten Sie Azure Data Studio mit der Datei geöffnet, um für die nächste Übung bereit zu sein.

## Rufen Sie die gespeicherte Prozedur

1. Markieren Sie den Abschnitt unter **Call the stored procedure**. Führen Sie den markierten Text aus. Dadurch wird die gespeicherte Prozedur aufgerufen, indem Werte an die Eingabeparameter übergeben werden.
1. Markieren Sie die beiden **SELECT**-Anweisungen, und führen Sie sie aus. Führen Sie den markierten Text aus. Sie können sehen, dass eine neue Zeile in die Tabelle für Gehege und fünf neue Zeilen in die Tabelle für Tiere eingefügt worden sind.

## Erstellen und Aufrufen einer Tabellenwertfunktion

1. Wählen Sie in Azure Data Studio **Datei** und dann **Datei öffnen** aus, und navigieren Sie danach zu den Skripts für das Lab. Wählen Sie **../Allfiles/Labs/05/Lab5_Table_Function.sql** und wählen Sie dann **Öffnen**.
1. Markieren Sie die erste **SELECT**-Anweisung, um zu prüfen, ob die Datenbank zoodb ausgewählt ist.
1. Markieren Sie die gespeicherte Prozedur **repopulate_zoo()**, und führen Sie sie aus, um mit frischen Daten zu beginnen.
1. Markieren Sie den Abschnitt unter **Create a table valued function**, und führen Sie ihn aus. Diese Funktion gibt eine Tabelle namens **enclosure_summary** zurück. Lesen Sie sich den Funktionscode durch, um zu verstehen, wie die Tabelle aufgefüllt wird.
1. Markieren Sie die beiden ausgewählten Anweisungen, und führen Sie sie aus, wobei Sie jedes Mal eine andere Gehege-ID eingeben.
1. Markieren Sie den Abschnitt unter **How to use a table valued function with a LATERAL join**, und führen Sie ihn aus. Hier sehen Sie, wie die Tabellenwertfunktion anstelle eines Tabellennamens in einem Join verwendet wird.

## Optionale Übung: integrierte Funktionen

1. Wählen Sie in Azure Data Studio **Datei** und dann **Datei öffnen** aus, und navigieren Sie danach zu den Skripts für das Lab. Wählen Sie **../Allfiles/Labs/05/Lab5_SimpleFunctions.sql** und wählen Sie dann **Öffnen**.
1. Markieren Sie die einzelnen Funktionen, und führen Sie sie aus, um zu ermitteln, wie sie funktionieren. Weitere Informationen zu den einzelnen Funktionen finden Sie in der [Onlinedokumentation](https://www.postgresql.org/docs/current/functions.html).
1. Schließen Sie Azure Data Studio, ohne die Skripts zu speichern.
1. Halten Sie Ihren Azure Database for PostgreSQL-Server an, damit Ihnen keine Gebühren in Rechnung gestellt werden, wenn Sie ihn nicht nutzen.

## Bereinigung

1. Löschen Sie die in dieser Übung erstellte Ressourcengruppe, um unnötige Azure-Kosten zu vermeiden.
1. Löschen Sie bei Bedarf den Ordner .\DP3021Lab.

