A single .exe binary which runs DOOM on DOS 6, Windows 95 and Windows 10 (and probably everything in between).

## Introduction

DOS and Windows .exe files both start with a common header (the DOS "MZ" header), which might suggest that you can run programs for one OS on the other one. However, this is complicated by a few facts: DOS support was dropped in Windows a long time ago (DOS binaries don't load on Windows 10, for example), and DOS programs use a completely different execution environment.

This repo builds a single .exe file with a "polyglot" MZ header which allows it to load one program (the original DOOM) when running inside DOS, and a different program (a custom static build of Chocolate DOOM) when running on Windows.

## The DOS binary

The DOS binary, `DOOMD.EXE`, is exactly the original DOOM for DOS, derived from the shareware version that was widely distributed. The only difference is that it has been packed with UPX to make it a bit smaller.

## The Windows binary

The Windows binary, `DOOMW.EXE`, is a custom build of Chocolate DOOM, a source port of DOOM which hews very closely to the original. Chocolate DOOM is based on libSDL 1.2; this custom build integrates it statically rather than linking to libSDL dynamically.

### Build instructions

0. `apt install mingw-w64 build-essential` on an Ubuntu 18.04 machine.
1. Download and unpack the source code for Chocolate DOOM 2.2.1, `SDL` 1.2.15, `SDL_mixer` 1.2.7, and `SDL_net` 1.2.8.
2. `cd SDL-1.2.15; ./configure --enable-shared --enable-static --disable-directx --host=i686-w64-mingw32; make; sudo make install`
2. `cd SDL_mixer-1.2.7; ./configure --enable-shared --enable-static  --host=i686-w64-mingw32; make; sudo make install`
3. `sudo mv /usr/local/lib/libSDL_mixer.* /usr/local/cross-tools/i386-mingw32/lib/; sudo mv /usr/local/include/SDL/SDL_mixer.h /usr/local/cross-tools/i386-mingw32/include/SDL/` (I could probably also have just set prefix correctly)
4. Apply this patch to `SDL_net` to eliminate the dependency on `iphlpapi.dll`:

```
diff -ur a/SDL_net-1.2.8/SDLnet.c SDL_net-1.2.8/SDLnet.c
--- a/SDL_net-1.2.8/SDLnet.c	2012-01-15 16:20:10.000000000 +0000
+++ b/SDL_net-1.2.8/SDLnet.c	2020-11-13 18:49:24.602850922 +0000
@@ -212,6 +212,7 @@
 		return 0;
     }
 
+#if 0
     if ((dwRetVal = GetAdaptersInfo(pAdapterInfo, &ulOutBufLen)) == ERROR_BUFFER_OVERFLOW) {
         pAdapterInfo = (IP_ADAPTER_INFO *) SDL_realloc(pAdapterInfo, ulOutBufLen);
         if (pAdapterInfo == NULL) {
@@ -219,6 +220,9 @@
         }
 		dwRetVal = GetAdaptersInfo(pAdapterInfo, &ulOutBufLen);
     }
+#else
+    return 0;
+#endif
 
     if (dwRetVal == NO_ERROR) {
         for (pAdapter = pAdapterInfo; pAdapter; pAdapter = pAdapter->Next) {
```
5. `cd SDL_net-1.2.8; ./configure --enable-shared --enable-static --host=i686-w64-mingw32; make; sudo make install`
6. `sudo rm /usr/local/cross-tools/i386-mingw32/lib/*.dll.a`. This eliminates the dynamic libraries to force static linking of SDL.
7. `cd chocolate-doom-2.2.1; ./configure --host=i686-w64-mingw32 --build=i386-linux LIBS="-lgdi32 -lwinmm -lwsock32"; make`
8. `i686-w64-mingw32-strip src/chocolate-doom.exe`

Voila, a statically built version of Chocolate DOOM which runs on everything from Windows 95 to Windows 10 (might need `msvcrt.dll` on old versions of Windows 95).

## The merged binary

`smash.py` merges the two binaries together. What it basically does is extend the size of the DOS header to make room for a PE header, appends the entire Windows binary to the DOS binary, then inserts the PE header into the slack space and rewrites the section offsets appropriately. On DOS, the `next_offset` in the header is ignored: DOS/4GW uses the executable size in the second and third WORDs of the DOS header to find the next binary to load. On Windows, the entire DOS header is ignored except for the `next_offset`, which points to the PE header and thus loads the Windows binary.

## Acknowledgements

Thanks [@angealbertini](https://twitter.com/angealbertini/status/1327138949408624642) ([angea](https://github.com/angea)) for this neat challenge!
