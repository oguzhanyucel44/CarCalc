# CarCalc — Tasarım Kararları

**Paket:** `com.oguzhanyucel.carcalc` · **Ad:** CarCalc
**Platform:** Android Automotive OS, tüm AAOS araçları (dikey + yatay)
**Model:** Ücretsiz, reklamsız, uygulama içi satın alma yok
**Tarih:** 2026-09-01 · Referans: `../prompt.md`

> Bu dosya kodlamaya başlamadan önce verilmiş kararları tutar. `prompt.md` "AAOS'ta ne çalışır"ı,
> bu dosya "CarCalc ne olacak"ı anlatır. Karar değişirse burası güncellenir.

---

## 1. Verilmiş kararlar

| Konu | Karar | Gerekçe |
|---|---|---|
| Mimari | **Şablonsuz normal Activity** (prompt.md §4.1) | Tuş takımı gerekiyor; Surface'e dokunma uygulamaya iletilmiyor (§5.4), Car App Library'de tuş takımı şablonu yok |
| Sürüş davranışı | **`distractionOptimized` YOK** | Araç hareket edince sistem uygulamayı kendi kilit ekranıyla kapatsın. EX30 File Explorer deseni, gerçek araçta doğrulanmış (§4.2) |
| `CarUxRestrictionsManager` | **Kullanılmıyor** | Sistem tüm uygulamanın üstünü kapattığı için bizim çizeceğimiz uyarı görünmez |
| Araç verisi | **Hiç okunmuyor** | Hesap makinesi araç verisine ihtiyaç duymuyor; §10-§12 bu projede geçersiz |
| İzinler | **Sıfır izin. `INTERNET` bile yok** | Veri güvenliği formu "veri toplanmıyor" ile tek adımda biter (§7.6a) |
| Arayüz teknolojisi | **Klasik XML View'lar** (Compose değil) | Gerçek araçta doğrulanmış yol (EX30 Browser, File Explorer) |
| Hedef cihaz | Tüm AAOS araçları, duyarlı düzen | Dikey ve yatay için ayrı `layout` / `layout-land` |
| Diller | EN (varsayılan) + TR | `values` / `values-tr` |
| SDK | `compileSdk 35`, `minSdk 29`, `targetSdk 35` | prompt.md §1 |
| Modül | Tek modül: `:automotive` | prompt.md §2 |

### Bilerek dışarıda bırakılanlar

- **Canlı döviz kuru.** `INTERNET` izni ⇒ veri güvenliği formu karmaşası + gizlilik politikasında ağ trafiği anlatma yükü + kur yanlışsa şikâyet. Kuru kullanıcı elle girer.
- **Bellek tuşları (M+/MR/MC).** Yerine hesap şeridi; şeritteki satıra dokununca değer ifadeye girer. Araçta hatırlamaya dayanan tuş kötü, görünen liste iyi.
- **Reklam ve analitik.** İzin listesini boş tutmanın tek yolu.
- **Gradle flavor'ları.** İleride Pro sürüm gelirse ayrılabilsin diye modüller bağımsız paketlerde tutulur, ama şimdi flavor kurulmaz.

---

## 2. Çekirdek tasarım fikri: tek klavye, değişen üst alan

Altı modülü altı ayrı ekran yapmak araçta kötü tasarım — sığ gezinme kuralına aykırı ve her modülde farklı giriş yöntemi doğurur.

> **Ekranın alt ~%55'i her zaman aynı büyük sayısal tuş takımı. Üst kısım modüle göre değişir.**
> Hesap modunda üst alan ifade + sonuç + şerit; diğer modüllerde bir form. Bir alana dokunulur, o alan
> seçili olur, aynı tuş takımıyla yazılır, sonuçlar yazdıkça canlı güncellenir.

Kazanç:

1. **Aracın kendi klavyesi hiç açılmaz.** AAOS'ta sistem klavyesi araçtan araca değişiyor. Kendi tuş takımımızla her araçta aynı deneyim ve garanti ≥76dp dokunma hedefi.
2. Kullanıcı hangi modülde olursa olsun sayıyı aynı yerden girer — tek kas hafızası.
3. Kod tarafında tek giriş motoru; her modül yalnızca "alanlar + formül" tanımlar. Yeni modül eklemek bir ekran değil, bir veri sınıfı.

