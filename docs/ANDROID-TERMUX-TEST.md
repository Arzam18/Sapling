# Sapling Android ARM64 Termux test

Copy the downloaded GitHub Actions artifact binary to:

`/storage/emulated/0/DroidFish/uci/Sapling`

Then run the following complete command in Termux. It copies the executable into Termux-private storage and performs explicit Vector128 and Scalar UCI smoke tests:

```bash
pkg update -y
pkg install -y coreutils binutils file
termux-setup-storage

set -euo pipefail
SRC="/storage/emulated/0/DroidFish/uci/Sapling"
ROOT="$HOME/sapling-android-test"
BIN="$ROOT/Sapling"
mkdir -p "$ROOT"

if [ ! -f "$SRC" ]; then
    echo "ERROR: Place the downloaded Sapling binary at: $SRC"
    exit 1
fi

cp "$SRC" "$BIN"
chmod 700 "$BIN"

file "$BIN"
sha256sum "$BIN"
readelf -h "$BIN" | grep -E 'Class:|Machine:'

run_test() {
    local backend="$1"
    local log="$ROOT/uci-${backend}.log"
    echo
    echo "===== Sapling backend: $backend ====="
    {
        echo uci
        sleep 1
        echo isready
        sleep 1
        echo "position startpos"
        echo "go depth 10"
        sleep 30
        echo quit
    } | timeout 40s env SAPLING_BACKEND="$backend" "$BIN" 2>&1 | tee "$log"

    grep -F "NNUE backend: $backend" "$log" >/dev/null || {
        echo "ERROR: expected backend $backend was not reported"
        return 1
    }
    grep -E '^bestmove [a-h][1-8][a-h][1-8][qrbn]?' "$log" >/dev/null || {
        echo "ERROR: no legal-looking bestmove line found; inspect $log"
        return 1
    }
}

run_test Vector128
run_test Scalar

echo
echo "Both backend UCI smoke tests completed. Logs: $ROOT"
```

This checks that the same binary can force each backend and produce a move. It is not a strength test. Use identical Hash, Threads, time control and opening book for the subsequent paired tournament comparison.
