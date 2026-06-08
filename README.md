Let me read the existing signalGuard context first.

Here's the entire approach with all files.

---

## Architecture Overview

```
preToolUse.ps1          postToolUse.ps1
     |                        |
  creates                  deletes
.signal.lock            .signal.lock
     |                        |
     └──────┐      ┌──────────┘
            ▼      ▼
        signalGuard.ts  (FileSystemWatcher on .signal.lock)
            |
     _trustedWrite = true/false
            |
     onWillSaveTextDocument  +  FileSystemWatcher on signal.json
            |
     ALLOW or BLOCK + REVERT
```

---

## File 1: `signal.json`
```json
{ "flag": false }
```
> Default state. Always committed as `false`. Never edit manually.

---

## File 2: `preToolUse.ps1`
```powershell
# Step 1: Create lock file — tells the extension a trusted write is coming
New-Item -Path ".signal.lock" -ItemType File -Force | Out-Null

# Step 2: Small delay to let the extension's FileSystemWatcher pick up the lock
Start-Sleep -Milliseconds 200

# Step 3: Set flag to true
'{ "flag": true }' | Set-Content -Path "./signal.json" -Encoding UTF8

Write-Host "✅ preToolUse complete — signal.json flag set to true"
```

---

## File 3: `postToolUse.ps1`
```powershell
# Step 1: Create lock file again (in case it was cleared mid-run)
New-Item -Path ".signal.lock" -ItemType File -Force | Out-Null

# Step 2: Small delay
Start-Sleep -Milliseconds 200

# Step 3: Reset flag to false
'{ "flag": false }' | Set-Content -Path "./signal.json" -Encoding UTF8

# Step 4: Remove lock — extension will now block all further edits
Remove-Item -Path ".signal.lock" -Force

Write-Host "✅ postToolUse complete — signal.json flag reset to false, lock removed"
```

---

## File 4: `signalGuard.ts`
```typescript
import * as vscode from 'vscode';
import * as fs from 'fs';
import * as path from 'path';

let _trustedWrite = false;
let _lastKnownGood: string | null = null;

export function registerSignalGuard(context: vscode.ExtensionContext) {
  const workspaceFolders = vscode.workspace.workspaceFolders;
  if (!workspaceFolders) return;

  const root = workspaceFolders[0].uri.fsPath;
  const signalPath = path.join(root, 'signal.json');
  const lockPath = path.join(root, '.signal.lock');

  // ─── Snapshot initial content ───────────────────────────────────────────────
  if (fs.existsSync(signalPath)) {
    _lastKnownGood = fs.readFileSync(signalPath, 'utf-8');
  }

  // ─── 1. Watch .signal.lock — controls the trust gate ────────────────────────
  const lockWatcher = vscode.workspace.createFileSystemWatcher(
    new vscode.RelativePattern(workspaceFolders[0], '.signal.lock')
  );

  lockWatcher.onDidCreate(() => {
    _trustedWrite = true;
    console.log('[SignalGuard] Lock created → trusted write OPEN');
  });

  lockWatcher.onDidDelete(() => {
    _trustedWrite = false;
    console.log('[SignalGuard] Lock deleted → trusted write CLOSED');
  });

  // ─── 2. Intercept saves from VS Code editor ──────────────────────────────────
  const saveGuard = vscode.workspace.onWillSaveTextDocument((event) => {
    const filePath = event.document.uri.fsPath;

    if (path.basename(filePath) !== 'signal.json') return;

    if (_trustedWrite) {
      // Trusted — update snapshot after save
      event.waitUntil(
        Promise.resolve([]).then(() => {
          setTimeout(() => {
            if (fs.existsSync(signalPath)) {
              _lastKnownGood = fs.readFileSync(signalPath, 'utf-8');
            }
          }, 100);
          return [];
        })
      );
      return;
    }

    // ❌ Untrusted — block and revert
    vscode.window.showErrorMessage(
      '🚫 signal.json is pipeline-controlled. Manual edits are not allowed.'
    );

    event.waitUntil(
      new Promise<vscode.TextEdit[]>((resolve) => {
        setTimeout(() => {
          vscode.commands.executeCommand('workbench.action.files.revert');
          resolve([]);
        }, 50);
      })
    );
  });

  // ─── 3. Watch signal.json at filesystem level (catches Cline prompt edits) ───
  const signalWatcher = vscode.workspace.createFileSystemWatcher(
    new vscode.RelativePattern(workspaceFolders[0], 'signal.json')
  );

  signalWatcher.onDidChange(() => {
    if (_trustedWrite) {
      // Trusted write — update snapshot
      if (fs.existsSync(signalPath)) {
        _lastKnownGood = fs.readFileSync(signalPath, 'utf-8');
        console.log('[SignalGuard] Trusted write — snapshot updated');
      }
      return;
    }

    // ❌ Untrusted filesystem write (e.g. Cline prompt, external editor)
    if (_lastKnownGood !== null) {
      fs.writeFileSync(signalPath, _lastKnownGood, 'utf-8');
      console.log('[SignalGuard] Untrusted write detected — restored snapshot');

      vscode.window.showErrorMessage(
        '🚫 signal.json was modified externally and has been restored. This file is pipeline-controlled.'
      );

      // Revert in-editor buffer if the file is open
      const openDoc = vscode.workspace.textDocuments.find(
        (d) => d.uri.fsPath === signalPath
      );
      if (openDoc) {
        vscode.commands.executeCommand('workbench.action.files.revert');
      }
    }
  });

  // ─── Cleanup ─────────────────────────────────────────────────────────────────
  context.subscriptions.push(saveGuard, lockWatcher, signalWatcher);

  console.log('[SignalGuard] Registered — signal.json is protected');
}
```

---

## File 5: `extension.ts` — wire it up
```typescript
import * as vscode from 'vscode';
import { registerSignalGuard } from './signalGuard';

export function activate(context: vscode.ExtensionContext) {
  registerSignalGuard(context);

  // ... rest of your existing activation code
}

export function deactivate() {}
```

---

## File 6: `.gitignore`
```
.signal.lock
```
> Never commit the lock file. If it's left behind after a crash, the extension would incorrectly trust all writes until VS Code restarts.

---

## Edge Case: Lock file left behind after a crash

Add this to `registerSignalGuard` at startup — cleans up stale locks:

```typescript
// At the top of registerSignalGuard, after getting lockPath:
if (fs.existsSync(lockPath)) {
  fs.unlinkSync(lockPath);
  console.log('[SignalGuard] Stale lock cleared on startup');
}
_trustedWrite = false;
```

---

## Full Flow Summary

```
preToolUse.ps1 runs
  → creates .signal.lock
  → extension sees lock: _trustedWrite = true
  → PS1 writes signal.json { flag: true }  ✅ allowed

Cline generates code
  → signal.json untouched

postToolUse.ps1 runs
  → creates .signal.lock (safety)
  → writes signal.json { flag: false }     ✅ allowed
  → deletes .signal.lock
  → extension sees lock gone: _trustedWrite = false

Human tries to edit signal.json
  → onWillSaveTextDocument fires
  → _trustedWrite = false → BLOCKED 🚫
  → error shown, file reverted

Cline prompt tries to edit signal.json
  → FileSystemWatcher fires
  → _trustedWrite = false → RESTORED 🚫
  → error shown, last known good content written back
```
