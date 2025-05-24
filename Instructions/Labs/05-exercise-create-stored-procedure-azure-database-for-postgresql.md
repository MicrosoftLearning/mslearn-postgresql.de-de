---
lab:
  title: Erstellen einer gespeicherten Prozedur in Azure Database for PostgreSQL
  module: Procedures and functions in PostgreSQL
---

# Erstellen einer gespeicherten Prozedur in Azure Database for PostgreSQL

In dieser Übung erstellen Sie einige gespeicherte Prozeduren und führen sie aus.

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

### Bereitstellen von Ressourcen in Ihrem Azure-Abonnement

Wenn Ihre Azure-Ressourcen bereits installiert sind, *überspringen Sie die Bereitstellung von Ressourcen*.

Dieser Schritt führt Sie durch die Verwendung von Azure CLI-Befehlen aus Visual Studio Code, um eine Ressourcengruppe zu erstellen und ein Bicep-Skript auszuführen, um die für die Durchführung dieser Übung erforderlichen Azure-Dienste in Ihrem Azure-Abonnement bereitzustellen.

> &#128221; Wenn Sie mehrere Module in diesem Lernpfad absolvieren, können Sie die Azure-Umgebung für alle Module gemeinsam nutzen. In diesem Fall müssen Sie diesen Schritt der Ressourcenzuteilung nur einmal ausführen.

1. Öffnen Sie Visual Studio Code, falls es noch nicht geöffnet ist, und öffnen Sie den Ordner, in den Sie das GitHub-Repository geklont haben.

1. Erweitern Sie den Ordner **mslearn-postgresql** im Explorer-Fenster.

1. Erweitern Sie den Ordner **Allfiles/Labs/Shared**.

1. Klicken Sie mit der rechten Maustaste auf den Ordner **Allfiles/Labs/Shared** und wählen Sie **Im integrierten Terminal öffnen**. Diese Auswahl öffnet ein Terminalfenster im Visual Studio Code-Fenster.

1. Das Terminal kann standardmäßig ein **PowerShell-Fenster** öffnen. Für diesen Abschnitt der Übung möchten Sie die **Bash-Shell** verwenden. Neben dem Symbol **+** befindet sich ein Dropdown-Pfeil. Wählen Sie diese Option aus und wählen Sie anschließend **Git Bash** oder **Bash** aus der Liste der verfügbaren Profile. Diese Auswahl öffnet ein neues Terminalfenster mit der **Bash-Shell**.

    > &#128221; Sie können das **PowerShell-Terminalfenster** schließen, wenn Sie möchten, aber es ist nicht erforderlich. Sie können mehrere Terminalfenster gleichzeitig geöffnet haben.

1. Führen Sie im Terminalfenster den folgenden Befehl aus, um sich bei Ihrem Azure-Konto anzumelden:

    ```bash
    az login
    ```

    Dieser Befehl öffnet ein neues Browserfenster, in dem Sie aufgefordert werden, sich bei Ihrem Azure-Konto anzumelden. Kehren Sie nach der Anmeldung zum Terminalfenster zurück.

1. Als Nächstes führen Sie drei Befehle aus, um Variablen zu definieren und so die redundante Eingabe zu reduzieren, wenn Sie Azure-CLI-Befehle zum Erstellen von Azure-Ressourcen verwenden. Die Variablen stehen für den Namen, den Sie Ihrer Ressourcengruppe zuweisen möchten (`RG_NAME`), die Azure-Region (`REGION`), in der die Ressourcen bereitgestellt werden, und ein zufällig generiertes Kennwort für den PostgreSQL-Admin-Login (`ADMIN_PASSWORD`).

    Im ersten Befehl ist die Region, die der entsprechenden Variablen zugewiesen ist, `eastus`, aber Sie können sie auch durch einen Ort Ihrer Wahl ersetzen.

    ```bash
    REGION=eastus
    ```

    Der folgende Befehl weist den Namen zu, der für die Ressourcengruppe verwendet werden soll, die alle in dieser Übung verwendeten Ressourcen enthält. Der Name der Ressourcengruppe, die der entsprechenden Variablen zugewiesen wird, lautet `rg-learn-work-with-postgresql-$REGION`, wobei `$REGION` der zuvor angegebene Speicherort ist. *Sie können jedoch einen beliebigen anderen Ressourcengruppennamen verwenden, der Ihren Anforderungen entspricht oder den Sie bereits verwenden.*

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

