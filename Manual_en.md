# Screenplay Script Language Manual

## 1. Introduction

This document describes the scripting language for automating and controlling presentations within the InfoRapid KnowledgeBase Builder. The scripts are used to define a sequence of actions that are automatically executed to create dynamic and interactive presentations.

At its core, a script is a JSON file containing an array of "action objects". Each object represents a single step in the presentation, such as clicking an element, typing text, or calling more complex macros.

### Basic Concepts
- **Item**: A node in the mind map diagram.
- **Relation**: A connecting edge between two items.
- **Action**: A single command within a script, represented by a JSON object.
- **Macro**: A predefined sequence of actions that can be invoked with a single command to simplify complex operations.

## 2. Basic Concepts

### 2.1. Script Structure

A script is a JSON array of action objects. The actions are executed strictly in the order in which they are defined in the array.

**Example:**
```json
[
  { "a": "ci", "i": ".id-show-sidebar" },
  { "a": "spt", "t": "The sidebar will now be opened." },
  { "m": "setZoom", "p": [50] }
]
```
This script would first click an element, then speak some text, and finally adjust the zoom level.

### 2.2. The Action Object

Each action object consists of key-value pairs. The most important key is `"a"` (for action) or `"m"` (for macro).

- `{"a": "..."}`: Defines an atomic action.
- `{"m": "..."}`: Defines the invocation of a macro.

Other keys in the object serve as parameters for the respective action or macro.

### 2.3. Identifying Elements

Elements (items, relations, UI buttons) can be addressed in various ways:

- **CSS Selector (`i`):** The most common way to address UI elements.
  - `{"a": "ci", "i": ".id-ok-button"}`
- **Item Index (`ii`, `if`, `it`):** To address items or relations in the diagram directly via their numerical index. The root item has the index `0`. This index is assigned based on a **Breadth-First Search (BFS)** traversal of the mind map: first the root item (`ii=0`), then all its children (e.g., `ii=1`, `ii=2`, etc., typically in creation order), then all children of `ii=1`, then children of `ii=2`, and so on, level by level. This makes `ii` a **predictable, absolute** index, which remains stable unless the item is deleted or the diagram structure is fundamentally altered. It is a highly reliable way to reference items if the creation order and structure are known.
  - `{"a": "cii", "ii": 0}` (Clicks the root item)
- **Name (`in`, `inf`, `int`):** Addresses an item by its displayed name (`in`) or a relation by the displayed names of its start (`inf`) and end (`int`) items.
  - `{"a": "pi", "in": "Introduction"}`
  - `{"a": "ort", "inf": "Start-Item-Name", "int": "End-Item-Name"}`
- **Stack (`bi` - Back Index):** Actions can place items/relations onto an internal "stack" (`uis`, `urs`). `bi` is a **relative** index that counts from the *end* of this stack. `bi=0` always refers to the *last* item pushed onto the stack. This is useful when elements are created dynamically and then need to be directly reused.

    **Important Stack Dynamics:**
    *   The `uis` (Update Item Stack) action pushes the *currently selected item* to the top of this stack, making it `bi=0`. Consequently, **all other items already on the stack have their `bi` value incremented by one.**
    *   Macros like `newItemName` and `newSimpleItems` internally use `uis` for each item they create. This means these macros will modify the stack by placing new items at `bi=0`, which consequently **increments the `bi` values of previously stacked items by one** for each item added.
    *   **Robust Strategy for Item Referencing:** When creating child items for a specific parent, **it is generally more robust to use `parentII` if the parent's item index (`ii`) is known and stable.** This avoids the complexities of dynamically shifting `bi` values. If `parentBI` is necessary (e.g., when `ii` is not directly predictable, or for complex dynamic stack manipulations), it is crucial to ensure that the intended parent item is positioned at `bi=0` just before the child creation. This can be achieved by:
        1.  Calculating the parent's current `bi` (accounting for any items added to the stack since the parent was last pushed).
        2.  Selecting the parent using `{"a": "si", "bi": <parent_current_bi>}`.
        3.  Immediately executing `{"a": "uis"}` to push the selected parent to the top of the stack, making it `bi=0`.
        4.  Then, calling `newItemName` or `newSimpleItems` with `parentII=-1` (to explicitly ignore `ii`) and `parentBI=0`.

### 2.4. Multi-Selection

The screenplay logic supports selecting multiple items to perform group actions. This requires enabling a special mode.

- **Workflow:**
    1.  Enable the mode using the `{"m": "enableMultiSelect"}` macro.
    2.  Select multiple items by providing an **array of indices** to the `ii` or `bi` parameter of an action like `si`.
    3.  Perform any desired group actions (e.g., formatting).
    4.  Clear the selection using `{"a": "dsi"}` or disable the mode with `{"m": "disableMultiSelect"}`.
