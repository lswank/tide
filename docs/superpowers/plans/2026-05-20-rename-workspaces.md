# Rename Workspaces Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let Users rename Workspaces from the Workspace rail (right-click → Rename, inline edit) and let Wrapped Agents rename them through `tide_rename_workspace`.

**Architecture:** Add a single `WorkspaceNavPort::rename_workspace(idx, name)` method as the choke point. Generalize `ContextMenuState` to a tagged `ContextMenuTarget` so the Workspace rail shares the existing context-menu chrome with the FileTree View. Introduce a `WorkspaceRenameState` modal that mirrors the existing `FileTreeRenameState` pattern (modal field + keyboard handler + text routing + inline render). Register the MCP tool by extending the existing tool list → method map → `cli_` dispatch pipeline.

**Tech Stack:** Rust 2021, hexagonal architecture (ports + adapters), wgpu-rendered native macOS app. Test framework: `cargo test`, behavior tests live in `crates/tide-app/src/application/behavior_tests/`.

**Spec:** `docs/specs/rename-workspaces.md`

---

## Pre-flight

Run once before starting Task 1 to confirm the baseline tests pass on this branch:

```bash
cd ~/Development/github.com/lswank/tide-rename-workspaces
cargo test -p tide-app --lib --no-fail-fast 2>&1 | tail -20
```

Expected: a green run (or the exact same failures as `upstream/main` — record the baseline failure count if any).

---

## Task 1: Port + service `rename_workspace`

Add the core mutation behind the port. No UI yet.

**Files:**
- Modify: `crates/tide-app/src/application/ports/inward/workspace_nav_port/mod.rs`
- Modify: `crates/tide-app/src/application/services/workspace_service/mod.rs`
- Test: `crates/tide-app/src/application/behavior_tests/workspace_behavior.rs`

- [ ] **Step 1: Write the failing test**

Append to `crates/tide-app/src/application/behavior_tests/workspace_behavior.rs` (after the existing tests):

```rust
// --- UC-1: RenameWorkspace ---

// UC-1 BR-1: rename_workspace updates the active Workspace name
#[test]
fn rename_workspace_updates_active_workspace_name() {
    use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;
    let mut app = app_with_two_workspaces();
    let active = app.ws.active;

    app.rename_workspace(active, "Renamed Active".to_string());

    assert_eq!(app.ws.workspaces[active].name, "Renamed Active");
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cargo test -p tide-app --lib rename_workspace_updates_active_workspace_name 2>&1 | tail -15
```

Expected: compile error — `WorkspaceNavPort` has no `rename_workspace` method.

- [ ] **Step 3: Add the port method signature**

Edit `crates/tide-app/src/application/ports/inward/workspace_nav_port/mod.rs`. In the `WorkspaceNavPort` trait, after the `ws_reorder` line, add:

```rust
    fn rename_workspace(&mut self, idx: usize, name: String);
```

- [ ] **Step 4: Implement in workspace_service**

Edit `crates/tide-app/src/application/services/workspace_service/mod.rs`. In the `impl WorkspaceNavPort for App` block (just before the `workspace_sidebar_item_rect` method at line 825), add:

```rust
    fn rename_workspace(&mut self, idx: usize, name: String) {
        let trimmed = name.trim();
        if trimmed.is_empty() {
            return;
        }
        if idx >= self.ws.workspaces.len() {
            return;
        }
        self.ws.workspaces[idx].name = trimmed.to_string();
        self.cache.invalidate_chrome();
    }
```

- [ ] **Step 5: Run test to verify it passes**

```bash
cargo test -p tide-app --lib rename_workspace_updates_active_workspace_name 2>&1 | tail -10
```

Expected: 1 passed.

- [ ] **Step 6: Commit**

```bash
git add crates/tide-app/src/application/ports/inward/workspace_nav_port/mod.rs \
        crates/tide-app/src/application/services/workspace_service/mod.rs \
        crates/tide-app/src/application/behavior_tests/workspace_behavior.rs
git commit -m "Add WorkspaceNavPort::rename_workspace in tide-app

UC-1 BR-1: rename the active Workspace through the port."
```

---

## Task 2: rename_workspace edge cases

Cover cold-stored rename, empty/whitespace rejection, out-of-bounds, and persistence across switch.

**Files:**
- Test: `crates/tide-app/src/application/behavior_tests/workspace_behavior.rs`

- [ ] **Step 1: Write the failing tests**

Append below the previous test:

```rust
// UC-1 BR-2: rename_workspace updates a cold-stored (inactive) Workspace
#[test]
fn rename_workspace_updates_cold_stored_workspace_name() {
    use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;
    let mut app = app_with_two_workspaces();
    let inactive = 1 - app.ws.active;

    app.rename_workspace(inactive, "Renamed Cold".to_string());

    assert_eq!(app.ws.workspaces[inactive].name, "Renamed Cold");
}

// UC-1 BR-3: empty / whitespace-only names are rejected
#[test]
fn rename_workspace_with_empty_name_is_a_no_op() {
    use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;
    let mut app = app_with_two_workspaces();
    let active = app.ws.active;
    let original = app.ws.workspaces[active].name.clone();

    app.rename_workspace(active, "".to_string());
    app.rename_workspace(active, "   ".to_string());

    assert_eq!(app.ws.workspaces[active].name, original);
}

// UC-1 BR-4: rename survives a switch out and back
#[test]
fn rename_workspace_persists_across_switch_out_and_back() {
    use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;
    let mut app = app_with_two_workspaces();
    let active = app.ws.active;
    let other = 1 - active;

    app.rename_workspace(active, "Persisted".to_string());
    app.switch_workspace(other);
    app.switch_workspace(active);

    assert_eq!(app.ws.workspaces[active].name, "Persisted");
}

// UC-1 BR-5: out-of-bounds index is a no-op
#[test]
fn rename_workspace_with_out_of_bounds_index_is_a_no_op() {
    use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;
    let mut app = app_with_two_workspaces();
    let len = app.ws.workspaces.len();
    let before: Vec<String> = app.ws.workspaces.iter().map(|w| w.name.clone()).collect();

    app.rename_workspace(len, "Nope".to_string());
    app.rename_workspace(len + 5, "Also nope".to_string());

    let after: Vec<String> = app.ws.workspaces.iter().map(|w| w.name.clone()).collect();
    assert_eq!(before, after);
}
```

- [ ] **Step 2: Run tests to verify they pass**

```bash
cargo test -p tide-app --lib rename_workspace_ 2>&1 | tail -15
```

Expected: 5 passed (including BR-1 from Task 1).

- [ ] **Step 3: Commit**

