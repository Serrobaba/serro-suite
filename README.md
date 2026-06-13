# Serro Suite — Geliştirici Rehberi (Mimari & Kurallar)

> **Yeni bir AI ajanına / geliştiriciye:** Bu dosya projenin tüm
> mimarisini, kurallarını ve dosya haritasını içerir. Bir değişiklik
> yapmadan önce **mutlaka bu dosyanın tamamını oku**. Aşağıdaki
> kurallar trial-and-error ile öğrenilmiş, ihlal edilince sistem
> kırılan kurallardır.

## Proje Nedir
Blender 5.x için **Serro Suite**: kod bilmeyen bir 3D sanatçısı
(UE5 pipeline) için yapılan modüler bir eklenti paketi. Omurga +
eklemlenebilir modüller (export, material, bake, transform, drawer)
+ Qt tabanlı yüzen "dock" arayüzü + tema sistemi + tek-tuş güncelleme.

## Mutlak Kurallar (İhlal = Sistem Kırılır)

### 1. THREAD MODELİ — Qt asla ana thread'de çalışmaz
- Qt KENDİ thread'inde (`qt_host.py`, "SerroQt" daemon) gerçek
  `app.exec()` ile döner. Blender ana döngüsünde **SIFIR Qt kodu**.
- **DEMİR KURAL:** Qt thread'inde `bpy` YASAK; ana thread'de
  `QWidget` YASAK. İhlali kilitlenme/segfault demek.
- Köprü: `bpy.app.timers` ile `_bridge` (~7Hz). Blender→Qt işler
  `run_in_qt()`, Qt→Blender aksiyonlar `enqueue_command()`.
