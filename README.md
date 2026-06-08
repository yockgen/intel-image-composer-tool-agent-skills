# ROS2 Image Builder — Skills & Template Migration

This project uses two Hermes Agent skills to build and customize OS disk
images with the `image-composer-tool`:

- **`image-composer-build`** — Build disk images from canonical YAML templates
- **`image-composer-custom`** — Extend base templates with extra packages/repos
  without modifying the originals
- **`ros2-with-tools`** — Custom user template: ROS2 Jazzy + nano + iperf3

---

## Project Structure

```
/data/hermes/osic/
├── image-composer-tool        # Build binary
├── image-templates/           # Canonical base templates (60+)
├── config/                    # OS provider configs
├── skills/                    # <-- checked into git (this project)
│   ├── image-composer-build/
│   │   ├── SKILL.md
│   │   ├── scripts/list-templates.py
│   │   └── references/gpg-key-workaround.md
│   ├── image-composer-custom/
│   │   ├── SKILL.md
│   │   └── scripts/customize-template.py
│   └── user-templates/
│       └── ros2-with-tools.yml
├── README.md
└── tutorial.md
```

### Skills (in `skills/` directory)

| Path | Contents |
|------|----------|
| `skills/image-composer-build/SKILL.md` | Skill definition — build workflow, pitfalls, one-shot recipes |
| `skills/image-composer-build/scripts/list-templates.py` | Discover & filter available base templates |
| `skills/image-composer-build/references/gpg-key-workaround.md` | Fix for GPG key verification failures in 3rd-party repos |
| `skills/image-composer-custom/SKILL.md` | Skill definition — customization workflow, options, examples |
| `skills/image-composer-custom/scripts/customize-template.py` | Injects packages/repos into a base template |

### User Template (in `skills/user-templates/`)

| File | Description |
|------|-------------|
| `skills/user-templates/ros2-with-tools.yml` | Ubuntu 24.04 + ROS2 Jazzy + nano + iperf3 |

---

## Migration to Another Machine

### Prerequisites on the target machine

- Hermes Agent installed
- `image-composer-tool` binary in the project directory
- `image-templates/` directory with canonical templates (from the build tool)
- Python package `pyyaml` (for the scripts) — `pip install pyyaml` if missing

### Step 1: Archive on the source machine

```bash
tar czf hermes-skills.tar.gz -C /data/hermes/osic skills/
```

### Step 2: Transfer the archive

```bash
# Examples:
scp hermes-skills.tar.gz user@target-machine:~/
# or rsync, USB, cloud storage, etc.
```

### Step 3: Extract on the target machine

```bash
# Extract into the project directory
mkdir -p /path/to/project
tar xzf hermes-skills.tar.gz -C /path/to/project
```

### Step 4: Install skills into Hermes (if you want them in Hermes directly)

To use them with `skill_view()` / `skills_list()` in Hermes:

```bash
cp -r /path/to/project/skills/image-composer-build ~/.hermes/skills/devops/
cp -r /path/to/project/skills/image-composer-custom ~/.hermes/skills/devops/
cp -r /path/to/project/skills/user-templates ~/.hermes/user-templates/
```

Or just reference the files directly from the project directory.

---

## Quick Reference

### Build the custom ROS2 image

```bash
cd /data/hermes/osic
sudo -E ./image-composer-tool build ~/.hermes/user-templates/ros2-with-tools.yml
```

### Create a new custom image variant

```bash
python3 ~/.hermes/skills/devops/image-composer-custom/scripts/customize-template.py \
  <base-template.yml> \
  --name my-custom-image \
  --add-packages "pkg1,pkg2" \
  --add-repo "http://example.com/repo noble main" \
  --build
```

### List available base templates

```bash
python3 ~/.hermes/skills/devops/image-composer-build/scripts/list-templates.py
# Filter by keyword:
python3 ~/.hermes/skills/devops/image-composer-build/scripts/list-templates.py ros2
```
