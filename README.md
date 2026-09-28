# ICP Blazor Hello World

A hello world template combining **Blazor WebAssembly (.NET 10)** on the frontend and **Motoko** on the backend, deployed fully on-chain on the **Internet Computer (ICP)**.

> The first working template for Blazor WASM + icp-cli + Motoko.

## Stack

| Layer    | Technology                          |
|----------|-------------------------------------|
| Frontend | Blazor WebAssembly (.NET 10)        |
| Backend  | Motoko canister                     |
| Bridge   | @dfinity/agent (webpack bundle)     |
| Platform | Internet Computer (ICP)             |
| CLI      | icp-cli                             |

## Project Structure

```
icp-blazor-hello/
├── icp.yaml                          # icp-cli project config
├── .gitignore
├── backend/
│   ├── canister.yaml                 # backend canister config
│   ├── backend.did                   # Candid interface (semicolons required!)
│   └── src/
│       └── main.mo                   # Motoko canister
└── frontend/
    ├── canister.yaml                 # frontend canister config (build + sync steps)
    ├── vendor/
    │   └── assetstorage.wasm.gz      # pinned asset canister WASM (no build-time download)
    └── BlazorFrontend/
        ├── BlazorFrontend.csproj     # .NET 10 Blazor WASM
        ├── Program.cs
        ├── App.razor
        ├── _Imports.razor            # required — Blazor namespace imports
        ├── package.json              # webpack + @dfinity/agent
        ├── webpack.config.js         # bundles icpAgent.ts → wwwroot/icpAgent.js
        ├── tsconfig.json
        ├── src/
        │   └── icpAgent.ts           # TypeScript ICP agent bridge
        ├── Layout/
        │   └── MainLayout.razor
        ├── Pages/
        │   └── Home.razor            # main UI with canister calls
        ├── Services/
        │   └── IcpAgentService.cs    # C# → JS interop service
        └── wwwroot/
            ├── index.html
            └── app.css
```

## How it works

```
Home.razor (C#)
  → IcpAgentService.cs (IJSRuntime)
    → window.IcpAgent.* (webpack bundle, defer loaded)
      → @dfinity/agent
        → Motoko backend canister on ICP
```

The key insight: use **webpack** to bundle `@dfinity/agent` into a plain JS file
loaded with `defer`, not as an ES module. This avoids the race condition between
the module loader and Blazor's JS interop system.

## Prerequisites

```bash
# icp-cli
npm install -g @icp-sdk/icp-cli @icp-sdk/ic-wasm

# Motoko toolchain
npm install -g ic-mops
```

### .NET 10 SDK

**Method 1: Ubuntu 24.04 LTS or newer (APT)**
```bash
sudo apt update
sudo apt install -y dotnet-sdk-10.0
```

> **Ubuntu 22.04 LTS?** Register the backports PPA first:
> ```bash
> sudo add-apt-repository ppa:dotnet/backports
> sudo apt update
> sudo apt install -y dotnet-sdk-10.0
> ```

**Method 2: Official Microsoft script (recommended for non-Ubuntu or if APT fails)**
```bash
wget https://dot.net/v1/dotnet-install.sh -O dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 10.0
```

After installing, add .NET to your PATH:
```bash
echo 'export PATH="$HOME/.dotnet:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Blazor WASM workload (required)
```bash
dotnet workload install wasm-tools
```

### Remove conflicting `icp` binary (Ubuntu/Debian)
Ubuntu ships a package called `renameutils` that installs its own unrelated `icp` binary.
Remove it before installing icp-cli:
```bash
sudo apt remove renameutils -y
```

Verify the correct `icp` is active:
```bash
icp --version  # should show icp-cli, not renameutils
```

## Run locally

```bash
# 1. Start the local ICP network (uses port 8002 to avoid colliding with other
#    local icp-cli projects that default to 8000 - see icp.yaml networks.gateway.port)
icp network start -d
icp network status   # Gateway Url: http://localhost:8002/

# 2. Deploy both canisters
icp deploy

