# Vakit — Ezan Vakti Android Uygulaması

"Vakit" ezan vakti web uygulamasının (`index.html`) Capacitor 7 ile Android'e dönüştürülmüş hali. Tüm UI, animasyon, ses ve fonksiyonellik birebir korunmuştur; web bağımlılıkları (Tailwind, fontlar) offline olarak gömülmüştür.

## Hızlı Başlangıç

### APK'yı cihaza yüklemek
Debug APK doğrudan bir Android telefona yüklenebilir:

```
android/app/build/outputs/apk/debug/app-debug.apk
```

Kurulum:
```bash
adb install android/app/build/outputs/apk/debug/app-debug.apk
```
ya da APK dosyasını telefona kopyalayip "Bilinmeyen kaynaklardan yükle" ile açın.

### Sıfırdan build almak

**Gereksinimler:**
- Node.js 20+ (test: v22.16)
- JDK 21 (Capacitor 7 şartı — Android Studio'nun JBR'i kullanılabilir: `C:\Program Files\Android\Android Studio\jbr`)
- Android SDK API 35 + Build-Tools 35
- `ANDROID_HOME` ortam değişkeni SDK dizinine ayarlı

**Komutlar:**
```bash
# 1. Bağımlılıkları kur (ilk seferlik)
npm install

# 2. Web asset'lerini Android'e senkronize et
npx cap sync android

# 3. Debug APK üret (JAVA_HOME'u JDK 21'e ayarlayın!)
cd android
export JAVA_HOME="C:/Program Files/Android/Android Studio/jbr"   # Windows örneği
./gradlew assembleDebug
```

Çıktı: `android/app/build/outputs/apk/debug/app-debug.apk`

### Geliştirme döngüsü (web tarafında değişiklik yapınca)

1. `public/index.html`'i düzenleyin (kaynak dosya `index.html` ile aynı içerik, sadece asset yolları offline).
2. `npx cap sync android` — değişiklikleri Android'e kopyalar.
3. `cd android && ./gradlew installDebug` — cihaza/emülatöre kurar.

Veya Android Studio'da açın: `npx cap open android`.

## Mimari

```
Ezan/
├── index.html                  # Orijinal web kaynağı (korunur, referans)
├── public/                     # Capacitor webDir — WebView'e yüklenen kısım
│   ├── index.html              # Web kaynağının offline asset yollarıyla kopyası
│   ├── js/tailwind.js          # Tailwind CDN'nin lokal kopyası (407 KB)
│   ├── css/fonts.css           # @font-face tanımları
│   └── fonts/*.woff2           # Inter + JetBrains Mono (latin + latin-ext, Türkçe karakterler dahil)
├── capacitor.config.json       # appId: com.ezan.vakit, appName: Vakit
├── package.json                # Capacitor 7 bağımlılıkları
└── android/                    # Capacitor-generated native Android projesi
    ├── app/src/main/AndroidManifest.xml   # INTERNET + ACCESS_NETWORK_STATE izinleri
    ├── app/src/main/res/xml/network_security_config.xml
    └── variables.gradle         # minSdk=23, targetSdk=35
```

## Uygulama Bilgileri

| Alan | Değer |
|------|-------|
| Package ID | `com.ezan.vakit` |
| Uygulama adı | Vakit |
| minSdk | 23 (Android 6.0) |
| targetSdk | 35 (Android 15) |
| APK boyutu | ~5.1 MB (debug) |
| Versiyon | 1.0 |

## Özellikler (hepsi web versiyonuyla birebir)

- 6 vakit kartı + canlı geri sayım sayacı
- Vakte göre dinamik arka plan gradyan teması
- Kıble pusulası + cihaz manyetometre sensörü (DeviceOrientation)
- Aylık imsakiye tablosu
- Aladhan API'den canlı veri (HTTPS) + localStorage önbellekleme (12 saat)
- Web Audio API ile sentezlenen ezan gong sesi
- 81 Türk il + global şehir arama
- **Kalıcı tercihler** (şehir, ses, hesaplama metodu, ofset) — Capacitor Preferences (Android SharedPreferences) üzerinde; uygulama verisi silinmedikçe seçili şehir asla sıfırlanmaz

## Kalıcı Depolama

Kullanıcı tercihleri (seçili şehir, ses açık/kapalı, hesaplama metodu, vakit ofsetleri) WebView `localStorage` yerine **Capacitor Preferences** plugin'iyle native Android **SharedPreferences**'a yazılır. Bu, seçtiğiniz şehrin:

- Uygulama kapatılıp açıldığında
- Telefon yeniden başlatıldığında
- WebView belleği temizlense bile

korunmasını sağlar. Sadece "Uygulama verilerini temizle" / "Clear data" hepsini siler.

**İlk açılış davranışı:** `loadPreferences()` async olarak çalışır — önce Preferences'tan okur, boşsa (ilk kurulum/upgrade) eski localStorage'dan otomatik taşır (migration), sonra UI'yı yükler. Bu nedenle açılışta kayıtlı şehir anında header'da görünür.

**Mimari:**
- `PrefStore` shim'i: native bridge varsa (`window.Capacitor.registerPlugin`) Preferences API kullanır, yoksa (browser önizleme) localStorage'a düşer
- `migrateLegacyLocalStorage()`: tek seferlik — eski sürümden kalma localStorage tercihlerini Preferences'a kopyalar
- API önbelleği (12 saatlik vakit verisi) hâlâ localStorage'da — geçici olduğu için SharedPreferences gerekmez

## Veri Akışı

```
Uygulama → WebView → fetch() → https://api.aladhan.com → JSON → render
                      ↓
                 localStorage (12 saat önbellek)
```

İzinler: `INTERNET` (API çağrısı), `ACCESS_NETWORK_STATE` (bağlantı durumu). Push bildirim, arka plan servisi veya konum (GPS) izni **yok** — ilk sürüm bunları içermiyor.

## Notlar

- **JDK 21 zorunlu**: Capacitor 7.6.8 kaynak seviyesi 21 gerektirir. JDK 17 ile build `error: invalid source release: 21` hatası verir. Android Studio'nun bundled JBR'i JDK 21 içerir.
- **Fontlar offline**: Tailwind ve fontlar `public/` altına gömüldüğü için uygulama UI için internet gerektirmez; sadece ezan vakti verileri için API çağrısı yapar.
- **Canlı pusula**: `DeviceOrientationEvent` Android WebView (Chromium) tarafından desteklenir; bazı cihazlarda manyetometre kalibrasyonu gerekebilir.
