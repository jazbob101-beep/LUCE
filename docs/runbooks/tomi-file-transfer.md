# Tomi File Transfer

## Scope

This runbook covers routine file movement to and from TT3 “Tomi” on the current TomiDock CDC-ECM network. It uses the existing LAN and the MacBook `data-share`; it does not use USB-storage mode or require routine topology changes.

The current Tomi runtime command surface is documented in [Tomi Runtime ABI](../reference/tomi-runtime-abi.md). This runbook links to that reference instead of repeating its BusyBox inventory or general compatibility rules.

## Current network context

| Device/interface | Address | Transfer role |
|---|---|---|
| DEAN-PC LAN | `192.168.1.117` | Stages files to the shared `T:` drive. |
| MacBook Wi-Fi | `192.168.1.173` | Receives Tomi SCP uploads and serves staged files over temporary HTTP. |
| ESP32-S3 Wi-Fi | `192.168.1.223` | Provides the existing LAN TCP/2323-to-Tomi-shell forward. |
| Tomi `usb0` | `192.168.77.1` | Tomi end of the ECM link. |
| ESP ECM peer | `192.168.77.2` | ESP end of the ECM link; not the MacBook file-server address. |

The existing TCP forward is `192.168.1.223:2323` to Tomi `192.168.77.1:23`. It is the normal interactive shell endpoint:

```sh
telnet 192.168.1.223 2323
```

Use the MacBook LAN address `192.168.1.173` for the file-transfer procedures below. The ESP address and Tomi's `.77.x` addresses have different roles.

The MacBook staging directory is `/home/chris/data-share` (`~/data-share`). DEAN-PC accesses the same share as `T:` or `\\macbook\data-share\`.

## Tomi to MacBook

### Canonical SCP procedure

The current proven direction is Tomi initiating SCP to the MacBook account `chris`:

```sh
/mnt/sdcard/opentom/bin/scp \
  -S /mnt/sdcard/opentom/bin/ssh \
  /path/to/FILENAME \
  chris@192.168.1.173:/home/chris/data-share/
```

The proof transfer on 2026-09-24 copied `/mnt/sdcard/opentom/tomidock-live-state-2026-09-24.txt` into the MacBook's `data-share` directory. That is operator-reported direct bench evidence; keep the source path and destination path appropriate to the file being moved.

After transfer, compare the source and destination byte counts and SHA-256 values. On Tomi, an explicit checksum command is:

```sh
/mnt/sdcard/opentom/extra/bin/busybox sha256sum /path/to/FILENAME
```

On the MacBook host, `sha256sum` and `wc -c` can be used to check the received file:

```sh
cd ~/data-share
sha256sum FILENAME
wc -c FILENAME
```

The digest and byte count must match the source. The byte count can also be read from Tomi's demonstrated `/bin/busybox ls -l /path/to/FILENAME` output. For important binaries, retain the original until both checks pass.

### Why bare SCP fails

The live Tomi `scp` binary uses `/usr/bin/dbclient` as its compiled/default SSH client path, and that file was absent during the 2026-09-24 live session. Invoking SCP without an alternate client failed with:

```text
/usr/bin/dbclient: No such file or directory
lost connection
```

The installed standalone SSH client is `/mnt/sdcard/opentom/bin/ssh`; pass it with SCP's `-S` option as shown above. Do not use bare SCP as the current method. Do not create a `/usr/bin/dbclient` symlink or assume a `tomi-scp` wrapper exists; neither is part of the adopted workflow.

Historical OpenTom build rules explain why the installed client is named `ssh`: they build `dropbear`, `dbclient`, `dropbearkey`, and `scp`, then copy `dbclient` into the distribution as `bin/ssh`. This source packaging fact is separate from the live observation of the missing compiled/default path.

## MacBook or DEAN-PC to Tomi

### Stage the file

From DEAN-PC PowerShell, copy the file into `T:`:

```powershell
Copy-Item -Force 'D:\Codex\TT3\path\FILENAME' 'T:\'
```

On the MacBook, the staged file appears in `~/data-share`. For an important file, record its expected byte count and SHA-256 there before downloading it.

### Serve the staging directory over HTTP

On the MacBook HOST, start the temporary server from the share directory:

```bash
cd ~/data-share
python3 -m http.server 8000 --bind 192.168.1.173
```

Leave it running only while the transfer is in progress. It serves files from the current directory; do not start it from a broader directory.

### Download on Tomi

Write to a temporary destination first. Use the explicit expanded BusyBox provider:

```sh
/mnt/sdcard/opentom/extra/bin/busybox wget \
  -O /desired/path/FILENAME.part \
  http://192.168.1.173:8000/FILENAME
