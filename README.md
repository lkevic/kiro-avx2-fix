# kiro-avx2-fix: Kiro IDE on pre-Haswell CPUs (no AVX2)

A workaround for Kiro IDE crashing silently on Linux with CPUs that don't support AVX2 instructions (Intel Ivy Bridge 2012 and earlier, AMD equivalent).

Related issue: [kirodotdev/Kiro#2741](https://github.com/kirodotdev/Kiro/issues/2741)

---

## ⚠️ Disclaimer

**Use this at your own risk.** This workaround involves recompiling a third-party native binary and replacing files inside Kiro's installation directory. While the original binary is backed up automatically, this is not an official fix and may break after Kiro updates. The authors take no responsibility for any damage to your system or Kiro installation. Always review scripts before running them with `sudo`.

---

## Symptoms

- Kiro appears to freeze on the login screen
- The IDE never redirects to the browser for authentication
- No obvious error is shown in the UI

What's actually happening under the hood is that the extension host crashes repeatedly with **exit code 132 (SIGILL — Illegal Instruction)** and Kiro gives up after 3 crashes in 5 minutes, leaving the UI frozen.

---

## Root Cause

The `kiro.kiroAgent` extension bundles `@lancedb/vectordb`, a vector database used for local AI embeddings. The prebuilt binary distributed with Kiro (`index.node`) is compiled with **AVX2 instructions**, which are only available on Intel Haswell (2013) and later CPUs.

On older CPUs (e.g. Intel Ivy Bridge i7-3770K), the OS sends a **SIGILL (Illegal Instruction)** signal the moment the binary is loaded, crashing the extension host before it can do anything — including opening the browser for login.

### How to confirm this is your issue

**Step 1 — Check if your CPU has AVX2:**
```bash
grep -o 'avx[^ ]*' /proc/cpuinfo | sort -u
```
If the output only shows `avx` (not `avx2`), your CPU is affected.

**Step 2 — Confirm AVX2 instructions in the LanceDB binary:**
```bash
objdump -d /usr/share/kiro/resources/app/extensions/kiro.kiro-agent/node_modules/@lancedb/vectordb-linux-x64-gnu/index.node | grep -c "ymm\|avx2\|avx512"
```
If this returns a number greater than 0 (it returned **461,398** in my case), the binary requires AVX2.

**Step 3 — Check the extension host crash in dmesg:**
```bash
sudo dmesg | grep -i "kiro\|sigill\|illegal" | tail -20
```
Look for lines mentioning your username or the Kiro process. If you see repeated entries with `code 132` or `SIGILL`, this confirms the crash.

---

## Workaround

Recompile LanceDB from source targeting your CPU architecture and replace the prebuilt binary.

### Prerequisites

```bash
# Node 22 (required by kiro.kiroAgent)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env

# Build tools
sudo apt install -y git build-essential protobuf-compiler binutils
```

### Steps

```bash
mkdir -p ~/repo/kiro-avx2-fix && cd ~/repo/kiro-avx2-fix

# Get LanceDB version bundled with Kiro
VERSION=$(python3 -c "import json; print(json.load(open('/usr/share/kiro/resources/app/extensions/kiro.kiro-agent/node_modules/@lancedb/vectordb-linux-x64-gnu/package.json'))['version'])")
echo "LanceDB version: $VERSION"

# Clone and checkout matching tag
git clone https://github.com/lancedb/lancedb.git
cd lancedb
git checkout "tags/v$VERSION"

# Install Node dependencies
cd node
npm install

# Patch arrow-arith compilation issue with Rust 1.94+
ARROW_ARITH=$(find ~/.cargo/registry -path "*/arrow-arith-*/src/temporal.rs" 2>/dev/null | head -1)
if [ -n "$ARROW_ARITH" ] && grep -q "d\.quarter()" "$ARROW_ARITH"; then
    sed -i 's/DatePart::Quarter => |d| d.quarter() as i32,/DatePart::Quarter => |d| ChronoDateExt::quarter(\&d) as i32,/' "$ARROW_ARITH"
    echo "arrow-arith patch applied."
fi

# Compile without AVX2
RUSTFLAGS="-C target-cpu=ivybridge" cargo build --release

# Backup original and replace
LANCEDB_PATH="/usr/share/kiro/resources/app/extensions/kiro.kiro-agent/node_modules/@lancedb/vectordb-linux-x64-gnu/index.node"
sudo cp "$LANCEDB_PATH" "${LANCEDB_PATH}.bak"
sudo cp ~/repo/kiro-avx2-fix/lancedb/target/release/liblancedb_node.so "$LANCEDB_PATH"

echo "Done. Restart Kiro."
```

---

## Additional notes

### arrow-arith patch

LanceDB 0.4.20 fails to compile with Rust 1.94+ due to an ambiguous method call in `arrow-arith`. The `quarter()` method now exists in multiple traits, causing an `E0034` error. The fix is to qualify the call explicitly:

```rust
// Before
DatePart::Quarter => |d| d.quarter() as i32,

// After
DatePart::Quarter => |d| ChronoDateExt::quarter(&d) as i32,
```

This is a legitimate upstream bug in `arrow-arith v51.0.0` that should be fixed there.

### After a Kiro update

If Kiro updates and the issue returns, repeat the steps above. The repo will already be cloned so you can skip that step — just fetch the new tag:

> 💡 **Tip:** You can use an AI assistant like [Claude](https://claude.ai) to automate these steps into a shell script. Just share this README and ask it to generate a `fix-kiro-lancedb.sh` that handles version detection, patching, compilation and replacement automatically. The script can also be made to check whether the installed binary actually requires AVX2 before doing anything, so it's safe to re-run after every Kiro update.

```bash
cd ~/repo/kiro-avx2-fix/lancedb
git fetch --tags
git checkout -f "tags/v<NEW_VERSION>"
cd node
npm install
RUSTFLAGS="-C target-cpu=ivybridge" cargo build --release
LANCEDB_PATH="/usr/share/kiro/resources/app/extensions/kiro.kiro-agent/node_modules/@lancedb/vectordb-linux-x64-gnu/index.node"
sudo cp "$LANCEDB_PATH" "${LANCEDB_PATH}.bak"
sudo cp ~/repo/kiro-avx2-fix/lancedb/target/release/liblancedb_node.so "$LANCEDB_PATH"
```

---

## Environment tested

| Component | Version |
|-----------|---------|
| CPU | Intel Core i7-3770K (Ivy Bridge, 2012) |
| OS | Ubuntu 24.04.4 LTS (kernel 6.17.0-19-generic) |
| Kiro | 0.11.34 (commit 7b506f30719296ba4f1aebfe383b426ffce0913e) |
| Kiro VSCode base | 1.107.1 |
| Kiro Electron | 39.6.0 |
| Kiro Node.js | 22.22.0 |
| LanceDB | 0.4.20 |
| Rust | 1.94.1 |
| Node (build) | 22.22.2 |

---

## License

MIT

