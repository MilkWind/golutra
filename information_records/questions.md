# Questions and Answers

## Q1: How does this project implement context isolation?

### Answer:

This is a **Tauri 2.x application** (not Electron). Tauri implements security and isolation through a different (but equally robust) mechanism compared to Electron's "context isolation". Here's how it works in this project:

#### 1. Isolated Webview

The frontend (Vue 3 application) runs in a secure webview with **no direct access to system resources**. It can only communicate with the backend through defined IPC commands.

#### 2. Capability-Based Security (Critical)

Tauri 2.x uses **capabilities** to control exactly what the frontend can access. This is configured in:

- `src-tauri/capabilities/default.json`

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "enables the default permissions",
  "windows": ["main", "main*", "terminal*", "workspace-selection*", "notification-preview"],
  "permissions": [
    "core:default",
    "dialog:default",
    "clipboard-manager:allow-read-text",
    "clipboard-manager:allow-write-text",
    "shell:allow-open",
    "core:window:allow-minimize",
    "core:window:allow-maximize",
    "core:window:allow-toggle-maximize",
    "core:window:allow-close",
    "core:window:allow-destroy",
    "core:window:allow-start-dragging",
    "core:window:allow-start-resize-dragging",
    "core:window:allow-is-maximized",
    "core:window:allow-show",
    "core:window:allow-unminimize",
    "core:window:allow-set-focus"
  ]
}
```

This file explicitly lists all permissions granted to the frontend for the specified windows.

#### 3. IPC-Only Communication

All communication between the frontend and backend happens **only through defined Tauri commands**. These commands are registered in:

- `src-tauri/src/ui_gateway/commands.rs`

```rust
//! UI 命令注册：统一导出 Tauri 命令入口。

use tauri::ipc::Invoke;

use super::app;
use super::message;
use super::monitoring;
use super::notification;
use super::platform;
use super::project_data;
use super::project_members;
use super::project_skills;
use super::skills;
use super::terminal;

pub(crate) fn export_commands() -> impl Fn(Invoke) -> bool + Send + Sync + 'static {
  tauri::generate_handler![
    app::terminal_open_window,
    app::workspace_selection_open_window,
    terminal::terminal_create,
    terminal::terminal_list_environments,
    terminal::terminal_attach,
    terminal::terminal_write,
    terminal::terminal_ack,
    terminal::terminal_set_active,
    terminal::terminal_emit_status,
    terminal::terminal_set_member_status,
    terminal::terminal_dispatch,
    terminal::terminal_resize,
    terminal::terminal_close,
    terminal::terminal_list_statuses,
    terminal::terminal_snapshot_lines,
    terminal::terminal_snapshot_text,
    terminal::terminal_dump_snapshot_lines,
    notification::notification_update_state,
    notification::notification_get_state,
    notification::notification_preview_hover,
    notification::notification_preview_hide,
    notification::notification_request_ignore_all,
    notification::notification_set_active_window,
    app::notification_open_terminal,
    app::notification_open_all_unread,
    app::notification_open_unread_conversation,
    monitoring::diagnostics_start_run,
    monitoring::diagnostics_end_run,
    monitoring::diagnostics_register_member,
    monitoring::diagnostics_register_session,
    monitoring::diagnostics_register_conversation,
    monitoring::diagnostics_register_window,
    monitoring::diagnostics_log_frontend_event,
    monitoring::diagnostics_log_frontend_batch,
    monitoring::diagnostics_log_snapshot_triplet,
    monitoring::diagnostics_log_chat_consistency,
    platform::platform_get_updater_status,
    platform::platform_get_activation_status,
    message::chat_ulid_new,
    message::chat_repair_messages,
    message::chat_clear_all_messages,
    message::chat_list_conversations,
    message::chat_get_messages,
    message::chat_mark_conversation_read_latest,
    message::chat_send_message,
    message::chat_send_message_and_dispatch,
    message::chat_create_group,
    message::chat_ensure_direct,
    message::chat_set_conversation_settings,
    message::chat_rename_conversation,
    message::chat_clear_conversation,
    message::chat_delete_conversation,
    message::chat_set_conversation_members,
    app::workspace_recent_list,
    app::workspace_open_folder,
    app::workspace_open,
    app::workspace_clear_window,
    app::storage_read_app,
    app::storage_write_app,
    app::storage_read_cache,
    app::storage_write_cache,
    app::storage_write_cache_text,
    app::storage_read_workspace,
    app::storage_write_workspace,
    app::avatar_list,
    app::avatar_store,
    app::avatar_delete,
    app::avatar_resolve_path,
    app::avatar_read,
    project_data::project_data_read,
    project_data::project_data_write,
    project_members::project_members_invite,
    project_members::project_members_purge_terminal,
    project_skills::project_skills_list,
    project_skills::project_skills_link,
    project_skills::project_skills_unlink,
    skills::skills_import_folder,
    skills::skills_remove_folder,
    skills::skills_open_folder
  ]
}
```

These commands are then registered with the Tauri app in `src-tauri/src/lib.rs:127`:

```rust
.invoke_handler(export_commands())
```

#### 4. Sandboxed Backend

The Rust backend runs with minimal privileges by default, and only the explicitly registered commands are available to the frontend.

#### Summary

While Tauri doesn't use the exact term "context isolation" like Electron, it provides equivalent (and often stronger) security through:

- Isolated webview environment
- Capability-based permission system
- IPC-only communication channel
- Minimal-privilege backend

## Q2: How does the project create terminal windows?

### Answer:

This project creates terminal windows through a frontend-backend workflow using Tauri's IPC commands. Here's the step-by-step process:

#### 1. Frontend Invocation

The frontend calls `openTerminalWindow` from `src/features/terminal/openTerminalWindow.ts`, which invokes the Tauri command `terminal_open_window`:

```typescript
// src/features/terminal/openTerminalWindow.ts
import { invoke } from '@tauri-apps/api/core';

