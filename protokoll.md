# Protokoll & Systemdesign

Dieses Dokument beschreibt ein dezentrales, datenschutzfreundliches Protokoll & System für die Vermittlung von Terminen zwischen zwei Parteien. Es soll u.a. für die privatsphäre-freundliche Vermittlung von Impfterminen im Rahmen der Covid-19 Pandemie eingesetzt werden.

## Akteure

Es gibt folgende Akteure im System:

* **Nutzer** buchen Termine bei **Anbietern** (z.B. Ärzten).
* **Anbieter** stellen **Nutzern** Termine zur Verfügung.
* **Vermittler** schaffen Vertrauen zwischen den Akteuren im System.
* **Backend-Betreiber** betreiben die zur Kommunikation zwischen den Akteuren benötigte IT-Infrastruktur.

## Anforderungen

Die Privatsphäre und Sicherheit von Nutzern und Anbietern soll möglichst gut geschützt werden. Vermittler sowie Backend-Betreiber sollen nicht in der Lage sein, Informationen über Nutzer oder Anbieter zu erlangen. Sie sollen auch nicht in der Lage sein nachzuvollziehen, ob zwischen spezifischen Anbietern und Nutzern Termine zustande gekommen sind, oder Details zu diesen Terminen aufdecken können. Vermittler und Backend-Betreiber sollen nichtsdestotrotz in der Lage sein, eine missbräuchliche Nutzung des Systems zu erkennen und zu unterbinden (ggf. mit Unterstützung von Nutzern und Anbietern). 

Das System soll zudem möglichst robust gegenüber Missbrauch durch Nutzer und Anbieter sein. Es soll verhindert werden, dass unseriöse Anbieter über das System Kontakt mit Nutzern aufnehmen oder Daten von diesen einsehen können, dass Nutzer über das System erlangte Termine weiterveräußern können (Arbitrage/Scalping), oder dass Akteure durch Spamming oder andere Techniken die Funktionalität des Systems für legitime Nutzer oder Anbieter beeinträchtigen können. Eine Kompromittierung einzelner Systemkomponenten oder Akteure soll die Privatsphäre- oder Sicherheitsgarantien des Systems möglichst wenig beeinträchtigen.

## Systemkomponenten

Das System besteht aus folgenden Komponenten:

* **Nutzer-Frontend**: Eine Web-/Desktop-/Mobil-Anwendung die von Nutzern eingesetzt wird um sich zu registrieren und Termine zu buchen. 
* **Anbieter-Frontend**: Eine Web-/Desktop-/Mobil-Anwendung die von Anbietern eingesetzt wird um sich zu registrieren und Termine zu vergeben.
* **Vermittler-Frontend**: Eine Web-/Desktop-/Mobil-Anwwendung die von Vermittlern eingesetzt wird um Anbieter zu verifizieren, das System zu überwachen und Missbrauch einzudämmen.
* **Daten- & Authentifizierungs-Backends**: Ein API-basiertes Backend-System, welches verschlüsselte Daten der obigen Akteure speichert, Anbieter authentifiziert sowie spezifische Daten von Nutzern oder Anbietern kryptographisch signiert. Ggf. werden hierzu Backend-Dienste in mehrere Dienste/APIs aufgeteilt.

Im Folgenden wird der Begriff "App" verwendet um die Anwendungen einzelner Akteure zu bezeichnen, hiermit ist nicht zwangsweise eine mobile App gemeint.

## Protokoll `v0.1`

Die folgenden Abschnitte beschreiben das Protokoll zur Terminvermittlung sowie die hierfür eingesetzten Systemkomponenten (kryptographische Operationen und Datenflüsse werden nicht im vollen Detail beschrieben).

### Hinweise

Zur Vereinfachung wird im Dokument von Verschlüsselung mithilfe von asymmetrischen (ECDH-)Schlüsselpaaren gesprochen. Hierbei ist jeweils eine Schlüsselableitung basierend auf einem (temporären oder permanenten) ECDH-Schlüsselpaar in Kombination mit einem symmetrischen Verschlüsselungsverfahren (z.B. AES-GCM) gemeint. Wenn im Rahmen einer bidirektionalen Kommunikation (z.B. zwischen Anbietern und Nutzern) von öffentlichen oder privaten Schlüsseln für die Verschlüsselung von Daten gesprochen wird ist weiterhin impliziert, dass diese Schlüssel über ein geeignetes Schlüsselableitungsverfahren (z.B. eine Diffie-Hellman Ratsche) für jede ausgetauschte Nachricht abgeändert werden.

