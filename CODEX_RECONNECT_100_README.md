# Codex Reconnect 100 Build Notes

This workspace keeps a custom Windows build of `@openai/codex` where the
Responses stream reconnect retry count is patched to `100`.

## What Was Changed

The reconnect counter shown as `Reconnecting... 1/5` comes from the Responses
stream retry logic, not from a current working `config.toml` TUI setting.

For `@openai/codex 0.142.5`, the relevant source is:

- `codex-rs/model-provider-info/src/lib.rs`
- `codex-rs/core/src/responses_retry.rs`

The build workflow patches these constants before compiling:

```rust
const DEFAULT_STREAM_MAX_RETRIES: u64 = 100;
const MAX_STREAM_MAX_RETRIES: u64 = 100;
```

The important part is that this value is compiled into `codex.exe`. Do not rely
on `config.toml` for this behavior if your installed Codex version no longer
honors that field.

## Current Local Files

- Built patched exe:
  `D:\coding\CodexFork\artifacts\codex-28513525811\codex.exe`
- Replacement script:
  `D:\coding\CodexFork\replace-codex-exe.ps1`
- Wait-until-unlocked replacement script:
  `D:\coding\CodexFork\replace-codex-when-free.ps1`
- Replacement log:
  `D:\coding\CodexFork\replace-codex-when-free.log`
- Success marker:
  `D:\coding\CodexFork\replace-codex-when-free.SUCCESS.txt`
- Failure marker:
  `D:\coding\CodexFork\replace-codex-when-free.FAILED.txt`

The npm-installed Codex binary path is:

```powershell
C:\Users\zyfang\AppData\Roaming\npm\node_modules\@openai\codex\node_modules\@openai\codex-win32-x64\vendor\x86_64-pc-windows-msvc\bin\codex.exe
```

## How The GitHub Actions Workflow Works

Workflow in the fork:

```text
zyfang2020/codex/.github/workflows/build-windows-codex-local.yml
```

It is a manual workflow. It does not permanently modify the upstream source
tree. Each run does this:

1. Checks out `openai/codex` at the requested ref, for example `rust-v0.142.5`.
2. Uses PowerShell to patch `codex-rs/model-provider-info/src/lib.rs`.
3. Inserts/updates tests that verify the default retry count and cap.
4. Runs:

   ```powershell
   cargo test -p codex-model-provider-info test_stream_max_retries --lib
   ```

5. Builds Windows `codex.exe`:

   ```powershell
   cargo build -p codex-cli --bin codex --release
   ```

6. Uploads an artifact named:

   ```text
   codex-windows-x64-local
   ```

The workflow inputs are:

- `codex_ref`: upstream ref/tag to build, for example `rust-v0.142.5`
- `reconnect_attempts`: retry count to compile in, normally `100`

## Build The Current Version Again

```powershell
& "C:\Program Files\GitHub CLI\gh.exe" workflow run build-windows-codex-local.yml `
  --repo zyfang2020/codex `
  --ref main `
  -f codex_ref=rust-v0.142.5 `
  -f reconnect_attempts=100
```

After it completes, download the artifact:

```powershell
mkdir D:\coding\CodexFork\artifacts\codex-new
& "C:\Program Files\GitHub CLI\gh.exe" run download <RUN_ID> `
  --repo zyfang2020/codex `
  --name codex-windows-x64-local `
  --dir D:\coding\CodexFork\artifacts\codex-new
```

Replace `<RUN_ID>` with the Actions run id.

## Update To A New Codex Version And Keep 100 Retries

1. Update npm Codex normally:

   ```powershell
   npm install -g @openai/codex@latest
   codex --version
   ```

2. Find the matching upstream Rust tag. It normally looks like:

   ```text
   rust-v0.xxx.x
   ```

3. Trigger the workflow for that tag:

   ```powershell
   & "C:\Program Files\GitHub CLI\gh.exe" workflow run build-windows-codex-local.yml `
     --repo zyfang2020/codex `
     --ref main `
     -f codex_ref=rust-v0.xxx.x `
     -f reconnect_attempts=100
   ```

4. Download the artifact.
5. Replace the npm vendor `codex.exe` with the new patched artifact.
6. Verify:

   ```powershell
   codex --version
   ```

If the source layout changes in a future version, the workflow will fail in the
patch or test step. In that case, locate the new retry constants and update the
workflow patch block.

## Replace The Installed EXE

If Codex is not running:

```powershell
powershell -ExecutionPolicy Bypass -File D:\coding\CodexFork\replace-codex-exe.ps1
```

If Codex is currently running and locking `codex.exe`, open another PowerShell
window and run:

```powershell
powershell -ExecutionPolicy Bypass -File D:\coding\CodexFork\replace-codex-when-free.ps1
```

Then exit the current Codex/TUI. The script waits until Windows releases the
old exe, backs it up, replaces it, and writes either:

```text
D:\coding\CodexFork\replace-codex-when-free.SUCCESS.txt
D:\coding\CodexFork\replace-codex-when-free.FAILED.txt
```

## Verify Replacement

Compare hashes:

```powershell
Get-FileHash "D:\coding\CodexFork\artifacts\codex-28513525811\codex.exe"
Get-FileHash "C:\Users\zyfang\AppData\Roaming\npm\node_modules\@openai\codex\node_modules\@openai\codex-win32-x64\vendor\x86_64-pc-windows-msvc\bin\codex.exe"
```

Run:

```powershell
codex --version
```

For the current custom build, the expected version string is still:

```text
codex-cli 0.142.5
```

The version string does not show the reconnect patch. The proof is the workflow
test passing plus the installed `codex.exe` hash matching the patched artifact.
