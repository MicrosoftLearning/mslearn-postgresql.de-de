---
lab:
  title: Einführung in Azure Database for PostgreSQL
  module: Explore PostgreSQL architecture
---

# Einführung in Azure Database for PostgreSQL

In dieser Übung erstellen Sie eine Instanz von Azure Database for PostgreSQL – Flexibler Server und konfigurieren den Aufbewahrungszeitraum für Sicherungen.

## Vor der Installation

Für diese Übung benötigen Sie ein eigenes Azure-Abonnement. Wenn Sie nicht über ein Azure-Abonnement verfügen, können Sie ein [kostenloses Azure-Testkonto](https://azure.microsoft.com/free) erstellen.

### Erstellen einer Ressourcengruppe

1. Navigieren Sie in Ihrem Webbrowser zum [Azure-Portal](https://portal.azure.com). Melden Sie sich mit einem Konto des Typs „Besitzer“ oder „Mitwirkender“ an.
2. Wählen Sie unter „Azure-Dienste“ die Option **Ressourcengruppen** und dann **+Erstellen** aus.
3. Überprüfen Sie, ob das richtige Abonnement angezeigt wird und geben Sie dann den Namen der Ressourcengruppe als **rg-PostgreSQL_Flexi** ein. Wählen Sie eine **Region** aus.
4. Klicken Sie auf **Überprüfen + erstellen**. Klicken Sie anschließend auf **Erstellen**.

### Erstellen eines flexiblen Azure Database for PostgreSQL-Servers

1. Wählen Sie unter „Azure-Dienste“ die Option **+ Ressource erstellen** aus.
    1. In **Suchen Sie den Marketplace**, geben Sie **`azure database for postgresql flexible server`** ein, wählen Sie **Azure Database for PostgreSQL – Flexibler Server** und klicken Sie auf **Erstellen**.
1. Füllen Sie auf der Registerkarte **Allgemeine Informationen** für „Flexibler Server“ die Felder wie folgt aus:
    1. Abonnement: Ihr Abonnement
    1. Ressourcengruppe: **rg-PostgreSQL_Flexi**.
    1. Servername: **psql-postgresql-fx9999** (Der Servername muss global eindeutig sein, ersetzen Sie also 9999 durch vier zufällige Zahlen).
    1. Region: Wählen Sie dieselbe Region wie für die Ressourcengruppe aus.
    1. PostgreSQL-Version: Wählen Sie 16.
    1. Workloadtyp: **Entwicklung**.
    1. Compute + Speicher: **Burstfähig, B1ms**. Wählen Sie **Server konfigurieren** aus, und überprüfen Sie die Konfigurationsoptionen. Nehmen Sie keine Änderungen vor und schließen Sie das Blatt in der rechten Ecke.
    1. Verfügbarkeitszone: keine Präferenz.
    1. Hohe Verfügbarkeit: Deaktiviert.
    1. Authentifizierungsmethode: **Nur PostgreSQL-Authentifizierung**
    1. Geben Sie in **Admin-Benutzername** **`demo`** ein.
    1. Geben Sie unter **Kennwort** ein entsprechend komplexes Kennwort ein.
    1. Klicken Sie auf **Weiter: Netzwerk >**.
1. Füllen Sie auf der Registerkarte **Netzwerk** für „Flexibler Server“ jedes Feld wie folgt aus:
    1. Konnektivitätsmethode: (o) Öffentlicher Zugang (erlaubte IP-Adressen) und privater Endpunkt
    1. Öffentlicher Zugriff, wählen Sie **Öffentlichen Zugriff auf diese Ressource über das Internet unter Verwendung einer öffentlichen IP-Adresse zulassen**
    1. Wählen Sie unter „Firewallregeln“ die Option **+ Aktuelle IP-Adresse des Clients hinzufügen** aus, um Ihre aktuelle IP-Adresse als Firewallregel hinzuzufügen. Sie können dieser Firewallregel optional einen aussagekräftigen Namen geben.
1. Klicken Sie auf **Überprüfen + erstellen**. Überprüfen Sie Ihre Einstellungen und wählen Sie dann **Erstellen** aus, um Ihren Azure Database for PostgreSQL – Flexibler Server zu erstellen. Wenn die Bereitstellung abgeschlossen ist, wählen Sie **Zur Ressource wechseln** aus, um den nächsten Schritt auszuführen.

## Untersuchen von Serverparametern

1. Klicken Sie unter **Einstellungen** auf **Serverparameter**.
1. Geben Sie in das Feld **Suchen, um Artikel zu filtern …** **`connections`** ein. Serverparameter im Zusammenhang mit Verbindungen werden angezeigt. Beachten Sie den Wert für **max_connections**. Nehmen Sie keine Änderungen vor.
1. Wählen Sie im linken Menü **Übersicht** aus, um **Serverparameter** zu verlassen.

## Ändern des Aufbewahrungszeitraums für Sicherungen

1. Navigieren Sie unter **Einstellungen** zum Blatt **Übersicht**, und wählen Sie **Compute + Speicher** aus. Dieser Abschnitt enthält Ihren aktuellen Computetarif und die Option zum Upgrade. Außerdem werden der bereitgestellte Speicherplatz und die Option zum Vergrößern des Speichers angezeigt.
1. Unter **Sicherungen** wird der Aufbewahrungszeitraum für Sicherungen in Tagen angezeigt. Ändern Sie über den Schieberegler den Aufbewahrungszeitraum der Sicherung in 14 Tage. Wählen Sie **Speichern** aus, um Ihre Änderungen zu übernehmen.
1. Wenn Sie diese Übung abgeschlossen haben, navigieren Sie zum Blatt **Übersicht** und klicken Sie auf **STOPP**, wodurch der Server angehalten wird.
    1. Solange der Server gestoppt ist, fallen keine Kosten an. Beachten Sie jedoch, dass der Server innerhalb von sieben Tagen neu gestartet wird, wenn Sie ihn nicht gelöscht haben.

## Optionale Übung: Konfigurieren eines hoch verfügbaren Servers

1. Wählen Sie unter „Azure-Dienste“ die Option **+ Ressource erstellen** aus.
    1. In **Suchen Sie im Marketplace**, geben Sie **`azure database for postgresql flexible server`** ein, wählen Sie **Azure Database for PostgreSQL – Flexibler Server** und klicken Sie auf **Erstellen**.
1. Füllen Sie auf der Registerkarte **Allgemeine Informationen** für „Flexibler Server“ die Felder wie folgt aus:
    1. Abonnement: Ihr Abonnement
    1. Ressourcengruppe: **rg-PostgreSQL_Flexi**.
    1. Servername: **psql-postgresql-fx8888** (Der Servername muss global eindeutig sein, ersetzen Sie also 8888 durch zufällige Zahlen).
    1. Region: Wählen Sie dieselbe Region wie für die Ressourcengruppe aus.
    1. PostgreSQL-Version: Wählen Sie 16.
    1. Workloadtyp: **Produktion (Klein/Mittel)**
    1. Compute + Speicher: Belassen Sie es bei **Allgemeiner Zweck**.
    1. Verfügbarkeitszone: Sie können dies bei „Keine Präferenz“ belassen und Azure wählt automatisch Verfügbarkeitszonen für Ihre primären und sekundären Server. Alternativ können Sie auch eine Verfügbarkeitszone angeben, die mit Ihrer Anwendung gekoppelt ist.
    1. Hochverfügbarkeit aktivieren: Aktivieren Sie diese Option. Beachten Sie die geschätzten Kosten, wenn diese Option aktiviert ist.
    1. Hochverfügbarkeitsmodus: Wählen Sie **Zonen redundant – ein Standby-Server ist immer in einer anderen Zone in derselben Region wie der Primärserver verfügbar**
    1. Authentifizierungsmethode: **Nur PostgreSQL-Authentifizierung**
    1. Geben Sie im Feld **Administratorbenutzername** den Eintrag **demo** ein.
    1. Geben Sie unter **Kennwort** ein entsprechend komplexes Kennwort ein.
    1. Klicken Sie auf **Weiter: Netzwerk >**.
1. Füllen Sie auf der Registerkarte **Netzwerk** für „Flexibler Server“ jedes Feld wie folgt aus:
    1. Konnektivitätsmethode: (o) Öffentlicher Zugang (erlaubte IP-Adressen) und privater Endpunkt
    1. Öffentlicher Zugriff, wählen Sie **Öffentlichen Zugriff auf diese Ressource über das Internet unter Verwendung einer öffentlichen IP-Adresse zulassen**
    1. Wählen Sie unter „Firewallregeln“ die Option **+ Aktuelle IP-Adresse des Clients hinzufügen** aus, um Ihre aktuelle IP-Adresse als Firewallregel hinzuzufügen. Sie können dieser Firewallregel optional einen aussagekräftigen Namen geben.
1. Klicken Sie auf **Überprüfen + erstellen**. Überprüfen Sie Ihre Einstellungen und wählen Sie dann **Erstellen**, um Ihren Azure Database for PostgreSQL – Flexibler Server zu erstellen. Wenn die Bereitstellung abgeschlossen ist, wählen Sie **Zur Ressource wechseln** aus, um den nächsten Schritt auszuführen.

### Überprüfen des neuen Servers

1. Wählen Sie unter **Einstellungen** die Option **Hochverfügbarkeit** aus. Hochverfügbarkeit ist aktiviert und die primäre Verfügbarkeitszone sollte Zone 1 sein.
    1. Die Standby-Verfügbarkeitszone wurde automatisch zugewiesen. Sie unterscheidet sich von der primären Verfügbarkeitszone und ist normalerweise Zone 2.

### Erzwingen eines Failovers

1. Wählen Sie auf dem Blatt **Hochverfügbarkeit** im Menü oben **Erzwungenes Failover** aus. Eine Meldung wird angezeigt. Wählen Sie **OK** aus.
1. Der Failovervorgang wird gestartet. Eine Meldung wird angezeigt, sobald der Failovervorgang erfolgreich abgeschlossen wurde.
1. Auf dem Blatt **Hochverfügbarkeit** können Sie sehen, dass die primäre Zone jetzt 2 und die Standbyverfügbarkeitszone 1 ist. Möglicherweise müssen Sie Ihren Browser aktualisieren, um die aktuellen Informationen zu sehen.
1. Nach Abschluss dieser Übung müssen Sie den Server löschen.

## Bereinigung

Für den Server für diese Labübung fallen Gebühren an. Löschen Sie die Ressourcengruppe **rg-PostgreSQL_Flexi**, sobald Sie die Übung abgeschlossen haben. Dadurch werden der Server und alle anderen Ressourcen, die Sie in dieser Lab-Übung bereitgestellt haben, entfernt.
