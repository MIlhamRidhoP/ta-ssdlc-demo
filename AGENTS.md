# AGENTS.md

Aturan kerja untuk agen AI di repo ini. Berlaku untuk Antigravity, Claude Code, Cursor, dan
sejenisnya. Baca seluruhnya sebelum menyentuh apa pun.

## Apa ini

Repo **demo**. Isinya framework SSDLC berbasis DevOps dengan analisis keamanan berbantuan LLM,
dijalankan pada satu objek uji yang **sengaja berisi kerentanan tertanam**. Tujuannya satu:
memperagakan pipeline bekerja dari awal sampai hasilnya terbit, di repo yang bersih dari data
eksperimen.

Repo ini bukan tempat penelitian dilakukan. Angka Tugas Akhir hidup di repo TA
(`MIlhamRidhoP/TA-ssdlc-devops-akademik`), dan apa pun yang dijalankan di sini tidak boleh dipakai
sebagai data eksperimen. Lihat `README.md` untuk asal salinan dan hash tree `src/`.

**Aplikasi SIA di `src/` adalah objek uji, bukan produk.** Ia berisi SQL injection, IDOR, mass
assignment, dan kelemahan lain yang ditanam dengan sengaja. **Jangan pernah dideploy** dan jangan
dihubungkan ke data sungguhan.

## Batas keras

1. **Jangan `git push`.** Tidak pernah, kecuali diminta eksplisit di pesan itu juga.
2. **Jangan memicu workflow** (`workflow_dispatch`, `gh workflow run`, push yang memicu CI). Setiap
   run jalur v2 membakar uang panggilan LLM berbayar.
3. **Jangan memanggil API LLM berbayar.** Gemini, Anthropic, dan OpenAI dipanggil hanya oleh
   pipeline, tidak pernah oleh agen.
4. **Jangan menulis ulang riwayat git.** Tanpa `rebase`, `--force`, `reset --hard`, atau `--amend`,
   kecuali diminta eksplisit.
5. **Jangan menyentuh berkas ini:**
   - `src/**` — objek uji. **Jangan "diperbaiki".** Kerentanannya adalah isi tugasnya; menambalnya
     membuat repo ini tidak ada gunanya. Hash tree `src/` dicatat di `README.md` dan harus tetap
     sama dengan objek uji di repo TA.
   - `prompts/v2/**` dan `prompts/remediasi-v1/**` — dikunci HASHES.txt masing-masing. Aturannya
     dua arah: `sha256sum -c` harus OK, **dan** folder itu hanya boleh berisi berkas yang tercantum
     di HASHES.txt. `sha256sum -c` tidak pernah mengeluh soal berkas yang TIDAK tercantum, jadi
     menambah satu varian prompt baru di situ lolos 8/8 OK. `tests/kunci-prompt.test.js` menutup
     arah kedua itu; pengecualiannya hanya HASHES.txt sendiri dan README.md.
   - Skrip penilaian: `scripts/llm/score.js`, `scripts/llm-score.js`, dan
     `scripts/buat-acuan-skor.js` — angkanya dibandingkan dengan acuan di repo TA.
6. **Jangan membaca, membuat, atau mengubah GitHub Secrets**, dan jangan menaikkan izin token
   di workflow.
7. **Jangan mencetak isi token atau API key** ke log, terminal, maupun berkas.
8. **Jangan menambah dependensi produksi.** Perkakas uji tetap di luar repo atau devDependency
   yang sudah ada.
9. **Jangan menulis trailer atribusi** (`Co-Authored-By`, "Generated with …") di pesan commit.
10. **Jangan menaruh path lokal atau info mesin** di berkas yang di-commit.

## Yang tidak ada di sini

Repo ini salinan sebagian repo TA. Yang sengaja ditinggalkan, beserta akibatnya:

| Tidak ada | Akibat |
|---|---|
| `dashboard/` (kecuali dua berkas data di bawah) | Tidak ada antarmuka developer di repo ini. Dashboard di repo TA membaca hasil repo ini lewat `dashboard/projects.json` |
| `SEEDED.md` | Ground truth kerentanan tidak ada, jadi **penilaian tidak bisa dijalankan** dan **jalur v1 (`scripts/gemini-scan.js`) gagal**. Pakai jalur v2 |
| `hasil-eksperimen/`, `PRODUCT.md`, dan `docs/` selain dua skema di bawah | Dokumen dan data TA tidak ikut |

Dua berkas `dashboard/` tetap ada karena **pipeline membacanya**, bukan karena dashboardnya:

- `dashboard/models.json` — daftar model sah. Workflow memvalidasi input `vendor_v2` dan `model`
  terhadap berkas ini dan **gagal tanpa berkas itu**.
- `dashboard/rekomendasi-cwe.json` — panduan umum per CWE (lapis 1), dibaca
  `scripts/llm/remediasi.js`.

Dari `docs/` hanya dua skema `results.json` yang ikut, karena berkas tes dan
`scripts/validasi-skema.js` membacanya: `docs/design/results-v2.schema.json` (skema 2, dipakai
`tests/gabung.test.js`) dan `docs/design/results.schema.json` (skema 1, kini tanpa pemakai di repo
ini; disalin supaya skema kedua jalur tetap terdokumentasi berdampingan).

**Berkas tes yang subjeknya tidak ada di sini sudah dikeluarkan, bukan dilemahkan.** Delapan berkas
tes di repo TA memuat `dashboard/app.js` atau `dashboard/fixtures/**`, yang tidak ikut disalin;
berkas itu tidak dibawa ke repo demo supaya `npm run test:tanpa-server` di sini hijau dan
kegagalan yang tersisa hanya yang bermakna. Daftarnya ada di `README.md`. Tes itu tetap utuh dan
tetap dijalankan di repo TA; **jangan menulis ulang versi tiruannya di sini.**

## Cara memverifikasi

```bash
npm run test:tanpa-server                           # tes yang tidak butuh server
cd prompts/v2 && sha256sum -c HASHES.txt            # harus 8/8 OK
cd prompts/remediasi-v1 && sha256sum -c HASHES.txt  # harus 3/3 OK
```

`sha256sum -c` hanya separuh aturannya; separuh lainnya, yaitu tidak ada berkas di luar daftar,
diperiksa `tests/kunci-prompt.test.js`. Lihat batas keras nomor 5.

`scripts/validasi-skema.js` adalah modul (`module.exports = { validasiSkema }`), bukan CLI.
Menjalankannya langsung keluar 0 tanpa memeriksa apa pun; ia dipakai dari berkas tes.

**`npm run test:tanpa-server` harus hijau seluruhnya.** Kalau ada yang merah, itu cacat yang perlu
dikejar, bukan keadaan normal.

**Tes API memang gagal, dan itu disengaja.** Tes API menegaskan perilaku yang aman, sedangkan
`src/` sengaja tidak aman, jadi kegagalannya justru bukti objek uji ini berisi celah seperti
seharusnya. **Jangan diperbaiki, jangan dihapus, jangan dilemahkan.** Berkas yang dimaksud:
`tests/api.test.js`, `tests/cuti.test.js`, `tests/profil.test.js`, `tests/khs.test.js`, dan
`tests/list-query.test.js`.

Untuk menjalankan tes API dibutuhkan Postgres dan server:

```bash
cp .env.example .env                       # lalu isi sendiri nilainya
docker compose up -d db
npm run seed:admin && npm run seed:dummy
npm start &
npm test
```

**Matikan lagi apa pun yang dinyalakan untuk pengujian.** Container `db` memakai
`restart: unless-stopped`, jadi ia akan menyala lagi setelah reboot sampai dihentikan:

```bash
docker compose down
```

## Cara bekerja di sini