### Kurzfassung

Die Terminvermittlung erfolgt in mehreren überwiegend automatisiert und asynchron ablaufenden Schritten:

* Vermittler werden vom Backend-Betreiber registriert.
* Anbieter registrieren sich und erbitten eine Verifikation durch Vermittler.
* Vermittler verifizieren Anbieter und gewähren diesen Zugang zum System.
* Nutzer registrieren sich und stellen (anonyme) Terminanfragen für ein gegebenes Einzugsgebiet.
* Anbieter rufen Terminanfragen ab und erstellen Terminangebote für spezifische Nutzer.
* Nutzer wählen aus den Angeboten Termine aus und buchen diese.
* Anbieter verifizieren Nutzerdaten und bestätigen die gebuchten Termine.
* Nutzer und Anbieter ändern oder stornieren ggf. gebuchte Termine.

Alle Kommunikation zwischen Anbietern und Nutzern erfolgt Ende-zu-Ende verschlüsselt, das Backend-System speichert die verschlüsselten Daten ohne Zuordnung zu spezifischen Anbietern oder Nutzern und kann somit kaum Metadaten über diese ableiten. Insofern externe Systeme (E-Mail, SMS) zur Kommunikation oder Verifikation eingesetzt werden erfolgt dies nur anlassbezogen durch Mitwirkung von Anbietern, entsprechende Kontaktdaten (E-Mail Adressen, Telefonnummern) sind nur Nutzern selbst sowie zweck- & anlassgebunden Betreiber externer Systeme zugänglich.

### 1. Vermittler-Initialisierung

Vermittler agieren als Zertifikats-Authorität (CA) im System und erstellen hierzu ein ECDSA-Signatur-Schlüsselpaar sowie ein zugehöriges Zertifikat $C_\mathrm{auth}$, welches als Vertrauensanker (Root-Zertifikat) im Backend hinterlegt wird. Weiterhin erstellen sie ECDH- & ECDSA-Schlüssepaare $(K_\mathrm{auth}^\mathrm{enc,priv}, K_\mathrm{auth}^\mathrm{enc,pub}), (K_\mathrm{auth}^\mathrm{sign,priv}, K_\mathrm{auth}^\mathrm{sign,pub})$, dessen öffentliche Schlüssel signiert im Backend abgelegt werden.

Die Vermittler-App erstellt für alle relevanten Einzugsgebiete (z.B. basierend auf Postleitzahlen) jeweils einen zufälligen Salt-Wert (32 Byte Zufallsdaten) und speichert die Liste der Einzugsgebiete mit relevanten Daten sowie den zugehörigen Salt-Werten im Backend. Diese ist von dort öffentlich abrufbar. Für jedes Einzugsgsbiet erstellt die Vermittler-App zudem ein ECDH-Schlüsselpaar $(K_\mathrm{r_i}^\mathrm{enc,priv}, K_\mathrm{r_i}^\mathrm{enc,pub})$. Der private Schlüssel $K_\mathrm{r_i}^\mathrm{enc,priv}$ wird im Rahmen der Anbieter-Initialisierung vom Backend mit dem öffentlichen Schlüssel relevanter Anbieter (siehe unten) verschlüsselt und für diese im Backend abgelegt. Der öffentliche Schlüssel wird gemeinsam mit den Daten der Einzugsgebiete veröffentlicht.

### 2. Backend-Initialisierung

Backends verfügen über einen zufallsgenerierten Schlüssel (32 Byte Zufallsdaten) mithilfe dessen über ein geeignetes Schlüsselableitungsverfahren (z.B. HKDF) Prioritätstokens abgeleitet werden, welche im weiteren Prozess zur Prioritätsermittlung eingesetzt werden. Dieser Schlüssel muss geheimgehalten werden um die Fairness der Terminvermittlung zu gewährleisten. Backends verfügen weiterhin über Signatur-Schlüsselpaare $(K_\mathrm{be}^\mathrm{sign,priv}, K_\mathrm{be}^\mathrm{sign,pub})$, welche zur Signierung von Prioritätstokens genutzt werden.

### 3. Anbieter-Initialisierung

