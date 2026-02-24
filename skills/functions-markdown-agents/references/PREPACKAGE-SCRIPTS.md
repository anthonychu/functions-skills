# prepackage.sh Scripts

The `prepackage.sh` hook assembles the deployment package in `infra/tmp/` by copying agent files and merging `infra/assets/` (runtime files). The script varies by layout.

**Important**: `infra/assets/` is copied **after** the user's files, so runtime files like `host.json`, `function_app.py`, and `.funcignore` will overwrite any user files with the same name. This is by design — those files are managed by the runtime. User dependencies go in `requirements.txt` (which gets merged with `extra-requirements.txt` automatically).

## Layout A: `src/` Subfolder

This is the default from the [sample repo](https://github.com/Azure-Samples/functions-markdown-agent). It copies `src/` into `infra/tmp/` and overlays `infra/assets/`.

```bash
#!/bin/bash
set -e

TMP_DIR="infra/tmp"

if [ -d "$TMP_DIR" ]; then
    rm -rf "$TMP_DIR"/*
else
    mkdir -p "$TMP_DIR"
fi

echo "Copying src/ contents to $TMP_DIR..."
cp -r src/. "$TMP_DIR/"

echo "Copying infra/assets contents to $TMP_DIR..."
cp -r infra/assets/. "$TMP_DIR/"

if [ -f "$TMP_DIR/extra-requirements.txt" ]; then
    echo "Merging extra-requirements.txt into requirements.txt..."
    echo "" >> "$TMP_DIR/requirements.txt"
    cat "$TMP_DIR/extra-requirements.txt" >> "$TMP_DIR/requirements.txt"
    rm "$TMP_DIR/extra-requirements.txt"
fi

echo "prepackage.sh completed successfully."
```

## Layout B: Root Layout

Replace `infra/hooks/prepackage.sh` with this version. It copies agent files from the project root into `infra/tmp/`, skipping infrastructure and build artifacts:

```bash
#!/bin/bash
set -e

TMP_DIR="infra/tmp"

if [ -d "$TMP_DIR" ]; then
    rm -rf "$TMP_DIR"/*
else
    mkdir -p "$TMP_DIR"
fi

echo "Copying agent project files to $TMP_DIR..."

# Copy all root-level files and directories EXCEPT infra, build artifacts, and git
for item in * .[!.]*; do
    [ -e "$item" ] || continue
    case "$item" in
        infra|azure.yaml|tmp|.git|.gitignore|.azure|.DS_Store|__pycache__)
            echo "  Skipping $item"
            ;;
        *)
            cp -r "$item" "$TMP_DIR/"
            ;;
    esac
done

# Copy infra/assets into infra/tmp (overwriting any conflicting files)
echo "Copying infra/assets contents to $TMP_DIR..."
cp -r infra/assets/. "$TMP_DIR/"

# Merge extra-requirements.txt into requirements.txt
if [ -f "$TMP_DIR/extra-requirements.txt" ]; then
    echo "Merging extra-requirements.txt into requirements.txt..."
    echo "" >> "$TMP_DIR/requirements.txt"
    cat "$TMP_DIR/extra-requirements.txt" >> "$TMP_DIR/requirements.txt"
    rm "$TMP_DIR/extra-requirements.txt"
fi

echo "prepackage.sh completed successfully."
```

### Customizing Exclusions

If the project has additional directories that should not be deployed (e.g., `docs/`, `test/`, large data files), add them to the `case` statement:

```bash
        infra|azure.yaml|tmp|.git|.gitignore|.azure|.DS_Store|__pycache__|docs|test)
```

Files that aren't part of the agent (e.g., `README.md`, `LICENSE`) will be copied but are harmless — the `.funcignore` in `infra/assets/` controls what goes into the final deployment package.
