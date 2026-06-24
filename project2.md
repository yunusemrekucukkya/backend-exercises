
### 🟢 KOLAY — Temel Middleware Kavramları

**CORS**

- `cors` middleware'ini sadece `http://localhost:5173` (örnek bir frontend origin) için aç, diğer originlerden gelen isteklerde tarayıcı hatası alındığını gözlemleyin.
- Bonus: Postman'dan istek atınca CORS hatası alıp almadığınıza bakın. Neden?

**Cookie**

- `GET /cookie/set` → `res.cookie("ziyaret", "1", { httpOnly: true })` ile cookie set etsin.
- `GET /cookie/oku` → o cookie'yi okuyup dönsün, yoksa "henüz ziyaret yok" desin.
- `GET /cookie/sil` → cookie'yi temizlesin (`clearCookie`).
- **Hedef:** `httpOnly`, `secure`, `maxAge` seçeneklerinin ne işe yaradığını anlama.

**Basit API Key Middleware**

- Header'da `x-api-key: gizli-123` yoksa veya yanlışsa 401 dönen bir middleware yaz, sadece belirli route grubuna uygula (`router.use`).
- **Hedef:** Middleware'in belirli route'lara nasıl scope edildiğini görme (global `app.use` vs `router.use`).

**Redis'e "Merhaba Dünya" (Redis'i ilk tanıma)**

