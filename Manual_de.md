# Handbuch der Screenplay-Skriptsprache

## 1. Einleitung

Dieses Dokument beschreibt die Skriptsprache zur Automatisierung und Steuerung von Präsentationen innerhalb des InfoRapid KnowledgeBase Builders. Die Skripte dienen dazu, eine Abfolge von Aktionen zu definieren, die automatisch ausgeführt werden, um dynamische und interaktive Präsentationen zu erstellen.

Ein Skript ist im Kern eine JSON-Datei, die ein Array von "Aktionsobjekten" enthält. Jedes Objekt repräsentiert einen einzelnen Schritt in der Präsentation, wie z.B. das Klicken eines Elements, das Eingeben von Text oder das Aufrufen komplexerer Makros.

### Grundbegriffe
- **Item**: Ein Knoten im Mindmap-Diagramm.
- **Relation**: Eine Verbindungskante zwischen zwei Items.
- **Aktion**: Ein einzelner Befehl innerhalb eines Skripts, repräsentiert durch ein JSON-Objekt.
- **Makro**: Eine vordefinierte Abfolge von Aktionen, die mit einem einzigen Befehl aufgerufen werden kann, um komplexe Operationen zu vereinfachen.

## 2. Grundkonzepte

### 2.1. Skript-Struktur

Ein Skript ist ein JSON-Array von Aktionsobjekten. Die Aktionen werden strikt in der Reihenfolge ausgeführt, in der sie im Array definiert sind.

**Beispiel:**
```json
[
  { "a": "ci", "i": ".id-show-sidebar" },
  { "a": "spt", "t": "Die Seitenleiste wird nun geöffnet." },
  { "m": "setZoom", "p": [50] }
]
```
Dieses Skript würde zuerst auf ein Element klicken, dann einen Text sprechen und schließlich den Zoom-Level anpassen.

### 2.2. Das Aktionsobjekt

Jedes Aktionsobjekt besteht aus Schlüssel-Wert-Paaren. Der wichtigste Schlüssel ist `"a"` (für Aktion) oder `"m"` (für Makro).

- `{"a": "..."}`: Definiert eine atomare Aktion.
- `{"m": "..."}`: Definiert den Aufruf eines Makros.

Weitere Schlüssel im Objekt dienen als Parameter für die jeweilige Aktion oder das Makro.

### 2.3. Identifizierung von Elementen

Elemente (Items, Relationen, UI-Buttons) können auf verschiedene Weisen adressiert werden:

- **CSS-Selektor (`i`):** Der häufigste Weg, um UI-Elemente zu adressieren.
  - `{"a": "ci", "i": ".id-ok-button"}`
- **Item-Index (`ii`, `if`, `it`):** Um Items oder Relationen im Diagramm direkt über ihren numerischen Index anzusprechen. Das Root-Item hat den Index `0`. Dieser Index wird basierend auf einer **Breitensuche (BFS)**-Durchquerung der Mindmap zugewiesen: zuerst das Root-Item (`ii=0`), dann alle seine Kinder (z.B. `ii=1`, `ii=2` usw., typischerweise in der Erstellungsreihenfolge), dann alle Kinder von `ii=1`, dann die Kinder von `ii=2` usw., Ebene für Ebene. Dies macht `ii` zu einem **vorhersehbaren, absoluten** Index, der stabil bleibt, es sei denn, das Item wird gelöscht oder die Diagrammstruktur grundlegend geändert. Es ist eine sehr zuverlässige Methode, um Items zu referenzieren, wenn die Erstellungsreihenfolge und Struktur bekannt sind.
  - `{"a": "cii", "ii": 0}` (Klickt das Root-Item)
- **Name (`in`):** Adressiert ein Item über seinen angezeigten Namen.
  - `{"a": "pi", "in": "Einleitung"}`
