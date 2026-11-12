# Wissen und Diskursmacht zwischen tagesschau und YouTube:
## Jupyter Notebook zur Erstellung eines Korpus aus Untertiteln der _tagesschau_ und _YouTube_-Kommentaren

Dies ist das Begleitmaterial für die Einreichung _Wissen und Diskursmacht zwischen Tagesschau und YouTube-Kommentaren_ für das Sonderheft der _Zeitschrift für Flucht- und Flüchtlingsforschung_ _Flucht, Migration und Bildung im digitalen Wandel_.

Dieses Jupyter Notebook beschreibt die notwendigen Schritte zur Erstellung des untersuchten Korpus als Pseudo-Code (d. h. Beschreibung der durchgeführten Schritte in Textform). Abschnitt 5 enthält ausführbaren Python-Code für die Erstellung der Netzwerkgrafiken (Abbildungen 3 und 4 im Forumsbeitrag). Um diese reproduzierbar zu machen, werden außerdem die annotierten XML-Dateien bereitgestellt. Die für dieses Minimalbeispiel zitierten Texte werden am Ende der Read-Me-Datei aufgelistet.

---

### Bibliotheken

```python
# Requirements
requests==2.33.1
beautifulsoup4==4.8.2
pandas==2.1.3
numpy==1.26.2
networkx==3.4.2
gravis==0.1.0
google-api-python-client==2.181.0
spacy==3.7.2
```

```python
# Libraries
import requests
from bs4 import BeautifulSoup
import traceback
from datetime import datetime, timedelta, timezone, time
import time
import pandas as pd
import re
import numpy as np
import glob
import os
import ast
from googleapiclient.discovery import build
import uuid
import networkx as nx
import gravis as gv
from collections import Counter
import spacy
```

---

### Hilfsfunktionen

Die folgenden Funktionen werden in den nachfolgenden Abschnitten verwendet.

```python
# Functions used in the code cells below
def get_subs_data(url, t_url_front, page, date_list, href_list, desc_list, sub_list, log_list):
    '''
    This function scrapes the website https://www.ardmediathek.de/sendung/tagesschau/Y3JpZDovL2Rhc2Vyc3RlLmRlL3RhZ2Vzc2NoYXU for the data needed to download subtitle files at a later point.
    Usage: Should be embeded in a loop or function that loops through every url per date in the corpus.
    '''
    r = requests.get(url)                                                                           
    if r.status_code == 200:
        # Parse response as JSON                                                                        
        data = r.json()     
        teasers = data.get('teasers', [])
        # Process each item in teasers
        for teaser in teasers:
            if teaser.get('id'):
                href = teaser.get('id')  
            else:
                href = 'NA'
            if teaser.get('broadcastedOn'):
                first_date = teaser.get('broadcastedOn')    
            else:
                first_date = 'NA'
            if teaser.get('shortTitle'):
                title = teaser.get('shortTitle')
            else:
                title = 'NA'
            # Test if subs are available, to save loop later
            sub_test = teaser.get('subtitled')
            
            # Define date here, before the conditional
            date = datetime.fromisoformat(first_date.replace('Z', '+00:00')).strftime('%d-%m-%Y')
            
            if re.search(r'20.?0?0? Uhr|tagesschau', title, re.IGNORECASE):                        # Only episodes of the main broadcast @ 8 pm

                date_list.append(date)
                if not href == 'NA':
                    href_list.append(t_url_front+href)
                else:
                    href_list.append('NA')
                new_soup = BeautifulSoup(requests.get(t_url_front+href).content, 'html.parser')
                
                # Find the <meta> tag with the attribute name="description"
                meta_tag = new_soup.find('meta', attrs={'name': 'description'})
                # Extract the value of the content attribute to get the synopsis of the episode topics
                if meta_tag:
                    content_value = meta_tag.get('content')
                    desc_list.append(content_value.replace('Themen der Sendung: ', ''))
                else:
                    desc_list.append('NA')
                # Find the URL to the subtitle file
                if sub_test == True:
                    tag = new_soup.find('script', {'id':"fetchedContextValue"}).text
                    if tag:
                        sub_url = re.search(r'[\"\'\']url[\"'']:[\"\'](https://api.ardmediathek.de/player-service/subtitle/webvtt/urn:ard:subtitle:[^\.]+.vtt)[\"\'']', tag).group(1)
                        sub_list.append(sub_url)
                    else:
                        sub_list.append('NA')
                elif sub_test == False:
                    tag = new_soup.find('script', {'id':"fetchedContextValue"}).text
                    if tag and not re.search(r'[\"\'']url[\"\'']:[\"\'](https://api.ardmediathek.de/player-service/subtitle/webvtt/urn:ard:subtitle:[^\.]+.vtt)[\"\'']', tag) == None:
                        sub_url = re.search(r'[\"\'']url[\"\'']:[\"\'](https://api.ardmediathek.de/player-service/subtitle/webvtt/urn:ard:subtitle:[^\.]+.vtt)[\"\'']', tag).group(1)
                    else:
                        sub_url = 'NA'
                    sub_list.append(sub_url)
                else:
                    sub_list.append('NA')
                print(f'Date {date} sucessful')
                log_list.append(f'Date {date} sucessful')
                # give the website some time to rest
                time.sleep(55)
            else:
                print(f'--> Date {date} skipped: title does not contain 20:00 Uhr')
    else:
        print(f"Page no: {str(page)}\tFailed to fetch data: {r.status_code}")
    
    return date_list, href_list, desc_list, sub_list, log_list

def latest_file(pattern, extension):
    '''
    This function finds the last edited file in a given directory.
    Usage example:
    hit = latest_file(f'{df_path}\\Links_*.pkl', 'pkl')
    '''
    file_list = glob.glob(pattern)
    latest = max(file_list, key=os.path.getctime) 
    hit = re.search(r'^(.+?)(\d{2}-\d{2}-\d{2}_\d{2}-\d{2})\.'+extension, latest)
    return hit, latest  # return both match object and file path
```

