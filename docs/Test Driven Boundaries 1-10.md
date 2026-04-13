```yaml
### task.tdb-002-global-shell.tela.yaml
version: "4.1.0"
task_id: "tdb-002-global-shell"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Build the Global UI Shell including the Sidebar, Status Bar, and base layout."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src/App.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
    - path: "src/components/Sidebar.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
    - path: "src/components/StatusBar.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Implement a permanent left Sidebar Navigation with routing links for Dashboard, Tasks, Snapshots, Packages, and Logs."
    - "Implement a bottom Status Bar displaying current application state, last action description, and Git branch."
    - "Ensure all UI state is managed via a reactive store configured to persist to ~/.nare/ui_state.json."
  must_not:
    - "Do not implement the internal content of the individual views (e.g., Dashboard, Tasks) in this task."
verification:
  ast_check:
    command: "npx prettier --check {file}"
    expected_exit: 0
  compiler_check:
    command: "npx tsc --noEmit"
    expected_exit: 0
  behavioral_checks:
    - name: "verify_shell_layout"
      command: "npm run test -- src/App.test.tsx"
      expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_ui_shell.sh"
```

```yaml
### task.tdb-003-dashboard.tela.yaml
version: "4.1.0"
task_id: "tdb-003-dashboard"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Implement the Dashboard View and Intent Input Panel."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src/views/Dashboard.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
    - path: "src/components/IntentPanel.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Create the Dashboard View with a Project Header showing the Git branch and Snapshot Status badge."
    - "Implement an Intent Input Panel featuring a text area (max 2000 characters) that auto-saves drafts to ~/.nare/drafts/ every 5 seconds."
    - "Add a 'Plan' button that triggers Phase 1: Planning, disabled if the snapshot is not CURRENT or the Intent is empty."
  must_not:
    - "Do not implement the Tauri Rust commands in this task; only implement the UI components and mock IPC calls."
verification:
  ast_check:
    command: "npx prettier --check {file}"
    expected_exit: 0
  compiler_check:
    command: "npx tsc --noEmit"
    expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_dashboard.sh"
```

```yaml
### task.tdb-004-snapshot-indexer.tela.yaml
version: "4.1.0"
task_id: "tdb-004-snapshot-indexer"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Implement the Snapshot Indexer Engine in the Rust backend."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src-tauri/src/snapshot_indexer.rs"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Implement the create_snapshot() command to traverse the workspace respecting .nareignore."
    - "Compute SHA-256 hashes for each file and parse them into semantic chunks."
    - "Generate a snapshot_manifest.json file to store the resulting structured representation."
  must_not:
    - "Do not integrate live vector DB embedding API calls in this specific task; mock the vector storage step for now."
verification:
  ast_check:
    command: "rustfmt --check {file}"
    expected_exit: 0
  compiler_check:
    command: "cargo check --message-format=short"
    expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_snapshot_indexer.sh"
```

```yaml
### task.tdb-005-orchestrator-planning.tela.yaml
version: "4.1.0"
task_id: "tdb-005-orchestrator-planning"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Implement the Oracle Planning Phase command in the Orchestrator."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src-tauri/src/orchestrator.rs"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Implement the plan(intent: String) Tauri command."
    - "The command must fetch top-K chunks, assemble the Oracle prompt, and invoke the Gemini API with a 120s timeout."
    - "Parse the returned YAML block, validate it against the NARE Task Schema, and save valid tasks to .nare/tasks/active/<uuid>.yaml with a PENDING status."
  must_not:
    - "Do not execute code; strictly write files to the task queue."
verification:
  ast_check:
    command: "rustfmt --check {file}"
    expected_exit: 0
  compiler_check:
    command: "cargo check --message-format=short"
    expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_orchestrator_plan.sh"
```

```yaml
### task.tdb-006-tasks-view.tela.yaml
version: "4.1.0"
task_id: "tdb-006-tasks-view"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Implement the Tasks View and Task Editor Modal."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src/views/TasksView.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
    - path: "src/components/TaskEditorModal.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Implement a sortable, filterable Task Table showing Task ID, Status, and Created timestamps."
    - "Build the Task Editor Modal capable of parsing and updating target YAML fields like intent_summary, target files, and verification commands."
    - "Include batch action capabilities for PENDING tasks (e.g., Delete Selected, Export Selected)."
  must_not:
    - "Do not couple the view to external state; use the centralized reactive store."
verification:
  ast_check:
    command: "npx prettier --check {file}"
    expected_exit: 0
  compiler_check:
    command: "npx tsc --noEmit"
    expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_tasks_view.sh"
```

