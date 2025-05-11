---
lab:
  title: Konfigurieren von Berechtigungen in Azure Database for PostgreSQL
  module: Secure Azure Database for PostgreSQL
---

# Konfigurieren von Berechtigungen in Azure Database for PostgreSQL

In diesen Labübungen weisen Sie RBAC-Rollen, um den Zugriff auf Azure Database for PostgreSQL-Ressourcen zu steuern, sowie PostgreSQL-GRANTS zu, um den Zugriff auf Datenbankvorgänge zu steuern.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Sollten Sie kein Azure-Abonnement besitzen, können Sie eine [kostenlose Azure-Testversion](https://azure.microsoft.com/free) erstellen.

Um diese Übungen durchzuführen, müssen Sie einen PostgreSQL-Server installieren, der mit Microsoft Entra ID, ehemals Azure Active Directory, verbunden ist.

### Erstellen einer Ressourcengruppe

1. Navigieren Sie in Ihrem Webbrowser zum [Azure-Portal](https://portal.azure.com). Melden Sie sich mit einem Konto des Typs „Besitzer“ oder „Mitwirkender“ an.
2. Wählen Sie unter „Azure-Dienste“ die Option **Ressourcengruppen** und dann **+Erstellen** aus.
3. Überprüfen Sie, ob das richtige Abonnement angezeigt wird und geben Sie dann den Namen der Ressourcengruppe als **rg-PostgreSQL_Entra** ein. Wählen Sie eine **Region** aus.
4. Klicken Sie auf **Überprüfen + erstellen**. Klicken Sie anschließend auf **Erstellen**.

## Erstellen eines flexiblen Azure Database for PostgreSQL-Servers

1. Wählen Sie unter „Azure-Dienste“ die Option **+ Ressource erstellen** aus.
    1. In **Suchen Sie im Marketplace**, geben Sie **`azure database for postgresql flexible server`** ein, wählen Sie **Azure Database for PostgreSQL – Flexibler Server** und klicken Sie auf **Erstellen**.
1. Füllen Sie auf der Registerkarte **Allgemeine Informationen** für „Flexibler Server“ die Felder wie folgt aus:
    1. Abonnement: Ihr Abonnement
    1. Ressourcengruppe: **rg-PostgreSQL_Entra**.
    1. Servername: **psql-postgresql-fx7777** (Servernamen müssen global eindeutig sein, ersetzen Sie daher 7777 durch vier zufällige Zahlen).
    1. Region: Wählen Sie dieselbe Region wie für die Ressourcengruppe aus.
    1. PostgreSQL-Version: Wählen Sie 16.
    1. Workloadtyp: **Entwicklung**.
    1. Compute + Speicher: **Burstfähig, B1ms**.
    1. Verfügbarkeitszone: keine Präferenz.
    1. Hochverfügbarkeit: deaktiviert lassen.
    1. Authentifizierungsmethode: Wählen Sie **PostgreSQL und Microsoft Entra-Authentifizierung**
    1. Festlegen des Microsoft Entra-Administrierenden: Wählen Sie **Administrator festlegen**
        1. Suchen Sie nach Ihrem Konto in **Microsoft Entra-Administratoren auswählen** und (o) Ihrem Konto und klicken Sie auf **Auswählen**.
    1. Geben Sie in **Admin-Benutzername** **`demo`** ein.
    1. Geben Sie unter **Kennwort** ein entsprechend komplexes Kennwort ein.
    1. Klicken Sie auf **Weiter: Netzwerk >**.
1. Füllen Sie auf der Registerkarte **Netzwerk** für „Flexibler Server“ jedes Feld wie folgt aus:
    1. Konnektivitätsmethode: (o) Öffentlicher Zugang (erlaubte IP-Adressen) und privater Endpunkt
    1. Öffentlicher Zugriff, wählen Sie **Öffentlichen Zugriff auf diese Ressource über das Internet unter Verwendung einer öffentlichen IP-Adresse zulassen**
    1. Wählen Sie unter „Firewallregeln“ die Option **+ Aktuelle IP-Adresse des Clients hinzufügen** aus, um Ihre aktuelle IP-Adresse als Firewallregel hinzuzufügen. Sie können dieser Firewallregel optional einen aussagekräftigen Namen geben. Fügen Sie außerdem **Hinzufügen 0.0.0.0 – 255.255.255.255** hinzu und klicken Sie auf **Weiter**.
1. Klicken Sie auf **Überprüfen + erstellen**. Überprüfen Sie Ihre Einstellungen und wählen Sie dann **Erstellen** aus, um Ihren Azure Database for PostgreSQL – Flexibler Server zu erstellen. Wenn die Bereitstellung abgeschlossen ist, wählen Sie **Zur Ressource wechseln** aus, um den nächsten Schritt auszuführen.

## Installieren von Azure Data Studio

So installieren Sie Azure Data Studio für Azure Database for PostgreSQL:

1. Navigieren Sie in einem Browser zum [Azure Data Studio herunterladen und installieren](/sql/azure-data-studio/download-azure-data-studio), und klicken Sie unter der Windows-Plattform auf **User installer (recommended)** (Benutzerinstallationsprogramm (empfohlen)). Die ausführbare Datei wird in Ihren Ordner „Downloads“ heruntergeladen.
1. Klicken Sie auf **Datei öffnen**.
1. Nun wird die Lizenzvereinbarung angezeigt. Lesen und **akzeptieren Sie die Vereinbarung**, und klicken sie auf **Weiter**.
1. Wählen Sie unter **Weitere Aufgaben auswählen** die Option **Zu PATH hinzufügen** sowie und alle weiteren erforderlichen Ergänzungen aus. Wählen Sie **Weiter** aus.
1. Das Dialogfeld **Bereit für die Installation** wird angezeigt. Überprüfen Sie Ihre Einstellungen. Klicken Sie auf **Zurück**, um Änderungen vorzunehmen, oder klicken Sie auf **Installieren**.
1. Das Dialogfeld **Completing the Azure Data Studio Setup Wizard** (Der Setup-Assistent für Azure Data Studio wird abgeschlossen) wird angezeigt. Wählen Sie **Fertig stellen**aus. Azure Data Studio wird gestartet.

### Installieren der PostgreSQL-Erweiterung

1. Öffnen Sie Azure Data Studio, falls das Programm noch nicht geöffnet ist.
1. Klicken Sie im linken Menü auf **Erweiterungen**, um den Bereich „Erweiterungen“ anzuzeigen.
1. Geben Sie **Subnetze** in die Suchleiste ein. Das Symbol der PostgreSQL-Erweiterung für Azure Data Studio wird angezeigt.
1. Wählen Sie **Installieren** aus. Die Erweiterung wird installiert.

### Verbinden mit Azure Database for PostgreSQL – Flexibler Server

1. Öffnen Sie Azure Data Studio, falls das Programm noch nicht geöffnet ist.
1. Klicken Sie im linken Menü auf **Verbindungen**.
1. Klicken Sie auf **Neue Verbindung**.
1. Wählen Sie unter **Details zur Verbindung** unter **Verbindungstyp** die Option **PostgreSQL** aus der Dropdownliste aus.
1. Geben Sie unter **Servername** den vollständigen Servernamen ein, so wie er im Azure-Portal angezeigt wird.
1. Behalten Sie unter **Authentifizierungstyp** „Kennwort“ bei.
1. Geben Sie unter „Benutzername“ und „Kennwort“ den Benutzernamen „**demo**“ und das oben erstellte komplexe Kennwort ein.
1. Klicken Sie auf [ x ] Kennwort merken.
1. Die verbleibenden Felder sind optional.
1. Wählen Sie **Verbinden**. Sie sind mit dem Azure Database for PostgreSQL-Server verbunden.
1. Eine Liste der Serverdatenbanken wird angezeigt. Dazu gehören Systemdatenbanken und Benutzerdatenbanken.

### Erstellen der Zoodatenbank

1. Navigieren Sie entweder zu dem Ordner mit Ihren Übungsskriptdateien oder laden Sie die Datei **Lab2_ZooDb.sql** von [MSLearn PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql/blob/main/Allfiles/Labs/02) herunter.
1. Öffnen Sie Azure Data Studio, falls das Programm noch nicht geöffnet ist.
1. Klicken Sie auf **Datei**, anschließend auf **Datei öffnen**, und navigieren Sie zu dem Ordner, in dem Sie die Skripts gespeichert haben. Wählen Sie **../Allfiles/Labs/02/Lab2_ZooDb.sql** und **Öffnen**. Wenn eine Vertrauensstellungswarnung angezeigt wird, wählen Sie **"Öffnen**" aus.
1. Führen Sie das Skript aus. Die zoodb-Datenbank wird erstellt.

## Erstellen eines neuen Benutzerkontos in Microsoft Entra ID

> [!NOTE]
> In den meisten Produktions- oder Entwicklungsumgebungen ist es sehr wahrscheinlich, dass Sie nicht über die Berechtigungen eines Abonnementkontos verfügen, um Konten für Ihren Microsoft Entra ID-Service zu erstellen.  In diesem Fall sollten Sie, sofern dies von Ihrer Organisation erlaubt ist, Ihren Microsoft Entra ID-Administrator bitten, ein Testkonto für Sie zu erstellen. Wenn Sie kein Entra-Konto für den Test erhalten können, überspringen Sie diesen Abschnitt und fahren Sie mit dem Abschnitt **GRANT-Zugriff auf Azure Database for PostgreSQL** fort. 

1. Melden Sie sich im [Azure-Portal](https://portal.azure.com) mit einem Besitzerkonto an und navigieren Sie zu Microsoft Entra ID.
1. Wählen Sie unter **Verwalten** die Option **Benutzer** aus.
1. Wählen Sie oben links **Neuer Benutzer** und dann **Neuen Benutzer erstellen** aus.
1. Geben Sie auf der Seite **Neuer Benutzer** die folgenden Details ein, und wählen Sie dann **Erstellen** aus:
    - **Benutzerprinzipalname:** Wählen Sie einen Prinzipalnamen
    - **Anzeigename:** Wählen Sie einen Anzeigenamen
    - **Kennwort:** Deaktivieren Sie **Kennwort automatisch generieren** und geben Sie dann ein sicheres Kennwort ein. Notieren Sie sich den Namen des Hauptbenutzers und das Kennwort.
    - Klicken Sie auf **Überprüfen und erstellen**.

    > [!TIP]
    > Notieren Sie sich beim Erstellen des Benutzers den vollständigen **Benutzerprinzipalnamen**, damit Sie ihn später verwenden können, um sich anzumelden.

### Zuweisen der Rolle „Leser“

1. Wählen Sie im Azure-Portal **Alle Ressourcen** und dann Ihre Azure Database for PostgreSQL-Ressource aus.
1. Wählen Sie **Zugriffssteuerung (IAM)** und dann **Rollenzuweisungen** aus. Das neue Konto erscheint nicht in der Liste.
1. Wählen Sie **+ Hinzufügen** und dann **Rollenzuweisung hinzufügen** aus.
1. Wählen Sie die Rolle **Leser** und dann **Weiter** aus.
1. Wählen Sie **+ Mitglieder auswählen**, fügen Sie das neue Konto, das Sie im vorherigen Schritt hinzugefügt haben, zur Mitgliederliste hinzu und wählen Sie dann **Weiter**.
1. Wählen Sie **Überprüfen und zuweisen** aus.

### Testen der Rolle „Leser“

1. Wählen Sie oben rechts im Azure-Portal Ihr Benutzerkonto und dann **Abmelden** aus.
1. Melden Sie sich als neuer Benutzende mit dem Benutzernamen und dem Kennwort an, die Sie sich notiert haben. Ersetzen Sie das Standardkennwort, wenn Sie dazu aufgefordert werden, und merken Sie sich das neue Kennwort.
1. Wählen Sie **Später fragen**, wenn Sie zur Multi-Faktor-Authentifizierung aufgefordert werden.
1. Wählen Sie auf der Homepage des Portals **Alle Ressourcen** und dann Ihre Azure Database for PostgreSQL-Ressource aus.
1. Wählen Sie **Stop** (Beenden) aus. Daraufhin wird ein Fehler angezeigt, da die Rolle „Leser“ lediglich zulässt, die Ressource anzuzeigen, nicht aber sie zu ändern.

### Zuweisen der Rolle „Mitwirkender“

1. Wählen Sie oben rechts im Azure-Portal das Benutzerkonto des neuen Kontos aus und wählen Sie dann **Abmelden**.
1. Melden Sie sich mit Ihrem ursprünglichen Besitzerkonto an.
1. Navigieren Sie zu Ihrer Azure Database for PostgreSQL-Ressource, und wählen Sie dann **Zugriffssteuerung (IAM)** aus.
1. Wählen Sie **+ Hinzufügen** und dann **Rollenzuweisung hinzufügen** aus.
1. Wählen Sie **Privilegierte Administratorrollen**
1. Wählen Sie die Rolle **Mitwirkender** und dann **Weiter** aus.
1. Fügen Sie das neue Konto, das Sie zuvor hinzugefügt haben, zur Mitgliederliste hinzu und wählen Sie dann **Weiter**.
1. Wählen Sie **Überprüfen und zuweisen** aus.
1. Wählen Sie **Rollenzuweisungen** aus. Das neue Konto verfügt nun über Zuweisungen für die Rollen „Leser“ und „Mitwirkender“.

## Testen der Rolle „Mitwirkender“

1. Wählen Sie oben rechts im Azure-Portal Ihr Benutzerkonto und dann **Abmelden** aus.
1. Melden Sie sich mit dem neuen Konto an, mit dem Benutzernamen und dem Kennwort, die Sie notiert haben.
1. Wählen Sie auf der Homepage des Portals **Alle Ressourcen** und dann Ihre Azure Database for PostgreSQL-Ressource aus.
1. Wählen Sie **Beenden** und dann **Ja** aus. Dieses Mal wird der Server ohne Fehler gestoppt, da dem neuen Konto die erforderliche Rolle zugewiesen wurde.
1. Wählen Sie **Start**, um sicherzustellen, dass der PostgreSQL-Server für die nächsten Schritte bereit ist.
1. Wählen Sie oben rechts im Azure-Portal das Benutzerkonto des neuen Kontos aus und wählen Sie dann **Abmelden**.
1. Melden Sie sich mit Ihrem ursprünglichen Besitzerkonto an.

## Gewähren von Zugriff auf Azure Database for PostgreSQL

1. Öffnen Sie Azure Data Studio und stellen Sie mithilfe des Benutzers **demo**, den Sie oben mit Administratorrechten festgelegt haben, eine Verbindung zu Ihrem Azure Database for PostgreSQL-Server her.
1. Führen Sie diesen Code im Abfragefenster für die Postgres-Datenbank aus. Es sollten zwölf Benutzerrollen zurückgegeben werden, u. a. die Rolle **demo**, die Sie zum Herstellen einer Verbindung verwenden:

    ```SQL
    SELECT rolname FROM pg_catalog.pg_roles;
    ```

1. Führen Sie den folgenden Code aus, um eine neue Rolle zu erstellen:

    ```SQL
    CREATE ROLE dbuser WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'R3placeWithAComplexPW!';
    GRANT CONNECT ON DATABASE zoodb TO dbuser;
    ```
    > [!NOTE]
    > Ersetzen Sie das Kennwort im obigen Skript durch ein komplexes Kennwort.

1. Um die neue Rolle aufzulisten, führen Sie die obige SELECT-Abfrage in **pg_catalog.pg_roles** erneut aus. Die Rolle **dbuser** sollte aufgelistet werden.
1. Damit die neue Rolle Daten in der Tabelle **Tier** in der Datenbank **zoodb** abfragen und ändern kann, führen Sie diesen Code in der zoodb-Datenbank aus:

    ```SQL
    GRANT SELECT, INSERT, UPDATE, DELETE ON animal TO dbuser;
    ```

## Testen der neuen Rolle

1. Wählen Sie in Azure Data Studio in der Liste der **VERBINDUNGEN** die Schaltfläche „Neue Verbindung“ aus.
1. Wählen Sie in der Liste **Verbindungstyp** die Option **PostgreSQL** aus.
1. Geben Sie im Textfeld **Servername** den vollqualifizierten Servernamen für Ihre Azure Database for PostgreSQL-Ressource ein. Sie können ihn aus dem Azure-Portal kopieren.
1. Wählen Sie in der Liste **Authentifizierungstyp** die Option **Kennwort** aus.
1. Geben Sie in das Textfeld **Benutzername** den Text **dbuser** und in das Textfeld **Kennwort** das komplexe Kennwort ein, mit dem Sie das Konto erstellt haben.
1. Aktivieren Sie das Kontrollkästchen **Kennwort speichern**, und wählen Sie dann **Verbinden** aus.
1. Wählen Sie **Neue Abfrage** aus, und führen Sie dann den folgenden Code aus:

    ```SQL
    SELECT * FROM animal;
    ```

1. Führen Sie den folgenden Code aus, um festzustellen, ob Sie über UPDATE-Berechtigung verfügen:

    ```SQL
    UPDATE animal SET name = 'Linda Lioness' WHERE ani_id = 7;
    SELECT * FROM animal;
    ```

1. Führen Sie den folgenden Code aus, um festzustellen, ob Sie über DROP-Berechtigung verfügen: Überprüfen Sie den Fehlercode, wenn ein Fehler auftritt:

    ```SQL
    DROP TABLE animal;
    ```

1. Führen Sie den folgenden Code aus, um festzustellen, ob Sie über GRANT-Berechtigung verfügen:

    ```SQL
    GRANT ALL PRIVILEGES ON animal TO dbuser;
    ```

Diese Tests zeigen, dass der neue Benutzer DML-Befehle (Data Manipulation Language) ausführen kann, um Daten abzufragen und zu ändern, aber keine DDL-Befehle (Data Definition Language) verwenden kann, um das Schema zu ändern. Außerdem kann der neue Benutzer keine neuen Rechte erteilen, um die Berechtigungen zu umgehen.

## Bereinigung

Sie werden diesen PostgreSQL-Server nicht mehr verwenden, also löschen Sie bitte die von Ihnen erstellte Ressourcengruppe, wodurch der Server entfernt wird.
