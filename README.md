# reXIP

A reader **and writer** for the `XIP2` archives that the game **DJMAX Online** keeps its data in — the
song catalogue, the shop stock, the icons, the UI text, the background scenes.

The archives sit in the game files as `system.pak`, `system_0001.pak` and so on.
Extract a file out, edit it, and pack it back into a PAK file that the game loads.

The provided release EXE is a single self-contained executable.
No .NET install, no dependencies, nothing to configure.

---

## Commands

```
reXIP dump    <process> [output.bin]            copy the running game's image to a file
reXIP keys    <dump.bin>                        extract the key tables into keyFiles\
reXIP list    <archive.pak> [filter]            list what is inside
reXIP extract <archive.pak> <path> [output]     extract one file (path * extracts all)
reXIP verify  <archive.pak> [how many]          read back and check crc/sum/hash
reXIP create  <output.pak> <folder|inner=local> ...  build a new .pak
reXIP crc     <crc.pak> <output.pak>            rebuild system.crc from the folder
```

Archives are looked up in the current directory, or in whatever `DJMAX_FILES` points at, when
the path you give does not exist as written.

## How to

First, run DJMAX. You don't need to login. You can even leave the game running in the
warning pop-up that states that there is no server to connect to.
Then:

```bash
reXIP dump DJMax dump.bin
```

this will dump the DJMax process from memory. This is where the keys for the PAK files are.
Next:

```bash
reXIP keys dump.bin
```

to extract the keys from the dump you just created.
With the keys in hand, you can now, for example:

```bash
reXIP extract system.pak * patch
# edit what you want inside the patch folder, and delete everything else
reXIP create system_0005.pak patch
```

A **folder** goes in whole, keeping the names relative to it, so `patch\System\shop\` lands as
`System\shop\`. Put in it only what you actually changed: the archive you build overrides those
entries and nothing else.

For a single file there is a shorthand that skips the folder:

```bash
reXIP create system_0005.pak "System\shop\ItemStock.csv=items.csv"
```

Copy the newly created PAK file into the game files folder and start the game.
**Remember to name the new PAK accordingly.** If the highest PAK in the folder is, for example,
'system_0005.pak', the file you created must be called 'system_0006.pak'.

**You do not have to touch `system.pak` or `crc.pak`.** The client counts the `system*.pak`
files in the folder and loads them in order, the last one winning, so a new numbered archive
overrides an entry in a 224 MB original without rewriting it. The startup integrity check walks
the list *inside* `system.crc`, which does not change, and does not notice an extra archive.

`crc` is there for the case where you do modify one of the originals.

## Where it looks for archives

The folder you run the command in. Set `DJMAX_FILES` to the game's files folder to reach the
archives by name from anywhere. Example:

```bash
set DJMAX_FILES=C:\Games\DJMAX\FILES
reXIP list system.pak
```

---

## Remember: You need the client's keys

The PAK files are encrypted, so you need the keys to decrypt them.
They only exist in the running process: `DJMax.exe` on disk is packed with ASProtect, so there
is nothing to read from the file. With the game running:

```bash
reXIP dump DJMax dump.bin
reXIP keys dump.bin
```

`dump` copies the process image to a file and `keys` lifts the two tables out of it, at
`0x0055BEB0` and `0x0055B6B0`. They land in `keyFiles\` and are verified before being written —
`keys` checks that encryption is the exact inverse of decryption, and refuses to leave anything
on disk if it is not.

> **It has to be a flat image, not a minidump.** `keys` reads `address - 0x400000` as a position
> in the file, so it needs the process image copied as-is — a file that starts with `MZ`.
> Sysinternals **ProcDump writes a minidump** (`MDMP`), a container whose addresses do not map
> linearly, and reading one gives zeros. That is what `reXIP dump` is for; `keys` recognises a
> minidump and says so rather than producing wrong keys.

`keyFiles\` is also read from the directory you run the command in, so you do not have to copy
the keys next to the executable.

> Different releases of the game use different key tables. The ones from a 2007 SNDA client will
> not open a later Chinese build, and vice versa — each needs a dump of its own.

---

## Before you start

**Back up whatever you replace.** `verify` re-reads an archive and checks every entry's CRC32,
byte sum and name hash, and `create` runs that same check on its own output before it finishes
— but a wrong file alongside the game executable still stops the game from starting.

Two things worth knowing about the game's own data files:

- the client's CSV reader treats **a space as a field separator** and **`'` as the start of a
  comment**, so a title with a space in it must not be written naively;
- several files mix **two code pages on the same line** — Chinese and Korean — because that is
  how the original localisation left them.

Some files are stored masked. `extract` hands them to you readable and `create` masks them again
on the way back in, so this is invisible unless you go looking.

---

## Building from source

For a single EXE file with embed runtime:
```bash
dotnet publish reXIP.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:DebugType=none -p:EnableCompressionInSingleFile=true -p:PublishTrimmed=false -o "reXIP" --nologo
```

For executable without embed runtime:
```bash
dotnet publish reXIP.csproj -c Release -r win-x64 \
  --self-contained true -p:PublishSingleFile=true
```

---

## Additional Info

The format itself — the header, the per-entry descriptors, the RSA-like key schedule and the
LZO variant used for the blocks — is documented in the `Pak/` sources themselves,
`XipArchive.cs` and `XipFormat.cs` above all.

> The source comments are in Portuguese. They carry what was measured and how, including the
> hypotheses that turned out wrong, and they are worth reading even in translation. The tool's
> own output is in English.

---

## Part of GrooveServer

reXIP is built from the same `Pak/` sources as the `pak` command inside GrooveServer, a
private-server emulator for the same game. The two are the same code with different
defaults for where they look for keys and archives.

---

## A Big Special Thanks To
Alejandro H of ADHSoft - reXIP is based on the tool [Xip-Pak-Extractor](https://github.com/ADHSoft/Xip-Pak-Extractor).

---

## Legal

reXIP ships no game data and no keys. It contains no game code, no game assets and no client binaries.
It reads PAK files you already own, using keys taken from your own executable of the client.
DJMAX Online is © Pentavision / Neowiz. This project is not affiliated with them.
