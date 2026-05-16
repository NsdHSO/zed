# Git Diff vs Remote — Design Spec

**Date:** 2026-05-16
**Status:** Draft (pending user review)
**Crate:** `git_ui` (with re-use of existing `project::git_store::branch_diff`)

## Summary

Add a new Zed action, `git: DiffVsRemote`, that lets the user diff the current working tree against any remote-tracking branch (e.g. `origin/main`, `upstream/dev`). Selection happens through a fuzzy picker over all `refs/remotes/*` entries, with a non-blocking `git fetch --all` triggered when the picker opens so the list is fresh by the time the user picks. The action is reachable from the command palette, the branch picker modal, and the git panel toolbar.

The feature reuses the existing `DiffBase::Merge { base_ref }` infrastructure already powering the `BranchDiff` action — only the picker and a thin deploy helper are new.

## Goals

- Let the user compare their working tree against an arbitrary remote-tracking branch in one shortcut.
- Keep the list of remote refs fresh without making the user wait.
- Reuse the existing `ProjectDiff` rendering pipeline (multibuffer, hunks, review flow) end-to-end.
- Three discoverable entry points (palette, branch picker, panel toolbar).

## Non-Goals

- No new config setting (e.g. `git_panel.branch_diff_base`). The existing `BranchDiff` action keeps its current "diff vs default branch" behavior.
- No new diff base variant. `DiffBase::Merge { base_ref }` already accepts arbitrary refs and is unchanged.
- No fetch of refs in repositories with no remotes — silent skip, no error toast.
- No support for diffing against arbitrary SHAs or tags in this iteration. Remote-tracking branches only.
- No remote management UI (add/remove/rename remote).

## Architecture

```
git_ui/
├── remote_diff_picker.rs        [NEW]   picker modal, fetch trigger
├── project_diff.rs              [edit]  deploy_remote_diff handler (mirrors deploy_branch_diff)
├── git_ui.rs                    [edit]  actions! add DiffVsRemote; register
├── git_panel.rs                 [edit]  toolbar button → dispatches DiffVsRemote
└── branch_picker.rs             [edit]  entry "Compare with remote..." → dispatches DiffVsRemote

project/src/git_store/
└── (no change — DiffBase::Merge { base_ref } already supports "origin/main")
```

The picker depends on `Repository::branches()` and `Repository::fetch()`. On selection, it calls `ProjectDiff::deploy_remote_diff(workspace, base_ref, window, cx)`, which mirrors `ProjectDiff::deploy_branch_diff` (currently at `crates/git_ui/src/project_diff.rs:116`) but injects the user-picked `base_ref` instead of resolving `repo.default_branch(true)`.

## Components

### 1. Action

Added to the `actions!(git, ...)` block in `crates/git_ui/src/project_diff.rs`:

```rust
/// Shows the diff between the working directory and a remote-tracking branch.
DiffVsRemote,
```

Unit action. Bindable. Surfaces in the command palette as `git: diff vs remote`.

### 2. `RemoteDiffPicker` — `crates/git_ui/src/remote_diff_picker.rs`

New file, ~250 LOC. Modeled on `branch_picker::BranchList`.

**State:**
```rust
pub struct RemoteDiffPicker {
    picker: Entity<Picker<RemoteDiffPickerDelegate>>,
}

struct RemoteDiffPickerDelegate {
    repo: Entity<Repository>,
    workspace: WeakEntity<Workspace>,
    all_remote_branches: Vec<Branch>,
    matches: Vec<StringMatch>,
    selected_index: usize,
    error: Option<SharedString>,
    fetch_task: Option<Task<()>>,
    _subscription: Subscription,
}
```

**Init:** Filter `repo.branches()` to entries where `branch.is_remote() == true`. Display label is `branch.name()` (e.g. `origin/main`).

**Auto-fetch:** On construction, spawn one `Task` that calls `repo.fetch(FetchOptions::All, askpass_delegate, cx).await`. `FetchOptions::All` maps to `git fetch --all`, so a single call updates every configured remote. Failure is surfaced via `show_error_toast` (matches `branch_picker.rs:25` pattern) and logged with `.log_err()`. The picker stays usable with cached refs while the fetch is in flight or after it fails.