```bash
git add crates/tide-app/src/application/behavior_tests/workspace_behavior.rs
git commit -m "Add Workspace rename edge-case behavior tests in tide-app

UC-1 BR-2..BR-5: cold-stored rename, empty no-op, persistence, out-of-bounds."
```

---

## Task 3: Introduce `ContextMenuTarget::FileTreeEntry` (refactor)

Wrap the existing file-tree-specific fields of `ContextMenuState` in a target enum. **No behavior change** — this is a structural refactor that all existing call sites must compile through.

**Files:**
- Modify: `crates/tide-app/src/domain/modal/mod.rs`
- Modify: `crates/tide-app/src/application/services/file_tree_service/mod.rs`
- Modify: `crates/tide-app/src/adapter/inward/mouse_adapter/mod.rs`
- Modify: `crates/tide-app/src/adapter/inward/text_routing_adapter/mod.rs`
- Modify: `crates/tide-app/src/adapter/outward/view/overlays/context_menu.rs` (only if it reads the file-tree fields)

- [ ] **Step 1: Refactor the `ContextMenuState` shape**

Edit `crates/tide-app/src/domain/modal/mod.rs`. Replace the existing `ContextMenuState` struct (around line 963) with:

```rust
#[derive(Debug, Clone)]
pub(crate) enum ContextMenuTarget {
    FileTreeEntry {
        entry_index: usize,
        path: PathBuf,
        is_dir: bool,
        is_app_bundle: bool,
        shell_idle: bool,
    },
}

pub(crate) struct ContextMenuState {
    pub target: ContextMenuTarget,
    pub position: Vec2,
    pub selected: usize,
}

impl ContextMenuState {
    pub fn items(&self) -> &'static [ContextMenuAction] {
        match &self.target {
            ContextMenuTarget::FileTreeEntry {
                is_dir,
                is_app_bundle,
                shell_idle,
                ..
            } => ContextMenuAction::items(*is_dir, *is_app_bundle, *shell_idle),
        }
    }

    /// Compute the popup rect, clamped to window bounds.
    pub fn geometry(&self, cell_height: f32, logical_width: f32, logical_height: f32) -> Rect {
        // ... existing geometry body unchanged
```

Keep the body of `geometry(...)` exactly as it is today. Only the struct shape and the `items()` body change.

- [ ] **Step 2: Update the file-tree right-click site**

Edit `crates/tide-app/src/adapter/inward/mouse_adapter/mod.rs` around line 263. Replace the existing `ContextMenuState { entry_index, path, is_dir, is_app_bundle, shell_idle, position, selected }` literal with:

```rust
                        ctx.modal_mut().context_menu = Some(crate::ContextMenuState {
                            target: crate::ContextMenuTarget::FileTreeEntry {
                                entry_index: index,
                                path,
                                is_dir,
                                is_app_bundle,
                                shell_idle,
                            },
                            position: pos,
                            selected: 0,
                        });
```

- [ ] **Step 3: Update the file-tree-service execute handler**

Edit `crates/tide-app/src/application/services/file_tree_service/mod.rs` around line 573–644 (`execute_context_menu_action`). Wrap the existing logic so it destructures the target:

```rust
    pub(crate) fn execute_context_menu_action(&mut self, action_index: usize) {
        let menu = match self.modal.context_menu.take() {
            Some(m) => m,
            None => return,
        };

        let (entry_index, path, is_dir, is_app_bundle, shell_idle) = match menu.target {
            crate::ContextMenuTarget::FileTreeEntry {
                entry_index,
                path,
                is_dir,
                is_app_bundle,
                shell_idle,
            } => (entry_index, path, is_dir, is_app_bundle, shell_idle),
            // future targets dispatched elsewhere
        };

        let items = crate::ContextMenuAction::items(is_dir, is_app_bundle, shell_idle);
        let action = match items.get(action_index) {
            Some(a) => *a,
            None => return,
        };

        match action {
            crate::ContextMenuAction::CdHere => {
                let path_str = path.to_string_lossy();
                let cmd = format!("cd {}\n", crate::shell_escape(&path_str));
                if let Some(tid) = self.focus.focused {
                    if let Some(crate::PaneKind::Terminal(pane)) = self.panes.get_mut(&tid) {
                        pane.backend.write(cmd.as_bytes());
                    }
                }
            }
            crate::ContextMenuAction::OpenTerminalHere => {
                self.split_pane(crate::tide_core::SplitDirection::Vertical, Some(path));
            }
            crate::ContextMenuAction::OpenApp => {
                let result = self.ports.process.open_with_default_app(&path);
                Self::log_app_handoff_failure(result, &path);
            }
            crate::ContextMenuAction::Delete => {
                let result = if is_dir {
                    self.ports.fs.remove_dir_all(&path)
                } else {
                    self.ports.fs.remove_file(&path)
                };
                if let Err(e) = result {
                    log::error!("Failed to delete {:?}: {}", path, e);
                }
                if let Some(tree) = self.ft.tree.as_mut() {
                    tree.refresh();
                }
                self.sync_file_tree_path_identity_cache();
                self.trigger_git_poll();
                self.cache.invalidate_chrome();
            }
            crate::ContextMenuAction::RevealInFinder => {
                if Self::is_app_bundle_directory(&path, is_dir) {
                    let _ = self.ports.process.reveal_in_finder(&path);
                } else if is_dir {
                    let _ = self.ports.process.open_with_default_app(&path);
                } else {
                    let _ = self.ports.process.reveal_in_finder(&path);
                }
            }
            crate::ContextMenuAction::Rename => {
                let file_name = path
                    .file_name()
                    .map(|n| n.to_string_lossy().to_string())
                    .unwrap_or_default();
                self.modal.file_tree_rename = Some(crate::FileTreeRenameState {
                    entry_index,
                    original_path: path,
                    input: crate::InputLine::with_text(file_name),
                });
                self.cache.invalidate_chrome();
            }
        }
        self.cache.needs_redraw = true;
    }
```

- [ ] **Step 4: Update the text_routing_adapter test fixture**

Edit `crates/tide-app/src/adapter/inward/text_routing_adapter/mod.rs` around line 481 — wherever a test constructs `ContextMenuState { … }` directly, switch to the new shape:

```rust
        app.modal.context_menu = Some(ContextMenuState {
            target: crate::ContextMenuTarget::FileTreeEntry {
                entry_index: 0,
                path: PathBuf::from("/tmp/x"),
                is_dir: false,
                is_app_bundle: false,
                shell_idle: true,
            },
            position: Vec2::new(0.0, 0.0),
            selected: 0,
        });
```

(If imports of `PathBuf` / `Vec2` / `ContextMenuState` are not already present at the top of the test module, add them.)

- [ ] **Step 5: Re-export `ContextMenuTarget` from the crate root**

