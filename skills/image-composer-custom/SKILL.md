---
name: image-composer-custom
description: "Customize image-composer templates with extra packages and repos, saving results to ~/.hermes/user-templates/ so the canonical templates stay pristine."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [image-composer, customization, templates, devops, ubuntu, debian]
    related_skills: [image-composer-build, plan]
---

# image-composer-custom

## Overview

Create custom disk image templates by extending canonical base templates with
additional packages, repositories, and metadata — without touching the original
`image-templates/` directory.

The customized template goes to `~/.hermes/user-templates/` so it persists across
sessions and can be reused, edited, or archived independently.

## When to Use

- User needs an existing base template **plus** a few extra packages
- User wants to add a third-party repo (ROS2, Docker, etc.) to a standard image
- User wants a reusable custom variant without modifying canonical templates
- User wants to quickly iterate: customize → build → test → tweak

## Usage

```bash
python3 ~/.hermes/skills/devops/image-composer-custom/scripts/customize-template.py \
    <base_template> --name <output_name> [options]
```

### Options

| Flag | Description |
|------|-------------|
| `--name <name>` | **Required.** Output template name (without .yml) |
| `--desc <text>` | Custom description for the new template |
| `--add-packages "pkg1,pkg2"` | Comma-separated extra packages to install |
| `--add-repo "url [codename [component]]"` | Add a package repo (can use multiple times) |
| `--add-repo-key <url>` | GPG key URL for the added repos |
| `--repo-codename <name>` | Default codename for repos (default: noble) |
| `--repo-component <name>` | Default component for repos (default: main) |
| `--list-base` | List all available base templates |
| `--list` | List existing user-customized templates |
| `--build` | Build the image immediately after customization |

### Examples

**Add ROS2 Jazzy to the minimal Ubuntu 24.04 raw image:**

```bash
python3 customize-template.py ubuntu24-x86_64-minimal-raw.yml \
    --name my-ros2-image \
    --desc "Ubuntu 24.04 minimal + ROS2 Jazzy" \
    --add-packages "ros-jazzy-ros-base,ros-jazzy-demo-nodes-py" \
    --add-repo "http://packages.ros.org/ros2/ubuntu noble main" \
    --add-repo-key "https://raw.githubusercontent.com/ros/rosdistro/master/ros.key"
```

**Add dev tools to the DL Streamer image and build:**

```bash
python3 customize-template.py ubuntu24-x86_64-dlstreamer.yml \
    --name my-dlstreamer-dev \
    --desc "DL Streamer with dev tools" \
    --add-packages "vim,git,htop,build-essential,cmake" \
    --build
```

**Add Docker repo to a Debian 13 minimal image:**

```bash
python3 customize-template.py debian13-x86_64-minimal-raw.yml \
    --name debian-docker \
    --add-packages "docker-ce,docker-ce-cli,containerd.io" \
    --add-repo "https://download.docker.com/linux/debian bookworm stable" \
    --add-repo-key "https://download.docker.com/linux/debian/gpg"
```

## Workflow

### 1. Find a Base Template

Use the discoverability script from the `image-composer-build` skill:

```bash
python3 ~/.hermes/skills/devops/image-composer-build/scripts/list-templates.py
```

Or list base templates directly:

```bash
python3 customize-template.py --list-base
```

### 2. Customize

Choose a base template and specify your additions:

```bash
python3 customize-template.py <base> --name <custom> --add-packages "pkg1,pkg2"
```

The script:
1. Copies the base template to `~/.hermes/user-templates/<name>.yml`
2. Adds extra packages (deduplicated)
3. Adds extra repos with optional GPG key
4. Updates the image name and metadata description

### 3. Review the Custom Template

```bash
cat ~/.hermes/user-templates/<name>.yml
```

### 4. Build

Build from the user template directory:

```bash
cd /data/hermes/osic && sudo -E ./image-composer-tool build /home/user/.hermes/user-templates/<name>.yml
```

Or automatically with `--build`.

### 5. List and Manage User Templates

```bash
# List all
python3 customize-template.py --list

# Delete a template
rm ~/.hermes/user-templates/<name>.yml
```

## How It Works

The script performs four operations on the base template YAML:

1. **Metadata update** — sets `image.name` and `metadata.description` to the
   custom name
2. **Package injection** — appends to `systemConfig.packages`, skipping
   duplicates
3. **Repo injection** — appends to `packageRepositories`, with optional GPG
   key URL, skipping duplicates by URL
4. **Output** — writes the modified template to `~/.hermes/user-templates/`

The canonical `image-templates/` directory is **never modified**.

## Common Pitfalls

1. **Repo GPG key failures** — if the image-composer-tool rejects a GPG key,
   use `pkey: "[trusted=yes]"` instead of a key URL. You can edit the generated
   file in `~/.hermes/user-templates/` manually after creation.

2. **Package names must match the repo** — if you add a repo but the package
   name is wrong, the build will fail at the "resolving dependencies" stage.
   Check the repo's Packages listing first.

3. **Disk size** — extra packages need space. If the base template is 4 GiB
   and you're adding ROS2, Docker, or large SDKs, edit `disk.size` in the
   custom template before building.

4. **Build from user-templates** — the build command must reference the full
   path: `sudo -E ./image-composer-tool build /home/user/.hermes/user-templates/<name>.yml`
   (relative paths won't work if the user templates dir isn't under cwd).

## Verification Checklist

- [ ] `python3 customize-template.py --list` shows the new template
- [ ] Template YAML parses: `python3 -c "import yaml; yaml.safe_load(open('PATH'))"`
- [ ] Extra packages appear in `systemConfig.packages`
- [ ] Extra repos appear in `packageRepositories`
- [ ] Build succeeds with the user template path
