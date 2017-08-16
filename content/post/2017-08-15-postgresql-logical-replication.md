---
title: '[PostgreSQL] Logical Replication di PostgreSQL 10'
author: saifulmuhajir
type: post
date: 2017-08-15T07:49:34+07:00
url: /postgresql-logical-replication=/
categories:
  - PostgreSQL
tags:
  - postgres
  - PostgreSQL
  - replication

---
Kondisi saat ini, sampai PostgreSQL 9.x, untuk replikasi database dengan batasan tertentu alias tidak replikasi data secara keseluruhan pada cluster database dapat dilakukan dengan dua cara yaitu _dump-restore,_ dan replikasi dengan memanfaatkan _trigger._ Jika menggunakan trigger, maka pilihan alatnya adalah **[Londiste] (https://wiki.postgresql.org/wiki/Londiste_Tutorial_(Skytools_2))** dari Skytools atau **[Bucardo] (https://bucardo.org/wiki/Bucardo)** yang masing-masing memiliki nilai plus dan minus. Dengan memanfaatkan trigger ini replikasi cukup bisa diandalkan karena nyaris realtime. Sementara memanfaatkan dump-restore tentu saja tidak dapat dilakukan secara realtime akan tetapi masing-masing host memiliki sedikit ketergantungan. Intinya adalah, apa yang sesuai untuk kebutuhan saja.

Namun, berkat kerja keras developer Postgres, urusan replikasi dengan batasan tidak lagi menyusahkan berkat kehadiran **[LOGICAL REPLICATION] (https://www.postgresql.org/docs/10/static/logical-replication.html).**

Dengan **_logical replication,_** replikasi tidak lagi harus dilakukan secara menyeluruh pada cluster, melainkan dapat dibatasi pada tabel-tabel tertentu saja. Dan ini berjalan secara _realtime_ layaknya streaming replication. Dengan demikian satu _provider (master)_ dapat mengirimkan replikasi tabel-tabel pada banyak _subscriber (slave)_ atau sebaliknya. Selain itu dengan logical replication kita bisa melakukan replikasi pada level row atau kolom tertentu.

Logical replication memanfaatkan metode publish dan subscribe dengan cara kerja yaitu subscriber akan menarik data dari provider(-provider) yang di daftarnya. Dan jika diperlukan subscriber akan mem-publish data yang diterimanya untuk dapat diambil oleh subscriber lain di bawahnya (cascaded).

Dari penjelasan di atas, dapat ditarik kesimpulan bahwa _logical replication_ bisa berguna pada banyak kasus contohnya:

* upgrade versi Postgres tanpa downtime
* database analitik kumpulan berbagai cluster
* pemisahan database dari cluster tertentu
* dan lain sebagainya

