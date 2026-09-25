# Tomi Runtime ABI

## Scope

This is the current command and utility compatibility reference for TT3 “Tomi”. It describes the live baseline reported on 2026-09-24 and separates that observation from historical image and session evidence. “ABI” here means the practical user-space command, shell, and utility surface; it does not describe a compiled binary calling convention.

The 2026-09-24 inventories below were captured directly from the live device and are authoritative for the current baseline. They are applet inventories, not a guarantee that every option or behavior commonly associated with an applet is supported.

## Operating assumptions

- Prefer commands and options directly established on Tomi.
- Use the explicit expanded BusyBox path when you need an applet that stock BusyBox does not provide, or when the selected implementation must be unambiguous:
  `/mnt/sdcard/opentom/extra/bin/busybox APPLET ...`
- The expanded BusyBox applet set captured below is a strict superset of the stock applet set. This does not mean it has every GNU option or behavior.
- Bare root-filesystem command names may resolve through stock BusyBox applet links. For reproducible instructions, use an explicit binary/applet path when the provider matters.
- Do not infer standalone OpenTom executables from either BusyBox inventory.
- Use conservative shell syntax and avoid relying on unrecorded builtins. The current capture lists `ash`, `sh`, and `bash` applets but does not include a current shell-builtin inventory.
- Do not import command availability or compatibility from another TomTom model as if it were Tomi evidence.

## BusyBox builds

### Stock

Path: `/bin/busybox`  
Reported version: `BusyBox v1.22.1 (2026-08-18 12:24:53 UTC) multi-call binary.`

Exact live applet inventory captured 2026-09-24:

```text
[
[[
arp
ash
basename
bash
cat
chmod
cp
cut
date
depmod
df
dmesg
du
echo
env
false
fbset
fdflush
ftpd
getty
grep
halt
head
hostid
hwclock
ifconfig
ifdown
ifenslave
ifplugd
ifup
insmod
ip
kill
killall
ln
login
ls
lsmod
lspci
lsusb
mkdir
mknod
modinfo
modprobe
more
mount
mv
netstat
nice
pidof
ping
poweroff
ps
pwd
reboot
rm
rmdir
rmmod
route
sed
setserial
sh
sleep
strings
stty
swapoff
swapon
sync
tcpsvd
telnetd
test
touch
true
tty
udhcpd
umount
uname
vi
watchdog
wc
wget
```

### Expanded

Path: `/mnt/sdcard/opentom/extra/bin/busybox`  
Reported version: `BusyBox v1.22.1 (2026-08-19 00:26:06 UTC) multi-call binary.`

Exact live applet inventory captured 2026-09-24:

```text
[
[[
arp
ash
basename
bash
cat
chmod
cp
cpio
cut
date
depmod
df
dmesg
du
echo
env
false
fbset
fdflush
ftpd
getty
grep
gunzip
gzip
halt
head
hostid
hwclock
ifconfig
ifdown
ifenslave
ifplugd
ifup
insmod
ip
kill
killall
ln
login
ls
lsmod
lspci
lsusb
md5sum
mkdir
mknod
modinfo
modprobe
more
mount
mv
netstat
nice
pidof
ping
poweroff
ps
pwd
reboot
rm
rmdir
rmmod
route
sed
setserial
sh
sha256sum
sleep
strings
stty
swapoff
swapon
sync
tar
tcpsvd
telnetd
test
touch
true
tty
udhcpd
umount
uname
unzip
vi
watchdog
wc
wget
zcat
```

### Expanded-only delta

The captured expanded-only applets are exactly:

```text
cpio
gunzip
gzip
md5sum
sha256sum
tar
unzip
zcat
```

No applet was reported as stock-only. In particular, `mv` and `chmod` are in both inventories.

## Preferred command policy

Use the stock applet set for basic commands when its behavior is sufficient. Use the explicit expanded BusyBox for expanded-only applets and for commands where selecting the implementation matters. Examples:

```sh
/mnt/sdcard/opentom/extra/bin/busybox sha256sum FILE
/mnt/sdcard/opentom/extra/bin/busybox md5sum FILE
```

Do not treat applet presence as proof of support for a flag. Prefer explicit paths in scripts and runbooks that must work independently of `PATH` or applet symlinks.

## Known unavailable commands

The 2026-09-24 Tomi compatibility capture records these commands as unavailable:

```text
tail
find
awk
expr
setsid
nohup
usleep
tr
httpd
nc
which
file
```

Use only a demonstrated alternative for the task at hand. Do not replace one unavailable command with another familiar utility unless that utility is separately established.

## Unestablished commands: do not assume

The following commands are not established as available in the current Tomi runtime and must not be used in generated Tomi instructions unless separately verified:

