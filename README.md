# srt

Container images with the SRT (Secure Reliable Transport) tools from [Haivision/srt](https://github.com/Haivision/srt). SRT carries live video over UDP with retransmission of lost packets, a fixed latency and optional AES encryption. `srt-live-transmit` moves a stream between SRT, UDP and standard input or output, `srt-file-transmit` sends a file, and `srt-tunnel` carries TCP connections over SRT. libsrt is compiled from the release tag and linked into the tools. The default image also contains FFmpeg. The images are rebuilt when SRT publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by Haivision or the SRT Alliance. Report problems with the image in this repository and SRT bugs at [Haivision/srt](https://github.com/Haivision/srt/issues).

## Quick start

Listen on UDP port 9000 and write the stream that arrives to `stream.ts`:

```sh
docker run --rm -p 9000:9000/udp ghcr.io/randomcontainers/srt 'srt://:9000' file://con > stream.ts
```

On another host, stream a video file to it with FFmpeg from the default image, with the listener's name or IP address in place of `<host>`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint ffmpeg \
  ghcr.io/randomcontainers/srt -loglevel error -re -i input.mp4 -c copy \
  -f mpegts 'srt://<host>:9000?pkt_size=1316'
```

`srt-live-transmit` takes an input URI and an output URI. `srt://:9000` listens on port 9000, `srt://<host>:9000` connects to a listener, and `file://con` is standard input or output. Input that arrives while there is no connection is discarded, so the stream starts once the two sides are connected. The tool keeps running at the end of its input. After a disconnect, a listener accepts the next connection and a caller connects again. Stop it with Ctrl+C, or pass `-a:no` to exit when the connection closes or `-t:<seconds>` to exit after a fixed time.

MPEG-TS goes over SRT in messages of 1316 bytes, seven TS packets of 188 bytes, which is what receivers such as FFmpeg expect. Set `pkt_size=1316` on FFmpeg's `srt://` and `udp://` outputs. With SRT 1.5.7, do not pipe a stream into `srt-live-transmit file://con` to send it: when a read from standard input returns less than a full chunk, it still sends the whole chunk, padded with zeros ([Haivision/srt#3388](https://github.com/Haivision/srt/issues/3388), fixed after 1.5.7). Send from FFmpeg directly, as above, or over UDP as in the first command below.

A few more commands:

```sh
# Send an MPEG-TS stream that arrives on local UDP port 5000, for example from an encoder, to a listener
docker run --rm --network host ghcr.io/randomcontainers/srt udp://127.0.0.1:5000 'srt://<host>:9000'

# Receive on port 9000 and pass the stream on to UDP port 5001, for a player or another tool
docker run --rm --network host ghcr.io/randomcontainers/srt 'srt://:9000' udp://127.0.0.1:5001

# Record what arrives on port 9000 to MKV with FFmpeg (default image), until the sender disconnects
docker run --rm --init -p 9000:9000/udp --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/srt -c \
  'srt-live-transmit -a:no srt://:9000 file://con | ffmpeg -loglevel error -i - -c copy recording.mkv'
```

`--init` makes Ctrl+C stop a container started with `--entrypoint sh`.

Running the image without arguments prints the options of `srt-live-transmit`. Its status messages go to standard error. With `file://con` as the output it refuses `-v`, and statistics (`-s:<packets>`) need `-statsout:<file>`, so that nothing but the stream goes to standard output. The [srt-live-transmit documentation](https://github.com/Haivision/srt/blob/master/docs/apps/srt-live-transmit.md) lists the URI parameters.

## Encryption

Set the same `passphrase`, 10 to 80 characters, on both sides. The key length is 128 bits unless the sender asks for more with `pbkeylen=24` (AES-192) or `pbkeylen=32` (AES-256); leave it unset on the receiver. A peer with a different passphrase or none is rejected.

```sh
docker run --rm -p 9000:9000/udp ghcr.io/randomcontainers/srt \
  "srt://:9000?passphrase=$(cat srt-passphrase)" file://con > stream.ts
```

Reading the passphrase from a file keeps it out of your shell history, but it is still part of the container's command line, which `ps` and `docker inspect` show to other users of the host. Use letters and digits, or percent-encode other characters, since the passphrase is part of a URI. SRT encrypts the payload, not the packet headers.

## Networking

SRT runs over UDP. A listener needs its port published as UDP, `-p 9000:9000/udp`, or the host's network with `--network host`. A caller needs no published port. `srt://:9000` listens on every interface of the container; `srt://127.0.0.1:9000?mode=listener` listens on one address. `srt-live-transmit` and `srt-file-transmit` refuse ports below 1024.

Packets that are lost are sent again until the latency runs out, 120 ms by default. Set `latency` in milliseconds, for example `srt://:9000?latency=500`; the larger value of the two sides is used. Upstream suggests three to four times the round-trip time.

SRT 1.5.6 fixed CVE-2026-55869, a buffer overflow in the handling of KMREQ/KMRSP (key material) messages, and CVE-2026-55868, an encryption downgrade. SRT 1.5.7 completed both fixes. Use 1.5.7 or later on every SRT endpoint that hosts you do not trust can reach, and check the libsrt version of other SRT software before you expose it.

## Sending files and tunnelling TCP

`srt-file-transmit` sends one file in SRT's file mode, which delivers every byte instead of keeping a fixed latency. The receiver writes into an existing directory, under the name the sender passes as `streamid`, and exits after one file. It refuses names that contain a path and names of files that already exist.

```sh
# Receiver: writes incoming/video.mp4
mkdir -p incoming
docker run --rm -p 9000:9000/udp --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --entrypoint srt-file-transmit ghcr.io/randomcontainers/srt 'srt://:9000' file:///work/incoming

# Sender
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --entrypoint srt-file-transmit ghcr.io/randomcontainers/srt file:///work/video.mp4 'srt://<host>:9000?streamid=video.mp4'
```

`srt-tunnel <listen-uri> <call-uri>` accepts connections on the first URI and connects each one to the second, so two tunnels can carry a TCP connection over SRT between two sites:

```sh
# Site A: accept TCP on local port 2222 and call site B over SRT
docker run --rm --init --network host --entrypoint srt-tunnel ghcr.io/randomcontainers/srt \
  tcp://127.0.0.1:2222 'srt://<site-b>:9000'

# Site B: accept SRT on port 9000 and connect to the SSH server
docker run --rm --init --network host --entrypoint srt-tunnel ghcr.io/randomcontainers/srt \
  'srt://:9000' tcp://127.0.0.1:22
```

`srt-tunnel` has no signal handling of its own, which is why these commands use `--init`.

## What is in the image

| Area | Details |
|---|---|
| Tools | `srt-live-transmit`, `srt-file-transmit` and `srt-tunnel` in `/usr/local/bin` |
| libsrt | Linked into each tool, so the image has no `libsrt.so` |
| Encryption | AES-CTR with 128, 192 or 256-bit keys, through the distro's OpenSSL |
| Media | `srt-live-transmit`: SRT as listener, caller or rendezvous, UDP including multicast, RTP as input, and `file://con` for standard input and output |
| Default image | FFmpeg (`ffmpeg` and `ffprobe`), for encoding, recording and converting streams |

Not included: connection bonding, the AES-GCM mode (an API preview in SRT 1.5 that needs a build option), `srt-ffplay` (a script that pipes a stream into ffplay), the test programs, and the libsrt library, headers and pkg-config files for building your own programs. `srt-live-transmit` 1.5 reads and writes files only through standard input and output: upstream has disabled `file://` paths in it. The commit and the CMake options are in `/usr/local/share/randomcontainers/srt/buildinfo`.

## Default or slim

Use the default image (`latest`) for general work. It adds FFmpeg, so one container can encode or record a stream and send or receive it over SRT (see [Quick start](#quick-start)). `slim` has the SRT tools and the libraries they need, without FFmpeg. Use it to build your own image, or for relays that only move a stream between SRT and UDP.

FFmpeg's own `srt://` input and output use the libsrt of the FFmpeg image, not the one in the SRT tools; [randomcontainers/ffmpeg](https://github.com/randomcontainers/ffmpeg) describes it. The default image includes FFmpeg, which is licensed under the GNU General Public License (GPL-3.0-or-later). Use `slim` if your policy excludes GPL. The default image is also published as `ghcr.io/randomcontainers/srt-ffmpeg`, built in the [srt-ffmpeg](https://github.com/randomcontainers/srt-ffmpeg) repository with the same contents and a different digest.

## Tags

`<version>` is an SRT release such as `1.5.7`. `<minor>` and `<major>` are its shorter forms, `1.5` and `1`, and follow the newest release in that series.

| Default (with FFmpeg) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current SRT version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

`srt-live-transmit` handles files only through standard input and output, so a redirection on the host, as in the [Quick start](#quick-start), needs no mount. `srt-file-transmit`, `-statsout` and `-logfile` work with files, so mount the directory that holds them.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no FFmpeg, so FFmpeg updates do not rebuild it. The entrypoint is `["tini", "--", "srt-live-transmit"]`, so `CMD` holds the input and output URIs. This image relays a stream from its listener to another host:

```dockerfile
FROM ghcr.io/randomcontainers/srt:slim-alpine@sha256:...
CMD ["srt://:9000?latency=500", "srt://<host>:9000?latency=500"]
```

The image runs as UID 1000, so switch to `USER root` to install distro packages, then back to `USER 1000:1000`. The packages the tools need are listed in `/usr/local/share/randomcontainers/srt/runtime-deps`. To pick up new SRT releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/srt:latest \
  --repo randomcontainers/srt --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/srt-ffmpeg` are built in that repository, so verify them with `--repo randomcontainers/srt-ffmpeg` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/srt:latest --format '{{ json .SBOM }}'
```

SRT is compiled here, so it is not in the SBOM and image scanners do not check it. Its version and commit are in `/usr/local/share/randomcontainers/srt/buildinfo`.

SRT attaches only Windows installers to its releases and does not sign its tags, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [Haivision/srt](https://github.com/Haivision/srt) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. It also stops unless every tool loads OpenSSL's libcrypto and no shared libsrt.

## Updates

The project checks the releases of [Haivision/srt](https://github.com/Haivision/srt/releases) every 15 minutes. A release is picked up once it is 24 hours old; release candidates are skipped. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new FFmpeg image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t srt:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

libsrt and the SRT tools are licensed under the [Mozilla Public License 2.0](https://github.com/Haivision/srt/blob/master/LICENSE). Parts of libsrt keep the terms of the code they come from:

- BSD-3-Clause: the UDT core, from the University of Illinois.
- Zlib: the MD5 implementation by L. Peter Deutsch (Aladdin Enterprises).
- Unlicense: the atomics wrapper in `srtcore/atomic.h`.
- Public domain: the byte-order macros in `srtcore/utilities.h`.

The license texts are in `/usr/local/share/randomcontainers/srt/licenses/`: `LICENSE` (MPL-2.0), `LICENSE.udt`, `LICENSE.md5` and `LICENSE.atomic`. Links to the source are in `/usr/local/share/randomcontainers/srt/source`, and the `v<version>` release of this repository has the source of each version it builds, `srt-<version>.tar.gz`, an archive of the pinned commit. The image's license label is `MPL-2.0 AND BSD-3-Clause AND Zlib AND Unlicense`.

The default image adds FFmpeg, licensed under GPL-3.0-or-later; see [randomcontainers/ffmpeg](https://github.com/randomcontainers/ffmpeg#licenses) for its sources and license files. OpenSSL, libstdc++ and the other libraries from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