Anbieter authentifizieren sich zunächst gegenüber dem Backend und erhalten hierdurch Zugang zur Anbieter-App. Sie geben dort relevante Daten ihrer Praxis ein, sowie Hinweise die sie Nutzern bei der Terminvergabe anzeigen möchten. Die Anbieter-App generiert ein ECDH-Schlüsselpaar $(K_\mathrm{op_i}^\mathrm{enc,priv},K_\mathrm{op_i}^\mathrm{enc,pub})$, verschlüsselt die angebenen Daten mit dem öffentlichen Vermittler-Schlüssel und lädt diese zur Verifikation ins Backend hoch, verknüpft mit einer zufälligen ID (32 Byte Zufallsdaten). Vermittler rufen die Daten von dort ab, entschlüsseln und verifizieren sie und signieren erfolgreich verifizierte Daten mit ihrem privaten Signierschlüssel $K_\mathrm{auth}^\mathrm{sign,priv}$. Zusätzlich signieren sie den öffentlichen Schlüssel $K_\mathrm{op_i}^\mathrm{enc,pub}$ des Anbieters. Die Vermittler-App verschlüsselt diese Daten wiederum mit dem Anbieter-Schlüssel und legt sie unter der gegebenen ID im Backend ab. Der signierte öffentliche Anbieter-Schlüssel wird zusätzlich über das Backend publiziert, die zugehörigen Anbieter-Daten werden mit einem zufallsgenerierten symmetrischen Schlüssel der Anbieter-App verschlüsselt und ebenfalls dort abgelegt (dies erlaubt Vermittlern die spätere Kontrolle von Anbietern). Die Anbieter-App lädt die signierten Daten vom Backend herunter und entschlüsselt sie.

Anbieter erstellen nun in ihrer App Termine, welche lokal in der App gespeichert werden. Damit ist die Initialisierung auf Anbieter-Seite abgeschlossen. Die App muss zur Terminvermittlung idealerweise kontinuierlich, zumindest jedoch regelmäßig geöffnet werden.

### 4. Nutzer-Initialisierung

Nutzer öffnen die Nutzer-App, welche ohne Authentifizierung genutzt werden kann. Sie geben relevante personenbezogene Daten (Name, Anschrift, ggf. E-Mail und Telefonnummer) an. Die Nutzer-App generiert ein ECDH-Schlüsselpaar $(K_\mathrm{u_i}^\mathrm{enc,priv},K_\mathrm{u_i}^\mathrm{enc,pub})$ sowie einen Nonce-Wert (32 Byte Zufallsdaten), verknüpft diesen mit den angegebenen Daten und erstellt hieraus einen Hashwert $H_i$. Dieser Wert wird an das Authentifizierungsbackend gesendet, welches ein Prioritätstoken $P_i$ generiert (32-Byte Zufallsdaten) und gemeinsam mit dem Hashwert $H_i$ vom Backend signiert: $S_i^{PH} = \mathrm{sign}((H_i, P_i),K_\mathrm{be}^\mathrm{sign,priv})$. Die Signatur verknüpft das Prioritätstoken mit den von Nutzern angegebenen Daten (ohne diese zu kennen) und verhindert so, dass Nutzer das Token weiterveräußern können.

Nutzer wählen nun ein Einzugsgebiet aus und erstellen für dieses eine anonyme Terminanfrage. Diese wird ggf. mit zusätzlichen Informationen (z.B. gewünschter Impfstoff) versehen. Die Daten der Terminanfrage werden von der Nutzer-App gemeinsam mit einer zufällig generierten ID (32-Byte Zufallsdaten) mit dem öffentlichen Schlüssel des Einzugsgebiets verschlüsselt und gemeinsam mit dem Prioritätstoken ins Backend hochgeladen. Das Backend stellt hierbei sicher, dass Prioritätstoken nicht mehrfach hochgeladen werden können. Damit ist die Nutzer-Initialisierung abgeschlossen.

### 5. Terminangebot erstellen

Die von Nutzern in Einzugsgebiete hochgeladenen Terminwünsche können vom Backend anhand der angegebenen Prioritätstoken geordnet werden. Hierdurch entsteht eine dynamische, nach Priorität sortierte Liste von Nutzern für ein jeweiliges Einzugsgebiet. Anbieter-Apps fordern von diesen Listen jeweils eine kleine Anzahl von Nutzeranfragen an. Sie können diese mit dem privaten Schlüssel des Einzugsgebiets (welcher ihnen über das Backend verschlüsselt zur Verfügung gestellt wird) entschlüsseln. Sie prüfen die Terminanfrage gegen die verfügbaren Kapazitäten, erstellen Terminangebote, verschlüsseln diese gemeinsam mit den signierten Praxis-Daten mit dem öffentlichen Schlüssel des Nutzers und legen die Daten unter der von der Nutzer-App bereitgestellten ID im Backend ab. Die Daten werden zusätzlich mit dem privaten Signaturschlüssel signiert. Jedes Terminangebot wird hierbei mit einer zufälligen ID versehen.

