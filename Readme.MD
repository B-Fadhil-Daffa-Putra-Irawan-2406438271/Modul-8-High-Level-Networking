# Reflection

## 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?

Dalam gRPC, terdapat tiga metode komunikasi utama yang masing-masing punya
keunggulan tersendiri. Unary RPC adalah metode paling sederhana di mana klien
mengirim satu permintaan dan menerima satu respons, mirip seperti request-response
pada HTTP biasa. Metode ini paling cocok digunakan untuk operasi sederhana yang
tidak memerlukan data berkelanjutan, seperti proses pembayaran pada implementasi
MyPaymentService kita. Server Streaming RPC memungkinkan klien mengirim satu
permintaan namun menerima banyak respons secara bertahap dari server, seperti
yang terlihat pada get_transaction_history yang mengalirkan 30 data transaksi
sekaligus. Metode ini ideal untuk kasus seperti pengambilan data besar atau
laporan real-time. Sementara itu, Bi-directional Streaming RPC seperti pada
service chat kita memungkinkan klien dan server saling mengirim pesan secara
bersamaan, sangat cocok untuk aplikasi yang butuh komunikasi dua arah secara
real-time seperti chat atau game online.

## 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?

Keamanan adalah aspek krusial yang perlu diperhatikan saat membangun gRPC service
di Rust. Dari sisi autentikasi, kita perlu menambahkan mekanisme verifikasi
identitas klien, misalnya menggunakan JWT token yang dikirim melalui metadata
request dan divalidasi oleh interceptor di sisi server. Untuk otorisasi, perlu
diterapkan kontrol akses berbasis peran atau RBAC agar setiap klien hanya bisa
mengakses service yang memang menjadi haknya. Enkripsi data juga wajib
diimplementasikan menggunakan TLS/SSL melalui ServerTlsConfig yang tersedia
di Tonic, sehingga semua data yang dikirim melalui jaringan terlindungi dari
penyadapan. Selain itu, validasi input dari setiap pesan Protobuf yang masuk
juga penting untuk mencegah data yang tidak valid merusak logika bisnis di server.

## 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?

Mengelola bidirectional streaming di Rust gRPC bukanlah hal yang mudah dan
menghadirkan beberapa tantangan nyata. Manajemen konkurensi menjadi tantangan
utama karena kita harus mengelola aliran data masuk dan keluar secara bersamaan
menggunakan tokio::spawn dan mpsc channel, yang jika tidak ditangani dengan
benar bisa menyebabkan deadlock atau race condition. Penanganan error juga menjadi
lebih kompleks karena koneksi bisa terputus kapan saja dari sisi klien maupun
server, sehingga kita perlu memastikan stream ditutup dengan bersih tanpa
menyebabkan panic. Backpressure juga perlu diperhatikan karena jika server tidak
mampu memproses pesan secepat klien mengirimnya, buffer channel bisa penuh dan
menyebabkan masalah memori. Terakhir, manajemen state per koneksi seperti sesi
pengguna menjadi rumit tanpa mekanisme shared state yang aman seperti Arc<Mutex<>>.

## 4. What are the advantages and disadvantages of using the tokio_stream::wrappers::ReceiverStream for streaming responses in Rust gRPC services?

ReceiverStream adalah wrapper yang sangat membantu dalam menjembatani channel
mpsc milik Tokio dengan antarmuka streaming gRPC. Keuntungan utamanya adalah
kemudahan integrasi dengan ekosistem async Tokio sehingga pengiriman pesan
non-blocking bisa dilakukan dengan lebih alami. Selain itu, kita bisa dengan
mudah membuat background task yang menghasilkan data stream secara independen
dari handler gRPC utama. Namun di sisi lain, ReceiverStream memiliki keterbatasan
dalam hal flow control karena pengirim bisa membanjiri penerima jika ukuran buffer
channel tidak dikonfigurasi dengan tepat. Lapisan abstraksi tambahan yang
diberikannya juga bisa membuat proses debugging lifecycle stream menjadi lebih
sulit dibandingkan mengelola stream secara langsung.

## 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?

Agar kode gRPC di Rust mudah dirawat dan dikembangkan, ada beberapa pendekatan
yang bisa diterapkan. Pertama, setiap service sebaiknya dipisahkan ke dalam modul
atau file tersendiri, misalnya payment.rs, transaction.rs, dan chat.rs, sehingga
tanggung jawab tiap bagian kode menjadi jelas. Kedua, logika yang dipakai bersama
seperti logging, tipe error, dan validasi input sebaiknya dikumpulkan dalam modul
common atau utils agar tidak terjadi duplikasi kode. Ketiga, penggunaan trait
abstraction memungkinkan kita untuk membuat mock service saat pengujian tanpa
harus bergantung pada implementasi nyata. Keempat, nilai-nilai konfigurasi seperti
alamat server sebaiknya diambil dari environment variable agar lebih fleksibel
saat deployment di berbagai lingkungan.

