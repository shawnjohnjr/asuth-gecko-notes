This file contains notes/snippets as I go.  I previously did some basic
investigation and foolishly stored everything in ephemeral gedit buffers which
ended up disappearing, so... not making that mistake again, I hope:

## Setting up local stuff given crash-stats info.
- Click on the linked Build ID from crash-stats.  This will be a date.  Also,
  looking above that line, you'll see a "version".  Also, the "Android CPU ABI"
  is of note.
- This results in a Release catalog.
  - For android, there should be a "Fennec" tab which immediately filters things
    down.  There may be multiple versions displayed.  Ignore the version you
    don't care about.
  - If the CPU ABI was "armeabi-v7a", that means we want the android-api-16
    platform, I think.  And I think v8 or something means aarch64 maybe. Unsure.
  - We probably also want the en-US build.
- Download the APK.
- unzip the APK somewhere.  It's just a zipfile.
- *the libraries are all xz compressed*, so we need to unxz them.
- I've renamed them to have .xz extensions and then run 'unxz' on them.

You can now find the interesting files in a few places:
- ./assets/armeabi-v7a/libxul.so
- ./lib/armeabi-v7a/libmozglue.so

You can know run `r2` against one of those files.

### Symbols

Er, I somehow had found a file like
fennec-61.0a1.en-US.android-aarch64.crashreporter-symbols.zip last time, but
I'm not sure where it came from.  However, the "modules" tab of the crash
reporter, does link directly to the symbol file when available.

## Settings, which you can put in your ~/.radare2rc file

### turn on pseudocode
```
e asm.pseudo = true
```

### tell it where to find the Mozilla symbol server.
```
e pdb.autoload = 1
e pdb.extract = 1
e pdb.server = https://symbols.mozilla.org
```
