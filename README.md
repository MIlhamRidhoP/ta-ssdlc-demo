# ta-ssdlc-demo

Repo demo untuk framework SSDLC berbasis DevOps dengan analisis keamanan berbantuan LLM.

> [!CAUTION]
> **Kode di `src/` sengaja berisi kerentanan. Jangan dideploy.**
>
> Aplikasi SIA (sistem informasi akademik) di `src/` adalah **objek uji**, bukan produk. Ia
> mengandung SQL injection, IDOR, mass assignment, kelemahan otorisasi, dan celah lain yang
> ditanam dengan sengaja sebagai bahan analisis. Jangan dijalankan di jaringan yang bisa diakses
> publik, jangan dihubungkan ke data sungguhan, dan jangan dipakai sebagai contoh cara menulis
> kode. Container dan server di repo ini hanya untuk dijalankan secara lokal lalu dimatikan lagi.

## Tujuan

Memperagakan framework bekerja utuh di repo yang bersih: pipeline GitHub Actions menjalankan build
dan uji, SAST SonarCloud, lalu analisis LLM (deteksi atas seluruh `src/`, kemudian verifikasi per
alert SonarCloud), dan menerbitkan hasilnya sebagai `results.json` skema 2 ke branch
`security-results`.

Repo ini **bukan** tempat penelitian dilakukan. Angka dan data eksperimen Tugas Akhir hidup di repo
TA, dan apa pun yang dijalankan di sini tidak boleh dipakai sebagai data eksperimen.

## Asal salinan

| Bagian | Asal |
|---|---|
| Framework (`.github/workflows/`, `scripts/`, `prompts/`, `tests/`, `migrations/`, berkas build) | Repo TA `MIlhamRidhoP/TA-ssdlc-devops-akademik`, branch `feat/prompt-v2`, commit `a52cfd5110062e4f76ee693fa5e03da246adb520` |
| Objek uji (`src/`) | Repo TA yang sama, branch `eksperimen/seeded` |

`src/` disalin byte-identik. Hash tree git-nya:

```
ef5ff340dcb93e8153e52262bb79b64ea031e3cd
```

Angka itu sama dengan hash tree `src/` pada branch `eksperimen/seeded` di repo TA, dan bisa
diperiksa ulang kapan saja:

```bash
# di repo ini
git rev-parse HEAD:src

# di repo TA
git ls-tree eksperimen/seeded src
```

Pipeline memakai hash tree `src/` — bukan nama branch dan bukan kesamaan commit — untuk memastikan
laporan SonarCloud yang dipakai benar-benar berasal dari kode yang sedang dianalisis. Karena itu
angka di atas adalah pengikat kedua repo, dan harus tetap sama.

## Yang tidak ikut disalin

Repo ini salinan sebagian. Yang sengaja ditinggalkan, beserta akibatnya:

- **`dashboard/`** — Security Dashboard tetap tinggal di repo TA dan membaca hasil repo ini dari
  jarak jauh. Dua berkas datanya ikut karena **pipeline membacanya**, bukan karena dashboardnya:
  `dashboard/models.json` (daftar model sah; workflow memvalidasi input terhadapnya dan gagal tanpa
  berkas itu) dan `dashboard/rekomendasi-cwe.json` (panduan umum per CWE, lapis 1).
- **`SEEDED.md`** — ground truth kerentanan tidak ikut, jadi penilaian otomatis tidak bisa
  dijalankan di sini dan jalur v1 (`scripts/gemini-scan.js`) akan gagal. Pakai jalur v2.
- **`hasil-eksperimen/`, `PRODUCT.md`** — data dan dokumen TA.
- **`docs/`** kecuali dua skema `results.json` (`docs/design/results-v2.schema.json` dan
  `docs/design/results.schema.json`), yang ikut karena berkas tes membacanya.

## Tes dashboard ada di repo TA

Delapan berkas tes dari repo TA **tidak ikut disalin**, karena subjeknya tidak ada di sini: mereka
memuat `dashboard/app.js` atau `dashboard/fixtures/**`, sedangkan dashboard tetap tinggal di repo
TA. Tanpa berkasnya, tes itu hanya bisa gagal dengan `MODULE_NOT_FOUND` — kegagalan yang tidak
mengatakan apa pun tentang kode di repo ini.

| Berkas tes (ada di repo TA) | Subjeknya |
|---|---|
| `tests/tautan-github.test.js` | pembangun URL GitHub di `dashboard/app.js` |
| `tests/urutan-run.test.js` | pengurut riwayat run di `dashboard/app.js` |
| `tests/pipeline.test.js` | konstanta `PIPELINE` di `dashboard/app.js` versus workflow |
| `tests/results-v2.test.js` | fixture `dashboard/fixtures/v2/**` terhadap skema |
| `tests/remediasi.test.js` | rekomendasi lapis 2 atas fixture dashboard |
| `tests/publikasi.test.js` | penerbitan index, atas fixture dashboard |
| `tests/skenario.test.js` | sumbu skenario, atas fixture dashboard |
| `tests/rekomendasi-cwe.test.js` | pemetaan CWE terhadap fixture dashboard dan `dashboard/app.js` |