1. (Überspringen, wenn Sie Ihr Standardabonnement verwenden). Wenn Sie Zugriff auf mehr als ein Azure-Abonnement haben und Ihr Standardabonnement *nicht* das Abonnement ist, in dem Sie die Ressourcengruppe und andere Ressourcen für diese Übung erstellen möchten, führen Sie diesen Befehl aus, um das entsprechende Abonnement festzulegen. Ersetzen Sie dabei das Token `<subscriptionName|subscriptionId>` durch den Namen oder die ID des Abonnements, das Sie verwenden möchten:

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. (Überspringen Sie diesen Schritt, wenn Sie bereits eine Ressourcengruppe verwenden.) Führen Sie den folgenden Azure CLI-Befehl aus, um Ihre Ressourcengruppe zu erstellen:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. Verwenden Sie schließlich die Azure CLI, um ein Bicep-Bereitstellungsskript auszuführen, um Azure-Ressourcen in Ihrer Ressourcengruppe bereitzustellen:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Das Bicep-Bereitstellungsskript stellt die für diese Übung erforderlichen Azure-Service in Ihrer Ressourcengruppe bereit. Die eingesetzten Ressourcen sind eine Azure-Datenbank für PostgreSQL – Flexibler Server. Das Bicep-Skript erstellt auch eine Datenbank, die in der Befehlszeile als Parameter konfiguriert werden kann.

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

## Verbinden mit der PostgreSQL-Erweiterung in Visual Studio Code

In diesem Abschnitt stellen Sie eine Verbindung mit dem PostgreSQL-Server mithilfe der PostgreSQL-Erweiterung in Visual Studio Code her. Sie verwenden die PostgreSQL-Erweiterung, um SQL-Skripts auf dem PostgreSQL-Server auszuführen.

1. Öffnen Sie Visual Studio Code, falls es noch nicht geöffnet ist, und öffnen Sie den Ordner, in den Sie das GitHub-Repository geklont haben.

1. Wählen Sie im linken Menü das **PostgreSQL-Symbol** aus.

    > &#128221; Wenn Sie das PostgreSQL-Symbol nicht sehen, wählen Sie das Symbol **Erweiterungen** und suchen Sie nach **PostgreSQL**. Wählen Sie die Erweiterung **PostgreSQL** von Microsoft aus und klicken Sie auf **Installieren**.

1. Wenn Sie bereits eine Verbindung mit Ihrem PostgreSQL-Server erstellt haben, fahren Sie mit dem nächsten Schritt fort. So erstellen Sie eine neue Verbindung:

    1. Wählen Sie in der Erweiterung **PostgreSQL** die Option **+ Verbindung hinzufügen**, um eine neue Verbindung hinzuzufügen.

    1. Geben Sie im Dialogfeld **NEUE VERBINDUNG** die folgenden Informationen ein:

        - **Servername**: <Ihr-Server-Name>.postgres.database.azure.com
        - **Authentifizierungstyp**: Kennwort
        - **Benutzername**: pgAdmin
        - **Kennwort**: Das zufällige Kennwort, das Sie zuvor generiert haben.
        - Aktivieren Sie das Kontrollkästchen **Kennwort speichern**.
        - **Verbindungsname**: <Ihr-Server-Name>

    1. Überprüfen Sie die Verbindung, indem Sie **Verbindung testen** auswählen. Wenn die Verbindung erfolgreich hergestellt wurde, wählen Sie **Speichern und verbinden**, um die Verbindung zu speichern. Andernfalls überprüfen Sie die Verbindungsinformationen und versuchen Sie es erneut.