```yaml
### task.tdb-007-undo-redo.tela.yaml
version: "4.1.0"
task_id: "tdb-007-undo-redo"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Implement the Undo/Redo System via the Rust Undo Manager."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src-tauri/src/undo_manager.rs"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Implement an Undo Manager maintaining two LIFO stacks: undo_stack and redo_stack."
    - "Ensure each stack entry stores action_type, forward_patch, reverse_patch, and timestamp."
    - "Expose Tauri commands that the frontend can bind to Ctrl+Z and Ctrl+Shift+Z."
  must_not:
    - "Do not allow undo operations for external side effects like Git commits or immutable log entries."
verification:
  ast_check:
    command: "rustfmt --check {file}"
    expected_exit: 0
  compiler_check:
    command: "cargo check --message-format=short"
    expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_undo_redo.sh"
```

```yaml
### task.tdb-008-package-export.tela.yaml
version: "4.1.0"
task_id: "tdb-008-package-export"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Implement the Package Export View modal for bundling TDBs."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src/views/PackageExportView.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Build a modal with a Task Selection Table showing all PENDING and AWAITING_HUMAN tasks."
    - "Implement an 'Include System Instructions' checkbox (defaulting to unchecked)."
    - "Add a payload size monitor that displays a warning banner if the generated string exceeds 50,000 characters."
    - "Ensure the 'Copy Button' is disabled if the size exceeds 100,000 characters, directing users to the Download button."
  must_not:
    - "Do not modify the file system directly from the UI; call a Tauri command to execute the download export."
verification:
  ast_check:
    command: "npx prettier --check {file}"
    expected_exit: 0
  compiler_check:
    command: "npx tsc --noEmit"
    expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_package_export.sh"
```

```yaml
### task.tdb-009-repository-sync.tela.yaml
version: "4.1.0"
task_id: "tdb-009-repository-sync"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Implement the Git synchronization and hash verification logic."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src-tauri/src/orchestrator/sync.rs"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Implement the sync() command that executes git pull --ff-only on the default branch."
    - "After pulling, compare each target file's current hash against the hash_before recorded in the task's context."
    - "If a file hash has changed for an AWAITING_HUMAN task, automatically update its status to COMPLETED."
  must_not:
    - "Do not attempt to automatically resolve Git merge conflicts; abort and log SYNC_CONFLICT."
verification:
  ast_check:
    command: "rustfmt --check {file}"
    expected_exit: 0
  compiler_check:
    command: "cargo check --message-format=short"
    expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_repo_sync.sh"
```

```yaml
### task.tdb-010-logs-and-settings.tela.yaml
version: "4.1.0"
task_id: "tdb-010-logs-and-settings"
created: "2026-04-13T11:24:20Z"
master_builder: "Alec Borman"
intent_summary: "Implement the Settings View and Immutable Audit Log Viewer."
target:
  repository: "nare-hitl-ui"
  files:
    - path: "src/views/SettingsView.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
    - path: "src/views/LogsView.tsx"
      hash_before: "ACTUATOR_TO_POPULATE"
    - path: "src-tauri/src/keychain.rs"
      hash_before: "ACTUATOR_TO_POPULATE"
boundary:
  must:
    - "Implement the Settings View to manage the Gemini API Key and GitHub PAT, routing them to the OS keychain via Rust."
    - "Build the Logs View table to parse and display the immutable .nare/logs/audit.jsonl file with filters for Level and Component."
    - "Configure UI inputs for Oracle model selection, chunk size, and themes."
  must_not:
    - "Do not store API credentials in plaintext configuration files; enforce keychain storage."
verification:
  ast_check:
    command: "rustfmt --check {file} && npx prettier --check src/views/*.tsx"
    expected_exit: 0
  compiler_check:
    command: "cargo check --message-format=short && npx tsc --noEmit"
    expected_exit: 0
surgery:
  method: "deterministic_script"
  script_language: "bash"
  script_path: "./scripts/build_logs_settings.sh"
```
