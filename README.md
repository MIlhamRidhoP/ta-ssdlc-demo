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
- **`hasil-eksperimen/`, `docs/`, `PRODUCT.md`** — data dan dokumen TA.

Akibatnya sebagian berkas tes gagal memuat modul (`dashboard/app.js`,
`dashboard/fixtures/**`, `docs/design/results*.schema.json`). Tes itu **tidak diubah dan tidak
dihapus**: ia tetap sahih di repo TA, dan di sini kegagalannya adalah soal lingkup salinan, bukan
cacat. Lihat `AGENTS.md`.

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
npm run test:tanpa-server
```

Tes API butuh Postgres dan server yang hidup:

```bash
cp .env.example .env                       # lalu isi nilainya sendiri
docker compose up -d db
npm run seed:admin && npm run seed:dummy
npm start &
npm test
docker compose down                        # jangan ditinggalkan hidup
```

**Ada tes yang memang gagal, dan itu bukti framework ini ada gunanya.** Objek uji sengaja tidak
aman, sedangkan tes API menegaskan perilaku yang aman. Jangan diperbaiki, jangan dilemahkan,
jangan dihapus. Sebab kegagalan yang kedua — berkas dashboard dan skema yang tidak ikut disalin —
dijelaskan di bagian "Yang tidak ikut disalin".

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
