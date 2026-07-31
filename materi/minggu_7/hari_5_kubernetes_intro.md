---
title: Hari 5 - Pengantar Kubernetes
parent: Minggu 7 - Containerization
nav_order: 6
---

# Hari 5 — Pengantar Kubernetes: Pod, Deployment, Service

*Jumat, 2 jam. Level konsep + kosakata dasar — hands-on penuhnya di `latihan_containerization_mini_project.md`.*

## Tujuan Belajar

- [ ] Menjelaskan kenapa orchestration dibutuhkan di luar apa yang Docker Compose sudah bisa lakukan
- [ ] Menjelaskan konsep Pod, Deployment, dan Service di Kubernetes
- [ ] Menjelaskan model deklaratif Kubernetes ("desired state") vs perintah imperatif (`docker run`)
- [ ] Menjalankan perintah `kubectl` dasar terhadap cluster lokal (minikube)

## Untuk Instruktur

Jangan mulai dari fitur Kubernetes yang canggih (auto-scaling, self-healing) sebagai "keajaiban" — mulai dari **masalah konkret** yang tidak bisa dijawab Compose: "kalau 1 container Postgres di laptop kamu crash tengah malam, siapa yang menyalakannya lagi? Kalau traffic naik 10x, siapa yang menambah instance-nya otomatis?" Compose tidak menjawab keduanya (Compose cuma soal 1 mesin, tidak ada mekanisme "recover otomatis" bawaan) — Kubernetes dirancang khusus untuk menjawab pertanyaan-pertanyaan ini.

## Konsep & Sintaks

### Kenapa Butuh Orchestration di Luar Compose

Docker Compose menjawab "jalankan beberapa container di **1 mesin**". Kubernetes menjawab pertanyaan yang lebih besar:

| Kebutuhan | Docker Compose | Kubernetes |
|---|---|---|
| Multi-container di 1 mesin | Ya | Ya (tapi biasanya overkill untuk ini saja) |
| Multi-**node** (banyak mesin/server) | Tidak didesain untuk ini | Ya — ini kekuatan utamanya |
| Self-healing (container crash → otomatis dinyalakan lagi) | Tidak otomatis (perlu `restart: always`, tapi terbatas di 1 mesin yang sama) | Ya, bawaan — kalau 1 node mati, Kubernetes bisa menjadwalkan ulang pod ke node lain |
| Scaling otomatis berdasarkan beban | Tidak ada bawaan | Ya (Horizontal Pod Autoscaler) |
| Load balancing antar banyak instance | Manual/tambahan | Bawaan lewat konsep Service |

Untuk skala mini project roadmap ini (1 laptop, beban kecil), Compose **lebih dari cukup** — Kubernetes baru benar-benar dibutuhkan saat skala aplikasi sudah butuh **banyak mesin** dan **uptime tinggi** yang tidak bisa dijamin manual. Minggu ini fokus paham konsepnya, bukan karena mini project butuh skala itu.

### Model Deklaratif: "Desired State", Bukan Perintah Langkah-demi-Langkah

Beda fundamental dari `docker run`/`docker compose up` (imperatif — "jalankan ini sekarang"): Kubernetes bekerja dengan model **deklaratif** — kamu mendeskripsikan **state yang diinginkan** ("saya mau selalu ada 3 replika aplikasi ini berjalan"), dan **Kubernetes terus-menerus bekerja** menjaga kondisi nyata tetap sesuai deskripsi itu, termasuk memperbaiki sendiri kalau ada yang menyimpang (misal 1 replika crash, otomatis dijadwalkan ulang tanpa campur tangan manual).

```
Imperatif (docker run)              Deklaratif (Kubernetes)
"Jalankan container X sekarang"      "Saya mau state: 3 replika X selalu berjalan"
    |                                     |
Kamu yang urus kalau ada yang mati   Kubernetes terus mengawasi & memperbaiki
```

### Pod

**Unit terkecil** yang bisa di-deploy di Kubernetes — **bukan** container itu sendiri (beda dari Docker Compose, di mana unitnya langsung container). 1 Pod biasanya membungkus **1 container** (kasus paling umum), tapi bisa berisi lebih dari 1 container yang **harus** selalu berjalan bersama & berbagi network/storage yang sama (kasus lanjutan, jarang untuk pemula).

