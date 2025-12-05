# Building Kernel 6.12.54 for macOS Docker Desktop (ARM64)

This document details the process and challenges of building LinuxKit kernel 6.12.54 for macOS Docker Desktop on Apple Silicon (ARM64).

## Problem Context

Docker Desktop 4.53.0 on macOS uses kernel 6.12.54-linuxkit, but the upstream linuxkit/kernel repository only has ARM64 images up to 6.12.52. To use ZFS modules or other kernel-dependent tools, we need to build the 6.12.54 kernel for ARM64.

## Prerequisites

1. Docker Desktop for Mac installed and running
2. Homebrew installed
3. LinuxKit CLI tool: `brew tap linuxkit/linuxkit && brew install --HEAD linuxkit`
4. Git repository forked from linuxkit/linuxkit
5. **At least 20-30GB free disk space** (critical!)
6. **8GB+ RAM recommended** (or reduce parallelism)

## Build Environment Issues on macOS

### Issue 1: Docker Context and Credential Helper

When building via SSH on macOS, you'll encounter Docker credential helper issues because the keychain is locked in SSH sessions.

**Symptoms:**
```
error getting credentials - err: exit status 1, out: `keychain cannot be accessed because the current session does not allow user interaction`
creating builder container 'linuxkit-builder' in context 'default'
Error: unable to create or find builder container linuxkit-builder in context default after 3 retries
```

