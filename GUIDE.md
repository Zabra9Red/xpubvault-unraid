# XPUBVAULT on Unraid — the complete guide

XPUBVAULT watches wallets. You give it an extended public key, an output
descriptor or an address; it derives, verifies and keeps every address in an
encrypted vault, and — if you point it at your own node — shows balances.
It never accepts a private key or a seed phrase, and it never holds anything
that could spend.

An extended public key is not a secret that spends, but it reveals every
address of a wallet, past and future. This guide treats it that way.

**Contents**

1. [Before you start](#1-before-you-start)
2. [Install](#2-install)
3. [The template's fields](#3-the-templates-fields)
4. [First start](#4-first-start)
5. [Everyday use](#5-everyday-use)
6. [Unlocking after an array start](#6-unlocking-after-an-array-start)
7. [Modes, and real network isolation](#7-modes-and-real-network-isolation)
8. [Balances from your own node](#8-balances-from-your-own-node)
9. [The dashboard tile](#9-the-dashboard-tile)
10. [What dashboards can see while the vault is locked](#10-what-dashboards-can-see-while-the-vault-is-locked)
11. [Backups and restores](#11-backups-and-restores)
12. [Updating](#12-updating)
13. [Maintenance from the terminal](#13-maintenance-from-the-terminal)
14. [Troubleshooting](#14-troubleshooting)
15. [Removing it](#15-removing-it)

---

## 1. Before you start

- **Unraid 6.12 or later**, on `amd64` or `arm64`.
- **A passphrase you will not lose**, of at least 12 characters. Nobody can
  recover it. Setup also gives you a **recovery code**, once: have paper
  ready.
- **Decide how the vault unlocks after a reboot** (section 6). The safest
  answer needs nothing extra: you type the passphrase.
- **Optional:** a USB stick on *Unassigned Devices*, if you want unattended
  starts with a keyfile; a second machine running `tang`, if you want
  network-bound unlocking; your own Electrum server or node, if you want
  balances.

### Check the image first (optional, recommended)

The image is signed by the workflow that built it. With
[cosign](https://docs.sigstore.dev/cosign/system_config/installation/) on any
machine:

```sh
cosign verify ghcr.io/zabra9red/xpubvault@sha256:fb04441eec74672c683670d5205c5285d5cc5ab02974fd4399768d274c532b92 \
  --certificate-identity https://github.com/Zabra9Red/xpubvault/.github/workflows/release.yml@refs/tags/v0.16.2 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

The template pins the image to that digest, so what you verified is what
Unraid pulls.

---

## 2. Install

### 2.1 The template

In Unraid's terminal:

```sh
wget -O /boot/config/plugins/dockerMan/templates-user/my-XPUBVAULT.xml \
  https://raw.githubusercontent.com/zabra9red/xpubvault-unraid/main/xpubvault.xml
```

### 2.2 The data folder — do this before the first start

```sh
mkdir -p /mnt/user/appdata/xpubvault
chown -R 10999:10999 /mnt/user/appdata/xpubvault
chmod 700 /mnt/user/appdata/xpubvault
```

The container runs as its own user, `10999`, not as Unraid's shared `99`,
and refuses a data folder that anyone else can read. If you skip this, it
stops at once and prints the exact command to run.

> Unraid's **New Permissions** tool makes shares readable by everyone. After
> using it, run the `chown` and `chmod` lines again.

### 2.3 Add the container

**Docker → Add Container → Template → XPUBVAULT** (under *User templates*).
Leave the defaults for a first run and press **Apply**.

Do not edit **Extra Parameters** (switch to *Advanced View* to see it):

```
--user 10999:10999 --read-only --tmpfs /tmp:size=64m,noexec,nosuid,nodev --cap-drop=ALL --security-opt=no-new-privileges:true --ulimit memlock=67108864 --ulimit core=0
```

That line is the container's hardening: its own user, a read-only root
filesystem, no capabilities, no new privileges, a `noexec` `/tmp`, no core
dumps, and the locked-memory budget that keeps keys out of swap. Without it
the container runs with Docker's defaults.

---

## 3. The template's fields

| Field | Default | What it is |
|---|---|---|
| **WebUI Port** | `8760` | HTTPS. The container generates its own certificate; you check its fingerprint once (section 4). |
| **Data** | `/mnt/user/appdata/xpubvault` | Vault, database, configuration. Owned by `10999`, mode `700`. |
| **Mode** | `AIRGAP` | `AIRGAP`, `LOCAL` or `PUBLIC` (section 7). `AIRGAP` makes no outbound connection at all. |
| **Keys** | *(empty)* | Optional. A keyfile for unattended starts, and the widget key when the dashboard projection is on. **Must be a different device from appdata** (section 6). |
| **Keyfile** *(advanced)* | *(empty)* | The keyfile's name under `/keys`, e.g. `xpubvault.key`, to unlock with at every start. |
| **Tang at start** *(advanced)* | `0` | `1` unlocks at every start through the vault's Tang servers; needs Mode `LOCAL`. |
| **Host names** *(advanced)* | *(empty)* | Names you open the WebUI by, comma-separated (`tower.lan,vault.home`). IP addresses always work. |

Two settings have no field; add them with **Add another Path, Port,
Variable…** only if you need them:

| Variable | Default | What it does |
|---|---|---|
| `XV_IDLE_LOCK_MINUTES` | `60` | Locks the vault after this long without a request. |
| `XV_ALLOW_PUBLIC` | *(unset)* | `1` makes Mode `PUBLIC` selectable at all. |

There is **no field for a passphrase**, and you should not add one
(`XV_PASSPHRASE`): it would sit in `docker inspect`, in the template file on
the flash drive, and so in every flash backup.

---

## 4. First start

1. **Read the log** (Docker → the container's icon → *Logs*). It shows two
   things:
   - a **setup code**, valid for one hour and usable once;
   - the **SHA-256 fingerprint** of the certificate it generated.
2. **Open the WebUI** (the container's icon → *WebUI*, or
   `https://<server address>:8760/`). The browser warns about the
   certificate, because no authority signed it. Open the certificate's
   details and **compare its SHA-256 fingerprint with the one in the log**.
   If they match, accept it. That comparison is what protects your
   passphrase on the way to the server.
3. **Enter the setup code.** This enrols the browser you are using; nobody
   else can claim the instance afterwards without a code from the log.
4. **Choose the passphrase.** At least 12 characters; a strength meter tells
   you when it is acceptable. Four or more random words work well.
5. **Write the recovery code down** when it is shown, then **type it back**.
   Nothing is created until you do. The recovery code opens the vault when
   the passphrase is lost, and repairs it after a bad restore. It is shown
   once.
6. **Choose what dashboards may see while the vault is locked.** The default
   is **Off**: nothing. Leave it there unless you want the dashboard tile to
   show totals after a reboot (section 10).

The vault is now set up and unlocked.

---

## 5. Everyday use

The WebUI has four screens.

**Add** — one box, no type selector. Paste or drop anything:

- an xpub / ypub / zpub, with or without its key origin;
- an output descriptor;
- an address, or a whole list of them;
- a wallet export (Sparrow, Electrum, Bitcoin Core, …) or a text file;
- a QR image, or — with **Scan with the camera** — a QR code held up to the
  camera, including the animated ones Sparrow, Keystone and Passport show;
- an `.xvault` bundle from the phone app or another vault (it asks for its
  export code).

A clean item is watched at once, with **Undo** for 30 seconds. A list
becomes a table whose rejected rows can be retried. A key that could be
several things asks which. **A private key or a seed phrase is refused and
the box is cleared** — clear it wherever else you pasted it.

**Wallets** — each wallet's next receive address with its QR code (**New
address** moves on, deliberately), and every address with its path. Balances
read "no backend" until you add one (section 8); they are never shown as
zero when they are simply unknown.

**Vault** — the encrypted files as they are on disk; a downloadable archive;
a hand-off viewer; an `.xvault` bundle of the watch list for the phone app;
a plaintext export, behind the passphrase again.

**Settings** — mode, data backends, the passphrase and other ways to unlock,
devices, widgets, the certificate's fingerprint.

**Lock now**, in the header, locks the vault. It also locks by itself after
60 minutes without a request, and — always — when the container restarts.

### Other browsers and devices

**Settings → Devices → Pair a device** gives a pairing code for another
browser. Each device is listed and can be revoked on its own.

---

## 6. Unlocking after an array start

The vault starts **locked**. Choose how it opens — safest first.

### 6.1 Your passphrase (the default)

Open the WebUI after each start and type it. Nothing to configure, and
nothing on the server can open the vault without you.

### 6.2 A keyfile on a separate device

For restarts with nobody there. The keyfile must live **somewhere a thief of
the disks does not get**: a USB stick mounted by *Unassigned Devices*, or a
tmpfs filled at array start from somewhere that is not the flash drive.

A keyfile in a folder under `/mnt/user/appdata` sits beside the vault it
opens, and inside every appdata backup: it is refused.

1. Mount the stick with Unassigned Devices, e.g. at `/mnt/disks/vaultkey`,
   and make it the container user's:

   ```sh
   chown 10999:10999 /mnt/disks/vaultkey && chmod 700 /mnt/disks/vaultkey
   ```

2. Create the keyfile, once:

   ```sh
   docker run --rm --network none --user 10999:10999 --read-only --cap-drop ALL \
     -v /mnt/disks/vaultkey:/keys \
     ghcr.io/zabra9red/xpubvault:0.16.2 xpubvault keyfile new /keys/xpubvault.key
   ```

3. Edit the container: set **Keys** to `/mnt/disks/vaultkey`. Apply.
4. In the WebUI, unlocked: **Settings → Encryption → Key slots**, add a
   keyfile slot (it asks for the passphrase again).
5. Edit the container again: set **Keyfile** to `xpubvault.key`. Apply.

The vault now unlocks itself **at start** — and only at start: after an idle
lock it stays locked until you unlock it.

If the stick is on the same physical disk as appdata, the vault still
unlocks but shows `KEYFILE_SAME_DEVICE` for as long as it runs: against a
stolen disk that keyfile protects nothing. A keyfile readable by group or
other is refused (`chmod 400` it).

Pull the stick after boot if you can. A key device left plugged in beside
the disks is taken with them.

### 6.3 Tang (network-bound)

The vault unlocks while a `tang` server on your network answers, and never
once its disks have left that network.

1. Run `tang` on a second machine.
2. Set **Mode** to `LOCAL`.
3. In the WebUI: **Settings → Encryption → Tang server**. Trust the
   key whose thumbprint `tang-show-keys` prints on the Tang machine.
4. Set **Tang at start** to `1`.

It fails safe: no Tang server answering means locked. Firewall the Tang
server so that only this server's address reaches it.

### 6.4 `XV_PASSPHRASE` — do not

Possible, and the service warns loudly when it is used. The passphrase then
appears in `docker inspect`, in the template XML under
`/boot/config/plugins/dockerMan/templates-user/`, and in every flash backup —
which many people upload to Unraid Connect or a cloud drive.

---

## 7. Modes, and real network isolation

| Mode | Outbound connections |
|---|---|
| `AIRGAP` | None. No balances, no sync. |
| `LOCAL` | Only to backends and Tang servers **you** configure, and only on private address ranges. |
| `PUBLIC` | Also public servers; every host reached is logged. Needs `XV_ALLOW_PUBLIC=1` and a confirmation in the UI. |

Change it in **Settings → Mode** or with the template's **Mode** field.

On Unraid's `bridge` network, `AIRGAP` is enforced by the application, and
the header says so: *AIRGAP (application-enforced)*, in amber. For isolation
by the network itself:

- a custom `br0` address on a VLAN whose firewall drops all egress except
  the backends you configure; or
- `--network none` in Extra Parameters — but then there is no WebUI either;
  that is for one-off terminal commands (section 13).

The service cannot see a firewall, so the amber label stays either way.

---

## 8. Balances from your own node

Balances come from servers you run, and from nothing else.

1. Set **Mode** to `LOCAL`.
2. **Settings → Data backends → Add a backend**. It asks for the passphrase
   again, because it changes where your addresses are sent.

| Kind | Serves |
|---|---|
| `electrum` (electrs, Fulcrum, ElectrumX) | Bitcoin and its forks, with history |
| `esplora`, `blockbook` | Bitcoin and its forks, with history |
| `bitcoin-core` | Balances by scanning the UTXO set, on request; no history |
| `evm` (Erigon, Reth, Nethermind, Geth) | An EVM chain, by chain id; ERC-20 too |
| `cosmos` | The Cosmos chains |
| `monero-wallet-rpc` | Monero view-only wallets |

Name the backend by IP address or host name. TLS can be `none` (your LAN),
`pinned` — **Read it from the server** fetches the certificate's fingerprint
for you to compare — or `ca`.

Syncs run when something is added, when the vault unlocks, every 30 minutes,
and on **Sync now**. A backend that knows history also finds addresses past
the gap window. Bitcoin Core alone cannot tell a spent-out address from an
unused one, and the wallet says so (`PARTIAL`) rather than stopping silently.

---

## 9. The dashboard tile

Optional, and separate from the container: a tile on Unraid's dashboard with
the wallet count, the totals and the sync age.

1. **Plugins → Install Plugin**, with:

   ```
   https://raw.githubusercontent.com/zabra9red/xpubvault-unraid/main/xpubvault.plg
   ```

   Every file is inside the plugin; nothing else is downloaded.

2. In the WebUI: **Settings → Widgets → New token**, scope `summary`. Copy
   the token — it is shown once.

3. Put it in a file **on the array, never on the flash drive** (the tile
   refuses `/boot`):

   ```sh
   mkdir -p /mnt/user/appdata/xpubvault-tile
   printf '%s\n' 'PASTE-THE-TOKEN-HERE' > /mnt/user/appdata/xpubvault-tile/widget.token
   chmod 600 /mnt/user/appdata/xpubvault-tile/widget.token
   ```

4. **Settings → Utilities → XPUBVAULT**, and fill in:

   | Setting | Value |
   |---|---|
   | Service address | `https://<server address>:8760` — an address the certificate covers (an IP address, or one of **Host names**) |
   | Token file | `/mnt/user/appdata/xpubvault-tile/widget.token` |
   | Certificate | `/mnt/user/appdata/xpubvault/tls/cert.pem` |

A widget token can read the summary and nothing else: not an address, not a
key, not a label. Revoke it in **Settings → Widgets** at any time; other
tokens keep working.

What the tile shows:

| Vault | Tile |
|---|---|
| Unlocked | Live figures |
| Locked, projection on | The last figures, with a lock and their age |
| Locked, projection off | "locked" |
| Locked, projection older than its lifetime | "stale", no figures |

---

## 10. What dashboards can see while the vault is locked

Nothing, by default. **Settings → Widgets** offers three choices:

- **Off** — a locked vault answers "locked" and nothing else.
- **Aggregates** — per-coin totals, rounded to four significant figures, and
  only the counts you tick. No address, key, label, path or transaction.
- **Aggregates + next address** — the same, plus one receive address of the
  wallet you pick, under an alias you give it. It does not rotate while the
  vault is locked.

> Whatever is visible without the passphrase is, by definition, not protected
> by the passphrase. The projection is protected by the device it sits on,
> not by anything you know.

That copy is encrypted under a key of its own, and where that key lives
decides what it is worth against a stolen disk — best first:

1. **Sealed to the TPM**: append to **Extra Parameters**
   `--device=/dev/tpmrm0 --group-add <gid>`, where `<gid>` is the group that
   owns the device (`stat -c %g /dev/tpmrm0` in the Unraid terminal). There
   is no TPM field in the template on purpose: Unraid passes every field to
   `docker run`, and an empty device field stops the container from being
   created at all.
2. **On the Keys device**, when **Keys** is a different device from appdata.
3. **On the data volume** — against a stolen disk this is equivalent to
   plaintext, and the WebUI says so on every page.

The copy expires after 7 days. It is destroyed when you turn it off, press
**Purge now**, rekey, or after 10 wrong passphrases in a row.

---

## 11. Backups and restores

**What to back up:** `/mnt/user/appdata/xpubvault/db` and
`/mnt/user/appdata/xpubvault/vault`, **together, from the same moment**. They
are one unit. Everything in them is ciphertext, plus a header and a manifest
that name nothing.

**What to leave out:**

- `/mnt/user/appdata/xpubvault/widget/` — the dashboard copy; it belongs to
  the machine.
- the **Keys** device — a keyfile in the same backup as the vault it opens
  defeats both.

`rsync -a`, restic, the appdata backup plugins and the mover are all safe:
a changed record always has a new modification time.

**A backup keeps the credentials of its day.** Changing the passphrase later
does not lock an older backup; it still opens with the passphrase it was
made under.

**Restoring:** stop the container, restore `db/` and `vault/` from the
**same** snapshot, fix the ownership (`chown -R 10999:10999`, `chmod 700`),
start. If the two halves come from different snapshots the vault refuses to
open (`E_STORE_TAMPERED`) and names what is stale; `xpubvault repair`
(section 13) recovers what survives, with the recovery code.

**A portable copy:** **Vault → Download the encrypted archive** in the
WebUI. It restores into an empty data folder with `xpubvault import vault`.

---

## 12. Updating

The template names an exact version and digest, never `latest`, so nothing
changes under you. To update:

1. Read what changed, and verify the new image (section 1).
2. Fetch the new template over the old one:

   ```sh
   wget -O /boot/config/plugins/dockerMan/templates-user/my-XPUBVAULT.xml \
     https://raw.githubusercontent.com/zabra9red/xpubvault-unraid/main/xpubvault.xml
   ```

   or edit the container and change **Repository** to the new
   `…:<version>@sha256:<digest>` yourself.
3. Back up first (section 11), then **Apply**. The vault starts locked.

Your own settings in the template (Keys, Keyfile, Mode, Host names) are
overwritten by step 2's download: note them first, or use the edit route.

---

## 13. Maintenance from the terminal

The image is also the command-line tool. **Stop the container first**, then
run a one-off with the same hardening and no network:

```sh
xv() {
  docker run --rm -it --network none --user 10999:10999 --read-only --cap-drop ALL \
    --security-opt no-new-privileges:true --tmpfs /tmp:size=64m,noexec,nosuid,nodev \
    --ulimit core=0 --ulimit memlock=67108864 \
    -v /mnt/user/appdata/xpubvault:/data \
    ghcr.io/zabra9red/xpubvault:0.16.2 xpubvault "$@"
}
```

Passphrases and codes are asked for at the terminal, without echo. They are
never taken from the command line.

| Command | When |
|---|---|
| `xv verify --data /data --deep` | Check every file against the manifest. |
| `xv auth reset --data /data` | **Locked out of the WebUI** — every enrolled browser lost. Revokes all devices and prints a new setup code. Widget tokens are kept. |
| `xv repair --data /data` | After `E_STORE_TAMPERED`. Reports first and changes nothing; add `--apply` to rebuild over what survives. Needs the **recovery code**. |
| `xv rekey --data /data` | A passphrase or keyfile **may be compromised**. New key, everything re-encrypted, a new passphrase and a new recovery code. Keyfile slots are dropped. Older backups still open with the old credentials — destroy them. |
| `xv export vault --data /data --out /data/vault.tar.zst` | The encrypted archive, from the terminal. |
| `xv targets --data /data` | The watch list, as JSON. |

Start the container again afterwards.

---

## 14. Troubleshooting

**The container stops at once.** Read its log. It names the problem and the
command: almost always the data folder's ownership or mode (section 2.2), or
a data folder mounted read-only.

**`docker pull` answers `denied`.** The image has not been made public yet,
or the digest in the template is not the published one.

**`docker: bad format for path:` when you press Apply.** The container was
added from the 0.16.1 template, whose optional **TPM** field Unraid passes as
`--device=''` when it is left empty. Fetch the current template again
(section 2.1), or edit the container, remove the **TPM** field (it is under
**Show more settings**) and Apply. With the 0.16.1 image, remove an empty
**Keyfile** field too: that version reads it as a keyfile with no name and
stops at once. 0.16.2 treats an empty field as a field not set.

**The browser warns about the certificate.** Expected: the container made it
itself. Compare the fingerprint with the log (section 4), or with **Settings
→ certificate fingerprint** from a browser you already trust.

**"This service does not answer to that host name."** You opened the WebUI
by a name the certificate does not cover. Use the IP address, or add the name
to **Host names** and restart.

**The setup code is refused.** It is valid for one hour and works once.
Restart the container for a new one — or, if the instance was already
claimed, see `xv auth reset`.

**A warning that keys may reach swap.** Extra Parameters lost
`--ulimit memlock=67108864`. Restore the whole line (section 2.3).

**`KEYFILE_REFUSED` at start.** The keyfile is under the data folder, or
readable by group or other (`chmod 400`), or the name in **Keyfile** is
wrong. The vault stays locked and waits for the passphrase.

**`KEYFILE_SAME_DEVICE`.** The keyfile is on the same disk as appdata
(section 6.2). It works; it protects nothing against that disk being taken.

**Balances say "no backend".** No backend has answered for that chain: Mode
is `AIRGAP`, or none is configured, or it is unreachable (**Settings → Data
backends** shows its last error).

**The tile says "locked" after a reboot.** The projection is off (section
10), which is the default — or the vault is simply locked and you chose to
show nothing.

**The passphrase is lost.** Unlock with the **recovery code** (the unlock
page offers it), then set a new passphrase in Settings.

**The passphrase and the recovery code are both lost.** The vault cannot be
opened by anyone. What it watched is public data you can paste in again.

**After Unraid's New Permissions tool the container will not start.** Run
the `chown` and `chmod` lines of section 2.2 again.

---

## 15. Removing it

1. **Docker → XPUBVAULT → Remove** (with the image, if you like).
2. Delete `/boot/config/plugins/dockerMan/templates-user/my-XPUBVAULT.xml`.
3. Remove the tile under **Plugins**, and its token file.
4. Delete `/mnt/user/appdata/xpubvault` — **this is the vault**. It is
   encrypted; without the passphrase, recovery code or a keyfile it is
   unreadable. Old backups of it remain readable to whoever has the
   credentials of their day.
5. Wipe or destroy the keyfile device.
