# Rol Sistemi Referans Paketi — Beni Oku

Bu klasör, **ServisPort** uygulamasındaki rol sistemini (teknisyen / müdür / admin) başka
Flutter + Firebase uygulamalarında tekrar kullanabilmen için hazırlandı. Dosyalar hem senin
okuman hem de **Claude'a referans olarak vermen** için yazıldı.

## Klasörde ne var?

| Dosya | Ne işe yarar? |
|---|---|
| `BENI_OKU.md` | Bu dosya. Claude ile nasıl kullanacağını anlatır. |
| `ROL_SISTEMI_REFERANS.md` | **Asıl dosya.** Rol sisteminin tamamı: mantık, veri yapısı, kod şablonları, güvenlik kuralları, sık yapılan hatalar. Claude'a bunu ver. |
| `FIREBASE_KURULUM.md` | Kendi Firebase projeni sıfırdan oluşturup uygulamaya bağlama adımları. |
| `firestore.rules` | Kopyalayıp doğrudan kullanabileceğin, güvenliği sıkılaştırılmış Firestore kuralları. |

---

## Claude ile en verimli kullanım (3 yol)

### 1) Claude Code (terminal) — en güçlü yol

Yeni projenin klasöründe Claude Code'u aç ve şunu yaz:

```
~/Desktop/Rol_Sistemi_Referans/ROL_SISTEMI_REFERANS.md dosyasını oku ve bu projeye
rol sistemini kur. Önce bana bir plan göster, onaylarsam uygula.
```

**Kalıcı yapmak istersen:** Projenin kökündeki `CLAUDE.md` dosyasına şu satırı ekle.
Claude o projede her oturumda referansı otomatik bilir:

```
@~/Desktop/Rol_Sistemi_Referans/ROL_SISTEMI_REFERANS.md
```

> İpucu: Masaüstündeki klasörü silersen bağlantı kopar. Daha sağlamı: klasörü projenin
> içine (`docs/rol_sistemi/`) kopyala ve `@docs/rol_sistemi/ROL_SISTEMI_REFERANS.md` yaz.

### 2) claude.ai (web / masaüstü uygulaması)

- Sohbete `ROL_SISTEMI_REFERANS.md` dosyasını **sürükle-bırak** ve isteğini yaz.
- Sık kullanacaksan bir **Project** oluştur, dosyayı *Project knowledge* kısmına ekle.
  O projedeki her sohbet referansı bilir.

### 3) Sadece bir parçası lazımsa

Dosyanın tamamı yerine ilgili bölümü kopyalayıp yapıştırabilirsin
(ör. sadece "Firestore Güvenlik Kuralları" bölümü).

---

## Hazır örnek istekler (kopyala-yapıştır)

- `Referanstaki rol sistemini bu projeye kur. Rollerim: personel, şef, admin.`
- `Referanstaki firestore.rules dosyasını benim koleksiyonlarıma (siparisler, urunler) uyarla.`
- `Referansa göre "muhasebe" diye yeni bir rol ekle; sadece raporları görebilsin.`
- `Bu projedeki rol kontrollerini referanstaki "Kontrol Listesi"ne göre denetle, eksikleri söyle.`
- `Firestore kurallarımı Firebase Emulator ile test edecek testleri yaz.`

## Daha iyi sonuç için 5 kısa ipucu

1. **Rollerini ve kimin neyi yapabileceğini baştan söyle.** Bir tablo bile yeter
   ("teknisyen: kendi işlerini görür, müdür: hepsini görür + siler").
2. **Önce plan iste**, sonra uygulat. Büyük değişikliklerde hatayı erkenden yakalarsın.
3. **Adım adım ilerle:** önce rol modeli, sonra giriş, sonra ekran kısıtları, en son kurallar.
4. **Güvenliği uygulamaya değil kurallara emanet et.** Claude'a "bu kısıtlama
   firestore.rules'ta da var mı?" diye sor. (Referansta neden önemli olduğu anlatılıyor.)
5. **Test ettir:** "Teknisyen hesabıyla başka birinin servisini okumayı dene, reddedilmeli"
   gibi somut senaryolar ver.

## Kendi Firebase'ini bağlamak

Adım adım anlatım `FIREBASE_KURULUM.md` dosyasında. Özetle:
Firebase Console'da proje aç → Authentication ve Firestore'u aç →
`flutterfire configure` çalıştır → `firestore.rules` dosyasını yükle →
ilk admin kullanıcını elle oluştur.
