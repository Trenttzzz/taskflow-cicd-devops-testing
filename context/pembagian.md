Berikut pembagian tugas untuk 7 orang tanpa kode.

Orang	Role	Fokus Utama	Tanggung Jawab

- Orang 1	Application Quality Engineer	S1 — Bug Fix, Test, Coverage	Mencari dan memperbaiki 3 bug tersembunyi, menambahkan minimal 2 test case baru, memastikan semua unit test lulus, memastikan race test lulus, dan memastikan coverage minimal 75%.

- Orang 2	CI Pipeline Engineer	S2 — CI Pipeline Otomatis	Membuat pipeline CI sesuai tool kelompok, mengatur trigger untuk push dan pull request, menjalankan vet, unit test, race test, integration test dengan PostgreSQL, coverage gate, build binary, serta menyimpan coverage report sebagai artifact.

- Orang 3	Docker & Registry Engineer	S3 — Docker Image dan Container Registry	Mengurus Docker build, memastikan penggunaan multi-stage Dockerfile, membuat tag image berbasis commit SHA, push image ke registry, membuktikan image bisa di-pull, serta membuat perbandingan ukuran image multi-stage dan single-stage.

- Orang 4	Deployment & Smoke Test Engineer	S4 — Smoke Test Post-Deploy	Menjalankan aplikasi hasil deployment, membuat smoke test otomatis untuk endpoint /health dan /api/v1/stats, memastikan pipeline gagal jika smoke test gagal, serta menyiapkan bukti smoke test sukses dan gagal.

- Orang 5	Notification & Release Evidence Engineer	S4 — Notifikasi dan Bukti Release	Mengatur notifikasi pipeline sukses dan gagal melalui Slack, Telegram, atau email. Mengumpulkan bukti release seperti branch, commit SHA, waktu pipeline, link pipeline run, screenshot notifikasi, dan status pipeline.

- Orang 6	Rollback & Release Management Engineer	S5 — Rollback Strategy	Mengatur strategi rollback, memastikan tag stable hanya diperbarui setelah pipeline sukses, membuat prosedur rollback, menyiapkan demo rollback ke image sebelumnya, serta membuktikan aplikasi kembali normal setelah rollback.

- Orang 7	Security, Report & Presentation Lead	S6 + Laporan + Presentasi	Mengintegrasikan minimal 2 kategori security scan, misalnya SCA, SAST, secret scanning, atau container image scanning. Selain itu, bertanggung jawab menggabungkan laporan akhir, menyusun alur presentasi, dan mengoordinasikan demo live.


Pembagian ini mengikuti struktur skenario dan rubrik tugas: S1, S2, S3, dan S6 masing-masing bernilai besar, sedangkan S4 dan S5 tetap penting karena berpengaruh besar pada demo live dan bukti pipeline end-to-end.