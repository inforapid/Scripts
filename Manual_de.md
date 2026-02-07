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
- **Index (`ii`, `if`, `it`):** Um Items oder Relationen im Diagramm direkt über ihren numerischen Index anzusprechen (das Root-Item hat den Index `0`).
  - `{"a": "cii", "ii": 0}` (Klickt das Root-Item)
- **Name (`in`):** Adressiert ein Item über seinen angezeigten Namen.
  - `{"a": "pi", "in": "Einleitung"}`
- **Stack (`bi`):** Aktionen können Items/Relationen auf einen internen "Stack" legen (`uis`, `urs`). Spätere Aktionen können diese Elemente dann über ihren Index auf dem Stack wiederverwenden (`bi`, "back index"). Dies ist nützlich, wenn Elemente dynamisch erstellt und dann direkt weiterverwendet werden sollen.
  - `{"a": "csi", "bi": 0}` (Klickt das zuletzt auf den Stack gelegte Item)

## 3. Aktions-Referenz (Schlüssel "a")

Die folgende Tabelle beschreibt die atomaren Aktionen, die über den Schlüssel `"a"` gesteuert werden.

| Aktion (`a`) | Beschreibung | Parameter |
| :--- | :--- | :--- |
| `atp` | **Append Text to Panel**: Fügt HTML-Text zum Textpanel hinzu. | `t`: Der HTML-Text. |
| `blur` | **Blur**: Löst das `blur`-Event auf einem Element aus. | `i`: CSS-Selektor des Elements. |
| `ci` | **Click Item**: Klickt auf ein UI-Element. | `i`: CSS-Selektor. `iifs`: (Optional) Selektor für einen iFrame, in dem das Element liegt. `et`: (Optional) Event-Typ (z.B. "mousedown"). |
| `cii` | **Click Item by Index**: Klickt auf ein Item im Diagramm anhand seines Index. | `ii`: Index des Items. |
| `ciis` | **Click Item Selector by Index**: Klickt auf den *Selektor* eines Items anhand seines Index. | `ii`: Index des Items. |
| `cri` | **Click Relation by Index**: Klickt auf eine Relation anhand der Indizes von Start- und End-Item. | `if`: Index des Start-Items. `it`: Index des End-Items. |
| `csg` | **Close Side Gallery**: Schließt eine angegebene Side Gallery. | `sg`: CSS-Selektor der Side Gallery. |
| `csi` | **Click Stack Item**: Klickt auf ein Item vom Stack. | `bi`: Index vom Ende des Stacks (0 = letztes). |
| `csis` | **Click Stack Item Selector**: Klickt auf den Selektor eines Items vom Stack. | `bi`: Index vom Ende des Stacks. |
| `csr` | **Click Stack Relation**: Klickt auf eine Relation vom Stack. | `bi`: Index vom Ende des Stacks. |
| `ctvc` | **Click Table View Cell**: Klickt auf eine Zelle in der Tabellenansicht. | `col`: Spaltenindex. `row`: Zeilenindex. |
| `dcii` | **Double Click Item by Index**: Doppelklick auf ein Item anhand seines Index. | `ii`: Index des Items. |
| `dcsi` | **Double Click Stack Item**: Doppelklick auf ein Item vom Stack. | `bi`: Index vom Ende des Stacks. |
| `dib` | **Drag Item By**: Zieht ein Element um ein gegebenes Delta. | `i`: Selektor des Elements. `dx`: Pixel in X-Richtung. `dy`: Pixel in Y-Richtung. |
| `dift` | **Drag Item From To**: Zieht ein Diagramm-Item zu einem anderen. | `iif`/`bif`: Index/Stack-Index des Start-Items. `iit`/`bit`: Index/Stack-Index des Ziel-Items. |
| `...` | *Diverse weitere Drag-Aktionen für spezielle Kontexte (z.B. `difctd`, `diftvtc`)* | Siehe `screenplay.js` für Details. |
| `focus` | **Focus**: Löst das `focus`-Event auf einem Element aus. | `i`: CSS-Selektor des Elements. |
| `htp` | **Hide Text Panel**: Versteckt das Textpanel. | |
| `kd` | **Key Down**: Simuliert das Drücken einer Taste. | `i`: Selektor des Elements, das den Fokus hat. `kc`: Der Key-Code der Taste. |
| `mcte` | **Move Cursor to End**: Bewegt den Cursor in einem Textfeld ans Ende. | `iifs`: Selektor des iFrames, der das Textfeld enthält. |
| `oit` | **Open Item Toolbar**: Öffnet die Toolbar eines Items (durch Klick). | `ii`/`bi`: Index/Stack-Index des Items. |
| `ort` | **Open Relation Toolbar**: Öffnet die Toolbar einer Relation. | `if`, `it` / `bi`: Indizes der Items oder Stack-Index der Relation. |
| `pi` | **Present Item**: Zoomt und zentriert die Ansicht auf ein Item. | `in`: Name des Items. `hi`: (Optional) Hit-Index, falls mehrere Items denselben Namen haben. `spt`: (Optional) `true`, um den Item-Text danach zu sprechen. |
| `si` | **Select Item**: Selektiert ein Item, ohne es zu klicken. | `ii`/`bi`: Index/Stack-Index des Items. |
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
| `execItemCommand` | Führt einen Befehl im "Item bearbeiten"-Menü aus. |1) Selektor des Befehls (z.B. `.id-new-item`). |
| `execInsertCommand` | Führt einen Befehl im "Einfügen"-Menü aus. |1) Selektor des Befehls (z.B. `.id-insert-textnote`). |
| `setLayoutMode` | Setzt den Layout-Modus des Diagramms. |1) Selektor für den gewünschten Modus (z.B. `.id-diagram-layout-free`). |
| `selectDiagramMode` | Wählt einen Diagramm-Anzeigemodus (z.B. 2D, 3D, Tabelle). |1) Selektor für den Modus (z.B. `.id-diagram-mode-3d`). |
| `setShowLabels` | Stellt ein, wie Labels angezeigt werden (z.B. immer anzeigen, nur bei Hover). |1) Selektor für die Label-Anzeigeoption (z.B. `.id-show-labels-always`). |
| `selectCategory` | Wählt eine Kategorie in einem Dropdown-Menü aus. |1) Name der Kategorie. |
| `showCategoryItems`| Zeigt die Galerie mit Items einer bestimmten Kategorie. |1) Name der Kategorie. |
| `clickItemToolbar` | Klickt einen Button in der Toolbar eines Items. |1) Item-Index des Items.<br>2) Stack-Index des Items (vom Ende gezählt, z.B. `0` für das zuletzt selektierte Item).<br>3) Selektor des Toolbar-Buttons (z.B. `.id-edit-item`). |
| `newSimpleItems` | Erstellt mehrere einfache Items nacheinander. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) Item-Index des Eltern-Items.<br>3) Stack-Index des Eltern-Items.<br>4) Array von Item-Arrays, wobei jedes Item-Array `[Name, Beschreibung, Hyperlink, Kategorie]` enthält. |
| `newItemName` | Erstellt ein neues Item mit nur einem Namen. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) Item-Index des Eltern-Items.<br>3) Stack-Index des Eltern-Items.<br>4) Name des neuen Items. |
| `newRelation` | Erstellt eine neue Relation zwischen zwei Items. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) Item-Index des Start-Items.<br>3) Stack-Index des Start-Items.<br>4) Item-Index des Ziel-Items.<br>5) Stack-Index des Ziel-Items. |
| `editRelation` | Bearbeitet die Eigenschaften einer Relation. |1) Item-Index des Start-Items der Relation.<br>2) Item-Index des Ziel-Items der Relation.<br>3) Stack-Index der Relation (vom Ende gezählt).<br>4) Name der Relation.<br>5) (Boolean) Steuerung der Sichtbarkeit der Relation (entspricht der Checkbox 'Als Tabelle anzeigen' im Dialog).<br>6) Name der Rück-Relation (Back-Relation Name).<br>7) (Boolean) Steuerung der Sichtbarkeit der Rück-Relation (entspricht der Checkbox 'Als Tabelle anzeigen' im Dialog).<br>8) Beschreibung der Relation.<br>9) Kategorie der Relation. |
| `setZoom` | Stellt den Zoom-Faktor ein. |1) Wert für die Verschiebung des Zoom-Reglers (relativ, z.B. `50` für eine halbe Verschiebung nach rechts). |
| `setDistance` | Stellt den Abstand der Items ein. |1) Wert für die Verschiebung des Abstands-Reglers (relativ, z.B. `50`). |
| `setRotation` | Stellt die Drehung des Diagramms ein (3D). |1) Wert für die Verschiebung des Rotations-Reglers (relativ, z.B. `50`). |
| `setPerspective` | Stellt die Perspektive ein (3D). |1) Wert für die Verschiebung des Perspektiv-Reglers (relativ, z.B. `50`). |
| `saveView` | Speichert die aktuelle Ansicht als Lesezeichen. |1) Name des Lesezeichens.<br>2) (Boolean) `true`, um das Root-Item zu speichern; `false` sonst.<br>3) (Boolean) `true`, um die Ansicht als initiale Ansicht festzulegen; `false` sonst. |
| `restoreView` | Stellt eine gespeicherte Ansicht wieder her. |1) ID des Lesezeichens. |
| `setItemColor` | Setzt die Farbe eines Items. |1) Item-Index des zu formatierenden Items.<br>2) Stack-Index des zu formatierenden Items.<br>3) Index der Farbe in der Farbpalette (0-basiert, z.B. `0` für die erste Farbe). |
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

