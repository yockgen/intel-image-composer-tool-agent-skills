# Hermes Skills for Image Builder

This project contains Hermes Agent skills to build and customize OS disk
images using the **image-composer-tool** — a declarative YAML-based image
builder. It supports 8+ OS families (Ubuntu, Debian, RCD10/Rocky Linux,
AZL3, ELXR, EMT3, and others) and produces raw, vhdx, qcow2, vmdk, iso,
and initrd artifacts.

The build tool itself lives at:
**https://github.com/open-edge-platform/image-composer-tool**

The skills here are checked into git so they can be versioned, shared, and
migrated between machines.

---

## Skills

| Skill | Purpose |
|-------|---------|
| **`image-composer-build`** | Build disk images from YAML templates — full workflow, pitfalls, one-shot recipes |
| **`image-composer-custom`** | Extend base templates with extra packages and external repos without touching the originals |

Both skills live under `skills/` and support the `skill_view()` / `skills_list()`
commands in Hermes Agent when installed to `~/.hermes/skills/devops/`.

---

## Project Structure

```
.
├── skills/
│   ├── image-composer-build/
│   │   ├── SKILL.md                         # Build workflow, pitfalls, recipes
│   │   ├── scripts/list-templates.py        # Discover & filter base templates
│   │   └── references/
│   │       ├── gpg-key-workaround.md        # Fix GPG verification failures
│   │       └── non-fatal-chroot-warnings.md # systemd-boot EFI warnings
│   └── image-composer-custom/
│       ├── SKILL.md                         # Customization workflow, options, examples
│       ├── scripts/customize-template.py    # Injects packages/repos into a base template
│       └── references/
│           ├── error-demo-non-existent-package.md
│           ├── external-repo-docker-test.md # Tested: Ubuntu + Docker CE from docker.com
│           └── rcd10-customization-example.md # Tested: RCD10 + nano + iperf3
├── user-templates/                          # Custom templates (`.gitignored`, per-machine)
├── tutorial.md                              # 7-step walkthrough from discovery to build
├── README.md
```

User-customized templates (`~/.hermes/user-templates/`) are per-machine,
not checked in.

---

## Quick Start

```bash
# 1. List available base templates
python3 skills/image-composer-build/scripts/list-templates.py

# 2. Customize a base template with extra packages
python3 skills/image-composer-custom/scripts/customize-template.py \
  ubuntu24-x86_64-minimal-raw.yml \
  --name my-dev-image \
  --desc "Ubuntu 24.04 + dev tools" \
  --add-packages "git,vim,htop"

# 3. Add a default login user (edit the generated YAML, or use the snippet below)
python3 -c "
import yaml
path = '$HOME/.hermes/user-templates/my-dev-image.yml'
with open(path) as f:
    data = yaml.safe_load(f)
data.setdefault('systemConfig', {}).setdefault('users', []).append({
    'name': 'user', 'password': 'user', 'groups': ['sudo']
})
with open(path, 'w') as f:
    yaml.dump(data, f, default_flow_style=False)
"

# 4. Build
sudo -E ./image-composer-tool build ~/.hermes/user-templates/my-dev-image.yml
```

---

## External Repositories (Generic)

You can add packages from **any** external apt or rpm repository — not just
the default OS repos. The `--add-repo` / `--add-repo-key` flags work the
same way for Docker, ROS2, EPEL, NodeSource, Microsoft, or your own internal
mirror. Only the URL, GPG key, and package names change.

```bash
python3 skills/image-composer-custom/scripts/customize-template.py \
  ubuntu24-x86_64-minimal-raw.yml \
  --name my-docker-image \
  --add-packages "docker-ce,docker-ce-cli,containerd.io" \
  --add-repo "https://download.docker.com/linux/ubuntu noble stable" \
  --add-repo-key "https://download.docker.com/linux/ubuntu/gpg"
```

See `tutorial.md` (step 7) for a full walkthrough.

---

## Requirements

- **image-composer-tool** binary — get it from:
  https://github.com/open-edge-platform/image-composer-tool
- **Python 3** + **pyyaml** — `pip install pyyaml` if missing
- **Hermes Agent** (optional) — if using `skill_view()` to load the skills
- **Root/sudo** — the build tool needs loop device access for disk images

---

## Migration to Another Machine

```bash
# Archive skills (everything under skills/ is portable)
tar czf image-builder-skills.tar.gz skills/ tutorial.md README.md

# Transfer & extract on the target
tar xzf image-builder-skills.tar.gz -d /path/to/project

# Install into Hermes (optional)
cp -r skills/image-composer-build ~/.hermes/skills/devops/
cp -r skills/image-composer-custom ~/.hermes/skills/devops/
```

On the target machine you also need the `image-composer-tool` binary and
the `image-templates/` directory from the upstream repo.