### Alan paylaşımı — ölçülerek belirlendi

Tuş takımı **ağırlıkla değil, ihtiyacı kadar** yer alıyor (`wrap_content`); kalan her şey içeriğe gidiyor. İlk deneme tuş takımına 1,25 ağırlık veriyordu; emülatörde form + sonuçlara yalnızca 465 px kaldı ve sonuçların ilki dışında hepsi katlamanın altında kaldı. Şimdiki hâlde içerik ~600 px alıyor.

**Ana sonuç kaydırılan alanın dışında, sabit bir şeritte duruyor.** Yakıt modülünde 6 alan + 2 seçim şeridi ekranı zaten dolduruyor; kullanıcı yazarken en önemli sayıyı görmek zorunda.

**Seçili alan otomatik görünüre kaydırılıyor.** `=` ile sonraki alana geçildiğinde alan katlamanın altında kalabiliyordu (yatayda ölçüldü). `View.requestRectangleOnScreen` kullanılıyor — `ScrollView.requestChildRectangleOnScreen` doğrudan çocuk beklediği için derindeki alan kutusunda sessizce işe yaramıyor.

### Dikey düzen (EX30 ve benzeri)

```
+----------------------------------------------+
| Hesap  Yakıt  Şarj  Çevirici  Lastik  Kredi  |  ~88dp, yatay kaydırmalı sekme
+----------------------------------------------+
|                              1.250 x 4,7     |  ifade / form etiketleri
|                                  5.875,0     |  SONUÇ - büyük, sağa dayalı
|                                              |
|  --- hesap şeridi (son 20 satır) ---         |  satıra dokun -> değeri geri çağır
+----------------------------------------------+
|    7        8        9        ÷              |
|    4        5        6        ×              |
|    1        2        3        −              |  ağırlıklı ızgara, her tuş >=76dp
|    ±        0        ,        +              |
|    C        <x       ( )      =              |
+----------------------------------------------+
```

### Yatay düzen (GM, Ford, Renault vb.)

Aynı bileşenler iki sütuna ayrılır: solda içerik, sağda tuş takımı. `layout-land` ayrı dosya, aynı kod yolu.

**Yatayda tuş takımı 5 satır × 5 sütun.** 1024×768'de tuş takımına kalan yükseklik ~460 dp; 6 satır kullanılırsa satır başına 68 dp düşüyor ve **≥76 dp dokunma hedefi ihlal oluyor** (emülatörde ölçüldü). 5 satırda satır başına ~84 dp çıkıyor. Aynı 21 tuş, dördüncü sütun "değiştirici" sütunu oluyor (±, ondalık ayracı, 0):

```
C   ⌫   (   %   ÷
7   8   9   ±   ×
4   5   6   ,   −
1   2   3   0   +
        =  (tam genişlik)
```

Seçim `res/values-land/bools.xml` içindeki `keypad_compact` ile yapılıyor — kodda yön kontrolü yok, kaynak çözümlemesi hallediyor.

### Durum kalıcılığı — zorunlu

Sürüş başlayınca sistem uygulamayı kapatır ve **Activity yok edilebilir**. `onPause`'da diske yazılacaklar: geçerli ifade, seçili modül, tüm form alanları, hesap şeridi, birim tercihleri. Açılışta geri yüklenir. Kullanıcı park edip döndüğünde yarım kalan hesabı ekranda bulmalı.

---

## 3. Modüller ve formüller

Ortak arayüz: her modül bir alan listesi + bir hesaplama fonksiyonu.

```kotlin
data class Field(val id: String, val labelRes: Int, val unit: UnitKind, val optional: Boolean = false)
interface CalcModule {
    val fields: List<Field>
    fun compute(v: Map<String, Double>): List<Result>   // etiket + değer + birim
}
```

### 3.1 Hesap
4 işlem, parantez, yüzde, işaret değiştirme. Doğru öncelik kuralı (çarpma/bölme önce). Sonuç `=` ile şerite yazılır.

### 3.2 Yakıt
Girdi: litre · km · birim fiyat · (isteğe bağlı) kişi sayısı

