# AGENTS.md — Catatan untuk model/agent lain

Repositori ini adalah **fork `Liliya2727/mali-panvk`** (PanVK + kbase) yang dipakai untuk eksperimen di device **Infinix X6739 / Mali-G77 MC9 / Android 16 / kernel MediaTek 5.10.272**.

**Riwayat lengkap sesi, data profil, perintah build, dan semua eksperimen ada di [`GEMINI.md`](GEMINI.md). Baca file itu dulu sebelum mengubah apa pun.**

## Ringkasan 30 detik

- Tujuan awal: naikkan FPS game DMC4 recomp (`io.github.belmantegu.dmc4recomp`).
- **PanVK v9 JM SEKARANG JALAN TANPA GPU FAULT** di kbase MediaTek kernel 5.10 ini!
  - Biang keladi `TRANSLATION_FAULT READ` + cascade `JOB_READ_FAULT 0x42` di menu "Press Start" ternyata **bug upstream PanVK v9 JM** di `src/panfrost/vulkan/jm/panvk_vX_cmd_buffer.c`: `cmd_prepare_tiler_context()` lupa mengisi `batch->tiler.ctx.valhall.layer_offset = layer_idx;`, sehingga render pass multi-layer (cubemap 6 layer) membaca polygon list keluar batas pada layer ≥ 1.
  - Perbaikan tambahan di `kbase_kmod.c`/`.h` + `panvk_vX_gpu_queue.c`: urutan `MEM_EXEC_INIT` sebelum `MEM_JIT_INIT`, cek `BASE_MEM_SAME_VA` saat `exec` BO di-downgrade, `STICKY_RESOURCE_MAP`/`UNMAP` untuk dma-buf import, hapus `SKIP_CACHE_START` (`0x8000`) dari `KBASE_JD_REQ_FRAGMENT`, tambah `dsb st` sebelum `JOB_SUBMIT`, dan `panvk_pool_invalidate_maps()` di `panvk_kbase_report_faults()`.
- **Perbandingan FPS di DMC4 (180p, menu Press Start):**
  - **PanVK v7:** 3–5 FPS (0 GPU fault di `dmesg`, tapi compile shader awal di `bi_register_allocate` lambat hingga ~110 detik/shader; setelah masuk `mesa_shader_cache` cuma ~26 ms).
  - **`malisx2` (repo terpisah di `~/malisx2`):** **10 FPS** di 180p (720p: 2 FPS) berkat compiler **Kraid**.
- Rekomendasi: untuk optimasi FPS DMC4 tetap lanjut di `~/malisx2`; patch PanVK v9 JM di repo ini sudah bersih dan layak di-commit / di-PR ke upstream.

## Aturan

- Fix PanVK v9 JM (`panvk_vX_cmd_buffer.c`, `kbase_kmod.c`, `kbase_kmod.h`, `panvk_vX_gpu_queue.c`) sudah teruji 0 GPU fault di Mali-G77 MC9 / MediaTek kernel 5.10.
- `src/compiler/clc/meson.build`, `src/util/libsync.h`, `build-host/`, `build-android/`, `.build-wrappers/`, `subprojects/.wraplock` adalah artefak/workaround build Termux. Jangan ikut di-commit.
- `GEMINI.md` dan `AGENTS.md` adalah dokumentasi handoff — boleh di-update kalau ada temuan baru.
- Jangan menaruh file hasil dekompilasi atau binary Arm di sini.
- Saat mengganti driver di DMC4, selalu **hapus driver lama** dulu lalu import zip baru, karena `DriverStore` menyimpan status `driver_pending`.
- Setelah ganti properti `debug.malisx2.*`, **force-stop + buka ulang game** — nilainya dibaca sekali saat library di-load.

## Status git saat penulisan

Fix PanVK v9 JM + update dokumentasi sudah di-commit terpisah dari workaround build Termux:
```
 M src/compiler/clc/meson.build        build workaround Termux (SPIRV-Tools)
 M src/util/libsync.h                  workaround API 24 untuk host tools
?? .build-wrappers/
?? subprojects/.wraplock
```

## Kontak antar-sesi

Kalau model lain melanjutkan: baca `GEMINI.md` → bagian **5. Langkah berikutnya yang disarankan**. Jangan mulai dari awal.