**Refresh on event:** Subscribe to the active `Repository` and refresh `all_remote_branches` whenever `RepositoryEvent::StatusesChanged` fires (this is the event emitted after `fetch` updates refs).

**Confirm:** Closes the modal and calls `ProjectDiff::deploy_remote_diff(workspace, selected_branch.name().into(), window, cx)`.

**Empty state:** Picker shows the placeholder "No remote branches. Run `git: fetch` first." when `all_remote_branches.is_empty()` and no error is present.

### 3. `ProjectDiff::deploy_remote_diff` — `crates/git_ui/src/project_diff.rs`

Mirrors `deploy_branch_diff` (line 116):

```rust
fn deploy_remote_diff(
    workspace: &mut Workspace,
    base_ref: SharedString,
    window: &mut Window,
    cx: &mut Context<Workspace>,
) {
    telemetry::event!("Git Remote Diff Opened");
    // Look for an existing ProjectDiff with DiffBase::Merge { .. } and re-use it,
    // calling branch_diff.set_diff_base(DiffBase::Merge { base_ref }, cx) instead of
    // resolving repo.default_branch(true). Same activation + needs_switch logic
    // as deploy_branch_diff. New ProjectDiff is created via new_with_base_ref(...)
    // if none exists.
}
```

A small `ProjectDiff::new_with_base_ref` helper is added alongside `new_with_default_branch` (line 354). It is the same body, parametrized over the initial `base_ref`.

### 4. Entry Points

**Command palette:** In `ProjectDiff::register` (line 98), add:
```rust
workspace.register_action(|workspace, _: &DiffVsRemote, window, cx| {
    open_remote_diff_picker(workspace, window, cx);
});
```

`open_remote_diff_picker` is a free function in `remote_diff_picker.rs` that:
1. Pulls the active repository from `workspace.project()`.
2. Early-returns with a toast "No git repository in workspace" if none.
3. Calls `workspace.toggle_modal(window, cx, |window, cx| RemoteDiffPicker::new(...))`.

**Branch picker** (`crates/git_ui/src/branch_picker.rs`): Add a menu entry "Compare with remote..." to the existing branch-list popover (alongside the existing `FilterRemotes`/`DeleteBranch` actions). Selecting it dismisses the branch picker and dispatches `DiffVsRemote`.

**Git panel toolbar** (`crates/git_ui/src/git_panel.rs`): Add a button next to the existing diff button. Tooltip: "Diff vs remote". On click, `window.dispatch_action(DiffVsRemote.boxed_clone(), cx)`.

### 5. Fetch Helper

Reuses `Repository::fetch(FetchOptions::All, askpass_delegate, cx)` (declared at `crates/git/src/repository.rs:941`). `FetchOptions::All` lives in `crates/git/src/repository.rs:553`. No new git-layer code; no need to enumerate remotes from the UI layer.

## Data Flow

```
User triggers DiffVsRemote (palette / branch picker / panel button)
   │
   ▼
open_remote_diff_picker(workspace, window, cx)
   • get active_repository
   • cx.new RemoteDiffPicker { repo, all_remote_branches: [], fetch_task: None }
   • workspace.toggle_modal(picker)
   │
   ▼
RemoteDiffPicker::new
   • spawn load_initial_branches:
       repo.branches().await → filter is_remote() → push to delegate.all_branches → cx.notify
   • spawn fetch_task:
       repo.fetch(FetchOptions::All, AskPassDelegate, cx).log_err()
       (single `git fetch --all` invocation)
   • subscribe to RepositoryEvent::StatusesChanged → reload branches list
   │
   ▼
User types query → fuzzy_nucleo match against branch.name() → matches updated → cx.notify
   │
   ▼
Fetch completes → repo emits StatusesChanged → subscription fires → reload branches → re-fuzzy-match query → cx.notify
   │
   ▼
User confirms selection (Enter)
   • dismiss modal
   • call ProjectDiff::deploy_remote_diff(workspace, base_ref, window, cx)
   │
   ▼
ProjectDiff::deploy_remote_diff
   • find existing ProjectDiff with DiffBase::Merge { .. } → activate + set_diff_base
   • OR new ProjectDiff via Self::new_with_base_ref(project, workspace, base_ref, window, cx)
   • branch_diff.set_diff_base(DiffBase::Merge { base_ref }, cx)
   │
   ▼
BranchDiff (existing) updates tree_diff via DiffTreeType::MergeBase { base: base_ref, head: HEAD }
   │
   ▼
ProjectDiff renders multibuffer with hunks (existing path)
```