- **Stack (`bi` - Back Index):** Aktionen können Items/Relationen auf einen internen "Stack" legen (`uis`, `urs`). `bi` ist ein **relativer** Index, der vom *Ende* dieses Stacks zählt. `bi=0` bezieht sich immer auf das *zuletzt* auf den Stack gelegte Item. Dies ist nützlich, wenn Elemente dynamisch erstellt und dann direkt wiederverwendet werden sollen.

    **Wichtige Stack-Dynamik:**
    *   Die Aktion `uis` (Update Item Stack) legt das *aktuell selektierte Item* oben auf den Stack, wodurch es zu `bi=0` wird. Folglich **erhöht sich der `bi`-Wert aller anderen Items, die sich bereits auf dem Stack befinden, um eins.**
    *   Makros wie `newItemName` und `newSimpleItems` verwenden intern `uis` für jedes von ihnen erstellte Item. Dies bedeutet, dass diese Makros den Stack verändern, indem sie neue Items auf `bi=0` legen, wodurch sich folglich **die `bi`-Werte zuvor gestapelter Items für jedes hinzugefügte Item um eins erhöhen.**
    *   **Robuste Strategie für die Item-Referenzierung:** Beim Erstellen von Kind-Items für ein bestimmtes Elternteil ist es **im Allgemeinen robuster, `parentII` zu verwenden, wenn der Item-Index (`ii`) des Elternteils bekannt und stabil ist.** Dies vermeidet die Komplexität dynamisch verschiebender `bi`-Werte. Falls `parentBI` erforderlich ist (z.B. wenn `ii` nicht direkt vorhersehbar ist oder für komplexe dynamische Stack-Manipulationen), ist es entscheidend sicherzustellen, dass das beabsichtigte Eltern-Item *direkt vor der Kindererstellung* auf `bi=0` positioniert ist. Dies kann erreicht werden durch:
        1.  Ermittlung des aktuellen `bi` des Elternteils (unter Berücksichtigung aller Items, die seit dem letzten Push des Elternteils auf den Stack gelegt wurden).
        2.  Selektieren des Elternteils mit `{"a": "si", "bi": <aktueller_bi_des_elternteils>}`.
        3.  Direkt danach Ausführen von `{"a": "uis"}` um das selektierte Elternteil an die Spitze des Stacks zu legen, wodurch es zu `bi=0` wird.
        4.  Anschließend den Aufruf von `newItemName` oder `newSimpleItems` mit `parentII=-1` (um `ii` explizit zu ignorieren) und `parentBI=0`.

### 2.4. Mehrfachauswahl

Die Skriptlogik unterstützt die Auswahl mehrerer Items, um Gruppenaktionen durchzuführen. Dies erfordert die Aktivierung eines speziellen Modus.

- **Arbeitsablauf:**
    1.  Aktivieren Sie den Modus mit dem Makro `{"m": "enableMultiSelect"}`.
    2.  Wählen Sie mehrere Items aus, indem Sie ein **Array von Indizes** an den `ii`- oder `bi`-Parameter einer Aktion wie `si` übergeben.
    3.  Führen Sie beliebige Gruppenaktionen durch (z.B. Formatierung).
    4.  Heben Sie die Auswahl mit `{"a": "dsi"}` auf oder deaktivieren Sie den Modus mit `{"m": "disableMultiSelect"}`.
- **Wichtig:** Wenn der Mehrfachauswahlmodus nicht aktiviert ist, wird bei der Auswahl eines neuen Items die Auswahl eines zuvor ausgewählten Items aufgehoben.

## 3. Aktions-Referenz (Schlüssel "a")

Die folgende Tabelle beschreibt die atomaren Aktionen, die über den Schlüssel `"a"` gesteuert werden.

