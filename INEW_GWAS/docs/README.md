# INEW_GWAS — Teaching README (Status: Step 0)

**Purpose.** This repository is a stepwise, command-first, config-driven reconstruction of the INEW GWAS pipeline, aligned with SU-PBL artifact names and logic. The README is the primary executable document; every external tool is invoked as an explicit shell command. All parameters will flow from `config/config.yaml` via a small YAML→env bridge.

**Ground rules.**
- Single source of truth: only `config/config.yaml` will hold parameters and paths.
- Command-first: PLINK, awk, Rscript, TASSEL, STRUCTURE, etc. are called directly (no opaque wrappers).
- Modularity: each numbered block documents inputs, outputs, rationale, and commands.
- Idempotence and safety: steps warn before overwriting and are copy-paste runnable.

> **Do not proceed beyond Step 0.** Wait for “Step 1” instructions before adding `config/config.yaml` or any Makefile rules.

**Repository layout (created in Step 0).**
# 1) Create the helper file and fill it with code
cat > INEW_GWAS/bin/config_env <<'EOF'
#!/usr/bin/env bash
# Export scalar keys from a YAML file as environment variables.
# Usage:
#   eval "$(bin/config_env)"                  # reads config/config.yaml
#   eval "$(bin/config_env path/to/file.yaml)"
#   bin/config_env --help
#
# Notes:
# - Requires: python3 with PyYAML (yaml) available.
# - Flattens nested keys with underscores, uppercases, and prefixes with CFG_.
#   Example: tools.plink -> CFG_TOOLS_PLINK
# - If config is missing, prints a notice and exits 0 (no exports).

set -eu

if [ "${1-}" = "--help" ]; then
  cat <<'HLP'
config_env — print `export` lines for YAML scalar keys.

Examples:
  eval "$(bin/config_env)"                 # from config/config.yaml
  eval "$(bin/config_env other.yaml)"     # from a specific file

Conventions:
  - Variables are prefixed with CFG_ and UPPER_SNAKE_CASED.
  - Nested maps are flattened with underscores (e.g., a.b.c -> CFG_A_B_C).
Requirements:
  - python3 and PyYAML (`pip install pyyaml`)

HLP
  exit 0
fi

CONFIG_PATH="${1:-config/config.yaml}"
if [ ! -f "$CONFIG_PATH" ]; then
  echo "config_env: no config file found at '$CONFIG_PATH' (nothing to export)." >&2
  exit 0
fi

python3 - "$CONFIG_PATH" <<'PY'
import sys, os, re, shlex
try:
    import yaml
except Exception as e:
    sys.stderr.write("config_env: PyYAML not available — install with `pip install pyyaml`.\n")
    sys.exit(1)

def flatten(d, prefix=()):
    if isinstance(d, dict):
        for k, v in d.items():
            key = re.sub(r'[^A-Za-z0-9_]', '_', str(k))
            yield from flatten(v, prefix + (key,))
    elif isinstance(d, (list, tuple)):
        for i, v in enumerate(d):
            yield from flatten(v, prefix + (str(i),))
    else:
        yield ("_".join(prefix), d)

path = sys.argv[1]
with open(path, "r", encoding="utf-8") as fh:
    data = yaml.safe_load(fh) or {}

for key, val in flatten(data):
    # export only scalars
    if isinstance(val, (str, int, float, bool)) or val is None:
        name = "CFG_" + re.sub(r'__+', '_', key.upper())
        sval = "" if val is None else str(val)
        print(f"export {name}={shlex.quote(sval)}")
PY