- Pencereler `_live_windows` listesinde GÜÇLÜ referansla tutulur
  (yoksa Python GC anında yok eder — "dock açılıp kayboluyor" bug'ı).

### 2. SEGMENT MİMARİSİ — her dosyanın TEK sorumluluğu
| Segment | Dosya | Sorumluluk |
|---|---|---|
| GÖRÜNÜM | `qt_theme.py` > `DEFAULT_QSS` | TEK tema kaynağı (QSS). Renk/font/boşluk. |
| MANTIK | `qt_layers.py` | Pencere/dock widget'ları, etkileşim. |
| ALTYAPI | `qt_host.py` | Thread, köprü, app yaşam döngüsü. DOKUNMA. |
| TEMA TERCİH | `theme_panel.py` | N-panel font seçici (kullanıcı). |
| GÜNCELLEME | `updater.py` | GitHub'dan sürüm çekme. |

### 3. TEK KAYNAK İLKESİ — tema tek yerde
- Tema SADECE `qt_theme.DEFAULT_QSS`'te tanımlı. Ayrı bir
  `serro_theme.qss` dosyası YOKTUR (vardı, iki-kaynak ayrışması
  font bug'ına yol açtı, silindi).
- Profil teması içerik HASH'i ile tazelenir (`ensure_theme_file`):
  `DEFAULT_QSS` değişince profildeki eski kopya otomatik güncellenir.
- Kullanıcı `<profil>/serro_theme.qss.user` oluşturursa O kutsaldır,
  güncellemeler dokunmaz.

### 4. SESSİZ YUTMA YASAK
- Her `try/except` bir hatayı yutuyorsa, o hata bir yerde GÖRÜNMELİ
  (panelde kırmızı satır / konsol traceback). "Neden çalışmıyor?"
  sorusunu karanlıkta bırakan kod yasak. (Geçmişte register çağrısı
  sessizce atlandı, koca sistem pakete girip hiç çağrılmadı.)
- Her yama uygulandıktan sonra `grep` ile DOĞRULANIR.

### 5. ÇOCUK GÜVENLİĞİ / KISITLI CONTEXT
- Tüm bpy operatör çağrıları 3D View `temp_override` ile sarılır
  (kısıtlı-context zırhı). `loader.load_default` ASLA True olamaz.

## Dosya Haritası (`serro_core/`)
- `__init__.py` — `SERRO_VERSION` (sürüm burada; 4 yerde bump edilir:
  bu + manifest + nsi + rc).
- `loader.py` — modül yükleyici (KONTRAT: `load_default=False`).
- `paths.py`, `config.py`, `state.py`, `core_api.py` — çekirdek.
- `qt_host.py` — **THREAD MODELİ** (v3). Kilitlenmenin kökten çözümü.
- `qt_layers.py` — dock + LayersWindow widget'ı (~1300 satır).
  `create_window` (panel), `create_dock` (DockShell çerçevesiz kabuk),
  `edge_at`/`clamp_pos` (saf, test edilir), `make_bridge`.
- `qt_theme.py` — tema motoru: `DEFAULT_QSS`, `load_fonts`,
  font tercihleri (`load_prefs`/`save_prefs`/`build_user_qss`).
- `theme_panel.py` — N-panel > Serro > Tema (segment font seçici).
- `updater.py` — GitHub Releases güncelleme.
- `qt_installer.py` — PySide6 indirme (profile pip --target).
- `settings.py` — Ayarlar paneli + register zinciri (her şeyi bağlar).
- `fonts/` — Inter, Space Grotesk, JetBrains Mono (OFL).
- `modules/` — eklemlenebilir modüller (her biri loader ile yüklenir):
  - `serro_export` (v0.4, UE5), `serro_material` (v0.3),
    `serro_bake` (v0.4), `serro_transform` (v0.9, Ölç/Origin/Slide),
    `serro_drawer` (Katmanlar motoru + Photoshop-layer adlandırma).

## Derleme Ritüeli
1. Sürümü 4 yerde artır: `__init__.py` SERRO_VERSION,
   `blender_manifest.toml` version, `serro_installer.nsi`
   PRODUCT_VERSION, `serro_launcher.rc` (iki format).
2. `__pycache__` temizle.
3. Launcher: windres + mingw gcc derle.
4. `makensis` ile installer.
5. **GitHub güncellemesi için:** `serro_core/` klasörünü
   `serro_core.zip` olarak paketle (kök: `serro_core/...`).
   updater bu yapıyı bekler.

## Test Disiplini
- `tests/test_qt.py` (offscreen, alt-süreçte `test_qt_widgets.py`
  koşturur), `test_loader.py`, `test_faz2.py`, `test_faz3.py`.
- Saf mantık (edge_at, clamp_pos, sürüm parse) ayrı test edilir.
- **FONT-GUARD** anti-regresyon testleri: iki-kaynak, reload,
  fallback. Bu testler bilerek kondu; kaldırma.
- Her teslim çift koşu (deterministik olduğunu kanıtlamak için).

## Kullanıcı Tercihleri (önemli bağlam)
- Türkçe konuşur, kod bilmez. Her teslimde adım adım BENIOKU +
  ekran görüntülü feedback ister. Tek zip tercih eder.
- Dürüst sınır bildirimi ve "neden"li kök analizi çok değer verir.
- Ortak dil: **SAHNE = ana koleksiyon** (scene.collection'ın
  doğrudan çocuğu). Koyu/mor (#7c5cff) sleek tema. Metre birimi,
  2 UV kanalı, RGBA packed bake.
- "Solid" ister (chrome düşmanı); dock "bize ait" hissetmeli.

## Kullanıcı Profilinde Tutulan Dosyalar (paketin DIŞINDA)
`<BLENDER_USER_RESOURCES>/` altında:
- `serro_theme.qss` (+`.v` damga) — aktif tema (otomatik yönetilir).
- `serro_theme.qss.user` — kullanıcı özel teması (varsa kutsal).
- `serro_theme.qss.prefs.json` — font tercih seçimleri.
- `serro_theme.qss.repo.json` — GitHub repo bilgisi (updater).
Bunlar güncellemeden ETKİLENMEZ (dizin dışında).

## Yapılacaklar (omurga dokümanından kalan)
- GeoNodes kütüphanesi (madde 6), JSON sahne transferi (madde 4),
  Q4 tema senkronu + Serro Workspace kompozisyonu, splash/autosave.