- **Important:** If multi-select mode is not enabled, selecting a new item will deselect any previously selected item.

## 3. Action Reference (Key "a")

The following table describes the atomic actions controlled by the `"a"` key.

| Action (`a`) | Description | Parameters |
| :--- | :--- | :--- |
| `atp` | **Append Text to Panel**: Appends HTML text to the text panel. | `t`: The HTML text. |
| `blur` | **Blur**: Triggers the `blur` event on an element. | `i`: CSS selector of the element. |
| `ci` | **Click Item**: Clicks a UI element. | `i`: CSS selector. `iifs`: (Optional) Selector for an iFrame containing the element. `et`: (Optional) Event type (e.g., "mousedown"). |
| `cii` | **Click Item by Index**: Clicks an item in the diagram by its index. | `ii`: Index (or array of indices) of the item. |
| `ciis` | **Click Item Selector by Index**: Clicks the *selector* of an item by its index. | `ii`: Index (or array of indices) of the item. |
| `cri` | **Click Relation by Index**: Clicks a relation by the indices of start and end items. | `if`: Index of the start item. `it`: Index of the end item. |
| `csg` | **Close Side Gallery**: Closes a specified side gallery. | `sg`: CSS selector of the side gallery. |
| `csi` | **Click Stack Item**: Clicks an item from the stack. | `bi`: Index (or array of indices) from the end of the stack (0 = last). |
| `csis` | **Click Stack Item Selector**: Clicks the selector of an item from the stack. | `bi`: Index (or array of indices) from the end of the stack. |
| `csr` | **Click Stack Relation**: Clicks a relation from the stack. | `bi`: Index (or array of indices) from the end of the stack. |
| `ctvc` | **Click Table View Cell**: Clicks a cell in the table view. | `col`: Column index. `row`: Row index. |
| `dcii` | **Double Click Item by Index**: Double-clicks an item by its index. | `ii`: Index of the item. |
| `dcsi` | **Double Click Stack Item**: Double-clicks an item from the stack. | `bi`: Index from the end of the stack. |
| `dib` | **Drag Item By**: Drags an element by a given delta. | `i`: Selector of the element. `dx`: Pixels in X direction. `dy`: Pixels in Y direction. |
| `dift` | **Drag Item From To**: Drags a diagram item to another diagram item. | `iif`/`bif`: Index/Stack index of the item to drag (source). `iit`/`bit`: Index/Stack index of the target item (where to drag to). |
| `drft` | **Drag Relation End Point From To**: Drags an endpoint of an existing relation from a source item to a target item to re-route the relation. | `iif`/`bif`: Index/Stack index of the source item from which the relation endpoint is dragged.<br>`inf`: Name of the source item from which the relation endpoint is dragged.<br>`iit`/`bit`: Index/Stack index of the target item to which the relation endpoint is dragged.<br>`int`: Name of the target item to which the relation endpoint is dragged. |
| `difctd` | **Drag Item From Category To Diagram**: Drags an item from a category (e.g., in the sidebar) to a specific item in the diagram. | `if`: CSS selector of the source item in the category. `iit`/`bit`: Index/Stack index of the target item in the diagram. |
| `difdtc` | **Drag Item From Diagram To Category**: Drags an item from the diagram to a category area (e.g., the sidebar) to remove or move it. | `iif`/`bif`: Index/Stack index of the item to drag in the diagram. `it`: CSS selector of the target category area. |
| `difcttv` | **Drag Item From Category To Table View**: Drags an item from a category (e.g., in the sidebar) to a specific cell in the table view. | `if`: CSS selector of the source item in the category. `colt`: Column index of the target cell (0-based). `rowt`: Row index of the target cell (0-based). |
| `diftvtc` | **Drag Item From Table View To Category**: Drags an item from a cell in the table view to a category area. | `colf`: Column index of the source cell (0-based). `rowf`: Row index of the source cell (0-based). `it`: CSS selector of the target category area. |
| `dtvcft` | **Drag Table View Cell From To**: Drags an item from one table view cell to another table view cell. | `colf`: Column index of the source cell (0-based). `rowf`: Row index of the source cell (0-based). `colt`: Column index of the target cell (0-based). `rowt`: Row index of the target cell (0-based). |
| `dsi` | **Deselect All Items**: Clears all current selections in the diagram. | |
| `focus` | **Focus**: Triggers the `focus` event on an element. | `i`: CSS selector of the element. |
| `htp` | **Hide Text Panel**: Hides the text panel. | |
| `kd` | **Key Down**: Simulates pressing a key. | `i`: Selector of the element that has focus. `kc`: The key code of the key. |
| `mcte` | **Move Cursor to End**: Moves the cursor to the end in a text field. | `iifs`: Selector of the iFrame containing the text field. |
| `oit` | **Open Item Toolbar**: Opens an item's toolbar (by clicking). | `ii`/`bi`: Index/Stack index of the item. |
| `ort` | **Open Relation Toolbar**: Opens a relation's toolbar. | `if`, `it` / `bi`: Indices of the items or stack index of the relation.<br>`inf`, `int`: Names of the start and end items of the relation. |
| `pi` | **Present Item**: Zooms and centers the view on an item. | `in`: Name of the item. `hi`: (Optional) Hit index, if multiple items have the same name. `spt`: (Optional) `true` to speak the item text afterward. |
| `si` | **Select Item**: Selects an item without clicking it. | `ii`/`bi`: Index/Stack index of the item. |
| `siiv` | **Scroll Item Into View**: Scrolls the diagram to make a specific item visible and, if possible, centered. | `ii`: Index of the item. `bi`: Stack index of the item. |
| `sit` | **Set Ignore Templates**: Determines whether templates should be ignored. | `it`: Boolean value. |
| `spt` | **Speak Text**: Reads text aloud using speech output. | `t`: The text to be spoken. |
| `st` | **Select Text**: Selects text in an input field. | `i`: Selector of the field. `iifs`: (Optional) iFrame selector. `t`: The text to be selected. |
| `stp` | **Show Text Panel**: Shows the text panel with initial content. | `t`: The initial HTML text. |
| `tt` | **Type Text**: Types text into a focused field. | `i`, `iifs`: Selector of the field. `t`: The text. `pt`: (Optional) `true` for direct insertion. `tc`: (Optional) `true` to trigger "changed" event. |
| `ttd` | **Type Text Direct**: Types text without prior focus. | `i`: Selector of the field. `t`: The text. `pt`: (Optional) `true` for insertion. |
| `ttkb` | **Type Text with Keyboard**: Simulates text input keystroke by keystroke. | `i`: Selector of the field. `t`: The text. |
| `tspt` | **Toggle Speak Text**: Globally turns speech output on or off. | |
| `uis` | **Update Item Stack**: Puts the currently selected item onto the stack. | `fcd`: (Optional) `true` to read from the current diagram. |
| `urs` | **Update Relation Stack**: Puts the currently selected relation onto the stack. | |
| `slow`, `normal`, `fast` | Changes the global script execution speed. | |