## 6. Beispiele

### Beispiel 1: Zwei Items erstellen und verbinden

```json
[
  {
    "m": "newItemName",
    "p": ["-1", "0", ".id-new-item", "Erstes Item"]
  },
  {
    "a": "uis"
  },
  {
    "m": "newItemName",
    "p": ["-1", "0", ".id-new-item", "Zweites Item"]
  },
  {
    "a": "uis"
  },
  {
    "m": "newRelation",
    "p": [false, -1, 1, -1, 0]
  }
]
```
**Erklärung:**
1. Erstellt ein neues Item mit dem Namen "Erstes Item" als Kind des Root-Items (`-1` ist der Index für das Root-Item in diesem Kontext, `0` ist der Stack-Index).
2. Legt das neu erstellte Item auf den Stack (`uis`).
3. Erstellt ein zweites Item mit dem Namen "Zweites Item".
4. Legt auch dieses auf den Stack.
5. Erstellt eine Relation zwischen dem zweiten (`bi`: 0) und dem ersten (`bi`: 1) Item auf dem Stack.

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
1. Setzt die Farbe des Root-Items (`ii`: 0) auf die 5. Farbe (`eq(4)`) in der Palette.
2. Zoomt in die Ansicht hinein.
3. Stellt eine zuvor gespeicherte Ansicht (Kamerposition, Zoom etc.) wieder her und wartet, bis das Diagramm neu gezeichnet wurde.
