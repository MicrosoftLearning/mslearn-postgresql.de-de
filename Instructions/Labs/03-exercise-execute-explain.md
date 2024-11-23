---
lab:
  title: Ausführen der EXPLAIN-Anweisung
  module: Understand PostgreSQL query processing
---

# Ausführen der EXPLAIN-Anweisung

In dieser Übung sehen Sie sich die EXPLAIN-Funktion an und zeigen den Ausführungsplan an, den der PostgreSQL-Plan für eine bereitgestellte Anweisung generiert.

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

    ![Screenshot der Azure-Symbolleiste mit dem Cloud Shell-Symbol, das durch eine rote Box hervorgehoben ist.](media/03-portal-toolbar-cloud-shell.png)

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
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Wenn das Skript aufgrund der Anforderung, die Vereinbarung über die verantwortungsvolle KI zu akzeptieren, keine KI-Ressource erstellen kann, kann der folgende Fehler auftreten. Verwenden Sie in diesem Fall die Benutzeroberfläche des Azure-Portals, um eine Azure-KI-Services-Ressource zu erstellen und führen Sie das Bereitstellungsskript dann erneut aus.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Ehe Sie fortfahren

Stellen Sie sicher, dass Folgendes zutrifft:

1. Sie haben Azure Database for PostgreSQL – Flexibler Server installiert und gestartet. Dies sollte durch das vorherige Bicep-Skript installiert worden sein.
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
1. Öffnen Sie Azure Data Studio und stellen Sie eine Verbindung zu Ihrem Azure Database for PostgreSQL – Flexibler Server her, der mit dem Bicep-Skript erstellt wurde. Geben Sie den Benutzernamen **pgAdmin** und das Kennwort **das zufällige Admin-Kennwort** ein, das Sie zuvor erstellt haben.
1. Wenn Sie die zoodb-Datenbank noch nicht erstellt haben, wählen Sie **Datei**, **Datei öffnen** und navigieren Sie zu dem Ordner, in dem Sie die Skripts gespeichert haben. Wählen Sie **../Allfiles/Labs/02/Lab2_ZooDb.sql** und **Öffnen**.
   1. Markieren Sie die Anweisungen **DROP** und **CREATE** und führen Sie sie aus.
   1. Zeigen Sie oben im Bildschirm mithilfe des Dropdownpfeils die Datenbanken auf dem Server an, einschließlich zoodb und Systemdatenbanken. Wählen Sie die Datenbank **zoodb**.
   1. Markieren Sie die Abschnitte **Tabellen erstellen**, **Fremdschlüssel erstellen** und **Tabellen befüllen** und führen Sie sie aus.
   1. Markieren Sie die 3 **SELECT**-Anweisungen am Ende des Skripts und führen Sie sie aus, um zu überprüfen, ob die Tabellen erstellt und aufgefüllt wurden.

## Verwenden von EXPLAIN ANALYZE

1. Navigieren Sie im [Azure-Portal](https://portal.azure.com) zu Ihrer Instanz von Azure Database for PostgreSQL Flexible Server. Überprüfen Sie, ob der Server gestartet wurde, oder starten Sie ihn bei Bedarf neu.
1. Öffnen Sie Azure Data Studio, und stellen Sie eine Verbindung mit Ihrer Instanz von Azure Database for PostgreSQL Flexible Server her.
1. Klicken Sie auf **Datei**, anschließend auf **Datei öffnen**, und navigieren Sie zu dem Ordner, in dem Sie die Skripts gespeichert haben. Öffnen Sie **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql**. Stellen Sie bei Bedarf erneut eine Verbindung mit dem Server her.
1. Klicken Sie auf **Ausführen**, um die Abfrage auszuführen. Dadurch wird die Datenbank „zoodb“ neu aufgefüllt.
1. Wählen Sie Datei, **Datei öffnen** und wählen Sie **../Allfiles/Labs/03/Lab3_explain.sql**.
1. In der Lab-Datei, im Abschnitt **1. Untersuchen Sie EXPLAIN ANALYZE** und führen Sie Anweisung A und Anweisung B getrennt aus.
    1. Welche Anweisung hat die Datenbank aktualisiert und warum?
    1. Wie viele Millisekunden dauerte es, um die Anweisung A zu planen?
    1. Welche Ausführungszeit hat Anweisung B?

## Verwenden von EXPLAIN

1. Markieren Sie in der Lab-Datei im Abschnitt **2. Untersuchen Sie EXPLAIN** und führen Sie diese Anweisung aus.
    1. Welcher Sortierschlüssel wurde verwendet und warum?
1. Markieren Sie in der Lab-Datei im Abschnitt **3. Untersuchen Sie die EXPLAIN-Optionen** und führen Sie jede Anweisung einzeln aus. Vergleichen Sie die Abfrageplanstatistiken für jede Option.

## Bereinigung

1. Löschen Sie die in dieser Übung erstellte Ressourcengruppe, um unnötige Azure-Kosten zu vermeiden.
1. Löschen Sie bei Bedarf den Ordner **.\DP3021Lab**.