| Aktion (`a`) | Beschreibung | Parameter |
| :--- | :--- | :--- |
| `atp` | **Append Text to Panel**: Fügt HTML-Text zum Textpanel hinzu. | `t`: Der HTML-Text. |
| `blur` | **Blur**: Löst das `blur`-Event auf einem Element aus. | `i`: CSS-Selektor des Elements. |
| `ci` | **Click Item**: Klickt auf ein UI-Element. | `i`: CSS-Selektor. `iifs`: (Optional) Selektor für einen iFrame, in dem das Element liegt. `et`: (Optional) Event-Typ (z.B. "mousedown"). |
| `cii` | **Click Item by Index**: Klickt auf ein Item im Diagramm anhand seines Index. | `ii`: Index (oder Array von Indizes) des Items. |
| `ciis` | **Click Item Selector by Index**: Klickt auf den *Selektor* eines Items anhand seines Index. | `ii`: Index (oder Array von Indizes) des Items. |
| `cri` | **Click Relation by Index**: Klickt auf eine Relation anhand der Indizes von Start- und End-Item. | `if`: Index des Start-Items. `it`: Index des End-Items. |
| `csg` | **Close Side Gallery**: Schließt eine angegebene Side Gallery. | `sg`: CSS-Selektor der Side Gallery. |
| `csi` | **Click Stack Item**: Klickt auf ein Item vom Stack. | `bi`: Index (oder Array von Indizes) vom Ende des Stacks (0 = letztes). |
| `csis` | **Click Stack Item Selector**: Klickt auf den Selektor eines Items vom Stack. | `bi`: Index (oder Array von Indizes) vom Ende des Stacks. |
| `csr` | **Click Stack Relation**: Klickt auf eine Relation vom Stack. | `bi`: Index (oder Array von Indizes) vom Ende des Stacks. |
| `ctvc` | **Click Table View Cell**: Klickt auf eine Zelle in der Tabellenansicht. | `col`: Spaltenindex. `row`: Zeilenindex. |
| `dcii` | **Double Click Item by Index**: Doppelklick auf ein Item anhand seines Index. | `ii`: Index des Items. |
| `dcsi` | **Double Click Stack Item**: Doppelklick auf ein Item vom Stack. | `bi`: Index vom Ende des Stacks. |
| `dib` | **Drag Item By**: Zieht ein Element um ein gegebenes Delta. | `i`: Selektor des Elements. `dx`: Pixel in X-Richtung. `dy`: Pixel in Y-Richtung. |
| `dift` | **Drag Item From To**: Zieht ein Diagramm-Item zu einem anderen. | `iif`/`bif`: Index/Stack-Index des Start-Items. `iit`/`bit`: Index/Stack-Index des Ziel-Items. |
| `dsi` | **Deselect All Items**: Hebt die gesamte aktuelle Auswahl im Diagramm auf. | |
| `...` | *Diverse weitere Drag-Aktionen für spezielle Kontexte (z.B. `difctd`, `diftvtc`)* | Siehe `screenplay.js` für Details. |
| `focus` | **Focus**: Löst das `focus`-Event auf einem Element aus. | `i`: CSS-Selektor des Elements. |
| `htp` | **Hide Text Panel**: Versteckt das Textpanel. | |
| `kd` | **Key Down**: Simuliert das Drücken einer Taste. | `i`: Selektor des Elements, das den Fokus hat. `kc`: Der Key-Code der Taste. |
| `mcte` | **Move Cursor to End**: Bewegt den Cursor in einem Textfeld ans Ende. | `iifs`: Selektor des iFrames, der das Textfeld enthält. |
| `oit` | **Open Item Toolbar**: Öffnet die Toolbar eines Items (durch Klick). | `ii`/`bi`: Index/Stack-Index des Items. |
| `ort` | **Open Relation Toolbar**: Öffnet die Toolbar einer Relation. | `if`, `it` / `bi`: Indizes der Items oder Stack-Index der Relation. |
| `pi` | **Present Item**: Zoomt und zentriert die Ansicht auf ein Item. | `in`: Name des Items. `hi`: (Optional) Hit-Index, falls mehrere Items denselben Namen haben. `spt`: (Optional) `true`, um den Item-Text danach zu sprechen. |
| `si` | **Select Item**: Selektiert ein Item, ohne es zu klicken. | `ii`/`bi`: Index (oder Array von Indizes) des Items. |
| `sit` | **Set Ignore Templates**: Legt fest, ob Templates ignoriert werden sollen. | `it`: Boolean-Wert. |
| `spt` | **Speak Text**: Liest einen Text über die Sprachausgabe vor. | `t`: Der zu sprechende Text. |
| `st` | **Select Text**: Markiert Text in einem Eingabefeld. | `i`: Selektor des Feldes. `iifs`: (Optional) iFrame-Selektor. `t`: Der zu markierende Text. |
| `stp` | **Show Text Panel**: Zeigt das Textpanel mit initialem Inhalt an. | `t`: Der initiale HTML-Text. |
| `tt` | **Type Text**: Gibt Text in ein fokussiertes Feld ein. | `i`, `iifs`: Selektor des Feldes. `t`: Der Text. `pt`: (Optional) `true` zum direkten Einfügen. `tc`: (Optional) `true` um "changed"-Event auszulösen. |
| `ttd` | **Type Text Direct**: Gibt Text ohne vorherigen Fokus ein. | `i`: Selektor des Feldes. `t`: Der Text. `pt`: (Optional) `true` zum Einfügen. |
| `ttkb` | **Type Text with Keyboard**: Simuliert die Texteingabe Tastenanschlag für Tastenanschlag. | `i`: Selektor des Feldes. `t`: Der Text. |
| `tspt` | **Toggle Speak Text**: Schaltet die Sprachausgabe global an oder aus. | |
| `uis` | **Update Item Stack**: Legt das aktuell selektierte Item auf den Stack. | `fcd`: (Optional) `true` um aus dem aktuellen Diagramm zu lesen. |
| `urs` | **Update Relation Stack**: Legt die aktuell selektierte Relation auf den Stack. | |
| `slow`, `normal`, `fast` | Ändern die globale Ausführungsgeschwindigkeit des Skripts. | |

