# GEMINI.md — Handoff lengkap sesi malisx2 / PanVK

Dokumen ini ditulis supaya model lain yang melanjutkan sesi tidak mengulang eksperimen yang sudah gagal.
Baca juga `AGENTS.md` di folder yang sama untuk aturan singkat.

## 1. Perangkat & lingkungan

- **Device:** Infinix X6739, chipset `mt6893` (MediaTek Dimensity 1200)
- **GPU:** Mali-G77 MC9 (Mesa arch **v9**, frontend **job-manager / JM**, compiler Kraid)
- **Android:** 16, kernel `5.10.272-StargazerKernel-CelestialSphere`
- **Root:** KernelSU (`su` → uid 0, context `u:r:ksu:s0`)
- **Node GPU:** `/dev/mali0` (kbase). `/dev/dri/card0` = `mediatek-drm` (display), **bukan panfrost**
- **GPU governor:** sudah `performance` (`persist.sys.azenith.default_maligpu_gov`), suhu 41–48 °C → bukan throttling
- **Termux:** clang 21.1.8, rustc 1.99.1, meson 1.12.1 (pip), bindgen 0.73.2 (`pkg install rust-bindgen`), mako/PyYAML/packaging (pip), patchelf, glslangValidator, libclc, spirv-llvm-translator, spirv-tools
- **NDK:** tidak terpasang. Build dilakukan **native di device** dengan wrapper compiler `--target=aarch64-linux-android30`

## 2. Aplikasi target

Game: `io.github.belmantegu.dmc4recomp` (DMC4 recomp, Xbox360→Vulkan via RexGPU).
Memiliki UI **Driver Vulkan → IMPORTAR DRIVER (.ZIP)** format AdrenoTools (`meta.json` + `.so`) lewat `libmodvk.so` / `MODKIT_DRIVER_DIR` / `nativeSetenv`.
Data driver tersimpan di `/data/data/io.github.belmantegu.dmc4recomp/files/drivers/<driver_id>/`.
Preferensi: `/data/data/io.github.belmantegu.dmc4recomp/shared_prefs/modkit.xml`.

## 3. Repo `~/malisx2` — status & patch

Driver Vulkan untuk ARMSX2, target awal G615 v11, tapi jalan juga di v9/G77.

### Patch yang sudah dibuat (BELUM di-commit)
1. **`src/vk/mali_jm.h` + `src/vk/mali_jm_queue.c`** — `batch_frag` scratch disimpan di `struct mali_jm_queue` dan dipakai ulang. Sebelumnya `malloc`/`free` tiap `vkQueueSubmit`. Dampak kecil, risiko nol.
2. **`src/vk/mali_cmd_gfx.h`, `src/vk/mali_image.c`, `src/vk/mali_fb.c`** — `MALI_PRELOAD_EARLY_MIN_VERTICES` jadi global hidden yang bisa di-override:
   - env `MALISX2_PRELOAD_VERTS`
   - Android `debug.malisx2.preload_verts`
   - Default tetap `10000` → tidak ada perubahan perilaku.

`git status`: 5 file modified + zip release lama untracked.

### Build malisx2
```sh
cd ~/malisx2
export KRAID_ISA_XML_PATH=$PWD/mesa/src/panfrost/compiler/kraid/isa-v9-v14.xml
meson setup build-release --buildtype release -Db_ndebug=true -Dpoly=disabled
ninja -C build-release -j4
llvm-strip -o libvulkan_malisx2.so build-release/src/libvulkan_malisx2.so
```
- `-Dpoly=disabled` aman: GS/tess via libpoly hanya dipakai `PAN_ARCH >= 10`.
- **rustc 1.99** tidak lagi meneruskan `--cfg` ke proc-macro. Sudah ada patch lokal di `mesa/src/panfrost/compiler/kraid/proc/macros.rs` (checkout Mesa, gitignored) supaya membaca env `KRAID_ISA_XML_PATH`. Tanpa itu build gagal 708 error.
- Commit Mesa yang benar: `b5f8787e44083b4840f605b595e79fcf61077175` (branch `malisx2` dari `bmdhacks/armsx2-turnip`), checkout di `~/malisx2/mesa`.

### Measurement / tuning
```sh
su -c 'setprop debug.malisx2.measure "timing,dir=/sdcard/Android/data/io.github.belmantegu.dmc4recomp/files/malisx2,direct"'
su -c 'setprop debug.malisx2.preload_verts 10000'
```
Properti dibaca sekali saat library di-load → **wajib force-stop + buka ulang game** setelah ganti.
Saat ini: `measure=off`, `preload_verts=10000`.