## 4. Macro Reference (Key "m")

Macros are shortcuts for a chain of actions. They are invoked via the `"m"` key. The parameters for a macro are passed in the `"p"` key as an array. The order of the parameters in the array is crucial, as the macro uses the values according to its internal definition.

| Macro (`m`) | Description | Parameters (`p`) |
| :--- | :--- | :--- |
| `showSidebar` | **Show Sidebar**: Displays the main sidebar. Clicks the sidebar toggle button if the sidebar is not already visible. | - |
| `showSideGallery` | **Show Side Gallery**: Displays the main sidebar and then opens a specific side gallery. |1) `gallerySelector`: CSS selector of the gallery to open (e.g., `.id-viewmode`). |
| `pinSideGallery` | **Pin Side Gallery**: Opens a specific side gallery and then pins it, ensuring it remains open. |1) `gallerySelector`: CSS selector of the gallery to pin (e.g., `.id-editmode`). |
| `closeSideGallery` | **Close Side Gallery**: Closes a specific side gallery. |1) `gallerySelector`: CSS selector of the gallery to close (e.g., `.id-viewmode`). |
| `disableMultiSelect` | Disables multi-selection mode, returning to single-selection behavior. | - |
| `enableMultiSelect` | Enables multi-selection mode. This must be called before attempting to select multiple items. | - |
| `execItemCommand` | **Execute Item Command**: Opens the "Edit Mode" side gallery if not already open, then executes a specific command related to an item. |1) `commandSelector`: CSS selector of the command to execute (e.g., `.id-new-item`, `.id-delete-item`). |
| `execInsertCommand` | **Execute Insert Command**: Opens the "Edit Mode" side gallery if not already open, then executes a specific insert command. |1) `commandSelector`: CSS selector of the command to execute (z.B. `.id-insert-textnote`). |
| `execUndoCommand` | **Execute Undo Command**: Opens the "Edit Mode" side gallery, selects a specific undo command from the list of recent actions, and then closes the gallery. |1) `undoItemId`: The ID of the specific undo item (e.g., `1` for the last action). |
| `setLayoutMode` | **Set Layout Mode**: Opens the "View Mode" side gallery and sets the layout mode of the diagram (e.g., hierarchical, circular). |1) `layoutSelector`: CSS selector for the desired layout mode (e.g., `.id-diagram-layout-free`, `.id-diagram-layout-tree`). |
| `selectDiagramMode` | **Select Diagram Mode**: Opens the "View Mode" side gallery and selects a diagram display mode (e.g., 2D, 3D, table). |1) `modeSelector`: CSS selector for the desired mode (e.g., `.id-diagram-mode-3d`, `.id-diagram-mode-table`). |
| `setShowLabels` | **Set Show Labels**: Opens the "View Mode" side gallery and sets how labels are displayed in the diagram (e.g., always, on hover, never). |1) `labelOptionSelector`: CSS selector for the desired label display option (e.g., `.id-show-labels-always`). |
| `selectCategory` | **Select Category**: Selects a specific category in a dropdown menu. |1) `categoryName`: The name of the category to select. |
| `showCategoryItems` | **Show Category Items**: Displays the gallery with items of a specific category. |1) `categoryName`: The name of the category whose items should be displayed. |
| `clickItemToolbar` | **Click Item Toolbar**: Opens an item's toolbar and then clicks a button within that toolbar. |1) `itemIndex`: Item index of the item (use `-1` if addressing by `stackIndex`).<br>2) `stackIndex`: Stack index of the item (0 for the last selected item, `-1` if addressing by `itemIndex`).<br>3) `buttonSelector`: CSS selector of the toolbar button (e.g., `.id-edit-item`). |
| `clickRelationToolbar` | **Click Relation Toolbar**: Opens a relation's toolbar and then clicks a button within that toolbar. |1) `fromItemIndex`: Item index of the relation's start item (use `-1` if addressing by `stackIndex`).<br>2) `toItemIndex`: Item index of the relation's target item (use `-1` if addressing by `stackIndex`).<br>3) `stackIndex`: Stack index of the relation (0 for the last selected relation, `-1` if addressing by item indices).<br>4) `buttonSelector`: CSS selector of the toolbar button (e.g., `.id-edit-relation`). |
| `clickQuickAccessToolbar` | **Click Quick Access Toolbar**: Clicks a button in the quick access toolbar if it's not already selected, and waits for the diagram to update. |1) `buttonSelector`: CSS selector of the button in the quick access toolbar. |
| `clickQuickAccessToolbarDirect` | **Click Quick Access Toolbar Direct**: Directly clicks a button in the quick access toolbar if it's not already selected. |1) `buttonSelector`: CSS selector of the button in the quick access toolbar. |
| `importWikipediaArticle` | **Import Wikipedia Article**: Imports a Wikipedia article for an item. Opens the Wikipedia import dialog via the item toolbar, confirms the import, and updates the item stack with the newly imported item. |1) `itemStackIndex`: Stack index of the item to which the article should be imported. |
| `pinEditItemDialog` | **Pin Edit Item Dialog**: Pins the "Edit Item" dialog, ensuring it remains open if it's not already pinned. | - |
| `openNewItemDialog` | **Open New Item Dialog**: Opens the dialog for creating a new item. Sets whether templates should be ignored, then clicks the "New Item" button. |1) `ignoreTemplates`: (Boolean) `true` to ignore templates; `false` otherwise.<br>2) `parentItemIndex`: Item index of the parent item under which the new item should be created (use `-1` if `parentStackIndex` is used).<br>3) `parentStackIndex`: Stack index of the parent item (use `0` if the parent was just pushed with `uis`; `-1` if `parentItemIndex` is used). |
| `createNewItem` | **Create New Item**: Fills out the fields of the "Edit Item" dialog (name, description, hyperlink, category) and optionally supports inserting tables into the text fields. |1) `itemName`: The name of the item.<br>2)-13) `nameTableParams`: Parameters for the `insertTable` macro in the name field (rows, columns, cell contents).<br>14) `itemDescription`: The description of the item.<br>15)-26) `descriptionTableParams`: Parameters for the `insertTable` macro in the description field (rows, columns, cell contents).<br>27) `itemHyperlink`: The hyperlink of the item.<br>28) `itemCategory`: The category of the item. |
| `createNewItemAndContinue` | **Create New Item And Continue**: Creates a new item using the `createNewItem` macro and then closes the dialog with "OK". |1)-28) `itemParams`: All parameters passed to the `createNewItem` macro. |
| `createNewSimpleItem` | **Create New Simple Item**: Creates a new item with only the basic information (name, description, hyperlink, category). A simplified version of `createNewItem`. |1) `itemName`: The name of the item.<br>2) `itemDescription`: The description of the item.<br>3) `itemHyperlink`: The hyperlink of the item.<br>4) `itemCategory`: The category of the item. |
| `createNewSimpleItemAndContinue` | **Create New Simple Item And Continue**: Creates a simple item using `createNewSimpleItem` and then closes the dialog with "OK". |1)-4) `simpleItemParams`: All parameters passed to the `createNewSimpleItem` macro. |
| `newItemDialogCancel` | **New Item Dialog Cancel**: Clicks the "Cancel" button in the "Edit Item" dialog. | - |
| `newItemDialogOk` | **New Item Dialog OK**: Clicks the "OK" button in the "Edit Item" dialog and pushes the newly created item onto the stack. | - |
| `newItem` | **New Item**: Opens the "New Item" dialog, fills it out, and confirms with "OK". |1) `ignoreTemplates`: (Boolean) `true` to ignore templates; `false` otherwise.<br>2) `parentItemIndex`: Item index of the parent item.<br>3) `parentStackIndex`: Stack index of the parent item.<br>4)-31) `itemCreationParams`: All other parameters required for item creation (name, description, etc.). |
| `newSimpleItems` | **New Simple Items**: Creates multiple simple items one after another. **Note:** This macro implicitly uses `uis` for *each child* created, pushing them to the stack in order and consequently incrementing the `bi` values of other items already on the stack. **Prefer `parentII` if the parent's item index is predictable.** If using `parentBI`, ensure it correctly points to the intended parent at the time of execution, potentially using the `si` then `uis` strategy as described in section 2.3. |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) Item index of the parent item (use `-1` if addressing by `parentBI`).<br>3) Stack index of the parent item (use `0` if parent was just pushed with `uis`).<br>4) Array of item arrays, where each item array contains `[Name, Description, Hyperlink, Category]`. All four parameters for each item array must be provided, using empty strings for unused values. |
| `newItemName` | **New Item Name**: Creates a new item with only a name. **Note:** This macro implicitly uses `uis` for the newly created item, pushing it to `bi=0` on the stack and consequently incrementing the `bi` values of other items already on the stack. **Prefer `parentII` if the parent's item index is predictable.** If using `parentBI`, ensure it correctly points to the intended parent at the time of execution, potentially using the `si` then `uis` strategy as described in section 2.3. |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) Item index of the parent item (use `-1` if addressing by `parentBI`).<br>3) Stack index of the parent item (use `0` if parent was just pushed with `uis`).<br>4) Name of the new item. All four parameters for the macro must be provided, using empty strings for unused values in the `newItem` internal call. |
| `newRelation` | **New Relation**: Creates a new relation between two diagram items. This macro simulates dragging a connection line from a start item to a target item. |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) `fromItemIndex`: Item index of the start item.<br>3) `fromStackIndex`: Stack index of the start item.<br>4) `toItemIndex`: Item index of the target item.<br>5) `toStackIndex`: Stack index of the target item. |
| `newRelationFromCategoryItemToDiagram` | **New Relation From Category Item To Diagram**: Creates a new relation by dragging an item from a category (e.g., sidebar) to an item in the diagram. |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) `categoryName`: Name of the category the source item is from.<br>3) `categoryItemName`: Name of the source item in the category.<br>4) `diagramItemIndex`: Item index of the target item in the diagram.<br>5) `diagramStackIndex`: Stack index of the target item in the diagram. |
| `newRelationFromDiagramItemToCategory` | **New Relation From Diagram Item To Category**: Creates a new relation by dragging an item from the diagram to an item in a category (e.g., sidebar). |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) `categoryName`: Name of the category the target item belongs to.<br>3) `diagramItemIndex`: Item index of the source item in the diagram.<br>4) `diagramStackIndex`: Stack index of the source item in the diagram.<br>5) `categoryItemName`: Name of the target item in the category. |
| `newRelationFromCategoryItemToTableView` | **New Relation From Category Item To Table View**: Creates a new relation by dragging an item from a category (e.g., sidebar) to a cell in the table view. |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) `categoryName`: Name of the category the source item is from.<br>3) `categoryItemName`: Name of the source item in the category.<br>4) `targetColumnIndex`: Column index of the target cell (0-based).<br>5) `targetRowIndex`: Row index of the target cell (0-based). |
| `editRelationEndPoint` | **Edit Relation End Point**: Modifies the endpoint of an existing relation. The first three parameters (`fromItemIndex`, `toItemIndex`, `relationStackIndex`) are used to select the relation to be edited. To then move either the start or end point of the relation, the following four parameters (`newFromItemIndex`, `newFromStackIndex`, `newToItemIndex`, `newToStackIndex`) are used. One fills either the parameters for the new start point (`newFromItemIndex`, `newFromStackIndex`) and sets the parameters for the end point (`newToItemIndex`, `newToStackIndex`) to `-1`, or vice versa. The respective index of the new start or end point must be specified. |1) `fromItemIndex`: Item index of the start item of the relation to be edited.<br>2) `toItemIndex`: Item index of the target item of the relation to be edited.<br>3) `relationStackIndex`: Stack index of the relation to be edited.<br>4) `newFromItemIndex`: Item index of the *new* start item for the relation (use `-1` if the endpoint is being moved).<br>5) `newFromStackIndex`: Stack index of the *new* start item for the relation (use `-1` if the endpoint is being moved).<br>6) `newToItemIndex`: Item index of the *new* target item for the relation (use `-1` if the start point is being moved).<br>7) `newToStackIndex`: Stack index of the *new* target item for the relation (use `-1` if the start point is being moved). |
| `editRelationEndPointByItemName` | **Edit Relation End Point By Item Name**: Changes the endpoint of an existing relation by identifying the relation and the new endpoint using the names of start and target items. |1) `fromItemName`: Name of the start item of the relation to be edited.<br>2) `toItemName`: Name of the target item of the relation to be edited.<br>3) `newFromItemName`: Name of the *new* start item for the relation (use an empty string if the endpoint is being moved).<br>4) `newToItemName`: Name of the *new* target item for the relation (use an empty string if the start point is being moved). |
| `clickButton` | **Click Button**: Clicks a specific button. |1) `buttonSelector`: CSS selector of the button to click. |
| `dragViewSliderBy` | **Drag View Slider By**: Drags a view slider (e.g., zoom, distance, rotation) by a specific amount. |1) `sliderSelector`: CSS selector of the slider (e.g., `.id-viewmode-zoom`).<br>2) `deltaX`: Pixels by which the slider should be moved (positive for right, negative for left). |
| `resetViewSlider` | **Reset View Slider**: Resets a view slider to its default value. |1) `sliderSelector`: CSS selector of the slider (e.g., `.id-viewmode-zoom`). |
| `newRelationFromTableViewItemToCategory` | **New Relation From Table View Item To Category**: Creates a new relation by dragging an item from a cell in the table view to an item in a category (e.g., sidebar). |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) `categoryName`: Name of the category the target item belongs to.<br>3) `sourceColumnIndex`: Column index of the source cell (0-based).<br>4) `sourceRowIndex`: Row index of the source cell (0-based).<br>5) `categoryItemName`: Name of the target item in the category. |
| `editRelation` | **Edit Relation**: Edits the properties of a relation via the edit dialog. |1) `fromItemIndex`: Item index of the relation's start item.<br>2) `toItemIndex`: Item index of the relation's target item.<br>3) `stackIndex`: Stack index of the relation (counted from the end).<br>4) `relationName`: Name of the relation.<br>5) `showAsTable`: (Boolean) Control of relation visibility (corresponds to the 'Show as Table' checkbox in the dialog).<br>6) `backRelationName`: Name of the back-relation.<br>7) `backShowAsTable`: (Boolean) Control of back-relation visibility.<br>8) `description`: Description of the relation.<br>9) `category`: Category of the relation. |
| `setZoom` | **Set Zoom**: Sets the zoom factor of the diagram. |1) `zoomValue`: Value for the zoom slider displacement (relative, e.g., `50` for half a displacement to the right). |
| `resetZoom` | **Reset Zoom**: Resets the zoom factor of the diagram to its default value. | - |
| `setDistance` | **Set Distance**: Sets the distance between items in the diagram. |1) `distanceValue`: Value for the distance slider displacement (relative, e.g., `50`). |
| `resetDistance` | **Reset Distance**: Resets the distance between items in the diagram to its default value. | - |
| `setRotation` | **Set Rotation**: Sets the rotation of the diagram (in 3D mode). |1) `rotationValue`: Value for the rotation slider displacement (relative, e.g., `50`). |
| `resetRotation` | **Reset Rotation**: Resets the rotation of the diagram (in 3D mode) to its default value. | - |
| `setPerspective` | **Set Perspective**: Sets the perspective of the diagram (in 3D mode). |1) `perspectiveValue`: Value for the perspective slider displacement (relative, e.g., `50`). |
| `resetPerspective` | **Reset Perspective**: Resets the perspective of the diagram (in 3D mode) to its default value. | - |
| `saveView` | **Save View**: Saves the current view as a bookmark. |1) `bookmarkName`: Name of the bookmark.<br>2) `saveRootItem`: (Boolean) `true` to save the root item; `false` otherwise.<br>3) `setAsInitialView`: (Boolean) `true` to set the view as the initial view; `false` otherwise. |
| `restoreView` | **Restore View**: Restores a saved view by its ID. |1) `bookmarkId`: ID of the bookmark. |
| `itemCategoryFilter` | **Item Category Filter**: Filters the displayed items in the diagram by category. |1) `categoryName`: Name of the category to filter by. |
| `relationCategoryFilter` | **Relation Category Filter**: Filters the displayed relations in the diagram by category. |1) `categoryName`: Name of the category to filter by. |
| `setItemColor` | **Set Item Color**: Sets the color of an item. |1) `itemIndex`: Item index of the item to format (use `-1` if addressing by `stackIndex`).<br>2) `stackIndex`: Stack index of the item to format (use `0` if the item was just pushed with `uis` or selected and pushed using `si` then `uis`).<br>3) `colorIndex`: Index of the color in the color palette (0-based, e.g., `0` for the first color). |
| `setItemFontSize` | **Set Item Font Size**: Sets the font size of an item. |1) `itemIndex`: Item index of the item to format.<br>2) `stackIndex`: Stack index of the item to format.<br>3) `fontSizeSelector`: Selector for the desired font size (e.g., `.id-font-size-small`, `.id-font-size-medium`, `.id-font-size-large`). |
| `setRelationColor` | **Set Relation Color**: Sets the color of a relation. |1) `fromItemIndex`: Item index of the start item of the relation.<br>2) `toItemIndex`: Item index of the target item of the relation.<br>3) `relationStackIndex`: Stack index of the relation.<br>4) `colorIndex`: Index of the color in the color palette (0-based). |
| `setRelationLineStyle` | **Set Relation Line Style**: Sets the line style of a relation. |1) `fromItemIndex`: Item index of the start item of the relation.<br>2) `toItemIndex`: Item index of the target item of the relation.<br>3) `relationStackIndex`: Stack index of the relation.<br>4) `lineStyleSelector`: Selector for the desired line style. |
| `setRelationVisibleUpToLevel` | **Set Relation Visible Up To Level**: Sets the visibility level for a relation. |1) `fromItemIndex`: Item index of the start item of the relation.<br>2) `toItemIndex`: Item index of the target item of the relation.<br>3) `relationStackIndex`: Stack index of the relation.<br>4) `levelSelector`: Selector for the desired visibility level. |
| `setTextNote` | **Set Text Note**: Adds or edits a text note for an item. |1) `itemIndex`: Item index of the item the note belongs to.<br>2) `stackIndex`: Stack index of the item the note belongs to.<br>3) `textContent`: Text content of the note (HTML or plain text). |
| `insertTable` | **Insert Table**: Inserts a table into a rich text field (name or description). |1) `toolbarSelector`: CSS selector of the toolbar, containing the "insert table" button.<br>2) `rowCount`: Number of rows.<br>3) `columnCount`: Number of columns.<br>4) `iframeSelector`: CSS selector of the iframe, containing the rich text field.<br>5)-14) `cellContents`: The contents of the first 10 cells (optional). |
| `closeFirstTableViewColumn` | **Close First Table View Column**: Closes the first column in the table view. | - |
| `closeLastTableViewColumn` | **Close Last Table View Column**: Closes the last column in the table view. | - |
| `openFirstTableViewColumnChilds` | **Open First Table View Column Childs**: Opens the child items of the first column in the table view. | - |
| `searchItemName` | **Search Item Name**: Searches for an item by its name. |1) `itemName`: The name of the item to search for. |
| `presentItem` | **Present Item**: Zooms and centers the view on an item, and can optionally speak the item's name. |1) `itemName`: Name of the item.<br>2) `hitIndex`: (Optional) Hit index, if multiple items have the same name.<br>3) `speakText`: (Optional) `true` to speak the item text afterward. |
| `presentAndSpeakItem` | **Present And Speak Item**: Zooms and centers the view on an item and speaks the item's name. |1) `itemName`: Name of the item.<br>2) `hitIndex`: (Optional) Hit index, if multiple items have the same name. |
| `waitUntilVisible` | **Wait Until Visible**: Waits until a specific element becomes visible. |1) `elementSelector`: CSS selector of the element to wait for. |
| `waitUntilInvisible` | **Wait Until Invisible**: Waits until a specific element becomes invisible. |1) `elementSelector`: CSS selector of the element to wait for. |