## 4. Makro-Referenz (Schlüssel "m")

Makros sind Abkürzungen für eine Kette von Aktionen. Sie werden über den Schlüssel `"m"` aufgerufen. Die Parameter für ein Makro werden im Schlüssel `"p"` als Array übergeben. Die Reihenfolge der Parameter im Array ist entscheidend, da das Makro die Werte entsprechend seiner internen Definition verwendet.

| Makro (`m`) | Beschreibung | Parameter (`p`) |
| :--- | :--- | :--- |
| `showSidebar` | Zeigt die Haupt-Seitenleiste an. | - |
| `showSideGallery` | Öffnet eine bestimmte Galerie in der Seitenleiste. |1) CSS-Selektor der zu öffnenden Galerie. |
| `closeSideGallery` | Schließt eine bestimmte Galerie. |1) CSS-Selektor der zu schließenden Galerie. |
| `disableMultiSelect` | Deaktiviert den Mehrfachauswahl-Modus und kehrt zum Einzelauswahl-Verhalten zurück. | - |
| `enableMultiSelect` | Aktiviert den Mehrfachauswahl-Modus. Muss aufgerufen werden, bevor versucht wird, mehrere Items auszuwählen. | - |
| `execItemCommand` | Führt einen Befehl im "Item bearbeiten"-Menü aus. |1) Selektor des Befehls (z.B. `.id-new-item`). |
| `execInsertCommand` | Führt einen Befehl im "Einfügen"-Menü aus. |1) Selektor des Befehls (z.B. `.id-insert-textnote`). |
| `setLayoutMode` | Setzt den Layout-Modus des Diagramms. |1) Selektor für den gewünschten Modus (z.B. `.id-diagram-layout-free`). |
| `selectDiagramMode` | Wählt einen Diagramm-Anzeigemodus (z.B. 2D, 3D, Tabelle). |1) Selektor für den Modus (z.B. `.id-diagram-mode-3d`). |
| `setShowLabels` | Stellt ein, wie Labels angezeigt werden (z.B. immer anzeigen, nur bei Hover). |1) Selektor für die Label-Anzeigeoption (z.B. `.id-show-labels-always`). |
| `selectCategory` | Wählt eine Kategorie in einem Dropdown-Menü aus. |1) Name der Kategorie. |
| `showCategoryItems`| Zeigt die Galerie mit Items einer bestimmten Kategorie. |1) Name der Kategorie. |
| `clickItemToolbar` | Klickt einen Button in der Toolbar eines Items. |1) Item-Index des Items.<br>2) Stack-Index des Items (vom Ende gezählt, z.B. `0` für das zuletzt selektierte Item).<br>3) Selektor des Toolbar-Buttons (z.B. `.id-edit-item`). |
| `newSimpleItems` | Erstellt mehrere einfache Items nacheinander. **Hinweis:** Dieses Makro verwendet implizit `uis` für *jedes erstellte Kind*, wodurch diese der Reihe nach auf den Stack gelegt werden und sich folglich der `bi`-Wert anderer Items auf dem Stack erhöht. **Bevorzugen Sie `parentII`, wenn der Item-Index des Elternteils vorhersehbar ist.** Wenn Sie `parentBI` verwenden, stellen Sie sicher, dass es zum Zeitpunkt der Ausführung korrekt auf das beabsichtigte Elternteil verweist, möglicherweise unter Verwendung der `si`- und `uis`-Strategie, wie in Abschnitt 2.3 beschrieben. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) Item-Index des Eltern-Items (verwenden Sie `-1`, wenn Sie über `parentBI` adressieren).<br>3) Stack-Index des Eltern-Items (verwenden Sie `0`, wenn das Elternteil gerade mit `uis` auf den Stack gelegt wurde).<br>4) Array von Item-Arrays, wobei jedes Item-Array `[Name, Beschreibung, Hyperlink, Kategorie]` enthält. Alle vier Parameter für jedes Item-Array müssen angegeben werden, verwenden Sie leere Zeichenketten für ungenutzte Werte. |
| `newItemName` | Erstellt ein neues Item mit nur einem Namen. **Hinweis:** Dieses Makro verwendet implizit `uis` für das neu erstellte Item, wodurch es auf `bi=0` auf den Stack gelegt und sich folglich der `bi`-Wert anderer Items auf dem Stack um eins erhöht. **Bevorzugen Sie `parentII`, wenn der Item-Index des Elternteils vorhersehbar ist.** Wenn Sie `parentBI` verwenden, stellen Sie sicher, dass es zum Zeitpunkt der Ausführung korrekt auf das beabsichtigte Elternteil verweist, möglicherweise unter Verwendung der `si`- und `uis`-Strategie, wie in Abschnitt 2.3 beschrieben. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) Item-Index des Eltern-Items (verwenden Sie `-1`, wenn Sie über `parentBI` adressieren).<br>3) Stack-Index des Eltern-Items (verwenden Sie `0`, wenn das Elternteil gerade mit `uis` auf den Stack gelegt wurde).<br>4) Name des neuen Items. Alle vier Parameter für das Makro müssen angegeben werden, verwenden Sie leere Zeichenketten für ungenutzte Werte im internen `newItem`-Aufruf. |
| `newRelation` | Erstellt eine neue Relation zwischen zwei Items. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) Item-Index des Start-Items.<br>3) Stack-Index des Start-Items.<br>4) Item-Index des Ziel-Items.<br>5) Stack-Index des Ziel-Items. |
| `editRelation` | Bearbeitet die Eigenschaften einer Relation. |1) Item-Index des Start-Items der Relation.<br>2) Item-Index des Ziel-Items der Relation.<br>3) Stack-Index der Relation (vom Ende gezählt).<br>4) Name der Relation.<br>5) (Boolean) Steuerung der Sichtbarkeit der Relation (entspricht der Checkbox 'Als Tabelle anzeigen' im Dialog).<br>6) Name der Rück-Relation (Back-Relation Name).<br>7) (Boolean) Steuerung der Sichtbarkeit der Rück-Relation (entspricht der Checkbox 'Als Tabelle anzeigen' im Dialog).<br>8) Beschreibung der Relation.<br>9) Kategorie der Relation. |
| `setZoom` | Stellt den Zoom-Faktor ein. |1) Wert für die Verschiebung des Zoom-Reglers (relativ, z.B. `50` für eine halbe Verschiebung nach rechts). |
| `setDistance` | Stellt den Abstand der Items ein. |1) Wert für die Verschiebung des Abstands-Reglers (relativ, z.B. `50`). |
| `setRotation` | Stellt die Drehung des Diagramms ein (3D). |1) Wert für die Verschiebung des Rotations-Reglers (relativ, z.B. `50`). |
| `setPerspective` | Stellt die Perspektive ein (3D). |1) Wert für die Verschiebung des Perspektiv-Reglers (relativ, z.B. `50`). |
| `saveView` | Speichert die aktuelle Ansicht als Lesezeichen. |1) Name des Lesezeichens.<br>2) (Boolean) `true`, um das Root-Item zu speichern; `false` sonst.<br>3) (Boolean) `true`, um die Ansicht als initiale Ansicht festzulegen; `false` sonst. |
| `restoreView` | Stellt eine gespeicherte Ansicht wieder her. |1) ID des Lesezeichens. |
| `setItemColor` | Setzt die Farbe eines Items. |1) Item-Index des zu formatierenden Items (verwenden Sie `-1`, wenn Sie über `stackIndex` adressieren).<br>2) Stack-Index des zu formatierenden Items (verwenden Sie `0`, wenn das Item gerade mit `uis` auf den Stack gelegt wurde).<br>3) Index der Farbe in der Farbpalette (0-basiert, z.B. `0` für die erste Farbe). |
| `setItemFontSize` | Setzt die Schriftgröße eines Items. |1) Item-Index des zu formatierenden Items.<br>2) Stack-Index des zu formatierenden Items.<br>3) Selektor für die gewünschte Schriftgröße (z.B. `.id-font-size-small`, `.id-font-size-medium`, `.id-font-size-large`). |
| `setTextNote` | Fügt einem Item eine Textnotiz hinzu oder bearbeitet sie. |1) Item-Index des Items, zu dem die Notiz gehört.<br>2) Stack-Index des Items, zu dem die Notiz gehört.<br>3) Textinhalt der Notiz (HTML- oder Klartext). |