Edit `crates/tide-app/src/main.rs` (or wherever `ContextMenuState`, `ContextMenuAction`, `FileTreeRenameState` are re-exported — search for `ContextMenuState`):

```bash
grep -n "ContextMenuState\|ContextMenuAction" crates/tide-app/src/main.rs | head -5
```

Add `ContextMenuTarget` to the same re-export. If the line looks like:

```rust
pub(crate) use crate::domain::modal::{ContextMenuAction, ContextMenuState, FileTreeRenameState, ...};
```

change to:

```rust
pub(crate) use crate::domain::modal::{ContextMenuAction, ContextMenuState, ContextMenuTarget, FileTreeRenameState, ...};
```

- [ ] **Step 6: Build and run the existing context-menu tests**

```bash
cargo build -p tide-app 2>&1 | tail -20
cargo test -p tide-app --lib context_menu 2>&1 | tail -15
cargo test -p tide-app --lib file_tree 2>&1 | tail -15
```

Expected: build green; all existing context-menu and file-tree tests still pass.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "Extract ContextMenuTarget from ContextMenuState in tide-app

Refactor: wrap file-tree fields in ContextMenuTarget::FileTreeEntry so
the Workspace rail can share the same menu chrome (no behavior change)."
```

---

## Task 4: Add `ContextMenuTarget::WorkspaceSidebarItem` + workspace items list

Extend the target enum and teach `ContextMenuAction::items()` to return the workspace menu (just `Rename` for now).

**Files:**
- Modify: `crates/tide-app/src/domain/modal/mod.rs`

- [ ] **Step 1: Add the variant + the workspace items list**

Edit `crates/tide-app/src/domain/modal/mod.rs`. In the `ContextMenuTarget` enum, add a variant:

```rust
    WorkspaceSidebarItem {
        ws_index: usize,
    },
```

In the `impl ContextMenuAction` block (near `FILE_ACTIONS`, `DIR_ACTIONS`, etc.), add:

```rust
    const WORKSPACE_ACTIONS: [ContextMenuAction; 1] = [ContextMenuAction::Rename];
```

Add a helper for workspace items (sibling to the existing `items` fn):

```rust
    pub fn workspace_items() -> &'static [ContextMenuAction] {
        &Self::WORKSPACE_ACTIONS
    }
```

Update `ContextMenuState::items()`:

```rust
    pub fn items(&self) -> &'static [ContextMenuAction] {
        match &self.target {
            ContextMenuTarget::FileTreeEntry {
                is_dir,
                is_app_bundle,
                shell_idle,
                ..
            } => ContextMenuAction::items(*is_dir, *is_app_bundle, *shell_idle),
            ContextMenuTarget::WorkspaceSidebarItem { .. } => {
                ContextMenuAction::workspace_items()
            }
        }
    }
```

- [ ] **Step 2: Build**

```bash
cargo build -p tide-app 2>&1 | tail -10
```

Expected: green.

- [ ] **Step 3: Commit**

```bash
git add crates/tide-app/src/domain/modal/mod.rs
git commit -m "Add WorkspaceSidebarItem ContextMenu target in tide-app

UC-2: the Workspace rail menu offers Rename. Items computed via
ContextMenuAction::workspace_items()."
```

---

## Task 5: Right-click on Workspace rail item opens the ContextMenu

**Files:**
- Modify: `crates/tide-app/src/adapter/inward/mouse_adapter/mod.rs`
- Test: `crates/tide-app/src/application/behavior_tests/workspace_behavior.rs`

- [ ] **Step 1: Write the failing test**

Append to `workspace_behavior.rs`:

```rust
// UC-2 BR-1: right-click on a Workspace rail item opens a ContextMenu with Rename
#[test]
fn right_click_on_workspace_sidebar_item_opens_context_menu_with_rename() {
    use crate::adapter::inward::mouse_adapter;
    use crate::tide_core::Vec2;
    use winit::event::MouseButton;

    let mut app = app_with_two_workspaces();
    // Force layout so workspace_sidebar_item_rect resolves
    app.compute_layout();

    // Position cursor over workspace 0's item
    let rect = {
        use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;
        app.workspace_sidebar_item_rect(0)
    }
    .expect("workspace item rect");
    app.interaction.last_cursor_pos = Vec2::new(rect.x + rect.width / 2.0, rect.y + rect.height / 2.0);

    let proxy = crate::adapter::outward::platform_native::test_window_proxy();
    mouse_adapter::handle_mouse_down(&mut app, MouseButton::Right, &proxy);

    let menu = app
        .modal
        .context_menu
        .as_ref()
        .expect("context menu open");
    match &menu.target {
        crate::ContextMenuTarget::WorkspaceSidebarItem { ws_index } => assert_eq!(*ws_index, 0),
        other => panic!("expected WorkspaceSidebarItem, got {:?}", other),
    }
    let labels: Vec<&'static str> = menu.items().iter().map(|a| a.label()).collect();
    assert_eq!(labels, vec!["Rename"]);
}
```

> **Note for the implementing engineer:** `test_window_proxy()` may not exist. Grep for how other mouse-adapter tests construct a `WindowProxy`:
>
> ```bash
> grep -rn "handle_mouse_down\|WindowProxy\|test_window_proxy" crates/tide-app/src/application/behavior_tests/ | head -5
> grep -rn "WindowProxy::new\|fn .*WindowProxy" crates/tide-app/src/ | head -5
> ```
>
> Use whatever helper already exists. If none exists, use the simplest stub — `WindowProxy` is `_window: &WindowProxy` in `handle_mouse_input_core` and is currently unused for non-titlebar paths.

- [ ] **Step 2: Run the test to verify it fails**

```bash
cargo test -p tide-app --lib right_click_on_workspace_sidebar_item_opens_context_menu_with_rename 2>&1 | tail -15
```

Expected: FAIL — `context_menu` is `None`.

- [ ] **Step 3: Add the right-click branch in mouse_adapter**

Edit `crates/tide-app/src/adapter/inward/mouse_adapter/mod.rs`. In `handle_mouse_input_core`, before the existing `if button == MouseButton::Right { if ctx.ft().visible { ... } }` branch (around line 221), add a Workspace rail branch:

```rust
    // Right-click on Workspace rail item
    if button == MouseButton::Right {
        let pos = ctx.last_cursor_pos();
        let new_hover =
            crate::adapter::inward::click_adapter::hit_test::compute_hover_target(ctx, pos);
        if let Some(crate::state::drag_types::HoverTarget::WorkspaceSidebarItem(idx)) = new_hover {
            ctx.modal_mut().context_menu = None;
            ctx.modal_mut().file_tree_rename = None;
            ctx.modal_mut().context_menu = Some(crate::ContextMenuState {
                target: crate::ContextMenuTarget::WorkspaceSidebarItem { ws_index: idx },
                position: pos,
                selected: 0,
            });
            ctx.request_redraw();
            return;
        }
    }
