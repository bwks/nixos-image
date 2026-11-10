# NixOS image for Sherpa

Builds the qcow2 disk image for [Sherpa](https://github.com/bwks/sherpa)'s `nixos_linux` lab
nodes. NixOS does not publish a cloud-init enabled disk image, so this flake builds one:
`flake.nix` picks the NixOS release, `flake.lock` pins the exact packages, and
`configuration.nix` sets the system up for Sherpa:

- cloud-init, which applies Sherpa's accounts, SSH keys, hostname and management address on
  first boot;
- interfaces named `eth0`, `eth1`, ..., as in the manifest;
- passwordless `sudo` for the `wheel` group, which Sherpa's accounts join;
- SSH password logins allowed, as on Sherpa's other Linux nodes;
- the serial console, for `sherpa console`.

Add your own packages and services to `configuration.nix` before building.

## Build

With [Nix](https://nixos.org/download/) installed:

```
nix --extra-experimental-features "nix-command flakes" build \
  .#nixosConfigurations.sherpa-node.config.system.build.images.qemu
cp -L result/*.qcow2 /some/path/nixos.qcow2
```

Without Nix installed, build it in the official `nixos/nix` container. The build runs a small VM,
so the container needs `/dev/kvm`:

```
docker run --rm --device /dev/kvm -v "$PWD:/src" -w /src nixos/nix sh -c '
  git config --global --add safe.directory /src
  nix --extra-experimental-features "nix-command flakes" --option sandbox false \
    --option system-features "kvm nixos-test benchmark big-parallel" \
    build .#nixosConfigurations.sherpa-node.config.system.build.images.qemu
  mkdir -p out && cp -L result/*.qcow2 out/nixos.qcow2'
```

The image is written to `out/nixos.qcow2`, owned by `root`. A build takes about 5 minutes. To
reuse downloaded packages between builds, add `-v sherpa-nix-store:/nix` to keep the Nix store in
a Docker volume.

To move to newer packages within the release, run `nix flake update` before building.

## Import

```
sherpa server image import \
  --src /some/path/nixos.qcow2 \
  --model nixos_linux \
  --version 26.05 \
  --default
```

See the [NixOS page](https://github.com/bwks/sherpa-docs/blob/main/docs/nodes/linux/nixos-linux.md)
of the Sherpa docs for the node's defaults.