export type TerminalWindowOptions = {
  reuse?: boolean;
  workspaceId?: string;
  workspaceName?: string;
  workspacePath?: string;
  autoTab?: boolean;
};

export type TerminalWindowResult = {
  label: string;
  reused: boolean;
};

export const openTerminalWindow = async (options?: TerminalWindowOptions) => {
  if (typeof window === 'undefined') {
    return null;
  }

  return invoke<TerminalWindowResult>('terminal_open_window', {
    reuse: options?.reuse ?? true,
    workspaceId: options?.workspaceId,
    workspaceName: options?.workspaceName,
    workspacePath: options?.workspacePath,
    autoTab: options?.autoTab,
  });
};
```

#### 2. Backend Command Implementation

The actual window creation happens in `src-tauri/src/ui_gateway/app.rs` via the `terminal_open_window` Tauri command:

```rust
// src-tauri/src/ui_gateway/app.rs

// Constants for window labels and sizes (lines 42-74)
static WINDOW_COUNTER: AtomicUsize = AtomicUsize::new(1);
const TERMINAL_WINDOW_LABEL: &str = "terminal-main";
const TERMINAL_WINDOW_DEFAULT_WIDTH: f64 = 1200.0;
const TERMINAL_WINDOW_DEFAULT_HEIGHT: f64 = 800.0;
const TERMINAL_WINDOW_MIN_WIDTH: f64 = 720.0;
const TERMINAL_WINDOW_MIN_HEIGHT: f64 = 520.0;
const TERMINAL_WINDOW_MAX_WIDTH: f64 = 1500.0;
const TERMINAL_WINDOW_MAX_HEIGHT: f64 = 1000.0;

// Generate terminal window labels (lines 154-179)
fn next_terminal_label(reuse: bool) -> String {
  if reuse {
    TERMINAL_WINDOW_LABEL.to_string()
  } else {
    let suffix = WINDOW_COUNTER.fetch_add(1, Ordering::Relaxed);
    format!("terminal-{suffix}")
  }
}

