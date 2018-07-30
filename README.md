# DOCKER SWARM

Docker memperkenalkan swarm mode yang memiliki kemampuan untuk men-deploy banyak container ke banyak host, menggunakan overlay network untuk service discovery dengan built-in load balancer untuk menskalakan layanan (services)

Docker swarm memperkenalkan 3 konsep baru, antara lain:
* Node : adalah bagian dari docker engine yang terhubung dengan Swarm. Node adalah manager atau worker, dimana manager menjadwalkan container akan berjalan dimana, worker akan mengeksekusi task. Secara default, maneger adalah worker.
* Services : adalah konsep yang berhubungan dengan sekumpulan tasks untuk dieksekusi oleh worker. Contohnya adalah service pada HTTP server yang berjalan sebagai docker container pada tiga node.
* Load Balancing: docker menyertakan load balancer untuk memproses request dari semua kontainer dalam service.

## Create Swarm Mode Cluster

Untuk menginisialisasi swarm mode digunakan perintah init

` docker swarm init `

setelah menjalankan command, docker engine akan tahu bagaimana bekerja dengan cluster dan menjadi manager. Hasil inisialisasi adalah token yang digunakan untuk menambahkan node dengan aman.

## Join Cluster

Dengan berjalannya swarm mode, maka dapat ditambahkan node tambahan dan menjalankan perintah dari semua node tersebut. Apabila node hilang, contohnya ketika terjadi crash, maka container yang berjalan dalam host-host tersebut akan otomatis me-rescheduled ke node lain. Proses rescheduling memastikan kita tidak kehilangan kapasitas dan memiliki availability yang tinggi.

pada tiap node yang ingin kita tambahkan ke cluster, gunakan docker CLI untuk menggabunggannya ke grup yang sudah ada. Penggabungan dilakukan dengan menunjukkan host lain ke manajer dalam cluster, dalam hal ini adalah host pertama.

Hal pertama yang dilakukan setelahnya adalah mendapatkan token yang dibutuhkan untuk menambahkan worker ke dalam cluster.

` token=$(docker -H 172.17.0.12:2345 swarm join-token -q worker) && echo $token `

pada host kedua, gabungkan ke cluster dengan me-request akses melalui manager. Token disediakan sebagai parameter tambahan

` docker swarm join 172.17.0.12:2377 --token $token `

Secara default, manager akan secara otomatis menerima nodes baru yang dimasukkan dalam cluster. Untuk melihat node pada cluster digunakan perintah `docker node ls`

## Membuat Overlay Network

Perintah berikut akan membuat overlay network dengan nama skynet. Semua container yang terdaftar pada network ini dapat berkomunikasi satu dengan yang lain, tidak memandang node mana mereka di-deploy

` docker network create -d overlay skynet `

## Deploy Service

Pada kasus ini, kita men-deploy docker image katacoda/docker-http-server. Kita mendefinisikan nama yang sederhana dari sebuah service yang disebut http dan di-attach ke network skynet yang baru dibuat.

Untuk memastikan replikasi dan availability, kita menjalankan dua instances, replika dan container dari cluster.

Kemudian kita melakukan load balance kepada dua container tersebut pada port 80. Mengirim HTTP request ke semua node pada cluster akan memproses request dari 1 container dalam cluster. Node dimana request diterima bisa tidak berada pada node dimana container merespons. Tetapi, request docker load-balances ditujukan pada semua container yang ada. Perintah yang digunakan adalah:

` docker service create --name http --network skynet --replicas 2 -p 80:80 katacoda/docker-http-server `

## Inspect State

Kita dapat melihat daftar dari semua task yang yang terhubung dengan service dari cluster. Pada hal ini, tiap task adalah container docker `service ps http`

Kita dapat melihat detail konfigurasi dari service via `docker service inspect --pretty http`

Pada tiap nodem kita dapat menanyakan task apa yang sedang berjalan. `docker node ps self`

Menggunakan ID dari node, kita dapat meng-query host secara individu `docker node ps $(docker node ls -q | head -n1)`

## Scale Service

Pada saat ini, kita memiliki 2 container yang sudah di-load-balance berjalan, yang memproses request kita `curl docker`

Perintah di bawah ini akan menskalakan http service kita untuk berjalan pada 5 container.

`docker service scale http=5`

Pada tiap host, kita akan melihat node tambahan dimulai
Load balancer akan secara otomatis diupdate. Request akan diproses ke container baru.
