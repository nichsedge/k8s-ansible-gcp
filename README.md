# Kubernetes on Bare Metal: GCP VM Instances

## Skema Infrastruktur TI

![infrastruktur](https://user-images.githubusercontent.com/55334589/173588977-b8a1cdb8-0738-4d0d-95c6-afcb56acdee8.png)

Pada infrastruktur tersebut akan dibuat kubernetes cluster dengan berbagai layanan di dalamnya. Layanan tersebut terdiri dari content management system, web profile, file sharing, aplikasi meeting, dan monitoring. Layanan content management system menggunakan container dari **wordpress**, layanan web profile menggunakan container **spring-petclinic**, layanan file sharing menggunakan container dari **owncloud**, layanan aplikasi meeting menggunakan container dari **jitsi**, dan layanan monitoring menggunakan **kubernetes dashboard** dari kubernetes serta layanan monitoring VM instances bawaan dari Google Cloud Platform. Selain, itu, pembuatan kubernetes cluster tersebut dilakukan melalui automasi menggunakan **Ansible** serta akan dilakukan testing menggunakan **jmeter**.

## Requirements

Diperlukan sebanyak 8 instance dengan spesifikasi seperti di bawah ini. Pastikan OS yang digunakan adalah **Ubuntu 20.04** dan storage yang diperlukan dari masing-masing instance minimal adalah 10 GB.

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled.png)

## Ansible

Instalasi Ansible dapat dilakukan pada PC lokal atau menggunakan instance VM di cloud sebagai control node. Tutorial ini menggunakan WSL pada PC lokal untuk memudahkan konfigurasi. Referensi instalasi lebih lanjut dapat dilihat melalui dokumentasi Ansible atau melalui sumber berikut [https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-20-04#step-1-%E2%80%94-installing-ansible). Berikut merupakan tahap yang dilakukan.

1. Install Ansible pada WSL.

```bash
sudo apt update
sudo apt install ansible
```

Pastikan ansible sudah terinstall

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%201.png)

2. Menghubungkan control node dengan managed node.

Setelah ansible telah diinstall, ansible perlu melakukan komunikasi dengan setiap node yang ada. Dengan demikian, pastikan anda dapat SSH ke setiap VM instance yang telah dibuat. Untuk dapat SSH ke setiap VM instance, buatlah ssh key terlebih dahulu pada PC lokal. 

- Input:

```bash
# Generate key pada control node. Ikuti langkah pembuatannya
ssh-keygen -t rsa -f ~/.ssh/<FILE_NAME> -C <USER> -b 2048
# Lihat isi key yang dibuat kemudian copy isi file tersebut.
cat ~/.ssh/<FILE_NAME>.pub
```

Ubah `<FILE_NAME>` dan `<USER>` sesuai kebutuhan.

- Output

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%202.png)

Setelah itu, tambahkan isi dari output tersebut ke `SSH Keys` pada GCP Project agar PC lokal dapat SSH ke setiap VM instance pada project. 

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%203.png)

3. Clone repository ini pada PC lokal dan lakukan penyesuaian pada file `hosts`. Ubah IP pada tiap baris sesuai dengan `External IP` dari VM Instance dan sesuaikan `ansible_user` dengan `<USER>` dari ssh tadi.

> Akan lebih baik apabila dapat diautomasi dari pembuatan instance hingga memasukkan IP yang sesuai ke file `host` dari instance tersebut. Hal tersebut dapat dilakukan menggunakan Terraform atau menggunakan script linux secara manual. Namun, hal tersebut belum dapat dilakukan karena kami masih kurang familiar dengan tools tersebut. 

4. Jalankan 

```bash
chmod +x ansible_script.sh
./ansible_script.sh
```

Scripts tersebut terdiri dari inisiasi host dan pembuatan user `ubuntu` pada setiap node agar ansible dapat berkomunikasi dengan berbagai node untuk melakukan instalasi yang diperlukan. Setelah itu, dilakukan juga instalasi berbagai dependensi yang diperlukan untuk membuat kubernetes cluster. Luaran dari scripts tersebut adalah kubernetes cluster telah berjalan dengan node `master` sebagai kubernetes master nya, dan node lainnya sebagai workers agar dapat dilakukan auto deployment dan auto scaling aplikasi dengan mudah pada tiap node. Pastikan tidak ada error setelah menjalankan scripts tersebut. 