fn terminal_label_for_workspace(workspace_id: &str) -> String {
  let hash = hash_bytes(workspace_id.as_bytes());
  format!("terminal-workspace-{hash}")
}

// Result struct (lines 478-483)
#[derive(Serialize)]
pub(crate) struct TerminalWindowOpenResult {
  label: String,
  reused: bool,
}

// Main command implementation (lines 584-704)
#[tauri::command]
pub(crate) async fn terminal_open_window(
  app: AppHandle,
  reuse: Option<bool>,
  workspace_id: Option<String>,
  workspace_name: Option<String>,
  workspace_path: Option<String>,
  auto_tab: Option<bool>,
) -> Result<TerminalWindowOpenResult, String> {
  let reuse = reuse.unwrap_or(true);
  let auto_tab = auto_tab.unwrap_or(true);

  // Check for existing workspace or reusable terminal window
  let workspace_label = workspace_id.as_deref().map(terminal_label_for_workspace);
  if let Some(label) = workspace_label.as_deref() {
    if let Some(window) = app.get_webview_window(label) {
      // Show existing window
      let _ = window.show();
      let _ = window.unminimize();
      let _ = window.set_focus();
      return Ok(TerminalWindowOpenResult {
        label: label.to_string(),
        reused: true,
      });
    }
  } else if reuse {
    if let Some(window) = app.get_webview_window(TERMINAL_WINDOW_LABEL) {
      // Show existing window
      let _ = window.show();
      let _ = window.unminimize();
      let _ = window.set_focus();
      return Ok(TerminalWindowOpenResult {
        label: TERMINAL_WINDOW_LABEL.to_string(),
        reused: true,
      });
    }
  }

  // Create new terminal window
  let label = workspace_label.unwrap_or_else(|| next_terminal_label(reuse));
  let title = workspace_name
    .as_deref()
    .map(|name| format!("{name} - Terminal"))
    .unwrap_or_else(|| "Terminal".to_string());
  let init_payload = json!({
    "id": workspace_id.as_deref(),
    "name": workspace_name.as_deref(),
    "path": workspace_path.as_deref()
  });
  let init_script = format!(
    "window.__GOLUTRA_VIEW__ = 'terminal'; window.__GOLUTRA_WORKSPACE__ = {init_payload}; window.__GOLUTRA_TERMINAL_AUTO_TAB__ = {auto_tab};"
  );

  let (window_width, window_height) = resolve_chat_window_size(
    &app,
    TERMINAL_WINDOW_MIN_WIDTH,
    TERMINAL_WINDOW_MIN_HEIGHT,
    TERMINAL_WINDOW_MAX_WIDTH,
    TERMINAL_WINDOW_MAX_HEIGHT,
    TERMINAL_WINDOW_DEFAULT_WIDTH,
    TERMINAL_WINDOW_DEFAULT_HEIGHT,
  );
  let min_width = TERMINAL_WINDOW_MIN_WIDTH.min(window_width);
  let min_height = TERMINAL_WINDOW_MIN_HEIGHT.min(window_height);

  // Build window using Tauri's WebviewWindowBuilder
  let window_builder = WebviewWindowBuilder::new(&app, label.clone(), WebviewUrl::App("index.html".into()))
    .initialization_script(&init_script)
    .title(title)
    .inner_size(window_width, window_height)
    .min_inner_size(min_width, min_height)
    .resizable(true)
    .center()
    .decorations(false)
    .transparent(true)
    .shadow(!cfg!(target_os = "windows"));
  #[cfg(target_os = "macos")]
  let window_builder = window_builder.title_bar_style(tauri::TitleBarStyle::Overlay);
  let window = window_builder
    .build()
    .map_err(|err| format!("failed to create terminal window: {err}"))?;
  apply_windows_rounding(&window);

  let _ = window.show();
  let _ = window.set_focus();

  Ok(TerminalWindowOpenResult { label, reused: false })
}
```

#### 3. Command Registration

The `terminal_open_window` command is registered in `src-tauri/src/ui_gateway/commands.rs:18`:

```rust
app::terminal_open_window,
```

#### 4. Permissions for Terminal Windows

Terminal windows are granted permissions via `src-tauri/capabilities/default.json`, which includes `"terminal*"` in the windows list to match all terminal window labels.

#### Key Window Labels

- `terminal-main`: Default reusable terminal window
- `terminal-<number>`: New non-reusable terminal windows (uses atomic counter)
- `terminal-workspace-<hash>`: Workspace-specific terminal windows (hashed workspace ID)

## Q3: How does the supervisor call members and what's the internal logic?

### Answer:

This project's "supervisor" (orchestrator) dispatches messages to terminal members through a coordinated frontend-backend workflow. Here's the detailed logic:

#### 1. Frontend: Terminal Orchestrator Store

The entry point for terminal dispatch is `src/stores/terminalOrchestratorStore.ts`:

```typescript
// src/stores/terminalOrchestratorStore.ts
// Lines 186-208: Dispatch conversation messages to terminals
const dispatchConversationToTerminals = async (payload: {
  conversation: Conversation;
  text: string;
  mentions?: MessageMentionsPayload;
  senderId: string;
  senderName: string;
}) => {
  const targets = resolveTerminalTargets(payload.conversation, payload.mentions, payload.senderId);
  if (targets.length === 0) {
    return;
  }
  for (const memberId of targets) {
    const request: TerminalDispatchRequest = {
      memberId,
      conversationId: payload.conversation.id,
      conversationType: payload.conversation.type,
      senderId: payload.senderId,
      senderName: payload.senderName,
      text: payload.text,
    };
    void enqueueTerminalDispatch(request);
  }
};

