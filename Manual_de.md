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
- **Name (`in`, `inf`, `int`):** Adressiert ein Item über seinen angezeigten Namen (`in`) oder eine Relation über die angezeigten Namen ihrer Start- (`inf`) und End- (`int`) Items.
  - `{"a": "pi", "in": "Einleitung"}`
  - `{"a": "ort", "inf": "Start-Item-Name", "int": "End-Item-Name"}`
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
| `dift` | **Drag Item From To**: Zieht ein Diagramm-Item zu einem anderen Diagramm-Item. | `iif`/`bif`: Index/Stack-Index des zu ziehenden Items (Quelle). `iit`/`bit`: Index/Stack-Index des Ziel-Items (wohin gezogen werden soll). |
| `drft` | **Drag Relation End Point From To**: Zieht einen Endpunkt einer existierenden Relation von einem Quell-Item zu einem Ziel-Item, um die Relation neu zu verknüpfen. | `iif`/`bif`: Index/Stack-Index des Quell-Items, von dem der Relationsendpunkt gezogen wird.<br>`inf`: Name des Quell-Items, von dem der Relationsendpunkt gezogen wird.<br>`iit`/`bit`: Index/Stack-Index des Ziel-Items, zu dem der Relationsendpunkt gezogen wird.<br>`int`: Name des Ziel-Items, zu dem der Relationsendpunkt gezogen wird. |
| `difctd` | **Drag Item From Category To Diagram**: Zieht ein Item aus einer Kategorie (z.B. in der Seitenleiste) auf ein bestimmtes Item im Diagramm. | `if`: CSS-Selektor des Quell-Items in der Kategorie. `iit`/`bit`: Index/Stack-Index des Ziel-Items im Diagramm. |
| `difdtc` | **Drag Item From Diagram To Category**: Zieht ein Item aus dem Diagramm in einen Kategoriebereich (z.B. die Seitenleiste), um es zu entfernen oder zu verschieben. | `iif`/`bif`: Index/Stack-Index des zu ziehenden Items im Diagramm. `it`: CSS-Selektor des Ziel-Kategoriebereichs. |
| `difcttv` | **Drag Item From Category To Table View**: Zieht ein Item aus einer Kategorie (z.B. in der Seitenleiste) auf eine bestimmte Zelle in der Tabellenansicht. | `if`: CSS-Selektor des Quell-Items in der Kategorie. `colt`: Spaltenindex der Zielzelle (0-basiert). `rowt`: Zeilenindex der Zielzelle (0-basiert). |
| `diftvtc` | **Drag Item From Table View To Category**: Zieht ein Item von einer Zelle in der Tabellenansicht in einen Kategoriebereich. | `colf`: Spaltenindex der Quellzelle (0-basiert). `rowf`: Zeilenindex der Quellzelle (0-basiert). `it`: CSS-Selektor des Ziel-Kategoriebereichs. |
| `dtvcft` | **Drag Table View Cell From To**: Zieht ein Item von einer Zelle in der Tabellenansicht zu einer anderen Zelle in der Tabellenansicht. | `colf`: Spaltenindex der Quellzelle (0-basiert). `rowf`: Zeilenindex der Quellzelle (0-basiert). `colt`: Spaltenindex der Zielzelle (0-basiert). `rowt`: Zeilenindex der Zielzelle (0-basiert). |
| `dsi` | **Deselect All Items**: Hebt die gesamte aktuelle Auswahl im Diagramm auf. | |
| `focus` | **Focus**: Löst das `focus`-Event auf einem Element aus. | `i`: CSS-Selektor des Elements. |
| `htp` | **Hide Text Panel**: Versteckt das Textpanel. | |
| `kd` | **Key Down**: Simuliert das Drücken einer Taste. | `i`: Selektor des Elements, das den Fokus hat. `kc`: Der Key-Code der Taste. |
| `mcte` | **Move Cursor to End**: Bewegt den Cursor in einem Textfeld ans Ende. | `iifs`: Selektor des iFrames, der das Textfeld enthält. |
| `oit` | **Open Item Toolbar**: Öffnet die Toolbar eines Items (durch Klick). | `ii`/`bi`: Index/Stack-Index des Items. |
| `ort` | **Open Relation Toolbar**: Öffnet die Toolbar einer Relation. | `if`, `it` / `bi`: Indizes der Items oder Stack-Index der Relation.<br>`inf`, `int`: Namen der Start- und End-Items der Relation. |
| `pi` | **Present Item**: Zoomt und zentriert die Ansicht auf ein Item. | `in`: Name des Items. `hi`: (Optional) Hit-Index, falls mehrere Items denselben Namen haben. `spt`: (Optional) `true`, um den Item-Text danach zu sprechen. |
| `si` | **Select Item**: Selektiert ein Item, ohne es zu klicken. | `ii`/`bi`: Index (oder Array von Indizes) des Items. |
| `siiv` | **Scroll Item Into View**: Scrollt das Diagramm so, dass ein bestimmtes Item sichtbar und, wenn möglich, zentriert wird. | `ii`: Index des Items. `bi`: Stack-Index des Items. |
| `sit` | **Set Ignore Templates**: Legt fest, ob Templates ignoriert werden sollen. | `it`: Boolean-Wert. |
| `sna` | **Skip Next Action**: Überspringt die nächste Aktion, falls `true`. | `skip`: (Boolean) `true` zum Überspringen der nächsten Aktion. |
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
| `showSidebar` | **Show Sidebar**: Zeigt die Haupt-Seitenleiste an. Klickt auf den Button zum Anzeigen der Seitenleiste, falls diese noch nicht sichtbar ist. | - |
| `showSideGallery` | **Show Side Gallery**: Zeigt die Haupt-Seitenleiste an und öffnet dann eine spezifische Seiten-Galerie. |1) `gallerySelector`: CSS-Selektor der zu öffnenden Galerie (z.B. `.id-viewmode`). |
| `pinSideGallery` | **Pin Side Gallery**: Öffnet eine spezifische Seiten-Galerie und pinnt sie fest, sodass sie geöffnet bleibt. |1) `gallerySelector`: CSS-Selektor der zu pinnenden Galerie (z.B. `.id-editmode`). |
| `closeSideGallery` | **Close Side Gallery**: Schließt eine spezifische Seiten-Galerie. |1) `gallerySelector`: CSS-Selektor der zu schließenden Galerie (z.B. `.id-viewmode`). |
| `disableMultiSelect` | **Disable Multi-Select**: Deaktiviert den Mehrfachauswahl-Modus im Diagramm und kehrt zum Einzelauswahl-Verhalten zurück. | - |
| `enableMultiSelect` | **Enable Multi-Select**: Aktiviert den Mehrfachauswahl-Modus im Diagramm. Dies muss aufgerufen werden, bevor versucht wird, mehrere Items auszuwählen. | - |
| `execItemCommand` | **Execute Item Command**: Öffnet die "Bearbeiten"-Seiten-Galerie, falls noch nicht geschehen, und führt dann einen spezifischen Befehl für ein Item aus. |1) `commandSelector`: CSS-Selektor des auszuführenden Befehls (z.B. `.id-new-item`, `.id-delete-item`). |
| `execInsertCommand` | **Execute Insert Command**: Öffnet die "Bearbeiten"-Seiten-Galerie, falls noch nicht geschehen, und führt dann einen spezifischen "Einfügen"-Befehl aus. |1) `commandSelector`: CSS-Selektor des auszuführenden Befehls (z.B. `.id-insert-textnote`). |
| `execUndoCommand` | **Execute Undo Command**: Öffnet die "Bearbeiten"-Seiten-Galerie, wählt einen spezifischen Rückgängig-Befehl aus der Liste der letzten Aktionen aus und schließt dann die Galerie. |1) `undoItemId`: Die ID des spezifischen Rückgängig-Eintrags (z.B. `1` für die letzte Aktion). |
| `setLayoutMode` | **Set Layout Mode**: Öffnet die "Ansichts"-Seiten-Galerie und setzt den Layout-Modus des Diagramms (z.B. hierarchisch, kreisförmig). |1) `layoutSelector`: CSS-Selektor für den gewünschten Layout-Modus (z.B. `.id-diagram-layout-free`, `.id-diagram-layout-tree`). |
| `selectDiagramMode` | **Select Diagram Mode**: Öffnet die "Ansichts"-Seiten-Galerie und wählt einen Diagramm-Anzeigemodus (z.B. 2D, 3D, Tabelle) aus. |1) `modeSelector`: CSS-Selektor für den gewünschten Modus (z.B. `.id-diagram-mode-3d`, `.id-diagram-mode-table`). |
| `setShowLabels` | **Set Show Labels**: Öffnet die "Ansichts"-Seiten-Galerie und stellt ein, wie Labels im Diagramm angezeigt werden (z.B. immer, bei Hover, nie). |1) `labelOptionSelector`: CSS-Selektor für die gewünschte Label-Anzeigeoption (z.B. `.id-show-labels-always`). |
| `selectCategory` | **Select Category**: Wählt eine spezifische Kategorie in einem Dropdown-Menü aus. |1) `categoryName`: Der Name der auszuwählenden Kategorie. |
| `showCategoryItems` | **Show Category Items**: Zeigt die Galerie mit Items einer bestimmten Kategorie an. |1) `categoryName`: Der Name der Kategorie, deren Items angezeigt werden sollen. |
| `clickItemToolbar` | **Click Item Toolbar**: Öffnet die Toolbar eines Items und klickt dann einen Button in dieser Toolbar. |1) `itemIndex`: Item-Index des Items (verwenden Sie `-1`, wenn über `stackIndex` adressiert wird).<br>2) `stackIndex`: Stack-Index des Items (0 für das zuletzt selektierte Item, `-1` wenn über `itemIndex` adressiert wird).<br>3) `buttonSelector`: CSS-Selektor des Toolbar-Buttons (z.B. `.id-edit-item`). |
| `clickRelationToolbar` | **Click Relation Toolbar**: Öffnet die Toolbar einer Relation und klickt dann einen Button in dieser Toolbar. |1) `fromItemIndex`: Item-Index des Start-Items der Relation (verwenden Sie `-1`, wenn über `stackIndex` adressiert wird).<br>2) `toItemIndex`: Item-Index des Ziel-Items der Relation (verwenden Sie `-1`, wenn über `stackIndex` adressiert wird).<br>3) `stackIndex`: Stack-Index der Relation (0 für die zuletzt selektierte Relation, `-1` wenn über Item-Indizes adressiert wird).<br>4) `buttonSelector`: CSS-Selektor des Toolbar-Buttons (z.B. `.id-edit-relation`). |
| `clickQuickAccessToolbar` | **Click Quick Access Toolbar**: Klickt auf einen Button in der Schnellzugriffsleiste, falls dieser noch nicht selektiert ist, und wartet, bis das Diagramm aktualisiert ist. |1) `buttonSelector`: CSS-Selektor des Buttons in der Schnellzugriffsleiste. |
| `clickQuickAccessToolbarDirect` | **Click Quick Access Toolbar Direct**: Klickt direkt auf einen Button in der Schnellzugriffsleiste, falls dieser noch nicht selektiert ist. |1) `buttonSelector`: CSS-Selektor des Buttons in der Schnellzugriffsleiste. |
| `importWikipediaArticle` | **Import Wikipedia Article**: Importiert einen Wikipedia-Artikel für ein Item. Öffnet den Wikipedia-Importdialog über die Item-Toolbar, bestätigt den Import und aktualisiert den Item-Stack mit dem neuen Item. |1) `itemStackIndex`: Stack-Index des Items, zu dem der Artikel importiert werden soll. |
| `pinEditItemDialog` | **Pin Edit Item Dialog**: Pinnt den "Item bearbeiten"-Dialog fest, sodass er geöffnet bleibt, falls er noch nicht angepinnt ist. | - |
| `openNewItemDialog` | **Open New Item Dialog**: Öffnet den Dialog zum Erstellen eines neuen Items. Setzt, ob Templates ignoriert werden sollen, und klickt dann den "Neues Item"-Button. |1) `ignoreTemplates`: (Boolean) `true`, um Templates zu ignorieren; `false` sonst.<br>2) `parentItemIndex`: Item-Index des Eltern-Items, unter dem das neue Item erstellt werden soll (verwenden Sie `-1`, wenn `parentStackIndex` verwendet wird).<br>3) `parentStackIndex`: Stack-Index des Eltern-Items (verwenden Sie `0`, wenn das Elternteil gerade mit `uis` auf den Stack gelegt wurde; `-1` wenn `parentItemIndex` verwendet wird). |
| `createNewItem` | **Create New Item**: Füllt die Felder des "Item bearbeiten"-Dialogs aus (Name, Beschreibung, Hyperlink, Kategorie) und unterstützt optional das Einfügen von Tabellen in die Textfelder. |1) `itemName`: Der Name des Items.<br>2)-13) `nameTableParams`: Parameter für die `insertTable`-Makro im Namensfeld (Reihen, Spalten, Zellinhalte).<br>14) `itemDescription`: Die Beschreibung des Items.<br>15)-26) `descriptionTableParams`: Parameter für die `insertTable`-Makro im Beschreibungsfeld (Reihen, Spalten, Zellinhalte).<br>27) `itemHyperlink`: Der Hyperlink des Items.<br>28) `itemCategory`: Die Kategorie des Items. |
| `createNewItemAndContinue` | **Create New Item And Continue**: Erstellt ein neues Item unter Verwendung der `createNewItem`-Makro und schließt den Dialog anschließend mit "OK". |1)-28) `itemParams`: Alle Parameter, die an die `createNewItem`-Makro übergeben werden. |
| `createNewSimpleItem` | **Create New Simple Item**: Erstellt ein neues Item mit nur den grundlegenden Informationen (Name, Beschreibung, Hyperlink, Kategorie). Eine vereinfachte Version von `createNewItem`. |1) `itemName`: Der Name des Items.<br>2) `itemDescription`: Die Beschreibung des Items.<br>3) `itemHyperlink`: Der Hyperlink des Items.<br>4) `itemCategory`: Die Kategorie des Items. |
| `createNewSimpleItemAndContinue` | **Create New Simple Item And Continue**: Erstellt ein einfaches Item unter Verwendung von `createNewSimpleItem` und schließt den Dialog anschließend mit "OK". |1)-4) `simpleItemParams`: Alle Parameter, die an die `createNewSimpleItem`-Makro übergeben werden. |
| `newItemDialogCancel` | **New Item Dialog Cancel**: Klickt auf den "Abbrechen"-Button im "Item bearbeiten"-Dialog. | - |
| `newItemDialogOk` | **New Item Dialog OK**: Klickt auf den "OK"-Button im "Item bearbeiten"-Dialog und legt das neu erstellte Item auf den Stack. | - |
| `newItem` | **New Item**: Öffnet den "Neues Item"-Dialog, füllt ihn aus und bestätigt mit "OK". |1) `ignoreTemplates`: (Boolean) `true`, um Templates zu ignorieren; `false` sonst.<br>2) `parentItemIndex`: Item-Index des Eltern-Items.<br>3) `parentStackIndex`: Stack-Index des Eltern-Items.<br>4)-31) `itemCreationParams`: Alle weiteren Parameter, die für die Item-Erstellung benötigt werden (Name, Beschreibung, etc.). |
| `newSimpleItems` | **New Simple Items**: Erstellt mehrere einfache Items nacheinander. **Hinweis:** Dieses Makro verwendet implizit `uis` für *jedes erstellte Kind*, wodurch diese der Reihe nach auf den Stack gelegt werden und sich folglich der `bi`-Wert anderer Items auf dem Stack erhöht. **Bevorzugen Sie `parentII`, wenn der Item-Index des Elternteils vorhersehbar ist.** Wenn Sie `parentBI` verwenden, stellen Sie sicher, dass es zum Zeitpunkt der Ausführung korrekt auf das beabsichtigte Elternteil verweist, möglicherweise unter Verwendung der `si`- und `uis`-Strategie, wie in Abschnitt 2.3 beschrieben. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) Item-Index des Eltern-Items (verwenden Sie `-1`, wenn Sie über `parentBI` adressieren).<br>3) Stack-Index des Eltern-Items (verwenden Sie `0`, wenn das Elternteil gerade mit `uis` auf den Stack gelegt wurde).<br>4) Array von Item-Arrays, wobei jedes Item-Array `[Name, Beschreibung, Hyperlink, Kategorie]` enthält. Alle vier Parameter für jedes Item-Array müssen angegeben werden, verwenden Sie leere Zeichenketten für ungenutzte Werte. |
| `newItemName` | **New Item Name**: Erstellt ein neues Item mit nur einem Namen. **Hinweis:** Dieses Makro verwendet implizit `uis` für das neu erstellte Item, wodurch es auf `bi=0` auf den Stack gelegt und sich folglich der `bi`-Wert anderer Items auf dem Stack um eins erhöht. **Bevorzugen Sie `parentII`, wenn der Item-Index des Elternteils vorhersehbar ist.** Wenn Sie `parentBI` verwenden, stellen Sie sicher, dass es zum Zeitpunkt der Ausführung korrekt auf das beabsichtigte Elternteil verweist, möglicherweise unter Verwendung der `si`- und `uis`-Strategie, wie in Abschnitt 2.3 beschrieben. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) Item-Index des Eltern-Items (verwenden Sie `-1`, wenn Sie über `parentBI` adressieren).<br>3) Stack-Index des Eltern-Items (verwenden Sie `0`, wenn das Elternteil gerade mit `uis` auf den Stack gelegt wurde).<br>4) Name des neuen Items. Alle vier Parameter für das Makro müssen angegeben werden, verwenden Sie leere Zeichenketten für ungenutzte Werte im internen `newItem`-Aufruf. |
| `endNewRelation` | **End New Relation**: Schließt den Dialog zum Erstellen einer Relation und entfernt die Relation vom Stack. | - |
| `newRelation` | **New Relation**: Erstellt eine neue Relation zwischen zwei Diagramm-Items. Dieses Makro simuliert das Ziehen einer Verbindungslinie von einem Start- zu einem Ziel-Item. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) `fromItemIndex`: Item-Index des Start-Items.<br>3) `fromStackIndex`: Stack-Index des Start-Items.<br>4) `toItemIndex`: Item-Index des Ziel-Items.<br>5) `toStackIndex`: Stack-Index des Ziel-Items. |
| `newRelationFromCategoryItemToDiagram` | **New Relation From Category Item To Diagram**: Erstellt eine neue Relation, indem ein Item aus einer Kategorie (z.B. Seitenleiste) zu einem Item im Diagramm gezogen wird. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) `categoryName`: Name der Kategorie, aus der das Quell-Item stammt.<br>3) `categoryItemName`: Name des Quell-Items in der Kategorie.<br>4) `diagramItemIndex`: Item-Index des Ziel-Items im Diagramm.<br>5) `diagramStackIndex`: Stack-Index des Ziel-Items im Diagramm. |
| `newRelationFromDiagramItemToCategory` | **New Relation From Diagram Item To Category**: Erstellt eine neue Relation, indem ein Item aus dem Diagramm zu einem Item in einer Kategorie (z.B. Seitenleiste) gezogen wird. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) `categoryName`: Name der Kategorie, zu der das Ziel-Item gehört.<br>3) `diagramItemIndex`: Item-Index des Quell-Items im Diagramm.<br>4) `diagramStackIndex`: Stack-Index des Quell-Items im Diagramm.<br>5) `categoryItemName`: Name des Ziel-Items in der Kategorie. |
| `newRelationFromCategoryItemToTableView` | **New Relation From Category Item To Table View**: Erstellt eine neue Relation, indem ein Item aus einer Kategorie (z.B. Seitenleiste) zu einer Zelle in der Tabellenansicht gezogen wird. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) `categoryName`: Name der Kategorie, aus der das Quell-Item stammt.<br>3) `categoryItemName`: Name des Quell-Items in der Kategorie.<br>4) `targetColumnIndex`: Spaltenindex der Zielzelle (0-basiert).<br>5) `targetRowIndex`: Zeilenindex der Zielzelle (0-basiert). |
| `newRelationFromTableViewItemToCategory` | **New Relation From Table View Item To Category**: Erstellt eine neue Relation, indem ein Item aus einer Zelle in der Tabellenansicht zu einem Item in einer Kategorie (z.B. Seitenleiste) gezogen wird. |1) (Boolean) `ignoreTemplates` – legt fest, ob Templates ignoriert werden sollen.<br>2) `categoryName`: Name der Kategorie, zu der das Ziel-Item gehört.<br>3) `sourceColumnIndex`: Spaltenindex der Quellzelle (0-basiert).<br>4) `sourceRowIndex`: Zeilenindex der Quellzelle (0-basiert).<br>5) `categoryItemName`: Name des Ziel-Items in der Kategorie. |
| `editRelation` | **Edit Relation**: Bearbeitet die Eigenschaften einer Relation über den Bearbeitungsdialog. |1) `fromItemIndex`: Item-Index des Start-Items der Relation.<br>2) `toItemIndex`: Item-Index des Ziel-Items der Relation.<br>3) `stackIndex`: Stack-Index der Relation (vom Ende gezählt).<br>4) `relationName`: Name der Relation.<br>5) `showAsTable`: (Boolean) Steuerung der Sichtbarkeit der Relation (entspricht der Checkbox 'Als Tabelle anzeigen' im Dialog).<br>6) `backRelationName`: Name der Rück-Relation.<br>7) `backShowAsTable`: (Boolean) Steuerung der Sichtbarkeit der Rück-Relation.<br>8) `description`: Beschreibung der Relation.<br>9) `category`: Kategorie der Relation. |
| `editRelationEndPoint` | **Edit Relation End Point**: Ändert den Endpunkt einer bestehenden Relation. Die ersten drei Parameter (`fromItemIndex`, `toItemIndex`, `relationStackIndex`) dienen der Auswahl der zu bearbeitenden Verbindungslinie. Um anschließend den Start- oder Endpunkt der Verbindungslinie zu verschieben, werden die folgenden vier Parameter (`newFromItemIndex`, `newFromStackIndex`, `newToItemIndex`, `newToStackIndex`) genutzt. Man befüllt entweder die Parameter für den neuen Startpunkt (`newFromItemIndex`, `newFromStackIndex`) und setzt die Parameter für den Endpunkt (`newToItemIndex`, `newToStackIndex`) auf `-1`, oder umgekehrt. Der jeweilige Index des neuen Start- oder Endpunktes muss angegeben werden. |1) `fromItemIndex`: Item-Index des Start-Items der zu bearbeitenden Relation.<br>2) `toItemIndex`: Item-Index des Ziel-Items der zu bearbeitenden Relation.<br>3) `relationStackIndex`: Stack-Index der zu bearbeitenden Relation.<br>4) `newFromItemIndex`: Item-Index des *neuen* Start-Items für die Relation (verwenden Sie `-1`, falls der Endpunkt verschoben wird).<br>5) `newFromStackIndex`: Stack-Index des *neuen* Start-Items für die Relation (verwenden Sie `-1`, falls der Endpunkt verschoben wird).<br>6) `newToItemIndex`: Item-Index des *neuen* Ziel-Items für die Relation (verwenden Sie `-1`, falls der Startpunkt verschoben wird).<br>7) `newToStackIndex`: Stack-Index des *neuen* Ziel-Items für die Relation (verwenden Sie `-1`, falls der Startpunkt verschoben wird). |
| `editRelationEndPointByItemName` | **Edit Relation End Point By Item Name**: Ändert den Endpunkt einer bestehenden Relation, indem die Relation und der neue Endpunkt über die Namen von Start- und Ziel-Items identifiziert werden. |1) `fromItemName`: Name des Start-Items der zu bearbeitenden Relation.<br>2) `toItemName`: Name des Ziel-Items der zu bearbeitenden Relation.<br>3) `newFromItemName`: Name des *neuen* Start-Items für die Relation (verwenden Sie einen leeren String, falls der Endpunkt verschoben wird).<br>4) `newToItemName`: Name des *neuen* Ziel-Items für die Relation (verwenden Sie einen leeren String, falls der Startpunkt verschoben wird). |
| `clickButton` | **Click Button**: Klickt auf einen spezifischen Button. |1) `buttonSelector`: CSS-Selektor des zu klickenden Buttons. |
| `dragViewSliderBy` | **Drag View Slider By**: Zieht einen View-Slider (z.B. Zoom, Abstand, Rotation) um einen bestimmten Wert. |1) `sliderSelector`: CSS-Selektor des Sliders (z.B. `.id-viewmode-zoom`).<br>2) `deltaX`: Pixel, um die der Slider verschoben werden soll (positiv für rechts, negativ für links).<br>3) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `resetViewSlider` | **Reset View Slider**: Setzt einen View-Slider auf seinen Standardwert zurück. |1) `sliderSelector`: CSS-Selektor des Sliders (z.B. `.id-viewmode-zoom`).<br>2) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `setZoom` | **Set Zoom**: Stellt den Zoom-Faktor des Diagramms ein. |1) `zoomValue`: Wert für die Verschiebung des Zoom-Reglers (relativ, z.B. `50` für eine halbe Verschiebung nach rechts).<br>2) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `resetZoom` | **Reset Zoom**: Setzt den Zoom-Faktor des Diagramms auf den Standardwert zurück. |1) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `setDistance` | **Set Distance**: Stellt den Abstand der Items im Diagramm ein. |1) `distanceValue`: Wert für die Verschiebung des Abstands-Reglers (relativ, z.B. `50`).<br>2) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `resetDistance` | **Reset Distance**: Setzt den Abstand der Items im Diagramm auf den Standardwert zurück. |1) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `setRotation` | **Set Rotation**: Stellt die Drehung des Diagramms (im 3D-Modus) ein. |1) `rotationValue`: Wert für die Verschiebung des Rotations-Reglers (relativ, z.B. `50`).<br>2) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `resetRotation` | **Reset Rotation**: Setzt die Drehung des Diagramms (im 3D-Modus) auf den Standardwert zurück. |1) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `setPerspective` | **Set Perspective**: Stellt die Perspektive des Diagramms (im 3D-Modus) ein. |1) `perspectiveValue`: Wert für die Verschiebung des Perspektiv-Reglers (relativ, z.B. `50`).<br>2) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `resetPerspective` | **Reset Perspective**: Setzt die Perspektive des Diagramms (im 3D-Modus) auf den Standardwert zurück. |1) `skipCloseSideGallery`: (Optional, Boolean) `true`, um das Schließen der Side Gallery zu überspringen. |
| `saveView` | **Save View**: Speichert die aktuelle Ansicht als Lesezeichen. |1) `bookmarkName`: Name des Lesezeichens.<br>2) `saveRootItem`: (Boolean) `true`, um das Root-Item zu speichern; `false` sonst.<br>3) `setAsInitialView`: (Boolean) `true`, um die Ansicht als initiale Ansicht festzulegen; `false` sonst. |
| `restoreView` | **Restore View**: Stellt eine gespeicherte Ansicht anhand ihrer ID wieder her. |1) `bookmarkId`: ID des Lesezeichens. |
| `itemCategoryFilter` | **Item Category Filter**: Filtert die angezeigten Items im Diagramm nach Kategorie. |1) `categoryName`: Name der Kategorie, nach der gefiltert werden soll. |
| `relationCategoryFilter` | **Relation Category Filter**: Filtert die angezeigten Relationen im Diagramm nach Kategorie. |1) `categoryName`: Name der Kategorie, nach der gefiltert werden soll. |
| `setItemColor` | **Set Item Color**: Setzt die Farbe eines Items. |1) `itemIndex`: Item-Index des zu formatierenden Items (verwenden Sie `-1`, wenn Sie über `stackIndex` adressieren).<br>2) `stackIndex`: Stack-Index des zu formatierenden Items (verwenden Sie `0`, wenn das Item gerade mit `uis` auf den Stack gelegt wurde).<br>3) `colorIndex`: Index der Farbe in der Farbpalette (0-basiert, z.B. `0` für die erste Farbe). |
| `setItemFontSize` | **Set Item Font Size**: Setzt die Schriftgröße eines Items. |1) `itemIndex`: Item-Index des zu formatierenden Items.<br>2) `stackIndex`: Stack-Index des zu formatierenden Items.<br>3) `fontSizeSelector`: Selektor für die gewünschte Schriftgröße (z.B. `.id-font-size-small`, `.id-font-size-medium`, `.id-font-size-large`). |
| `setRelationColor` | **Set Relation Color**: Setzt die Farbe einer Relation. |1) `fromItemIndex`: Item-Index des Start-Items der Relation.<br>2) `toItemIndex`: Item-Index des Ziel-Items der Relation.<br>3) `relationStackIndex`: Stack-Index der Relation.<br>4) `colorIndex`: Index der Farbe in der Farbpalette (0-basiert). |
| `setRelationLineStyle` | **Set Relation Line Style**: Setzt den Linienstil einer Relation. |1) `fromItemIndex`: Item-Index des Start-Items der Relation.<br>2) `toItemIndex`: Item-Index des Ziel-Items der Relation.<br>3) `relationStackIndex`: Stack-Index der Relation.<br>4) `lineStyleSelector`: Selektor für den gewünschten Linienstil. |
| `setRelationVisibleUpToLevel` | **Set Relation Visible Up To Level**: Stellt ein, bis zu welchem Level eine Relation sichtbar ist. |1) `fromItemIndex`: Item-Index des Start-Items der Relation.<br>2) `toItemIndex`: Item-Index des Ziel-Items der Relation.<br>3) `relationStackIndex`: Stack-Index der Relation.<br>4) `levelSelector`: Selektor für das gewünschte Sichtbarkeitslevel. |
| `setTextNote` | **Set Text Note**: Fügt einem Item eine Textnotiz hinzu oder bearbeitet sie. |1) `itemIndex`: Item-Index des Items, zu dem die Notiz gehört.<br>2) `stackIndex`: Stack-Index des Items, zu dem die Notiz gehört.<br>3) `textContent`: Textinhalt der Notiz (HTML- oder Klartext). |
| `insertTable` | **Insert Table**: Fügt eine Tabelle in ein Rich-Text-Feld (Name oder Beschreibung) ein. |1) `toolbarSelector`: CSS-Selektor der Toolbar, die den "Tabelle einfügen"-Button enthält.<br>2) `rowCount`: Anzahl der Zeilen.<br>3) `columnCount`: Anzahl der Spalten.<br>4) `iframeSelector`: CSS-Selektor des iFrames, der das Rich-Text-Feld enthält.<br>5)-14) `cellContents`: Die Inhalte der ersten 10 Zellen (optional). |
| `closeFirstTableViewColumn` | **Close First Table View Column**: Schließt die erste Spalte in der Tabellenansicht. | - |
| `closeLastTableViewColumn` | **Close Last Table View Column**: Schließt die letzte Spalte in der Tabellenansicht. | - |
| `openFirstTableViewColumnChilds` | **Open First Table View Column Childs**: Öffnet die Kind-Items der ersten Spalte in der Tabellenansicht. | - |
| `searchItemName` | **Search Item Name**: Sucht nach einem Item anhand seines Namens. |1) `itemName`: Der Name des zu suchenden Items. |
| `presentItem` | **Present Item**: Zoomt und zentriert die Ansicht auf ein Item und kann optional den Item-Namen sprechen. |1) `itemName`: Name des Items.<br>2) `hitIndex`: (Optional) Hit-Index, falls mehrere Items denselben Namen haben.<br>3) `speakText`: (Optional) `true`, um den Item-Text danach zu sprechen. |
| `presentAndSpeakItem` | **Present And Speak Item**: Zoomt und zentriert die Ansicht auf ein Item und spricht den Item-Namen. |1) `itemName`: Name des Items.<br>2) `hitIndex`: (Optional) Hit-Index, falls mehrere Items denselben Namen haben. |
| `waitUntilVisible` | **Wait Until Visible**: Wartet, bis ein spezifisches Element sichtbar wird. |1) `elementSelector`: CSS-Selektor des Elements, auf das gewartet werden soll. |
| `waitUntilInvisible` | **Wait Until Invisible**: Wartet, bis ein spezifisches Element unsichtbar wird. |1) `elementSelector`: CSS-Selektor des Elements, auf das gewartet werden soll. |

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
| `wse` | **Wait Speech End**: Wartet, bis die vorhergehende Sprachausgabe beendet ist, bevor die nächste Aktion ausgeführt wird. Dies ist nützlich, wenn eine `spt`-Aktion im Hintergrund läuft. | `{"a": "dsi", "wse": true}` |

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
  { "wa": 1000 },
  { "a": "spt", "t": "Jetzt könnte ich eine Gruppenaktion durchführen. Vorerst hebe ich die Auswahl einfach wieder auf." },
  { "a": "dsi" },
  { "a": "spt", "t": "Zuletzt deaktiviere ich den Mehrfachauswahl-Modus." },
  { "m": "disableMultiSelect" }
]
```