# 3. Open the frontend URL printed by icp deploy (friendly domain), e.g.
#    frontend: http://frontend.local.localhost:8002/
#    The canister-id form works too: http://<frontend-canister-id>.localhost:8002/
```

On WSL, if `icp deploy` stops at `Creating canisters:` with a frozen
`unlock keyring` terminal, see
[the keyring gotcha](#wsl-keyring-freeze--failed-to-load-identity--secret-service-unlock-prompt-was-dismissed--warn-copy-mode--unlock-keyring-ubuntu)
below — a `plaintext` local identity makes deploys fully headless.

## Deploy to mainnet

```bash
icp deploy --network ic
```

## Known gotchas

### `failed to fetch wasm file` on the frontend canister

A deploy used to fail with:

```
[frontend] ✘ Failed to build canister: failed to fetch wasm file
ERR [frontend] > Fetching wasm: https://github.com/dfinity/sdk/releases/latest/download/assetstorage.wasm.gz
```

That URL came from the `@dfinity/asset-canister` recipe. The recipe downloads the
asset canister module from GitHub's `releases/latest` **on every single build**
and passes no `sha256`, and icp-cli only reuses its package cache for a remote
WASM when a checksum is given (`Remote without sha256: always downloads`). So any
transport failure — DNS, TLS, a connection reset, a timeout, or a proxy/CDN that
blocks GitHub's release-assets host — aborts the deploy, even though nothing in
the project changed.

`frontend/canister.yaml` therefore no longer uses that recipe. The module is
vendored at `frontend/vendor/assetstorage.wasm.gz` and loaded by a local
`pre-built` step with a `sha256`, so building the frontend never needs network
access for the canister WASM. The recipe's asset-upload step is kept verbatim as
a `sync:` plugin step. To bump the asset canister version, follow
[frontend/vendor/README.md](frontend/vendor/README.md) and update the `sha256`
in `frontend/canister.yaml`.

### `_Imports.razor` is required
Without `_Imports.razor`, Blazor component events silently do nothing.
Always include `@using Microsoft.JSInterop` in it.

### Port 8000 conflict
This project is configured to use port `8002` (`icp.yaml` → `networks.gateway.port`) to avoid colliding with other local `icp-cli` projects that default to `8000` (each project’s `pocket-ic` locks the gateway port). If you see:

```
Error: port 8000 is already in use by the local network of another project at /...
```

either stop the other project (`icp network stop` in that project folder) or change the port in this project’s `icp.yaml`.

If `icp network start` exits with status 101 for another reason, check the log:
```bash
cat .icp/cache/networks/local/network-launcher/stderr.log
```
Common causes:
- Docker container mapped to port 8000: `docker ps | grep 8000` → `docker stop <container>`
- WSL `socat` mirror (`TCP-LISTEN:8001,fork TCP:127.0.0.1:8000`) blocking `8001`: stop it (`pkill socat`) or pick a free port like `8002/8010`
- `ss -tulpn | grep :8002` → `kill -9 <pid>`

### WSL keyring freeze — `failed to load identity` / `Secret Service: unlock prompt was dismissed` / `warn copy mode : unlock keyring (ubuntu)`

`icp-cli` stores identities in the OS keyring by default (`--storage keyring`). In WSL there is no GUI keyring daemon, so `icp deploy` tries to pop a `gnome-keyring` unlock dialog that freezes as a “copy mode” terminal. Dismissing it makes every `Creating canisters:` step fail:

```
Error: failed to load identity
Caused by:
  0: failed to load password from keyring entry
  1: Couldn't access platform secure storage: Secret Service: unlock prompt was dismissed
```

**Fix (WSL, local replica only): use a `plaintext` identity — no prompts, headless**

```bash
# 1. Create a local-only plaintext identity (no keyring, no password)
icp identity new local --storage plaintext

# 2. Make it the default (or use --identity local per command)
icp identity default local

# 3. Verify it works without a prompt
icp identity principal   # should print a principal like gxiai-...-mqe
icp identity list        # * local  <principal>

# 4. Then retry the network + deploy (8002 avoids the port clash)
icp network start -d
icp network status
icp deploy
```

Keep your existing `deployer`/`default` keyring identities for mainnet! Only use the `local` plaintext identity on `pocket-ic` (ephemeral, no funds at risk). If you need encryption locally, use `--storage password` instead:

```bash
echo \"$(openssl rand -base64 24)\" > ~/.config/icp-cli/.icp-password
chmod 600 ~/.config/icp-cli/.icp-password
icp identity new local-pw --storage password --storage-password-file ~/.config/icp-cli/.icp-password
icp deploy --identity local-pw --identity-password-file ~/.config/icp-cli/.icp-password
```

See `scripts/fix-wsl-keyring.sh` for an automated version of the above.

### `Invalid certificate: Certificate is signed more than 5 minutes in the future`

Clicking **Get Greeting** (a query call) fails with:

```
Error: JS Error: Error while making call: Invalid certificate: Certificate is signed more than 5 minutes in the future.
Certificate time: 2026-09-28T22:32:19.195Z Current time: 2026-09-28T21:32:13.979Z
```

This is a **clock skew between the replica and your browser**, not a canister bug.
`@dfinity/agent` tolerates only **5 minutes** of drift when validating the
certified query response. In a WSL setup the replica uses the WSL clock while the
browser uses the Windows clock, and those can disagree by a long way.

Diagnose it (compare WSL against a network reference):

```bash
date -u                                     # WSL / replica clock
curl -sI https://example.com | grep -i '^date:'   # real time from the internet
powershell.exe -NoProfile -Command 'Get-Date -Format o'
powershell.exe -NoProfile -Command 'w32tm /query /status'   # look for "not synchronized"
```

Fix the skew at the source (Windows side, needs **Administrator**):

```powershell
# In an elevated PowerShell / Terminal
w32tm /resync /force
# If that reports "Access is denied (0x80070005)" you are not elevated.
# Also enable Settings → Time & language → Date & time →
#   "Set time automatically" and "Set time zone automatically", then click "Sync now".
```

```bash
# Or restart WSL so it re-syncs its clock from the Windows host
wsl.exe --shutdown
```

The frontend also defends against this locally: `icpAgent.ts` passes
`verifyQuerySignatures: false` **only for local hosts** (`localhost` /
`*.localhost`), because local query responses are already trust-on-first-use via
`fetchRootKey()`. Mainnet keeps full verification enabled, so the real fix for a
skewed host clock is still to re-sync the clock as shown above.
## License

MIT
