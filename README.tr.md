🇹🇷 Türkçe | [🇬🇧 English](README.md)

# Göl İşletmeciliği Yönetim Sistemi

Van Gölü'ndeki rekreasyonel balıkçı aktivitelerini yönetmek için geliştirilmiş full-stack bir web uygulaması. Kullanıcılar tekne ve ekipman kiralayabilir, göl bölgelerinin interaktif coğrafi haritasını inceleyebilir, bölge bazlı forumlara katılabilir ve etkinlikleri takip edebilir. Yöneticiler gelir, kullanıcı harcamaları ve bölge istatistiklerini kapsayan analitik panellere erişebilir.

---

## Teknoloji Stack'i

| Katman | Teknoloji | Versiyon |
|---|---|---|
| Frontend | React | ^19.2.0 |
| Frontend | Vite | ^7.2.4 |
| Frontend | Leaflet + React-Leaflet | ^1.9.4 / ^5.0.0 |
| Frontend | React Hot Toast | ^2.6.0 |
| Frontend | React Icons | ^5.5.0 |
| Backend | Node.js + Express | ^5.1.0 |
| Backend | pg (PostgreSQL sürücüsü) | ^8.16.3 |
| Backend | jsonwebtoken | ^9.0.3 |
| Backend | bcrypt | ^6.0.0 |
| Backend | Supabase JS istemcisi | ^2.86.0 |
| Veritabanı | PostgreSQL (Supabase cloud) | — |
| Veritabanı | PostGIS eklentisi | — |

---

## Mimari Genel Bakış

```
Tarayıcı (React SPA)
    │
    │  HTTP + Bearer JWT
    ▼
Express API  (port 3000)
    │  Controller → Service → SQL
    ▼
Supabase PostgreSQL  (cloud)
    │
    └── PostGIS geometri sütunları
        (tekne konumları, göl bölge poligonları, hotspot'lar)
```

**İstek akışı:** React bileşenleri `src/api/api.js`'i çağırır; bu modül `localStorage`'dan JWT'yi alarak Express REST API'ye iletir. Her route modülü bir controller'a, controller ise `pg.Pool` üzerinden parametreli SQL çalıştıran bir service'e delege eder. ORM kullanılmamıştır.

**Auth:** JWT, girişte oluşturulur (7 günlük geçerlilik, HS256). `authMiddleware` korunan route'larda token'ı doğrular; `adminMiddleware` ek olarak `role_id`'yi kontrol eder.

**Coğrafi veri:** Tekne konumları ve göl bölgesi poligonları PostGIS geometrisi (SRID 4326) olarak saklanır. Backend bunları `ST_AsGeoJSON` ile GeoJSON olarak döndürür; frontend Leaflet haritasında gösterir.

**Radar simülasyonu:** `backend/services/radarSimulation.js`, arka planda çalışarak `boats.current_geom` sütununa rastgele/simüle koordinatlar yazar. Bu gerçek GPS verisi değildir.

---

## Özellikler

### Kullanıcılar
- Kayıt ve giriş (JWT tabanlı oturumlar, parolalar bcrypt ile hashlenir)
- Kişisel kiralama geçmişini filtreli görüntüleme
- Kendi harcama istatistiklerini görme

### Harita
- Van Gölü'nün interaktif Leaflet haritası
- Tıklanabilir bölge poligonları — bölge seçimi kenar çubuğu içeriğini filtreler
- Veritabanından çekilen canlı tekne işaretçileri (konumlar simüle edilmiştir, GPS değil)
- Bölge başına hotspot işaretçileri

### Tekne Kiralama
- Mevcut tekneleri fiyata göre sıralı listeleme
- Belirli bir süre için tekne kiralama (`fiyat_per_saat × saat` hesabıyla)
- Aktif kiralamaları görüntüleme ve tamamlama
- Yönetici: tekne oluşturma, düzenleme, silme

### Ekipman Kiralama
- Teknelerle aynı kiralama akışı, balıkçılık ekipmanlarına uygulanmış
- Tüm aktif ekipman kiralamalarını tek seferde tamamlamak için "tümünü iade et" uç noktası
- Yönetici: ekipman envanterini yönetme

### Aktiviteler / Etkinlikler
- Bölge bazında zamana göre sınıflandırılmış aktivite listesi (geçmiş, güncel, yaklaşan)
- Yönetici: aktivite oluşturma, düzenleme, silme