## 5. Steuerungs-Eigenschaften

Diese Eigenschaften können jedem Aktionsobjekt hinzugefügt werden, um die Ausführung zu steuern.

| Eigenschaft | Beschreibung | Beispiel |
| :--- | :--- | :--- |
| `wa` | **Wait After**: Warte eine bestimmte Zeit in Millisekunden *nach* der Ausführung der Aktion. | `{"a": "ci", "i": ".btn", "wa": 500}` |
| `waudr`| **Wait After Until Diagram is Ready**: Warte nach der Aktion, bis das Diagramm fertig gerendert ist. Nützlich bei Aktionen, die das Layout verändern. | `{"m": "setLayoutMode", "p": [".id-layout-2d"], "waudr": true}` |
| `wto` | **Wait Timeout**: Maximale Wartezeit in ms für `waudr`. Standard ist 10000. | `{"a": "ci", "i": ".btn", "waudr": true, "wto": 5000}` |
| `wb` | **Wait Before**: Wenn `mbh` oder `mbv` genutzt wird, definiert dies die maximale Wartezeit in ms, bis die Bedingung erfüllt ist. | `{"a": "ci", "i": ".btn", "mbv": ".dialog", "wb": 3000}` |
| `mbh` | **Must Be Hidden**: Die Aktion wird nur ausgeführt, wenn das angegebene Element *nicht* sichtbar ist. | `{"a": "ci", "i": ".open", "mbh": ".dialog"}` |
| `mbv` | **Must Be Visible**: Die Aktion wird nur ausgeführt, wenn das angegebene Element sichtbar ist. | `{"a": "ci", "i": ".close", "mbv": ".dialog"}` |
| `mbs` | **Must Be Selected**: Die Aktion wird nur ausgeführt, wenn der Button/Item selektiert ist. | `{"a": "ci", "i": ".other-btn", "mbs": ".toggle-btn"}` |
| `mnbs` | **Must Not Be Selected**: Die Aktion wird nur ausgeführt, wenn der Button/Item *nicht* selektiert ist. | `{"a": "ci", "i": ".toggle-btn", "mnbs": ".toggle-btn"}` |
| `eine` | **Execute If Not Empty**: Die Aktion wird nur ausgeführt, wenn der Wert dieses Schlüssels nicht leer ist. Nützlich in Makros, um optionale Aktionen zu steuern. | `{"a": "tt", "t": "Optionaler Text", "eine": "Optionaler Text"}` |

