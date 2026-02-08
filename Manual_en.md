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
- **Name (`in`):** Addresses an item by its displayed name.
  - `{"a": "pi", "in": "Introduction"}`
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
| `dift` | **Drag Item From To**: Drags a diagram item to another. | `iif`/`bif`: Index/Stack index of the start item. `iit`/`bit`: Index/Stack index of the target item. |
| `dsi` | **Deselect All Items**: Clears all current selections in the diagram. | |
| `...` | *Various other drag actions for specific contexts (e.g., `difctd`, `diftvtc`)* | See `screenplay.js` for details. |
| `focus` | **Focus**: Triggers the `focus` event on an element. | `i`: CSS selector of the element. |
| `htp` | **Hide Text Panel**: Hides the text panel. | |
| `kd` | **Key Down**: Simulates pressing a key. | `i`: Selector of the element that has focus. `kc`: The key code of the key. |
| `mcte` | **Move Cursor to End**: Moves the cursor to the end in a text field. | `iifs`: Selector of the iFrame containing the text field. |
| `oit` | **Open Item Toolbar**: Opens an item's toolbar (by clicking). | `ii`/`bi`: Index/Stack index of the item. |
| `ort` | **Open Relation Toolbar**: Opens a relation's toolbar. | `if`, `it` / `bi`: Indices of the items or stack index of the relation. |
| `pi` | **Present Item**: Zooms and centers the view on an item. | `in`: Name of the item. `hi`: (Optional) Hit index, if multiple items have the same name. `spt`: (Optional) `true` to speak the item text afterward. |
| `si` | **Select Item**: Selects an item without clicking it. | `ii`/`bi`: Index/Stack index of the item. |
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
| `showSidebar` | Displays the main sidebar. | - |
| `showSideGallery` | Opens a specific gallery in the sidebar. |1) CSS selector of the gallery to open. |
| `closeSideGallery` | Closes a specific gallery. |1) CSS selector of the gallery to close. |
| `disableMultiSelect` | Disables multi-selection mode, returning to single-selection behavior. | - |
| `enableMultiSelect` | Enables multi-selection mode. This must be called before attempting to select multiple items. | - |
| `execItemCommand` | Executes a command in the "Edit Item" menu. |1) Selector of the command (e.g., `.id-new-item`). |
| `execInsertCommand` | Executes a command in the "Insert" menu. |1) Selector of the command (e.g., `.id-insert-textnote`). |
| `setLayoutMode` | Sets the layout mode of the diagram. |1) Selector for the desired mode (e.g., `.id-diagram-layout-free`). |
| `selectDiagramMode` | Selects a diagram display mode (e.g., 2D, 3D, table). |1) Selector for the mode (e.g., `.id-diagram-mode-3d`). |
| `setShowLabels` | Sets how labels are displayed (e.g., always show, only on hover). |1) Selector for the label display option (e.g., `.id-show-labels-always`). |
| `selectCategory` | Selects a category in a dropdown menu. |1) Name of the category. |
| `showCategoryItems`| Displays the gallery with items of a specific category. |1) Name of the category. |
| `clickItemToolbar` | Clicks a button in an item's toolbar. |1) Item index of the item.<br>2) Stack index of the item (counted from the end, e.g., `0` for the last selected item).<br>3) Selector of the toolbar button (e.g., `.id-edit-item`). |
| `newSimpleItems` | Creates multiple simple items one after another. **Note:** This macro implicitly uses `uis` for *each child* created, pushing them to the stack in order and consequently incrementing the `bi` values of other items already on the stack. **Prefer `parentII` if the parent's item index is predictable.** If using `parentBI`, ensure it correctly points to the intended parent at the time of execution, potentially using the `si` then `uis` strategy as described in section 2.3. |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) Item index of the parent item (use `-1` if addressing by `parentBI`).<br>3) Stack index of the parent item (use `0` if parent was just pushed with `uis`).<br>4) Array of item arrays, where each item array contains `[Name, Description, Hyperlink, Category]`. All four parameters for each item array must be provided, using empty strings for unused values. |
| `newItemName` | Creates a new item with only a name. **Note:** This macro implicitly uses `uis` for the newly created item, pushing it to `bi=0` on the stack and consequently incrementing the `bi` values of other items already on the stack. **Prefer `parentII` if the parent's item index is predictable.** If using `parentBI`, ensure it correctly points to the intended parent at the time of execution, potentially using the `si` then `uis` strategy as described in section 2.3. |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) Item index of the parent item (use `-1` if addressing by `parentBI`).<br>3) Stack index of the parent item (use `0` if parent was just pushed with `uis`).<br>4) Name of the new item. All four parameters for the macro must be provided, using empty strings for unused values in the `newItem` internal call. |
| `newRelation` | Creates a new relation between two items. |1) (Boolean) `ignoreTemplates` – specifies whether templates should be ignored.<br>2) Item index of the start item.<br>3) Stack index of the start item.<br>4) Item index of the target item.<br>5) Stack index of the target item. |
| `editRelation` | Edits the properties of a relation. |1) Item index of the relation's start item.<br>2) Item index of the relation's target item.<br>3) Stack index of the relation (counted from the end).<br>4) Name of the relation.<br>5) (Boolean) Control of relation visibility (corresponds to the 'Show as Table' checkbox in the dialog).<br>6) Name of the back-relation (Back-Relation Name).<br>7) (Boolean) Control of back-relation visibility (corresponds to the 'Show as Table' checkbox in the dialog).<br>8) Description of the relation.<br>9) Category of the relation. |
| `setZoom` | Sets the zoom factor. |1) Value for the zoom slider displacement (relative, e.g., `50` for half a displacement to the right). |
| `setDistance` | Sets the distance between items. |1) Value for the distance slider displacement (relative, e.g., `50`). |
| `setRotation` | Sets the rotation of the diagram (3D). |1) Value for the rotation slider displacement (relative, e.g., `50`). |
| `setPerspective` | Sets the perspective (3D). |1) Value for the perspective slider displacement (relative, e.g., `50`). |
| `saveView` | Saves the current view as a bookmark. |1) Name of the bookmark.<br>2) (Boolean) `true` to save the root item; `false` otherwise.<br>3) (Boolean) `true` to set the view as the initial view; `false` otherwise. |
| `restoreView` | Restores a saved view. |1) ID of the bookmark. |
| `setItemColor` | Sets the color of an item. |1) Item index of the item to format (use `-1` if addressing by `stackIndex`).<br>2) Stack index of the item to format (use `0` if the item was just pushed with `uis` or selected and pushed using `si` then `uis`).<br>3) Index of the color in the color palette (0-based, e.g., `0` for the first color). |
| `setItemFontSize` | Sets the font size of an item. |1) Item index of the item to format.<br>2) Stack index of the item to format.<br>3) Selector for the desired font size (e.g., `.id-font-size-small`, `.id-font-size-medium`, `.id-font-size-large`). |
| `setTextNote` | Adds or edits a text note for an item. |1) Item index of the item the note belongs to.<br>2) Stack index of the item the note belongs to.<br>3) Text content of the note (HTML or plain text). |

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
  { "a": "wa", "wa": 1000 },
  { "a": "spt", "t": "Now I could perform a group action. For now, I will just deselect them." },
  { "a": "dsi" },
  { "a": "spt", "t": "Finally, I will disable multi-selection mode." },
  { "m": "disableMultiSelect" }
]
```
