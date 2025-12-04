# LinuxKit

[![CircleCI](https://circleci.com/gh/linuxkit/linuxkit.svg?style=svg)](https://circleci.com/gh/linuxkit/linuxkit)

LinuxKit, a toolkit for building custom minimal, immutable Linux distributions.

- Secure defaults without compromising usability
- Everything is replaceable and customisable
- Immutable infrastructure applied to building Linux distributions
- Completely stateless, but persistent storage can be attached
- Easy tooling, with easy iteration
- Built with containers, for running containers
- Designed to create [reproducible builds](./docs/reproducible-builds.md) [WIP]
- Designed for building and running clustered applications, including but not limited to container orchestration such as Docker or Kubernetes
- Designed from the experience of building Docker Editions, but redesigned as a general-purpose toolkit
- Designed to be managed by external tooling, such as [Infrakit](https://github.com/docker/infrakit) (renamed to [deploykit](https://github.com/docker/deploykit) which has been archived in 2019) or similar tools
- Includes a set of longer-term collaborative projects in various stages of development to innovate on kernel and userspace changes, particularly around security

LinuxKit currently supports the `x86_64`, `arm64`, and `s390x` architectures on a variety of platforms, both as virtual machines and baremetal (see [below](#booting-and-testing) for details).

## Subprojects

- [LinuxKit kubernetes](https://github.com/linuxkit/kubernetes) aims to build minimal and immutable Kubernetes images. (previously `projects/kubernetes` in this repository).
- [LinuxKit LCOW](https://github.com/linuxkit/lcow) LinuxKit images and utilities for Microsoft's Linux Containers on Windows.
- [linux](https://github.com/linuxkit/linux) A copy of the Linux stable tree with branches LinuxKit kernels.
- [virtsock](https://github.com/linuxkit/virtsock) A `go` library and test utilities for `virtio` and Hyper-V sockets.
- [rtf](https://github.com/linuxkit/rtf) A regression test framework used for the LinuxKit CI tests (and other projects).
- [homebrew](https://github.com/linuxkit/homebrew-linuxkit) Homebrew packages for the `linuxkit` tool.

## Getting Started

### Build the `linuxkit` tool

LinuxKit uses the `linuxkit` tool for building, pushing and running VM images.

Simple build instructions: use `make` to build. This will build the tool in `bin/`. Add this
to your `PATH` or copy it to somewhere in your `PATH` eg `sudo cp bin/* /usr/local/bin/`. Or you can use `sudo make install`.

If you already have `go` installed you can use `go install github.com/linuxkit/linuxkit/src/cmd/linuxkit@latest` to install the `linuxkit` tool.

On MacOS there is a `brew tap` available. Detailed instructions are at [linuxkit/homebrew-linuxkit](https://github.com/linuxkit/homebrew-linuxkit),
the short summary is
```
brew tap linuxkit/linuxkit
brew install --HEAD linuxkit
```

Build requirements from source using a container
- GNU `make`
- Docker
- optionally `qemu`

For a local build using `make local`
- `go`
- `make`
- `go get -u golang.org/x/lint/golint`
- `go get -u github.com/gordonklaus/ineffassign`

### Building images

Once you have built the tool, use

```
linuxkit build linuxkit.yml
```
to build the example configuration. You can also specify different output formats, eg `linuxkit build --format raw-bios linuxkit.yml` to
output a raw BIOS bootable disk image, or `linuxkit build --format iso-efi linuxkit.yml` to output an EFI bootable ISO image. See `linuxkit build -help` for more information.

### Booting and Testing

You can use `linuxkit run <name>` or `linuxkit run <name>.<format>` to
execute the image you created with `linuxkit build <name>.yml`.  This
will use a suitable backend for your platform or you can choose one,
for example VMWare.  See `linuxkit run --help`.

Currently supported platforms are:
- Local hypervisors
  - [Virtualization.Framework (macOS)](docs/platform-virtualization-framework.md) `[x86_64, arm64]`
  - [HyperKit (macOS)](docs/platform-hyperkit.md) `[x86_64]`
  - [Hyper-V (Windows)](docs/platform-hyperv.md) `[x86_64]`
  - [qemu (macOS, Linux, Windows)](docs/platform-qemu.md) `[x86_64, arm64, s390x]`
  - [VMware (macOS, Windows)](docs/platform-vmware.md) `[x86_64]`
- Cloud based platforms:
  - [Amazon Web Services](docs/platform-aws.md) `[x86_64]`
  - [Google Cloud](docs/platform-gcp.md) `[x86_64]`
  - [Microsoft Azure](docs/platform-azure.md) `[x86_64]`
  - [OpenStack](docs/platform-openstack.md) `[x86_64]`
  - [Scaleway](docs/platform-scaleway.md) `[x86_64]`
- Baremetal:
  - [deploy.equinix.com](docs/platform-equinixmetal.md) `[x86_64, arm64]`
  - [Raspberry Pi Model 3b](docs/platform-rpi3.md)  `[arm64]`


#### Running the Tests

The test suite uses [`rtf`](https://github.com/linuxkit/rtf) To
install this you should use `make bin/rtf && make install`. You will
also need to install `expect` on your system as some tests use it.

To run the test suite:

```
cd test
rtf -v run -x
```

This will run the tests and put the results in a the `_results` directory!

Run control is handled using labels and with pattern matching.
To run add a label you may use:

```
rtf -v -l slow run -x
```

To run tests that match the pattern `linuxkit.examples` you would use the following command:

```
rtf -v run -x linuxkit.examples
```

## Building your own customised image

To customise, copy or modify the [`linuxkit.yml`](linuxkit.yml) to your own `file.yml` or use one of the [examples](examples/) and then run `linuxkit build file.yml` to
generate its specified output. You can run the output with `linuxkit run file`.

The yaml file specifies a kernel and base init system, a set of containers that are built into the generated image and started at boot time. You can specify the type
of artifact to build eg `linuxkit build -format vhd linuxkit.yml`.

If you want to build your own packages, see this [document](docs/packages.md).

### Yaml Specification

The yaml format specifies the image to be built:

- `kernel` specifies a kernel Docker image, containing a kernel and a filesystem tarball, eg containing modules. The example kernels are built from `kernel/`
- `init` is the base `init` process Docker image, which is unpacked as the base system, containing `init`, `containerd`, `runc` and a few tools. Built from `pkg/init/`
- `onboot` are the system containers, executed sequentially in order. They should terminate quickly when done.
- `services` is the system services, which normally run for the whole time the system is up
- `files` are additional files to add to the image

For a more detailed overview of the options see [yaml documentation](docs/yaml.md)

## Architecture and security

There is an [overview of the architecture](docs/architecture.md) covering how the system works.

There is an [overview of the security considerations and direction](docs/security.md) covering the security design of the system.

## Roadmap

This project was extensively reworked from the code we are shipping in Docker Editions, and the result is not yet production quality. The plan is to return to production
quality during Q3 2017, and rebase the Docker Editions on this open source project during this quarter. We plan to start making stable releases on this timescale.

This is an open project without fixed judgements, open to the community to set the direction. The guiding principles are:
- Security informs design
- Infrastructure as code: immutable, manageable with code
- Sensible, secure, and well-tested defaults
- An open, pluggable platform for diverse use cases
- Easy to use and participate in the project
- Built with containers, for portability and reproducibility
- Run with system containers, for isolation and extensibility
- A base for robust products

## Development reports

There are monthly [development reports](reports/) summarising the work carried out each month.

## Building Custom Kernel for macOS Docker Desktop (ARM64)

This section documents the process for building a custom LinuxKit kernel for macOS Docker Desktop on Apple Silicon (ARM64). This is necessary when Docker Desktop uses a kernel version that doesn't have pre-built ARM64 images available upstream.

### Problem Context

Docker Desktop 4.53.0 on macOS uses kernel 6.12.54-linuxkit, but the upstream linuxkit/kernel repository only has ARM64 images up to 6.12.52. To use ZFS modules or other kernel-dependent tools, we need to build the 6.12.54 kernel for ARM64.

### Prerequisites

1. Docker Desktop for Mac installed and running
2. Homebrew installed
3. LinuxKit CLI tool: `brew tap linuxkit/linuxkit && brew install --HEAD linuxkit`
4. Git repository forked from linuxkit/linuxkit

### Build Environment Issues on macOS

When building via SSH on macOS, you'll encounter Docker credential helper issues because the keychain is locked in SSH sessions. The linuxkit build process needs to pull the buildkit image, which requires Docker Hub authentication.

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

### File Changes Made

#### 1. Update Kernel Version

**File:** `kernel/6.12.x/build-args`

Update the kernel version:
```
KERNEL_VERSION=6.12.54
```

#### 2. Add BUILDERS Parameter Support to Makefile

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

### Build Commands

#### Check Docker Contexts and Buildx Builders

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

#### Building the Kernel

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

### Build Process Details

The build command will:
1. Check local cache for existing image
2. Check Docker Hub registry
3. Pull base images (Alpine, buildkit)
4. Create a `linuxkit-builder` container in the specified context
5. Run buildkit to compile the kernel
6. Package kernel, modules, and firmware into a Docker image
7. Tag as `linuxkit/kernel:6.12.54-<hash>`

### Troubleshooting

**Error: "unable to create or find builder container"**
- Ensure `BUILDERS="linux/arm64=desktop-linux"` is specified
- Verify Docker Desktop is running
- Check that docker-credential-desktop is in PATH

**Error: "keychain cannot be accessed"**
- Add `/Applications/Docker.app/Contents/Resources/bin` to PATH
- Alternatively, run from a logged-in terminal session (not SSH)

**Error: "context default" when desktop-linux is active**
- This means the BUILDERS parameter isn't being passed correctly
- Verify the Makefile changes are applied
- Ensure you're using the updated Makefile

### Next Steps After Kernel Build

Once the kernel is built:
1. Tag it for your organization: `docker tag linuxkit/kernel:6.12.54-<hash> datadatdat/kernel:6.12.54`
2. Push to Docker Hub: `docker push datadatdat/kernel:6.12.54`
3. Update zfs-linuxkit to reference your custom kernel image
4. Build ZFS modules against the custom kernel

### Git Workflow

Commits made:
```bash
# In linuxkit repo
git add kernel/6.12.x/build-args
git add kernel/Makefile
git commit -m "Update kernel to 6.12.54 and add BUILDERS parameter support"
git push
```

## Adopters

We maintain an incomplete list of [adopters](ADOPTERS.md). Please open a PR if you are using LinuxKit in production or in your project, or both.

## FAQ

See [FAQ](docs/faq.md).

Released under the [Apache 2.0 license](LICENSE).
