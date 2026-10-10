# Üniversite İçi Messenger — Sistem Analizi ve Mimari Tasarım

> **Durum:** Taslak v0.1 · **Tarih:** 10.10.2026
> **Amaç:** Üniversite içinde kullanılacak, kurumsal yapıyla (birimler, fakülteler, bölümler, kurullar, komisyonlar) entegre çalışan güvenli bir mesajlaşma platformunun yapısını tanımlamak.

---

## İçindekiler

1. [Yönetici Özeti](#1-yönetici-özeti)
2. [Kapsam ve Hedefler](#2-kapsam-ve-hedefler)
3. [Paydaşlar ve Kullanıcı Tipleri](#3-paydaşlar-ve-kullanıcı-tipleri)
4. [Üniversite Organizasyon Modeli](#4-üniversite-organizasyon-modeli)
5. [Grup (Konuşma) Taksonomisi](#5-grup-konuşma-taksonomisi)
6. [Otomatik / Manuel / Hibrit Grup Kurulumu](#6-otomatik--manuel--hibrit-grup-kurulumu)
7. [Kural Motoru (Membership Rules)](#7-kural-motoru-membership-rules)
8. [Senkronizasyon ve Yaşam Döngüsü](#8-senkronizasyon-ve-yaşam-döngüsü)
9. [Yetkilendirme Modeli](#9-yetkilendirme-modeli)
10. [Mesajlaşma Özellikleri](#10-mesajlaşma-özellikleri)
11. [Veri Modeli](#11-veri-modeli)
12. [Sistem Mimarisi](#12-sistem-mimarisi)
13. [Entegrasyonlar](#13-entegrasyonlar)
14. [Güvenlik, KVKK ve Uyumluluk](#14-güvenlik-kvkk-ve-uyumluluk)
15. [Ölçek ve Performans Hedefleri](#15-ölçek-ve-performans-hedefleri)
16. [Uç Durumlar (Edge Cases)](#16-uç-durumlar-edge-cases)
17. [API Taslağı](#17-api-taslağı)
18. [Yol Haritası](#18-yol-haritası)
19. [Riskler](#19-riskler)
20. [Açık Sorular](#20-açık-sorular)

---

## 1. Yönetici Özeti

Bu uygulamayı WhatsApp/Slack benzeri genel bir mesajlaşma uygulamasından ayıran şey **kurumsal yapı farkındalığıdır.** Sistemin kalbi mesaj iletimi değil, **"kim, hangi grupta, hangi yetkiyle, neden bulunuyor?"** sorusunu doğru ve otomatik cevaplayan **Organizasyon ve Grup Katmanı**dır.

Temel tasarım kararları:

| Karar | Seçim | Gerekçe |
|---|---|---|
| Organizasyon modeli | Ağaç (hiyerarşi) + çoklu görevlendirme | Bir kişi aynı anda birden fazla birimde/görevde olabilir |
| Grup kurulumu | **Otomatik**, **Manuel**, **Hibrit** olmak üzere 3 mod | Fakülte/bölüm grupları veriden türetilir; komisyonlar elle kurulur; ikisinin karışımı da gerekir |
| Üyelik kaynağı | Her üyeliğin bir **kaynağı (source)** vardır | Senkronizasyonun manuel eklenen kişiyi silmemesi için şart |
| Senkronizasyon | **İstenen durum ↔ gerçek durum farkı (reconcile)** | Idempotent, tekrar çalıştırılabilir, denetlenebilir |
| Yetki | RBAC (rol) + ABAC (özellik/politika) | "Öğrenci, rektöre doğrudan mesaj atabilir mi?" gibi kurumsal politikalar |
| Kurulum | Üniversite sunucularında (on‑premise) veya yurtiçi bulut | KVKK, veri yerelliği |
| Mimari | Modüler monolit → gerektikçe servislere ayrılır | Küçük ekip, hızlı teslim, ileride ölçeklenebilir |

---

## 2. Kapsam ve Hedefler

### 2.1 Kapsam İçi
- Bire bir (DM) ve grup mesajlaşma
- Kurumsal yapıdan **otomatik** oluşan gruplar (fakülte, bölüm, birim, sınıf/ders şubesi)
- **Manuel** kurulan gruplar (komisyonlar, proje ekipleri, çalışma grupları)
- **Duyuru kanalları** (tek yönlü, okundu takibi)
- Kurumsal rehber (kim kimdir, hangi birim, unvan)
- Dosya paylaşımı, arama, bildirimler
- Web, iOS, Android, masaüstü istemcileri
- Yönetim paneli (grup, kural, politika, denetim kayıtları)

### 2.2 Kapsam Dışı (ilk sürüm)
- Resmi yazışma (EBYS / e‑imza) — mesajlaşma resmi evrak yerine geçmez, ancak ileride EBYS'ye "bağlantı" verilebilir
- Görüntülü toplantı (ilk etapta Zoom/Teams/Jitsi entegrasyonu ile çözülür)
- Öğrenme yönetim sistemi (LMS) işlevleri

### 2.3 Başarı Kriterleri
- Yeni dönem başında ders/bölüm gruplarının **insan müdahalesi olmadan** hazır olması
- Görev değişikliğinde (ör. yeni bölüm başkanı) yetkilerin **24 saat içinde** (hedef: 15 dk) güncellenmesi
- Resmi duyuruların okunma oranının ölçülebilmesi
- E‑posta trafiğinde birim içi iletişimde belirgin azalma

---

## 3. Paydaşlar ve Kullanıcı Tipleri

### 3.1 Kullanıcı Tipleri (Kişi Türü)

| Tip | Açıklama | Veri Kaynağı |
|---|---|---|
| `ACADEMIC` | Öğretim üyesi / görevlisi, araştırma görevlisi | Personel (İK) sistemi |
| `ADMINISTRATIVE` | İdari personel | Personel (İK) sistemi |
| `STUDENT` | Ön lisans, lisans, yüksek lisans, doktora | Öğrenci Bilgi Sistemi (ÖBS) |
| `CONTRACTOR` | Yarı zamanlı, ders saati ücretli, dış hizmet | İK veya manuel |
| `GUEST` | Misafir öğretim üyesi, Erasmus, dış komisyon üyesi | Manuel (süreli) |
| `ALUMNI` | Mezun (opsiyonel, kısıtlı) | ÖBS |
| `SYSTEM` | Bot / entegrasyon hesabı | Sistem |

> Bir kişi **birden fazla tipe** sahip olabilir (ör. hem araştırma görevlisi hem doktora öğrencisi). Bu nedenle tip, kullanıcıya tek bir alan olarak değil **görevlendirme (affiliation)** kayıtları üzerinden bağlanır.

### 3.2 Yönetici Rolleri (Platform Seviyesi)

| Rol | Yetki |
|---|---|
| `SUPER_ADMIN` | Tüm sistem (Bilgi İşlem) |
| `ORG_ADMIN` | Kurumsal yapı, kurallar, politikalar |
| `UNIT_ADMIN` | Kendi birimi ve alt birimlerinde grup yönetimi (ör. fakülte sekreteri) |
| `COMPLIANCE_OFFICER` | Denetim kaydı, yasal saklama (legal hold), dışa aktarma — *mesaj okuma yetkisi yalnızca resmi süreçle* |
| `HELPDESK` | Kullanıcı destek, hesap kilidi, cihaz çıkışı |

---

## 4. Üniversite Organizasyon Modeli

### 4.1 Hiyerarşi

Üniversite yapısı bir **ağaç** olarak modellenir. Her düğüm bir **Organizasyon Birimi (OrgUnit)**'dir.

```
Üniversite
├── Rektörlük
│   ├── Rektör Yardımcılıkları
│   └── Genel Sekreterlik
│       ├── Personel Daire Başkanlığı
│       ├── Öğrenci İşleri Daire Başkanlığı
│       ├── Bilgi İşlem Daire Başkanlığı
│       └── ...
├── Fakülteler
│   ├── Mühendislik Fakültesi
│   │   ├── Dekanlık
│   │   ├── Fakülte Sekreterliği
│   │   ├── Bilgisayar Mühendisliği Bölümü
│   │   │   ├── Yazılım Anabilim Dalı
│   │   │   └── Donanım Anabilim Dalı
│   │   │       └── Program: Bilgisayar Müh. (TR) / (EN)
│   │   └── Elektrik-Elektronik Müh. Bölümü
│   └── Tıp Fakültesi ...
├── Enstitüler (Lisansüstü Eğitim Enstitüsü ...)
├── Yüksekokullar / Meslek Yüksekokulları
└── Araştırma ve Uygulama Merkezleri
```

### 4.2 Birim Tipleri (`OrgUnitType`)

`UNIVERSITY`, `RECTORATE`, `VICE_RECTORATE`, `GENERAL_SECRETARIAT`, `DEPARTMENT_DIRECTORATE` (daire başkanlığı), `FACULTY`, `INSTITUTE`, `SCHOOL` (yüksekokul), `VOCATIONAL_SCHOOL` (MYO), `ACADEMIC_DEPARTMENT` (bölüm), `DIVISION` (anabilim dalı), `PROGRAM`, `RESEARCH_CENTER`, `OFFICE` (şube/birim), `OTHER`.

> Tipler **sabit kodlanmamalı**, tablo olarak tutulmalı; her üniversitenin yapısı farklıdır (ör. "Koordinatörlük", "Ofis").

### 4.3 Görevlendirme (Affiliation) — En Kritik Kavram

Kişi ile birim arasındaki ilişki **çoktan çoğa** ve **zamana bağlıdır**:

```
Affiliation
  person        → Dr. Ayşe Yılmaz
  org_unit      → Bilgisayar Mühendisliği Bölümü
  kind          → ACADEMIC_STAFF | ADMIN_STAFF | STUDENT | MANAGER
  position      → Bölüm Başkanı | Doç. Dr. | Öğrenci (3. sınıf) ...
  is_primary    → true/false     (asıl kadro mu, görevlendirme mi)
  valid_from    → 2025-09-01
  valid_until   → 2028-08-31     (null = süresiz)
  source        → HR_SYNC | SIS_SYNC | MANUAL
  attributes    → { "class_year": 3, "degree": "BSc", "language": "EN" }
```

Örnek: Dr. Ayşe Yılmaz'ın 4 aktif görevlendirmesi olabilir:
1. Bilgisayar Müh. Bölümü — Doç. Dr. (asıl kadro)
2. Bilgisayar Müh. Bölümü — Bölüm Başkanı (idari görev)
3. Lisansüstü Eğitim Enstitüsü — Danışman
4. Rektörlük — Kalite Komisyonu Üyesi (bu bir **komite üyeliği**dir, bkz. 4.4)

### 4.4 Pozisyonlar ve Kurullar

**Pozisyon (Position):** Unvan ve idari görevler ayrı tutulur.
- *Akademik unvan:* Prof. Dr., Doç. Dr., Dr. Öğr. Üyesi, Öğr. Gör., Arş. Gör.
- *İdari görev:* Rektör, Rektör Yrd., Dekan, Dekan Yrd., Bölüm Başkanı, Bölüm Başkan Yrd., Anabilim Dalı Başkanı, Müdür, Fakülte Sekreteri, Daire Başkanı, Şube Müdürü...

**Kurul / Komisyon (Committee):** Mevzuattan gelen kurullar ile üniversitenin kendi oluşturduğu komisyonlar.

| Tür | Örnek | Kurulum |
|---|---|---|
| Kanuni kurul (pozisyondan türer) | Senato, Üniversite Yönetim Kurulu, Fakülte Kurulu, Fakülte Yönetim Kurulu, Bölüm Kurulu | **Hibrit** — üyelerin çoğu pozisyondan otomatik (dekanlar, müdürler), bir kısmı seçimle manuel (fakülte temsilcisi) |
| Kalıcı komisyon | Kalite Komisyonu, Etik Kurul, Eğitim Komisyonu, Burs Komisyonu | **Manuel** |
| Geçici komisyon | Akreditasyon Çalışma Grubu, Sınav Komisyonu, Jüri | **Manuel + bitiş tarihli** |

---

## 5. Grup (Konuşma) Taksonomisi

### 5.1 Konuşma Türleri (teknik)

| Tür | Açıklama | Kim yazar? |
|---|---|---|
| `DIRECT` | 1‑1 mesaj | İki taraf |
| `GROUP_DM` | Küçük, isimsiz, geçici grup (≤ 10 kişi) | Herkes |
| `GROUP` | İsimli, kalıcı grup | Herkes (politika ile kısıtlanabilir) |
| `CHANNEL` | Konu bazlı, katılımı açık/kapalı | Herkes |
| `ANNOUNCEMENT` | Tek yönlü duyuru | Yalnızca yetkililer; diğerleri tepki/okundu |
| `SPACE` | Birden fazla grubu/kanalı kapsayan çatı (ör. "Mühendislik Fakültesi" alanı) | — |

**`SPACE` (Alan) kavramı** büyük birimlerde gürültüyü önler. Örneğin:

```
SPACE: Mühendislik Fakültesi
  ├── 📢 Fakülte Duyuruları         (ANNOUNCEMENT, tüm fakülte, otomatik)
  ├── 👥 Fakülte Akademik Personel  (GROUP, otomatik)
  ├── 👥 Fakülte İdari Personel     (GROUP, otomatik)
  ├── 🏛️ Fakülte Kurulu             (GROUP, hibrit)
  ├── 🏛️ Fakülte Yönetim Kurulu     (GROUP, hibrit)
  └── # genel-sohbet                (CHANNEL, katılım isteğe bağlı)
```

### 5.2 Grup Kategorileri (iş anlamı)

| Kategori | Örnek | Varsayılan Mod | Üye Kaynağı |
|---|---|---|---|
| `INSTITUTION` | Tüm Üniversite Duyuruları, Tüm Personel | Otomatik | Tüm aktif kullanıcılar / personel |
| `ADMIN_UNIT` | Bilgi İşlem Daire Bşk., Öğrenci İşleri | Otomatik | İK birim ataması |
| `FACULTY` | Mühendislik Fak. Personeli / Öğrencileri | Otomatik | Birim ağacı (alt birimler dahil) |
| `ACADEMIC_DEPT` | Bilgisayar Müh. Bölümü Akademik Kadro | Otomatik | Bölüm ataması |
| `DIVISION` | Yazılım Anabilim Dalı | Otomatik | Anabilim dalı ataması |
| `COHORT` | Bilgisayar Müh. 2024 Girişliler / 3. Sınıflar | Otomatik | ÖBS sınıf/giriş yılı |
| `COURSE_SECTION` | BİL301 – Şube 2 – 2026 Güz | Otomatik | ÖBS ders kayıtları + dersi veren hoca |
| `STATUTORY_BOARD` | Senato, Fakülte Kurulu, Bölüm Kurulu | Hibrit | Pozisyon kuralları + manuel seçilmiş üyeler |
| `COMMITTEE` | Kalite Komisyonu, Burs Komisyonu | Manuel | Görevlendirme yazısına göre yönetici ekler |
| `ROLE_BASED` | Tüm Bölüm Başkanları, Tüm Dekan Yrd., Tüm Fakülte Sekreterleri | Otomatik | Pozisyon |
| `PROJECT` | TÜBİTAK 1001 Proje Ekibi | Manuel | Kullanıcı oluşturur |
| `CLUB` | Öğrenci Kulüpleri | Manuel (onaylı) | SKS onayı ile |
| `ADVISOR` | Danışman – Danışmanlık öğrencileri | Otomatik | ÖBS danışman ataması |
| `AD_HOC` | Serbest çalışma grupları | Manuel | Kullanıcı oluşturur |

### 5.3 İsimlendirme Standardı

Otomatik gruplar şablonla isimlendirilir, böylece birim adı değişince grup adı da güncellenir:

```
{unit.short_name} · {audience}            → "Bil. Müh. · Akademik Kadro"
{course.code} · Şube {section} · {term}   → "BİL301 · Şube 2 · 2026 Güz"
{unit.short_name} · {year}. Sınıf         → "Bil. Müh. · 3. Sınıf"
```

Her otomatik grubun ayrıca değişmeyen bir **slug/anahtar**ı olur:
`auto:dept:{unit_id}:academic`, `auto:course:{term}:{course_code}:{section}` — senkronizasyon bu anahtarla eşleştirir (ad değil!).

---

## 6. Otomatik / Manuel / Hibrit Grup Kurulumu

### 6.1 Üç Mod

| Mod | Grubu kim oluşturur? | Üyeleri kim belirler? | Kullanıcı ayrılabilir mi? |
|---|---|---|---|
| **AUTO** | Sistem (şablon/kural) | Yalnızca kural | Hayır (zorunlu gruplar) / Evet ama bildirimleri kapatabilir |
| **MANUAL** | Yetkili kişi | Grup yöneticisi | Politikaya bağlı |
| **HYBRID** | Sistem veya yetkili | Kural **+** manuel ekleme **+** manuel hariç tutma | Kural üyeleri hayır, manuel üyeler politikaya bağlı |

### 6.2 Üyelik Kaynağı (Membership Source) — Hibritin Anahtarı

Her üyelik kaydı **neden** orada olduğunu bilir. Bir kişinin aynı grupta birden fazla kaynaktan üyeliği olabilir:

```
GroupMembership (grup: Fakülte Kurulu, kişi: Prof. X)
  sources:
    - RULE  (rule_id: "dekan-ve-bolum-baskanlari")
    - MANUAL(added_by: fakulte_sekreteri, reason: "Senato temsilcisi", until: 2027-09-01)
```

Kurallar:
- Kişi, **en az bir aktif kaynağı** olduğu sürece grupta kalır.
- Senkronizasyon yalnızca **kendi kaynağını (RULE)** ekler/siler, **MANUAL** kaynağa dokunmaz.
- **EXCLUDE** (hariç tutma) kaydı kuralın üstündedir: kural kişiyi eklemek istese bile eklenmez (ör. uzun süreli izindeki personel, isteğe bağlı çıkış).
- Manuel ekleme **bitiş tarihli** olabilir (geçici görevlendirme, misafir).

Öncelik sırası: `EXCLUDE` > `MANUAL` > `RULE`

### 6.3 Akış: Otomatik Grup Şablonu

Otomatik gruplar tek tek değil **şablon (GroupTemplate)** ile tanımlanır. Şablon, organizasyon ağacındaki her uygun düğüm için bir grup üretir:

```yaml
template: dept-academic-staff
applies_to:
  org_unit_type: ACADEMIC_DEPARTMENT
create:
  kind: GROUP
  category: ACADEMIC_DEPT
  key: "auto:dept:{unit.id}:academic"
  name: "{unit.short_name} · Akademik Kadro"
  space: "auto:space:{unit.parent(FACULTY).id}"
  mode: AUTO
membership_rule:
  all:
    - affiliation.org_unit: { equals_or_descendant_of: "{unit.id}" }
    - affiliation.kind: ACADEMIC_STAFF
    - affiliation.active: true
roles:
  - role: ADMIN
    rule: { position: { in: [BOLUM_BASKANI, BOLUM_BASKAN_YRD] }, org_unit: "{unit.id}" }
  - role: ADMIN
    rule: { position: { in: [BOLUM_SEKRETERI] }, org_unit: "{unit.id}" }
settings:
  posting: ALL_MEMBERS
  allow_leave: false
  history_visible_to_new_members: true
on_source_removed: ARCHIVE      # birim kapatılırsa grup arşivlenir
```

Bu şablon sayesinde yeni bir bölüm açıldığında (İK/ÖBS'ye eklendiğinde) gruplar **kendiliğinden** oluşur.

### 6.4 Önerilen Varsayılan Otomatik Gruplar

| # | Şablon | Her biri için | Üyeler |
|---|---|---|---|
| 1 | Üniversite Duyuru Kanalı | Tek | Herkes (yazma: Rektörlük, Kurumsal İletişim) |
| 2 | Tüm Personel | Tek | Tüm akademik + idari |
| 3 | Birim Personeli | Her idari birim | Birimde çalışanlar |
| 4 | Fakülte Duyuruları | Her fakülte/enstitü/MYO | Fakültedeki herkes (öğrenci dahil) |
| 5 | Fakülte Akademik Kadro | Her fakülte | Akademik personel |
| 6 | Fakülte İdari Kadro | Her fakülte | İdari personel |
| 7 | Bölüm Akademik Kadro | Her bölüm | Bölüm akademisyenleri |
| 8 | Bölüm Öğrenci Duyuruları | Her bölüm | Bölüm öğrencileri + akademisyenler |
| 9 | Sınıf Grubu | Bölüm × sınıf | Öğrenciler + sınıf danışmanı |
| 10 | Ders Şubesi | Dönem × ders × şube | Kayıtlı öğrenciler + öğretim elemanı + asistan |
| 11 | Danışman Grubu | Her danışman | Danışman + danışmanlık öğrencileri |
| 12 | Rol Grupları | Pozisyon | Tüm Dekanlar, Tüm Bölüm Başkanları, Tüm Fakülte Sekreterleri |
| 13 | Kanuni Kurullar | Senato, ÜYK, Fakülte Kurulu, FYK, Bölüm Kurulu | **Hibrit** |

> Öneri: 9–11 numaralı gruplar her kurumda istenmeyebilir; şablonlar **açılıp kapatılabilir** olmalı.

### 6.5 Manuel Grup Oluşturma Politikası

Kim, hangi kategoride grup açabilir?

| Kategori | Oluşturabilecek kişi | Onay |
|---|---|---|
| `COMMITTEE` | ORG_ADMIN, UNIT_ADMIN, Rektör Yrd., Dekan | Yok (görevlendirme yazısı no'su opsiyonel alan) |
| `PROJECT` | Akademik personel | Yok |
| `CLUB` | Öğrenci (kulüp başkanı) | SKS Daire Başkanlığı onayı |
| `AD_HOC` | Herkes | Yok, ama öğrenciler için üye limiti ve süre sınırı |

Manuel gruplarda **"kural ile toplu ekleme"** kolaylığı olmalı: "Mühendislik Fakültesi'ndeki tüm Bölüm Başkanlarını ekle" dendiğinde sistem iki seçenek sunar:
- **Anlık kopyala** (snapshot → manuel üyeler; sonradan değişmez)
- **Bağlı kal** (kural eklenir → grup HYBRID olur; bölüm başkanı değişince otomatik güncellenir)

---

## 7. Kural Motoru (Membership Rules)

### 7.1 Gereksinimler
- Organizasyon ağacında **alt birimleri kapsama** (`descendant_of`)
- Kişi tipi, pozisyon, unvan, öğrenci sınıfı, giriş yılı, program dili, öğrenim düzeyi, statü (aktif/izinli/kayıt donduran)
- `all` / `any` / `not` ile birleştirme
- Tarih koşulları (görevlendirme aktif mi?)
- **Önizleme:** Kural kaydedilmeden "bu kural 1.243 kişi içeriyor, ilk 50 kişi:" listesi
- **Açıklanabilirlik:** "Bu kişi bu grupta neden var?" → hangi kural/hangi görevlendirme

### 7.2 Kural DSL (JSON)

```json
{
  "all": [
    { "field": "affiliation.kind", "op": "eq", "value": "STUDENT" },
    { "field": "affiliation.org_unit", "op": "descendant_of", "value": "unit_eng_faculty" },
    { "field": "affiliation.attributes.class_year", "op": "in", "value": [3, 4] },
    { "not": { "field": "person.status", "op": "eq", "value": "FROZEN" } }
  ]
}
```

### 7.3 Uygulama Önerisi
- Kurallar SQL'e derlenir (PostgreSQL). Ağaç sorguları için **closure table** veya `ltree` kullanılır.
- Kural değerlendirme sonucu bir **küme** (person_id seti) üretir; reconcile bu küme ile mevcut RULE kaynaklı üyeleri karşılaştırır.
- Büyük kurallar (ör. "tüm öğrenciler", 30.000 kişi) için grup üyeliği satır satır değil, **dinamik hedef kitle** olarak saklanabilir (yalnızca duyuru kanallarında; bkz. 15.2).

---

## 8. Senkronizasyon ve Yaşam Döngüsü

### 8.1 Veri Akışı

```
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ İK / Personel│   │     ÖBS      │   │  LDAP / AD   │
 └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
        │ (API / DB view / CSV / webhook)      │
        ▼                  ▼                  ▼
 ┌─────────────────────────────────────────────────┐
 │           Connector'lar (Adaptör katmanı)        │
 │   normalize → Person, OrgUnit, Affiliation,      │
 │               Course, Enrollment                 │
 └───────────────────────┬─────────────────────────┘
                         ▼
 ┌─────────────────────────────────────────────────┐
 │        Directory (Kurumsal Rehber) Servisi       │
 │   değişiklik olayları: person.updated,           │
 │   affiliation.created/ended, orgunit.moved ...   │
 └───────────────────────┬─────────────────────────┘
                         ▼
 ┌─────────────────────────────────────────────────┐
 │       Provisioning (Grup Kurulum) Motoru         │
 │  1) Şablonlar → olması gereken gruplar           │
 │  2) Kurallar  → olması gereken RULE üyelikleri   │
 │  3) Diff      → ekle / çıkar / rol değiştir      │
 │  4) Uygula    → Messaging çekirdeği + audit log  │
 └─────────────────────────────────────────────────┘
```

### 8.2 Senkronizasyon Stratejisi
- **Olay bazlı (event‑driven):** Kaynak sistem webhook/CDC destekliyorsa değişiklik anında işlenir.
- **Periyodik tam reconcile:** Her gece (ör. 03:00) tüm şablon ve kurallar yeniden hesaplanır; olay kaçırılmışsa düzeltilir.
- **Dönem başı toplu iş:** Ders şubesi grupları ders kayıt döneminde saatlik çalışır.
- **Güvenlik freni (circuit breaker):** Tek bir senkronda üyelerin %X'inden fazlası silinecekse (ör. kaynak sistem boş veri döndü) işlem **durdurulur** ve yöneticiye onaya düşer. *Bu çok önemli: hatalı bir CSV ile tüm üniversitenin gruplardan atılması engellenir.*
- **Kuru çalıştırma (dry‑run):** Her senkronun "ne değişecek" raporu üretilebilir.

### 8.3 Grup Yaşam Döngüsü

```
          oluştur
  (yok) ─────────▶ ACTIVE ──────────▶ READ_ONLY ─────────▶ ARCHIVED ─────▶ DELETED
                    │  ▲   dönem bitti    (yazma kapalı)      (gizli, aranabilir)  (saklama süresi doldu)
                    │  │
                    ▼  │
                  SUSPENDED (yönetici dondurdu)
```

| Olay | Davranış |
|---|---|
| Birim kapatıldı (İK'dan silindi) | Grup `READ_ONLY` → 90 gün sonra `ARCHIVED` |
| Birim birleştirildi | Eski gruplar `READ_ONLY`, yeni birim grubu oluşur, eski gruba bağlantı mesajı |
| Birim adı değişti | Grup adı şablondan yeniden üretilir, anahtar (key) değişmez |
| Dönem bitti (ders şubesi) | Not giriş tarihinden X gün sonra `READ_ONLY`, 1 yıl sonra `ARCHIVED` |
| Geçici komisyonun bitiş tarihi geldi | Yöneticiye uyarı → uzatma veya arşiv |
| Komisyon yöneticisi ayrıldı | Grup **sahipsiz kalamaz**: üst birim yöneticisine devredilir |

### 8.4 Kişi Yaşam Döngüsü

| Olay | Davranış |
|---|---|
| Yeni personel/öğrenci | Hesap oluşur (SSO ilk girişte veya önceden), kurallar uygun gruplara ekler |
| Birim değişikliği | Eski birim gruplarından çıkar, yenilere girer (aynı senkronda) |
| İdari görev bitti (ör. bölüm başkanlığı) | Pozisyon kaynaklı yönetici rolü düşer, akademik üyelik devam eder |
| Ücretsiz izin / askerlik | Politika: gruplarda kalır ama `INACTIVE` görünür veya geçici hariç tutulur |
| Kayıt donduran öğrenci | Ders gruplarından çıkar, bölüm duyurularında kalır |
| Mezuniyet | Öğrenci gruplarından çıkar; `ALUMNI` tipine geçer veya hesap kapanır (politika) |
| İşten ayrılma | Hesap anında devre dışı, oturumlar sonlandırılır, mesajlar saklama politikasına göre kalır |
| Vefat | Hesap dondurulur, özel işlem (profil "anısına" vs. kurum kararı) |

---

## 9. Yetkilendirme Modeli

### 9.1 Grup İçi Roller

| Rol | Yetkiler |
|---|---|
| `OWNER` | Grubu silme/arşivleme, yönetici atama, ayarlar |
| `ADMIN` | Üye ekleme/çıkarma (manuel kaynak), ayarlar, mesaj sabitleme, mesaj silme |
| `MODERATOR` | Mesaj gizleme/silme, kullanıcı susturma |
| `POSTER` | Duyuru kanalında yazma yetkisi |
| `MEMBER` | Okuma/yazma (grup ayarına göre) |
| `READER` | Yalnızca okuma |

Roller de üyelik gibi **kaynaklıdır**: Bölüm başkanı kuraldan `ADMIN` olur; görevi bitince otomatik düşer. Manuel verilen rol ise kalır.

### 9.2 Kurumsal İletişim Politikaları (ABAC)

Platform genelinde kim kime ulaşabilir? Yapılandırılabilir politika tablosu:

| Gönderen \ Alıcı | Öğrenci | Akademik | İdari | Üst Yönetim |
|---|---|---|---|---|
| **Öğrenci** | ✅ (opsiyonel: aynı bölüm) | ✅ yalnızca **ders hocası / danışman** veya hoca "açık kapı" seçtiyse | ✅ yalnızca hizmet birimleri (Öğrenci İşleri vb.) | ❌ (kurumsal kanal üzerinden) |
| **Akademik** | ✅ | ✅ | ✅ | ✅ |
| **İdari** | ✅ | ✅ | ✅ | ✅ |
| **Üst Yönetim** | ✅ | ✅ | ✅ | ✅ |

Ek politika örnekleri:
- Öğrenciler yalnızca öğrencilerden oluşan grup açabilir; personel ekleyemez (kulüp danışmanı hariç).
- Mesai dışı bildirimler: Personel için "rahatsız etme" saatleri varsayılan açık (22:00–08:00), **acil** işaretli mesajlar hariç.
- Belirli birimler (Rektörlük) için DM yerine **"talep/kuyruk"** modeli (paylaşılan gelen kutusu).
- Dışarıdan misafir hesapları yalnızca eklendikleri gruplarda görünür, rehbere erişemez.

Politika motoru için öneri: Basit kural tablosu ile başlanır; karmaşıklaşırsa **OPA (Open Policy Agent)** veya **Cedar** ile politika‑kod (policy‑as‑code).

### 9.3 Rehber Görünürlüğü
- Personel herkesi görür (unvan, birim, dahili telefon, ofis).
- Öğrenciler personeli görür; diğer öğrencileri yalnızca ortak gruplarda görür (KVKK — veri minimizasyonu).
- Öğrenci numarası, T.C. kimlik no gibi alanlar **asla** rehberde gösterilmez.

---

## 10. Mesajlaşma Özellikleri

### 10.1 MVP
- Metin, emoji, tepkiler, yanıt (reply), düzenleme/silme (süre sınırlı)
- Konu dizileri (thread)
- @bahsetme: `@kişi`, `@grup`, `@here`, `@all` (büyük gruplarda `@all` yalnızca yöneticiler)
- Dosya / görsel / ses kaydı paylaşımı (boyut sınırı, virüs taraması)
- Okundu / iletildi bilgisi (DM ve küçük gruplarda; büyük gruplarda kapalı)
- Push bildirim, bildirim tercihleri (grup bazında sessize alma)
- Tam metin arama (Türkçe dil desteği — ı/i, ş/s, ekler)
- Sabitlenmiş mesajlar, favori konuşmalar
- Çevrimiçi durumu ve durum mesajı ("Derste", "İzinde", "Toplantıda")

### 10.2 Kuruma Özel Özellikler
- **Resmi Duyuru:** "Okudum, anladım" onay butonu; yöneticiye okunma raporu (kim okudu/okumadı, birim bazında oran). Okumayanlara hatırlatma.
- **Hedefli Duyuru:** Kural ile hedef kitle seçimi ("Mühendislik Fakültesi 4. sınıf öğrencileri") → grup oluşturmadan tek seferlik duyuru.
- **Anket / Oylama:** Toplantı tarihi belirleme, gizli oylama (kurullar için; ancak resmi oylama yerine geçmez).
- **Zamanlanmış mesaj.**
- **Toplantı gündemi ve karar notu** (kurul grupları için şablon).
- **Paylaşılan gelen kutusu:** "Öğrenci İşleri" gibi birim hesabına gelen mesajları birden fazla personel cevaplar, atama/çözüldü durumu.
- **Akademik takvim entegrasyonu:** Dönem başında grup oluşumu, sınav haftası hatırlatmaları.
- **Bot / Entegrasyon API'si:** ÖBS'den "notunuz girildi" bildirimi, kütüphane hatırlatmaları vb.
- **Acil durum yayını:** Deprem/tahliye gibi durumlarda tüm kullanıcılara "rahatsız etme"yi aşan bildirim (yalnızca yetkili rol).

### 10.3 Sonraki Aşamalar
- Sesli/görüntülü arama (WebRTC / Jitsi / LiveKit)
- Mesaj çevirisi (uluslararası öğrenciler için TR↔EN)
- Uçtan uca şifreleme (DM'lerde opsiyonel; bkz. 14.3)

---

## 11. Veri Modeli

### 11.1 Varlık İlişki Özeti

```
Person ─┬─< Affiliation >── OrgUnit ──< OrgUnit (parent)
        │        │
        │        └── Position
        ├─< CommitteeSeat >── Committee
        ├─< Enrollment >── CourseSection ── Course, Term
        └─< GroupMembership >── Conversation ──< Message ──< Attachment
                 │                    │
                 └─< MembershipSource └── GroupTemplate / MembershipRule
```

### 11.2 Tablo Taslakları (PostgreSQL)

```sql
-- ===== Kurumsal Yapı =====
CREATE TABLE org_unit_type (
  code        TEXT PRIMARY KEY,        -- FACULTY, ACADEMIC_DEPARTMENT ...
  label_tr    TEXT NOT NULL,
  label_en    TEXT NOT NULL
);

CREATE TABLE org_unit (
  id           UUID PRIMARY KEY,
  external_id  TEXT,                   -- İK/ÖBS'deki kod
  source       TEXT NOT NULL,          -- HR_SYNC | SIS_SYNC | MANUAL
  type_code    TEXT NOT NULL REFERENCES org_unit_type(code),
  parent_id    UUID REFERENCES org_unit(id),
  name_tr      TEXT NOT NULL,
  name_en      TEXT,
  short_name   TEXT,
  path         LTREE NOT NULL,         -- hızlı alt-ağaç sorgusu
  status       TEXT NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE | CLOSED | MERGED
  merged_into  UUID REFERENCES org_unit(id),
  valid_from   DATE, valid_until DATE,
  UNIQUE (source, external_id)
);
CREATE INDEX ON org_unit USING GIST (path);

CREATE TABLE person (
  id             UUID PRIMARY KEY,
  sso_subject    TEXT UNIQUE,          -- SSO'dan gelen benzersiz kimlik
  email          TEXT UNIQUE,
  display_name   TEXT NOT NULL,
  title_prefix   TEXT,                 -- Prof. Dr. vb.
  status         TEXT NOT NULL,        -- ACTIVE | ON_LEAVE | FROZEN | GRADUATED | LEFT | SUSPENDED
  locale         TEXT DEFAULT 'tr',
  created_at     TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE position (
  code        TEXT PRIMARY KEY,        -- DEKAN, BOLUM_BASKANI, PROF, ARS_GOR ...
  kind        TEXT NOT NULL,           -- ACADEMIC_TITLE | ADMIN_ROLE | STUDENT_LEVEL
  label_tr    TEXT NOT NULL,
  rank        INT                      -- sıralama/kıdem
);

CREATE TABLE affiliation (
  id            UUID PRIMARY KEY,
  person_id     UUID NOT NULL REFERENCES person(id),
  org_unit_id   UUID NOT NULL REFERENCES org_unit(id),
  kind          TEXT NOT NULL,         -- ACADEMIC_STAFF | ADMIN_STAFF | STUDENT | MANAGER | GUEST
  position_code TEXT REFERENCES position(code),
  is_primary    BOOLEAN DEFAULT false,
  attributes    JSONB DEFAULT '{}',    -- class_year, degree, entry_year, program_lang ...
  valid_from    DATE NOT NULL,
  valid_until   DATE,
  source        TEXT NOT NULL,
  external_id   TEXT
);
CREATE INDEX ON affiliation (org_unit_id) WHERE valid_until IS NULL;
CREATE INDEX ON affiliation USING GIN (attributes);

CREATE TABLE committee (
  id            UUID PRIMARY KEY,
  name          TEXT NOT NULL,
  kind          TEXT NOT NULL,         -- STATUTORY | PERMANENT | TEMPORARY
  owner_unit_id UUID REFERENCES org_unit(id),
  decision_ref  TEXT,                  -- görevlendirme/karar sayısı
  valid_from    DATE, valid_until DATE
);

CREATE TABLE committee_seat (
  committee_id  UUID REFERENCES committee(id),
  person_id     UUID REFERENCES person(id),
  seat_role     TEXT,                  -- BASKAN | UYE | RAPORTOR | SEKRETER
  valid_from    DATE, valid_until DATE,
  PRIMARY KEY (committee_id, person_id, valid_from)
);

-- ===== Akademik (ÖBS kaynaklı) =====
CREATE TABLE term (id TEXT PRIMARY KEY, starts_on DATE, ends_on DATE);  -- 2026-FALL
CREATE TABLE course_section (
  id UUID PRIMARY KEY, term_id TEXT REFERENCES term(id),
  course_code TEXT, section TEXT, org_unit_id UUID REFERENCES org_unit(id),
  UNIQUE (term_id, course_code, section)
);
CREATE TABLE enrollment (
  section_id UUID REFERENCES course_section(id),
  person_id  UUID REFERENCES person(id),
  role       TEXT,                     -- STUDENT | INSTRUCTOR | ASSISTANT
  status     TEXT,                     -- ACTIVE | DROPPED
  PRIMARY KEY (section_id, person_id, role)
);

-- ===== Grup Kurulumu =====
CREATE TABLE group_template (
  id          UUID PRIMARY KEY,
  code        TEXT UNIQUE NOT NULL,
  definition  JSONB NOT NULL,          -- bkz. 6.3
  enabled     BOOLEAN DEFAULT true,
  version     INT NOT NULL DEFAULT 1
);

CREATE TABLE membership_rule (
  id              UUID PRIMARY KEY,
  conversation_id UUID NOT NULL,
  expression      JSONB NOT NULL,      -- bkz. 7.2
  grants_role     TEXT NOT NULL DEFAULT 'MEMBER',
  created_by      UUID REFERENCES person(id),
  created_at      TIMESTAMPTZ DEFAULT now()
);

-- ===== Mesajlaşma =====
CREATE TABLE conversation (
  id            UUID PRIMARY KEY,
  kind          TEXT NOT NULL,         -- DIRECT | GROUP_DM | GROUP | CHANNEL | ANNOUNCEMENT | SPACE
  category      TEXT,                  -- FACULTY | COMMITTEE | COURSE_SECTION ...
  mode          TEXT NOT NULL,         -- AUTO | MANUAL | HYBRID
  key           TEXT UNIQUE,           -- auto:dept:{id}:academic (otomatikler için)
  template_id   UUID REFERENCES group_template(id),
  org_unit_id   UUID REFERENCES org_unit(id),     -- bağlı birim (varsa)
  committee_id  UUID REFERENCES committee(id),
  space_id      UUID REFERENCES conversation(id),
  name          TEXT,
  description   TEXT,
  settings      JSONB NOT NULL DEFAULT '{}',      -- posting, allow_leave, history ...
  state         TEXT NOT NULL DEFAULT 'ACTIVE',   -- ACTIVE | READ_ONLY | SUSPENDED | ARCHIVED
  expires_at    TIMESTAMPTZ,
  created_by    UUID REFERENCES person(id),
  created_at    TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE group_membership (
  conversation_id UUID REFERENCES conversation(id),
  person_id       UUID REFERENCES person(id),
  role            TEXT NOT NULL,       -- kaynaklardan hesaplanan en yüksek rol
  joined_at       TIMESTAMPTZ DEFAULT now(),
  last_read_seq   BIGINT DEFAULT 0,
  notify_level    TEXT DEFAULT 'ALL',  -- ALL | MENTIONS | NONE
  PRIMARY KEY (conversation_id, person_id)
);

CREATE TABLE membership_source (
  conversation_id UUID,
  person_id       UUID,
  source_type     TEXT NOT NULL,       -- RULE | TEMPLATE | MANUAL | EXCLUDE | SELF_JOIN
  source_ref      TEXT,                -- rule_id / added_by
  role            TEXT NOT NULL DEFAULT 'MEMBER',
  reason          TEXT,
  valid_until     TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (conversation_id, person_id, source_type, source_ref)
);

CREATE TABLE message (
  id              BIGINT GENERATED ALWAYS AS IDENTITY,
  conversation_id UUID NOT NULL,
  seq             BIGINT NOT NULL,     -- konuşma içi sıra numarası
  sender_id       UUID NOT NULL,
  thread_root_id  BIGINT,
  kind            TEXT NOT NULL,       -- TEXT | FILE | SYSTEM | POLL | ANNOUNCEMENT
  body            TEXT,
  metadata        JSONB,
  client_msg_id   UUID,                -- idempotent gönderim
  created_at      TIMESTAMPTZ DEFAULT now(),
  edited_at       TIMESTAMPTZ,
  deleted_at      TIMESTAMPTZ,
  PRIMARY KEY (conversation_id, seq)
) PARTITION BY HASH (conversation_id);   -- veya zamana göre

CREATE TABLE announcement_ack (
  conversation_id UUID, message_seq BIGINT, person_id UUID,
  acked_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (conversation_id, message_seq, person_id)
);

-- ===== Denetim =====
CREATE TABLE audit_log (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  at          TIMESTAMPTZ DEFAULT now(),
  actor_id    UUID,                    -- null = sistem/senkron
  action      TEXT NOT NULL,           -- membership.added, group.archived, rule.updated ...
  target_type TEXT, target_id TEXT,
  before      JSONB, after JSONB,
  sync_run_id UUID
);

CREATE TABLE sync_run (
  id UUID PRIMARY KEY, connector TEXT, started_at TIMESTAMPTZ, finished_at TIMESTAMPTZ,
  status TEXT,                         -- SUCCESS | FAILED | HALTED_SAFETY | DRY_RUN
  stats JSONB                          -- {added: 120, removed: 4, ...}
);
```

### 11.3 Etkin Rolün Hesaplanması

```
effective_role(person, conversation) =
    if EXISTS source EXCLUDE (aktif)        → üye değil
    else max_role( aktif RULE/TEMPLATE/MANUAL/SELF_JOIN kaynakların rolleri )
    else                                     → üye değil
```

`group_membership` tablosu bu hesabın **önbelleğidir** (materialized); mesaj iletiminde hızlı okuma için kullanılır, kaynak tablo değiştiğinde yeniden hesaplanır.

---

## 12. Sistem Mimarisi

### 12.1 Yapı Seçenekleri

| Seçenek | Artı | Eksi | Değerlendirme |
|---|---|---|---|
| **A) Hazır açık kaynak** (Matrix/Synapse, Rocket.Chat, Mattermost, Zulip) | Mesaj çekirdeği, istemciler hazır | Kurumsal grup/kural katmanı yok; derin özelleştirme zor; UI tutarlılığı | Pilot için hızlı, uzun vadede kısıtlayıcı |
| **B) Hazır çekirdek + özel Org/Provisioning servisi** | Mesajlaşma çözülmüş, değer katan kısma odaklanılır | İki sistemin senkronu, hazır ürünün sınırları | **Orta yol** — ör. Matrix + özel "Provisioner" (Matrix Application Service) |
| **C) Tamamen özel** | Tam kontrol, tek veri modeli, kurumsal UX | En uzun geliştirme süresi; istemciler sıfırdan | Ekip ve süre yeterliyse **en iyi uzun vadeli ürün** |

**Öneri:** Ekip kapasitesine göre:
- Ekip küçükse ve 3–4 ayda pilot gerekiyorsa → **B** (Matrix + özel Org/Provisioning servisi). Matrix, federasyonu kapatılarak kurum içi kullanılabilir, Element istemcisi markalanabilir.
- Uzun vadeli bir ürün hedefleniyorsa → **C**, aşağıdaki mimariyle. Bu dokümandaki Org/Grup katmanı her iki seçenekte de **aynıdır**; bu yüzden ilk yatırım o katmana yapılmalıdır.

### 12.2 Önerilen Mimari (Seçenek C)

```
                    ┌──────────── İstemciler ─────────────┐
                    │  Mobil (Flutter / React Native)     │
                    │  Web (React/Next.js)  Masaüstü      │
                    └──────┬────────────────────┬─────────┘
                       HTTPS (REST/GraphQL)   WebSocket
                           │                    │
                    ┌──────▼────────────────────▼─────────┐
                    │      API Gateway / Load Balancer      │
                    │  (TLS, rate limit, SSO token kontrol) │
                    └──────┬────────────────────┬──────────┘
          ┌────────────────┼────────────────────┼─────────────────┐
          │                ▼                    ▼                 │
          │  ┌──────────────────┐   ┌────────────────────────┐    │
          │  │  API (Modüler    │   │ Realtime Gateway       │    │
          │  │  Monolit)        │   │ (WebSocket bağlantıları,│   │
          │  │  - auth          │   │  presence, typing)     │    │
          │  │  - directory     │   └──────────┬─────────────┘    │
          │  │  - groups        │              │                  │
          │  │  - messaging     │◀── Pub/Sub ──┘                  │
          │  │  - policy        │   (Redis Streams / NATS)        │
          │  │  - announcements │                                 │
          │  │  - admin         │                                 │
          │  └───┬──────┬───────┘                                 │
          │      │      │       ┌───────────────────────┐         │
          │      │      └──────▶│ Worker'lar             │         │
          │      │              │ - provisioning/sync    │         │
          │      │              │ - push bildirim        │         │
          │      │              │ - arama indeksleme     │         │
          │      │              │ - dosya tarama/önizleme│         │
          │      │              └───────────────────────┘         │
          │      ▼                                                │
          │  PostgreSQL  ·  Redis  ·  OpenSearch  ·  MinIO (S3)    │
          └───────────────────────────────────────────────────────┘
                          ▲                ▲
                  Connector'lar      APNs / FCM (push)
              (İK, ÖBS, LDAP, SSO)
```

### 12.3 Teknoloji Önerileri

| Katman | Öneri | Alternatif |
|---|---|---|
| Backend | **Go** veya **TypeScript (NestJS)** | Elixir/Phoenix (gerçek zamanlıda çok güçlü), Kotlin/Spring |
| Gerçek zamanlı | WebSocket + Redis Pub/Sub veya NATS | Centrifugo (hazır WS sunucusu) |
| Veritabanı | PostgreSQL 16+ (`ltree`, JSONB, partition) | — |
| Kuyruk / iş | NATS JetStream veya Redis Streams; zamanlanmış işler için Temporal/River/BullMQ | RabbitMQ, Kafka (bu ölçekte fazla) |
| Arama | OpenSearch (Türkçe analyzer) | PostgreSQL FTS (başlangıç için yeterli) |
| Dosya | MinIO (S3 uyumlu) + ClamAV | — |
| Mobil | **Flutter** (tek kod tabanı, iOS+Android+masaüstü) | React Native |
| Web | React + TypeScript | — |
| Push | FCM + APNs (içerik **gönderilmez**, yalnızca "yeni mesaj" sinyali — gizlilik) | — |
| Kimlik | Üniversite SSO (CAS/SAML/OIDC) → Keycloak aracı olarak | — |
| Gözlemlenebilirlik | Prometheus + Grafana + Loki + OpenTelemetry | — |
| Dağıtım | Docker + Kubernetes (veya başlangıçta Docker Compose) | — |

### 12.4 Mesaj İletim Akışı

1. İstemci `client_msg_id` ile mesajı gönderir (tekrar gönderimde çift kayıt oluşmaz).
2. API: kimlik → **politika kontrolü** (yazma yetkisi, grup durumu, iletişim politikası).
3. Konuşmaya `seq` atanır, PostgreSQL'e yazılır.
4. Pub/Sub'a olay yayılır → Realtime Gateway çevrimiçi üyelere iletir.
5. Çevrimdışı üyeler için push worker → bildirim tercihleri, sessiz saatler, `@mention` kontrolü.
6. Arama worker'ı indeksler.
7. İstemci yeniden bağlandığında `last_seq`'ten sonrasını çeker (senkronizasyon tutarlılığı).

---

## 13. Entegrasyonlar

| Sistem | Ne alınır? | Yöntem | Sıklık |
|---|---|---|---|
| **SSO** (CAS/SAML/OIDC, LDAP/AD) | Kimlik doğrulama, temel kullanıcı bilgisi | OIDC/SAML | Her girişte |
| **İK / Personel Sistemi** | Personel, birim, kadro, idari görev, izin durumu | REST API / DB view / gece CSV | Olay + gece |
| **ÖBS** | Öğrenci, program, sınıf, ders şubeleri, kayıtlar, danışman | REST API / DB view | Saatlik (kayıt dönemi), günlük |
| **EBYS** | (ileride) Komisyon görevlendirme yazıları → komisyon oluşturma | API | Olay |
| **LMS** (Moodle vb.) | Ders şubesi eşleşmesi, derin bağlantı | LTI / API | — |
| **Takvim** (Exchange/Google) | Toplantı daveti, durum ("toplantıda") | Graph API / CalDAV | — |
| **E‑posta** | Okunmamış mesaj özeti, davet | SMTP | — |

**Connector tasarımı:** Her kaynak için bir adaptör; ortak bir **normalize edilmiş şemaya** (Person, OrgUnit, Affiliation, Course, Enrollment) dönüştürür. Böylece üniversite İK sistemi değiştirdiğinde yalnızca adaptör yeniden yazılır.

> Türkiye'deki üniversitelerde İK/ÖBS genellikle yerli yazılımlardır ve çoğu zaman yalnızca veritabanı görünümü (view) veya web servis (SOAP) sunar. İlk adım, Bilgi İşlem ile **hangi verinin hangi formatta** alınabileceğinin netleştirilmesidir (bkz. Açık Sorular).

---

## 14. Güvenlik, KVKK ve Uyumluluk

### 14.1 KVKK
- **Aydınlatma metni** ve ilk girişte gösterim; açık rıza gerektiren işlemler (ör. profil fotoğrafı, konum) ayrıştırılmalı.
- **Veri minimizasyonu:** ÖBS/İK'dan yalnızca gruplama için gereken alanlar alınır (T.C. kimlik no, adres, not bilgisi **alınmaz**).
- **Veri yerelliği:** Sunucular Türkiye'de (kampüs veri merkezi veya yurtiçi bulut). Push servisleri (FCM/APNs) yurtdışında olduğu için push içeriğine **mesaj metni konmaz**.
- **Saklama süreleri** (politika ile yapılandırılabilir): ör. DM 2 yıl, ders grupları dönem + 1 yıl, kurul grupları 10 yıl, denetim kaydı 2 yıl.
- **Silme / unutulma hakkı:** Hesap kapanışında kişisel veriler anonimleştirilir; kurumsal saklama yükümlülüğü olanlar hariç.
- **VERBİS** kaydının güncellenmesi.

### 14.2 Güvenlik
- SSO + zorunlu **MFA** (personel için), oturum ve cihaz yönetimi, uzaktan oturum kapatma
- Aktarımda TLS 1.3, depolamada şifreleme (disk + hassas alanlar)
- Dosyalar için virüs taraması, tür/boyut kısıtlaması, imzalı ve süreli indirme bağlantıları
- Rate limiting, spam/kötüye kullanım tespiti, şikâyet (raporlama) mekanizması
- Rol tabanlı yönetim paneli; tüm yönetici işlemleri **denetim kaydı**na düşer
- Düzenli sızma testi, bağımlılık taraması
- Mobilde: ekran görüntüsü kısıtı (opsiyonel, hassas gruplar), jailbreak/root uyarısı, uygulama kilidi (PIN/biyometri)

### 14.3 Uçtan Uca Şifreleme (E2EE) Kararı
Kurumsal ortamda E2EE bir **ödünleşimdir**:
- ✅ Gizlilik maksimum
- ❌ Sunucu tarafı arama, yasal saklama (legal hold), disiplin soruşturmasında delil, cihaz kaybında geçmiş kurtarma zorlaşır

**Öneri:** İlk sürümde **sunucu tarafı şifreleme** (encryption at rest) + sıkı erişim denetimi. Mesaj içeriğine erişim yalnızca resmi süreçle (yazılı talep, iki yetkili onayı, denetim kaydı). E2EE ileride yalnızca DM'lerde opsiyonel olarak değerlendirilebilir.

### 14.4 Denetim
Kayıt altına alınacaklar: giriş/çıkış, yönetici işlemleri, grup oluşturma/silme, üyelik değişiklikleri (kaynağıyla), kural değişiklikleri, senkron çalışmaları, dışa aktarma, içerik erişim talepleri.

---

## 15. Ölçek ve Performans Hedefleri

### 15.1 Kapasite Varsayımları (orta–büyük devlet/vakıf üniversitesi)

| Metrik | Değer |
|---|---|
| Toplam kullanıcı | 30.000 (25.000 öğrenci, 5.000 personel) |
| Eşzamanlı bağlantı (zirve) | ~8.000–10.000 |
| Günlük mesaj | 300.000 – 1.000.000 |
| Toplam grup | ~5.000 (ders şubeleri dönem başına ~2.500) |
| En büyük grup | Tüm üniversite (30.000) — yalnızca duyuru |

### 15.2 Büyük Gruplar İçin Tasarım
- 1.000+ üyeli gruplar varsayılan olarak `ANNOUNCEMENT` veya "yazma kısıtlı" olmalı.
- Büyük gruplarda okundu bilgisi kişi bazında gönderilmez; yalnızca **toplu sayaç** (duyurularda onay hariç).
- "Fan‑out on read": büyük kanallarda mesaj her kullanıcının kutusuna kopyalanmaz; kullanıcı kanalın `seq`'ini okur.
- Senkronizasyonda büyük üyelik değişiklikleri parti (batch) halinde işlenir; sistem mesajı "120 kişi eklendi" şeklinde **tek** mesaj olur, 120 ayrı mesaj değil.

### 15.3 Hedefler (SLO)
- Mesaj iletim gecikmesi (p95): < 300 ms (kampüs içi)
- Erişilebilirlik: %99,5 (dönem başı ve sınav haftalarında kritik)
- Görev değişikliğinin gruplara yansıması: olay bazlı < 15 dk, en geç gece reconcile

---

## 16. Uç Durumlar (Edge Cases)

| Senaryo | Çözüm |
|---|---|
| Kişi hem Arş. Gör. hem doktora öğrencisi | İki ayrı görevlendirme; hem personel hem öğrenci gruplarında. Politika: personel yetkileri geçerli |
| Çift anadal / yandal öğrencisi | Her program için ayrı görevlendirme → her iki bölüm grubu |
| Başka fakülteden ders veren hoca | Ders şubesi grubuna enrollment'tan girer; o fakültenin personel grubuna girmez |
| Bir kişinin iki farklı birimde idari görevi var | İki görevlendirme, iki grupta da yönetici |
| Bölüm başkanı vekili (geçici) | Bitiş tarihli MANUAL rol kaynağı veya İK'da vekâlet kaydı |
| Dekan değişti | Eski dekanın `MANAGER` görevlendirmesi biter → kurallar yöneticiliği yeni dekana devreder; grup sahipliği de kuraldan geliyorsa otomatik |
| İK sistemi hatalı/boş veri gönderdi | Güvenlik freni (8.2) → senkron durur, yönetici onayı |
| Bölüm kapatıldı ama öğrencileri mezun olana kadar devam ediyor | Birim `CLOSING` durumu: yeni öğrenci yok, mevcut gruplar devam |
| Fakülte birleşmesi / bölüm başka fakülteye taşındı | `org_unit.parent_id` değişir → `path` güncellenir → üst seviye gruplar reconcile ile yeniden hesaplanır |
| Komisyonda dış üye (başka üniversiteden) | `GUEST` hesap, süreli, yalnızca o grupta görünür, rehbere erişim yok |
| Erasmus gelen öğrenci | ÖBS'de varsa otomatik; yoksa süreli GUEST |
| Aynı isimli iki kişi | Rehberde birim/unvan gösterilerek ayrıştırma |
| Manuel eklenmiş kişiyi kural da istiyor, sonra kural düşüyor | Kaynak modeli sayesinde MANUAL kaynak kaldığı için grupta kalır |
| Yönetici, kuralın eklediği kişiyi çıkarmak istiyor | Silinemez; yerine **EXCLUDE** kaydı oluşturulur (gerekçe zorunlu, denetime düşer) |
| Kullanıcı zorunlu gruptan çıkmak istiyor | Çıkamaz; ancak bildirimleri kapatabilir/arşivleyebilir |
| Ders şubesi birleştirildi (ÖBS) | Eski şube grubu `READ_ONLY` + yönlendirme mesajı |

---

## 17. API Taslağı

```
# Kimlik
POST   /auth/sso/callback
POST   /auth/refresh
DELETE /auth/sessions/{id}

# Rehber
GET    /directory/people?q=&unit=&type=
GET    /directory/people/{id}
GET    /directory/units/tree
GET    /directory/units/{id}/members

# Konuşmalar
GET    /conversations?category=&space=
POST   /conversations                         # manuel grup
GET    /conversations/{id}
PATCH  /conversations/{id}                    # ad, açıklama, ayarlar
POST   /conversations/{id}/archive
GET    /conversations/{id}/members
POST   /conversations/{id}/members            # manuel ekleme {person_ids, role, valid_until, reason}
DELETE /conversations/{id}/members/{pid}      # manuel kaynağı kaldırır
POST   /conversations/{id}/exclusions         # kural üyesini hariç tut
GET    /conversations/{id}/members/{pid}/why  # üyelik açıklaması (kaynaklar)

# Kurallar
GET    /conversations/{id}/rules
POST   /conversations/{id}/rules
POST   /rules/preview                         # {expression} → {count, sample[]}

# Mesajlar
GET    /conversations/{id}/messages?after_seq=&limit=
POST   /conversations/{id}/messages           # {client_msg_id, body, thread_root?}
PATCH  /conversations/{id}/messages/{seq}
DELETE /conversations/{id}/messages/{seq}
POST   /conversations/{id}/messages/{seq}/ack # resmi duyuru onayı
GET    /conversations/{id}/messages/{seq}/ack-report

# Yönetim
GET    /admin/templates
PUT    /admin/templates/{code}
POST   /admin/sync/{connector}/run?dry_run=true
GET    /admin/sync/runs/{id}
POST   /admin/sync/runs/{id}/approve          # güvenlik freni onayı
GET    /admin/audit?actor=&action=&from=&to=
GET    /admin/policies
PUT    /admin/policies/{code}

# Gerçek zamanlı (WebSocket olayları)
message.created | message.updated | message.deleted
conversation.updated | membership.changed
typing | presence | read.updated
```

---

## 18. Yol Haritası

### Faz 0 — Keşif (2–3 hafta)
- Bilgi İşlem ile İK/ÖBS/SSO veri erişimi netleştirme
- Organizasyon ağacının çıkarılması ve doğrulanması
- Politika kararları (öğrenci–personel iletişimi, saklama süreleri) için yönetim onayı
- Teknoloji kararı (Seçenek B mi C mi)

### Faz 1 — Çekirdek / MVP (8–12 hafta)
- SSO girişi, kurumsal rehber
- İK connector'ı → birim ve personel senkronu
- Otomatik gruplar: Tüm Personel, Birim, Fakülte, Bölüm akademik/idari
- Manuel gruplar: komisyonlar
- DM, grup mesajlaşma, dosya, bildirim, temel arama
- Web + mobil (iOS/Android)
- Yönetim paneli: grup listesi, üyelik açıklaması, denetim kaydı
- **Pilot:** Rektörlük + 1 fakülte personeli

### Faz 2 — Öğrenciler ve Hibrit (6–8 hafta)
- ÖBS connector'ı → öğrenciler, sınıflar, ders şubeleri, danışmanlar
- İletişim politikaları (ABAC)
- Hibrit gruplar (kanuni kurullar), EXCLUDE, süreli manuel üyelik
- Duyuru kanalları + okundu/onay raporu
- Güvenlik freni, dry‑run, senkron raporları
- **Yaygınlaştırma:** tüm personel + pilot fakülte öğrencileri

### Faz 3 — Kurumsal Özellikler (sürekli)
- Paylaşılan gelen kutusu (birim hesapları)
- Anket, zamanlanmış mesaj, toplantı notu şablonları
- Bot/entegrasyon API'si, LMS/takvim entegrasyonu
- Sesli/görüntülü görüşme
- Gelişmiş arama, mesaj çevirisi
- Tüm üniversite yaygınlaştırma

---

## 19. Riskler

| Risk | Etki | Önlem |
|---|---|---|
| İK/ÖBS verisi eksik/hatalı (birim ağacı güncel değil) | Yanlış gruplar, güven kaybı | Faz 0'da veri kalitesi analizi; yönetim panelinde manuel düzeltme katmanı (OrgUnit override) |
| Benimsenme düşük (WhatsApp alışkanlığı) | Proje başarısız | Üst yönetim sponsorluğu, resmi duyuruların buraya taşınması, mobil deneyimin kaliteli olması |
| Bildirim yorgunluğu (çok fazla otomatik grup) | Kullanıcı uygulamayı sessize alır | Varsayılan bildirim seviyeleri, SPACE yapısı, büyük gruplarda yazma kısıtı |
| Öğrenci–hoca iletişiminde mesai dışı baskı | Personel memnuniyetsizliği | Sessiz saatler, "ofis saatleri" durumu, politika |
| KVKK ihlali | Hukuki/itibari risk | Veri minimizasyonu, DPO onayı, denetim kaydı |
| Senkron hatasıyla toplu üyelik silinmesi | Kriz | Güvenlik freni, dry‑run, geri alma (audit'ten geri yükleme) |
| Dönem başı yük patlaması | Kesinti | Yük testi, otomatik ölçekleme, senkronların yayılması |

---

## 20. Açık Sorular

Projeye başlamadan **karar verilmesi gerekenler**:

1. **Veri kaynakları:** İK ve ÖBS hangi yazılımlar? API/DB view/CSV hangisi mümkün? Gerçek zamanlı değişiklik bildirimi var mı?
2. **SSO:** Üniversitede CAS, SAML (Shibboleth/EduGAIN), Azure AD veya LDAP hangisi kullanılıyor?
3. **Öğrenciler ilk sürümde dahil mi?** Yoksa önce yalnızca personel mi?
4. **Öğrenci → akademisyen DM** serbest mi, sadece ders hocası/danışmana mı, yoksa hiç mi?
5. **Ders şubesi grupları** LMS (Moodle/Blackboard) ile çakışıyor mu? İstenen mi?
6. **Kanuni kurulların** üye listesi nereden gelecek? (İK'da kurul üyeliği tutuluyor mu, yoksa tamamen manuel mi?)
7. **Saklama süreleri** ve yasal saklama gereksinimleri neler? Hukuk Müşavirliği görüşü?
8. **Barındırma:** Kampüs veri merkezi mi, yurtiçi bulut mu?
9. **Mezunlar** sistemde kalacak mı?
10. **Ekip:** Kaç geliştirici, hangi teknolojilerde deneyimli? (Seçenek B/C kararını doğrudan etkiler.)
11. **Marka/isim:** Uygulamanın adı, kurumsal kimlik kuralları.
12. **Dil:** Türkçe + İngilizce arayüz zorunlu mu? (Uluslararası öğrenci/personel)

---

### Ek A — Terimler Sözlüğü

| Terim | Anlamı |
|---|---|
| OrgUnit | Organizasyon birimi (fakülte, bölüm, daire başkanlığı vb.) |
| Affiliation | Kişinin bir birimle zamana bağlı ilişkisi (kadro, görev, öğrencilik) |
| Template | Organizasyon ağacından otomatik grup üreten şablon |
| Rule | Grup üyeliğini belirleyen dinamik kural |
| Membership Source | Bir üyeliğin neden var olduğu (kural, manuel, hariç tutma) |
| Reconcile | Olması gereken durum ile mevcut durumu karşılaştırıp farkı uygulama |
| Space | Birden fazla grubu bir çatı altında toplayan alan |
| Circuit breaker / Güvenlik freni | Anormal büyüklükte değişikliklerde senkronu durduran mekanizma |