5. SSH ke dalam master/controller node kubernetes

Setelah kubernetes cluster telah berhasil dibuat, node master dapat menjalankan berbagai perintah untuk melakukan deployment pada node lainnya. Untuk itu, lakukan SSH ke node master dari PC lokal. 

```bash
ssh -i ~/.ssh/<FILE_NAME> ubuntu@<EXTERNAL_IP_MASTER>
```

Lakukan penyesuaian `<FILE_NAME>` dari SSH yang telah dibuat tadi.

6. Pastikan setiap VM instance telah terhubung

Perlu dipastikan bahwa kubernetes cluster telah muncul dan setiap node workers berstatus `Ready` untuk menerima arahan dari node master. 

Input:

```bash
kubectl get node
```

Output:

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%205.png)

7. Tambahkan label pada setiap node untuk memudahkan deployment pada step selanjutnya.

Penambahan label pada setiap node dilakukan agar nantinya pod terdeploy pada node yang sesuai dengan labelnya.

Input:

```bash
kubectl label nodes master group=master
kubectl label nodes wprofile1 group=wprofile
kubectl label nodes wprofile2 group=wprofile
kubectl label nodes wprofile3 group=wprofile
kubectl label nodes monitoring group=monitoring
kubectl label nodes file-sharing group=file-sharing
kubectl label nodes meeting-app group=meeting-app
kubectl label nodes db group=db
```

Pastikan label telah ditambahkan dengan:

`kubectl get nodes --show-labels`

8. Clone scripts dari node master dan jalankan perintah berikut

```bash
git clone https://github.com/filedumpamal/k8s-ansible-gcp.git
cd k8s-ansible-gcp/scripts-kubernetes/

kubectl apply -f 1-web-profile-deployment/wordpress/
kubectl apply -f 1-web-profile-deployment/petclinic/

kubectl apply -k 3-file-sharing/owncloud/

kubectl apply -f 2-monitoring/
```

Untuk deployment meeting-app,

`cd 4-meeting-application/` dan jalankan perintah pada README.md

Berbagai perintah tersebut dijalankan untuk membuat deployment aplikasi ke node yang sesuai dengan label. Agar pod terdeploy ke node dengan label yang sesuai, terdapat baris 

```yaml
...
nodeSelector:
  group: wprofile
...
```
pada objek kubernetes dengan tipe deployment.

9. Cek pod telah terdeploy di node yang sesuai dengan cara

```bash
kubectl get pods --output=wide
```

10. Cek berbagai objek kubernetes yang telah dibuat dengan cara seperti pada gambar berikut

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%206.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%207.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%208.png)

11. Setting firewall pada GCP console untuk node yang menjalankan service pada port yang sesuai agar service dapat diakses darimanapun.

```bash
gcloud compute firewall-rules create <SETTING_NAME> --allow tcp:<PORT> --source-tags=<VM_INSTANCE_1>,<VM_INSTANCE_2>,<...> --source-ranges=0.0.0.0/0
```

12. Cek masing-masing layanan dapat berjalan dengan lancar. 

- Petclinic

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%209.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2010.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2011.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2012.png)

- Wordpress

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2013.png)

- Owncloud

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2014.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2015.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2016.png)

- Jitsi

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2017.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2018.png)

- Kubernetes Dashboard

Untuk dapat melihat kubernetes dashboard, jalankan perintah berikut agar mendapatkan token untuk login.

Input:

```bash
kubectl get svc -n kubernetes-dashboard
kubectl get secret -n kubernetes-dashboard $(kubectl get serviceaccount admin-user -n kubernetes-dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

Output:

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2019.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2020.png)

Copy token tersebut ke halaman awal untuk login

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2021.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2022.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2023.png)

13. Apabila telah selesai, dapat exit dari node master dengan cara

```bash
exit
```

## Delete semua resources

Untuk melakukan penghapusan semua objek kubernetes yang telah dibuat, dapat menggunakan perintah berikut.

1. SSH ke node master
2. cd directory github

```bash
kubectl delete -f 1-web-profile-deployment/wordpress/
kubectl delete -f 1-web-profile-deployment/petclinic/
kubectl delete -k 3-file-sharing/owncloud/
kubectl delete ns jitsi
kubectl delete -f 2-monitoring/
```

# Testing dan Monitoring

Akan dilakukan testing layanan dengan skenario pertama yaitu tanpa menggunakan load balancer, dan skenario kedua yaitu menggunakan load balancer. Load balancer yang digunakan adalah load balancer bawaan dari Kubernetes. Pada Kubernetes, setiap pods dimana aplikasi berjalan biasanya bersifat sementara dan mendapatkan IP address baru setiap deployment selanjutnya. Pods biasanya secara dinamis dihapus dan dibuat ulang pada setiap deployment. Tanpa adanya kubernetes service, diperlukan tracking terhadap IP adress dari setiap pod. Oleh karena itu, LoadBalancer service digunakan agar dapat mengekspose pod ke luar sesuai dengan metode yang digunakan oleh cloud provider dengan mudah. Layanan yang akan dilakukan pengetesan adalah petclinic dengan melakukan form filling sesuai dengan skenario yang akan dijelaskan lebih lanjut pada bagian selanjutnya. 

## Instalasi Monitoring Tools

Agar dapat melakukan monitoring secara komprehensif, dapat menggunakan layanan monitoring bawaan dari GCP. Untuk itu, perlu dilakukan instalasi agent pada tiap node dengan menuju halaman berikut. 

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2024.png)

Jalankan command tersebut di cloud shell hingga Ops Agent berstatus hijau. 

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2025.png)

## Skenario Testing

Testing ini dilakukan menggunakan tools Apache JMeter. Untuk instalasi dapat mengacu pada https://jmeter.apache.org/usermanual/get-started.html. Selanjutnya, jalankan ApacheJMeter.jar untuk membuka aplikasi. 

Secara umum, siklus penggunaan JMeter adalah:

- Menambahkan Thread Group;
- Menambahkan HTTP Request;
- Menambahkan report untuk menampilkan hasil Performance test;
- Menjalankan Performance test;
- Mengamati dan mengalisis report dari performance test.

Akan dibuat thread group dengan spesifikasi sebagai berikut:
- Number of Threads, yaitu jumlah user sebanyak 100 orang
- Ramp-up period, yaitu 200 detik
- Setiap 2 detik (Ramp-up period/Number of Threads = 200/100 = 2), akan mengirimkan 10 request ke server.
- Total jumlah sample = 1000 (100 x 10)

Berikut merupakan rincian dari pengetesan dan hasilnya. 

### Tanpa load balancer

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2026.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2027.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2028.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2029.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2030.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2031.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2032.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2033.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2034.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2035.png)

### Dengan load balancer

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2036.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2037.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2038.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2039.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2040.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2041.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2042.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2043.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2044.png)

## Monitoring setelah testing

Dapat dilihat dari gambar di atas, untuk pengetesan tanpa load balancer dimulai dari 13:54:17 hingga 13:57:42. Dan pengetesan menggunakan load balancer dari 13:59:00 hingga 14:02:27. 

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2045.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2046.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2047.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2048.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2049.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2050.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2051.png)

![Untitled](Kubernetes%20on%20Bare%20Metal%20GCP%20VM%20Instances%20eb9a10402c3b4ff5bb30be233f68db1b/Untitled%2052.png)

Warna hijau, biru tua, dan abu-abu masing-masing menunjukkan node wprofile1, wprofile2, dan wprofile3 untuk layanan web profile. Warna pink menunjukkan node master sebagai load balancer. Secara keseluruhan, dibandingkan dengan concurrent request dengan load balancer, aktivitas server cenderung lebih berat dari sisi CPU utilization dan network.

## Berbagai Masalah dan Solusi Teknis

Berikut adalah kendala yang dihadapi selama implementasi beserta **solusi teknis yang direkomendasikan** untuk memperbaikinya:

### 1. Masalah Penjadwalan Wordpress (`nodeSelector`)

**Masalah:** Wordpress tidak dapat berjalan dengan benar saat menggunakan `nodeSelector`, sehingga harus dibiarkan tanpa setting tersebut yang mengakibatkan pod ter-deploy secara acak.

**Solusi Teknis:**
Pastikan label pada node worker sudah sesuai. Untuk memastikan Wordpress berjalan di node yang tepat (misalnya `wprofile`), tambahkan konfigurasi `nodeSelector` pada `wordpress-deployment.yaml`:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        group: wprofile
```
Pastikan node target telah dilabeli dengan perintah: `kubectl label nodes <nama-node> group=wprofile`.