```

`ctx.request_redraw()` must already exist as a `MousePorts` method — verify with `grep -n "fn request_redraw" crates/tide-app/src/adapter/inward/mouse_adapter/mod.rs`. If not, the surrounding code uses `ctx.request_redraw()` already (it's called multiple times in this file), so the trait already provides it.

- [ ] **Step 4: Run the test to verify it passes**

```bash
cargo test -p tide-app --lib right_click_on_workspace_sidebar_item_opens_context_menu_with_rename 2>&1 | tail -10
```

Expected: 1 passed.

- [ ] **Step 5: Commit**

```bash
git add crates/tide-app/src/adapter/inward/mouse_adapter/mod.rs \
        crates/tide-app/src/application/behavior_tests/workspace_behavior.rs
git commit -m "Add Workspace rail right-click ContextMenu in tide-app

UC-2 BR-1: right-click on a Workspace rail item opens the ContextMenu
with target ContextMenuTarget::WorkspaceSidebarItem."
```

---

## Task 6: `WorkspaceRenameState` modal + opening it from the menu

Add the modal struct, wire it into `ModalStack`, and make `execute_context_menu_action` open it when the workspace menu's Rename action fires.

**Files:**
- Modify: `crates/tide-app/src/domain/modal/mod.rs`
- Modify: `crates/tide-app/src/application/services/file_tree_service/mod.rs` (the only `execute_context_menu_action` lives there; we split out workspace dispatch into a separate fn on the workspace service)
- Modify: `crates/tide-app/src/application/services/workspace_service/mod.rs`
- Modify: `crates/tide-app/src/main.rs` (re-export `WorkspaceRenameState`)
- Test: `crates/tide-app/src/application/behavior_tests/workspace_behavior.rs`

- [ ] **Step 1: Add `WorkspaceRenameState`**

Edit `crates/tide-app/src/domain/modal/mod.rs`. Below the `FileTreeRenameState` struct (around line 1131), add:

```rust
// ──────────────────────────────────────────────
// Workspace rail inline rename state
// ──────────────────────────────────────────────

pub(crate) struct WorkspaceRenameState {
    pub ws_index: usize,
    pub input: InputLine,
}
```

`InputLine` is already imported at the top of this file (`use super::state::input_line::{abbreviate_path, expand_tilde, InputLine};`).

In `ModalStack`, add the field (between `file_tree_rename` and `branch_cleanup`):

```rust
    pub workspace_rename: Option<WorkspaceRenameState>,
```

Initialize it in `ModalStack::new()`:

```rust
            workspace_rename: None,
```

Add to `is_any_open()`:

```rust
            || self.workspace_rename.is_some()
```

Add to `close_all()`:

```rust
        self.workspace_rename = None;
```

- [ ] **Step 2: Re-export `WorkspaceRenameState`**

Edit `crates/tide-app/src/main.rs` (the same re-export edited in Task 3 Step 5). Add `WorkspaceRenameState` to the list alongside `FileTreeRenameState`.

- [ ] **Step 3: Write the failing tests**

Append to `workspace_behavior.rs`:

```rust
// UC-2 BR-2: selecting Rename opens workspace_rename seeded with current name
#[test]
fn selecting_rename_in_workspace_context_menu_opens_workspace_rename_modal() {
    use crate::ContextMenuTarget;
    let mut app = app_with_two_workspaces();
    app.modal.context_menu = Some(crate::ContextMenuState {
        target: ContextMenuTarget::WorkspaceSidebarItem { ws_index: 0 },
        position: crate::tide_core::Vec2::new(0.0, 0.0),
        selected: 0,
    });
    let original = app.ws.workspaces[0].name.clone();

    app.execute_context_menu_action(0);

    let modal = app
        .modal
        .workspace_rename
        .as_ref()
        .expect("workspace_rename modal open");
    assert_eq!(modal.ws_index, 0);
    assert_eq!(modal.input.text, original);
    assert!(app.modal.context_menu.is_none(), "context menu consumed");
}

// UC-2 BR-7: ModalStack::is_any_open() and close_all() include workspace_rename
#[test]
fn workspace_rename_participates_in_modal_is_any_open_and_close_all() {
    let mut app = app_with_two_workspaces();
    assert!(!app.modal.is_any_open());

    app.modal.workspace_rename = Some(crate::WorkspaceRenameState {
        ws_index: 0,
        input: crate::InputLine::with_text("X".to_string()),
    });
    assert!(app.modal.is_any_open());

    app.modal.close_all();
    assert!(app.modal.workspace_rename.is_none());
}
```

- [ ] **Step 4: Run the tests to verify failure**

```bash
cargo test -p tide-app --lib selecting_rename_in_workspace_context_menu 2>&1 | tail -10
cargo test -p tide-app --lib workspace_rename_participates_in_modal 2>&1 | tail -10
```

Expected: first FAILS (no dispatch yet), second PASSES (Step 1 already wired it).

- [ ] **Step 5: Add workspace dispatch on `execute_context_menu_action`**

Edit `crates/tide-app/src/application/services/file_tree_service/mod.rs:573`. Change the early extraction so workspace targets short-circuit to a workspace service handler:

```rust
    pub(crate) fn execute_context_menu_action(&mut self, action_index: usize) {
        let menu = match self.modal.context_menu.take() {
            Some(m) => m,
            None => return,
        };

        match menu.target {
            crate::ContextMenuTarget::FileTreeEntry {
                entry_index,
                path,
                is_dir,
                is_app_bundle,
                shell_idle,
            } => self.execute_file_tree_context_menu_action(
                action_index, entry_index, path, is_dir, is_app_bundle, shell_idle,
            ),
            crate::ContextMenuTarget::WorkspaceSidebarItem { ws_index } => {
                self.execute_workspace_context_menu_action(action_index, ws_index)
            }
        }
    }

    fn execute_file_tree_context_menu_action(
        &mut self,
        action_index: usize,
        entry_index: usize,
        path: PathBuf,
        is_dir: bool,
        is_app_bundle: bool,
        shell_idle: bool,
    ) {
        let items = crate::ContextMenuAction::items(is_dir, is_app_bundle, shell_idle);
        let action = match items.get(action_index) {
            Some(a) => *a,
            None => return,
        };
        // ... existing match action { ... } body, using local `entry_index`, `path`, etc.
        // Existing body verbatim — paste the match expression from the old function.
        self.cache.needs_redraw = true;
    }
