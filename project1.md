### 🟢KOLAY — Doğrudan Proxy + Basit Filtre

**`GET /api/posts`**

- jsonplaceholder'dan `/posts` çek, olduğu gibi dön.

**`GET /api/posts/:id`**

- Tek post getir, bulunamazsa 404 dön (jsonplaceholder boş obje `{}` döner, sen kontrol edip kendi 404'ünü üretmelisin).

**`GET /api/users/:id`**

- Kullanıcıyı getir, response'u kendi tipine (TS interface) map'leyip sadece `id, name, email, city` alanlarını dön (gereksiz alanları at).

**`GET /api/todos?completed=true`**

- jsonplaceholder'daki tüm todos'u çek, **kendi backend'inde** query param'a göre filtrele (jsonplaceholder'ın kendi filtresini kullanma, sen JS ile filtrele).

**`POST /api/posts`**

- Body'den `{ title, body, userId }` al, TS interface ile tipini doğrula (`title` ve `body` string, boş olamaz; `userId` number olmalı).
- Validasyon geçerse jsonplaceholder'a `POST /posts` at, dönen veriyi (jsonplaceholder genelde `id: 101` ekleyerek döner) kendi response'unla beraber dön.
- Validasyon geçmezse jsonplaceholder'a hiç istek atma, direkt 400 dön.

**Hedef:** Dış API'den veri çekme, type tanımlama, response shaping, manuel filtreleme.

---

### 🟡 ORTA — Birden Fazla Endpoint'i Birleştirme

**`GET /api/posts/:id/full`**

- Tek bir post getir.
- O post'un yazarını (`userId` üzerinden `/users/:id`) getir.
- O post'a ait yorumları (`/posts/:id/comments`) getir.
- Üçünü birleştirip şu şekilde dön:

```json
{
  "post": { "id": 1, "title": "...", "body": "..." },
  "author": { "name": "...", "email": "..." },
  "comments": [ { "name": "...", "email": "...", "body": "..." } ]
}
```

- **Önemli:** Bu istekler paralel olarak atılacak. Tek tek değil.

**`GET /api/users/:id/dashboard`**

- Kullanıcının bilgisi + kaç post yazdığı + kaç todo'su tamamlanmış (sayı olarak) + kaç albümü olduğunu tek response'ta dön.
- Burada `/posts?userId=X`, `/todos?userId=X`, `/albums?userId=X` çağrılarını **paralel** yap.

**`GET /api/albums/:id/photos`**

- Albümdeki fotoğrafları getir, ama her birinin `url` alanını at, sadece `thumbnailUrl` ve `title` dön (response transformation).

**`GET /api/posts/most-commented?limit=5`**

- Tüm postları çek, her biri için yorum sayısını bul (yine paralel istek!), yorum sayısına göre sırala, en çok yorumlanan `limit` kadar post'u dön.
- **Hedef:** `Promise.all`, veri birleştirme (aggregation), sıralama, N+1 sorununu paralel istekle çözme.

**`PUT /api/posts/:id`**

- Body'den tam post objesi al (`title`, `body`, `userId`), `id`'yi URL'den al.
- jsonplaceholder'a `PUT /posts/:id` at.
- **Önce** o id'li post'un gerçekten var olup olmadığını `GET /posts/:id` ile kontrol et (jsonplaceholder var olmayan id'lerde bile 500 vermeden sahte 200 dönebiliyor — bu yüzden kendi 404 kontrolünü sen yapmalısın, ör: id > 100 ise yok say).
- **Hedef:** Write işleminden önce "var mı?" kontrolü yapma — gerçek API'lerde sık görülen pattern.

**`PATCH /api/posts/:id`**

- Body'den **sadece** `{ title }` veya `{ body }` gibi kısmi alan al (PUT'tan farkı: tüm alanı değil, sadece değişen alanı gönderiyoruz).
- jsonplaceholder'a `PATCH /posts/:id` at, dönen sonucu kendi orijinal post verinle birleştirip (`{ ...orijinalPost, ...patchEdilenAlanlar }`) dön.
- **Hedef:** PUT ile PATCH'in farkını pratikte yaşamak.

**`DELETE /api/posts/:id`**

- jsonplaceholder'a `DELETE /posts/:id` at.
- Silinen postun bilgisini (silmeden önce `GET` ile çekip) response'ta "ne silindi" diye dön: `{ message: "Post silindi", silinenPost: {...} }`.
- **Hedef:** Silme işleminden önce veriyi yedekleme/loglama pratiği — gerçek sistemlerde "soft delete" ya da audit log mantığının başlangıcı.

---

### 🔴 ZOR — Çoklu Kaynak, Gruplama, İstatistik, Hata Yönetimi

**`GET /api/users/:id/report`**  
Tam bir kullanıcı raporu — en kapsamlı endpoint:

```json
{
  "user": { "name": "...", "email": "...", "company": "..." },
  "stats": {
    "totalPosts": 10,
    "totalComments": 50,
    "totalTodos": 20,
    "completedTodos": 12,
    "completionRate": "60.00%",
    "totalAlbums": 5,
    "totalPhotos": 250
  },
  "topPost": { "title": "...", "commentCount": 8 }
}
```

- `completionRate` hesaplanmalı (`completedTodos / totalTodos * 100`, sıfıra bölme kontrolü!).
- `topPost`, kullanıcının en çok yorum alan postu olmalı (yine yorum sayılarını paralel çekip karşılaştırma).
- Bu endpoint'te en az 5 farklı dış API çağrısı olacak, hepsi mümkün olduğunca paralel çalışmalı.

**`GET /api/albums/grouped-by-user`**

- Tüm kullanıcıları ve tüm albümleri çek.
- Albümleri `userId`'ye göre grupla.
- Her grup için kullanıcı adıyla beraber albüm listesini dön:

```json
[
  { "userName": "Leanne Graham", "albumCount": 10, "albums": [...] },
  ...
]
```

**`GET /api/comments/suspicious`**

- Tüm yorumları çek.
- Email formatı geçersiz olanları (regex ile basit bir kontrol) veya `body` alanı 10 karakterden kısa olanları "şüpheli" say.
- Bunları `postId`'ye göre grupla, hangi post'ta kaç şüpheli yorum var göster.
- **Hedef:** Regex validasyon + filtreleme + gruplama birlikte.

**`GET /api/health-check`**

- jsonplaceholder'ın 6 endpoint'ine (`/posts`, `/comments`, `/albums`, `/photos`, `/todos`, `/users`) aynı anda istek at.
- Her birinin yanıt süresini (`Date.now()` farkıyla) ve başarılı/başarısız olduğunu ölç.
- Biri başarısız olsa bile diğerleri etkilenmesin (`Promise.allSettled` kullanmaları gerekiyor — `Promise.all` burada patlar!).

```json
{
  "posts": { "status": "ok", "responseTimeMs": 120 },
  "comments": { "status": "ok", "responseTimeMs": 95 },
  ...
}
```

- **Hedef:** `Promise.allSettled` vs `Promise.all` farkını öğrenme, hata izolasyonu.

**`POST /api/posts/:id/comment-and-notify`**  
İki write işlemini birleştiren senaryo:

- Body: `{ name, email, body }` (yorum bilgisi).
- Önce postun var olduğunu kontrol et (yoksa 404).
- jsonplaceholder'a `POST /posts/:id/comments` ile yorumu ekle (jsonplaceholder bu nested write route'unu da destekler).
- Yorum "eklendikten sonra" o post'un yazarına (userId üzerinden) ait email bilgisini çek.
- Response'ta sahte bir "bildirim" objesi dön:

```json
{
  "comment": { ... },
  "notification": {
    "to": "yazarın-email@...",
    "message": "Postunuza yeni bir yorum geldi"
  }
}
```

- **Hedef:** Write + read işlemlerini zincirleme, gerçekçi bir "yorum at → yazarı bul → bildirim simüle et" akışı.

**`POST /api/posts/bulk`**

- Body: `{ titles: string[] }` — birden fazla başlık gönderilir.
- Her başlık için ayrı bir `POST /posts` isteği at (hepsi aynı `userId: 1` ile).
- Hepsini **paralel** (`Promise.all`) at, ama biri başarısız olursa diğerlerinin sonucunu kaybetme (`Promise.allSettled` kullan).
- Response: kaçı başarılı, kaçı başarısız, başarılı olanların listesi.
- **Hedef:** Toplu (bulk) write işlemi + kısmi hata toleransı.

**`PATCH /api/users/:id/sync`**  
En zoru — gerçek bir "senkronizasyon" simülasyonu:

- Kullanıcının mevcut bilgisini çek.
- Kullanıcının tüm postlarını çek, kaç tane olduğunu say.
- Eğer post sayısı belirli bir eşiği geçiyorsa (örnek: > 5), kullanıcının `company.name` alanını `PATCH /users/:id` ile `"Aktif Yazar"` olarak güncelle.
- Geçmiyorsa hiçbir write işlemi yapma, sadece "güncellemeye gerek yok" mesajı dön.
- **Hedef:** Okuma sonucuna göre **koşullu** write kararı verme — iş mantığı (business logic) kurma pratiği.

---

### Genel Kurallar (Tüm Seviyeler İçin)

1. Her response için TS interface/type zorunlu (`any` kullanmak yasak).
2. Dış API'ye giden istek başarısız olursa (network hatası, 404 vs.) kendi backend'in 500 değil, anlamlı bir hata + status code dönmeli.
3. Her route `asyncHandler` ile sarılmalı, tekrar eden try/catch yazılmamalı.
4. Zor seviyedeki her endpoint, en az bir yerde `Promise.all` veya `Promise.allSettled` kullanmak zorunda.