| Çıktı | Formül |
|---|---|
| L/100 km | `litre / km × 100` |
| mpg (US) | `235,215 ÷ (L/100km)` |
| mpg (UK) | `282,481 ÷ (L/100km)` |
| km/L | `km / litre` |
| Dolum tutarı | `litre × birim fiyat` |
| km başı maliyet | `(L/100km × birim fiyat) / 100` |
| Kişi başı | `yol maliyeti / kişi sayısı` |

Elektrikli karşılığı aynı modülde: `kWh/100km = kWh / km × 100`, `km/kWh = km / kWh`.

### 3.3 Şarj (EV)
Girdi: batarya kapasitesi (kWh) · mevcut % · hedef % · şarj gücü (kW) · birim fiyat (₺/kWh) · verim

| Çıktı | Formül |
|---|---|
| Gereken enerji | `kapasite × (hedef − mevcut) / 100` |
| Süre | `enerji ÷ (kW × verim)` |
| Tutar | şebekeden çekilen: `enerji / verim × fiyat` |

Verim varsayılanı **AC 0,90 / DC 0,95**, kullanıcı değiştirebilir.

> **Dürüstlük şartı:** DC hızlı şarjda akım yüksek SoC'ye doğru düşer (taper), gerçek süre uzar.
> Sonuç ekranda **"yaklaşık"** olarak etiketlenecek ve modülde tek satırlık bir açıklama duracak.
> Bunu yazmazsak "yanlış hesaplıyor" yorumu kesin gelir.

### 3.4 Çevirici
Değer + kaynak birim + hedef birim. Kategoriler ve sabitler:

| Kategori | Birimler |
|---|---|
| Mesafe | km, mil, m, ft, yd, in |
| Hacim | L, US gal, UK gal, mL, ft³ |
| **Basınç** | **1 bar = 14,5038 psi = 100 kPa** |
| Sıcaklık | °C, °F, K |
| Ağırlık | kg, lb, ton, US ton |
| Hız | km/h, mph, m/s, knot |
| Tork | Nm, lb·ft, kgf·m |
| **Güç** | **hp (mekanik) = 0,7457 kW** · **PS (metrik) = 0,7355 kW** — ayrı ayrı |
| Yakıt ekonomisi | L/100km ↔ mpg(US) ↔ mpg(UK) ↔ km/L |

hp ile PS'i ayırmak bir araç uygulamasında ciddiyet göstergesi; tek "beygir" birimi koymayacağız.

### 3.5 Lastik ebadı
Girdi: eski ve yeni ebat (genişlik / oran / jant)

| Çıktı | Formül |
|---|---|
| Dış çap | `jant × 25,4 + 2 × (genişlik × oran / 100)` mm |
| Çevre | `π × çap` |
| Çap farkı | `%` ve mm |
| Hız göstergesi sapması | `(yeni çevre / eski çevre − 1) × 100` |
| Gösterge 100 km/h iken gerçek hız | `100 × yeni çevre / eski çevre` |

### 3.6 Kredi
Girdi: tutar · vade (ay) · aylık faiz % · (anahtar) TR vergileri

| Çıktı | Formül |
|---|---|
| Aylık taksit | `P × i ÷ (1 − (1+i)^−n)` |
| Toplam geri ödeme | `taksit × n` |
| Toplam faiz | `toplam − P` |

> **TR notu:** Banka teklifleri KKDF (%15) ve BSMV (%15) içerdiği için düz anüite tutmaz.
> İsteğe bağlı "TR vergileri" anahtarı efektif faizi `i × 1,30` ile hesaplar; kapalıyken saf anüite.
> Sonuç her hâlde "yaklaşık, bankanın teklifi farklı olabilir" diye etiketlenir.

---

## 4. Görsel dil — "Enstrüman"

Koyu gri zemin, tek turkuaz vurgu. Araç göstergesi hissi; EX30'un kendi arayüzüyle çelişmez.

### Palet

| Token | Gece (varsayılan) | Gündüz |
|---|---|---|
| `bg` | `#101315` | `#F2F4F5` |
| `surface` | `#1A1F22` | `#FFFFFF` |
| `key` (tuş yüzeyi) | `#242A2E` | `#E7EBED` |
| `accent` | `#00C4B4` | `#00796B` |
| `accentDim` | `#0A8C81` | `#4DB6AC` |
| `textPrimary` | `#ECEFF1` | `#14191C` |
| `textSecondary` | `#8A9499` | `#5A666C` |
| `danger` (C, geri sil) | `#FF6B5A` | `#D64530` |

