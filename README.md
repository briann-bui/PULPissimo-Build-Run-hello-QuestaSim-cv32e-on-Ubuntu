# PULPissimo – Build & Run "hello" (QuestaSim, cv32e) on Ubuntu

This README walks you through installing **Bender**, building the **RISC‑V GNU toolchain** used by PULPissimo, setting up **QuestaSim** environment variables, and running the `hello` example on the **cv32e40p** core. It also includes a troubleshooting note for a common `vopt_tb` error (with a screenshot placeholder) and the quick workaround.

> Tested environment: Ubuntu 20.04/22.04, QuestaSim 2021.2_1, cv32e40p configuration.

---

## 0) Prerequisites (Ubuntu)

Make sure you have the standard build dependencies:

```bash
sudo apt-get update
sudo apt-get install -y \
  autoconf automake autotools-dev curl python3 \
  libmpc-dev libmpfr-dev libgmp-dev gawk build-essential \
  bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
```

---

## 1) Install **Bender** locally

Place the archive in the PULPissimo utils bin folder and make it executable.

```bash
cd /home/minhnhut/Downloads/pulpissimo-main/utils/bin
# Extract
tar -xzf bender-0.28.0-x86_64-linux-gnu-ubuntu18.04.tar.gz

# Make executable
chmod +x bender

# Verify installation
./bender --version
```

You should see something like:

```
Bender 0.28.0
```

> If you want `bender` on your PATH, you can symlink it to a directory already on PATH or export a PATH entry in your shell *session* (avoid permanent changes unless needed).

---

## 2) Build the **RISC‑V GNU toolchain** (PULP fork)

Clone the PULP‑platform toolchain and build into `/opt/riscv`.

```bash
# Clone (recursive is important)
cd ~
git clone --recursive https://github.com/pulp-platform/riscv-gnu-toolchain.git

# 1) go to repo root
cd ~/riscv-gnu-toolchain

# 2) out-of-tree build dir
mkdir -p build && cd build

# 3) configure (install to /opt/riscv)
# NOTE: upstream often does NOT support rv32imfcxpulpv3 fully.
# Use the first line; if it fails, fall back to rv32imac.
../configure --prefix=/opt/riscv --with-arch=rv32imfcxpulpv3 --with-abi=ilp32 --enable-multilib \
  || ../configure --prefix=/opt/riscv --with-arch=rv32imac --with-abi=ilp32 --enable-multilib

# 4) build & install
make -j"$(nproc)"
sudo make install
```

> After installation, the toolchain binaries will be under `/opt/riscv/bin`.

---

## 3) Set up environment for **QuestaSim** run (temporary for this session)

> **Important**: Export these variables **only in the current shell** when running PULPissimo. Do **not** put them permanently into `~/.bashrc` to avoid unexpected licensing side effects.

```bash
# Set the path to the Questasim simulation assets inside PULPissimo
export VSIM_PATH="$HOME/pulpissimo/target/sim/questasim"

# Path to your QuestaSim binary
export VSIM="$HOME/Downloads/mentor_graphic/questasim/install/questasim/linux_x86_64/vsim"

# RISC-V toolchain path
export PULP_RISCV_GCC_TOOLCHAIN="/opt/riscv"
export PATH="$PULP_RISCV_GCC_TOOLCHAIN/bin:$PATH"
```

Optional quick checks:

```bash
"$VSIM" -version || true
riscv32-unknown-elf-gcc --version
```

---

## 4) Prepare PULPissimo for cv32e40p and run the **hello** example

From the **root** of your PULPissimo repo:

```bash
cd ~/pulpissimo

# Load cv32e40p platform settings
source sw/pulp-runtime/configs/pulpissimo_cv32.sh

# Go to the hello example
cd pulp-rt-examples/hello

# Clean, build, and run
make clean && make all && make run
```

If everything is correct, QuestaSim should invoke the `vopt_tb` optimized design and you’ll see UART output for the `hello` application in the transcript.

---

## 5) Verifying build artifacts (optional diagnostics)

If you want to confirm the optimized design is present in the built library:

```bash
# After a first PULPissimo optimize/build, the optimized design is named vopt_tb
cd ~/pulpissimo/pulp-rt-examples/hello/build
"$VSIM" -64 -c -do "vmap -modelsimini ./modelsim.ini work ./work; vdir; quit -f" | grep -i vopt_tb || true
# Expected to print something like: "# OPTIMIZED DESIGN vopt_tb"
```