```

(The existing match body is unchanged — just move it under the new method. Local bindings `entry_index`, `path`, `is_dir`, `is_app_bundle`, `shell_idle` are now parameters, so the body compiles.)

- [ ] **Step 6: Implement `execute_workspace_context_menu_action` on workspace_service**

Edit `crates/tide-app/src/application/services/workspace_service/mod.rs`. Outside the `impl WorkspaceNavPort for App` block, in a free `impl App` block (or extend an existing one), add:

```rust
impl App {
    pub(crate) fn execute_workspace_context_menu_action(
        &mut self,
        action_index: usize,
        ws_index: usize,
    ) {
        let items = crate::ContextMenuAction::workspace_items();
        let action = match items.get(action_index) {
            Some(a) => *a,
            None => return,
        };

        if ws_index >= self.ws.workspaces.len() {
            return;
        }

        match action {
            crate::ContextMenuAction::Rename => {
                let current = self.ws.workspaces[ws_index].name.clone();
                self.modal.workspace_rename = Some(crate::WorkspaceRenameState {
                    ws_index,
                    input: crate::InputLine::with_text(current),
                });
                self.cache.invalidate_chrome();
            }
            _ => {}
        }
        self.cache.needs_redraw = true;
    }
}
```

- [ ] **Step 7: Run the tests**

```bash
cargo test -p tide-app --lib selecting_rename_in_workspace_context_menu 2>&1 | tail -10
cargo test -p tide-app --lib workspace_rename_participates_in_modal 2>&1 | tail -10
cargo build -p tide-app 2>&1 | tail -10
```

Expected: both pass; build green.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "Add WorkspaceRenameState modal in tide-app

UC-2 BR-2, BR-7: ContextMenu Rename for a Workspace rail item opens
workspace_rename seeded with the current name. Modal participates in
is_any_open() and close_all()."
```

---

## Task 7: Keyboard handler for `workspace_rename`

Mirror `handle_file_tree_rename_key`: Esc cancels, Enter commits, edit keys mutate the InputLine.

**Files:**
- Modify: `crates/tide-app/src/adapter/inward/keyboard_adapter/modal.rs`
- Modify: `crates/tide-app/src/adapter/inward/keyboard_adapter/mod.rs`
- Modify: `crates/tide-app/src/application/ports/inward/file_tree_port/mod.rs` (or wherever the file-tree-rename completion port lives — we need a parallel `complete_workspace_rename`)
- Modify: `crates/tide-app/src/application/services/workspace_service/mod.rs`
- Modify: `crates/tide-app/src/app.rs` (port wiring — `complete_file_tree_rename` is wired here at line 1157)
- Test: `crates/tide-app/src/application/behavior_tests/workspace_behavior.rs`

- [ ] **Step 1: Add the port method**

First find where `complete_file_tree_rename` is declared as a port — `grep -n "complete_file_tree_rename" crates/tide-app/src/application/ports/inward/`. The expected location is `application/ports/inward/file_tree_port/mod.rs:11`. We'll add a parallel method on the workspace port.

Edit `crates/tide-app/src/application/ports/inward/workspace_nav_port/mod.rs`. After `rename_workspace`, add:

```rust
    fn complete_workspace_rename(&mut self);
```

- [ ] **Step 2: Implement it on workspace_service**

Edit `crates/tide-app/src/application/services/workspace_service/mod.rs`. Inside the `impl WorkspaceNavPort for App` block:

```rust
    fn complete_workspace_rename(&mut self) {
        let state = match self.modal.workspace_rename.take() {
            Some(s) => s,
            None => return,
        };
        let new_name = state.input.text.trim().to_string();
        if new_name.is_empty() {
            // No change — cancel
            self.cache.invalidate_chrome();
            return;
        }
        if state.ws_index < self.ws.workspaces.len() {
            self.ws.workspaces[state.ws_index].name = new_name;
        }
        self.cache.invalidate_chrome();
    }
```

- [ ] **Step 3: Wire the port on `App`**

Edit `crates/tide-app/src/app.rs` near `complete_file_tree_rename` (line 1157). Add a parallel forwarder if the port is implemented through the `impl App` trampoline pattern. If the keyboard adapter calls `ctx.complete_workspace_rename()` directly via the `KeyboardPorts` trait, the implementation above is sufficient — verify by:

```bash
grep -n "trait KeyboardPorts" crates/tide-app/src/adapter/inward/keyboard_adapter/mod.rs
grep -n "WorkspaceNavPort" crates/tide-app/src/adapter/inward/keyboard_adapter/mod.rs
```

If `KeyboardPorts` already extends `WorkspaceNavPort`, no app.rs forwarder is needed. Otherwise add `WorkspaceNavPort` to the `KeyboardPorts` trait bound list.

- [ ] **Step 4: Add the modal key handler**

Edit `crates/tide-app/src/adapter/inward/keyboard_adapter/modal.rs`. After `handle_file_tree_rename_key`:

```rust
pub(super) fn handle_workspace_rename_key(
    ctx: &mut impl KeyboardPorts,
    key: Key,
    modifiers: &Modifiers,
) {
    match key {
        Key::Escape => {
            ctx.modal_mut().workspace_rename = None;
            ctx.invalidate_chrome();
        }
        Key::Enter => {
            ctx.complete_workspace_rename();
        }
        Key::Backspace => {
            if let Some(ref mut rename) = ctx.modal_mut().workspace_rename {
                rename.input.backspace();
            }
            ctx.invalidate_chrome();
        }
        Key::Delete => {
            if let Some(ref mut rename) = ctx.modal_mut().workspace_rename {
                rename.input.delete_char();
            }
            ctx.invalidate_chrome();
        }
        Key::Left => {
            if let Some(ref mut rename) = ctx.modal_mut().workspace_rename {
                rename.input.move_cursor_left();
            }
            ctx.invalidate_chrome();
        }
        Key::Right => {
            if let Some(ref mut rename) = ctx.modal_mut().workspace_rename {
                rename.input.move_cursor_right();
            }
            ctx.invalidate_chrome();
        }
        Key::Char(ch) => {
            if !modifiers.ctrl && !modifiers.meta {
                if let Some(ref mut rename) = ctx.modal_mut().workspace_rename {
                    rename.input.insert_char(ch);
                }
                ctx.invalidate_chrome();
            }
        }
        _ => {}
    }
    ctx.request_redraw();
}
```

- [ ] **Step 5: Route to it from `keyboard_adapter/mod.rs`**

Edit `crates/tide-app/src/adapter/inward/keyboard_adapter/mod.rs` around line 182–184. Below the existing file-tree-rename interception, add:

```rust
    // Workspace rename modal interception
    if ctx.modal().workspace_rename.is_some() {
        modal::handle_workspace_rename_key(ctx, key, &modifiers);
        return;
    }
```

- [ ] **Step 6: Write the failing tests**

Append to `workspace_behavior.rs`:

```rust
// UC-2 BR-3: Enter commits the new name and closes the modal
#[test]
fn enter_in_workspace_rename_commits_new_name() {
    use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;
    let mut app = app_with_two_workspaces();
    app.modal.workspace_rename = Some(crate::WorkspaceRenameState {
        ws_index: 0,
        input: crate::InputLine::with_text("Committed".to_string()),
    });

    app.complete_workspace_rename();

    assert_eq!(app.ws.workspaces[0].name, "Committed");
    assert!(app.modal.workspace_rename.is_none());
}

// UC-2 BR-4: Escape cancels and closes the modal without renaming
#[test]
fn escape_in_workspace_rename_cancels() {
    use crate::adapter::inward::keyboard_adapter;
    use crate::tide_input::{Key, Modifiers};
    let mut app = app_with_two_workspaces();
    let original = app.ws.workspaces[0].name.clone();
    app.modal.workspace_rename = Some(crate::WorkspaceRenameState {
        ws_index: 0,
        input: crate::InputLine::with_text("Should Not Apply".to_string()),
    });

    keyboard_adapter::handle_key_press(&mut app, Key::Escape, Modifiers::default());

    assert_eq!(app.ws.workspaces[0].name, original);
    assert!(app.modal.workspace_rename.is_none());
}

// UC-2 BR-5: empty commit leaves the name unchanged
#[test]
fn empty_commit_in_workspace_rename_leaves_name_unchanged() {
    use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;
    let mut app = app_with_two_workspaces();
    let original = app.ws.workspaces[0].name.clone();
    app.modal.workspace_rename = Some(crate::WorkspaceRenameState {
        ws_index: 0,
        input: crate::InputLine::with_text("   ".to_string()),
    });

    app.complete_workspace_rename();

    assert_eq!(app.ws.workspaces[0].name, original);
    assert!(app.modal.workspace_rename.is_none());
}
```

> **Note:** `keyboard_adapter::handle_key_press` is a guess at the entrypoint name. The engineer should `grep -n "pub.*fn handle_key" crates/tide-app/src/adapter/inward/keyboard_adapter/mod.rs` and use whatever the public entry point is. If the keyboard test pattern in the existing modal_behavior tests uses a different shape, copy that shape — `crates/tide-app/src/application/behavior_tests/modal_behavior.rs:346` (`escape_closes_file_tree_rename`) is the canonical example.

- [ ] **Step 7: Run the tests**

```bash
cargo test -p tide-app --lib workspace_rename 2>&1 | tail -15
```

Expected: 3 passes.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "Add workspace_rename keyboard handler in tide-app

UC-2 BR-3..BR-5: Enter commits, Escape cancels, empty commit leaves
the name unchanged."
```

---

## Task 8: Text routing for `workspace_rename`

Mirror the existing file_tree_rename routing in `text_routing_adapter`.

**Files:**
- Modify: `crates/tide-app/src/adapter/inward/text_routing_adapter/mod.rs`
- Test: `crates/tide-app/src/application/behavior_tests/text_input_routing.rs`

- [ ] **Step 1: Write the failing test**

Append to `crates/tide-app/src/application/behavior_tests/text_input_routing.rs` (mirror the existing `text_goes_to_file_tree_rename_when_active` test at line 222):

```rust
#[test]
fn text_goes_to_workspace_rename_input_when_active() {
    use crate::WorkspaceRenameState;
    let mut app = App::new();
    app.ws.workspaces.push(crate::update::workspace_infra_service::Workspace {
        name: "WS".into(),
        layout: crate::tide_layout::SplitLayout::new(),
        focused: None,
        panes: Default::default(),
    });
    app.modal.workspace_rename = Some(WorkspaceRenameState {
        ws_index: 0,
        input: crate::InputLine::new(),
    });

    app.route_text_input("h");
    app.route_text_input("i");

    let modal = app.modal.workspace_rename.as_ref().unwrap();
    assert_eq!(modal.input.text, "hi");
}
```

> **Note:** The exact entrypoint name (`route_text_input`) and how tests construct an `App` are visible in the file_tree_rename routing test above — copy its shape exactly. Also extend the `Destination` enum in `text_routing_adapter/mod.rs` if the existing code uses such an enum (line 19 lists `FileTreeRename`); add `WorkspaceRename` and route to it.

- [ ] **Step 2: Add the routing branch**

Edit `crates/tide-app/src/adapter/inward/text_routing_adapter/mod.rs`. Add `WorkspaceRename` to the `Destination` enum (line 19). In `route_destination` (the function that picks the destination, around line 59), add — **after** the `file_tree_rename` check:

```rust
    if modal.workspace_rename.is_some() {
        return Destination::WorkspaceRename;
    }
```

In the actual text-write path (around line 158, where the `file_tree_rename.input` is appended to), add a parallel branch:

```rust
                if let Some(ref mut rename) = self.modal.workspace_rename {
                    rename.input.insert_char(ch_or_string_char);
                }
```

- [ ] **Step 3: Run the test**

```bash
cargo test -p tide-app --lib text_goes_to_workspace_rename_input_when_active 2>&1 | tail -10
```

Expected: 1 passed.

- [ ] **Step 4: Commit**

```bash
git add crates/tide-app/src/adapter/inward/text_routing_adapter/mod.rs \
        crates/tide-app/src/application/behavior_tests/text_input_routing.rs
git commit -m "Route text input to workspace_rename in tide-app

UC-2 BR-6: characters typed while workspace_rename is open go to the
modal's InputLine."
```

---

## Task 9: Render the InputLine over the Workspace rail item

When `workspace_rename` is open for index `i`, replace the static name draw with an InputLine widget at the rail's `item_rect`.

**Files:**
- Modify: `crates/tide-app/src/adapter/outward/view/chrome/titlebar.rs`
- Modify: `crates/tide-app/src/adapter/outward/view/ime.rs` (cursor visibility uses the same gating as `file_tree_rename`)

- [ ] **Step 1: Find the existing file_tree_rename input render**

Read `crates/tide-app/src/adapter/outward/view/chrome/file_tree.rs:272–360` to see the pattern (input text + caret draw). We will mirror that shape in titlebar.rs for the workspace rail items.

```bash
sed -n '270,360p' crates/tide-app/src/adapter/outward/view/chrome/file_tree.rs
```

- [ ] **Step 2: Branch on `workspace_rename` in titlebar.rs**

Edit `crates/tide-app/src/adapter/outward/view/chrome/titlebar.rs` at the workspace item draw loop (lines 623–725). Right before the `display_name` computation (around line 683), add a fast path:

```rust
            let is_renaming = matches!(
                app.modal.workspace_rename.as_ref(),
                Some(state) if state.ws_index == i
            );
            if is_renaming {
                let state = app.modal.workspace_rename.as_ref().unwrap();
                // Use the full content width for the input — no truncation
                let (text_x, text_y) = (
                    content_x + WS_SIDEBAR_ITEM_PAD_H,
                    item_rect.y + (item_rect.height - cs.height) / 2.0,
                );
                renderer.draw_chrome_text(
                    &state.input.text,
                    Vec2::new(text_x, text_y),
                    TextStyle {
                        foreground: p.tab_text_focused,
                        background: None,
                        bold: true,
                        dim: false,
                        italic: false,
                        underline: false,
                    },
                    inset,
                );
                // Caret
                let caret_x = text_x + state.input.text[..state.input.cursor].chars().count() as f32 * cs.width;
                renderer.draw_chrome_rect(
                    Rect::new(caret_x, text_y, 1.5, cs.height),
                    p.tab_text_focused,
                );
                continue;
            }