### Forum
- Bölge kapsamlı gönderiler (başlık, içerik, fotoğraf URL'leri)
- Gönderilere yorum ve beğeni
- Yönetici: kullanıcı başına forum istatistiklerini görüntüleme

### Yönetici Analitiği
- **AdminStatsPanel:** kullanıcı harcama sıralaması, forum gönderi/yorum sayıları, bölge aktivite/gönderi/süre istatistikleri, popüler bölgeler
- **AccountingPanel:** aylık gelir dağılımı, tekne ve ekipman gelir karşılaştırması, zaman içi trend analizi
- **RentalHistoryPanel:** durum filtreli tam kiralama geçmişi

---

## Klasör Yapısı

```
balik-proje/
├── backend/
│   ├── config/db.js          # Supabase'e bağlı pg.Pool
│   ├── controllers/          # 8 controller (alan başına bir tane)
│   ├── middleware/           # auth, admin, hata işleyici, istek loglayıcı
│   ├── routes/               # 9 Express router
│   ├── services/             # 11 service + radarSimulation.js
│   ├── utils/geojson.js
│   └── server.js
├── frontend/
│   └── src/
│       ├── api/api.js        # Tek fetch sarmalayıcı (~430 satır)
│       ├── components/
│       │   ├── GameMap/      # Leaflet harita + işaretçiler
│       │   ├── Sidebar/      # 5 sekmeli kenar çubuğu (Bilgi, Tekne, Ekipman, Forum, Hesap)
│       │   ├── features/     # Alan bazlı liste/kart bileşenleri
│       │   ├── forms/        # Giriş, Kayıt, Tekne, Ekipman, Aktivite formları
│       │   ├── modals/       # Onay, ImageLightbox, GönderiOluştur
│       │   ├── panels/       # Yönetici panelleri
│       │   └── ui/           # Button, Card, Input, Modal, Badge, vb.
│       ├── App.jsx
│       └── main.jsx
├── database/
│   ├── create_activities.sql
│   ├── create_equipment_rentals.sql
│   ├── insert_sample_activities.sql
│   ├── project_queries.sql       # 10 karmaşık analitik sorgu
│   └── advanced_queries.sql
└── .env
```

---

## Kurulum ve Çalıştırma

### Ön Koşullar
- Node.js ≥ 18
- Şeması uygulanmış bir Supabase projesi (`database/` SQL dosyalarına bakın)
- Doldurulmuş `.env` dosyası (aşağıya bakın)

### 1. Ortam Değişkenleri

Proje kök dizininde aşağıdaki içerikle bir `.env` dosyası oluşturun:

```env
PORT=3000
JWT_SECRET=<güçlü-rastgele-bir-secret-ile-değiştirin>
JWT_EXPIRES_IN=7d
SUPABASE_URL=https://<projeniz>.supabase.co
SUPABASE_ANON_KEY=<anon-anahtarınız>
DATABASE_URL=postgresql://postgres.<projeniz>:<parola>@aws-0-eu-central-1.pooler.supabase.com:6543/postgres
```

### 2. Veritabanı

SQL dosyalarını Supabase SQL Editörü'nde (ya da `psql` ile) şu sırayla çalıştırın:

```
database/create_activities.sql
database/create_equipment_rentals.sql
database/insert_sample_activities.sql   # isteğe bağlı örnek veri
```

### 3. Backend

```bash
cd backend
npm install
node server.js
# API http://localhost:3000 adresinde dinler
```

Yönetici kullanıcı oluşturmak için:

```bash
node create_admin_user.js
```

### 4. Frontend

```bash
cd frontend
npm install
npm run dev
# Geliştirme sunucusu http://localhost:5173 adresinde
```

Üretim derlemesi:

```bash
npm run build   # çıktı frontend/dist/ dizininde
```

---

## Bilinen Eksikler ve Yapılacaklar

- **Fotoğraf yükleme uygulanmamış.** Veritabanında `post_photos` tablosu mevcut ve frontend fotoğraf URL'lerini gösteriyor, ancak backend'de dosya yükleme uç noktası yok. Fotoğraflar önceden barındırılmış URL olarak girilmeli.
- **Ödeme işleme uygulanmamış.** Şemada `payments` tablosu var, ancak herhangi bir ödeme altyapısı entegre edilmemiş. Kiralama tutarları kaydediliyor fakat gerçek bir tahsilat yapılmıyor.
- **Radar simülasyonu gerçek değil.** Haritadaki tekne konumları `radarSimulation.js` tarafından üretiliyor; gerçek GPS cihazlarından gelmiyor.
- **E-posta doğrulaması yok.** Kayıt, onay e-postası gönderilmeksizin herhangi bir e-posta adresini kabul ediyor.
- **Rate limiting yok.** Auth uç noktaları (`/register`, `/login`) kaba kuvvet saldırılarına karşı korumasız.
- **`.env`'deki JWT secret bir yer tutucudur.** Yerel dışındaki herhangi bir ortama geçmeden önce `super-secret-change-this` değeri değiştirilmeli.
- **`SUPABASE_ANON_KEY` `.env` dosyasına işlenmiş.** Bu dosya versiyon kontrolünde olmamalı. Repo herkese açıksa veya açıktıysa anahtarı yenileyin.
- **Tekne müsaitliği yarış koşulu.** Service, tekne satırını kilitlemek için `SELECT FOR UPDATE` kullanıyor; ancak eş zamanlı yük altındaki davranış doğrulanmamış.
