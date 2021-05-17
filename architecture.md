# Dezentrale Architektur sofort-impfen

## Beteiligte

- App Imfling
- App / Akteur Ärztin
- Vermittlungsserver


## App Impfling:
### Technische Basis
- App / WebApp (tbd)
- Ziel: geringe Einstiegshürde, oftmals One Time Use

### Informationen, die vorgehalten werden
- speichert Impfpräferenzen (Impfstoff)
- speichert PLZ

### Aufgaben
- fetcht Terminslots als `Termin-Token`
- nimmt Kontakt zu Arzt auf bei Terminanfrage


## App Ärztin
### Technische Basis
- Web App
- Ziel: ist einfach zu installierende Browseranwendung

### Informationen, die vorgehalten werden
- Grundinformationen zur Praxis (Name, Anschrift, Anfahrt)
- verfügbare Terminslots

### Aufgaben
- hinterlegt verfügbare Termine über `Termin-Token`
- verifiziert Terminanmeldungen anhand von Prüfsummen / Code / QR-Code


## Vermittlung
### Technische Basis
- Server-Backend

### Informationen, die vorgehalten werden
- Ziel: extrem wenig bis gar keine Informationen vorhanden
- kennt Zuordnung der PLZ Gebiete

### Funktionen
- broadcastet `Termin-Token` an Clients nach Spielregeln (zeitliche Rahmenbedingungen, etc.)

## Prozess Terminsuche:

1. Praxis stellt Anzahl an `Termin-Token` mit Datum / Impfstoff zur Verfügung
2. Vermittlung nimmt `Termin-Token` in Pool auf
3. Vermittlung verteilt `Termin-Token` nach PLZ an entsprechende Clients in gewissen Zeitabständen / Push
4. Clients suchen sich im Pool `Termin-Token` für die für sie passenden Anforderungen heraus
5. Client pullt `Termin-Token` und nimmt Kontakt zu App Ärztin auf
6. Userin bestätigt Termin und reserviert Termin bei Ärztin
7. Ärztin broadcastet neue Liste von `Termin-Token` ohne bestätigten Termin
8. Vermittlung broadcastet entsprechend aktuelle Liste von `Termin-Token`

## Prozess Terminreservierung / Termin

1. Userin und Praxis handeln `Verifizierungs-Token` aus, der dann als Bestätigung dient
2. Userin geht mit `Verifizierungs-Token` zu Termin und wird mit `Verifizierungs-Token` entsprechend von Ärztin verifiziert 

TBD: Weiterer Onboarding Flow mit Dokumenten zur Vorbereitung, Infos in QR-Code oder lesbarem Token etc. 


## Tokens

`Termin-Token` Token für Terminreservierung, bekommt Halbwertszeit, ist in der Form nur eine gewisse Zeit gültig und nur temporär auf Praxis rückrechenbar

`Verifizierungs-Token` Authtoken für Ausweisung Termin beim Arzt, besteht aus leicht lesbaren Code

## Security Implikationen
### Verhinderung von Abgreifen aller PLZ-Informationen
PLZ im Impfling-Client kann zwar geändert werden, greift dann aber erst beim nächsten Broadcast-Zyklus. Mehrere Änderungen in Folge sind zu berücksichtigen 

### Weitere Punkte tbd
- nur eine Vermittlung pro Client
- Verifikation falscher Tokens, die einseitig ins System eingebracht werden von Impflingen
etc…