```

The `inset` rect variable is the workspace sidebar inset — verify the variable name in scope at that point (`inset` is defined at the top of the sidebar draw block, around line 599).

- [ ] **Step 3: Gate the system caret in ime.rs**

Edit `crates/tide-app/src/adapter/outward/view/ime.rs:26`. Where the OR chain currently reads `|| app.modal.file_tree_rename.is_some();`, add:

```rust
        || app.modal.file_tree_rename.is_some()
        || app.modal.workspace_rename.is_some();
```

- [ ] **Step 4: Smoke-build**

```bash
cargo build -p tide-app 2>&1 | tail -10
```

Expected: green. No new tests in this task — render is visual. The text-routing test from Task 8 already confirms the model behavior.

- [ ] **Step 5: Commit**

```bash
git add crates/tide-app/src/adapter/outward/view/chrome/titlebar.rs \
        crates/tide-app/src/adapter/outward/view/ime.rs
git commit -m "Render workspace_rename InputLine on the Workspace rail in tide-app

UC-2: when workspace_rename is open, the rail item draws the InputLine
+ caret in place of the static name."
```

---

## Task 10: MCP tool `tide_rename_workspace`

**Files:**
- Modify: `crates/tide-app/src/adapter/inward/cli_adapter/mcp.rs`
- Modify: `crates/tide-app/src/adapter/inward/cli_adapter/commands.rs`
- Test: `crates/tide-app/src/application/behavior_tests/cli_workspace_routing.rs`

- [ ] **Step 1: Write the failing tests**

Append to `crates/tide-app/src/application/behavior_tests/cli_workspace_routing.rs`. **First** find a working pattern for cli tests in the same file:

```bash
sed -n '1,80p' crates/tide-app/src/application/behavior_tests/cli_workspace_routing.rs
```

Use whatever helper exists in that module to invoke a `cli_` function and assert on the JSON result. Then add:

```rust
// UC-3 BR-1: cli_rename_workspace with index renames the target Workspace
#[test]
fn cli_rename_workspace_with_index_renames_targeted_workspace() {
    let mut app = test_app_with_two_workspaces();
    let params = serde_json::json!({ "ws_index": 1, "name": "Via Index" });

    let result = crate::adapter::inward::cli_adapter::commands::cli_rename_workspace_for_test(
        &mut app, params,
    )
    .expect("ok result");

    assert_eq!(app.ws.workspaces[1].name, "Via Index");
    assert_eq!(result["ws_index"], 1);
    assert_eq!(result["name"], "Via Index");
}

// UC-3 BR-2: cli_rename_workspace without index renames the active Workspace
#[test]
fn cli_rename_workspace_without_index_renames_active_workspace() {
    let mut app = test_app_with_two_workspaces();
    let active = app.ws.active;
    let params = serde_json::json!({ "name": "Via Active" });

    let _ = crate::adapter::inward::cli_adapter::commands::cli_rename_workspace_for_test(
        &mut app, params,
    )
    .expect("ok result");

    assert_eq!(app.ws.workspaces[active].name, "Via Active");
}

// UC-3 BR-3: cli_rename_workspace with empty name is a no-op
#[test]
fn cli_rename_workspace_with_empty_name_is_a_no_op() {
    let mut app = test_app_with_two_workspaces();
    let active = app.ws.active;
    let original = app.ws.workspaces[active].name.clone();
    let params = serde_json::json!({ "name": "   " });

    let _ = crate::adapter::inward::cli_adapter::commands::cli_rename_workspace_for_test(
        &mut app, params,
    );

    assert_eq!(app.ws.workspaces[active].name, original);
}

// UC-3 BR-4: cli_rename_workspace with out-of-bounds index returns an error
#[test]
fn cli_rename_workspace_with_out_of_bounds_index_returns_error() {
    let mut app = test_app_with_two_workspaces();
    let len = app.ws.workspaces.len();
    let params = serde_json::json!({ "ws_index": len + 5, "name": "Nope" });

    let result = crate::adapter::inward::cli_adapter::commands::cli_rename_workspace_for_test(
        &mut app, params,
    );

    assert!(result.is_err());
}
```

> **Note:** `cli_rename_workspace_for_test` and `test_app_with_two_workspaces` are placeholders. Use whatever helpers `cli_workspace_routing.rs` already defines — the existing tests in that file will show the convention.

- [ ] **Step 2: Add the cli function**

Edit `crates/tide-app/src/adapter/inward/cli_adapter/commands.rs`. Below the existing workspace-related cli fns (e.g. after `cli_observe_workspace`), add:

```rust
fn cli_rename_workspace(ctx: &mut crate::App, params: Value) -> Result<Value, CliError> {
    use crate::application::ports::inward::workspace_nav_port::WorkspaceNavPort;

    let name = params
        .get("name")
        .and_then(|v| v.as_str())
        .ok_or(CliError::InvalidParams("name is required".into()))?
        .to_string();

    let ws_index = match params.get("ws_index") {
        Some(v) => v
            .as_u64()
            .ok_or(CliError::InvalidParams("ws_index must be a number".into()))?
            as usize,
        None => ctx.ws_active(),
    };

    if ws_index >= ctx.ws.workspaces.len() {
        return Err(CliError::InvalidParams(format!(
            "ws_index {ws_index} out of bounds (len {})",
            ctx.ws.workspaces.len()
        )));
    }

    ctx.rename_workspace(ws_index, name);

    Ok(serde_json::json!({
        "ws_index": ws_index,
        "name": ctx.ws.workspaces[ws_index].name.clone(),
    }))
}
```

> **Note:** the exact `CliError` variant name (`InvalidParams`) is a guess. Verify with `grep -n "enum CliError" crates/tide-app/src/adapter/inward/cli_adapter/protocol.rs` and use the right variant — the existing `cli_split` / `cli_focus_pane` functions show the convention.

- [ ] **Step 3: Wire dispatch**

Edit `crates/tide-app/src/adapter/inward/cli_adapter/commands.rs` around line 150 (the dispatch table). Add (keeping alphabetical or near-grouping consistent with neighbors):

```rust
            "rename-workspace" => cli_rename_workspace(self, params),