### 5.1 Debugging Hilfen

Für schnelleres Debugging und Entwicklung können Sie die folgenden Aktionen am Anfang Ihres Skripts einfügen:
```json
{ "a": "fast" },
{ "a": "tspt" }
```
- `{"a": "fast"}`: Stellt die globale Skriptausführungsgeschwindigkeit auf schnell ein, wodurch Verzögerungen minimiert werden.
- `{"a": "tspt"}`: Schaltet die Sprachausgabe aus, wodurch das Skript keinen Text spricht.

## 6. Beispiele

### Beispiel 1: Zwei Items erstellen und verbinden

```json
[
  {
    "m": "newItemName",
    "p": [false, -1, 0, "Erstes Item"]
  },
  {
    "m": "newItemName",
    "p": [false, -1, 0, "Zweites Item"]
  },
  {
    "m": "newRelation",
    "p": [false, -1, 1, -1, 0]
  }
]
```
**Erklärung:**
1. Erstellt ein neues Item mit dem Namen "Erstes Item" als Kind des Root-Items (standardmäßig, wenn kein Elternteil mit `parentII` angegeben ist) und legt es implizit auf den Stack (`uis`), wodurch es zu `bi=0` wird.
2. Erstellt ein zweites Item mit dem Namen "Zweites Item" (ebenfalls standardmäßig, wenn kein Elternteil mit `parentII` angegeben ist) und legt es implizit auf den Stack (`uis`), wodurch es zu `bi=0` wird. Der `bi`-Wert des "Ersten Items" erhöht sich dadurch auf `1`.
3. Erstellt eine Relation zwischen dem "Zweiten Item" (`bi`: 0) und dem "Ersten Item" (`bi`: 1) auf dem Stack.

