# AGENTS.md — Catatan untuk model/agent lain

Repositori ini adalah **fork `Liliya2727/mali-panvk`** (PanVK + kbase) yang dipakai untuk eksperimen di device **Infinix X6739 / Mali-G77 MC9 / Android 16 / kernel MediaTek 5.10.272**.

**Riwayat lengkap sesi, data profil, perintah build, dan semua eksperimen ada di [`GEMINI.md`](GEMINI.md). Baca file itu dulu sebelum mengubah apa pun.**

## Ringkasan 30 detik

- Tujuan awal: naikkan FPS game DMC4 recomp (`io.github.belmantegu.dmc4recomp`).
- Driver `malisx2` (repo terpisah di `~/malisx2`) **jalan**: 2 FPS di DMC4. PanVK **tidak jalan**: GPU fault di menu "Press Start".
- Sudah dicoba dan **gagal**: patch `GROW_ON_GPF`, patch `MEM_JIT_INIT` di JM, hapus shader cache, 4+ versi driver, perbaikan dlopen (`libc++_shared.so`, `__hash_memory`). Fault tetap ada (`TRANSLATION_FAULT`, `JOB_READ_FAULT 0x42`).
- Kesimpulan: **PanVK v9 JM tidak kompatibel dengan kbase Mediatek di kernel ini**. Jangan mengulang eksperimen yang sama.
- Rekomendasi: kembali ke `~/malisx2` untuk optimasi.

## Aturan

- Perubahan di repo ini bersifat **eksperimen**. Patch `kbase_kmod.c` saat ini **tidak memperbaiki masalah** — jangan di-commit sebagai "fix". Kalau mau disimpan, commit sebagai WIP dengan pesan yang jujur.
- `build-host/`, `build-android/`, `.build-wrappers/`, `subprojects/.wraplock` adalah artefak build Termux. Jangan ikut di-commit.
- `GEMINI.md` dan `AGENTS.md` adalah dokumentasi handoff — boleh di-update kalau ada temuan baru.
- Jangan menaruh file hasil dekompilasi atau binary Arm di sini.
- Saat mengganti driver di DMC4, selalu **hapus driver lama** dulu lalu import zip baru, karena `DriverStore` menyimpan status `driver_pending`.
- Setelah ganti properti `debug.malisx2.*`, **force-stop + buka ulang game** — nilainya dibaca sekali saat library di-load.

## Status git saat penulisan

```
 M src/compiler/clc/meson.build        build workaround (SPIRV-Tools)
 M src/panfrost/lib/kmod/kbase_kmod.c  eksperimen GPF + JIT (tidak berhasil)
 M src/util/libsync.h                  workaround API 24 untuk host tools
?? .build-wrappers/
?? subprojects/.wraplock
```

## Kontak antar-sesi

Kalau model lain melanjutkan: baca `GEMINI.md` → bagian **5. Langkah berikutnya yang disarankan**. Jangan mulai dari awal.