---

### 1. Untertitel der _tagesschau_

In diesem Abschnitt werden zunächst (1.1) Informationen aus der ARD-Mediathek gescraped. Uns interessieren für jede verfügbare Sendung im Untersuchungszeitraum (20. Februar 2022 – 25. Februar 2025) jeweils die URL des Videos, die URL der Untertiteldatei, das Datum der Sendung und die Themenübersicht. Nach einem ersten Durchlauf werden die Daten auf Fehler überprüft und als Dataframe für das Herunterladen der Untertitel vorbereitet. In 1.2 werden die Untertiteldateien heruntergeladen und lokal zwischengespeichert.

#### 1.1 Metadaten und Download-URLs

Der für den Download der Untertitel benötigte Dataframe ist für den Zeitraum 20.02.2022–25.02.2025 bereits erstellt und überprüft. Um Resourcen und Zeit zu sparen, empfiehlt es sich, diesen zu verwenden.

__Anmerkung__: Um eine Belastung des Servers zu vermeiden, orientiert der Code sich an dem Abfragelimit der Tagesschau API (V. 2.0.1, https://tagesschau.api.bund.dev/) (d. h. max. 60 Abfragen pro Minute). Wegen der Zeitpuffer zur Entlastung der Website läuft der Code bis zu 20 Stunden. Es empfiehlt sich also, den übergeordneten for-Loop auf mehrere Teile aufzuteilen und den Code über Nacht laufen zu lassen.

---

## Metadaten und URLs für die Untertitel abrufen

Pseudo-Code auf Basis einer Übersetzung des ursprünglichen Python-Codes durch das Large Language Model (LLM) Claude Sonnet 4.6 (Anthropic, 2026). Die finale Fassung wurde überprüft und angepasst.

### Setup

- Arbeitsverzeichnisse für **Dataframes** (`.pkl`) und **Excel-Dateien** (`.xlsx`) festlegen
- Diese Verzeichnisse anlegen, falls sie noch nicht existieren
- **Logdatei** initialisieren

### Vorhandene Daten laden (falls vorhanden)

- Im Dataframe-Ordner nach gespeicherten `Links_*.pkl`-Dateien suchen
- Falls eine oder mehrere gefunden werden:
  - Die **aktuellste** Datei als `df_existing` laden
  - Das `previous_run`-Label und den Zeitstempel aus dem Dateinamen extrahieren
- Falls keine `PKL`-Datei gefunden wird: `previous_run = None` setzen

### Scraping-Durchlauf konfigurieren

- Den **vorderen Teil** der Ziel-URL festlegen (`t_url_front = 'https://www.ardmediathek.de/video/tagesschau/tagesschau-20-00-uhr/das-erste/'`)
- `date_list, href_list, desc_list, sub_list, log_list` als leere Listen erstellen
- Einen **Seitenbereich** und ein **Durchlauf-Label** wählen (z. B. `"first"`, `"second"`)
- `size = 30` setzen (Anzahl der Videos pro API-Seite)
- Range für die zu durchlaufenden Seitenzahlen setzen z. B. `p_range`

**Anmerkung:** `p_range` muss je nach Aktualität des ältesten verfügbaren Videos angepasst werden.
Der Parameter `pageNumber=0` enthält die jeweils neuesten Videos der Streaming-Plattform. Mit
`p_range=range(10, 48)` und `size=30` werden etwa drei Jahre Sendematerial gescraped (Stand
24. November 2025 deckt dies den gesamten Korpuszeitraum ab), allerdings beträgt die Laufzeit
des Codes ca. 20 Stunden. Es empfiehlt sich daher, den Bereich in drei Teile aufzuteilen, z. B.:

- Teil 1: `range(10, 22)`
- Teil 2: `range(23, 35)`
- Teil 3: `range(36, 48)`

### Seiten scrapen

Für jede `page` im Seitenbereich:
1. Warnung ausgeben und abbrechen, falls das aktuelle **Durchlauf-Label bereits im Dateinamen** eines gespeicherten Dataframes vorkommt
2. Die **API-URL** für die jeweilige Seite zusammensetzen: `url = f'https://api.ardmediathek.de/page-gateway/widgets/ard/asset/Y3JpZDovL2Rhc2Vyc3RlLmRlL3RhZ2Vzc2NoYXU?pageNumber={page}&pageSize={size}'`
3. `get_subs_data()` aufrufen → fügt den jeweiligen Listen Datum, Links, Beschreibungen und Untertitel-URLs hinzu
4. **60 Sekunden warten**, um den Server zu schonen (siehe https://tagesschau.api.bund.dev/)
5. Erfolg oder Fehler im Log festhalten; eine Fangnetzoption für Fehler einbauen z. B. `try: ... except: continue`

### Ergebnisse speichern

- Die neu gesammelten Listen zu einem **DataFrame** zusammenführen
- Falls zuvor ein bestehender DataFrame geladen wurde: alten und neuen **zusammenführen** und als Pickle (`PKL`) speichern
- Andernfalls: den neuen DataFrame direkt als Pickle speichern
- Dateinamen-Format: `Links_{run}_{timestamp}.pkl`

### Fehlgeschlagene Einträge exportieren

- Zeilen herausfiltern, bei denen `Sub_Link == 'NA'` (Untertitel-URL konnte nicht abgerufen werden)
- Diese Zeilen zur manuellen Nachbearbeitung als **Excel-Datei** speichern: `Links_NA_{timestamp}.xlsx`

### Log schreiben

- End-Zeitstempel festhalten
- Das vollständige Log in eine `.txt`-Datei schreiben: `get_metadata_log_{timestamp}.txt`

---
Ende dieses Abschnitts der LLM-Übersetzung.

## Sendungen ohne Untertitellink überprüfen

In 1.1 wurde die Exceldatei `Links_NA_DATUM.xlsx` exportiert, die Zeilen enthält, für die kein Link zur Untertiteldatei gefunden wurde. Die Dateien können noch manuell überprüft werden, um eventuell falsch annotierte Sendungen zu finden. Die manuelle Nachannotation wird in der Spalte `Anno` festgehalten (`yes|no`), damit diese im nächsten Schritt wieder maschinell verarbeitet werden kann.

Falls bei der Nachannotation Sendungen identifiziert werden, für die der Code oben keine URL zur VTT-Datei hervorgebracht hat, kann diese manuell über die Netzwerkanalyse eines Browsers kopiert werden.

---

## Excel-Datei bereinigen

Pseudo-Code auf Basis einer Übersetzung des ursprünglichen Python-Codes durch das Large Language Model (LLM) Claude Sonnet 4.6 (Anthropic, 2026). Die finale Fassung wurde überprüft und angepasst.

### Setup
- **Logdatei** initialisieren
- Annotationscode festlegen z. B. `anno = 'yes'`
- Arbeitsverzeichnisse für **Dataframes** (`.pkl`) und **Excel-Dateien** (`.xlsx`) festlegen

### Vorhandene Dateien laden
- Mit `latest_file()` die aktuellste `Links_*.pkl`-Datei im Dataframe-Ordner suchen und laden
- Mit `latest_file()` die aktuellste `Links_*.xlsx`-Datei im Excel-Ordner suchen und laden
- Falls keine passende Datei gefunden wird oder der Dateiname nicht dem erwarteten Muster entspricht: Fehlermeldung ausgeben und im Log festhalten

### Excel-Datei verarbeiten
- Aus der Excel-Datei nur die Zeilen herausfiltern, bei denen `Anno == 'yes'`
  (d. h. manuell annotierte Einträge, die nachträglich ergänzt werden sollen)
- Die Spalte `Anno` aus diesem gefilterten DataFrame entfernen

### Dataframes zusammenführen und speichern
- Gefilterten DataFrame mit dem bestehenden Pickle-DataFrame **zusammenführen**
- Das Ergebnis als neue Pickle-Datei speichern: `Links_final_{timestamp}.pkl`
- Anzahl der hinzugefügten Zeilen im Log festhalten

### Log schreiben
- End-Zeitstempel festhalten
- Das vollständige Log in eine `.txt`-Datei schreiben: `manual_annotation_log_{timestamp}.txt`

---
Ende dieses Abschnitts der LLM-Übersetzung.

### 2. Herunterladen der Untertitel

Nun können die VTT-Dateien heruntergeladen werden. Da auch hier die ARD-Mediathek sehr oft angefragt wird, wird das Abfragelimit der tagesschau API als Orientierung genommen. Demnach kann auch dieser Prozess in mehreren Durchläufen durchgeführt werden.

## Untertiteldateien herunterladen

Pseudo-Code auf Basis einer Übersetzung des ursprünglichen Python-Codes durch das Large Language Model (LLM) Claude Sonnet 4.6 (Anthropic, 2026). Die finale Fassung wurde überprüft und angepasst.

### Setup
- **Logdatei** initialisieren
- Arbeitsverzeichnisse für **Untertitel** (`.txt`), **Dataframes** (`.pkl`) festlegen; Ausgabeordner anlegen, falls nicht vorhanden

### Aktuellsten finalen Dataframe laden
- Mit `latest_file()` die aktuellste `Links_final_*.pkl`-Datei laden
- **Übergabe:** `sub_df` (enthält Datum und Untertitel-URL für jede Sendung)
- Falls keine Datei gefunden wird oder der Dateiname nicht dem Muster entspricht: Fehlermeldung ausgeben und im Log festhalten

**Anmerkung:** Die Datei `tagesschau_korpus.csv`enthält bereits die Links zum Video und der Untertiteldatei, die YouTube-Video-ID und Kommentare zur Verfügbarkeit der Untertitel und dem Datumsabgleich für den gesamten Untersuchungszeitraum. Der Code kann bei einer Reproduktion dementsprechend angepasst werden, um Ressourcen zu sparen.

### Untertitel herunterladen
Für jede Sendung (`date`) im Dataframe `sub_df`:
1. Untertitel-URL (`Sub_Link`) für das jeweilige Datum abrufen
2. Falls `Sub_Link == 'NA'`: Eintrag überspringen und als nicht verfügbar vermerken
3. Andernfalls: HTTP-GET-Anfrage an die Untertitel-URL senden und Antworttext speichern
4. Datei lokal als `{date}.txt` im Unterordner `subtitles/` abspeichern
5. **10 Sekunden warten**, um den Server zu schonen
6. Fehler abfangen (`try: ... except: continue`) und im Log festhalten

### Dataframe aktualisieren und speichern
- Dateinamen der heruntergeladenen Dateien als neue Spalte `Datei` dem Dataframe hinzufügen
- Aktualisierten Dataframe als `Sub_Links_{timestamp}.pkl` speichern
- **Übergabe:** `Sub_Links_{timestamp}.pkl` (wird in Abschnitt 2.1 weiterverwendet)

### Log schreiben
- End-Zeitstempel festhalten
- Das vollständige Log in eine `.txt`-Datei schreiben: `scrape_subtitles_log_{timestamp}.txt`

---
Ende dieses Abschnitts der LLM-Übersetzung.

#### 2.1 Datumsabgleich zwischen Dateinamen und Mediathek

## Datumsabgleich durchführen

Pseudo-Code auf Basis einer Übersetzung des ursprünglichen Python-Codes durch das Large Language Model (LLM) Claude Sonnet 4.6 (Anthropic, 2026). Die finale Fassung wurde überprüft und angepasst.

### Setup
- **Logdatei** initialisieren
- Arbeitsverzeichnisse für **Dataframes** (`.pkl`) und **Excel-Dateien** (`.xlsx`) festlegen

### Aktuellsten Untertitel-Dataframe laden
- Mit `latest_file()` die aktuellste `Sub_Links_*.pkl`-Datei laden
- **Übergabe:** `sub_df` aus Abschnitt 2
- Falls keine Datei gefunden wird: Fehlermeldung ausgeben und im Log festhalten

### Datumsabgleich
Für jede Sendung (`date`) im Dataframe `sub_df`:
1. Video-URL (`Link`) für das jeweilige Datum abrufen
2. Falls `Link == 'NA'`: Eintrag als nicht verfügbar vermerken und überspringen
3. Andernfalls: Sendungsseite der ARD-Mediathek aufrufen und das dort angezeigte Datum extrahieren
4. Abgerufenes Datum mit dem Datum im Dataframe vergleichen:
   - Stimmen die Daten überein → Status `'OK'` vermerken
   - Stimmen die Daten **nicht** überein → Status `'Date mismatch'` vermerken und `mismatch = True` setzen
5. **10 Sekunden warten**, um den Server zu schonen
6. Fehler abfangen (`try: ... except: continue`) und im Log festhalten

**Anmerkung:** Bei Datumsabweichungen kann die korrekte Zuordnung durch Öffnen der jeweiligen `.txt`-Datei manuell überprüft werden. Die Datumsangabe findet sich in den ersten Zeilen der Untertitel, z. B.: `<c.textBlue>live untertitelt (28.01.2024)</c>`.

### Ergebnisse speichern
- Abgleichsergebnisse als neue Spalte `Datumsabgleich` dem Dataframe hinzufügen
- Falls **keine** Datumsabweichung vorliegt (`mismatch == False`): als `Final_Links_{timestamp}.pkl` speichern
  - **Übergabe:** `Final_Links_{timestamp}.pkl` (wird in Abschnitt 2.2 und 4 weiterverwendet)
- Falls eine oder mehrere Abweichungen vorliegen (`mismatch == True`): zur manuellen Überprüfung als `Final_Links_{timestamp}.xlsx` exportieren

### Log schreiben
- End-Zeitstempel festhalten
- Das vollständige Log in eine `.txt`-Datei schreiben: `date_verification_log_{timestamp}.txt`

---
Ende dieses Abschnitts der LLM-Übersetzung.

#### 2.2 Letzte Überprüfung

Sind alle Dateien im Ordner `subtitles/` vorhanden, die dort sein sollten?

## Vollständigkeit der Untertiteldateien prüfen

Pseudo-Code auf Basis einer Übersetzung des ursprünglichen Python-Codes durch das Large Language Model (LLM) Claude Sonnet 4.6 (Anthropic, 2026). Die finale Fassung wurde überprüft und angepasst.

### Schritt 1: Ordnerinhalt mit Dataframe abgleichen
- Mit `latest_file()` die aktuellste `Sub_Links_*.pkl`-Datei laden
- Für jede Datei im Ordner `subtitles/`: Datum aus dem Dateinamen extrahieren und mit dem entsprechenden `Datumsabgleich`-Eintrag im Dataframe vergleichen
- Abweichungen oder fehlende Einträge ausgeben

### Schritt 2: Dataframe mit Ordnerinhalt abgleichen
- Mit `latest_file()` die aktuellste `Final_Links_*.pkl`-Datei laden
- **Übergabe:** `Final_Links_{timestamp}.pkl` aus Abschnitt 2.1
- Für jede Zeile im Dataframe prüfen, ob die zugehörige Datei (`{date}.txt`) im Ordner `subtitles/` vorhanden ist
- Ergebnis (`'Ja'` / `'Nein'`) als neue Spalte `Datei_vorhanden` festhalten
- Aktualisierten Dataframe als `Final_Links_after-check_{timestamp}.pkl` speichern
- **Übergabe:** `Final_Links_after-check_{timestamp}.pkl` (wird in Abschnitt 4 weiterverwendet)

---
Ende dieses Abschnitts der LLM-Übersetzung.

### 3. Herunterladen der YouTube-Daten

In diesem Abschnitt werden die YouTube-Daten mit der YouTube Data API v3 heruntergeladen und in den beiden Dataframes _videos_df_ und _comments_df_ gespeichert. Diese müssen nach Datenmanagementplan gelöscht werden, sobald der Anonymisierungsprozess abgeschlossen ist.

Bitte die Nutzerhinweise der YouTube Data API beachten. Wichtig ist vor allem das Kontingent von 10.000 Abfragetokens pro Tag. Das ist ausreichend für die hier durchgeführten Schritte, aber bei mehreren Durchläufen sollte etwa nach dem dritten Durchlauf eine 24-stündige Pause eingelegt werden. So kann sichergestellt werden, dass das Kontingent nicht überschritten wird.

```python
api_key = 'YOUR-API-KEY-HERE'           # Paste your API-key here
youtube = build('youtube', 'v3', developerKey=api_key)
playlist_id = 'PL4A2F331EE86DCC22' # Playlist of 8pm tagesschau broadcasts
```

Die folgenden Hilfsfunktionen werden für den Abruf der YouTube-Daten verwendet:

```python
def parse_timestamps(description):
    """Extract timestamp-title pairs from description text."""
    lines = description.split('\n')
    timestamps = []
   
    for line in lines:
        line = line.strip()
        # Match lines starting with MM:SS or HH:MM:SS format
        match = re.match(r'^(\d{1,2}:\d{2}(?::\d{2})?)\s+(.+)', line)
        if match:
            timestamp = match.group(1)
            title = match.group(2).strip()
            timestamps.append((timestamp, title))
   
    return timestamps

def get_comments_for_video(youtube, video_id):
    """
    Get all comments for a single video.
    Usage note: video_id is already recorded in tagesschau_korpus.csv for every broadcast that is available on YouTube and as a subtitled version on the ARD streaming plattform. The code can be adjusted to loop only through those video_ids.
    """
    try:
        request = youtube.commentThreads().list(
            part="snippet, replies",
            videoId=video_id,
            maxResults=100
        )
        comments = []
       
        # Execute the first request
        response = request.execute()
       
        # Process all pages
        while True:
            # Get the comments from the current response
            for item in response['items']:
                comment = item['snippet']['topLevelComment']['snippet']
                public = item['snippet']['isPublic']
                reply_count = item['snippet']['totalReplyCount']
                com_id = item['snippet']['topLevelComment']['id']
                level = 'oc'
                reply_to = 'NA'
                comments.append([
                    com_id,
                    level,
                    reply_to,
                    comment['authorDisplayName'],
                    comment['publishedAt'],
                    comment['likeCount'],
                    comment['textOriginal'],
                    comment['videoId'],
                    public,
                    reply_count
                ])
               
                # Process replies if they exist
                if reply_count > 0 and 'replies' in item:
                    for reply in item['replies']['comments']:
                        reply_comment = reply['snippet']
                        public_reply = 'NA'
                        reply_count_reply = 'NA'
                        rep_id = reply['id']
                        level_reply = 'rc'
                        reply_to_id = reply_comment['parentId']
                        comments.append([
                            rep_id,
                            level_reply,
                            reply_to_id,
                            reply_comment['authorDisplayName'],
                            reply_comment['publishedAt'],
                            reply_comment['likeCount'],
                            reply_comment['textOriginal'],
                            reply_comment['videoId'],
                            public_reply,
                            reply_count_reply
                        ])
           
            # Check if there's a next page
            if 'nextPageToken' not in response:
                break
               
            # Create and execute request for next page
            nextPageToken = response['nextPageToken']
            nextRequest = youtube.commentThreads().list(
                part="snippet, replies",
                videoId=video_id,
                maxResults=100,
                pageToken=nextPageToken
            )
            response = nextRequest.execute()
        
        return comments
    
    except Exception as e:
        print(f"Error fetching comments for video {video_id}: {e}")
        return []

def get_playlist_with_comments(
        youtube, 
        playlist_id, 
        stop_date="01-10-2025"      # Adjust to your projects needs
        ):
    """
    Get playlist videos with timestamps AND their comments in one go.
    Returns tuple of (videos_df, comments_df)
    """
    from datetime import datetime
    
    stop_date_obj = datetime.strptime(stop_date, "%d-%m-%Y")
   
    request = youtube.playlistItems().list(
                part='snippet, contentDetails',
                playlistId = playlist_id,
                maxResults = 50)
    response = request.execute()
    
    video_details = []
    all_comments = []
    videos_processed = 0
   
    while True:
        for item in response['items']:
            video_id = item['contentDetails']['videoId']
            timestamps = parse_timestamps(item['snippet']['description'])                  
            date_match = re.search(r'(\d{2})\.(\d{2})\.(\d{4})', item['snippet']['title'])
            date = f'{date_match.group(1)}-{date_match.group(2)}-{date_match.group(3)}' if date_match else None
            
            # Check if we've reached the stop date
            if date:
                current_date_obj = datetime.strptime(date, "%d-%m-%Y")
                if current_date_obj < stop_date_obj:
                    print(f"Reached cutoff date. Last video: {date}")
                    print(f"Total videos processed: {videos_processed}")
                    break
            
            # Add video details
            video_details.append([video_id, date, timestamps])
            
            # Get comments for this video
            print(f"Processing video {videos_processed + 1}: {item['snippet']['title'][:50]}...")
            video_comments = get_comments_for_video(youtube, video_id)
            all_comments.extend(video_comments)
            
            videos_processed += 1
        else:
            # Check if there's a next page (only if we didn't break from inner loop)
            if 'nextPageToken' not in response:
                break
               
            nextPageToken = response['nextPageToken']
            nextRequest = youtube.playlistItems().list(
                    part='snippet, contentDetails',
                    playlistId = playlist_id,
                    pageToken=nextPageToken,
                    maxResults = 50)
            response = nextRequest.execute()
            continue
        
        # If we broke from inner loop (reached stop date), break outer too
        break
   
    # Create DataFrames
    videos_df = pd.DataFrame(video_details, columns=['video_id', 'date', 'timestamps'])
    comments_df = pd.DataFrame(all_comments, columns=[
        'id', 'level', 'reply_to', 'author', 'updated_at',
        'like_count', 'text', 'video_id', 'public', 'reply_count'
    ])
    
    print(f"\nSummary:")
    print(f"Videos collected: {len(videos_df)}")
    print(f"Comments collected: {len(comments_df)}")
    
    return videos_df, comments_df

# Run the functions to get the data from the API:
# videos_df, comments_df = get_playlist_with_comments(youtube, "PL4A2F331EE86DCC22", stop_date="01-10-2025")
```

## YouTube-Daten abrufen

Pseudo-Code auf Basis einer Übersetzung des ursprünglichen Python-Codes durch das Large Language Model (LLM) Claude Sonnet 4.6 (Anthropic, 2026). Die finale Fassung wurde überprüft und angepasst.

### Playlist und Kommentare abrufen
- `get_playlist_with_comments()` aufrufen mit `playlist_id` und `stop_date`
- Die Funktion iteriert rückwärts seitenweise durch die Playlist und ruft für jedes Video `get_comments_for_video()` auf.
- Gestoppt wird, sobald ein früheres Datum als `stop_date` erreicht ist.
- Ergebnis: zwei Dataframes:
  - `videos_df`: Video-ID, Datum, Zeitstempel-Titel-Paare
  - `comments_df`: Kommentar-ID, Ebene (Originalkommentar/Antwort), Autor, Datum, Likes, Text, Video-ID
- **Übergabe:** `videos_df` und `comments_df` (werden in Abschnitt 4 weiterverwendet)

---

Ende dieses Abschnitts der LLM-Übersetzung.

**Anmerkung:** Bei öffentlichen Social-Media-Daten sind Datenschutz- und Urheberrecht zu beachten. Für die Ausarbeitung eines Datenmanagementplans sollten Art. 9 und Art. 14 der europäischen Datenschutz-Grundverordnung (Verordnung 2016/679, https://eur-lex.europa.eu/eli/reg/2016/679/oj?uri=CELEX:32016R0679) sowie § 2 und § 51 des deutschen Urheberrechtsgesetzes (UrhG) herangezogen werden.

#### Videos_df ergänzen

Bei manchen Videos fehlen die Beitragstitel und Sprungmarken in der Videobeschreibung, sind stattdessen aber in einem Kommentar der Tagesschau gepostet. Der Dataframe `videos_df` wird um diese ergänzt.

---

### 4. Parsen der Daten von YouTube und der Tagesschau

#### __Beim Parsen beachten__
- Für manche Daten gibt es entweder kein Video in der Mediathek oder es sind keine Untertitel vorhanden.
- Manchmal gibt es in der Themenübersicht einen *Hinweis* oder eine *Korrektur*. Diese sind dann auch so betitelt. Als separates Metaattribut angeben.
- Manchmal steht in der Beschreibung nur *tagesschau 20:00 Uhr*. Dies muss in der vrt-Datei ersetzt werden durch *description="NA"*.
- Der Haupttext ist immer in weiß erzählt. Sekundärtext (z. B. übersetzter Text) in anderen Farben. Blau für Paratext (z. B. non-verbaler Ton wie \* Gong \*). Metakommentare in Klammern.
- Der Name des/der Sprecher_in erscheint überwiegend an der selben Stelle in jeder VTT-Datei, in der Begrüßung (_Heute für sie im Studio ..._), jedoch nicht immer.
- Wenn Sprecher_innen automatisch erfasst werden, sollte dieses Attribut überprüft werden.

Komplette Liste für den hier untersuchten Zeitraum:

- `01-10-2022.txt`: → Sprecher_in nicht in Untertiteln benannt (im Ton: Constantin Schreiber)
- `02-05-2023.txt`: → Sprecher nicht in Untertiteln benannt (im Ton: Susanne Daubner)
- `06-03-2022.txt`: → Sprecher nicht in Untertiteln benannt (im Ton: Jens Riewa)
- `06-08-2022.txt`: *Guten Abend, meine Damen und Herren, ich begrüße Sie zur tagesschau.* → Sprecher nicht in Untertiteln benannt (im Ton: Susanne Daubner)
- `07-08-2022.txt`: *Guten Tag, meine Damen und Herren, ich begrüße Sie zur tagesschau.* → Sprecher: Jens Riewa
- `08-09-2024.txt`: *Guten Abend. Ich begrüße Sie zur 25.000 Ausgabe der tagesschau um 20 Uhr.* → Sprecherin: Susanne Daubner
- `08-12-2024.txt`: *Jubel und Freudenschüsse in Damaskus.* Begrüßung erst bei 00:00:30 → Sprecher nicht benannt (Jens Riewa)
- `22-03-2023.txt`: *Guten Abend, ich begrüße Sie zur tagesschau.* → Sprecher nicht benannt (Constantin Schreiber)
- `27-01-2025.txt`: *Was war, können wir nicht mehr ändern. Aber es darf nie wieder geschehen.* Begrüßung erst später → Sprecher nicht benannt (Constantin Schreiber)

#### Sprachmodell laden

Für die Tokenisierung, das Part-of-Speech-Tagging und die Named Entity Recognition wird die Python-Bibliothek spaCy 3.7.2 (Montani et al., 2023) verwendet.

```python
# Das hier verwendete Sprachmodell herunterladen
python -m spacy download de_core_news_sm
# spacy importieren und Sprachmodell laden 
import spacy
nlp = spacy.load('de_core_news_lg')
```

---

#### 4.1 Nicht-sprachlichen Text definieren

In den Untertiteln wird sowohl Nicht-Sprachliches als auch Metanarration (Übersetzungen, Archivton, etc.) mit Asterisk (`*`) vorne und hinten eingegrenzt. Diese beiden Typen müssen unterschieden werden.

## Nicht-verbale Elemente identifizieren

Pseudo-Code auf Basis einer Übersetzung des ursprünglichen Python-Codes durch das Large Language Model (LLM) Claude Sonnet 4.6 (Anthropic, 2026). Die finale Fassung wurde überprüft und angepasst.

### Setup
- Arbeitsverzeichnisse für **Untertitel**, **Dataframes** und **Excel-Dateien** festlegen
- Mit der Funktion `latest_file()` die aktuellste `Sub_Links_*.pkl`-Datei laden
- **Übergabe:** `Sub_Links_{timestamp}.pkl` aus Abschnitt 2
- Dataframe auf Zeilen filtern, bei denen `Datei_vorhanden == 'Ja'`
- Wörterbuch `secret_anchor_dict` mit manuell annotierten Sprecher:innen für Sonderfälle (s. o.) definieren

### Nicht-verbale Elemente sammeln
Für jede Datei im Ordner `subtitles/`:
1. Datum, Jahr und Monat aus dem Dateinamen extrahieren
2. Datei öffnen und Inhalt einlesen
3. Alle mit Asterisk (`*...*`) ausgezeichneten Strings extrahieren (potenzielle nicht-verbale Elemente)
4. Gefundene Strings und zugehörige Dateinamen in Listen speichern

### Ergebnisse speichern
- Gesammelte nicht-verbale Elemente als DataFrame zusammenstellen (`nv_df`)
- Als Pickle-Datei speichern: `nonverbal_{timestamp}.pkl`
- **Übergabe:** `nonverbal_{timestamp}.pkl` dient als Grundlage für die manuelle Klassifikation (s. u.)

---

#### Non-verbales annotieren

Die Untertitel enthalten non-verbalen Text z. B. _\*Gong\*_. Dieser muss von verbalem Text mit derselben Auszeichnung z. B. _\*Dies ist ein Beispiel\*_ getrennt werden. So kann das Korpus nicht-sprachlicher Text aus Suchabfragen herausgefiltert oder explizit gesucht werden. Für den Untersuchungszeitraum 20. Februar 2022 bis 25. Februar 2025 sind die entsprechenden Strings bereits in der Liste `nv_ls` in Abschnitt 4.2 übertragen.

---

#### 4.2 Textparsing

In diesem Abschnitt werden die Untertitel und die YouTube-Kommentare geparsed. Ausgegeben werden einzelne XML-Dateien für jede TS-Sendung, worin die jeweiligen Untertitel und YouTube-Kommentare enthalten sind, und eine VRT-Datei, mit der das Korpus kompiliert werden kann. Die VRT-Datei wird erstellt für die Kompilation des Korpus mit der IMS Open Corpus Workbench (Evert und Hardie, 2011). URL: https://cwb.sourceforge.io/.

Die folgenden Hilfsfunktionen werden für das Parsing verwendet:

```python
def conv_time(time_str):
    """
    Converts a time string HH:MM:SS[.fff] into seconds (float).
    E.g. "00:00:02.960" -> 2.96
    """
    hours, minutes, seconds = time_str.split(':')
    return int(hours) * 3600 + int(minutes) * 60 + float(seconds)
def conv_url(url_in, st, et):
    """
    Creates a URL that displays the individual video frame. To create well-formed XML, the & between startTime and endTime must be inserted as an HTML entity. However, this means the video doesn't end but simply continues playing. The start time still works.
    -> To display only the respective video frame, replace the HTML entity &amp; with an & character again
    """
    video_id = url_in.rstrip('/').split('/')[-1]
    fmt = f'{{st:.2f}}'  
    url_out = (
        f'https://www.ardmediathek.de/embed/{video_id}?startTime={fmt.format(st=st)}&amp;endTime={fmt.format(st=et)}'
    )
    return url_out
def escape_xml_attr(text):
    return re.sub(r'[&<>"\'']', lambda m: {
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&apos;'
    }[m.group()], text)
def find_section(ts_str, parsed_sections):
    """"
    Maps a time to a segment title.
    """
    ts_time = datetime.strptime(ts_str, "%H:%M:%S.%f").time()
    for i in range(len(parsed_sections) - 1, -1, -1):
        if parsed_sections[i][0] <= ts_time:
            unescaped = parsed_sections[i][1].rstrip(',')
            escaped = escape_xml_attr(unescaped)
            return escaped
        
    return None
def replace_mentions_efficient(text, uid_mapping):
    # Find all @mentions in the text
    mentions = re.findall(r'[^"]@[^ "]+', text)
    
    # Only process mentions that exist in our mapping
    for mention in set(mentions):  # Use set to avoid duplicates
        if mention in uid_mapping:
            text = text.replace(mention, uid_mapping[mention])
        else:
            text = text.replace(mention, '@user_404')
    return text
def tag_mentions(token, frame_dict, frame_url_dict, frame_text_dict):
    """Wrap references to points in time in the video into an XML tag"""
    match = re.search(r'([0-9]{1,2}:[0-9]{2})', token)
    # Convert matched time to datetime.time object
    ts = datetime.strptime(match.group(1), "%M:%S").time()
    mention_tag = None
    for f, (beg, end) in frame_dict.items():
        # Ensure beg and end are also datetime.time objects
        if isinstance(beg, str):
            beg = datetime.strptime(beg, "%H:%M:%S.%f").time()
        if isinstance(end, str):
            end = datetime.strptime(end, "%H:%M:%S.%f").time()
        if beg <= ts <= end:
            if mention_open:
                mention_tag = '</mention>'
            else:
                mention_tag = (f'<mention frame_no="{f}" frame_url="{frame_url_dict[f]}" frame_text="{html.escape(frame_text_dict[f], quote=True)}">')
            # break  # Stop after first match
    if mention_tag is None:
        mention_tag = '</mention>' if mention_open else '<mention frame_no="NA" frame_url="NA" frame_text=\"NA\">'
    return mention_tag
```
---
Ende dieses Abschnitts der LLM-Übersetzung.

## Untertitel und Kommentare parsen

Pseudo-Code auf Basis einer Übersetzung des ursprünglichen Python-Codes durch das Large Language Model (LLM) Claude Sonnet 4.6 (Anthropic, 2026). Die finale Fassung wurde überprüft und angepasst.

### Setup
- Arbeitsverzeichnisse für **Untertitel**, **Dataframes** und **XML-Ausgabe** festlegen; XML-Ordner anlegen, falls nicht vorhanden
- Mit `latest_file()` die aktuellste `Final_Links_after-check_*.pkl`-Datei laden
- **Übergabe:** `Final_Links_after-check_{timestamp}.pkl` aus Abschnitt 2.2
- Dataframe auf Zeilen filtern, bei denen `Datei_vorhanden == 'Ja'` → `ja_df`
- YouTube-Dataframes laden: `videos_df.pkl` und `comments_df.pkl`
- **Übergabe:** `videos_df` und `comments_df` aus Abschnitt 3
- Leere/fehlende Autorennamen in `comments_df` durch `'@anon'` ersetzen
- Anonymisierungstabelle (`anon.xlsx`) laden und als Wörterbuch `uid_mapping` bereitstellen (`username → uid`)
- `comments_df` und `videos_df` über `video_id` zusammenführen → `yt_df`
- Liste nicht-verbaler Elemente `nv_ls` definieren (aus Abschnitt 4.1)
- Wörterbuch `secret_anchor_dict` mit manuell annotierten Sprecher:innen für Sonderfälle definieren

### Für jede Untertiteldatei im Ordner `subtitles/`:

#### Metadaten extrahieren
1. Datum, Jahr, Monat aus Dateinamen extrahieren
2. Text-ID und Video-URL aus `ja_df` abrufen
3. Beitragstitel und Zeitstempel aus `videos_df` abrufen; falls nicht vorhanden: `sec_check = 'no'`
4. Datei öffnen und Inhalt einlesen
5. Anchorperson identifizieren: zunächst in `secret_anchor_dict` nachschlagen; andernfalls Begrüßungsformel *„Heute im Studio: …"* per Regex aus dem Untertiteltext extrahieren

#### Untertitel parsen
6. Untertitelblöcke bereinigen (Sonderzeichen, Zeitstempel normalisieren) und in Frames aufteilen
7. Für jeden Frame:
   - Frame-Nummer und Zeitstempel extrahieren
   - Textfarbe identifizieren → Diskursebene (Sprecher_in, Beitragsreporter_in, interviewte Personen) bestimmen (`main`, `meta`, `secondary`, `main/secondary`)
   - Frame-URL erstellen mit `conv_url()`
   - Beitragstitel zuordnen mit `find_section()`; bei Beitragswechsel: `<section>`-Tag öffnen/schließen
     - Hier wird die Videoeinteilung der YouTube-Versionen der Tagesschausendungen verwendet, falls vorhanden. Dafür wird der parameter `sec_check` in video_df festgehalten (s. o.).
   - Text von HTML-Tags bereinigen
   - Mit `*...*` ausgezeichnete Passagen klassifizieren:
     - In `nv_ls` enthalten → als `<nonverbal>` auszeichnen
     - Nicht in `nv_ls` → als `<trans>` (Übersetzung/Metanarration) auszeichnen
   - Text mit spaCy (Montani et al., 2023) tokenisieren, POS-taggen und ggf. Named Entities erkennen
     - POS = Part-of-Speech d. h. Wortart
   - Für das VRT-Format in diesem Schritt bereits im vertikalisierten Format in Spalten (`Token\tPOS\tLemma\tNE`) ausgeben
   - Für die XML-Version nur Tokentext zurückgeben

#### Kommentare parsen
8. Zum jeweiligen Sendungsdatum passende Originalkommentare aus `yt_df` filtern
9. Für jeden Originalkommentar:
   - `<thread>`- und `<post>`-Tags mit Metadaten (ID, anonymisierter Autor, Likes, Datum) erstellen
   - `@`-Verweise durch anonymisierte IDs aus `uid_mapping` ersetzen; nicht auflösbare Verweise als `@unknown` kennzeichnen und separat protokollieren
   - Zeitstempel-Verweise (z. B. `12:34`) mit `tag_mentions()` in `<mention>`-Tags verpacken
   - Text tokenisieren, taggen und ausgeben (VRT- und XML-Format, s. o.)
   - Die Struktur von Ursprungspost (OP) und Antwort(en) wird beibehalten, um den Kontext nachvollziehen zu können.

### Ergebnisse speichern
- Alle geparsten Sendungen als zusammenhängende VRT-Datei speichern: `tagesschau_{timestamp}.vrt`
- Nicht auflösbare `@`-Verweise als Excel-Datei exportieren: `replaced_tokens_{timestamp}.xlsx`
- Vollständiges Log speichern: `log_TS_{timestamp}.txt`

---

Ende der LLM-Übersetzung des Python-Codes in Pseudo-Code.

### Quellangaben

Anthropic. (2026). *Claude Sonnet 4.6* [Large language model]. https://www.anthropic.com/claude/sonnet

Evert, Stefan & Hardie, Andrew (2011), Twenty-first century Corpus Workbench: Updating a query architecture for the new millennium, in: _Proceedings of the Corpus Linguistics 2011 conference_. Birmingham, University of Birmingham, UK.

Montani, Ines et al. (2023). *explosion/spaCy: v3.7.2: Fixes for APIs and requirements* (v3.7.2). Zenodo. https://doi.org/10.5281/zenodo.10009823.