```
Pod
┌─────────────────────────┐
│  Container: consumer.py  │
│  (image custom kamu)     │
└─────────────────────────┘
```

Untuk mini project minggu ini, 1 Pod = 1 container (`streaming-demo` consumer) — pola paling sederhana dan paling umum untuk mulai belajar.

### Deployment

Mendeskripsikan **desired state** untuk sekumpulan Pod yang identik: image apa yang dipakai, berapa **replika** yang harus selalu berjalan, dan strategi update-nya. Kubernetes terus mengawasi jumlah Pod yang benar-benar berjalan, dan **otomatis membuat Pod baru** kalau ada yang mati/crash — inilah **self-healing** yang dimaksud.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: consumer
spec:
  replicas: 1                 # <- desired state: harus selalu ada 1 Pod berjalan
  selector:
    matchLabels:
      app: consumer
  template:
    metadata:
      labels:
        app: consumer
    spec:
      containers:
        - name: consumer
          image: ecommerce-consumer:v1
```

### Service

Pod itu **fana** — bisa mati & dibuat ulang oleh Deployment kapan saja, dan IP-nya **berubah** tiap kali dibuat ulang. **Service** menyediakan 1 alamat **stabil** (nama + IP tetap) yang otomatis mengarah ke Pod manapun yang sedang hidup & cocok dengan label yang ditentukan — menyelesaikan masalah "bagaimana Pod lain (atau dunia luar) menemukan Pod ini kalau IP-nya berubah-ubah".

```yaml
apiVersion: v1
kind: Service
metadata:
  name: consumer-service
spec:
  selector:
    app: consumer            # <- mengarah ke Pod manapun berlabel app=consumer
  ports:
    - port: 80
      targetPort: 8000