1. Falls noch keine Verbindung besteht, wählen Sie für Ihren PostgreSQL-Server die Option **Verbinden**. Sie sind nun mit dem Azure Database for PostgreSQL-Server verbunden.

1. Erweitern Sie den Serverknoten und die zugehörigen Datenbanken. Die vorhandenen Datenbanken werden aufgelistet.

1. Falls Sie die zoodb-Datenbank noch nicht erstellt haben, wählen Sie **Datei**, **Datei öffnen** und navigieren Sie zu dem Ordner, in dem Sie die Skripte gespeichert haben. Wählen Sie **../Allfiles/Labs/02/Lab2_ZooDb.sql** und **Öffnen**.

1. Vergewissern Sie sich unten rechts in Visual Studio Code, dass die Verbindung in Grün angezeigt wird. Ist dies nicht der Fall, sollte **PGSQL Disconnected** angezeigt werden. Wählen Sie den Text **PGSQL Disconnected** aus und wählen Sie anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette aus. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

    > &#128221; Sie können die Datenbank auch im Abfragebereich ändern. Sie können den Servernamen und den Datenbanknamen unter der Abfrageregisterkarte selbst notieren. Durch Auswahl des Datenbanknamens wird eine Liste der Datenbanken angezeigt. Wählen Sie die Datenbank `zoodb` aus der Liste aus.

1. Es ist an der Zeit, die Datenbank zu erstellen.

    1. Markieren Sie die Anweisungen **DROP** und **CREATE** und führen Sie sie aus.

    1. Wenn Sie nur die Anweisung **SELECT current_database()** markieren und ausführen, werden Sie feststellen, dass die Datenbank derzeit auf `postgres` festgelegt ist. Sie müssen sie in `zoodb` ändern.

    1. Wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus.

    1. Führen Sie die Anweisung **SELECT current_database()** erneut aus, um zu bestätigen, dass die Datenbank nun auf `zoodb` festgelegt ist.

    1. Markieren Sie die Abschnitte **Erstellte Tabellen**, **Fremdschlüssel erstellen** und **Tabellen befüllen** und führen Sie sie aus.

    1. Markieren Sie die 3 **SELECT**-Anweisungen am Ende des Skripts und führen Sie sie aus, um zu überprüfen, ob die Tabellen erstellt und aufgefüllt wurden.

## Erstellen Sie die gespeicherte Prozedur repopulate_zoo()

In diesem Abschnitt erstellen Sie die gespeicherte Prozedur `repopulate_zoo()`. Dieses Verfahren wird verwendet, um die Zoodatenbank mit Daten erneut zu befüllen. Der Vorgang kürzt alle Daten in den Tabellen und löscht sie anschließend. Anschließend werden die Tabellen mit neuen Daten befüllt.

1. Wählen Sie im Visual Studio Code-Fenster **Datei**, **Datei öffnen** und navigieren Sie dann zu den Lab-Skripten. Wählen Sie **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql** und wählen Sie dann **Öffnen**. Stellen Sie gegebenenfalls die Verbindung zum Server wieder her, indem Sie den Text **PGSQL Disconnected** auswählen und anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette auswählen. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

1. Führen Sie die Anweisung **SELECT current_database()** aus, um die aktuelle Datenbank zu überprüfen. Auch hier ist die Datenbank höchstwahrscheinlich auf `postgres` festgelegt. Ist dies der Fall, müssen Sie den Wert auf `zoodb` ändern. Wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus. Überprüfen Sie die Verbindung erneut, indem Sie die Anweisung **SELECT current_database()** ausführen.

1. Markieren Sie den Abschnitt unter **Erstellen einer gespeicherten Prozedur** von **DROP PROCEDURE** bis **END $$.** Führen Sie den markierten Text aus.

1. Lassen Sie Visual Studio Code geöffnet, um mit dem nächsten Abschnitt fortzufahren.

## Erstellen der gespeicherten new_exhibit() Prozedur

