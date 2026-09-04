# CarCalc

Android Automotive OS için hesap makinesi ve araç hesapları. Ücretsiz, reklamsız, **izinsiz**.

Tasarım kararları: [TASARIM.md](TASARIM.md) · AAOS genel rehberi: `../prompt.md`

---

## Ne yapıyor

| Modül | İçerik |
|---|---|
| **Hesap** | 4 işlem, parantez, yüzde, işaret; hesap şeridi (satıra dokun → değeri geri çağır) |
| **Yakıt** | L/100km · mpg (US/UK) · km/L · kWh/100km · km/kWh · dolum tutarı · km başı maliyet · kişi başı |
| **Şarj** | Gereken enerji, şebekeden çekilen enerji, süre, tutar, efektif birim maliyet |
| **Çevirici** | Mesafe, hacim, basınç, sıcaklık, ağırlık, hız, tork, güç (hp ve PS ayrı), yakıt/elektrik tüketimi |
| **Lastik** | Dış çap, çevre, hız göstergesi sapması, kilometre sapması |
| **Kredi** | Anüite taksiti, toplam ödeme, toplam faiz, TR vergileri (KKDF+BSMV) anahtarı |

Ayarlar: birim sistemi, tema (sistem/koyu/açık), dil (sistem/TR/EN), para birimi.

## Mimari

```
core/    Saf Kotlin, Android bağımlılığı YOK, tamamı birim testli
         Expression · Editor · Numbers · Units · Fuel · Charging · Converter · TireSize · Loan
ui/      Klasik View'lar, tek ortak tuş takımı
         KeypadView · CalcPane · FormPane · SettingsPane · Fmt · Ui
data/    Prefs (durum kalıcılığı)
```

Bir modül bir ekran değil, **bir alan listesi + bir formül** (`CalcModule`). Ortak form motoru alanları çizer, ortak tuş takımı doldurur. Yeni modül eklemek yeni bir Activity değil, bu arayüzün yeni bir uygulaması.

**Harici bağımlılık yok** — ne AndroidX ne Material. Sonucu: hızlı derleme, boş izin listesi, tek adımda biten veri güvenliği formu.

> **`isMinifyEnabled` kapalı kalmalı.** Etiketler `Resources.getIdentifier` ile çözümleniyor (`Fmt.text`); kod/kaynak küçültme açılırsa bu anahtarlar için keep kuralı gerekir.

## Neden şablonsuz Activity

Car App Library şablonları tuş takımı veremiyor ve tek serbest çizim yolu olan Surface'e gelen dokunmalar uygulamaya **hiç** iletilmiyor (`prompt.md` §5.4). Dokunulamayan bir hesap makinesi olmaz.

## Neden `distractionOptimized` yok

Hesap makinesinin sürerken çalışmasına gerek yok. Meta-veri yazılmadığı için araç hareket edince sistem uygulamanın üstünü kendi kilit ekranıyla kapatıyor ve Activity **yok edilebiliyor** — bu yüzden ifade, seçili sekme, tüm form alanları ve şerit `onPause`'da diske yazılıyor.

Bu davranış **emülatörde doğrulanamıyor** (`car_service` tamamen kapalı, `prompt.md` §6.9); yayın öncesi gerçek araçta test edilmesi zorunlu.

---

## Geliştirme

`java`, `gradle` ve `adb` PATH'te değil:

```bash
export JAVA_HOME="C:/Program Files/Android/Android Studio/jbr"
```

Birim testleri (emülatör gerekmez):

```bash
./gradlew :automotive:testDebugUnitTest
```

Debug APK:

```bash
./gradlew :automotive:assembleDebug
```

Emülatöre kurulum — AAOS'ta sürücü profili **user 10**:

```bash
adb install -r -t automotive/build/outputs/apk/debug/automotive-debug.apk
```

```bash
adb shell am start --user 10 -n com.oguzhanyucel.carcalc/.MainActivity
```

Ekran görüntüsü (PowerShell'de `>` yönlendirmesi PNG'yi bozuyor, cihaza yazıp çek):

```bash
adb shell screencap -p -d 4619827259835644672 /sdcard/s.png && adb pull /sdcard/s.png
```

İki AVD gerekiyor: `EX30_Portrait` (800×1280) ve `EX30_Landscape` (1024×768).

## Yayın öncesi eksikler

- `keystore.properties` (depoda yok, `.gitignore`'da) — `ex30-upload.jks` kullanılacak
- Play mağaza görselleri: 512×512 simge, 1024×500 tanıtım, 2 dikey + 2 yatay ekran görüntüsü
- `privacy/index.html` + GitHub Pages yayını
- `PLAY-CONSOLE.md` — mağaza metinleri (EN + TR)
- Otomotiv form faktörü başvurusu ve 12 testçi / 14 gün kapalı test

Ayrıntı: [TASARIM.md](TASARIM.md) §6.