Die Anbieter-App signalisiert dem Backend vor der Erstellung eines Terminangebots die konkrete Absicht. Das Backend assoziiert die Anfragen mit einem Prioritätstoken und entfernt dieses Token nach einer gegebenen Anzahl von erhaltenen Angeboten (zunächst temporär) aus der Prioritätsliste. Hierdurch wird vermieden, dass Nutzern die nicht responsiv sind eine sehr große Anzahl von Terminangeboten gemacht werden, während Nutzer mit schlechteren Prioritätstoken keine Angebote erhalten.

Optional generiert die App mithilfe eines externen Dienstes eine Benachrichtigung für den Nutzer (z.B. in Form einer E-Mail Benachrichtigung). Dies erfordert, dass der Nutzer in der Terminanfrage solche Kontaktdaten verschlüsselt zur Verfügung stellt (siehe unten).

Die Anbieter-App kann Terminangebote mehrfach vergeben um eine ausreichende Belegung von Terminen zu erreichen. Nutzer-Apps können bereits vergebene Termine dynamisch erkennen und ggf. eine neue Terminvergabe anfordern, falls alle Angebote bereits vergeben sind. Die optimale Überbuchungsrate hängt hierbei von verschiedenen Faktoren ab und muss im System dynamisch optimiert werden.

### 6. Terminangebot annehmen (Nutzer)

Die Nutzer-App prüft regelmäßig, ob unter der in Schritt 4 erstellten ID im Backend Daten verfügbar sind (gegebenenfalls erhält der Nutzer hierzu auch eine Benachrichtigung über einen externen Dienst). Falls ja, ruft die App diese Daten ab und entschlüsselt sie mit dem privaten Nutzer-Schlüssel. Anschließend prüft die Nutzer-App die Signaturen (Anbieter- & Vermittler-Signatur) aller Angebote und entfernt ungültige Angebote. Die gültigen Daten werden Nutzern zur Auswahl angezeigt. Diese wählen aus den Terminen einen passenden aus. Die Nutzer-App verschlüsselt nun die bei der Initialisierung angegebenen Nutzerdaten sowie das signierte Prioritätstoken mit dem in dem öffentlichen Schlüssel des Anbieters und lädt die Daten verschlüsselt mit der im Angebot angegebenen ID ans Backend hoch. Vorab kann eine Prüfung der noch verfügbaren IDs im Backend erfolgen (da Terminangebote an mehrere Nutzer parallel geschickt werden können).

### 7. Terminangebot bestätigen (Anbieter)

Die Anbieter-App prüft regelmäßig ob unter den in Schritt 5. erstellten IDs im Backend Daten verfügbar sind. Falls ja, ruft die App diese Daten ab und entschlüsselt sie mit dem privaten Schlüssel der in Schritt 5 generiert wurde. Anschließend prüft die App, ob das angegebene Prioritätstoken mit dem ursprünglich angegebenen Token übereinstimmt. Zusätzlich prüft sie, ob der signierte Hashwert mit dem von der App berechneten Hashwert der Nutzerdaten übereinstimmt. Gegebenenalls können Anbieter die angegebenen Nutzerdaten noch manuell überprüfen oder automatisch eine Verifikation über einen externen Dienst anfordern (siehe unten). Die App erstellt anschließend eine Terminbestätigung, verschlüsselt diese mit dem öffentlichen Schlüssel des Nutzers und legt sie im Backend unter der vorherigen ID ab. Optional generiert die App mithilfe eines externen Dienstes eine Benachrichtigung für den Nutzer (z.B. in Form einer E-Mail Benachrichtigung).

Damit ist die Terminvergabe abgeschlosen.

### 8. Termine absagen oder ändern

Nutzer können Termine über die App stornieren, indem sie unter der bekannten ID eine verschlüsselte Anfrage an den Anbieter senden. Die Anbieter-App kann hierauf wiederum ein neues Terminangebot generieren. Prinzipiell sind auch weitere Kommunikationsmöglichkeiten über diesen verschlüsselten Kanal denkbar, die Funktionalität kann dem Anwendungsfall hierzu leicht angepasst werden.

### Erweiterungen

Folgende Erweiterungen des Protokolls sind denkbar und können die Nutzbarkeit und Sicherheit erhöhen.