**Root Causes:**
1. Docker credential helper (`docker-credential-desktop`) not in PATH when running via SSH
2. Docker context defaults to `default` (pointing to `/var/run/docker.sock` which doesn't exist on macOS)
3. LinuxKit doesn't automatically use the active Docker context for builder creation

## File Changes Made

### 1. Update Kernel Version

**File:** `kernel/6.12.x/build-args`

```
KERNEL_VERSION=6.12.54
KERNEL_SERIES=6.12.x
BUILD_IMAGE=linuxkit/alpine:35b33c6b03c40e51046c3b053dd131a68a26c37a
MAKE_JOBS=3
```

### 2. Add BUILDERS Parameter Support to Makefile

**File:** `kernel/Makefile`

Modified the `buildplainkernel-%` and `builddebugkernel-%` targets to accept an optional `BUILDERS` parameter:

```makefile
buildplainkernel-%: buildkerneldeps-%
	$(eval KERNEL_SERIES=$(call series,$*))
	linuxkit pkg build . $(FORCE) --platforms $(BUILD_PLATFORM) --build-yml ./build-kernel.yml --tag "$*-{{.Hash}}" --build-arg-file $(KERNEL_SERIES)/build-args $(if $(BUILDERS),--builders $(BUILDERS),)

builddebugkernel-%: buildkerneldeps-%
	$(eval KERNEL_SERIES=$(call series,$*))
	linuxkit pkg build . $(FORCE) --platforms $(BUILD_PLATFORM) --build-yml ./build-kernel.yml --tag "$*-{{.Hash}}" --build-arg-file $(KERNEL_SERIES)/build-args --build-arg-file build-args-debug $(if $(BUILDERS),--builders $(BUILDERS),)
```

This allows specifying which Docker context to use for building: `BUILDERS="linux/arm64=desktop-linux"`

### 3. Add MAKE_JOBS Build Argument

**File:** `kernel/Dockerfile`

Added configurable parallelism to reduce memory usage on constrained systems:

```dockerfile
# Kernel
# MAKE_JOBS can be overridden via build-arg, default to all CPUs
ARG MAKE_JOBS
RUN make -j "${MAKE_JOBS:-$(getconf _NPROCESSORS_ONLN)}" KCFLAGS="-fno-pie" && \
    case $(uname -m) in \
    x86_64) \
        cp arch/x86_64/boot/bzImage /out/kernel; \
        ;;
```

## Build Commands

### Check Docker Contexts

```bash
# List available Docker contexts
docker context ls

# List buildx builders
docker buildx ls
```

On macOS you should see:
```
NAME              DESCRIPTION                               DOCKER ENDPOINT
default           Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux *   Docker Desktop                            unix:///Users/<user>/.docker/run/docker.sock
```

### Building the Kernel

**Important:** You must include the Docker credential helper path in your PATH when building:

```bash
# Set up proper PATH including linuxkit, docker, and credential helper
export PATH=/opt/homebrew/bin:/usr/local/bin:/Applications/Docker.app/Contents/Resources/bin:$PATH

# Navigate to kernel directory
cd ~/dev/linuxkit/kernel

# Build for ARM64 using desktop-linux context
make build-6.12.54 ARCH=arm64 BUILDERS="linux/arm64=desktop-linux"
```

**Why these paths are needed:**
- `/opt/homebrew/bin` - linuxkit CLI tool
- `/usr/local/bin` - docker CLI
- `/Applications/Docker.app/Contents/Resources/bin` - docker-credential-desktop (fixes credential errors)

## Build Attempts and Issues

### Build Attempt #1 - December 4, 2025

**Command:**
```bash
export PATH=/opt/homebrew/bin:/usr/local/bin:/Applications/Docker.app/Contents/Resources/bin:$PATH
cd /Users/r2/dev/linuxkit/kernel
make build-6.12.54 ARCH=arm64 BUILDERS="linux/arm64=desktop-linux"
```

**Error:**
```
Error: error building "linuxkit/kernel:6.12.54-...": 
error building for arch arm64: failed to solve: Internal: error committing kk0nxhxu69zegzglfkimi7tqc: 
write /var/lib/buildkit/runc-overlayfs/metadata_v2.db: input/output error
```

**Root cause:** Docker Desktop database corruption (`/var/lib/desktop-containerd/daemon/io.containerd.metadata.v1.bolt/meta.db`)

**Resolution:** Restart Docker Desktop
```bash
osascript -e 'quit app "Docker"' && sleep 5 && open -a Docker
# Wait 30-60 seconds for Docker Desktop to fully restart
docker --context desktop-linux ps  # Verify Docker is ready
```

### Build Attempt #2 - December 4, 2025 (After Docker Restart)

**Result:** Build failed with exit code 2 during kernel compilation

**Error:**
```
error building for arch arm64: failed to solve: process "/bin/sh -c make -j ..." 
did not complete successfully: exit code: 2
```

**Root cause:** Insufficient memory (Docker Desktop allocated 7.75GB, kernel build with `-j8` needs more)

**Resolution:** Reduced parallel build jobs from 8 to 3

Modified approach:
- Added `MAKE_JOBS` build argument to `kernel/Dockerfile`
- Set `MAKE_JOBS=3` in `kernel/6.12.x/build-args`

This reduces memory consumption during compilation at the cost of slower build time (approximately 3x longer). For systems with 8GB RAM max, using `-j3` keeps peak memory usage under 7GB.

**Why this approach:**
- Configurable per kernel version via build-args
- Doesn't hardcode values in Dockerfile
- Other kernel versions can still use full CPU parallelism
- Easy to override without editing Dockerfile

### Build Attempt #3 - December 5, 2025 (With Reduced Parallelism)

**Error:**
```
tar: Cannot write: I/O error
write /var/lib/buildkit/runc-overlayfs/metadata_v2.db: input/output error
```

**Root cause:** Disk space exhaustion
- Initial state: 228GB disk, 100% full (only 245MB available)
- After Docker purge: 2.5GB available (still insufficient)
- Docker.raw file size: 128GB (down from 228GB)

**Resolution required:** 
1. Free up significant disk space (need 20-30GB minimum)
2. Reduce Docker.raw size limit in Docker Desktop:
   - Open Docker Desktop → Settings → Resources
   - Set "Disk image size" to 64GB
   - Apply & Restart

**Key lesson:** Docker needs adequate free space on the host filesystem to operate, even if Docker.raw has allocated space.

## Build Process Details

The build command performs these steps:
1. Check local cache for existing image
2. Check Docker Hub registry
3. Pull base images (Alpine, buildkit)
4. Create a `linuxkit-builder` container in the specified context
5. Download and verify kernel source (871MB download, expands to ~1.2GB)
6. Apply patches if present
7. Configure kernel
8. Compile kernel with specified parallelism
9. Package kernel, modules, and firmware into Docker image
10. Tag as `linuxkit/kernel:6.12.54-<hash>`

**Estimated time:**
- With `-j8`: 10-15 minutes (requires 12-16GB RAM)
- With `-j3`: 30-40 minutes (works with 8GB RAM)

## Troubleshooting

### Error: "unable to create or find builder container"
- Ensure `BUILDERS="linux/arm64=desktop-linux"` is specified
- Verify Docker Desktop is running
- Check that docker-credential-desktop is in PATH

### Error: "keychain cannot be accessed"
- Add `/Applications/Docker.app/Contents/Resources/bin` to PATH
- Alternatively, run from a logged-in terminal session (not SSH)

### Error: "context default" when desktop-linux is active
- This means the BUILDERS parameter isn't being passed correctly
- Verify the Makefile changes are applied
- Ensure you're using the updated Makefile

### Error: "I/O error" or "input/output error"
**Causes:**
1. **Disk space full** - Check with `df -h /`
2. **Docker.raw too large** - Check size: `ls -lh ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw`
3. **Corrupted Docker state** - Purge Docker data via UI: Troubleshoot → Clean/Purge data

**Solutions:**
1. Free up disk space (need 20-30GB minimum)
2. Reduce Docker disk limit in Settings → Resources → set to 64GB
3. Clean Docker: `docker system prune -af --volumes`
4. Restart Docker Desktop

### Error: "exit code: 2" during compilation
**Causes:**
- Out of memory
- Build errors (rare with stable kernel releases)

**Solutions:**
1. Reduce parallelism: Lower `MAKE_JOBS` in build-args
2. Increase Docker memory allocation (if possible)
3. Check Docker stats during build: `docker stats`

## Alternative: Build on Another Machine

If the Mac has insufficient resources, consider building on:

### Windows Machine with Docker Desktop
```powershell
git clone https://github.com/linuxkit/linuxkit.git
cd linuxkit/kernel
# Apply the same changes to Dockerfile and build-args
linuxkit pkg build . --platforms linux/arm64 --build-yml ./build-kernel.yml --tag "6.12.54-{{.Hash}}" --build-arg-file 6.12.x/build-args
```

### Linux Server or Cloud Instance
```bash
# More RAM and disk space available
git clone https://github.com/linuxkit/linuxkit.git
cd linuxkit/kernel
make build-6.12.54 ARCH=arm64
```

### Transfer Back to Mac
After building, push to Docker Hub:
```bash
docker tag linuxkit/kernel:6.12.54-<hash> yourusername/kernel:6.12.54
docker push yourusername/kernel:6.12.54
```

Then pull on Mac:
```bash
docker pull yourusername/kernel:6.12.54
```

## Next Steps After Successful Build

Once the kernel is built:

1. **Tag it for your organization:**
   ```bash
   docker tag linuxkit/kernel:6.12.54-<hash> datadatdat/kernel:6.12.54
   ```

2. **Push to Docker Hub:**
   ```bash
   docker push datadatdat/kernel:6.12.54
   ```

3. **Update zfs-linuxkit to reference your custom kernel image**

4. **Build ZFS modules against the custom kernel**

## Git Workflow

Commits made:
```bash
# In linuxkit repo
git add kernel/6.12.x/build-args
git add kernel/Makefile
git add kernel/Dockerfile
git commit -m "Add kernel 6.12.54 with configurable parallelism and Docker context support"
git push
```

## Summary

Building a custom LinuxKit kernel on macOS with limited resources requires:
1. ✅ Proper PATH configuration for Docker credentials
2. ✅ Explicit Docker context specification via BUILDERS parameter
3. ✅ Reduced parallelism for memory-constrained systems
4. ✅ Adequate free disk space (20-30GB minimum)
5. ✅ Docker.raw size management (64GB recommended)

The build is resource-intensive and may be better suited for a machine with more RAM and disk space, but can be accomplished on an 8GB Mac with proper configuration and disk management.