Her renk token olarak tanımlanır; `values-night` yoksa gece ekranı göz alır ve inceleme notu olarak dönebilir. Metin/zemin kontrastı en az 4.5:1 tutulacak.

### Tipografi

| Öğe | Boyut |
|---|---|
| Sonuç | 56sp |
| İfade / form etiketi | 28sp |
| Tuş | 34sp |
| Form alan etiketi | 20sp |

Rakamlarda sabit genişlikli rakam (`tnum`) — sonuç değişirken sayılar zıplamaz.

### Etkileşim

- Dokunma hedefi **≥76dp**. Park hâli uygulamasında sistem zorlamıyor ama kol mesafesinden basılıyor.
- Araçlarda titreşim genelde yok ⇒ tuşa basınca **kısa renk/ölçek animasyonu şart**, yoksa kullanıcı bastığını anlamıyor.
- Sığ gezinme: her şey tek ekranda, sekmeyle geçiş. Alt menü içinde alt menü yok.

### Sayı biçimi

- **Ondalık ayracı yerel:** TR virgül, EN nokta. Tuşun üzerindeki işaret yerelle değişir, çözümleyici **ikisini de** kabul eder.
- Binlik ayraç görüntüde var, girdide yok.
- Birim varsayılanı **cihaz bölgesine göre** (TR/AB: L/100km, bar, ₺ · US: mpg, psi, $), ayarlardan tek dokunuşla değiştirilebilir.
- Büyük harf üretirken `uppercase(Locale("tr","TR"))` (prompt.md §7.6b).

---

## 5. Kod yapısı

```
com.oguzhanyucel.carcalc
├── MainActivity.kt
├── ui/
│   ├── KeypadView.kt        # ortak sayısal tuş takımı
│   ├── TapeView.kt          # hesap şeridi, satıra dokun -> geri çağır
│   ├── ModuleTabs.kt
│   └── FormView.kt          # Field listesinden form üretir
├── core/                    # SAF KOTLIN - Android bağımlılığı yok, tamamı test edilir
│   ├── Expression.kt        # tokenizer + shunting-yard + değerlendirici
│   ├── Numbers.kt           # yerel ayraç, biçimleme, ayrıştırma
│   ├── Fuel.kt · Charging.kt · Units.kt · TireSize.kt · Loan.kt
└── data/
    ├── Prefs.kt             # durum kalıcılığı
    └── Tape.kt
```

`buildFeatures { buildConfig = true }` — AGP 8'de varsayılan kapalı (prompt.md §2).
Karşılıklı referans veren property'lere tipi elle yaz (aynı bölüm).