## 6. In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic?

Implementasi MyPaymentService saat ini masih sangat sederhana karena langsung
mengembalikan respons sukses tanpa proses apapun. Untuk menangani logika
pembayaran yang lebih kompleks, beberapa langkah tambahan perlu ditambahkan.
Pertama, integrasi dengan payment gateway seperti Midtrans atau Stripe diperlukan
untuk memproses transaksi secara nyata. Kedua, validasi data pembayaran seperti
jumlah transaksi, identitas pengguna, dan ketersediaan dana perlu dilakukan sebelum
memproses request. Ketiga, mekanisme retry dan idempotency key perlu diterapkan
untuk menghindari transaksi ganda jika terjadi kegagalan jaringan. Keempat,
pencatatan log transaksi ke database dan pengiriman notifikasi ke pengguna juga
merupakan bagian penting dari alur pembayaran yang lengkap.

## 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?

Penggunaan gRPC sebagai protokol komunikasi membawa perubahan signifikan pada
arsitektur sistem terdistribusi. Dengan menggunakan Protocol Buffers sebagai format
data, kontrak antara service menjadi lebih ketat dan terdefinisi dengan baik,
sehingga mengurangi risiko ketidakcocokan data antar service. gRPC juga mendukung
berbagai bahasa pemrograman, sehingga service yang ditulis dalam Rust tetap bisa
berkomunikasi dengan service lain yang ditulis dalam Go, Python, atau Java tanpa
masalah interoperabilitas. Namun adopsi gRPC juga berarti tim perlu memahami
konsep Protocol Buffers dan HTTP/2 yang mungkin belum familiar, serta membutuhkan
tooling tambahan untuk debugging karena format binarynya tidak bisa dibaca langsung
seperti JSON.

## 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?

HTTP/2 yang menjadi fondasi gRPC membawa beberapa keunggulan dibanding HTTP/1.1.
Multiplexing memungkinkan banyak request dikirim dalam satu koneksi TCP secara
bersamaan tanpa blocking, sehingga latensi berkurang drastis terutama untuk
komunikasi antar microservice. Header compression pada HTTP/2 juga mengurangi
overhead bandwidth yang biasanya besar pada HTTP/1.1. Namun HTTP/2 juga punya
kekurangan, yaitu kompleksitas implementasi yang lebih tinggi dan dukungan
infrastruktur yang belum merata, misalnya beberapa load balancer lama belum
mendukung HTTP/2 dengan baik. Dibandingkan HTTP/1.1 dengan WebSocket untuk REST
API, gRPC lebih efisien untuk komunikasi antar service internal, namun WebSocket
masih lebih mudah diintegrasikan untuk aplikasi web browser karena dukungan
native-nya yang lebih luas.

## 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?

Model request-response pada REST API bekerja dengan baik untuk operasi yang
bersifat stateless dan tidak memerlukan komunikasi berkelanjutan. Namun untuk
kebutuhan komunikasi real-time, REST API harus mengandalkan teknik seperti
polling atau long-polling yang tidak efisien karena terus membuka koneksi baru.
Bidirectional streaming gRPC seperti pada service chat kita jauh lebih unggul
dalam skenario ini karena koneksi dibuka sekali dan kedua pihak bisa saling
mengirim pesan kapan saja tanpa overhead koneksi berulang. Hal ini membuat
responsivitas aplikasi jauh lebih baik, terutama untuk kasus seperti notifikasi
real-time, live tracking, atau sistem kolaborasi.

## 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?

Penggunaan Protocol Buffers pada gRPC mengharuskan kita mendefinisikan struktur
data secara eksplisit melalui file .proto sebelum bisa dikembangkan. Pendekatan
ini memiliki keunggulan besar dalam hal konsistensi data karena semua pihak yang
terlibat harus sepakat dengan skema yang sama, sehingga error akibat ketidakcocokan
tipe data bisa terdeteksi lebih awal saat kompilasi. Selain itu, format binary
Protocol Buffers jauh lebih ringkas dan cepat diproses dibanding JSON yang
berbasis teks. Di sisi lain, JSON pada REST API menawarkan fleksibilitas yang
lebih tinggi karena struktur datanya bisa berubah tanpa perlu mendefinisikan
ulang skema, yang berguna saat tahap prototyping atau ketika struktur data masih
sering berubah. Namun fleksibilitas ini juga menjadi kelemahan karena validasi
data harus dilakukan secara manual dan lebih rentan terhadap kesalahan runtime.