```

- [ ] **Step 4: Register the MCP tool**

Edit `crates/tide-app/src/adapter/inward/cli_adapter/mcp.rs`. In the tool list (the big array of `{ "name": "tide_..." }` objects), add an entry near the other workspace-related tools:

```rust
        serde_json::json!({
            "name": "tide_rename_workspace",
            "description": "Rename a Workspace by index (defaults to active). Empty names are rejected.",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "ws_index": { "type": "number", "description": "Optional; defaults to active." },
                    "name": { "type": "string" }
                },
                "required": ["name"]
            }
        }),
```

In the tool→method map (around line 573):

```rust
        "tide_rename_workspace" => "rename-workspace",
```

- [ ] **Step 5: Run the tests + build**

```bash
cargo test -p tide-app --lib cli_rename_workspace 2>&1 | tail -15
cargo build -p tide-app 2>&1 | tail -10
```

Expected: 4 passes, green build.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "Add tide_rename_workspace MCP tool in tide-app

UC-3: Wrapped Agents can rename Workspaces by index (defaults to active).
Empty names are no-ops; out-of-bounds index returns an error."
```

---

## Task 11: Final pass, lint, and PR prep

- [ ] **Step 1: Full test run**

```bash
cd ~/Development/github.com/lswank/tide-rename-workspaces
cargo test -p tide-app --lib --no-fail-fast 2>&1 | tail -30
```

Expected: identical pass/fail count to the pre-flight baseline, plus the new tests passing.

- [ ] **Step 2: Architecture lint**

```bash
./scripts/lint-arch.sh 2>&1 | tail -20
```

Expected: clean.

- [ ] **Step 3: Format / clippy if the project runs them**

```bash
cargo fmt -p tide-app -- --check 2>&1 | tail -10
cargo clippy -p tide-app --lib -- -D warnings 2>&1 | tail -30
```

If `cargo fmt --check` fails, run `cargo fmt -p tide-app` and amend or recommit.

- [ ] **Step 4: Push the branch**

```bash
git push -u origin feat/rename-workspaces 2>&1 | tail -10
```

- [ ] **Step 5: Open the PR**

Use the project's PR template. Reference issue #98 and list each affected use case + BR.

```bash
gh pr create \
  --repo team-attention/tide \
  --base main \
  --head lswank:feat/rename-workspaces \
  --title "Add Workspace rename in tide-app" \
  --body "$(cat <<'EOF'
Closes #98.

## Summary

Users can rename Workspaces via right-click on the Workspace rail → Rename, edit the inline InputLine, Enter to commit (Escape to cancel). Wrapped Agents can do the same via the new `tide_rename_workspace` MCP tool.

Spec: `docs/specs/rename-workspaces.md`.

## Bounded Context

- [x] `tide-app` — App orchestrator

## Domain Changes

**Entities/Aggregates modified:**
- Added `WorkspaceNavPort::rename_workspace(idx, name)` and `complete_workspace_rename()` methods.
- Added `WorkspaceRenameState { ws_index, input: InputLine }` to `ModalStack`.
- Refactored `ContextMenuState` to carry a `ContextMenuTarget` (`FileTreeEntry | WorkspaceSidebarItem`) so the Workspace rail shares the existing menu chrome with the FileTree View.
- Added `ContextMenuAction::workspace_items()` returning `[Rename]`.

**Invariants preserved:**
- PaneId sync untouched (rename does not interact with SplitLayout or panes).
- Modal exclusivity: `workspace_rename` participates in `is_any_open()` / `close_all()`; opening the modal consumes any open `context_menu`.
- Hexagonal direction: inward adapters call `WorkspaceNavPort::rename_workspace` / `complete_workspace_rename`; no direct mutation of `ws.workspaces[i].name`.

**Invariants changed:** None.

## Behavior Tests

```
mod workspace_behavior:
  rename_workspace_updates_active_workspace_name
  rename_workspace_updates_cold_stored_workspace_name
  rename_workspace_with_empty_name_is_a_no_op
  rename_workspace_persists_across_switch_out_and_back
  rename_workspace_with_out_of_bounds_index_is_a_no_op
  right_click_on_workspace_sidebar_item_opens_context_menu_with_rename
  selecting_rename_in_workspace_context_menu_opens_workspace_rename_modal
  workspace_rename_participates_in_modal_is_any_open_and_close_all
  enter_in_workspace_rename_commits_new_name
  escape_in_workspace_rename_cancels
  empty_commit_in_workspace_rename_leaves_name_unchanged

mod text_input_routing:
  text_goes_to_workspace_rename_input_when_active

mod cli_workspace_routing:
  cli_rename_workspace_with_index_renames_targeted_workspace
  cli_rename_workspace_without_index_renames_active_workspace
  cli_rename_workspace_with_empty_name_is_a_no_op
  cli_rename_workspace_with_out_of_bounds_index_returns_error
```

## How to Test

1. Launch Tide.
2. Open the Workspace rail (titlebar Workspace toggle).
3. Right-click on a Workspace item → `Rename`.
4. Type a new name → `Enter`. The rail label updates.
5. Switch Workspaces and switch back — the name persists.
6. As an agent: call `tide_rename_workspace { ws_index: 0, name: "Demo" }` over MCP and verify the rail updates.

## Out of scope

- Double-click on a Workspace rail item to enter rename. Tide currently has no general click-count tracking in `mouse_adapter`; adding it is its own design conversation. Filed as follow-up after this PR lands if maintainers want it.
EOF
)"
```

- [ ] **Step 6: Verify the PR URL**

```bash
gh pr view --repo team-attention/tide --json url,number 2>&1 | tail -5
```

Print the PR URL for the user.

---

## Self-Review Notes

- All 16 BRs in the spec (UC-1: 5, UC-2: 7, UC-3: 4) have a corresponding test in Tasks 1, 2, 5, 6, 7, 8, 10.
- UC-2 BR-1 (right-click opens menu): Task 5.
- UC-2 BR-2 (Rename opens modal seeded): Task 6 Step 3.
- UC-2 BR-3 / BR-4 / BR-5 (Enter / Escape / empty): Task 7 Step 6.
- UC-2 BR-6 (text routing): Task 8.
- UC-2 BR-7 (`is_any_open` + `close_all`): Task 6 Step 3.
- UC-3 BR-1..BR-4: Task 10 Step 1.
- No double-click is intentionally out of scope (called out in the PR body).
