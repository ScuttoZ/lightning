---
title: Reproducible builds
slug: repro
privacy:
  view: public
---
[Reproducible builds](https://reproducible-builds.org/) close the final gap in the lifecycle of open-source projects by allowing anyone to verify that a given binary was produced by compiling publicly available source code.

Core Lightning has provided a manifest of the binaries included in a release, along with signatures from the maintainers since version 0.6.2.

The steps involved in creating reproducible builds are:

- Creation of a known environment in which to build the source code
- Removal of variance during the compilation (randomness, timestamps, etc)
- Packaging of binaries
- Creation of a manifest (`SHA256SUMS` file containing the cryptographic hashes of the binaries and packages)
- Signing of the manifest by maintainers and volunteers that have reproduced the files in the manifest starting from the source.

The bulk of these operations are handled by the [`repro-build.sh`](https://github.com/ElementsProject/lightning/blob/master/tools/repro-build.sh) script, but some manual operations are required to setup the build environment. Since a binary is built against platform specific libraries we also need to replicate the steps once for each OS distribution and architecture, so the majority of this guide will describe how to set up those starting from a minimal trusted base. This minimal trusted base in most cases is the official installation medium from the OS provider.

Note: Since your signature certifies the integrity of the resulting binaries, please familiarize yourself with both the [`repro-build.sh`](https://github.com/ElementsProject/lightning/blob/master/tools/repro-build.sh) script, as well as with the setup instructions for the build environments before signing anything.

# Build Environment Setup

The build environments are a set of docker images that are created directly from the installation mediums and repositories from the OS provider. The following sections describe how to create those images. Don't worry, you only have to create each image once and can then reuse the images for future builds.

## Script cl-repro

The script `contrib/cl-repro.sh` covers below `Base image creation` and `Builder image setup` steps. You can skip these steps by simply running the `contrib/cl-repro.sh` script.

## Base image creation

Depending on the distribution that we want to build for the instructions to create a base image can vary. In the following sections we discuss the specific instructions for each distribution, whereas the instructions are identical again once we have the base image.

### Debian / Ubuntu and derivative OSs

For operating systems derived from Debian we can use the `debootstrap` tool to build a minimal OS image, that can then be transformed into a docker image. The packages for the minimal OS image are directly downloaded from the installation repositories operated by the OS provider.

We cannot really use the `debian` and `ubuntu` images from the docker hub, mainly because it'd be yet another trusted third party, but it is also complicated by the fact that the images have some of the packages updated. The latter means that if we disable the `updates` and `security` repositories for `apt` we find ourselves in a situation where we can't install any additional packages (wrongly updated packages depending on the versions not available in  
the non-updated repos).

The following table lists the codenames of distributions that we currently support:

- Ubuntu 22.04:
  - Distribution Version: 22.04
  - Codename: jammy
- Ubuntu 24.04:
  - Distribution Version: 24.04
  - Codename: noble
- Ubuntu 26.04:
  - Distribution Version: 26.04
  - Codename: resolute

Depending on your host OS release you might not have `debootstrap` manifests for versions newer than your host OS. Due to this we run the `debootstrap` commands in a container of the latest version itself:

```shell
for v in jammy noble resolute; do
  echo "Building base image for $v"
  docker run --rm -v $(pwd):/build ubuntu:$v \
	bash -c "apt-get update && apt-get install -y debootstrap && debootstrap $v /build/$v"
  tar -C $v -c . | docker import - $v
done
```

Verify that the image corresponds to our expectation and is runnable:

```shell
docker run noble cat /etc/lsb-release
```

Which should result in the following output for `noble`:

```shell
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04 LTS"
```

## Builder image setup

Once we have the clean base image we need to customize it to be able to build Core Lightning. This includes disabling the update repositories, downloading the build dependencies and specifying the steps required to perform the build.

For this purpose we have a number of Dockerfiles in the [`contrib/reprobuild`](https://github.com/ElementsProject/lightning/tree/master/contrib/reprobuild) directory that have the specific instructions for each base image.

We can then build the builder image by calling `docker build` and passing it the `Dockerfile`:

```shell
docker build -t cl-repro-jammy - < contrib/reprobuild/Dockerfile.jammy
docker build -t cl-repro-noble - < contrib/reprobuild/Dockerfile.noble
docker build -t cl-repro-resolute - < contrib/reprobuild/Dockerfile.resolute
```

Since we pass the `Dockerfile` through `stdin` the build command will not create a context, i.e., the current directory is not passed to `docker` and it'll be independent of the currently checked out version. This also means that you will be able to reuse the docker image for future builds, and don't have to repeat this dance every time. Verifying the `Dockerfile` therefore is  
sufficient to ensure that the resulting `cl-repro-<codename>` image is reproducible.

The dockerfiles assume that the base image has the codename as its image name.

# Building using the builder image

Finally, after finishing the environment setup we can perform the actual build. At this point we have a container image that has been prepared to build reproducibly. As you can see from the `Dockerfile` above we assume the source git repository gets mounted as `/repo` in the docker container. The container will clone the repository to an internal path, in order to keep the repository clean, build the artifacts there, and then copy them back to `/repo/release`.  
We'll need the release directory available for this, so create it now if it doesn't exist:`mkdir release`, then we can simply execute the following command inside the git repository (remember to checkout the tag you are trying to build):

```bash
docker run --rm -v $(pwd):/repo -ti cl-repro-jammy
docker run --rm -v $(pwd):/repo -ti cl-repro-noble
docker run --rm -v $(pwd):/repo -ti cl-repro-resolute
```

The last few lines of output also contain the `sha256sum` hashes of all artifacts, so if you're just verifying the build those are the lines that are of interest to you:

```shell
ee83cf4948228ab1f644dbd9d28541fd8ef7c453a3fec90462b08371a8686df8  /repo/release/clightning-v0.9.0rc1-Ubuntu-18.04.tar.xz
94bd77f400c332ac7571532c9f85b141a266941057e8fe1bfa04f054918d8c33  /repo/release/clightning-v0.9.0rc1.zip
```

Repeat this step for each distribution and each architecture you wish to sign. Once all the binaries are in the `release/` subdirectory we can sign the hashes.

# Signing the release manifest

The release captain is in charge of creating the manifest, whereas contributors and interested bystanders may contribute their signatures to further increase trust in the binaries.

## Script build-release
1: Pull latest code from master

2: Run the `tools/build-release.sh bin-Fedora bin-Ubuntu sign` script. This will create a release directory, build binaries for Fedora, and build binaries for Ubuntu (Jammy, Noble, and Resolute). Finally, it will sign the ZIP, Fedora, and Ubuntu builds.

## Manual
The release captain creates the manifest as follows:

```shell
cd release/
sha256sum *v0.9.0* > SHA256SUMS
gpg -sb --armor SHA256SUMS
```

# Co-signing the release manifest

## Script build-release
1: Pull latest code from master.

2: Rename checksum files, shared by the release captain, to `SHA256SUMS-v($VERSION)` and `SHA256SUMS-v($VERSION).asc`.

3: Copy above files in the lightning directory.

4: Run `tools/build-release.sh --verify` script. It will build binaries for Ubuntu (Jammy, Noble & Resolute), verify zip & Ubuntu builds while copying Fedora checksums from the release captain's file.

5. Then send the resulting `release/SHA256SUMS.asc` file to the release captain so it can be merged with the other signatures into `SHASUMS.asc`.

## Manual
Co-maintainers and contributors wishing to add their own signature verify that the `SHA256SUMS` and `SHA256SUMS.asc` files created by the release captain matches their binaries before also signing the manifest:

```shell
cd release/
gpg --verify SHA256SUMS.asc
sha256sum -c SHA256SUMS
cat SHA256SUMS | gpg -sb --armor > SHA256SUMS.new
```

Then send the resulting `SHA256SUMS.new` file to the release captain so it can be merged with the other signatures into `SHASUMS.asc`.

# Verifying a reproducible build

You can verify the reproducible build in two ways:

- Repeating the entire reproducible build, making sure from scratch that the binaries match. Just follow the instructions above for this.
- Verifying that the downloaded binaries match the hashes in `SHA256SUMS` and that the signatures in `SHA256SUMS.asc` are valid.

Assuming you have downloaded the binaries, the manifest and the signatures into the same directory, you can verify the signatures with the following:

```shell
gpg --verify SHA256SUMS.asc
```

And you should see a list of messages like the following:

```shell
gpg: assuming signed data in 'SHA256SUMS'
gpg: Signature made Fr 08 Mai 2020 07:46:38 CEST
gpg:                using RSA key 15EE8D6CAB0E7F0CF999BFCBD9200E6CD1ADB8F1
gpg: Good signature from "Rusty Russell <rusty@rustcorp.com.au>" [full]
gpg: Signature made Fr 08 Mai 2020 12:30:10 CEST
gpg:                using RSA key B7C4BE81184FC203D52C35C51416D83DC4F0E86D
gpg: Good signature from "Christian Decker <decker.christian@gmail.com>" [ultimate]
gpg: Signature made Fr 08 Mai 2020 21:35:28 CEST
gpg:                using RSA key 30DE693AE0DE9E37B3E7EB6BBFF0F67810C1EED1
gpg: Good signature from "Lisa Neigut <niftynei@gmail.com>" [full]
```

If there are any issues `gpg` will print `Bad signature`, it might be because the signatures in `SHA256SUMS.asc` do not match the `SHA256SUMS` file, and could be the result of a filename change. Do not continue using the binaries, and contact the maintainers, if this is not the case, a failure here means that the verification failed.

Next we verify that the binaries match the ones in the manifest:

```shell
sha256sum -c SHA256SUMS
```

Producing output similar to the following:

```shell
sha256sum: clightning-v24.11-Fedora-35-amd64.tar.gz: No such file or directory
clightning-v24.11-Fedora-35-amd64.tar.gz: FAILED open or read
clightning-v24.11-Ubuntu-22.04.tar.xz: OK
clightning-v24.11-Ubuntu-24.04.tar.xz: OK
clightning-v24.11-Ubuntu-26.04.tar.xz: OK
clightning-v24.11.zip: OK
sha256sum: WARNING: 1 listed file could not be read
```

Notice that the two files we downloaded are marked as `OK`, but we're missing one file. If you didn't download that file this is to be expected, and is nothing to worry about. A failure to verify the hash would give a warning like the following:

```shell
sha256sum: WARNING: 1 computed checksum did NOT match
```

If both the signature verification and the manifest checksum verification succeeded, then you have just successfully verified a reproducible build and, assuming you trust the maintainers, are good to install and use the binaries. Congratulations! 🎉🥳

# Beginner walkthrough: verifying a release from scratch

The sections above assume you are already comfortable with Docker, GPG and shell scripting. This section is a slower, copy-paste-friendly walkthrough of the most common task — **verifying that an official release binary really was built from the published source** — aimed at users who have not done this before. It does not replace the reference above; it just spells out every command, where to run it, and what you should expect to see.

Throughout this section:

- **"on the host"** means a normal terminal on your own computer.
- **"inside the container"** means commands that run automatically inside Docker. You will *not* type these yourself — they are listed only so you know what is happening.

In this example we verify the `v26.06` release built for **Ubuntu 24.04** (codename `noble`). Substitute your own version and codename as needed.

## Step 0 — Check your machine meets the requirements

Run these **on the host**:

```shell
uname -m
docker --version
docker info
```

What to look for:

- `uname -m` should print `x86_64`. These builder images are **amd64-only**; they will not work on an ARM machine (e.g. a Raspberry Pi or Apple-silicon Mac).
- `docker --version` should print a version. If the command is not found, install Docker first.
- `docker info` should print a block of information without errors. If instead you see `permission denied while trying to connect to the Docker daemon`, you have two options:
  - prefix **every** `docker` command in this guide with `sudo`, or
  - add your user to the `docker` group once (`sudo usermod -aG docker $USER`), then log out and back in.

You will also need a few gigabytes of free disk per distribution and some patience: a full build takes from several minutes to over half an hour depending on your CPU.

## Step 1 — Get the source code at the exact release

Run these **on the host**. If you do not already have the repository:

```shell
git clone https://github.com/ElementsProject/lightning.git
cd lightning
```

If you already have it, just `cd` into it. Then switch to the exact release tag you want to verify:

```shell
git checkout v26.06
```

Git will warn you that you are in *"detached HEAD"* state. **This is expected and fine** — it simply means you are looking at a fixed release rather than a moving branch.

Tip: the build copies committed code only, so make sure you have no local edits with `git status` (it should say *nothing to commit, working tree clean*). Any uncommitted changes you have made will be **ignored** by the build.

## Step 2 — Create the build environments (one time only)

This is the longest one-off step. The helper script builds the trusted base images and the `cl-repro-*` builder images for you, so you do not have to run the `debootstrap` and `docker build` commands from the reference sections by hand.

Run this **on the host**, from the root of the `lightning` directory:

```shell
sudo contrib/cl-repro.sh
```

Notes:

- The script uses `sudo` internally, which is why you run it with `sudo`.
- It creates one base image and one builder image per distribution (`jammy`, `noble`, `resolute`). Expect this to take a while and download a few hundred megabytes.
- You only have to do this **once per machine**. You can reuse the images for every future verification.

When it finishes, confirm the builder images exist **on the host**:

```shell
docker images | grep cl-repro
```

You should see `cl-repro-jammy`, `cl-repro-noble` and `cl-repro-resolute`.

## Step 3 — Create the output folder

The build writes its result into a `release/` folder. Create it **on the host**, from the root of the `lightning` directory:

```shell
mkdir -p release
```

The `-p` flag means the command succeeds even if the folder already exists, so it is safe to run again.

## Step 4 — Build the binary

Now build the binary for one distribution. Run this **on the host** (we use `noble` = Ubuntu 24.04 here):

```shell
docker run --rm -v "$(pwd)":/repo cl-repro-noble
```

What this does, all **inside the container** and automatically — you do not type anything else:

1. clones your checked-out source into a clean directory,
2. installs the exact, hash-pinned build dependencies,
3. compiles Core Lightning,
4. packages the result and copies it to your `release/` folder on the host.

> This command deliberately uses **no** `-i`/`-ti` flags. The build is fully automatic and needs no keyboard input. The problem flag is `-i` (interactive): running `-ti` from a script or any non-interactive shell fails with *"cannot attach stdin to a TTY-enabled container because stdin is not a terminal"*. A plain `docker run` (as shown above) always works. Adding `-t` on its own is harmless if you want prettier progress output — the project's nightly CI ([`.github/workflows/repro.yml`](https://github.com/ElementsProject/lightning/blob/master/.github/workflows/repro.yml)) uses `-t` — but it is not needed for verification.

When it finishes you are returned to your normal host prompt. To build for the other distributions, repeat with the matching image:

```shell
docker run --rm -v "$(pwd)":/repo cl-repro-jammy
docker run --rm -v "$(pwd)":/repo cl-repro-resolute
```

## Step 5 — Look at the hash you produced

Run this **on the host**:

```shell
sha256sum release/*.tar.xz
```

This prints a 64-character hash next to each file you built, for example:

```shell
61d2dadb125da4be5f9f887d0cc8f00aaafaad6bb335218b96426b1985ccbc67  release/clightning-v26.06-Ubuntu-24.04-amd64.tar.xz
```

That hash is the fingerprint of *your* build.

## Step 6 — Compare against the official release

Finally, check that your fingerprint matches the one the maintainers published.

1. Open the release page on GitHub for your version, e.g. `https://github.com/ElementsProject/lightning/releases/tag/v26.06`, and download the `SHA256SUMS` and `SHA256SUMS.asc` files **into your `release/` folder**.
2. Import the maintainers' public keys so GPG can check the signatures (you only need to do this once). Their key fingerprints are listed in the project's release documentation; import them with `gpg --recv-keys <fingerprint>` or from a downloaded key file with `gpg --import <file>`.
3. Then run, **on the host**:

```shell
cd release/
gpg --verify SHA256SUMS.asc
sha256sum -c SHA256SUMS --ignore-missing
```

How to read the result:

- `gpg --verify` should report `Good signature` for one or more maintainers. A `Good signature` with a warning that the key is *not certified* simply means you have not personally marked the key as trusted — that is normal.
- `sha256sum -c` should print `OK` next to each file you actually built. The `--ignore-missing` flag tells it to skip the files listed in `SHA256SUMS` that you did not build (for example the Fedora package), so those will not be reported as errors.

If you see `OK` for your file and a `Good signature`, your build matches the official release — you have verified it yourself. 🎉

If instead `sha256sum -c` prints `FAILED` for a file you built, **do not use that binary**: your build did not match the published one. Re-check that you built the same version (Step 1) on an `x86_64` machine, and if it still fails, contact the maintainers.

## Quick troubleshooting

| What you see | What it means / what to do |
| --- | --- |
| `permission denied while trying to connect to the Docker daemon` | Prefix `docker` commands with `sudo`, or add yourself to the `docker` group (Step 0). |
| `cannot attach stdin to a TTY-enabled container because stdin is not a terminal` | You added `-i`/`-ti` to the `docker run` build command. Drop the `-i` — a plain `docker run` is fine (`-t` alone is also OK). See Step 4. |
| `mkdir: cannot create directory 'release': File exists` | Harmless — the folder is already there. Use `mkdir -p release` to silence it. |
| `Unable to find image 'cl-repro-noble'` | The builder image was not created. Re-run Step 2. |
| `cargo`/`make` errors deep into the build | Usually a half-finished image or low disk space. Free up disk, then rebuild the image (Step 2) and try again. |
