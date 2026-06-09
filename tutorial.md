# Tutorial — Image Building with Hermes Skills

6 demo prompts that build on each other to showcase the full workflow
for new users.

---

## 1. Discovery — "What images can I build?"

```bash
python3 ~/.hermes/skills/devops/image-composer-build/scripts/list-templates.py
```

Filter by keyword to narrow results instantly:

```bash
python3 ~/.hermes/skills/devops/image-composer-build/scripts/list-templates.py ros2
python3 ~/.hermes/skills/devops/image-composer-build/scripts/list-templates.py cloud
python3 ~/.hermes/skills/devops/image-composer-build/scripts/list-templates.py iso
```

**What it shows:** 60+ templates organized by OS family (Ubuntu, Debian, AZL3,
ELXR, EMT3, RCD10), each with a human-readable description and output type
(raw, ISO, qcow2, etc.). The keyword filter narrows to exactly what you need.

---

## 2. Vanilla Build — "Build a minimal Ubuntu image"

```bash
sudo -E ./image-composer-tool build image-templates/ubuntu24-x86_64-minimal-raw.yml
```

**What it shows:** The core build flow end-to-end:

1. Template loading and configuration merge
2. Package metadata fetch from all configured repos
3. Package download with progress bars
4. Chroot installation (packages, kernel, bootloader)
5. UKI generation
6. Artifact compression (`.raw.gz`)
7. Build timings summary

Result: a bootable `.raw.gz` disk image.

---

## 3. Customize — "Add packages to a base image"

```bash
python3 ~/.hermes/skills/devops/image-composer-custom/scripts/customize-template.py \
  ubuntu24-x86_64-minimal-raw.yml \
  --name my-dev-image \
  --desc "Minimal Ubuntu with dev tools" \
  --add-packages "git,vim,htop,build-essential,cmake" \
  --build
```

**What it shows:** Template customization without touching the canonical
`image-templates/` directory. The `--build` flag chains directly into the
image build.

Outputs:
- `~/.hermes/user-templates/my-dev-image.yml` — reusable custom template
- Built artifact (`.raw.gz`)

---

## 4. Customize with External Repo — "Add ROS2 to the image"

```bash
python3 ~/.hermes/skills/devops/image-composer-custom/scripts/customize-template.py \
  ubuntu24-x86_64-minimal-raw.yml \
  --name ros2-dev-image \
  --desc "Ubuntu 24.04 + ROS2 Jazzy base" \
  --add-packages "ros-jazzy-ros-base,ros-jazzy-demo-nodes-py" \
  --add-repo "http://packages.ros.org/ros2/ubuntu noble main" \
  --add-repo-key "https://raw.githubusercontent.com/ros/rosdistro/master/ros.key"
```

Then build it:

```bash
sudo -E ./image-composer-tool build ~/.hermes/user-templates/ros2-dev-image.yml
```

**What it shows:** Third-party repository injection with GPG key. The script
adds the repo to `packageRepositories`, the key is downloaded and staged in the
image. The canonical templates remain pristine.

---

## 5. Error Handling — "What happens with a typo?"

```bash
python3 ~/.hermes/skills/devops/image-composer-custom/scripts/customize-template.py \
  ubuntu24-x86_64-minimal-raw.yml \
  --name error-demo \
  --add-packages "this-package-does-not-exist" \
  --build
```

**What it shows:** The customization script has **no package validation** — it
writes whatever name you give it into the template. The build tool catches it
at dependency resolution:

```
requested package '"this-package-does-not-exist"' not found in repo
found 1 packages in request of 2
Error: pre-processing failed: failed to download image packages:
one or more requested packages not found.
```

No disk image is created — failure is fast, early, and clear.

---

## 6. Full Workflow — "Ship a custom ROS2 image with nano and iperf3"

The complete start-to-finish demo:

```bash
# 1. Find the right base template
python3 ~/.hermes/skills/devops/image-composer-build/scripts/list-templates.py ros2
```

```bash
# 2. Customize with extra tools
python3 ~/.hermes/skills/devops/image-composer-custom/scripts/customize-template.py \
  robotics-demo-ubuntu24-x86_64.yml \
  --name ros2-prod-image \
  --desc "ROS2 Jazzy + networking tools for field testing" \
  --add-packages "nano,iperf3"
```