```text
readlink
stat
sort
uniq
xargs
hexdump
od
realpath
command -v
```

This category is deliberately different from the proven-unavailable list above. Lack of current proof is not proof of absence. An earlier USB-mode report found `readlink` unavailable in both stated BusyBox locations, but current standalone command resolution was not captured.

## Known option and behavior constraints

- `head -n 1` works; `head -1` does not.
- Expanded BusyBox `sha256sum` does not support `-c`. Hash files individually with the explicit expanded BusyBox path and compare the returned digest to the expected value.
- Do not assume GNU-only flags or that every common BusyBox option is enabled.
- Plain `grep` is available. The 2026-09-24 capture does not establish `grep -E`, `grep -A`, or `grep -B`; avoid those forms unless separately verified.
- `sleep` is present. Fractional-second support is not established; do not assume it.
- `ls -ld` was observed working in the current live session. Other `ls` option combinations remain subject to the general conservative-options rule.
- The current capture lists `printf` neither as a BusyBox applet nor as a separately confirmed shell builtin. A prior exact-image test found no `printf` applet or builtin in that tested V003 image. Avoid `printf` in portable Tomi instructions unless the live shell has been checked.
- `cat` is in both captured BusyBox applet lists. A past smoke procedure preferred `while IFS= read -r` for a particular file-display workflow; that was a conservative choice, not evidence that `cat` is absent.

## Hashing

Use the explicit expanded multicall binary for SHA-256:

```sh
/mnt/sdcard/opentom/extra/bin/busybox sha256sum FILE
```

The expanded build also provides `md5sum`. Its options beyond ordinary file hashing are not established here. Do not use `sha256sum -c`; the captured expanded BusyBox does not support that option.

An established wrapper is also available at:

```text
/mnt/sdcard/opentom/extra/bin/sha256sum
```

The explicit BusyBox invocation remains preferred in canonical instructions because it makes the provider unambiguous.

## Timing helpers

BusyBox `sleep` is present. Do not assume fractional sleeps. A separate executable named `tomi-usleep` was installed at `/mnt/sdcard/opentom/tomi-usleep` and successfully invoked in a 2026-08-23 terminal session. Its current presence and behavior were not established by the 2026-09-24 capability capture, so reverify it if the runtime or image has materially changed before depending on it.

## Standalone OpenTom utilities

These are standalone binaries, not BusyBox applets. The 2026-09-24 live observation reported:

| Path | Size | Reported observation |
|---|---:|---|
| `/mnt/sdcard/opentom/bin/scp` | 26,496 bytes | Present; supports `-S program`. |
| `/mnt/sdcard/opentom/bin/ssh` | 174,564 bytes | Present; `ssh -V` reports `Dropbear v2016.74`. |
| `/mnt/sdcard/opentom/bin/dropbear` | 182,708 bytes | Present. |
| `/mnt/sdcard/opentom/bin/dropbearkey` | 105,768 bytes | Present. |

No `/mnt/sdcard/opentom/bin/dbclient` was present at capture time. The standalone `scp` contains the literal compiled/default client path `/usr/bin/dbclient`; its `-S program` option is part of the observed tool surface. This document does not prescribe a complete file-transfer workflow.

## Command-generation rules

When writing Tomi-side commands:

1. Select a command from the captured stock or expanded inventory, or cite separate evidence for a standalone utility.
2. Use an explicitly demonstrated option form; otherwise keep arguments plain and minimal.
3. Use the expanded BusyBox path when calling `sha256sum`, `md5sum`, or an expanded-only applet.
4. Avoid the unavailable commands listed above, and do not assume commands in the unestablished list without separate proof.
5. Avoid `head -1`, `sha256sum -c`, unverified grep context/extended options, and fractional sleep assumptions.
6. Keep current runtime facts distinct from image-specific historical tests and older terminal sessions.

## Related runbooks

The [Tomi File Transfer runbook](../runbooks/tomi-file-transfer.md) owns the current SCP/SSH and HTTP-staging procedures; use this reference for command availability and compatibility.

## Provenance

Current live baseline: operator-provided direct-device capture dated 2026-09-24, preserved with the migration request; the captured inventories and observations above are presented as current bench facts. This reference does not claim independent reacquisition during migration.

Historical compatibility evidence: `/mnt/d/Codex/usbmode-authoritative-state-v004-blackbox-compat-20260903/COMMAND_COMPATIBILITY.md` and `EXACT_RUNTIME.json` describe an exact BusyBox extracted from a hash-verified V003 image, exercised under ARM emulation. Those results are image-scoped and do not replace the 2026-09-24 live inventories.