## 5. Control Properties

These properties can be added to any action object to control its execution.

| Property | Description | Example |
| :--- | :--- | :--- |
| `wa` | **Wait After**: Wait a specified time in milliseconds *after* the action is executed. | `{"a": "ci", "i": ".btn", "wa": 500}` |
| `waudr`| **Wait After Until Diagram is Ready**: Wait after the action until the diagram is finished rendering. Useful for actions that change the layout. | `{"m": "setLayoutMode", "p": [".id-layout-2d"], "waudr": true}` |
| `wto` | **Wait Timeout**: Maximum wait time in ms for `waudr`. Default is 10000. | `{"a": "ci", "i": ".btn", "waudr": true, "wto": 5000}` |
| `wb` | **Wait Before**: If `mbh` or `mbv` is used, this defines the maximum wait time in ms until the condition is met. | `{"a": "ci", "i": ".btn", "mbv": ".dialog", "wb": 3000}` |
| `mbh` | **Must Be Hidden**: The action is only executed if the specified element is *not* visible. | `{"a": "ci", "i": ".open", "mbh": ".dialog"}` |
| `mbv` | **Must Be Visible**: The action is only executed if the specified element is visible. | `{"a": "ci", "i": ".close", "mbv": ".dialog"}` |
| `mbs` | **Must Be Selected**: The action is only executed if the button/item is selected. | `{"a": "ci", "i": ".other-btn", "mbs": ".toggle-btn"}` |
| `mnbs` | **Must Not Be Selected**: The action is only executed if the button/item is *not* selected. | `{"a": "ci", "i": ".toggle-btn", "mnbs": ".toggle-btn"}` |
| `eine` | **Execute If Not Empty**: The action is only executed if the value of this key is not empty. Useful in macros to control optional actions. | `{"a": "tt", "t": "Optional Text", "eine": "Optional Text"}` |
| `wse` | **Wait Speech End**: Waits for the previous speech output to finish before executing the next action. This is useful when an `spt` action is running in the background. | `{"a": "dsi", "wse": true}` |