- **Bahasa Indonesia** untuk komentar kode, pesan commit, dan dokumentasi. Istilah teknis yang
  sudah baku (commit, branch, dispatch, false positive) dibiarkan apa adanya.
- **Komentar menjelaskan alasan, bukan isi baris.**
- **Pesan commit** memakai awalan `feat`/`fix`/`docs`/`test`/`style` dengan ruang lingkup, lalu
  badan yang menjelaskan sebab, bukan sekadar daftar perubahan.
- **Laporkan apa adanya.** Kalau tes gagal, katakan gagal beserta keluarannya. Kalau sebuah
  langkah dilewati, katakan dilewati. Jangan menyatakan sesuatu terverifikasi kalau belum
  dijalankan.
- **Perubahan framework dikerjakan di repo TA, bukan di sini.** Repo ini salinan; menambal di sini
  membuat keduanya menyimpang diam-diam. Kalau ada bug framework yang ketemu di sini, laporkan
  supaya diperbaiki di repo TA lalu disalin ulang.
- Perkakas sementara, skrip tambalan, dan screenshot **tidak masuk repo**.
- **Jangan memakai perintah cermin atau hapus massal di luar folder sementara milik sendiri.**
  Termasuk `robocopy /MIR`, `rm -rf`, `Remove-Item -Recurse -Force`, dan `git clean -xfd`. Perintah
  seperti ini menghapus menurut isi folder, bukan menurut apa yang Anda maksud, dan ia menembus
  salinan maupun tautan tanpa bertanya. Pernah terjadi di repo TA: `node_modules` terhapus sebagian
  oleh `robocopy /MIR` yang dipakai membersihkan worktree sementara, karena `ln -s` di Git Bash
  Windows diam-diam **menyalin** folder itu alih-alih menautkannya, sehingga cerminnya ikut
  menyasar isi aslinya. Akibatnya lima berkas tes gagal dengan `Cannot find module` dan sempat
  terbaca sebagai cacat kode, bukan sebagai lingkungan yang rusak. Hapus dengan menyebut sasaran
  yang spesifik, dan kalau folder yang dibersihkan mungkin berisi salinan atau tautan ke sesuatu
  yang masih dipakai, periksa dulu isinya sebelum menghapus.

  Di repo ini risikonya lebih besar dari sekadar `node_modules`: `src/` adalah objek uji yang
  terikat hash tree `ef5ff340dcb93e8153e52262bb79b64ea031e3cd`, dan satu cermin yang menyasar ke
  situ memutus pengikatnya dengan repo TA tanpa bunyi. Untuk tautan di Windows pakai junction
  (`New-Item -ItemType Junction`), bukan `ln -s`, dan lepaskan dengan `cmd /c rmdir` yang hanya
  menghapus tautannya, bukan isinya.

## Peta berkas

| Jalur | Isi |
|---|---|
| `src/` | Aplikasi SIA, objek uji berisi kerentanan tertanam. **Jangan diubah** |
| `.github/workflows/devsecops.yml` | Pipeline: rencana, build-test, sonarcloud, analisis (v1), analisis-v2 (matriks vendor), terbit-v2 (penerbit jalur v2), hasil (penerbit jalur v1) |
| `scripts/llm/` | Mesin analisis v2: deteksi, verifikasi, penyusun `results.json` skema 2, penggabung tiga vendor, penilaian |
| `prompts/v2/` | Prompt jalur v2: deteksi dan verifikasi. **Terkunci hash** |
| `prompts/remediasi-v1/` | Prompt rekomendasi lapis 2. **Terkunci hash** |
| `migrations/` | Skema Postgres, dijalankan otomatis oleh container `db` |
| `tests/` | `node --test`. Lihat "Cara memverifikasi" soal tes yang memang gagal |

## Kalau ragu

Tanya dulu. Keputusan yang menyangkut biaya panggilan LLM atau isi branch `security-results`
tidak boleh diambil sendiri oleh agen.
