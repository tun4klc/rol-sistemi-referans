# Rol Tabanlı Yetkilendirme (RBAC) Referansı — Flutter + Firebase

> **Claude için not:** Bu dosya, ServisPort uygulamasındaki rol sisteminin belgesi ve
> başka projelerde kullanılacak **önerilen şablondur**. Bir projeye uygularken:
> 1. Önce projedeki mevcut auth/rol kodunu incele, bu şablonla nasıl birleşeceğini planla.
> 2. Rol isimlerini, koleksiyon isimlerini ve yetkileri kullanıcının isteğine göre uyarla.
> 3. Her UI kısıtlamasının **firestore.rules içinde de** karşılığı olduğundan emin ol.
> 4. Bölüm 9'daki kontrol listesiyle bitir.

**Teknoloji yığını:** Flutter (Dart 3), Firebase Auth (e-posta/şifre), Cloud Firestore,
Riverpod 3 (`flutter_riverpod`), GoRouter.

---

## 1. Genel Bakış

Sistem üç katmandan oluşur. Üçü de olmalıdır; sadece biri yetmez.

| Katman | Görevi | Güvenlik değeri |
|---|---|---|
| **1. Veri** (Firestore `users` belgesi) | Kullanıcının rolünü ve şirketini saklar | Tek doğru kaynak |
| **2. Uygulama** (Riverpod + GoRouter + widget'lar) | Menüleri, sayfaları, butonları role göre gösterir/gizler | **Sadece kullanıcı deneyimi.** Atlatılabilir. |
| **3. Sunucu** (`firestore.rules`) | Her okuma/yazmayı role ve şirkete göre onaylar/reddeder | **Gerçek güvenlik** |

> ⚠️ En önemli kural: Uygulama kodu istemcide çalışır; kullanıcı onu değiştirebilir veya
> Firestore API'sini doğrudan çağırabilir. "Teknisyen sadece kendi işlerini görür" gibi
> her kısıt **mutlaka** `firestore.rules` içinde de yazılmalıdır.

---

## 2. Roller ve Yetki Matrisi

| Rol (Firestore değeri) | Açıklama |
|---|---|
| *(giriş yok)* — `guest` | Sadece giriş/kayıt ekranları |
| `teknisyen` | Sahada çalışan personel. Sadece kendine atanan işler. **Yeni kayıtların varsayılanı.** |
| `mudur` | Ekip yöneticisi. Şirketteki tüm işleri görür, siler, personeli yönetir. |
| `admin` | Şirket sahibi/yönetici. Müdürün her şeyi + kullanıcı rolü atama/silme + yönetim paneli. |

| Yetki (`Permission`) | teknisyen | mudur | admin |
|---|:-:|:-:|:-:|
| Kendi servislerini görme/güncelleme | ✅ | ✅ | ✅ |
| `viewAllServices` — Şirketin tüm servisleri | ❌ | ✅ | ✅ |
| `deleteService` — Servis silme | ❌ | ✅ | ✅ |
| `viewAllNotifications` — Tüm bildirimler | ❌ | ✅ | ✅ |
| `managePersonnel` — Personel sayfası | ❌ | ✅ | ✅ |
| `adminPanel` — Yönetim paneli | ❌ | ❌ | ✅ |
| `manageUsers` — Rol atama, kullanıcı silme | ❌ | ❌ | ✅ |

**Çoklu şirket (multi-tenancy):** Her kullanıcının bir `companyId`'si vardır. Hiçbir rol,
kendi şirketi dışındaki veriyi göremez. `companyId` atanmamış kullanıcı hiçbir şirket
verisine erişemez ve "onay bekleniyor" ekranında kalır.

---

## 3. Veri Modeli

### `users/{uid}` (belge ID'si = Firebase Auth UID)
```json
{
  "uid": "abc123",
  "name": "Ali Yılmaz",
  "email": "ali@ornek.com",
  "phone": "05xx...",
  "role": "teknisyen",          // "teknisyen" | "mudur" | "admin"
  "companyId": null,            // kayıtta null; admin atar
  "createdAt": "<serverTimestamp>"
}
```

### `services/{id}`
```json
{ "companyId": "firma_1", "technicianId": "abc123", "status": "pending", "...": "..." }
```

### `notifications/{id}`
```json
{ "companyId": "firma_1", "recipientId": "abc123", "...": "..." }
// Herkese giden bildirim için recipientId: "all"  (null KULLANMA — sorgulanamıyor)
```

### ServisPort'taki orijinal isimler (eşleme)
Orijinal uygulama Türkçe isimler kullanıyor. Yeni projelerde **ASCII İngilizce isimler**
önerilir (URL kodlama sorunları çıkmaz: `kullan%C4%B1c%C4%B1lar` gibi).

| Orijinal (ServisPort) | Şablon |
|---|---|
| `kullanıcılar` koleksiyonu | `users` |
| `rol` | `role` |
| `şirket_id` | `companyId` |
| `ad`, `telefon`, `kayıt_tarihi` | `name`, `phone`, `createdAt` |
| `servisler` | `services` |
| `bildirimler` | `notifications` |
| `recipientId: null` (herkese) | `recipientId: "all"` |
| `UserRole { guest, teknisyen, mudur, admin }` | `AppRole` + `Permission` |

---

## 4. Akışlar

### Kayıt
1. Firebase Auth ile hesap oluşturulur.
2. `users/{uid}` belgesi `role: "teknisyen"`, `companyId: null` ile yazılır.
   Kurallar başka bir rol veya şirket yazılmasına **izin vermez**.
3. Kullanıcı "onay bekleniyor" ekranını görür.
4. Admin, yönetim panelinden kullanıcıyı kendi şirketine ekler ve rolünü belirler.

### Giriş ve oturum
1. `FirebaseAuth.authStateChanges()` dinlenir → kullanıcı varsa `users/{uid}` belgesi
   **canlı** (`snapshots()`) dinlenir → `AppUser` oluşur.
2. Bu sayede uygulama yeniden açıldığında oturum ve rol korunur, admin rolü
   değiştirdiğinde ekran anında güncellenir.
3. GoRouter, `AppUser` değiştikçe `redirect`'i yeniden çalıştırır.

### İlk admin
İlk admin elle oluşturulur (Firebase Console'da `users/{uid}` belgesinde
`role: "admin"`, `companyId: "firma_1"` yazılır). Ayrıntı: `FIREBASE_KURULUM.md`.

---

## 5. Kod Şablonları

Önerilen dosya düzeni:
```
lib/core/auth/
  app_role.dart        # AppRole, Permission, yetki matrisi
  app_user.dart        # AppUser modeli + provider'lar
  auth_service.dart    # signIn / signUp / signOut
  role_gate.dart       # yetkiye göre widget gösterme
lib/core/routes/app_router.dart
```

### 5.1 `app_role.dart`
```dart
enum AppRole {
  teknisyen,
  mudur,
  admin;

  /// Firestore'daki metni role çevirir. Bilinmeyen/eksik değer en düşük yetki olur.
  static AppRole fromString(String? value) {
    switch (value?.toLowerCase().trim()) {
      case 'admin':
        return AppRole.admin;
      case 'mudur':
      case 'müdür':
        return AppRole.mudur;
      default:
        return AppRole.teknisyen;
    }
  }

  /// Firestore'a yazılacak değer.
  String get value => name;

  bool can(Permission permission) =>
      rolePermissions[this]!.contains(permission);
}

enum Permission {
  viewAllServices,
  deleteService,
  viewAllNotifications,
  managePersonnel,
  adminPanel,
  manageUsers,
}

/// Tek yetki matrisi. Yeni rol/yetki eklerken sadece burayı (ve kuralları) değiştir.
final Map<AppRole, Set<Permission>> rolePermissions = {
  AppRole.teknisyen: {},
  AppRole.mudur: {
    Permission.viewAllServices,
    Permission.deleteService,
    Permission.viewAllNotifications,
    Permission.managePersonnel,
  },
  AppRole.admin: Permission.values.toSet(),
};
```

> Neden `Permission`? Kodda `role == admin || role == mudur` yazmak yerine
> `role.can(Permission.deleteService)` yazılır. Yeni rol eklendiğinde onlarca `if`'i
> aramak gerekmez; matrisi güncellemek yeter. (ServisPort'ta bu kontrol ~15 yerde
> elle tekrarlanıyor.)

### 5.2 `app_user.dart`
```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'app_role.dart';

class AppUser {
  const AppUser({
    required this.uid,
    required this.name,
    required this.role,
    required this.companyId,
  });

  final String uid;
  final String name;
  final AppRole role;
  final String? companyId;

  bool get hasCompany => companyId != null;

  factory AppUser.fromDoc(String uid, Map<String, dynamic> data) => AppUser(
        uid: uid,
        name: data['name'] as String? ?? '',
        role: AppRole.fromString(data['role'] as String?),
        companyId: data['companyId'] as String?,
      );
}

/// Firebase oturumu (uygulama yeniden açılınca da korunur).
final firebaseUserProvider = StreamProvider<User?>(
  (ref) => FirebaseAuth.instance.authStateChanges(),
);

/// Oturumdaki kullanıcının profili; rol değişince canlı güncellenir.
/// null = giriş yapılmamış (guest).
final appUserProvider = StreamProvider<AppUser?>((ref) {
  final user = ref.watch(firebaseUserProvider).value;
  if (user == null) return Stream.value(null);

  return FirebaseFirestore.instance
      .collection('users')
      .doc(user.uid)
      .snapshots()
      .map((doc) {
    final data = doc.data();
    return data == null ? null : AppUser.fromDoc(user.uid, data);
  });
});
```

### 5.3 `auth_service.dart` (kayıt kısmı)
```dart
Future<void> signUp({
  required String email,
  required String password,
  required String name,
  required String phone,
}) async {
  final cred = await FirebaseAuth.instance
      .createUserWithEmailAndPassword(email: email, password: password);
  await cred.user!.updateDisplayName(name);

  // Rol ve şirket istemciden SEÇİLMEZ. Kurallar da sadece bu değerlere izin verir.
  await FirebaseFirestore.instance.collection('users').doc(cred.user!.uid).set({
    'uid': cred.user!.uid,
    'name': name,
    'email': email,
    'phone': phone,
    'role': AppRole.teknisyen.value,
    'companyId': null,
    'createdAt': FieldValue.serverTimestamp(),
  });
}
```

### 5.4 `app_router.dart` — sayfa koruması
```dart
import 'package:flutter/foundation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../auth/app_role.dart';
import '../auth/app_user.dart';

/// Hangi yol hangi yetkiyi ister. Listede olmayan yollar giriş yapan herkese açık.
const Map<String, Permission> routePermissions = {
  '/admin': Permission.adminPanel,
  '/personnel': Permission.managePersonnel,
};

const _publicRoutes = {'/login', '/register'};

String homeFor(AppRole role) => role.can(Permission.adminPanel) ? '/admin' : '/home';

final goRouterProvider = Provider<GoRouter>((ref) {
  // GoRouter'ı bir kez oluştur; kullanıcı değişince sadece redirect yeniden çalışsın.
  final refresh = ValueNotifier<int>(0);
  ref.listen(appUserProvider, (_, _) => refresh.value++);
  ref.onDispose(refresh.dispose);

  return GoRouter(
    initialLocation: '/splash',
    refreshListenable: refresh,
    redirect: (context, state) {
      final loc = state.matchedLocation;
      final userAsync = ref.read(appUserProvider);

      // Oturum henüz yükleniyor
      if (userAsync.isLoading) return loc == '/splash' ? null : '/splash';

      final user = userAsync.value;

      // Giriş yapılmamış
      if (user == null) return _publicRoutes.contains(loc) ? null : '/login';

      // Şirkete henüz atanmamış
      if (!user.hasCompany) return loc == '/pending' ? null : '/pending';

      // Giriş yapmış kullanıcı giriş/bekleme ekranlarında kalmasın
      if (_publicRoutes.contains(loc) || loc == '/splash' || loc == '/pending') {
        return homeFor(user.role);
      }

      // Yetki kontrolü
      for (final entry in routePermissions.entries) {
        if (loc.startsWith(entry.key) && !user.role.can(entry.value)) {
          return homeFor(user.role);
        }
      }
      return null;
    },
    routes: [
      // GoRoute(path: '/splash', ...), '/login', '/register', '/pending', '/home', '/admin', ...
    ],
  );
});
```

### 5.5 `role_gate.dart` — buton/bölüm gizleme
```dart
import 'package:flutter/widgets.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'app_role.dart';
import 'app_user.dart';

class RoleGate extends ConsumerWidget {
  const RoleGate({
    super.key,
    required this.permission,
    required this.child,
    this.fallback = const SizedBox.shrink(),
  });

  final Permission permission;
  final Widget child;
  final Widget fallback;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final user = ref.watch(appUserProvider).value;
    return (user != null && user.role.can(permission)) ? child : fallback;
  }
}

// Kullanım:
// RoleGate(permission: Permission.deleteService, child: IconButton(...))
```

Alt menü (ServisPort'taki `editorial_bottom_nav.dart` mantığı):
```dart
final items = [
  (label: 'Panel',     route: '/home',      permission: null),
  (label: 'Servisler', route: '/services',  permission: null),
  (label: 'Personel',  route: '/personnel', permission: Permission.managePersonnel),
  (label: 'Yönetim',   route: '/admin',     permission: Permission.adminPanel),
].where((i) => i.permission == null || user.role.can(i.permission!)).toList();
```

### 5.6 Sorgular — kurallarla uyumlu olmalı
Firestore kuralları **filtre değildir**: teknisyen tüm servisleri isteyip "kurallar
kendi kayıtlarımı süzsün" diyemez — sorgu tamamen reddedilir. Sorgu, kuralın izin
verdiği kümeyle birebir sınırlanmalıdır:

```dart
Query<Map<String, dynamic>> servicesQuery(AppUser user) {
  Query<Map<String, dynamic>> q = FirebaseFirestore.instance
      .collection('services')
      .where('companyId', isEqualTo: user.companyId);
  if (!user.role.can(Permission.viewAllServices)) {
    q = q.where('technicianId', isEqualTo: user.uid);
  }
  return q;
}

Query<Map<String, dynamic>> notificationsQuery(AppUser user) {
  Query<Map<String, dynamic>> q = FirebaseFirestore.instance
      .collection('notifications')
      .where('companyId', isEqualTo: user.companyId);
  if (!user.role.can(Permission.viewAllNotifications)) {
    q = q.where('recipientId', whereIn: [user.uid, 'all']);
  }
  return q;
}
```
Birden fazla `where` + `orderBy` bileşik indeks ister. İlk çalıştırmada hata mesajındaki
bağlantıya tıklayıp indeksi oluştur.

### 5.7 Admin: kullanıcıya rol/şirket atama
```dart
Future<void> assignUser(AppUser admin, String targetUid, AppRole role) {
  return FirebaseFirestore.instance.collection('users').doc(targetUid).update({
    'role': role.value,
    'companyId': admin.companyId,
  });
}

// Şirketi olmayan (onay bekleyen) kullanıcılar:
// FirebaseFirestore.instance.collection('users').where('companyId', isNull: true)
```

---

## 6. Firestore Güvenlik Kuralları

Tam ve kullanıma hazır dosya: **`firestore.rules`** (bu klasörde). Özet mantık:

| Koleksiyon | Okuma | Oluşturma | Güncelleme | Silme |
|---|---|---|---|---|
| `users` | Kendisi; aynı şirket; admin → şirketsiz kullanıcılar | Sadece kendisi, `role=teknisyen`, `companyId=null` | Kendisi (rol/şirket hariç); admin → kendi şirketi veya şirketsiz kullanıcı | Admin, kendi şirketi, kendisi hariç |
| `services` | Aynı şirket **ve** (müdür/admin **veya** `technicianId == uid`) | Aynı şirket ve (yönetici veya kendine atama) | Aynı şirket, `companyId` değişmez, yönetici veya kendi kaydı | Müdür/admin, aynı şirket |
| `notifications` | Aynı şirket ve (yönetici veya `recipientId` kendisi/`all`) | Aynı şirket | Aynı şirket ve (yönetici veya kendine ait) | Müdür/admin, aynı şirket |

Kuralları test etmek için:
```bash
firebase emulators:start --only auth,firestore
```
Claude'dan `@firebase/rules-unit-testing` ile test yazmasını isteyebilirsin.

---

## 7. ServisPort'taki Orijinal Uygulamada Bulunan Açıklar (tekrarlama!)

Şablon bunları zaten düzeltiyor; eski koda dokunurken bunlara dikkat et.

1. **Kısıtlar sadece istemcideydi.** Teknisyen filtresi (`jobs_list_tab.dart`) ve bildirim
   filtresi (`notifications_screen.dart`) Dart kodunda yapılıyordu; kurallar şirketteki
   herkesin her şeyi okumasına izin veriyordu.
2. **Kayıtta rol seçilebiliyordu.** `users` create kuralı `rol` alanını kısıtlamıyordu;
   API ile doğrudan `rol: "admin"` yazılabilirdi.
3. **Varsayılan şirket.** `şirket_id` kayıtta yazılmıyor, her yerde
   `'default_company_123'`'e düşülüyordu → tüm yeni kullanıcılar aynı "şirkette" buluşuyordu.
4. **Rol sadece bellekteydi.** `Notifier<UserRole>` uygulama kapanınca sıfırlanıyordu;
   Firebase oturumu açık olsa da kullanıcı tekrar giriş yapmak zorundaydı.
5. **Rol kontrolü kod içinde tekrarlanıyordu.** `role == admin || role == mudur`
   ~15 yerde elle yazılmıştı. Şablonda tek yetki matrisi var.
6. **Müdür/admin ayrımı belirsizdi.** Müdür girişte `/admin`'e yönlendiriliyordu ama
   menüde "Yönetim" sekmesi sadece admine görünüyordu. Şablonda `adminPanel` yetkisi net.
7. **Geliştirici konsolu** (`developer_console.dart`) rolü giriş yapmadan değiştirebilen
   butonlar içeriyordu. Bu tür araçları sadece `kDebugMode` altında tut, yayına çıkarma.

---

## 8. Uyarlama Rehberi

### Yeni rol eklemek (ör. `muhasebe`)
1. `AppRole` enum'una ekle, `fromString` içine metin karşılığını yaz.
2. `rolePermissions` matrisine yetkilerini yaz (gerekirse yeni `Permission` ekle).
3. `firestore.rules` içinde ilgili yardımcı fonksiyonu güncelle
   (ör. `function canSeeReports() { return myRole() in ['muhasebe', 'admin']; }`).
4. Admin panelindeki rol seçim listesine ekle.

### Farklı uygulamaya taşımak
- `services` / `notifications` yerine kendi koleksiyonlarını koy; "sahiplik" alanını
  (`technicianId`) kendi alanınla değiştir (ör. `ownerId`, `assignedTo`).
- Tek şirketli bir uygulamaysa `companyId` kontrollerini kaldırabilirsin; ama ileride
  lazım olabileceği için tutmak genelde daha kolaydır.

### Daha ileri seviye (büyük projeler için)
Kurallar her istekte `users` belgesini `get()` ile okur (ücretli okuma sayılır).
Çok kullanıcılı projelerde rolü **Firebase Custom Claims** ile token'a yazmak
(Cloud Functions + Admin SDK) daha hızlı ve güvenlidir. Kurallarda
`request.auth.token.role` olarak okunur. Bu, Blaze (ücretli) plan gerektirir.

---

## 9. Kontrol Listesi (bitirmeden önce)

- [ ] Kayıt olan kullanıcı `teknisyen` ve `companyId: null` ile oluşuyor.
- [ ] Kullanıcı kendi `role`/`companyId` alanını değiştiremiyor (kurallarda reddediliyor).
- [ ] Teknisyen başka birinin servisini **API ile bile** okuyamıyor.
- [ ] Başka şirketin verisi hiçbir rolle okunamıyor.
- [ ] Silme işlemleri kurallarda yönetici rolüne bağlı.
- [ ] Sorgular kurallarla uyumlu (teknisyen sorgusu `technicianId` filtresi içeriyor).
- [ ] Uygulama yeniden açıldığında oturum ve rol korunuyor.
- [ ] Admin rolü değiştirince kullanıcının ekranı güncelleniyor.
- [ ] Yetkisiz sayfaya URL ile gidilince yönlendirme yapılıyor.
- [ ] Debug/geliştirici araçları yayın sürümünde kapalı.
- [ ] Kurallar emülatörde test edildi ve `firebase deploy --only firestore:rules` ile yüklendi.
