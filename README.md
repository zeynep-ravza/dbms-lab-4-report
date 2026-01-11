# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

CEVAP : Sistem Programlama ve Veri Yapıları perspektifinden veritabanı performansını öne çıkaran temel husus, donanım darboğazlarını optimize edilmiş yazılım mimarisiyle aşmaktır. Sistem programlama açısından performans, milisaniyeler süren maliyetli disk erişimini minimize etmek amacıyla verinin byte yerine blok bazlı okunması, Buffer Pool mekanizmasıyla sık kullanılan verilerin RAM'de önbelleklenmesi ve WAL ilkesiyle rastgele disk yazımlarının sıralı hale getirilmesiyle sağlanır. Veri yapıları perspektifinde ise performans, milyonlarca kayıt üzerindeki arama işlemlerinin B+ Tree yapısı sayesinde logaritmik (O(log N)) karmaşıklığa indirilmesi ve bellek yönetiminin Çift Yönlü Bağlı Listeler kullanan LRU algoritmalarıyla işlemciye yük bindirmeden (O(1) sürede) gerçekleştirilmesiyle öne çıkar.

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [x]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [X]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [X]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [X]  LRU / CLOCK gibi algoritmaları
- [X]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [X]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [ ]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [X]  WAL (Write Ahead Log) İlkesi
- [X]  Log disk (fsync vs write) sistem çağrıları farkı

---

# Özet Tablo

| Kavram | Bellek (RAM) Karşılığı | Disk / DB Karşılığı  |
| :--- | :--- | :--- |
| **Adresleme** | Pointer (`DbPage *`) | Block ID (`Pgno`) + Offset |
| **Erişim Birimi** | Byte / Struct | Page (Sayfa/Blok) - 4KB |
| **Hız / Maliyet** | O(1) / Nanosaniye | Page I/O / Milisaniye |
| **Veri Yapısı** | Linked List (LRU Listesi) | B+ Tree (Nodes & Leaves) |
| **Cache Yönetimi** | OS Page Cache | Buffer Pool (Özelleşmiş LRU) |
| **Arama** | Hash Map (Cache Lookup) | `rowid` ile Ağaç Taraması |
| **Kalıcılık** | Volatile (Uçucu) | WAL (Write Ahead Log) + fsync |

---

# Video [Linki](https://youtu.be/ygw76Ve77-8) 
Rapor Videosu

---

# Açıklama

### Giriş
Bu çalışmanın amacı, Veritabanı Yönetim Sistemleri dersinde teorik olarak işlediğimiz mimari kavramların, gerçek dünyada en yaygın kullanılan ilişkisel veritabanı olan **SQLite** kaynak kodundaki karşılıklarını incelemektir.

Çalışma kapsamında **İşletim Sistemi perspektifi** (Disk I/O, Cache) ve **Veri Yapıları perspektifi** (B+ Tree, Algoritmalar) ele alınmış; ilgili mekanizmaların C dilindeki uygulamaları `src` dizini altındaki kaynak kodlar üzerinden analiz edilmiştir.

### 1. Sistem Perspektifi: Blok Bazlı Disk Erişimi ve Sayfalama
Veritabanı sistemlerinin performansını belirleyen en kritik faktör Disk I/O maliyetidir. İşletim sistemleri ve diskler veriye byte veya satır bazında değil, bloklar halinde erişir. SQLite mimarisinde bu yapının **"Page"** kavramıyla karşılandığını `src/pager.c` dosyasında gözlemledik.

* **Okuma İşlemi (Read):** `sqlite3PagerGet` fonksiyonunu incelediğimizde, veritabanının diskten veri isterken satır numarası değil, **`Pgno pgno` (Page Number)** parametresini kullandığını tespit ettik. Bu değişken, işletim sistemi seviyesindeki **Block ID** kavramının veritabanındaki karşılığıdır. Fonksiyonun dönüş değeri olan `DbPage`, verinin diskten 4KB’lık (varsayılan) bloklar halinde belleğe taşındığını kanıtlamaktadır.

* **Yazma İşlemi (Write):** Benzer şekilde `sqlite3PagerWrite` fonksiyonu, parametre olarak **`PgHdr *pPg` (Page Header)** yapısını almaktadır. Bu durum, SQLite’ın yazma işlemlerini de satır bazlı değil, sayfa bütünü üzerinden yönettiğini ve "Block I/O" prensibine sadık kaldığını göstermektedir.

### 2. Bellek Yönetimi: Buffer Pool ve LRU Algoritması
Disk erişimi maliyetli (milisaniye mertebesinde) olduğu için veritabanları sık kullanılan sayfaları RAM’de tutmak zorundadır. Bu yapıya **Buffer Pool** adı verilir. SQLite’ın bu yönetimi `src/pcache1.c` (Page Cache) modülünde gerçekleştirdiğini analiz ettik.