### Data profil DMC4 dengan malisx2 (sudah diambil, jangan ulang)
- `cpuprof`: `waitfence` **950–977 ms/s** (2–3 call/s), `submit` ~0.1 ms/s, drawidx ~450–690 draw/frame ~1 ms/s → **GPU-bound**, CPU driver bukan masalah.
- `timing` CSV: 769 submit, 24289 row.
  - 1 frag pass per frame `1280x720 rt=2 draws=152` → **400–613 ms** (~442 juta GPU cycles)
  - pass `draws=53` → ~49 ms; pass kecil 1–2 draw → ~1 ms
  - dispatch ~1.9 ms/submit, vtc ~3.87 ms/submit
- `preload_verts=0` **tidak mengubah FPS** → jalur ZS-preload bukan pelakunya.
- Interpretasi: overdraw fragmen; `can_fpk` di `pan_compiler.c:314` butuh shader tanpa `discard`, tanpa tulis depth/stencil, tanpa `outputs_read` → shader DMC4 dengan alpha-test mematikan FPK. **panvk punya syarat FPK yang sama**, jadi fork panvk tidak menyelesaikan ini.
- FPS malisx2 di DMC4: **2 FPS**. Driver sistem Mali: blackscreen. Driver lain: 0 FPS.

Zip patch malisx2 untuk DMC4: `/sdcard/malisx2-g77-tune.zip`.

## 4. Repo `~/mali-panvk` — status & eksperimen

Fork dari **Liliya2727/mali-panvk** (parent `JimVulkan/mali-panvk`), HEAD `58f2c55`, remote push ke `github.com/Liliya2727/mali-panvk.git`.
Clone JimVulkan lama sudah dihapus; folder `mali-panvk-liliya` sudah direname jadi `mali-panvk`.

PanVK = Mesa 27.0.0-devel, Vulkan 1.4, bicara ke `/dev/mali0` (kbase), sudah di-test upstream di G77 (Galaxy S20). Prebuilt release: `/sdcard/mali-panvk-r10.zip`.

### Kronologi kegagalan (jangan diulang)
1. **Prebuilt r10** — intro video jalan, tapi di "Press Start" 0 FPS. `dmesg`: `Unhandled Page fault ... TRANSLATION_FAULT, READ`, lalu ratusan `JOB_READ_FAULT 0x42`, `panvk: the kbase submission did not complete` → device lost.
2. **Hipotesis GROW_ON_GPF** — PanVK pakai `ALLOC_ON_FAULT` untuk tiler heap 2×128 MB + poly heap, commit cuma 1 MB, mengandalkan GPU fault untuk commit sisanya. Patch: `grow = false` di `kbase_kmod_bo_alloc` → commit semua halaman (~330 MB). **Tidak menyelesaikan**; fault sama.
3. **Fix dlopen** (beberapa iterasi):
   - v1: `.so` butuh `libc++_shared.so` dengan RUNPATH ke Termux → app tidak bisa akses.
   - v2: ganti NEEDED ke `libc++.so` → error `_ZNSt6__ndk14cerrE`.
   - v3: kembalikan `libc++_shared.so`, RUNPATH dihapus → error `_ZNSt6__ndk113__hash_memoryEPKvm`.
   - v4: tambah implementasi lokal `__hash_memory` (`.build-wrappers/hash_shim.cpp/.o`, dimasukkan lewat `cpp_link_args`) → **driver akhirnya ke-load**, intro video jalan, masih fault di menu.
   - Semua 42 simbol `_ZNSt6__ndk1` sekarang tersedia di `libc++_shared.so` milik game.
4. **Fix JIT (v5)** — tambah `MEM_JIT_INIT` juga di JM (`group_id=0`, `trim_level=0`), meniru malisx2. Hasil: **black screen sejak awal tapi FPS counter jalan**; fault berubah jadi `TRANSLATION_FAULT EXECUTE` di VA `0x0000000000041B00` (alamat sangat rendah → shader/job chain ngaco).
5. **Hapus shader cache** (`/data/data/.../cache/mesa_shader_cache`, `.../files/.cache/mesa_shader_cache`) → **tidak ada perubahan**.
6. **Driver lain yang dicoba:** `Mesa-PanVK-26.3.0-devel-795ca7ff` (universal dari repo Liliya) → blackscreen total. `PanVK-rynx-26.3.1-bionic`, `PanVK_faux123_v1.1.7/1.1.8` tersedia tapi belum semua dicoba tuntas.
7. Upstream sudah commit terbaru (`58f2c55`), tidak ada fix lebih baru.

### Kesimpulan PanVK
**PanVK v9 JM tidak kompatibel dengan kbase MediaTek di kernel 5.10 ini.** Bukan masalah flag satu baris. Untuk melanjutkan perlu capture atom + command stream dan melacak BO mana yang tidak di-map — proyek terpisah.

**Rekomendasi:** kembali ke malisx2 untuk optimasi FPS. Patch PanVK yang ada saat ini **tidak menyelesaikan masalah**; jangan di-commit sebagai "fix". Putuskan: revert atau commit sebagai WIP dengan pesan jelas.