#### Speichern von Schlüsselmaterial

Da Nutzer, Anbieter und Vermittler jeweils Schlüsselmaterial benötigen um das System zu nutzen ist es wichtig, dieses Material ausfallsicher zu speichern. Dies kann beispielsweise durch eine Speicherung als Datei oder über die verschlüsselte Ablage in einem Backend erfolgen. Im letzteren Fall kann ein zufallsgeneriertes Token oder nutzergewähltes Passwort eingesetzt werden um die Daten vor dem Upload lokal zu verschlüsseln. Das Passwort/Token kann von Nutzern notiert oder in Form eines Lesezeichens gespeichert werden. Wenn ein nutzergewähltes Passwort eingesetzt wird sollte zusätzlich eine zufallsgenerierte ID zur eindeutigen Identifikation der Daten gewählt werden. Im Falle eines zufallsbasierten Tokens kann aus diesem ein Passwort sowie eine ID mithilfe einer Schlüsselableitung erstellt werden. 

#### Nutzung externer Dienste für die Verifikation & Benachrichtung von Nutzern

Die Verifikation von Daten (z.B. E-Mail Adressen oder Telefonnummern) kann ein effektives Instrument zur Bekämpfung von Missbrauch sein. Eine solche Verifikation ist jedoch nicht privatsphäre-freundlich und zudem oft aufwendig und kostenspielig. Sie sollte daher nur anlassbezogen und möglichst spät im Prozess erfolgen. Eine Benachrichtigung von Nutzern über externe Dienste (z.B. via E-Mail) ist ebenfalls oft sinnvoll, sollte aber aus den gleichen Gründen ebenfalls nur anlassbezogen erfolgen.

Um solche Dienste zu ermöglichen können Systemkomponenten erstellt werden, die z.B. E-Mail oder SMS-Versand oder Empfang realisieren. Diese sollten nur anlassbezogen und authentifiziert genutzt werden können. Hierfür können die Dienste jeweils eigene ECDH-Schlüsselpaare erhalten, dessen öffentliche Schlüssel der Nutzer-App bekannt gemacht werden. Nutzerdaten wie z.B. E-Mail Adressen oder Telefonnummern können dann in der Nutzer-App mit dem öffentlichen Schlüssel des jweiligen Dienstes verschlüsselt werden und einer Terminanfrage mitgegeben werden. Anbieter können dann mithilfe der verschlüsselten Daten und einem zusätzlichen Authentifizierungstoken eine Verifikation oder Benachrichtigung über den entsprechenden externen Dienst anfordern. Dieser kann die Nutzerdaten mit dem privaten Schlüssel entschlüsseln und z.B. eine entsprechende E-Mail oder SMS generieren. Hierdurch wird sichergestellt, dass Anbieter zu keinem Zeitpunkt Zugang zu den entschlüsselten Daten erhalten und diese Daten nur anlassbezogen und unter Mitwirkung von Nutzern externen Diensten zur Verfügung gestellt werden können. Diese Dienste können unabhängig vom restlichen System betrieben werden. Da eine Verifikation nur anlassbezogen und ggf. nach einer ersten Prüfung der Daten durch Anbieter erfolgt kann das genutzte Volumen der externen Dienste stark reduziert werden, was wiederum die Kosten und den Aufwand entsprechend reduziert. Durch eine rein authentifizierte Nutzbarkeit der Dienste wird zudem das Missbrauchspotential deutlich gesenkt.

Kontakdaten zur Nutzung in externen Diensten können auch mehrfach verschlüsselt werden um die Zweckbindung zu erhöhen. Z.B. können E-Mail Adressen oder Telefonnummern zusätzlich mit dem öffentlichen Schlüssel des Einzugsgebietes verschlüsselt werden, um nur für Anbieter aus diesem Gebiet entschlüsselbar zu sein.

#### Gestaltung von Einzugsgebieten

Einzugsgebiete können nicht nur geographisch sondern auch nach weiteren Kriterien struktuiert werden. So ist denkbar, für einzelne Impfstoffe oder Altersgruppen jeweils separate Einzugsgebiete zu definieren. Dies kann die Terminfindung verbessern und die Anzahl an Datensätzen reduzieren, die konkret von Anbietern eingesehen werden müssen. Anbieter können dann ggf. auch mehreren Einzugsgebieten zugeordnet werden (z.B. falls sie Impfungen mit mehreren Impfstoffen anbieten).

#### Erfassung und Bekämpfung von Missbrauch

