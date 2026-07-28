# Finansal Pano

Kişisel gelir-gider takibi için tek dosyalı (`index.html`) bir web uygulaması. Bağımlılık yok, build adımı yok, kurulum yok — tarayıcıda açıp kullanmaya başlarsın.

## Özellikler

- **Aylık gelir & gider girişi** — her ay için ayrı gelir ve gider kalemleri
- **Tip sistemi** — özelleştirilebilir renk ve isimli tipler (Maaş, Kira, Market vb.)
- **Özet şerit** — toplam gelir, toplam gider ve kalan bakiye
- **Yıllık özet** — seçili yıldaki tüm aylar için kalan bakiye tablosu
- **Pasta grafikler** — gelir ve gider dağılımının tipe göre görselleştirilmesi
- **Çoklu ay girişi** — bir kalemi otomatik olarak N aya yayma
- **Ay klonlama** — önceki bir ayın verilerini aktif aya kopyalama
- **Sürükle & bırak** — kalemler liste içinde yeniden sıralanabilir
- **Mobil uyumlu** — hamburger menü ile drawer navigasyon (≤640px)
- **JSON dışa/içe aktarım** — verileri dosyaya kaydet veya dosyadan yükle
- **Uzak senkronizasyon** — JSONBin üzerinden GET/PUT ile yedekle ve geri yükle
- **localStorage** — veriler tarayıcıda otomatik saklanır

## Kullanım

`index.html` dosyasını herhangi bir tarayıcıda aç. Sunucu gerekmez.

```bash
open index.html
```

## Veri Yönetimi

| Buton | İşlev |
|-------|-------|
| ⬇ Dışa Aktar | Tüm veriyi `.json` dosyası olarak indirir |
| ⬆ İçe Aktar | JSON dosyasından veri yükler (mevcut veriyi siler) |
| ☁ Uzaktan Al | Uzak depodan veriyi çeker ve localStorage'ı günceller |
| ☁ Uzağa Gönder | Mevcut veriyi uzak depoya gönderir |
| 🗑 Temizle | localStorage'ı tamamen sıfırlar |

## Teknik

- Vanilla HTML5 / CSS3 / JavaScript
- Bağımlılık yok, build adımı yok
- Veri anahtarı: `localStorage['financial-board-v1']`
- Uzak depo: [JSONBin.io](https://jsonbin.io)
