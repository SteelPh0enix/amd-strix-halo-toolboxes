
# Building Containers Locally

If you want to build or customize the toolbox containers yourself (rather than using the pre-built Docker Hub images), this guide explains the process. Local builds are useful if you want to:

* Use a patched or forked version of llama.cpp
* Add additional tools or libraries
* Change the Fedora base image (Rawhide vs. stable)
* Audit every installed dependency

---

## 1. Prerequisites

* **Podman** (recommended on Fedora) or **Docker** (also fine)

---

## 2. Build an Image

Each backend has its own Dockerfile in `toolboxes/`.

**Example: Build the Vulkan RADV toolbox image**

```sh
cd toolboxes
podman build --no-cache -t llama-vulkan-radv -f Dockerfile.vulkan-radv .
```

**Example: Build the ROCm 10.0 toolbox image**

```sh
cd toolboxes
podman build --no-cache -t llama-rocm-10.0 -f Dockerfile.rocm-10.0 .
```

> You can use `docker build` if you prefer Docker.

---

## 3. Customizing the Build

* **llama.cpp version**: Most Dockerfiles accept `--build-arg REPO=...` and `--build-arg BRANCH=...`. The HRX Dockerfile instead accepts `STAGING_BRANCH`; its llama.cpp revision comes from that branch's pinned submodule.
* **Extra dependencies**: Add them to the Dockerfile as needed.
* **Other customizations**: Install tools, patch scripts, or swap to a different base image.

### Experimental HRX toolbox

The `hrx-staging` image follows AMD's [ggml staging build](https://github.com/ROCm/ggml-staging-automation). That repository pins the HRX and llama.cpp submodules and the required TheRock artifact run. It builds both projects and bundles the HRX/Loom/ROCm runtime libraries with llama.cpp. This is a different build path from the ROCm HIP Dockerfiles; `GGML_HRX=ON` is enabled by AMD's build script.

```sh
cd toolboxes
podman build -t llama-hrx-staging -f Dockerfile.hrx-staging .
podman run --rm llama-hrx-staging llama-cli --version
```

To try it on a Strix Halo host, provide both GPU devices:

```sh
toolbox create llama-hrx-staging --image localhost/llama-hrx-staging \
  -- --device /dev/dri --device /dev/kfd --group-add video --group-add render \
  --security-opt seccomp=unconfined
toolbox enter llama-hrx-staging
llama-cli --list-devices
```

Look for an `HRX0` `gfx1151` device before testing a model. Start with the [Qwen3-30B-A3B-Instruct-2507 Q4_K_M model](https://github.com/ggml-org/llama.cpp/discussions/27219) used for the initial HRX work, with `-fa 1 --no-mmap` as in the main README. AMD's staging branch is experimental and may not support every model or operation yet. The build downloads a pinned TheRock SDK and compiles HRX, so allow substantial disk space and build time. The existing workflow can build this image on demand with `backends=hrx-staging`; it is not in the automatic `all` set.

---

## 4. Using the Custom Image with Toolbx

Create a new toolbox using your freshly built image:

```sh
toolbox create llama-vulkan-radv --image localhost/llama-vulkan-radv \
  -- --device /dev/dri --group-add video --security-opt seccomp=unconfined
```

Replace the backend/image name and device/group options as needed (see main README Section 2.1).

---

## 5. Troubleshooting

* **Build fails (ROCm images especially):** Try building with more memory or swap.
* **Toolbox can't access GPU:** Make sure you pass the correct device/group options.

---

## 6. References

* [Fedora Toolbox Documentation](https://docs.fedoraproject.org/en-US/fedora-silverblue/toolbox/)
* [Podman Build Reference](https://docs.podman.io/en/latest/markdown/podman-build.1.html)
* [Docker Build Reference](https://docs.docker.com/engine/reference/commandline/build/)
