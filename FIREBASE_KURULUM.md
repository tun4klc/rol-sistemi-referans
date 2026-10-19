# Kendi Firebase Projeni Bağlama Rehberi

Bu rehber, başka birinin Firebase projesini (ör. ServisPort'un `servismobile-e4f1d`
projesi) kullanmadan **kendi Firebase projeni** oluşturup Flutter uygulamana bağlamayı
anlatır. Spark (ücretsiz) plan yeterlidir.

---

## Adım 1 — Firebase projesi oluştur
1. https://console.firebase.google.com adresine Google hesabınla gir.
2. **Proje ekle** → bir isim ver (ör. `benim-servis-app`) → Google Analytics isteğe bağlı.
3. Oluşan **Proje ID**'sini not et (ör. `benim-servis-app-1a2b3`).

## Adım 2 — Authentication'ı aç
1. Sol menü → **Build → Authentication → Başlayın**.
2. **Sign-in method** sekmesi → **E-posta/Şifre** → Etkinleştir → Kaydet.

## Adım 3 — Firestore'u aç
1. Sol menü → **Build → Firestore Database → Veritabanı oluştur**.
2. Konum: `eur3 (europe-west)` (Türkiye'ye yakın). **Sonradan değiştirilemez.**
3. **Production mode** seç (kuralları birazdan yükleyeceğiz).

## Adım 4 — Bilgisayara araçları kur (bir kerelik)
Node.js gerekir (https://nodejs.org → LTS sürüm).

```bash
npm install -g firebase-tools        # Firebase CLI
firebase login                       # tarayıcıda Google hesabınla giriş
dart pub global activate flutterfire_cli
```

`flutterfire` komutu bulunamazsa PATH'e ekle:
```bash
echo 'export PATH="$PATH:$HOME/.pub-cache/bin"' >> ~/.zshrc && source ~/.zshrc
```

## Adım 5 — Uygulamayı kendi projene bağla
Flutter projenin kök klasöründe (`pubspec.yaml`'ın olduğu yer):

```bash
flutterfire configure --project=benim-servis-app-1a2b3
```
- Platformları seç (iOS, Android, ...).
- Bu komut şunları **otomatik oluşturur/üzerine yazar**:
  - `lib/firebase_options.dart`
  - `ios/Runner/GoogleService-Info.plist`
  - `android/app/google-services.json`

`main.dart` içinde şu olmalı (ServisPort'ta zaten var):
```dart
await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
```

> **Başka birinin projesinden geliyorsan:** projedeki `.firebaserc` dosyasında eski
> proje ID'si yazar. `firebase use --add` ile kendi projeni seç veya dosyayı silip
> Adım 6'da yeniden oluştur. Eski `google-services.json` / `GoogleService-Info.plist`
> dosyaları varsa `flutterfire configure` bunları değiştirir.

> **Bundle ID / Paket adı:** `com.example...` ile yayına çıkılamaz. Yayın düşünüyorsan
> önce kendi ID'ni ver (ör. `com.adin.servisapp`), sonra `flutterfire configure`'ı çalıştır.

## Adım 6 — Güvenlik kurallarını yükle
```bash
firebase init firestore      # projeni seç; rules dosyası olarak firestore.rules kabul et
```
Bu klasördeki `firestore.rules` dosyasını projenin köküne kopyala (oluşanın üzerine), sonra:
```bash
firebase deploy --only firestore:rules
```

> Kuralların koleksiyon isimleri (`users`, `services`, `notifications`) uygulamadaki
> isimlerle **aynı** olmalı. ServisPort'un orijinal kodu `kullanıcılar`, `servisler`,
> `bildirimler` kullanıyor; ya kodu ya da kuralları uyarla (Claude'dan iste).

## Adım 7 — İlk admin kullanıcısını oluştur
Kayıt olan herkes `teknisyen` ve şirketsiz başlar; ilk admini elle yapman gerekir.

1. Uygulamadan normal şekilde **kayıt ol**.
2. Firebase Console → **Firestore → users** → kendi belgeni (UID ile) aç.
3. Alanları düzenle:
   - `role` → `admin`
   - `companyId` → `firma_1` (istediğin bir şirket kimliği; metin)
4. Uygulamada çıkış yapıp tekrar gir. Artık adminsin; diğer kullanıcıları yönetim
   panelinden şirketine ekleyebilirsin.

(Console'dan yapılan değişiklikler güvenlik kurallarından etkilenmez.)

## Adım 8 — Çalıştır ve test et
```bash
flutter pub get
flutter run
```
Test önerisi: ikinci bir hesap aç (teknisyen), adminle şirkete ekle, sonra teknisyen
hesabıyla yöneticiye ait ekranlara erişememeyi ve başkasının servisini görememeyi dene.

---

## Sık karşılaşılan hatalar

| Hata | Sebep / Çözüm |
|---|---|
| `[cloud_firestore/permission-denied]` | Kurallar isteği reddetti. Sorgu kuralla uyumlu mu? (Teknisyen sorgusunda `technicianId` filtresi var mı?) Kullanıcının `companyId`'si atanmış mı? |
| `The query requires an index` | Hata mesajındaki bağlantıya tıkla → indeks otomatik oluşur (birkaç dakika sürer). |
| `No Firebase App '[DEFAULT]' has been created` | `Firebase.initializeApp` çağrılmamış ya da `firebase_options.dart` eksik → `flutterfire configure`. |
| iOS'ta `pod install` hatası | `cd ios && pod repo update && pod install` |
| `flutterfire: command not found` | Adım 4'teki PATH satırı. |
| Kayıt oluyor ama hiçbir veri gelmiyor | Normal: kullanıcı şirkete atanmadı. Admin atamalı (Adım 7). |

## Ücretlendirme notu
Spark (ücretsiz) planda günlük 50.000 okuma / 20.000 yazma ücretsizdir; küçük ve orta
uygulamalar için fazlasıyla yeterli. Kurallardaki `get()` çağrıları da okuma sayılır.
Cloud Functions / Custom Claims kullanmak istersen Blaze (kullandıkça öde) plan gerekir.
