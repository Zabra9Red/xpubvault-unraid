# XPUBVAULT for Unraid

The Community Applications template and the optional dashboard tile for
**XPUBVAULT**, a watch-only wallet vault: paste an extended public key, an
output descriptor or an address, and it derives, verifies and keeps every
address in an encrypted vault. It never accepts a private key or a seed,
and it never holds anything that could spend.

This repository holds only what Unraid needs. The application's source is
not public.

**[The complete guide](GUIDE.md)** — install, first start, unlocking after a
reboot, balances from your own node, the dashboard tile, backups, updates,
maintenance and troubleshooting.

| File | What it is |
|---|---|
| `xpubvault.xml` | The container template |
| `icon.png` | Its icon |
| `xpubvault.plg` | The optional dashboard tile (every file inline, nothing downloaded at install) |

## The image

The template pulls `ghcr.io/zabra9red/xpubvault:0.16.1`, pinned to the
digest it was signed under:

```
sha256:e3ba0fefba4f6cf2b9063c40dc2bb2a700248e8409060555c424b43ae21f38ea
```

It is built for `linux/amd64` and `linux/arm64` from vendored sources with
the network off, and signed by the workflow that built it — keyless, in the
public transparency log. To check it before you run it
([cosign](https://docs.sigstore.dev/cosign/system_config/installation/)):

```sh
cosign verify ghcr.io/zabra9red/xpubvault@sha256:e3ba0fefba4f6cf2b9063c40dc2bb2a700248e8409060555c424b43ae21f38ea \
  --certificate-identity https://github.com/Zabra9Red/xpubvault/.github/workflows/release.yml@refs/tags/v0.16.1 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

If `docker pull` answers `denied`, the image has not been switched to public
yet.

## Install

1. Copy `xpubvault.xml` to `/boot/config/plugins/dockerMan/templates-user/`
   on the Unraid server, then **Docker → Add Container** and choose
   XPUBVAULT.
2. Before the first start, in a terminal:

   ```sh
   mkdir -p /mnt/user/appdata/xpubvault
   chown -R 10999:10999 /mnt/user/appdata/xpubvault
   chmod 700 /mnt/user/appdata/xpubvault
   ```

   Unraid's *New Permissions* tool makes shares readable by everyone; run
   the two commands again after using it.
3. Start the container and read its log: it prints a one-time **setup code**
   and the SHA-256 **fingerprint** of the certificate it generated.
4. Open the WebUI (HTTPS, port 8760), compare the certificate's fingerprint
   with the one in the log, and enter the setup code.

## Do not trim Extra Parameters

```
--user 10999:10999 --read-only --tmpfs /tmp:size=64m,noexec,nosuid,nodev --cap-drop=ALL --security-opt=no-new-privileges:true --ulimit memlock=67108864 --ulimit core=0
```

That line is the container's hardening: its own user, a read-only root, no
capabilities, no new privileges, no core dumps, and the locked-memory budget
that keeps keys out of swap. Without it the container runs with Docker's
defaults.

## Keys path

Leave **Keys** empty unless you want unattended starts. If you set it, it
must be a different device from appdata — an Unassigned Devices mount, a USB
stick — never a folder under `/mnt/user/appdata`: a keyfile beside the vault
it opens protects nothing against a stolen disk.

## The dashboard tile

Install `xpubvault.plg` from **Plugins → Install Plugin** with the URL

```
https://raw.githubusercontent.com/zabra9red/xpubvault-unraid/main/xpubvault.plg
```

It reads the service's summary with a widget token you create in the web UI
(Settings → Widgets). While the vault is locked it shows only what you chose
to allow there — nothing by default.

## Licence

Apache-2.0, for the files in this repository.