```

Compare the temporary file's byte count and SHA-256 with the staged source. On Tomi, calculate SHA-256 with:

```sh
/mnt/sdcard/opentom/extra/bin/busybox sha256sum /desired/path/FILENAME.part
```

Use `/bin/busybox ls -l /desired/path/FILENAME.part` to inspect the target byte count. On the MacBook, use `sha256sum FILENAME` and `wc -c FILENAME` for the source values. Do not promote the temporary file unless both values match.

After verification, rename the temporary file to its final name. If a file already exists at the final path, preserve it as needed before replacement; do not discard the only known-good copy.

Stop the temporary HTTP server after the transfer and verification are complete.

## Emergency and historical fallbacks

### FTP on the historical direct topology

The following stock-BusyBox FTP server pattern is a previously proven fallback, but belongs to the historical direct-link setup:

```sh
/bin/busybox tcpsvd -vE 0.0.0.0 2121 \
  /bin/busybox ftpd -w /mnt/sdcard/opentom/results &
```

Historical MacBook retrieval used Tomi at `192.168.1.10:2121`, for example:

```bash
curl -v ftp://192.168.1.10:2121/FILENAME -o FILENAME
```

This is not the preferred current TomiDock workflow. The current TomiDock mapping documented here exposes TCP/2323 to Tomi TCP/23; this runbook does not establish that TCP/2121 is reachable through that topology. Do not use the historical FTP command as if it were a current TomiDock service.

### Telnet capture for emergency text recovery

Telnet access followed by `cat` and host-side terminal capture has been used to recover text evidence when routine transfer was unavailable. Treat this as an emergency text/log recovery method, not routine transport. A terminal capture is not accepted as a byte-exact binary copy unless its raw capture preserves the bytes and the resulting file passes an independent size and SHA-256 comparison.

## Historical topology warning

Earlier direct MacBook-to-Tomi work used:

```text
MacBook tomi0: 192.168.1.11
Tomi:          192.168.1.10
HTTP:          http://192.168.1.11:8000/
```

Those `.1.10` and `.1.11` addresses belong to the historical direct topology. They are not the current TomiDock addresses. For current HTTP staging, use MacBook Wi-Fi `192.168.1.173`; for the current interactive shell forward, use ESP32-S3 `192.168.1.223:2323`. The `.77.1` and `.77.2` addresses are Tomi/ESP endpoints on the USB ECM link, not the MacBook HTTP server.

## Operational rules

- Use the current SCP command with `-S /mnt/sdcard/opentom/bin/ssh` for Tomi-to-MacBook transfers.
- Use the MacBook HTTP server and Tomi `wget` for MacBook/DEAN-PC-to-Tomi transfers.
- For important files, download or upload to a temporary name, compare byte count and SHA-256, then promote the file.
- Keep the temporary HTTP server rooted at `~/data-share` and stop it after transfer.
- Do not introduce USB-storage mode, routine network reconfiguration, a `dbclient` symlink, or an unimplemented wrapper as part of these workflows.
- Reserve FTP and Telnet capture for the specific historical or emergency cases described above.

## Related references

- [Tomi Runtime ABI](../reference/tomi-runtime-abi.md) — authoritative Tomi command and utility compatibility.
- `/mnt/d/Codex/TomiDock_Fresh_Thread_Handoff_2026-09-24.md` — network map, share mapping, older direct-topology examples, and prior staging instructions.

## Provenance

The current address map, explicit SCP invocation, bare-SCP failure, MacBook HTTP staging flow, and successful September 24 proof transfer come from the operator-provided live-session brief. The same current network and staging addresses are present in the TomiDock fresh-thread handoff. Historical Dropbear packaging behavior is supported by the OpenTom Makefile; old `.1.10/.1.11` addressing and `wget` activity appear in the Tomi terminal transcript. See the external reconciliation report for exact source paths, classifications, duplicates, and unresolved provenance gaps.