// Lines 92-106: Enqueue terminal dispatch requests
const enqueueTerminalDispatch = async (request: TerminalDispatchRequest) => {
  try {
    await terminalMemberStore.enqueueTerminalDispatch(request);
  } catch (error) {
    console.error('Failed to dispatch terminal message.', error);
  }
};
```

#### 2. Frontend: Terminal Member Store

Manages serial command dispatch queues per member to avoid interleaving in `src/features/terminal/terminalMemberStore.ts`:

```typescript
// src/features/terminal/terminalMemberStore.ts
// Lines 674-691: Serial dispatch queue per member
const enqueueTerminalDispatch = async (request: TerminalDispatchRequest) => {
  const workspace = currentWorkspace.value;
  if (!workspace) {
    return;
  }
  const member = members.value.find((candidate) => candidate.id === request.memberId);
  if (!hasTerminalConfig(member?.terminalType, member?.terminalCommand)) {
    return;
  }
  const memberKey = buildMemberKey(request.memberId, workspace.id);
  const chain = dispatchChains.get(memberKey) ?? Promise.resolve();
  const task = chain.then(
    () => dispatchTerminalMessage(request, member),
    () => dispatchTerminalMessage(request, member),
  );
  dispatchChains.set(memberKey, task);
  await task;
};