Berkas itu **tidak diubah, tidak dilemahkan, dan tidak ditulis ulang di sini** — hanya tidak
dibawa. Semuanya tetap utuh dan tetap dijalankan di repo TA, tempat dashboard dan fixture-nya
berada. Jalankan di sana untuk menguji dashboard.

Khusus `tests/rekomendasi-cwe.test.js`: subjek utamanya, `dashboard/rekomendasi-cwe.json`,
sebenarnya **ada** di repo ini karena pipeline membacanya. Enam dari delapan pemeriksaannya lulus
di sini; dua sisanya butuh fixture dashboard. Berkasnya tetap tidak dibawa supaya tidak ada tes
merah yang tidak bermakna, jadi validasi berkas pemetaan itu hidup di repo TA.

## Tes API sengaja gagal

Lima berkas tes API **tetap ada dan memang gagal**, dan itulah gunanya repo ini:

`tests/api.test.js`, `tests/cuti.test.js`, `tests/profil.test.js`, `tests/khs.test.js`,
`tests/list-query.test.js`

Tes itu menegaskan perilaku yang aman — otorisasi ditegakkan, input diparameterisasi, field
sensitif tidak bocor — sedangkan `src/` sengaja tidak aman. Kegagalannya adalah **buktinya objek
uji ini benar-benar berisi celah**, bukan cacat yang perlu ditambal. **Jangan diperbaiki, jangan
dilemahkan, jangan dihapus.**

Contohnya: `src/models/mahasiswaModel.js` menginterpolasi pola pencarian langsung ke SQL
(`WHERE ('${pola}' IS NULL OR ...)`) alih-alih memparameterisasinya. Akibat sampingannya, `GET
/api/mahasiswa` tanpa `?q=` menghasilkan literal `'null'`, dan `'null' IS NULL` bernilai false di
Postgres, sehingga list admin mengembalikan nol baris. Itu menjatuhkan `before`-hook di
`api`, `cuti`, dan `profil` sekaligus.

## Penyesuaian terhadap repo TA

Hanya identitas repo dan nama container yang berbeda; perilaku pipeline tidak diubah.

| Berkas | Perubahan |
|---|---|
| `sonar-project.properties` | `sonar.projectKey` menjadi `MIlhamRidhoP_ta-ssdlc-demo` |
| `.github/workflows/devsecops.yml` | `SONAR_PROJECT` menjadi `MIlhamRidhoP_ta-ssdlc-demo` |
| `scripts/gemini-scan.js` | Nilai bawaan `SONAR_PROJECT` menjadi proyek demo; bawaan `SONAR_BRANCH` menjadi `main`, karena branch `eksperimen/seeded` tidak ada di sini |
| `package.json` | `name`, `description`, `repository`, `bugs`, `homepage` menunjuk repo ini |
| `docker-compose.yml` | `container_name` dan nama network diberi awalan `ta_demo_`, supaya tidak bertabrakan dengan container repo TA di mesin yang sama |
| `AGENTS.md` | Ditulis ulang untuk repo demo |
| `README.md` | Berkas ini |

## Tes

```bash
npm ci
npm run test:tanpa-server      # harus hijau seluruhnya
```

Tes API butuh Postgres dan server yang hidup, dan **memang gagal** — lihat "Tes API sengaja
gagal" di atas:

```bash
cp .env.example .env                       # lalu isi nilainya sendiri
docker compose up -d db
npm run seed:admin && npm run seed:dummy
npm start &
npm test
docker compose down                        # jangan ditinggalkan hidup
```

Dua hasil itu dibaca berbeda: `test:tanpa-server` merah berarti ada yang rusak, sedangkan tes API
merah adalah keadaan yang diharapkan.

## Langkah setup yang menjadi tanggung jawab pemilik repo

Belum dikerjakan di repo ini dan harus dilakukan sendiri:

1. **GitHub Secrets** pada repo ini, dengan nama yang sama seperti di repo TA:
   - `SONAR_TOKEN`
   - `GEMINI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` (sesuai vendor yang akan dipakai)
2. **Proyek SonarCloud** `MIlhamRidhoP_ta-ssdlc-demo` di organization `milhamridhop`. Paket Free
   hanya menganalisis branch `main`, jadi biarkan analisis berjalan di `main` atau pilih input
   `sonar_branch_v2=main` saat dispatch.
3. **Push** isi repo ini ke `MIlhamRidhoP/ta-ssdlc-demo`.
4. **Branch `security-results`** tidak perlu dibuat manual. Langkah penerbitan membuatnya sendiri
   sebagai branch orphan kalau belum ada, memakai izin `contents: write` yang sudah dimiliki job
   penerbit. Tidak ada izin token yang ditambahkan untuk ini.
5. **Dispatch pipeline** lewat tab Actions. Jalur v2 memanggil API berbayar, jadi input
   `konfirmasi_v2` harus diisi `JALANKAN`; tanpa itu run ditolak sebelum ada panggilan apa pun.
6. **Dashboard di repo TA** sudah didaftarkan untuk membaca repo ini lewat
   `dashboard/projects.json`, dan tetap jalan selama branch `security-results` belum ada atau
   belum punya run.