* **Önbellekleme (Caching):** `pcache1Fetch` fonksiyonu, istenen sayfanın önce RAM’de (Hash Map üzerinde) olup olmadığını kontrol eder. Bu fonksiyon, disk I/O maliyetini **O(1)** seviyesindeki bellek erişim hızına düşüren temel mekanizmadır.

* **Sayfa Değiştirme (Page Replacement):** Bellek dolduğunda hangi sayfanın atılacağına karar vermek için **LRU (Least Recently Used)** algoritmasının kullanıldığını kod seviyesinde kanıtladık. `pcache1Unpin` fonksiyonu incelendiğinde, işi biten sayfaların `pLruNext` ve `pLruPrev` pointer’ları kullanılarak bir **Linked List** yapısına eklendiği görülmüştür. Bu liste, en az kullanılan sayfayı kuyruğun sonunda tutarak, tahliye gerektiğinde hangi sayfanın bellekten atılacağını belirler.

### 3. Veri Yapıları Perspektifi: B+ Tree ve Arama Algoritmaları
Verilerin disk bloklarında tutulması tek başına yeterli değildir; bu verilere hızlı erişim için organize edilmeleri gerekir. SQLite, veriyi disk üzerinde **B+ Tree** veri yapısıyla tutar.

* **Arama (Traversal):** `src/btree.c` dosyasındaki `sqlite3BtreeTableMoveto` fonksiyonu, bir kaydı bulmak için **`i64 intKey` (RowID)** parametresini kullanır. Fonksiyon, B+ Tree’nin kök düğümünden başlayarak, iç düğümler üzerinden yaprak düğümlere kadar iner. Bu kod analizi, teorik derslerde gördüğümüz "Ağaç üzerinde gezinme" (Tree Traversal) işleminin C dilindeki somut uygulamasıdır. Bu yapı sayesinde milyonlarca kayıt içeren tablolarda arama işlemi, tam tablo taraması yerine logaritmik karmaşıklıkla gerçekleştirilir.

### 4. Veri Güvenliği: WAL (Write Ahead Log) ve Kalıcılık
Veritabanlarının en önemli vaadi, sistem çökse bile verinin kaybolmamasıdır. SQLite’ın bunu sağlamak için **WAL** ilkesini kullandığını `src/wal.c` dosyasında tespit ettik.

* **Loglama ve fsync:** `sqlite3WalFrames` fonksiyonu, değişen sayfaları doğrudan ana veritabanı dosyasına yazmak yerine önce WAL dosyasına yazar. Fonksiyonun parametreleri arasında yer alan **`sync_flags`**, verinin sadece işletim sistemi önbelleğine yazılmakla kalmayıp, **`fsync`** sistem çağrısıyla diske fiziksel olarak kazınmasını garanti eder. Bu mekanizma, "Log Disk vs Write" ayrımının kod tarafındaki en net kanıtıdır.

Yapılan incelemeler sonucunda; SQLite'ın bir "kara kutu" olmadığı, işletim sistemi prensipleri (Sayfalama, Cache, I/O) ve Veri Yapıları algoritmalarının (B-Tree, Linked List) hassas bir bileşimi olduğu gösterilmiştir.



## VT Üzerinde İncelenen Kaynak Kodlar

Aşağıdaki bağlantılar, proje kapsamında analiz edilen SQLite (GitHub Mirror) kaynak kodlarını işaret etmektedir:

* **1. Blok Bazlı Disk Erişimi :** [pager.c - sqlite3PagerGet Fonksiyonu](https://github.com/sqlite/sqlite/blob/master/src/pager.c)

* **2. Sayfa Tabanlı Veri Yönetimi :** [pager.c - sqlite3PagerWrite Fonksiyonu](https://github.com/sqlite/sqlite/blob/master/src/pager.c)

* **3. Önbellekleme ve RAM Erişimi (Caching):** [pcache1.c - pcache1Fetch Fonksiyonu](https://github.com/sqlite/sqlite/blob/master/src/pcache1.c)

* **4. LRU Algoritması ve Sayfa Tahliyesi:** [pcache1.c - struct PgHdr1 ve pcache1Unpin](https://github.com/sqlite/sqlite/blob/master/src/pcache1.c)

* **5. Buffer Pool ile Disk I/O Optimizasyonu:** [pcache1.c - Buffer Pool Mekanizması](https://github.com/sqlite/sqlite/blob/master/src/pcache1.c)

* **6. B+ Tree Arama ve Gezinme:** [btree.c - sqlite3BtreeTableMoveto](https://github.com/sqlite/sqlite/blob/master/src/btree.c)

* **7. Veri Güvenliği ve WAL İlkesi:** [wal.c - sqlite3WalFrames Fonksiyonu](https://github.com/sqlite/sqlite/blob/master/src/wal.c)

* **8. Kalıcılık Garantisi (fsync vs write Ayrımı):** [wal.c - sqlite3WalFrames (sync_flags parametresi)](https://github.com/sqlite/sqlite/blob/master/src/wal.c)