```bash
# 3. Inspect the generated template
cat ~/.hermes/user-templates/ros2-prod-image.yml | head -30
```

```bash
# 4. Build the image
sudo -E ./image-composer-tool build ~/.hermes/user-templates/ros2-prod-image.yml
```

```bash
# 5. Verify the artifact
ls -lh workspace/ubuntu-ubuntu24-x86_64/imagebuild/ros2-prod-image/
```

---

## 7. External Repo (Generic) — "Ubuntu + Docker CE from docker.com"

This demo proves the `packageRepositories` mechanism is **not** ROS2-specific
— it works the same way for any external repo. Just change the URL, GPG key,
and package names:

```bash
# 1. Customize Ubuntu minimal with Docker's official repo
python3 ~/.hermes/skills/devops/image-composer-custom/scripts/customize-template.py \
  ubuntu24-x86_64-minimal-raw.yml \
  --name ubuntu-docker \
  --desc "Ubuntu 24.04 minimal with Docker CE from official repo" \
  --add-packages "docker-ce,docker-ce-cli,containerd.io,docker-buildx-plugin,docker-compose-plugin" \
  --add-repo "https://download.docker.com/linux/ubuntu noble stable" \
  --add-repo-key "https://download.docker.com/linux/ubuntu/gpg"
```

```bash
# 2. Add a default login user (required for SSH access)
#    Ubuntu uses 'sudo' group (not 'wheel')
python3 -c "
import yaml
path = '$HOME/.hermes/user-templates/ubuntu-docker.yml'
with open(path) as f:
    data = yaml.safe_load(f)
data.setdefault('systemConfig', {}).setdefault('users', []).append({
    'name': 'user',
    'password': 'user',
    'groups': ['sudo']
})
with open(path, 'w') as f:
    yaml.dump(data, f, default_flow_style=False)
print('user section added')
"
```

```bash
# 3. Build the image
sudo -E ./image-composer-tool build ~/.hermes/user-templates/ubuntu-docker.yml
```

```bash
# 4. Verify the artifact
ls -lh workspace/ubuntu-ubuntu24-x86_64/imagebuild/ubuntu-docker/
```

**Key takeaways:**

| Aspect | This demo | ROS2 demo (step 4) |
|--------|-----------|-------------------|
| Base template | `ubuntu24-x86_64-minimal-raw.yml` | same base |
| External repo | `download.docker.com/linux/ubuntu` | `packages.ros.org/ros2/ubuntu` |
| GPG key | Docker's GPG key | ROS2's GPG key |
| Packages | `docker-ce`, `containerd.io`, etc. | `ros-jazzy-ros-base`, etc. |
| User group | `sudo` (Ubuntu standard) | *(varies)* |
| **Mechanism** | `packageRepositories` — **identical** | `packageRepositories` — **identical** |

The only things that change between repos are:
1. The repo **URL**
2. The **GPG key** URL
3. The **package names** to install

Everything else (the YAML structure, the build pipeline, the GPG handling,
the artifact output) works exactly the same way. This mechanism supports
**any** apt-compatible external repo — Docker, ROS2, NodeSource, Microsoft,
EPEL (for RCD/Rocky), or your own internal mirror.

---

## Suggested Video Flow (3-4 minutes)

| Time | Scene | Action |
|------|-------|--------|
| 0:00 | **Discovery** | Run `list-templates.py ros2` — show the ROS2 template exists |
| 0:30 | **Customize** | Add `nano,iperf3` to the ROS2 template with one command |
| 1:00 | **Inspect output** | `cat ~/.hermes/user-templates/ros2-prod-image.yml` — highlight canonical template is untouched |
| 1:30 | **Build** | `sudo -E ./image-composer-tool build ...` — show progress bars and timestamps |
| 2:45 | **Verify** | `ls -lh workspace/.../imagebuild/ros2-prod-image/` — show the compressed artifact |
| 3:15 | **Error demo** | Run the typo demo — clean failure with no wasted time |

---

## Prerequisites

- `image-composer-tool` binary in the project directory
- `image-templates/` directory with canonical templates
- Python package `pyyaml` — `pip install pyyaml` if missing
- Hermes Agent with the `image-composer-build` and `image-composer-custom` skills
  installed under `~/.hermes/skills/devops/`