In diesem Abschnitt erstellen Sie die gespeicherte Prozedur `new_exhibit()`. Dieses Verfahren wird verwendet, um der Zoodatenbank eine neue Ausstellung hinzuzufügen. Die Prozedur fügt eine neue Zeile in die Gehegetabelle ein und fügt anschließend für jedes Tier in der Ausstellung Zeilen in die Tiertabelle ein.

1. Wählen Sie in Visual Studio Code **Datei**, **Datei öffnen** und navigieren Sie dann zu den Lab-Skripten. Wählen Sie **../Allfiles/Labs/05/Lab5_StoredProcedure.sql** und wählen Sie dann **Öffnen**. Stellen Sie gegebenenfalls die Verbindung zum Server wieder her, indem Sie den Text **PGSQL Disconnected** auswählen und anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette auswählen. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

1. Führen Sie die Anweisung **SELECT current_database()** aus, um die aktuelle Datenbank zu überprüfen. Auch hier ist die Datenbank höchstwahrscheinlich auf `postgres` festgelegt. Ist dies der Fall, müssen Sie den Wert auf `zoodb` ändern. Wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus. Überprüfen Sie die Verbindung erneut, indem Sie die Anweisung **SELECT current_database()** ausführen.

1. Markieren Sie die Anweisung **CALL repopulate_zoo()** und führen Sie sie aus, um mit dem Bereinigen der Daten zu beginnen.

1. Markieren Sie den Abschnitt unter **Erstellen einer gespeicherten Prozedur** von **DROP PROCEDURE** bis **END $$.** Führen Sie den markierten Text aus. Lesen Sie sich die Prozedur durch. Sie werden sehen, dass einige Eingabeparameter deklariert und verwendet werden, um Zeilen in die Tabelle der Gehege und die Tabelle der Tiere einzufügen.

1. Lassen Sie Visual Studio Code geöffnet, um mit dem nächsten Abschnitt fortzufahren.

## Rufen Sie die gespeicherte Prozedur

Nachdem Sie die gespeicherte Prozedur `new_exhibit()` erstellt haben, können Sie diese aufrufen, um ein neues Exponat zur Zoobank-Datenbank hinzuzufügen. Das Verfahren verwendet mehrere Eingabeparameter, darunter den Namen der Ausstellung, den Typ des Geheges und die Anzahl der Tiere in der Ausstellung.

1. Markieren Sie den Abschnitt nach dem Kommentar **Call the stored procedure**. Führen Sie den markierten Text aus. Dieses Skript ruft die gespeicherte Prozedur auf, indem es Werte an die Eingabeparameter übergibt.

1. Markieren Sie die beiden **SELECT**-Anweisungen, und führen Sie sie aus. Führen Sie den markierten Text aus. Sie stellen fest, dass eine neue Zeile in die Gehegetabelle und fünf neue Zeilen in die Tiertabelle eingefügt wurden.

## Erstellen und Aufrufen einer Tabellenwertfunktion

Es ist an der Zeit, eine Tabellenwertfunktion zu erstellen. Eine Tabellenwertfunktion ist eine benutzerdefinierte Funktion, die eine Tabelle zurückgibt. Sie können eine Tabellenwertfunktion in einer `SELECT`-Anweisung wie eine normale Tabelle verwenden.

1. Wählen Sie in Visual Studio Code **Datei**, **Datei öffnen** und navigieren Sie dann zu den Lab-Skripten. Wählen Sie **../Allfiles/Labs/05/Lab5_Table_Function.sql** und wählen Sie dann **Öffnen**. Stellen Sie gegebenenfalls die Verbindung zum Server wieder her, indem Sie den Text **PGSQL Disconnected** auswählen und anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette auswählen. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

1. Führen Sie die Anweisung **SELECT current_database()** aus, um die aktuelle Datenbank zu überprüfen. Auch hier ist die Datenbank höchstwahrscheinlich auf `postgres` festgelegt. Ist dies der Fall, müssen Sie den Wert auf `zoodb` ändern. Wählen Sie die Ellipse in der Menüleiste mit dem Symbol *Ausführen* und wählen Sie **PostgreSQL-Datenbank ändern**. Wählen Sie in der Liste der Datenbanken die Option `zoodb` aus. Überprüfen Sie die Verbindung erneut, indem Sie die Anweisung **SELECT current_database()** ausführen.