### 2. Konflik MySQL & Persistence Volume

**Masalah:** Terjadi konflik saat menjalankan beberapa MySQL di node `db` atau saat pod Wordpress dan Owncloud berjalan di node yang sama. Hal ini disebabkan karena semua konfigurasi Persistent Volume (PV) menggunakan direktori `hostPath` yang sama persis, yaitu `/mnt/data`. Akibatnya, data dari satu aplikasi menimpa data aplikasi lain.

**Solusi Teknis:**
Ubah konfigurasi PV untuk menggunakan sub-direktori yang unik untuk setiap layanan. Jangan arahkan semua ke `/mnt/data`.
Contoh perubahan pada YAML:
- **Wordpress MySQL:** `path: "/mnt/data/wordpress-db"`
- **Wordpress Files:** `path: "/mnt/data/wordpress-files"`
- **Owncloud MySQL:** `path: "/mnt/data/owncloud-db"`
- **Owncloud Files:** `path: "/mnt/data/owncloud-files"`

### 3. Penggunaan `hostPath` vs Cloud Storage

**Masalah:** Penggunaan `hostPath` untuk produksi tidak disarankan karena data terikat pada node fisik tertentu. Jika pod pindah node, data tidak akan terbawa.

**Solusi Teknis:**
- **Untuk Bare Metal/VM Manual:** Jika ingin tetap menggunakan penyimpanan lokal, pastikan menggunakan path yang unik (seperti poin 2) dan gunakan `nodeSelector` agar pod selalu kembali ke node yang sama yang memiliki data tersebut.
- **Rekomendasi Utama (GCP):** Gunakan **StorageClass** dengan provisioner `kubernetes.io/gce-pd`. Ini akan memungkinkan Kubernetes membuat Persistent Disk di GCP secara otomatis yang dapat di-mount ke node manapun pod berjalan.
  Contoh:
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: standard
  provisioner: kubernetes.io/gce-pd
  parameters:
    type: pd-standard
  ```

### 4. Konfigurasi Jitsi (HTTPS & IP)

**Masalah:** Layanan Jitsi error saat memulai meeting.

**Solusi Teknis:**
- **IP Address:** Pastikan environment variable `DOCKER_HOST_ADDRESS` pada `deployment.yaml` diisi dengan **External IP** dari node tempat Jitsi berjalan, bukan IP internal atau IP dummy. Jitsi Video Bridge (JVB) memerlukan ini untuk koneksi WebRTC.
- **SSL/HTTPS:** WebRTC modern membutuhkan koneksi HTTPS yang valid agar browser mengizinkan akses kamera/mikrofon. Self-signed certificate sering ditolak oleh browser. Solusinya adalah menggunakan domain asli dan mengonfigurasi Let's Encrypt, atau me-mount sertifikat SSL yang valid ke dalam container Jitsi.

### 5. Kompleksitas Monitoring

**Masalah:** Kesulitan konfigurasi Zabbix pada cluster Kubernetes.

**Solusi Teknis:**
Keputusan menggunakan **Google Cloud Operations Suite** (Monitoring VM Instance bawaan) sudah sangat tepat untuk infrastruktur berbasis VM di GCP karena instalasinya mudah (hanya install agent) dan terintegrasi langsung. Jika membutuhkan monitoring spesifik level Kubernetes (Pod/Container metrics) yang lebih mendalam di masa depan, pertimbangkan menggunakan stack **Prometheus & Grafana** yang merupakan standar industri untuk monitoring Kubernetes dan lebih mudah dikelola menggunakan Helm Charts.

## Referensi lain
Tes file sharing 

https://www.blazemeter.com/blog/how-performance-test-upload-and-download-scenarios-apache-jmeter

Tes video streaming

https://www.blazemeter.com/blog/load-testing-video-streaming-with-jmeter-learn-how