### Beispiel 2: Formatierung ändern

```json
[
  {
    "m": "setItemColor",
    "p": [0, -1, 4]
  },
  {
    "m": "setZoom",
    "p": [80]
  },
  {
    "m": "restoreView",
    "p": ["bookmark-id-123"],
    "waudr": true
  }
]
```
**Erklärung:**
1. Selektiert das Root-Item (`ii`: 0) und setzt seine Farbe auf die 5. Farbe (`eq(4)`) in der Palette. Hinweis: Bei Verwendung von `setItemColor` kann `stackIndex` (Parameter 2) `-1` sein, wenn `itemIndex` (Parameter 1) verwendet wird, oder `0`, wenn sich das Item an der Spitze des Stacks befindet.
2. Zoomt in die Ansicht hinein.
3. Stellt eine zuvor gespeicherte Ansicht (Kameraposition, Zoom etc.) wieder her und wartet, bis das Diagramm neu gezeichnet wurde.

### Beispiel 3: Mehrfachauswahl

```json
[
  { "a": "spt", "t": "Zuerst erstelle ich drei Items." },
  { "m": "newItemName", "p": [false, -1, 0, "Item A"] },
  { "m": "newItemName", "p": [false, -1, 0, "Item B"] },
  { "m": "newItemName", "p": [false, -1, 0, "Item C"] },
  { "a": "spt", "t": "Jetzt aktiviere ich die Mehrfachauswahl." },
  { "m": "enableMultiSelect" },
  { "a": "spt", "t": "Ich wähle alle drei Items über ein Array von Back-Indizes aus." },
  { "a": "si", "bi": [0, 1, 2] },
  { "a": "wa", "wa": 1000 },
  { "a": "spt", "t": "Jetzt könnte ich eine Gruppenaktion durchführen. Vorerst hebe ich die Auswahl einfach wieder auf." },
  { "a": "dsi" },
  { "a": "spt", "t": "Zuletzt deaktiviere ich den Mehrfachauswahl-Modus." },
  { "m": "disableMultiSelect" }
]
```