// Lines 647-667: Dispatch terminal message to backend
const dispatchTerminalMessage = async (request: TerminalDispatchRequest, member: Member) => {
  const entry = await ensureMemberSession(member, { openTab: false });
  if (!entry) {
    return;
  }
  await dispatchSession(entry.terminalId, buildCommandInput(request.text), {
    conversationId: request.conversationId,
    conversationType: request.conversationType,
    senderId: request.senderId,
    senderName: request.senderName,
  });
  await new Promise<void>((resolve) => {
    window.setTimeout(resolve, COMMAND_CONFIRM_DELAY_MS);
  });
  await dispatchSession(entry.terminalId, COMMAND_CONFIRM_SUFFIX, {
    conversationId: request.conversationId,
    conversationType: request.conversationType,
    senderId: request.senderId,
    senderName: request.senderName,
  });
};
```

#### 3. Backend: Orchestration Dispatch

The backend orchestrates chat-to-terminal dispatch in `src-tauri/src/orchestration/dispatch.rs`:

```rust
// src-tauri/src/orchestration/dispatch.rs
// Lines 76-196: Orchestrate chat dispatch to terminals
pub fn orchestrate_chat_dispatch(
    app: &AppHandle,
    window: &WebviewWindow,
    terminal_state: State<'_, TerminalManager>,
    chat_state: State<'_, ChatDbManager>,
    storage: &StorageManager,
    payload: ChatDispatchPayload,
) -> Result<(), String> {
    let member_ids = chat_get_conversation_member_ids(
        chat_state,
        payload.workspace_id.clone(),
        payload.conversation_id.clone(),
    )?;
    let member_set: HashSet<String> = member_ids.iter().cloned().collect();
    if member_set.is_empty() {
        log_chat_dispatch_skip(
            app,
            &payload,
            "no_conversation_members",
            json!({
                "memberCount": member_ids.len()
            }),
        );
        return Ok(());
    }

    let project_data = project_data::read_project_data(
        storage,
        &payload.workspace_path,
        &payload.workspace_id,
    )?;
    let member_configs = match project_data.data {
        Some(value) => collect_member_configs(&value),
        None => HashMap::new(),
    };
    let member_config_count = member_configs.len();

    let default_mentions = ChatDispatchMentions {
        mention_ids: Vec::new(),
        mention_all: false,
    };
    let mentions = payload.mentions.as_ref().unwrap_or(&default_mentions);
    let mut targets = resolve_targets(
        payload.conversation_type.as_str(),
        &member_ids,
        &member_set,
        payload.sender_id.as_str(),
        mentions,
    );
    if targets.is_empty() {
        log_chat_dispatch_skip(
            app,
            &payload,
            "no_targets",
            json!({
                "memberCount": member_ids.len()
            }),
        );
        return Ok(());
    }
    let targets_before_filter = targets.len();
    targets.retain(|id| member_configs.contains_key(id));
    if targets.is_empty() {
        log_chat_dispatch_skip(
            app,
            &payload,
            "targets_missing_config",
            json!({
                "targetsBeforeFilter": targets_before_filter,
                "memberConfigCount": member_config_count
            }),
        );
        return Ok(());
    }

    let context = TerminalDispatchContext {
        conversation_id: payload.conversation_id.clone(),
        conversation_type: payload.conversation_type.clone(),
        sender_id: payload.sender_id.clone(),
        sender_name: payload.sender_name.clone(),
        message_id: payload.message_id.clone(),
        client_trace_id: payload.client_trace_id.clone(),
        client_timestamp: payload.timestamp,
    };

    let batcher = app.state::<Arc<ChatDispatchBatcher>>();
    let mut dispatched_count = 0usize;
    let mut skipped_missing_terminal_config = 0usize;
    for target_id in targets {
        let Some(config) = member_configs.get(&target_id) else {
            continue;
        };
        if !has_terminal_config(config) {
            skipped_missing_terminal_config = skipped_missing_terminal_config.saturating_add(1);
            continue;
        }
        let terminal_id = ensure_backend_member_session(
            app,
            window,
            &terminal_state,
            config,
            &payload.workspace_id,
            &payload.workspace_path,
        )?;
        batcher.enqueue_for_terminal(app, terminal_id, payload.text.clone(), context.clone())?;
        dispatched_count = dispatched_count.saturating_add(1);
    }
    if dispatched_count == 0 {
        log_chat_dispatch_skip(
            app,
            &payload,
            "targets_no_terminal_config",
            json!({
                "targetsBeforeFilter": targets_before_filter,
                "memberConfigCount": member_config_count,
                "skippedMissingTerminalConfig": skipped_missing_terminal_config
            }),
        );
    }
    Ok(())
}
```

#### Summary of the Flow

1. **Frontend**: A chat message is sent, mentions are resolved to target terminal members
2. **Terminal Orchestrator**: Creates `TerminalDispatchRequest` objects for each target
3. **Terminal Member Store**: Manages a serial dispatch queue per member to avoid command interleaving
4. **Backend**: `orchestrate_chat_dispatch` ensures sessions exist and dispatches messages
5. **Terminal Engine**: Handles actual session management and command execution
