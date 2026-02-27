# PPOCRLabel Enhancement Implementation Plan

**Date:** February 27, 2026 **Status:** Planning Phase **Estimated Time:** 5-6
days

---

## Table of Contents

1. [Overview](#overview)
2. [Core Requirements](#core-requirements)
3. [Feature 1: Move Mode vs Edit Mode](#feature-1-move-mode-vs-edit-mode)
4. [Feature 2: Mouse Drag Selection Rectangle](#feature-2-mouse-drag-selection-rectangle)
5. [Feature 3: Drag Resize (Width/Height Only)](#feature-3-drag-resize-widthheight-only)
6. [Feature 4: Always-Visible Resize Control Panel](#feature-4-always-visible-resize-control-panel)
7. [Feature 5: Batch Resize Operations](#feature-5-batch-resize-operations)
8. [Feature 6: Rotation (Any Angle)](#feature-6-rotation-any-angle)
9. [Feature 7: Keyboard Shortcuts](#feature-7-keyboard-shortcuts)
10. [UI Layout](#ui-layout)
11. [Implementation Details](#implementation-details)
12. [File Structure](#file-structure)
13. [Testing Checklist](#testing-checklist)

---

## Overview

### Goals

- Add Label Studio-style Move Mode for faster workflow
- Enable batch operations via mouse drag selection
- Add always-visible on-screen controls (no modal dialogs)
- Implement resize from center (NOT Label Studio's anchor-based resize)
- Support rotation by any angle (not just 90° increments)
- Everything keyboard-accessible

### Design Principles

1. **No Modal Dialogs** - All controls visible on main screen
2. **Real-time Updates** - Changes apply instantly as you type/drag
3. **Batch Operations** - All operations work on multiple selected boxes
4. **Keyboard-First** - Every feature accessible via keyboard
5. **Always Visible** - Controls always on screen, disabled when not applicable

---

## Core Requirements

### Must Have

- ✅ Move Mode: Drag edges/corners to resize (NO point editing)
- ✅ Edit Mode: Edit individual vertex points (existing behavior)
- ✅ Mouse drag to select multiple boxes
- ✅ Resize from center (center stays fixed)
- ✅ Always-visible resize panel with real-time controls
- ✅ Batch resize via dragging any selected box
- ✅ Rotate by any angle (0-360°)
- ✅ Full keyboard shortcuts

### Nice to Have

- Preview during drag operations
- Visual guides for alignment
- Undo/redo for all operations

---

## Feature 1: Move Mode vs Edit Mode

### Purpose

Separate workspace interaction into two distinct modes:

- **Move Mode**: Fast workflow - select, move, resize width/height
- **Edit Mode**: Precise control - edit individual vertex points

### Move Mode Behavior

#### What You CAN Do:

- Click box to select (single selection)
- Ctrl+Click to add/remove from selection
- Drag in empty area to draw selection rectangle
- Click+drag box center to move
- Click+drag edge to resize width OR height
- Click+drag corner to resize both width AND height
- Use arrow keys to move selected boxes
- Delete selected boxes

#### What You CANNOT Do:

- Edit individual vertex points
- See or click on vertex handles
- Create new boxes (use Edit Mode)

#### Visual Indicators:

- Selected boxes show **solid border** (no vertex points)
- Edges and corners show **resize cursors** on hover:
  - Left/Right edges: ↔ cursor
  - Top/Bottom edges: ↕ cursor
  - Corners: diagonal ↗↖↘↙ cursors
- No point editing available

### Edit Mode Behavior

#### What You CAN Do:

- All existing PPOCRLabel functionality
- Click vertex to select it
- Drag vertex to move it
- Fine-tune box shapes point-by-point
- Use existing PPOCRLabel shortcuts for creation

#### Visual Indicators:

- Selected boxes show **vertex points** (circles/squares)
- Highlighted vertex changes color
- Create new box cursor (crosshair)

### Implementation Details

Extend the Canvas class in libs/canvas.py with mode constants (MOVE_MODE and
EDIT_MODE) and an interactionMode attribute defaulting to EDIT_MODE. Add
setMoveMode() and setEditMode() methods that toggle the mode and trigger a
canvas refresh. Modify the painting method to check the current mode and render
shapes accordingly—in Move Mode, draw only solid borders without vertex points;
in Edit Mode, show vertex points as usual.

### UI Elements

**Toolbar Buttons:**

In PPOCRLabel.py, create two checkable action buttons for Move Mode and Edit
Mode with keyboard shortcuts M and E respectively. Use QActionGroup to make the
mode buttons mutually exclusive so only one can be active at a time. Set Edit
Mode as the default checked state to maintain backward compatibility.

---

## Feature 2: Mouse Drag Selection Rectangle

### Purpose

Quickly select multiple boxes by dragging a selection rectangle (like Label
Studio).

### Behavior

#### Basic Selection:

1. Click in empty area (not on any box)
2. Hold and drag mouse
3. Dashed rectangle appears showing selection area
4. All boxes touching/inside rectangle are highlighted
5. Release mouse to finalize selection

#### Modifier Keys:

- **No modifier**: Replace current selection
- **Ctrl+Drag**: Add to existing selection
- **Alt+Drag**: Remove from existing selection
- **Shift+Drag**: Select intersecting boxes (even partially touching)

#### Visual Design:

```
Selection Rectangle Style:
- Border: Dashed blue line (2px)
- Fill: Semi-transparent blue (rgba(0, 123, 255, 0.1))
- Label: "Selecting: X boxes" near cursor
```

### Implementation Details

Add selection rectangle tracking properties to Canvas class (selectionRect,
selectionStartPoint, isDrawingSelectionRect). In mousePressEvent, detect clicks
in empty areas to initiate rectangle drawing. In mouseMoveEvent, update the
rectangle dimensions and highlight intersecting shapes in real-time. In
mouseReleaseEvent, finalize the selection by checking modifier keys (Ctrl to
add, Alt to remove, none to replace) and emit the selection changed signal.
Create helper methods to find shapes within the rectangle bounds using bounding
box intersection tests. In paintEvent, draw the selection rectangle with a
dashed blue border, semi-transparent fill, and a label showing the count of
boxes being selected.

---

## Feature 3: Drag Resize (Width/Height Only)

### Purpose

In Move Mode, drag edges/corners to resize boxes. Unlike Label Studio (which
anchors one edge), we resize **from center** so the box center stays fixed.

### Resize Modes

#### Mode 1: Resize from Center (DEFAULT)

- **Behavior**: Center point stays fixed, both sides expand/contract equally
- **Why**: Prevents boxes from moving around when resizing
- **Example**: Drag right edge → both left AND right edges move outward

#### Mode 2: Resize from Anchor (Label Studio Style)

- **Behavior**: Opposite edge stays fixed, dragged edge moves
- **Why**: For users familiar with Label Studio
- **Example**: Drag right edge → only right edge moves, left stays fixed

### Edge/Corner Detection

#### Edge Resize:

- **Top edge**: Changes height only
- **Bottom edge**: Changes height only
- **Left edge**: Changes width only
- **Right edge**: Changes width only

#### Corner Resize:

- **Top-left**: Changes both width and height
- **Top-right**: Changes both width and height
- **Bottom-left**: Changes both width and height
- **Bottom-right**: Changes both width and height

#### Detection Threshold:

- **Edge zone**: 10 pixels from edge
- **Corner zone**: 10x10 pixel square at corners
- **Hover cursor**: Changes based on what's detected

### Batch Resize

**Key Feature**: When multiple boxes are selected, dragging ANY box's
edge/corner resizes ALL selected boxes proportionally.

### Implementation Details

Add resize tracking attributes to Canvas including resizeFromCenter toggle,
resizingHandle identifier, resizeStartPoint, and originalShapeDimensions list.
Create a getResizeHandle method that checks if a point is within threshold
distance (10px) of shape edges or corners, prioritizing corners over edges.
Implement getResizeCursor to return appropriate directional cursors for each
handle type. In mousePressEvent when in Move Mode, detect handle clicks on
selected shapes and store original dimensions of all selected shapes. In
mouseMoveEvent, dynamically update cursor on hover and perform live resize on
all selected shapes based on mouse delta, calling either resizeFromCenter or
resizeFromAnchor methods accordingly. In mouseReleaseEvent, finalize the resize
by clearing tracking state and storing shapes for undo/redo.

Add width and height calculation methods to Shape class that find min/max of all
point coordinates. Implement resizeFromCenter method that calculates dimension
changes based on handle type (multiplied by 2 for bidirectional expansion),
computes scale factors, and transforms all points relative to the fixed center
point with minimum size constraints. Implement resizeFromAnchor method that
restores original points, determines which points belong to the dragged edge
based on their position relative to center, and translates only those points by
the mouse delta while keeping opposite edge fixed.

---

## Feature 4: Always-Visible Resize Control Panel

### Purpose

Provide on-screen controls for resizing and rotating selected boxes WITHOUT
modal dialogs. All changes apply in real-time.

### Panel Location

**Right side dock**, bottom section, below recognition results list.

### Panel States

#### State 1: No Selection (Disabled)

```
┌─────────────────────────────┐
│ SELECTED: 0 boxes           │ ← Gray text
│                             │
│ Width:  [◄][---]px[►]       │ ← All disabled/grayed
│ Height: [◄][---]px[►]       │
│                             │
│ Resize Mode:                │
│ ⦿ From Center               │ ← Always enabled (preference)
│ ○ From Anchor               │
│                             │
│ Batch: [+5][-5][+10][-10]   │ ← Disabled
│                             │
│ Rotation                    │
│ Angle: [0]° [Apply]         │ ← Disabled
│ Quick: [+15][−15][+90][−90] │ ← Disabled
│                             │
│ [Delete Selected (Del)]     │ ← Disabled
└─────────────────────────────┘
```

#### State 2: Single Box Selected

```
┌─────────────────────────────┐
│ SELECTED: 1 box             │ ← Green/bold
│                             │
│ Width:  [◄][150]px[►]       │ ← Shows exact value
│ Height: [◄][80]px[►]        │ ← Shows exact value
│                             │
│ Resize Mode:                │
│ ⦿ From Center               │ ← Selected mode
│ ○ From Anchor               │
│                             │
│ Batch: [+5][-5][+10][-10]   │ ← All enabled
│                             │
│ Rotation                    │
│ Angle: [0]° [Apply]         │ ← Can enter any angle
│ Quick: [+15][−15][+90][−90] │ ← Quick rotate buttons
│                             │
│ [Delete Selected (Del)]     │ ← Red, enabled
└─────────────────────────────┘
```

#### State 3: Multiple Boxes Selected

```
┌─────────────────────────────┐
│ SELECTED: 5 boxes           │ ← Bold, count
│                             │
│ Width:  [◄][165]px[►]       │ ← Shows average value
│ Height: [◄][95]px[►]        │ ← Shows average value
│                             │
│ Resize Mode:                │
│ ⦿ From Center               │ ← Applies to all
│ ○ From Anchor               │
│                             │
│ Batch: [+5][-5][+10][-10]   │ ← Adjusts ALL boxes
│                             │
│ Rotation                    │
│ Angle: [45]° [Apply]        │ ← Rotates ALL boxes
│ Quick: [+15][−15][+90][−90] │ ← Rotates ALL boxes
│                             │
│ [Delete Selected (Del)]     │ ← Deletes ALL boxes
└─────────────────────────────┘
```

### Real-Time Updates

#### When Dragging Box Edge:

1. User drags edge of one selected box
2. ALL selected boxes resize proportionally
3. Panel spinboxes update to show new dimensions
4. Changes apply immediately (no "Apply" button needed)

#### When Changing Spinbox:

1. User changes width spinbox from 150 to 160
2. ALL selected boxes resize immediately
3. Canvas updates in real-time
4. No delay or confirmation needed

#### When Clicking +5 Button:

1. User clicks [+5] batch adjust button
2. ALL selected boxes grow by 5px width AND height
3. Spinboxes update to show new values
4. Instant visual feedback

### Implementation Strategy

**Right Dock Layout:**

- Top section (30%): File list widget (existing)
- Middle section (30%): Recognition results list (existing)
- Bottom section (40%): Resize control panel (NEW, always visible)
- All sections in vertical layout

**Resize Control Panel Structure:**

- Container widget with styled background
- Vertical layout for all controls
- Sections separated by horizontal lines
- Each control group in a QGroupBox for organization

**Panel Controls:**

1. **Selection Label**
   - Bold text showing selected count
   - Color changes: Gray (none), Green (selected)
   - Format: "SELECTED: X boxes"

2. **Width Control Group**
   - Decrease button (◄): -1px
   - Spinbox: Direct input (1-10000px)
   - Increase button (►): +1px
   - Disabled when no selection

3. **Height Control Group**
   - Same layout as width
   - Independent control

4. **Resize Mode Radio Buttons**
   - Always enabled (preference setting)
   - "From Center" (default)
   - "From Anchor (Label Studio)"
   - Tooltips explain difference

5. **Batch Adjust Buttons**
   - 2x2 grid: [+5][-5][+10][-10]
   - Adjust both width AND height
   - Quick size changes

6. **Rotation Controls**
   - Angle spinbox: -360 to +360 degrees
   - Decimal support (0.1° precision)
   - Apply button for custom angles
   - Quick buttons: +15°, -15°, +90°, -90°

7. **Delete Button**
   - Red background when enabled
   - Gray when disabled
   - Keyboard hint (Del)

**Signal Connections:**

- Connect all controls to appropriate handlers
- Width/Height spinboxes trigger real-time resize
- Buttons trigger immediate actions
- Mode toggle updates canvas resize behavior
- Canvas signals update panel values

---

## Feature 6: Batch Resize Operations

### Purpose

Apply resize operations to multiple selected boxes simultaneously.

### Operations

#### 1. Spinbox Changes (Real-time)

Implement onWidthSpinChanged handler that blocks signals to prevent recursion,
calculates the delta between new and current width for each selected shape,
applies the resize transformation, updates the canvas display, and marks the
file as modified.

#### 2. Increment/Decrement Buttons

Create adjustWidth method that applies a pixel delta to all selected shapes,
updates the control panel values to reflect the changes, refreshes the canvas,
and marks the file as dirty.

#### 3. Batch Adjust Buttons

Implement batchAdjust method that stores original dimensions for each selected
shape, applies resizeFromCenter with both width and height deltas (divided by 2
for bidirectional scaling), updates control panel displays, refreshes canvas,
and sets dirty flag.

#### 4. Drag One, Resize All

In Canvas mouseMoveEvent, when a resize handle is active, calculate the mouse
position delta and iterate through all selected shapes, applying either
resizeFromCenter or resizeFromAnchor transformation to each using their stored
original dimensions, then emit dimension change signal and trigger repaint.

---

## Feature 7: Rotation (Any Angle)

### Purpose

Rotate selected boxes by any angle (not limited to 90° increments).

### UI Controls

#### Angle Spinbox:

- Range: -360° to +360°
- Allows decimal: 45.5°
- Positive = clockwise
- Negative = counter-clockwise
- Apply button executes rotation

#### Quick Buttons:

- `+15°` - Rotate 15° clockwise
- `−15°` - Rotate 15° counter-clockwise
- `+90°` - Rotate 90° clockwise
- `−90°` - Rotate 90° counter-clockwise

### Implementation Strategy

**Apply Rotation from Spinbox:** Read angle from spinbox in degrees, convert to
radians, save current state to undo stack, apply rotation to all selected
shapes, check if rotated points would exceed image boundaries, skip shapes that
violate bounds with a warning, refresh canvas, and set dirty flag.

**Shape Rotation Logic:** Calculate shape center point, iterate through vertices
computing vectors from center, apply 2D rotation matrix transformation using
cosine and sine of angle, and update point coordinates with transformed values.

**Rotation Matrix:** Use standard 2D rotation formulas with cos(θ) and sin(θ)
where positive angles rotate clockwise and negative angles rotate
counter-clockwise.

**Bounds Checking:** Pre-calculate final positions of all points after rotation,
test whether any point falls outside the image boundaries, and prevent the
entire rotation if any selected shape would go out of bounds.

---

## Feature 7: Keyboard Shortcuts

### Complete Shortcut List

#### Mode Switching

```
M              Switch to Move Mode
E              Switch to Edit Mode
```

#### Selection

```
Click box      Select single box
Ctrl+Click     Add/remove from selection
Drag empty     Draw selection rectangle
Ctrl+A         Select all boxes
Ctrl+D         Deselect all
Escape         Deselect all
```

#### Movement

```
Arrow keys     Move selected box(es) 1px
Shift+Arrow    Move selected box(es) 10px
```

#### Resize

```
Ctrl+]         Increase width +5px
Ctrl+[         Decrease width -5px
Ctrl+Shift+]   Increase height +5px
Ctrl+Shift+[   Decrease height -5px
R              Toggle resize mode (center/anchor)
```

#### Rotation

```
Ctrl+.         Apply current spinbox angle
Ctrl+,         Rotate -15° (counter-clockwise)
Ctrl+Shift+.   Rotate +15° (clockwise)
Ctrl+Alt+]     Rotate +90°
Ctrl+Alt+[     Rotate -90°
```

#### Standard Operations

```
Delete         Delete selected box(es)
Backspace      Delete selected box(es)
Ctrl+C         Copy selected box(es)
Ctrl+V         Paste copied box(es)
Ctrl+X         Cut selected box(es)
Ctrl+Z         Undo last operation
Ctrl+Y         Redo
Ctrl+S         Save annotations
```

#### Existing PPOCRLabel Shortcuts (Keep)

```
W              Create rectangle box (Edit mode)
Q              Create 4-point polygon (Edit mode)
Z/X/C/V        Select vertices 1/2/3/4 (Edit mode)
B              Deselect vertex (Edit mode)
Ctrl+E         Edit label text
D              Next image
A              Previous image
Ctrl++         Zoom in
Ctrl+-         Zoom out
Ctrl+Wheel     Zoom in/out
```

### Implementation Strategy

**Main Window Keyboard Handler:** Override keyPressEvent to capture key
combinations, check modifier state (Ctrl/Shift/Alt), dispatch to specific
handlers based on key combo, ensure no conflicts with legacy shortcuts, and
delegate unhandled keys to parent class.

**Mode Switching:** Bind M key to switch to Move Mode and E key for Edit Mode,
synchronize toolbar button visual states, update canvas interaction mode
property, and trigger display refresh.

**Selection Shortcuts:** Implement Ctrl+A to select all non-locked shapes,
Ctrl+D and Escape to clear selection, emit selectionChanged signals, and refresh
control panel enable/disable states.

**Resize Shortcuts:** Map Ctrl+] and Ctrl+[ to width adjustment, Ctrl+Shift+]
and Ctrl+Shift+[ to height adjustment by 5px increments, R key to toggle between
center and anchor resize modes, applying changes to all currently selected
boxes.

**Rotation Shortcuts:** Bind Ctrl+. to apply spinbox angle value, Ctrl+, for
-15° rotation, Ctrl+Shift+. for +15°, and Ctrl+Alt with brackets for ±90°
rotations, all applied to selected shapes.

**Canvas Keyboard Handler:** Filter keyboard events based on current mode,
process Edit Mode specific keys only when appropriate, forward Move Mode keys to
main window handler, and preserve all existing PPOCRLabel keyboard shortcuts.

---

## UI Layout

### Main Window Layout

```
┌──────────────────────────────────────────────────────────────────┐
│ Menu: File  Edit  View  Help                                     │
├───────────────────────────────────────┬──────────────────────────┤
│                                       │ Files:                   │
│                                       │ ┌──────────────────────┐ │
│                                       │ │ image1.jpg      [X]  │ │
│                                       │ │ image2.jpg      [√]  │ │
│   Canvas Area                         │ │ image3.jpg      [ ]  │ │
│   (Image with bounding boxes)         │ └──────────────────────┘ │
│                                       ├──────────────────────────┤
│                                       │ Recognition Results:     │
│                                       │ ┌──────────────────────┐ │
│                                       │ │ 1  Hello          │ │
│                                       │ │ 2  World          │ │
│                                       │ │ 3  Text           │ │
│                                       │ └──────────────────────┘ │
│                                       ├──────────────────────────┤
│                                       │ ╔══════════════════════╗ │
│                                       │ ║ RESIZE CONTROLS      ║ │
│                                       │ ║                      ║ │
│                                       │ ║ SELECTED: 2 boxes    ║ │
│                                       │ ║                      ║ │
│                                       │ ║ Width:  [◄][150][►]  ║ │
│                                       │ ║ Height: [◄][80][►]   ║ │
│                                       │ ║                      ║ │
│                                       │ ║ Resize Mode:         ║ │
│                                       │ ║ ⦿ From Center        ║ │
│                                       │ ║ ○ From Anchor        ║ │
│                                       │ ║                      ║ │
│                                       │ ║ Batch:               ║ │
│                                       │ ║ [+5][-5][+10][-10]   ║ │
│                                       │ ║                      ║ │
│                                       │ ║ Rotation             ║ │
│                                       │ ║ Angle: [0°][Apply]   ║ │
│                                       │ ║ [+15][−15][+90][−90] ║ │
│                                       │ ║                      ║ │
│                                       │ ║ [Delete Selected]    ║ │
│                                       │ ╚══════════════════════╝ │
├───────────────────────────────────────┴──────────────────────────┤
│ Toolbar: [Move Mode] [Edit Mode] │ [New] [Save] [Auto Rec] ... │
├──────────────────────────────────────────────────────────────────┤
│ Status: Image 2/50 | Selected: 2 boxes | Zoom: 100%             │
└──────────────────────────────────────────────────────────────────┘
```

### Toolbar Layout (Extended)

```
┌────────────────────────────────────────────────────────────────┐
│ [Move (M)] [Edit (E)] │ [New] [Del] [Save] [Auto] [Re-Rec] ... │
│    ↑ Mode Toggle         ↑ Existing buttons                    │
└────────────────────────────────────────────────────────────────┘
```

---

## Implementation Details

### File Structure

#### New Files (None Required)

All features integrate into existing files.

#### Modified Files

**1. libs/canvas.py** - Core canvas logic

- Add `interactionMode` (MOVE_MODE / EDIT_MODE)
- Add selection rectangle drawing
- Add edge/corner resize detection
- Add batch resize support
- Modify paint event to respect mode

**2. libs/shape.py** - Shape manipulation

- Add `getWidth()` / `getHeight()` methods
- Add `resizeFromCenter()` method
- Add `resizeFromAnchor()` method
- Enhanced `rotate()` method for any angle

**3. PPOCRLabel.py** - Main window

- Add always-visible resize control panel
- Add mode toggle buttons
- Add keyboard shortcut handling
- Connect signals for real-time updates
- Add batch operation methods

**4. resources/strings/strings-en.properties** - English UI text

Add localized string keys for all new UI elements including moveMode,
moveModeDetail, editMode, editModeDetail, resizeFromCenter, resizeFromAnchor,
batchAdjust, rotateAngle, applyRotation, and deleteSelected.

**5. resources/strings/strings-zh-CN.properties** - Chinese UI text

Add corresponding Chinese translations for all new UI string keys to maintain
full localization support for the new features.

### Signal Flow

```
User Action → Canvas Event → Signal Emission → Panel Update → Canvas Repaint

Example: User drags box edge
1. mouseMoveEvent() in Canvas detects resize
2. Apply resize to all selectedShapes
3. Emit shapeDimensionsChanged signal
4. Panel receives signal, updates spinboxes
5. Canvas.update() repaints

Example: User changes width spinbox
1. valueChanged signal emitted
2. onWidthSpinChanged() in MainWindow
3. Calculate delta for each shape
4. Apply resize to all selectedShapes
5. Canvas.update() repaints
6. Update spinbox (if needed)
```

### Real-Time Update Strategy

**Block Signals Pattern:**

Implement updateResizeControlValues method that temporarily blocks signals on
spinbox widgets, updates their displayed values programmatically, then
re-enables signals to prevent infinite recursion loops during synchronized
updates.

**Debouncing (if needed):**

For computationally expensive operations, create a single-shot QTimer that
delays execution by 100ms, restarting the timer on each value change to batch
rapid updates and only execute when user input stabilizes.

### State Management

**Canvas State Variables:**

- Current interaction mode (MOVE_MODE or EDIT_MODE)
- List of selected shapes
- Selection rectangle (if drawing)
- Resize handle being dragged (if resizing)
- Resize start point and original dimensions
- Undo/redo history stacks

**MainWindow State Variables:**

- Current mode ('move' or 'edit')
- Current width/height values from spinboxes
- Current angle value
- Clipboard for copy/paste operations
- Dirty flag for unsaved changes

---

## Testing Checklist

### Feature 1: Move Mode vs Edit Mode

**Move Mode:**

- [ ] Press M key switches to Move Mode
- [ ] Toolbar button reflects active mode
- [ ] Selected boxes show NO vertex points
- [ ] Cannot click on individual vertices
- [ ] Can drag box center to move
- [ ] Can drag edges to resize
- [ ] Resize cursors appear on hover
- [ ] Cannot create new boxes

**Edit Mode:**

- [ ] Press E key switches to Edit Mode
- [ ] Toolbar button reflects active mode
- [ ] Selected boxes show vertex points
- [ ] Can click vertices to select them
- [ ] Z/X/C/V keys select vertices
- [ ] Can drag vertices to move them
- [ ] W key creates new box
- [ ] Q key creates new polygon

### Feature 2: Mouse Drag Selection

**Basic Selection:**

- [ ] Click+drag in empty area starts selection
- [ ] Dashed rectangle appears
- [ ] Boxes in rectangle highlight
- [ ] Release finalizes selection
- [ ] Selection clears previous selection

**Modifier Keys:**

- [ ] Ctrl+drag adds to selection
- [ ] Alt+drag removes from selection
- [ ] Shift+drag selects intersecting boxes
- [ ] Works in both Move and Edit modes

**Visual Feedback:**

- [ ] Rectangle shows count near cursor
- [ ] Selected boxes highlight correctly
- [ ] Rectangle clears on release

### Feature 3: Drag Resize

**Edge Detection:**

- [ ] Hover over top edge shows ↕ cursor
- [ ] Hover over bottom edge shows ↕ cursor
- [ ] Hover over left edge shows ↔ cursor
- [ ] Hover over right edge shows ↔ cursor
- [ ] Hover over corners shows diagonal cursors

**Resize From Center:**

- [ ] Drag right edge → both sides expand equally
- [ ] Drag left edge → both sides expand equally
- [ ] Drag top edge → both sides expand equally
- [ ] Drag bottom edge → both sides expand equally
- [ ] Drag corner → all sides expand equally
- [ ] Center point DOES NOT move

**Resize From Anchor:**

- [ ] Switch to anchor mode
- [ ] Drag right edge → only right side moves
- [ ] Drag left edge → only left side moves
- [ ] Opposite edges stay fixed

**Batch Resize:**

- [ ] Select 3 boxes
- [ ] Drag edge of one box
- [ ] ALL 3 boxes resize simultaneously
- [ ] Proportions maintained

### Feature 4: Resize Control Panel

**Panel Visibility:**

- [ ] Panel always visible (never hidden)
- [ ] Panel in right dock, bottom section
- [ ] Panel below recognition results

**Panel States:**

- [ ] No selection: Shows "SELECTED: 0 boxes"
- [ ] No selection: All controls disabled/grayed
- [ ] 1 selection: Shows "SELECTED: 1 box"
- [ ] 1 selection: All controls enabled
- [ ] 1 selection: Shows exact dimensions
- [ ] 3 selections: Shows "SELECTED: 3 boxes"
- [ ] 3 selections: Shows average dimensions

**Real-Time Updates:**

- [ ] Drag box edge → spinboxes update
- [ ] Change spinbox → boxes resize instantly
- [ ] Click +5 → boxes grow immediately
- [ ] No delay or lag in updates

### Feature 5: Batch Operations

**Width/Height Controls:**

- [ ] Change width spinbox updates all selected
- [ ] Change height spinbox updates all selected
- [ ] Click ◄ decreases by 1px
- [ ] Click ► increases by 1px
- [ ] Works on single box
- [ ] Works on multiple boxes

**Batch Adjust:**

- [ ] Click +5 increases both dimensions
- [ ] Click -5 decreases both dimensions
- [ ] Click +10 increases both dimensions
- [ ] Click -10 decreases both dimensions
- [ ] Applies to ALL selected boxes
- [ ] Updates spinboxes after action

### Feature 6: Rotation

**Angle Input:**

- [ ] Can enter any value 0-360
- [ ] Can enter negative values
- [ ] Can enter decimals (45.5)
- [ ] Apply button rotates boxes
- [ ] Works on single box
- [ ] Works on multiple boxes

**Quick Buttons:**

- [ ] +15° rotates clockwise
- [ ] −15° rotates counter-clockwise
- [ ] +90° rotates 90° clockwise
- [ ] −90° rotates 90° counter-clockwise
- [ ] Applies to ALL selected boxes

**Bounds Checking:**

- [ ] Prevents rotation out of image bounds
- [ ] Shows warning if can't rotate
- [ ] Skips boxes that would go out of bounds

### Feature 7: Keyboard Shortcuts

**Mode Switching:**

- [ ] M switches to Move Mode
- [ ] E switches to Edit Mode

**Selection:**

- [ ] Ctrl+A selects all boxes
- [ ] Ctrl+D deselects all
- [ ] Escape deselects all

**Resize:**

- [ ] Ctrl+] increases width
- [ ] Ctrl+[ decreases width
- [ ] Ctrl+Shift+] increases height
- [ ] Ctrl+Shift+[ decreases height
- [ ] R toggles resize mode

**Rotation:**

- [ ] Ctrl+. applies spinbox angle
- [ ] Ctrl+, rotates -15°
- [ ] Ctrl+Shift+. rotates +15°
- [ ] Ctrl+Alt+] rotates +90°
- [ ] Ctrl+Alt+[ rotates -90°

**Standard:**

- [ ] Delete removes selected boxes
- [ ] Backspace removes selected boxes
- [ ] Ctrl+C copies
- [ ] Ctrl+V pastes
- [ ] Ctrl+Z undos
- [ ] Arrow keys move boxes

### Integration Tests

**Workflow 1: Quick Resize Multiple Boxes**

- [ ] Drag selection rectangle over 5 boxes
- [ ] All 5 boxes selected
- [ ] Panel shows "SELECTED: 5 boxes"
- [ ] Click +10 button
- [ ] All 5 boxes grow by 10px
- [ ] Spinboxes update
- [ ] Press Ctrl+Z
- [ ] All 5 boxes revert to original size

**Workflow 2: Precise Rotation**

- [ ] Click box to select
- [ ] Enter 37.4 in rotation spinbox
- [ ] Click Apply
- [ ] Box rotates 37.4°
- [ ] Box stays within bounds
- [ ] Ctrl+Z undos rotation

**Workflow 3: Batch Edit From Drag**

- [ ] Select 3 boxes (Ctrl+Click)
- [ ] Switch to Move Mode (M)
- [ ] Drag right edge of first box
- [ ] All 3 boxes resize together
- [ ] Panel updates dimensions
- [ ] Release mouse
- [ ] Changes saved to undo stack

**Workflow 4: Keyboard-Only Workflow**

- [ ] Ctrl+A selects all
- [ ] Ctrl+] x3 increases width 15px
- [ ] Ctrl+Shift+] x2 increases height 10px
- [ ] Ctrl+. applies custom rotation
- [ ] Delete removes all boxes
- [ ] Ctrl+Z undos delete

### Performance Tests

- [ ] Selecting 100+ boxes is responsive
- [ ] Dragging with 50+ boxes selected is smooth
- [ ] Real-time spinbox updates don't lag
- [ ] Selection rectangle draws without flicker
- [ ] Undo/redo works with large selections

### Edge Cases

- [ ] Resize doesn't allow boxes < 10px
- [ ] Rotate doesn't allow out of bounds
- [ ] Can't resize locked boxes
- [ ] Selection rectangle works at image edges
- [ ] Batch operations skip locked boxes
- [ ] Panel handles shapes with 0 dimensions
- [ ] Keyboard shortcuts don't conflict

---

## Development Timeline

### Phase 1: Foundation (Day 1-2)

- [ ] Add mode toggle buttons to toolbar
- [ ] Implement MOVE_MODE / EDIT_MODE in Canvas
- [ ] Modify paint event to respect modes
- [ ] Test mode switching

### Phase 2: Selection Rectangle (Day 2)

- [ ] Add selection rectangle drawing
- [ ] Implement shape detection in rect
- [ ] Add modifier key support
- [ ] Test multi-selection

### Phase 4: Drag Resize (Day 3-4)

- [ ] Add edge/corner detection
- [ ] Implement cursor changes
- [ ] Add resizeFromCenter() in Shape
- [ ] Add resizeFromAnchor() in Shape
- [ ] Test single box resize
- [ ] Test batch resize

### Phase 5: Control Panel (Day 4-5)

- [ ] Create panel UI
- [ ] Add all spinboxes and buttons
- [ ] Connect signals
- [ ] Implement real-time updates
- [ ] Test enable/disable logic
- [ ] Test with selections

### Phase 6: Rotation (Day 5-6)

- [ ] Add rotation controls to panel
- [ ] Implement any-angle rotation
- [ ] Add bounds checking
- [ ] Test rotation on multiple boxes

### Phase 7: Keyboard Shortcuts (Day 6)

- [ ] Implement all keyboard shortcuts
- [ ] Test shortcut conflicts
- [ ] Update documentation

### Phase 8: Polish & Testing (Day 6-7)

- [ ] Complete all test checklist items
- [ ] Fix bugs
- [ ] Optimize performance
- [ ] Update user documentation

---

## Notes & Considerations

### Design Decisions

1. **Why resize from center by default?**
   - Prevents boxes from "wandering" during resize
   - More intuitive for batch operations
   - Easier to maintain alignment

2. **Why always-visible panel?**
   - Faster workflow (no clicking to open dialogs)
   - Real-time feedback
   - Desktop-class tool experience

3. **Why separate Move/Edit modes?**
   - Prevents accidental vertex edits
   - Matches Label Studio UX
   - Clear mental model for users

4. **Why any-angle rotation?**
   - Document pages often skewed at odd angles
   - More flexibility than 90° increments
   - Professional feature request

### Future Enhancements

**Low Priority:**

- Visual alignment guides (snap to grid)
- Distribute boxes evenly
- Align boxes (left/right/top/bottom)
- Copy formatting between boxes
- Batch label editing
- Export/import resize presets
- Keyboard shortcut customization

**Dependencies:**

- PyQt5 >= 5.12
- numpy >= 1.19 (existing)
- opencv-python >= 4.5 (existing)

### Known Limitations

1. **Rotation with skewed boxes**: Complex polygons may not rotate perfectly
2. **Large batch operations**: 100+ boxes may have slight delay
3. **Touch input**: Panel designed for mouse/keyboard, touch may be awkward
4. **Undo/Redo**: Need to ensure all operations in undo stack

---

## Conclusion

This implementation plan provides a complete Label Studio-inspired workflow for
PPOCRLabel with:

✅ Fast Move Mode for common operations ✅ Batch selection via mouse drag ✅
Drag resize with center-based logic ✅ Always-visible real-time controls ✅
Any-angle rotation support ✅ Complete keyboard accessibility

**Estimated completion:** 6-7 days of focused development

**Ready for implementation!**

---

**Document Version:** 1.0 **Last Updated:** February 27, 2026 **Status:**
Final - Ready for Development