### File yang dimodifikasi di repo mali-panvk
```
 M src/compiler/clc/meson.build        SPIRV-Tools PR5534 check → warning (build workaround)
 M src/panfrost/lib/kmod/kbase_kmod.c  patch GROW_ON_GPF + MEM_JIT_INIT (eksperimen, gagal)
 M src/util/libsync.h                  fallback sync_merge/sync_wait API 24 (host tools)
?? .build-wrappers/                    cc26/cxx26 (target android30), hash_shim.cpp/.o
?? subprojects/.wraplock               lockfile meson
```
Build dirs: `build-host/`, `build-android/` (keduanya di-ignore).

### Build PanVK di Termux (native)
Host tools:
```sh
meson setup build-host -Dbuildtype=debugoptimized -Dmesa-clc=enabled -Dprecomp-compiler=enabled \
  -Dvulkan-drivers=panfrost -Dgallium-drivers= -Dplatforms= -Dllvm=enabled \
  -Dgles1=disabled -Dgles2=disabled -Degl=disabled -Dglx=disabled -Dgbm=disabled \
  -Dtools= -Dvideo-codecs= -Dallow-fallback-for=libdrm \
  --force-fallback-for=libdrm,expat,zlib -Dlibdrm:default_library=static \
  -Dc_link_args=-llog -Dcpp_link_args=-llog
ninja -C build-host -j4 src/compiler/clc/mesa_clc src/compiler/spirv/vtn_bindgen2 src/panfrost/clc/panfrost_compile
```
Driver (pakai wrapper `cc26`/`cxx26` = clang `--target=aarch64-linux-android30`):
```sh
PATH="$PWD/build-host/bin:$PWD/.build-wrappers:$PATH" CC=cc26 CXX=cxx26 meson setup build-android \
  -Dbuildtype=release -Db_ndebug=true -Dplatforms=android -Dplatform-sdk-version=31 \
  -Dandroid-stub=true -Dandroid-strict=false -Dandroid-libbacktrace=disabled \
  -Dvulkan-drivers=panfrost -Dgallium-drivers= -Dvulkan-layers= -Dtools= -Dvideo-codecs= \
  -Dxclipse-gralloc-deps= -Dmesa-clc=system -Dprecomp-compiler=system \
  -Dllvm=disabled -Dzstd=disabled -Dlmsensors=disabled -Dperfetto=false \
  -Dglx=disabled -Dgbm=disabled -Degl=disabled -Dgles1=disabled -Dgles2=disabled \
  -Dallow-fallback-for=libdrm,perfetto --force-fallback-for=expat,libdrm,zlib \
  -Dlibdrm:default_library=static -Dexpat:default_library=static -Dzlib:default_library=static
PATH="$PWD/build-host/bin:$PWD/.build-wrappers:$PATH" ninja -C build-android -j4 src/panfrost/vulkan/libvulkan_panfrost.so
llvm-strip -o libvulkan_panfrost.so build-android/src/panfrost/vulkan/libvulkan_panfrost.so
# cpp_link_args berisi hash_shim.o; patchelf --remove-rpath; NEEDED harus libc++_shared.so
```
Jangan pakai `-D__ANDROID_API__=31` (libc++ butuh `pthread_cond_clockwait` yang tidak ada di header Termux). Wrapper target android30 sudah cukup untuk `AHardwareBuffer_isSupported` (API 29) dan `pthread_cond_clockwait` (API 30).

Zip hasil: `/sdcard/mali-panvk-gpf-fix.zip` (versi v5, nama `PanVK Mali G77 kbase JIT fix v5`).

## 5. Langkah berikutnya yang disarankan

1. Kembali ke `~/malisx2`. Putuskan commit atau revert patch yang ada.
2. Uji render scale game 50% (`controls.xml` / `modkit.xml`, atau UI) — jika FPS naik ~3–4×, terkonfirmasi fill-rate bound.
3. Jika tetap ingin naikkan FPS lewat driver: port satu fast-path early-Z/FPK dari panvk ke malisx2, atau perbaiki policy preload warna. Jangan fork panvk penuh.
4. Jika tetap ingin PanVK: perlu capture atom/command stream saat fault dan identifikasi BO yang tidak di-map; ini baru bisa diputuskan setelah data itu ada.

## 6. Perintah berguna

```sh
# Set prop (harus lewat root, Termux biasa tidak bisa)
su -c '/system/bin/setprop debug.malisx2.measure off'
su -c '/system/bin/setprop debug.malisx2.preload_verts 10000'
/system/bin/getprop | grep -i malisx2

# Log
su -c 'logcat -d -s MESA:I | tail -30'
su -c 'dmesg 2>/dev/null | grep -iE "Unhandled Page fault|GPU fault" | tail -20'

# Paksa restart game
su -c 'am force-stop io.github.belmantegu.dmc4recomp'
```