1. Markieren Sie die gespeicherte Prozedur **CALL repopulate_zoo()**, und führen Sie sie aus, um mit bereinigten Daten zu beginnen.

1. Markieren Sie den Abschnitt nach dem Kommentar **Create a table valued function** und führen Sie ihn aus. Diese Funktion gibt eine Tabelle namens **enclosure_summary** zurück. Lesen Sie sich den Funktionscode durch, um zu verstehen, wie die Tabelle aufgefüllt wird.

1. Markieren Sie die beiden ausgewählten Anweisungen, und führen Sie sie aus, wobei Sie jedes Mal eine andere Gehege-ID eingeben.

1. Markieren Sie den Abschnitt unter dem Kommentar **How to use a table valued function with a LATERAL join** und führen Sie ihn aus. Dieses Skript zeigt, wie die Tabellenwertfunktion anstelle eines Tabellennamens in einer Verknüpfung verwendet wird.

## Integrierte Funktionen

In diesem Abschnitt erkunden Sie einige der integrierten Funktionen, die in PostgreSQL verfügbar sind. PostgreSQL verfügt über eine vielzahl integrierter Funktionen, mit denen Sie verschiedene Vorgänge für Daten ausführen können. Diese Funktionen können in SQL-Abfragen verwendet werden, um Daten zu bearbeiten und zu analysieren.

1. Wählen Sie in Visual Studio Code **Datei**, **Datei öffnen** und navigieren Sie dann zu den Lab-Skripten. Wählen Sie **../Allfiles/Labs/05/Lab5_InbuiltFunctions.sql** und anschließend **Öffnen**. Stellen Sie gegebenenfalls die Verbindung zum Server wieder her, indem Sie den Text **PGSQL Disconnected** auswählen und anschließend Ihre PostgreSQL-Serververbindung aus der Liste in der Befehlspalette auswählen. Wenn Sie nach einem Kennwort gefragt werden, geben Sie das zuvor erstellte Kennwort ein.

> &#128221; Die Funktionen in diesem Skript sind nicht spezifisch für die Zoodatenbank. Sie sind allgemeine PostgreSQL-Funktionen, die in jeder Datenbank verwendet werden können. Sie können sie in einer beliebigen Datenbank ausführen, einschließlich der `postgres`-Datenbank.

1. Markieren Sie die einzelnen Funktionen, und führen Sie sie aus, um zu ermitteln, wie sie funktionieren. Weitere Informationen finden Sie im Artikel der [Online-Dokumentation](https://www.postgresql.org/docs/current/functions.html) zu den einzelnen Funktionen.

1. Wenn Sie möchten, dass der PostgreSQL-Server weiterläuft, können Sie ihn einfach laufen lassen. Andernfalls können Sie den Server beenden, um unnötige Kosten im Bash-Terminal zu vermeiden. Um den Server zu beenden, führen Sie den folgenden Befehl aus:

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Ersetzen Sie `<your-server-name>` durch den Namen Ihres PostgreSQL-Servers.

    > &#128221; Sie können den Server auch über das Azure-Portal stoppen. Navigieren Sie im Azure-Portal zu **Ressourcengruppen**, und wählen Sie die zuvor erstellte Ressourcengruppe aus. Wählen Sie den PostgreSQL-Server aus und wählen Sie anschließend im Menü die Option **Beenden**.

1. Schließen Sie Visual Studio Code.

## Bereinigung

1. Wenn Sie diesen PostgreSQL-Server nicht mehr für andere Übungen benötigen, um unnötige Azure-Kosten zu vermeiden, löschen Sie die in dieser Übung erstellte Ressourcengruppe.

1. Löschen Sie bei Bedarf das Git Repository, das Sie zuvor geklont haben.