Sowohl von Nutzer- als auch von Anbieterseite ergeben sich mehrere Möglichkeiten, das System zu missbrauchen:

* Akteure können sich evtl. eine Registrierung als Anbieter im System verschaffen und damit Terminangebote an Nutzer schicken. Wenn Nutzer diese Angebote annehmen und den Akteuren ihre Daten zur Verfügung stellen können diese eventuell missbraucht werden.
* Akteure können eine große Zahl Prioritätstoken generieren und mit diesen das System in seiner Funktionsfähigkeit beeinträchtigen. Sie können ebenfalls versuchen, Prioritätstoken weiterzuverkaufen.
* Akteure können versuchen, das Backend-System durch eine große Anzahl an Abfragen überwältigen.

Um diese Missbrauchsszenarien zu verhindern sollte das System Gegenmaßnahmen vorsehen. Nutzer sollten z.B. in der Lage sein, im System registrierte Anbieter die sie für nicht seriös halten zu melden. Vermittler können solche Anbieter dann durch Depublikation ihres öffentlichen Signaturschlüssels aus dem System entfernen, Nutzer-Apps entfernen dann automatisiert alle Terminangebote dieser Anbieter beim Laden der Angebote. Allerdings kann auch eine solche "Melden" Funktion missbräuchlich eingesetzt werden, sie sollte daher ebenfalls Einschränkungen unterliegen und geeignete Sicherheitsmaßnahmen implementieren.

Um eine Überwältigung des Systems durch eine hohe Anzahl missbräuchlich erstellter Prioritätstoken zu vermeiden sind mehrere Gegenmaßnahmen denkbar. Zum einen kann die Überbuchungsrate von Terminen an die Anzahl missbräuchlich erstellter Token angepasst werden. Zusätzlich kann eine Verifizierung einzelner Nutzerdaten erfolgen. Hierzu werden Daten wie eine E-Mail Adresse oder eine Telefonnummer mithilfe eines externen Dienstes verifiziert (siehe oben). Dieser Dienst generiert dann eine Signatur für die angegebenen Daten, welche zusätzlich mit Kontextinformationen wie einem Prioritätstoken verknüpft werden kann. Anbieter können Nutzeranfragen, die nicht über verifizierte Kontaktdaten verfügen so aussortieren oder von den Nutzern eine Nachverifikation verlangen. Hierzu können wiederum externe Systeme genutzt werden. Dies kann die Hürde für die Manipulation des Systems durch böswillige Nutzer erhöhen.

Um eine Überwältigung des Systems durch eine hohe Anzahl von Backend-Anfragen zu vermeiden muss das Backend darauf ausgelegt werden, auch hohen Anfragevolumina standzuhalten. Da eine Authentifizierung von Nutzern generell nicht wünschenswert ist (da sich hieraus eine Vielzahl von Möglichkeiten zur Metadatenanalyse ergeben) muss sichergestellt werden, dass Nutzer Anfragen möglichst nur mit Kontextinformationen tätigen können. Dies ist im gegebenen System möglich da Anbieter unproblematischer authentifiziert werden können und diese initiativ ID-Werte erstellen, über welche Nutzer dann Daten an das Backend schicken können. Nutzer können somit nur unter Kenntnis dieser Werte Daten im Backend speichern, und die Speicherung von Daten unter diesen IDs kann über einfache Mechanismen begrenzt werden. Lediglich für die Speicherung von verschlüsselten Einstellungen/Nutzerdaten ist dies nicht möglich, dieses Backend-System kann jedoch getrennt vom restlichen System betrieben werden und kann aufgrund der einfachen Natur der Speicherung leicht für große Datenvolumina ausgelegt werden.

#### Dezentraler & föderierter Betrieb

Alle Komponenten des Systems können prinzipiell redundant und föderiert betrieben werden. Apps können mit mehreren Backends kommunizieren und von diesen Daten abrufen bzw. Daten an diese senden. Ebenso können mehrere Vermittler ein Backend nutzen um Anbieter und Einzugsgebiete zu registrieren. So ist es vorstellbar, die Registrierung und Überwachung von Anbietern regional jeweils spezifischen Akteuren zu übertragen. Ebenso können für unterschiedliche Regionen jeweils eigene Backends betrieben werden, oder mehrere Backends können redundant zu einem ausfallsichereren System kombiniert werden. Apps können ebenfalls dezentral und unabhängig vom Backend verteilt werden, sie können so z.B. auch an regionale Gegebenheiten angepasst werden.