**Cache invariant:** `all_remote_branches` always reflects the last successful `repo.branches()` call. Fetch is best-effort; failure leaves the cache intact.

**Cancellation:** Dropping the picker drops `fetch_task`. Background fetches continue or cancel per the existing `Repository::fetch` semantics — unchanged by this work.

## Error Handling

| Failure | Behavior | Surface |
|---|---|---|
| No active repository | Action no-op; toast "No git repository in workspace" | `open_remote_diff_picker` early-return |
| `repo.branches()` fails | Picker stays open with empty list + inline error label | `log_err()` + delegate stores `Option<SharedString>` error |
| `repo.fetch(FetchOptions::All)` fails (network, auth, no remotes configured) | Toast notification: "Failed to fetch from remotes: \<err\>"; picker still usable with cached refs | `show_error_toast` (already used in `branch_picker.rs:25`) |
| User picks ref that vanished mid-flow | `BranchDiff::tree_diff_base_task` returns error from `DiffTreeType::MergeBase` | Existing `ProjectDiff` error path |
| No remote branches exist at all | Picker shows "No remote branches. Run `git: fetch` first." empty-state | Delegate `placeholder_text` + empty match list |

Per project rules (`/Volumes/Working/forks/zed/zed/CLAUDE.md`):
- No `let _ = ...` on fallible operations.
- `.log_err()` for fire-and-forget fetch tasks.
- `?` inside `async move` for `branches()`/`get_remotes()`.
- Public picker constructors return `Result<_>` and are dispatched via `detach_and_notify_err`.
- No `unwrap()` or panicking indexing on user-controlled data.

## Testing

**Unit tests** in `remote_diff_picker.rs` (mirror `branch_picker.rs:1922+` `create_test_branches` pattern):

| Test | Asserts |
|---|---|
| `test_remote_picker_filters_to_remote_branches` | Given mixed local+remote `Branch` vec, picker shows only `is_remote() == true` entries |
| `test_remote_picker_displays_full_ref_name` | Match candidates use `branch.name()` form (`origin/main`, not just `main`) |
| `test_remote_picker_fuzzy_match` | Query `"orig main"` matches `origin/main` |
| `test_remote_picker_empty_state` | No remote branches → empty-state message; no panic |
| `test_remote_picker_confirm_dispatches_diff` | Selecting `origin/main` triggers `deploy_remote_diff` with `base_ref == "origin/main"` |
| `test_remote_picker_repo_event_refreshes_list` | Emit `RepositoryEvent::StatusesChanged` → branches reloaded |

**Integration tests** in `project_diff.rs`:

| Test | Asserts |
|---|---|
| `test_deploy_remote_diff_creates_new_view` | Fresh workspace → `deploy_remote_diff` with `base_ref` → new `ProjectDiff` item added, `DiffBase::Merge { base_ref }` set |
| `test_deploy_remote_diff_reuses_existing_branch_diff_view` | Pre-existing `ProjectDiff` with `DiffBase::Merge` → reused, `set_diff_base` called with new `base_ref` |

**No fetch tests at unit level.** `Repository::fetch` is already covered in `crates/git` and requires real-remote infrastructure. The picker simply calls it under `detach_and_log_err`; mocking is not necessary for this layer.

**GPUI executor:** Use `cx.background_executor().timer(...)` per project rules — no `smol::Timer`.

**Run:** `cargo test --workspace -p git_ui` and `./script/clippy`.

## Open Questions

None at design time. Implementation may surface picker-styling choices (e.g. whether to show `tracking_status()` hints inline); those are local decisions, not spec-level.

## Release Notes

```
Release Notes:

- Added `git: diff vs remote` action to compare the working tree against a remote-tracking branch chosen from a fuzzy picker (also reachable from the branch picker and the git panel toolbar). The picker auto-fetches all remotes in the background so the list stays fresh.
```