### 5.1 Debugging Aids

For faster debugging and development, you can add the following actions at the beginning of your script:
```json
{ "a": "fast" },
{ "a": "tspt" }
```
- `{"a": "fast"}`: Sets the global script execution speed to fast, minimizing delays.
- `{"a": "tspt"}`: Toggles the speech output off, preventing the script from speaking text.

## 6. Examples

### Example 1: Create and Connect Two Items

```json
[
  {
    "m": "newItemName",
    "p": [false, -1, 0, "First Item"]
  },
  {
    "m": "newItemName",
    "p": [false, -1, 0, "Second Item"]
  },
  {
    "m": "newRelation",
    "p": [false, -1, 1, -1, 0]
  }
]
```
**Explanation:**
1. Creates a new item named "First Item" as a child of the root item (by default, if no parent is specified using `parentII`) and implicitly pushes it onto the stack (`uis`), making it `bi=0`.
2. Creates a second item named "Second Item" (again, by default, if no parent is specified using `parentII`) and implicitly pushes it onto the stack (`uis`), making it `bi=0`. The "First Item" is now at `bi=1` (its `bi` value has incremented).
3. Creates a relation between the "Second Item" (`bi`: 0) and the "First Item" (`bi`: 1) on the stack.

### Example 2: Change Formatting

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
**Explanation:**
1. Selects the root item (`ii`: 0) and sets its color to the 5th color (`eq(4)`) in the palette. Note: when using `setItemColor`, `stackIndex` (parameter 2) can be `-1` if `itemIndex` (parameter 1) is used, or `0` if the item is at the top of the stack.
2. Zooms in the view.
3. Restores a previously saved view (camera position, zoom etc.) and waits until the diagram has been redrawn.

### Example 3: Multi-Selection

```json
[
  { "a": "spt", "t": "First, I will create three items." },
  { "m": "newItemName", "p": [false, -1, 0, "Item A"] },
  { "m": "newItemName", "p": [false, -1, 0, "Item B"] },
  { "m": "newItemName", "p": [false, -1, 0, "Item C"] },
  { "a": "spt", "t": "Now, I will enable multi-selection." },
  { "m": "enableMultiSelect" },
  { "a": "spt", "t": "I will select all three items using an array of back-indices." },
  { "a": "si", "bi": [0, 1, 2] },
  { "wa": 1000 },
  { "a": "spt", "t": "Now I could perform a group action. For now, I will just deselect them." },
  { "a": "dsi" },
  { "a": "spt", "t": "Finally, I will disable multi-selection mode." },
  { "m": "disableMultiSelect" }
]
```