**Hesap katmanı arayüzden tamamen ayrı ve tamamı birim testli.** Ücretsiz bile olsa yanlış hesaplayan bir hesap makinesi affedilmez; bu katman emülatöre hiç girmeden doğrulanır (prompt.md §9'daki en işe yarar alışkanlık).

---

## 6. Yayın planı ve takvim

### Engel: 12 testçi / 14 gün

Geliştirici hesabı 13 Kasım 2023 sonrası açılmış kişisel hesap ⇒ üretime çıkmadan önce **12 test kullanıcısının 14 gün kesintisiz kapalı testte kalması** gerekiyor.

> **Doğrulanacak risk:** Manifest'te `android.hardware.type.automotive` `required="true"` olacak
> (`false` yapmak uygulamayı araçlara hiç dağıtmıyor — prompt.md §7.4a), yani uygulama **telefona
> kurulamaz**. Play'in saydığı şey yalnızca katılım (opt-in) mi, yoksa kurulum da gerekiyor mu —
> bilinmiyor. Play Console kapalı test panelindeki gereksinim göstergesinden bizzat doğrulanacak.

**Sonuç: Play Console tarafı kodla paralel, hatta önce başlar.** 14 günlük sayaç kodun cilası sürerken işlemeli; bu iki haftayı boş geçirmek en pahalı hata olur.

### Fazlar

| Faz | İş | Not |
|---|---|---|
| **0 — hemen** | Play Console: otomotiv form faktörü başvurusu · gizlilik politikası sayfası (GitHub Pages) · kapalı test kanalı + 12 testçi toplama | Form faktörü onayı gelmeden **AAB yükleme ekranına girme** — her deneme bir `versionCode` yakar (§7.4b) |
| 1 | İskelet + `core/` hesap motoru + birim testleri | Emülatörsüz |
| 2 | Arayüz: tuş takımı, sekmeler, şerit; dikey + yatay | İki AVD |
| 3 | Altı modül | Her biri `CalcModule` |
| 4 | Emülatör doğrulaması + Play görselleri | `input tap` ile uçtan uca (§6.10 — şablonsuzda kısıt yok) |
| 5 | Dahili test → **gerçek EX30'da sürüş kilidi doğrulaması** | Emülatörde test edilemiyor (§6.9) |
| 6 | Kapalı test 14 gün → üretim | |

### Play Console kalemleri

| Kalem | Durum |
|---|---|
| Form faktörü kategorisi | Şablonsuz uygulamada **Navigasyon seçilmeyecek**; "Araçlar/Tools" (§7.4a notu) |
| Gizlilik politikası URL'si | İzin olmasa bile **zorunlu**. GitHub Pages'te yayınla, URL'yi bir kez aç ve doğrula |
| Veri güvenliği formu | "Veri toplanmıyor" — izin listesi boş, birleşmiş release manifest'inden doğrula |
| Tüccar (trader) beyanı | Para kazanılmadığı için "tüccar değil"; adres herkese açık gösterilmez — Console'dan teyit et |
| Ekran görüntüleri | 2 × 800×1280 (`EX30_Portrait`) + 2 × 1024×768 (`EX30_Landscape`) |
| Simge / tanıtım | 512×512 + 1024×500, **bu projeye özel yeniden üretilecek** (§7.6b) |
| Launcher simgesi | `ic_launcher_foreground.xml` değiştirilecek — iskelet kopyalanırsa önceki uygulamanın simgesi geliyor |
| Mağaza metni | EN + TR; ad ≤30, kısa açıklama ≤80, tam açıklama ≤4000 |
| İmzalama | Mevcut `ex30-upload.jks` / alias `ex30-upload` kullanılabilir (§7.1) |

**Mağaza metninde açıkça yazılacak:** uygulama **park hâlinde** kullanılmak üzere tasarlandı, araç hareket ederken sistem tarafından kapatılır. Yazılmazsa ilk düşük puanlı yorum bu olur.

---

## 7. Doğrulama planı

| Katman | Yöntem |
|---|---|
| Hesap motoru | Saf Kotlin birim testleri — emülatörsüz, her formül için |
| Arayüz akışı | Emülatörde uçtan uca `adb shell input tap` — şablonsuz uygulamada hiçbir kısıt yok (§6.10) |
| Düzen | İki AVD: `EX30_Portrait` 800×1280 + `EX30_Landscape` 1024×768 (§6.1) |
| Gece/gündüz | Her iki AVD'de iki temada ekran görüntüsü |
| Durum kalıcılığı | `am force-stop` sonrası yeniden açıp ifadenin döndüğünü gör |
| **Sürüş kilidi** | **Yalnızca gerçek EX30'da.** Emülatörde `car_service` tamamen kapalı (§6.9) |

Ekran görüntüsü alırken: `adb shell screencap -p -d <id> /sdcard/shot.png` + `adb pull` — PowerShell'de `>` yönlendirmesi PNG'yi bozuyor (§6.5).

---

## 8. Açık riskler

| Risk | Etki | Ne yapacağız |
|---|---|---|
| 12 testçi otomotiv-only uygulamayı kuramayabilir | Üretime çıkış tıkanır | Faz 0'da kapalı test kanalını aç, gereksinim göstergesini izle |
| Otomotiv form faktörü incelemesi üretimde daha sıkı | Gecikme | Faz 0'da başvur, koddan önce |
| Sürüş kilidi emülatörde doğrulanamıyor | Yanlış davranış araçta ortaya çıkar | Faz 5'te gerçek araçta zorunlu doğrulama |
| DC şarj süresi tahmininin gerçekten sapması | Kötü yorum | Sonucu "yaklaşık" etiketle, sebebini uygulamada yaz |
| Kredi taksitinin banka teklifiyle tutmaması | Kötü yorum | TR vergileri anahtarı + "yaklaşık" etiketi |