---

## 6) Troubleshooting

### A) Symlink creation errors: `ln: failed to create symbolic link ... File exists`

Some Makefile rules create symlinks under `pulp-rt-examples/hello/build`:

* `modelsim.ini` → `$VSIM_PATH/modelsim.ini`
* `boot` → `$VSIM_PATH/boot`
* `tcl_files` → `$VSIM_PATH/tcl_files`
* `waves` → `$VSIM_PATH/waves`

If these already exist (or were accidentally created as normal folders), `ln -s` can fail. Fix:

```bash
cd ~/pulpissimo/pulp-rt-examples/hello/build
rm -rf boot tcl_files waves modelsim.ini

ln -s "$VSIM_PATH/modelsim.ini" modelsim.ini
ln -s "$VSIM_PATH/boot" boot
ln -s "$VSIM_PATH/tcl_files" tcl_files
ln -s "$VSIM_PATH/waves" waves
```

Also ensure your `work` library points to the optimized one:

```bash
rm -f work && ln -s ~/pulpissimo/build/questasim/work work
```

Then re-run from the example directory:

```bash
cd ~/pulpissimo/pulp-rt-examples/hello
make run
```

### B) **Common QuestaSim error**: `Failed to find design unit vopt_tb`

You may see an error like this (screenshot placeholder below):

```
# vsim -c -quiet vopt_tb -t ps
# ** Error: (vopt-13130) Failed to find design unit vopt_tb.
#         Searched libraries:
#             work
# Error loading design
```

**Quick workaround (continue the run even if a prior symlink step fails):**

```bash
cd ~/pulpissimo/pulp-rt-examples/hello
make -i run
```

> The `-i` tells `make` to **ignore non-fatal errors** (e.g., re‑creating an already existing symlink). This is safe when your `build/` symlinks already point to the correct locations and the optimized design `vopt_tb` exists.

If the issue persists, ensure the optimized design is really present and that the local `modelsim.ini` maps `work` to the correct `./work`:

```bash
# Check optimized design
cd ~/pulpissimo/pulp-rt-examples/hello/build
"$VSIM" -64 -c -do "vmap -modelsimini ./modelsim.ini work ./work; vdir; quit -f" | grep -i vopt_tb || true

# If not shown, rebuild the QuestaSim libs from PULPissimo root (once):
cd ~/pulpissimo/build/questasim
# examine optimize.log to confirm creation of vopt_tb
grep -n "Optimized design name is vopt_tb" optimize.log || true
```

If needed, re‑generate/optimize from PULPissimo’s standard flow (e.g., `make` in the sim build dir) and ensure `hello/build/work` links to that optimized `work`.

---

## 7) (Optional) Makefile hardening for symlinks

If you frequently hit `ln -s ... File exists` and prefer a permanent fix, you can locally change the symlink commands to force overwrite by adding `-f` (and `-n`):

```bash
# Example (edit carefully):
sed -i 's/ln -s /ln -snf /g' ~/pulpissimo/sw/pulp-runtime/rules/pulpos/default_rules.mk
```

This ensures repeated runs won’t fail when the symlinks already exist. (Keep in mind this modifies your local copy.)

---

## Appendix – Sanity checklist

* `bender --version` prints `0.28.0` (or the version you installed)
* `riscv32-unknown-elf-gcc --version` prints from `/opt/riscv/bin`
* `echo $VSIM_PATH` → `~/pulpissimo/target/sim/questasim`
* `echo $VSIM` points to your `vsim` binary
* `source sw/pulp-runtime/configs/pulpissimo_cv32.sh` done before building examples
* `hello/build/modelsim.ini` exists and `work` library resolves to `./work`
* `hello/build/work` is a symlink to `~/pulpissimo/build/questasim/work`
* `vdir` shows `OPTIMIZED DESIGN vopt_tb`

---

## Screenshot placeholder (error & fix)

Include your error screenshot here for future reference:

* **Error**: `Failed to find design unit vopt_tb` in QuestaSim transcript
* **Immediate Fix**: Run from example root

```bash
cd ~/pulpissimo/pulp-rt-examples/hello
make -i run
```

That’s it. Happy simulating! 🚀