- `GET /redis/yaz?key=isim&value=Ahmet` → Redis'e `SET isim Ahmet` yaz.
- `GET /redis/oku?key=isim` → Redis'ten oku, yoksa "bulunamadı" dön.
- `GET /redis/sil?key=isim` → `DEL` ile sil.
- **Hedef:** Redis'in sadece bir key-value deposu olduğunu, DB'den farklı (RAM'de, hızlı, basit) çalıştığını anlamak. Henüz TTL, cache mantığı yok — sadece "Redis'le nasıl konuşuyorum" pratiği.

**Ziyaret Sayacı (Redis ile basit state)**

- `GET /redis/ziyaret-sayaci` → her çağrıldığında Redis'te bir sayaç `INCR` ile 1 artsın, güncel sayıyı dönsün.
- Sunucuyu yeniden başlatıp sayacın **sıfırlanmadığını** (bellekteki array'den farklı olarak) fark etsinler.
- **Hedef:** Redis'in process'ten bağımsız, kalıcı (persistece) bir veri deposu olduğunu görmek — bu, sonraki TTL/cache konularına geçişin temeli.

---

### 🟡 ORTA — JWT ile Auth

**`POST /auth/register`**

- Body: `{ email, password }`. Email zaten varsa 409 dön.
- `bcrypt.hash` ile şifreyi hashleyip bellekteki `users` array'ine ekle.

**`POST /auth/login`**

- Email + şifre doğrula (`bcrypt.compare`).
- Doğruysa:
    - **Access token** (JWT, 15 dakika ömürlü) → response body'de dön.
    - **Refresh token** (JWT, 7 gün ömürlü) → `httpOnly` cookie olarak set et.

**`auth.middleware.ts` — JWT Doğrulama**

- `Authorization: Bearer <token>` header'ından access token'ı al, doğrula.
- Geçersiz/süresi dolmuşsa 401 dön, geçerliyse `req.user = { id, email }` set edip `next()`.
- Bu middleware'i Faz 1'deki **yazma işlemlerine** (POST/PUT/PATCH/DELETE) uygula — artık post oluşturmak için login olmak gerekiyor.

**`POST /auth/logout`**

- Refresh token cookie'sini temizle.

**Hedef:** Access/refresh token ayrımı, middleware ile route koruma, stateless auth mantığı.

**TTL'li Cache (Redis'in asıl gücü — ilk gerçek kullanım)**

- `GET /api/posts` isteğine gelen jsonplaceholder sonucunu Redis'e `SET posts:all <data> EX 60` ile yaz.
- Aynı istek 60 saniye içinde tekrar gelirse jsonplaceholder'a gitmeden Redis'ten dön, response'a `"source": "cache"` ekle; gitmişse `"source": "api"` ekle.
- Postman'da arka arkaya 2 kere çağırıp ikincide hızının arttığını (`Date.now()` farkıyla ölçtürerek) ve `"source"` değerinin değiştiğini gözlemle.
- **Hedef:** TTL kavramını ve "neden cache kullanırız" sorusunu pratikte yaşamak — burada artık Redis'in DB/array'den ne farkı var net şekilde anlaşılıyor.

**Basit Rate Limit (TTL + INCR birlikte)**

- `POST /auth/login` route'una: her IP için Redis'te sayaç (`INCR`), ilk istekte `EXPIRE 60`. Dakikada 10'dan fazla istek → 429.
- **Hedef:** Önceki iki Redis dersini (sayaç + TTL) birleştirip gerçek bir güvenlik problemine uygulamak.

---

### 🔴 ZOR — Redis ile Gerçek Backend Sorunlarını Çözme

**Rate Limiting (Redis)**

- `rateLimit.middleware.ts`: her IP için Redis'te bir sayaç tut (`INCR`), ilk istekte `EXPIRE 60` (saniye) set et.
- Dakikada 10 istekten fazla atan IP'ye 429 (`Too Many Requests`) dön.
- Bunu özellikle `POST /auth/login` route'una uygula — kaba kuvvet (brute-force) saldırı senaryosunu simüle et.
- **Hedef:** Redis'in sayaç + TTL ile rate-limit'te nasıl kullanıldığını anlama.

**Response Cache (Redis)**

- `cache.middleware.ts`: `GET /api/posts` isteğine gelen sonucu Redis'e `SET key value EX 60` (60 saniye TTL) ile yaz.
- Aynı istek 60 saniye içinde tekrar gelirse jsonplaceholder'a gitme, Redis'ten dön (response'a `"source": "cache"` ekleyip görsünler).
- `POST /api/posts` çalıştığında ilgili cache key'ini `DEL` ile temizle (cache invalidation).
- **Hedef:** Cache-aside pattern, TTL, cache invalidation — gerçek production sistemlerinin temel sorunu.

**Refresh Token Yönetimi (Redis ile Blacklist/Whitelist)**

- Login olunca üretilen refresh token'ı Redis'e `SET refresh:<userId> <token> EX 604800` (7 gün) ile kaydet.
- `POST /auth/refresh`: cookie'deki refresh token'ı al, Redis'teki kayıtla eşleşiyor mu kontrol et (eşleşmiyorsa = çalıntı/geçersiz token, 401 dön), eşleşiyorsa yeni bir access token üret.
- `POST /auth/logout`'ta Redis'teki kaydı da `DEL` ile sil — yani logout olduktan sonra o refresh token bir daha asla geçerli olmasın (normal JWT'de bu mümkün değildir, Redis bu yüzden kullanılır).
- **Hedef:** JWT'nin "stateless ama logout'ta iptal edilemiyor" probleminin Redis ile nasıl çözüldüğünü kavrama — bu, junior seviyenin çoğunun anlamadığı kritik bir konu.

**Role-Based Middleware**

- Kullanıcı objesine bir `role: "user" | "admin"` alanı ekle (register'da default `"user"`).
- `role.middleware.ts`: sadece `role === "admin"` olanların geçmesine izin ver.
- `DELETE /api/posts/:id` route'unu hem `auth.middleware` hem `role.middleware("admin")` ile koru (middleware zincirleme: `router.delete("/:id", authMiddleware, roleMiddleware("admin"), controller)`).
- **Hedef:** Middleware zincirleme (chaining) ve sıralamanın önemini görme.

---

### Genel Kurallar

1. JWT secret'ları asla kod içine yazma, `.env` dosyasından oku (`dotenv` paketi).
2. Redis bağlantısı kopuksa uygulama çökmemeli — `redis.ts` içinde bağlantı hatası yakalanmalı.
3. Şifreler **asla** plain text loglanmamalı veya response'ta dönülmemeli (`password` alanını response'tan exclude et).
4. Middleware sırası önemli: `cors` → `cookie-parser` → `express.json()` → rate limit → auth → route.

### Öğrenmeniz Gereken Teknolojiler

1. Redis (https://redis.io/)
2. Postman (https://www.postman.com/)

### Gerekli Kütüphaneler


```bash
npm install express cors cookie-parser jsonwebtoken bcrypt ioredis dotenv
npm install -D typescript ts-node-dev @types/express @types/cors @types/cookie-parser @types/jsonwebtoken @types/bcrypt @types/node
```

| Kütüphane       | Ne işe yarar                                                                                           |
| --------------- | ------------------------------------------------------------------------------------------------------ |
| `express`       | Server framework                                                                                       |
| `cors`          | Cross-origin istek izinleri                                                                            |
| `cookie-parser` | `req.cookies` üzerinden cookie okuma                                                                   |
| `jsonwebtoken`  | JWT üretme/doğrulama                                                                                   |
| `bcrypt`        | Şifre hashleme                                                                                         |
| `ioredis`       | Redis client (Node için en yaygın, `redis` paketinden farklı olarak Promise tabanlı API'si daha rahat) |
| `dotenv`        | `.env` dosyasını okuma                                                                                 |
| `ts-node-dev`   | TS dosyasını derlemeden çalıştırma + otomatik restart (dev ortamı)                                     |