```

**Analogi**: Deployment itu seperti manajer yang menjamin "selalu ada minimal 1 kasir buka" (mengatur pekerja/Pod), Service itu seperti **nomor antrian tetap** yang selalu mengarahkan kamu ke kasir manapun yang sedang bertugas saat itu — kamu tidak perlu tahu siapa kasirnya hari ini, cukup tahu "nomor antrian"-nya.

### `kubectl` Dasar

```bash
kubectl apply -f deployment.yaml     # terapkan/update desired state dari file manifest
kubectl get pods                     # lihat Pod yang sedang berjalan
kubectl get deployments
kubectl get services
kubectl logs <pod-name>              # lihat log dari 1 Pod
kubectl delete -f deployment.yaml    # hapus resource yang didefinisikan file itu
```

## Kesalahan Umum

1. **Mengira Pod dan container adalah hal yang sama.** Pod adalah **pembungkus** di sekitar 1 (atau lebih) container — perbedaan ini penting karena beberapa fitur Kubernetes (network, storage) beroperasi di level Pod, bukan langsung di level container.
2. **Mengira `kubectl delete pod <nama>` akan "mematikan" aplikasinya secara permanen.** Kalau Pod itu dikelola oleh Deployment, menghapusnya cuma memicu Kubernetes **membuat Pod baru** menggantikannya (sesuai desired state `replicas`) — ini justru bukti self-healing bekerja, bukan cara yang tepat untuk benar-benar menghentikan aplikasi (untuk itu, hapus Deployment-nya, bukan Pod-nya).
3. **Menganggap Kubernetes selalu "lebih baik" dari Compose, jadi harus dipakai di semua kasus.** Untuk aplikasi kecil/single-machine, Kubernetes menambah kompleksitas operasional signifikan (belajar YAML tambahan, manajemen cluster) tanpa manfaat nyata — Compose tetap pilihan yang tepat untuk skala itu. Kubernetes "terbayar" pada skala yang benar-benar butuh multi-node & high availability.
4. **Bingung kenapa Pod di minikube tidak bisa langsung diakses dari `localhost` browser seperti container Compose.** Karena minikube menjalankan cluster di dalam **VM/container terisolasi**-nya sendiri — perlu `kubectl port-forward` atau `minikube service` untuk menjembatani akses dari mesin host, beda dari `ports:` di Compose yang langsung publish ke host.

## Latihan

1. Jelaskan ke rekan kerja: kenapa Kubernetes dianggap "overkill" untuk mini project roadmap ini (1 laptop, 1 komponen), tapi tetap penting dipelajari konsepnya untuk karier data engineer jangka panjang.
2. Deployment dengan `replicas: 3` — salah satu dari 3 Pod-nya crash. Apa yang akan dilakukan Kubernetes, dan berapa lama kira-kira sampai jumlah Pod kembali ke 3 (secara konsep, tidak perlu angka pasti)?
3. Jelaskan kenapa Service dibutuhkan **di atas** Deployment — kenapa tidak cukup mengandalkan IP Pod secara langsung untuk saling terhubung antar komponen?
4. Bandingkan model imperatif (`docker run`) vs deklaratif (`kubectl apply`) dari sudut pandang: apa yang terjadi kalau container/Pod tiba-tiba mati secara tidak sengaja di tengah malam saat tidak ada orang yang mengawasi?

## Kunci Jawaban & Pembahasan

**1.** Kubernetes "overkill" untuk mini project ini karena skalanya sangat kecil (1 komponen, 1 laptop, tidak butuh multi-node atau high availability sungguhan) — menambah kompleksitas operasional (belajar manifest YAML, `kubectl`, konsep cluster) yang manfaatnya tidak terasa di skala ini, Compose sudah lebih dari cukup. Tapi penting dipelajari konsepnya karena Kubernetes adalah **standar industri** untuk deployment production skala menengah-besar — banyak perusahaan data engineering menjalankan Airflow, Spark, dan layanan lain di atas Kubernetes (atau layanan terkelola berbasis Kubernetes seperti GKE/EKS) justru karena kebutuhan self-healing & scaling yang tidak terasa perlu di laptop pribadi, tapi krusial saat sistem melayani banyak pengguna nyata.

**2.** Kubernetes akan **otomatis membuat 1 Pod baru** untuk menggantikan yang crash — ini konsekuensi langsung dari model deklaratif: Kubernetes terus membandingkan state nyata (2 Pod hidup) dengan desired state (`replicas: 3`), begitu ada selisih, langsung dijadwalkan Pod baru tanpa perlu perintah manual. Waktunya biasanya **cepat** (detik hingga sekitar semenit), tergantung waktu image di-pull (kalau belum ada di node itu) dan waktu startup aplikasi di dalam container — tidak butuh menunggu manusia menyadari & menjalankan perintah recovery manual, beda total dari skenario container Compose yang mati tengah malam tanpa siapapun mengawasi.

**3.** Kalau langsung mengandalkan IP Pod, komponen lain harus terus-menerus tahu IP **terbaru** tiap kali Pod diganti/dijadwalkan ulang (yang seperti dibahas di atas, terjadi otomatis & bisa kapan saja) — ini rapuh dan butuh mekanisme "penemuan IP baru" tersendiri di level aplikasi. Service menghilangkan masalah ini dengan menyediakan 1 alamat **stabil** yang tidak pernah berubah, terlepas dari berapa kali Pod di baliknya diganti — konsumen Service tidak perlu tahu atau peduli IP Pod yang sesungguhnya sedang melayani permintaan saat itu.

**4.** Model **imperatif** (`docker run`, tanpa `restart: always`/orchestrator): container yang mati **tetap mati** sampai ada manusia (atau script eksternal) yang secara sadar menjalankan perintah untuk menyalakannya lagi — tidak ada mekanisme bawaan yang "mengawasi & memperbaiki sendiri". Model **deklaratif** (Kubernetes, lewat Deployment): sistem terus-menerus membandingkan state nyata vs desired state secara otomatis, tanpa butuh manusia menyadari insidennya sama sekali — Pod baru akan dijadwalkan menggantikan yang mati **sebelum** manusia manapun bahkan sempat melihat notifikasi kalau ada. Ini beda filosofis yang mendasar: imperatif butuh manusia sebagai "pengawas aktif", deklaratif memindahkan peran pengawasan itu ke sistem itu sendiri.
