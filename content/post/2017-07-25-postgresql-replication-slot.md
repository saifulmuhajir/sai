---
title: '[PostgreSQL] Replication Slot'
author: saifulmuhajir
type: post
date: 2017-07-25T03:47:34+00:00
url: /postgresql-replication-slot/
categories:
  - PostgreSQL
tags:
  - postgres
  - PostgreSQL
  - replication

---
Sejak **PostgreSQL 9.4**, ada parameter baru di Postgres yaitu **max\_replication\_slot**. Apa sih ini?

**Replication slot** adalah mekanisme dalam Postgres yang secara otomatis akan memastikan bahwa WAL files dari host master dijaga keberadaannya sampai semua standby server menerimanya. Dengan kata lain, slot replikasi akan memastikan bahwa **WAL files yang ada di host master TIDAK dihapus** sampai semua server standby sudah mengambilnya. Slot replikasi ini sebenarnya bisa digantikan dengan parameter _wal\_keep\_segments_ dan _archive\_command_, hanya saja cara kerja keduanya berbeda karena slot memastikan kebutuhan WAL files terpenuhi sedangkan _wal\_keep\_segments_ menggunakan parameter jumlah file. Adapun _archive\_command_ memberikan beban kerja kepada server untuk memprosesnya.

Kelebihan lain slot replikasi adalah host master memastikan bahwa **penghapusan baris/record tidak mengakibatkan konflik recovery** dengan server standby, bahkan jika server standby dalam kondisi terputus untuk beberapa saat. Parameter _hot\_standby\_feedback_ sebenarnya bisa menghalau konflik recovery, hanya saja ia tidak menjamin record yang terhapus pada host master dijaga karena standby putus koneksi. Parameter lain yang mungkin digunakan sebelum adanya slot replikasi adalah _vacuum\_defer\_cleanup\_age_ yang harus diset ke nilai cukup tinggi untuk menjamin standby terhindar dari konflik. Slot replikasi menghapuskan kedua masalah pada parameter sebelumnya.

Adapun karakter pada slot replikasi adalah berikut:

  * parameter yang digunakan adalah _max\_replication\_slot_
  * hanya bisa diset saat server startup
  * default nilainya 0
  * _WAL\_level_ harus pada status replica atau di atasnya
  * slot replikasi harus diberi nama yang bisa terdiri dari _lowercase, underscore, dan/atau angka_
  * slot replikasi yang telah dibuat dapat dilihat pada view _pg\_replication\_slots_
  * contoh perintah pembuatan slot physical: 
`SELECT * FROM pg_create_physical_replication_slot('standby_no_1');`
  * contoh perintah pembuatan slot logical: 
`SELECT * FROM pg_create_logical_replication_slot('standby_no_1');`
  * contoh slot pada standby (recovery.conf): 
`primary_slot_name = 'standby_no_1'`
  * contoh perintah penghapusan slot: 
`SELECT * FROM pg_drop_replication_slot('standby_no_1');`
