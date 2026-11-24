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
- FPS malisx2 di DMC4: **2 FPS** (720p), **10 FPS** (180p). PanVK v7: **3–5 FPS** (180p, 0 GPU fault). Driver sistem Mali: blackscreen. Driver lain: 0 FPS.

Zip patch malisx2 untuk DMC4: `/sdcard/malisx2-g77-tune.zip`.

## 4. Repo `~/mali-panvk` — status & eksperimen

Fork dari **Liliya2727/mali-panvk** (parent `JimVulkan/mali-panvk`), remote push ke `github.com/Liliya2727/mali-panvk.git`.
Clone JimVulkan lama sudah dihapus; folder `mali-panvk-liliya` sudah direname jadi `mali-panvk`.

PanVK = Mesa 27.0.0-devel, Vulkan 1.4, bicara ke `/dev/mali0` (kbase), sudah di-test upstream di G77 (Galaxy S20). Prebuilt release: `/sdcard/mali-panvk-r10.zip`.

### Kronologi investigasi & penyelesaian
1. **Prebuilt r10** — intro video jalan, tapi di "Press Start" 0 FPS. `dmesg`: `Unhandled Page fault ... TRANSLATION_FAULT, READ`, lalu ratusan `JOB_READ_FAULT 0x42`, `panvk: the kbase submission did not complete`.
2. **Hipotesis GROW_ON_GPF** — `grow = false` di `kbase_kmod_bo_alloc` → commit semua halaman (~330 MB). Tidak menyelesaikan fault.
3. **Fix dlopen (v1–v4)** — tambah shim lokal `__hash_memory` (`.build-wrappers/hash_shim.cpp/.o`, dimasukkan lewat `cpp_link_args`), hapus RUNPATH, NEEDED `libc++_shared.so` → driver berhasil ke-load.
4. **Regresi JIT (v5)** — memanggil `MEM_JIT_INIT` *sebelum* `MEM_EXEC_INIT` membuat `MEM_EXEC_INIT` gagal, sehingga kbase menurunkan BO shader ke `SAME_VA` dan mengembalikan cookie `0x41000`. Karena `kbase_kmod_bo_alloc` tidak mengecek `req.out.flags & BASE_MEM_SAME_VA`, GPU mencoba mengeksekusi cookie `0x41B00` → black screen.
5. **Fix kbase + diagnostik fault (v6, `/sdcard/mali-panvk-g77-v6.zip`)**:
   - Benahi urutan `MEM_EXEC_INIT` sebelum `MEM_JIT_INIT` (dan `MEM_JIT_INIT` hanya untuk CSF) + cek `req.out.flags & BASE_MEM_SAME_VA` di `kbase_kmod_bo_alloc`.
   - Tambah `STICKY_RESOURCE_MAP`/`UNMAP` di `kbase_kmod_bo_import`/`bo_free`, hapus `SKIP_CACHE_START` (`0x8000`) dari `KBASE_JD_REQ_FRAGMENT` (`0x1u`), tambah `dsb st` sebelum `JOB_SUBMIT`.
   - Tambah `panvk_pool_invalidate_maps()` di `panvk_kbase_report_faults()` + log `atom_layout` (`V2_FRAME_NR` 64B MediaTek, UK 11.31).
   - Hasil capture logcat `MESA`: semua batch single-layer (batch 0–20) **sukses**, lalu fault pertama selalu terjadi di **batch 21 & 74** (render pass **6 layer / cubemap**): `job 6` (layer 0) DONE, **`job 7` (layer 1) langsung `TRANSLATION_FAULT READ (0xC2)` di slot 0 (`TERMINATED`)** pada offset identik `...17E80` → memicu cascade `JOB_READ_FAULT 0x42`.
6. **ROOT CAUSE & FIX (v7, `/sdcard/mali-panvk-g77-v7.zip`)**:
   - Di `src/panfrost/vulkan/jm/panvk_vX_cmd_buffer.c` (`cmd_prepare_tiler_context`), v9 JM membuat 1 `TILER_CONTEXT` per layer tapi lupa mengisi `batch->tiler.ctx.valhall.layer_offset = layer_idx;` (tetap 0).
   - Akibatnya `pan_fb.c` menghitung `cfg.internal_layer_index = info->layer - 0 = 1` untuk layer 1 padahal tiler context layer 1 hanya punya 1 layer (index 0) → GPU membaca polygon list keluar batas!
   - Setelah ditambah `batch->tiler.ctx.valhall.layer_offset = layer_idx;`: **0 GPU fault di `dmesg`**! Game tembus sampai menu Press Start dengan **3–5 FPS di 180p** (catatan: compile shader pertama di `bi_register_allocate` lambat hingga ~110 detik/shader untuk shader besar Xbox360, setelah masuk `mesa_shader_cache` turun jadi ~26 ms).

### Kesimpulan PanVK
**PanVK v9 JM sekarang BERJALAN BERSIH (0 GPU fault) di kbase MediaTek kernel 5.10 ini.**
Namun untuk performa DMC4, **`malisx2` (10 FPS @ 180p) masih 2–3× lebih cepat** daripada PanVK v7 (3–5 FPS @ 180p) karena `malisx2` menggunakan compiler **Kraid** (compile jauh lebih cepat dan register spilling lebih rendah dibanding `bi_register_allocate`).
Catatan opsional: `grow = true` (ALLOC_ON_FAULT) di `kbase_kmod.c` belum diuji ulang setelah fix `valhall.layer_offset` — bisa diuji lagi nanti untuk hemat ~330 MB RAM.

### File yang dimodifikasi di repo mali-panvk
Fix yang di-commit:
- `src/panfrost/vulkan/jm/panvk_vX_cmd_buffer.c` — fix `batch->tiler.ctx.valhall.layer_offset = layer_idx;` untuk multi-layer FBD v9 JM.
- `src/panfrost/lib/kmod/kbase_kmod.c` + `kbase_kmod.h` — urutan `MEM_EXEC_INIT`, fallback `SAME_VA` untuk `exec` BO, `STICKY_RESOURCE_MAP`/`UNMAP`, `KBASE_JD_REQ_FRAGMENT = 0x1u`, `dsb st`, `grow = false`.
- `src/panfrost/vulkan/jm/panvk_vX_gpu_queue.c` — `panvk_pool_invalidate_maps()` di `panvk_kbase_report_faults()`.

Workaround build Termux (tetap unstaged, jangan di-commit):
```
 M src/compiler/clc/meson.build        SPIRV-Tools PR5534 check → warning (build workaround)
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

Zip hasil: `/sdcard/mali-panvk-g77-v7.zip` (versi v7, nama `PanVK Mali G77 v9 layer fix v7`, 0 GPU fault).

## 5. Langkah berikutnya yang disarankan

1. Kembali ke `~/malisx2` untuk optimasi FPS DMC4 (sudah terbukti **10 FPS @ 180p**, 2–3× lebih cepat dari PanVK v7 karena compiler Kraid).
2. Di `~/malisx2`, profil 180p dengan `debug.malisx2.measure` untuk melihat pass mana yang masih memakan waktu di 180p (apakah pass beresolusi tetap seperti shadow/cubemap, atau overdraw/FPK).
3. Opsional untuk `~/mali-panvk`: uji ulang `grow = true` (`ALLOC_ON_FAULT`) di `kbase_kmod.c` sekarang setelah bug `valhall.layer_offset` selesai, dan pertimbangkan PR fix `panvk_vX_cmd_buffer.c` ke upstream `Liliya2727/mali-panvk`.

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
