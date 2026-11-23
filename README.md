[README.md](https://github.com/user-attachments/files/35265245/README.md)
# DengeTabak — QR geri bildirimi ve izlenebilir etki hesabı

20 Kasım 2026 tarihinde bu görev kapsamında geliştirilen yerel yarışma demosu. Gerçek kampüs pilotu, gözlenmiş tasarruf veya 100 puan garantisi değildir. Kamera ve tartı kullanılmaz.

## Çalıştırma

ZIP'i tamamen çıkarın. `index.html` dosyasını Chrome veya Edge ile açın. İnternet, kurulum, API anahtarı veya kütüphane indirmesi gerekmez. `core.js` ve `app.js` aynı klasörde kalmalıdır.

Tarayıcı yerel dosyalarda depolamayı engelliyorsa Python 3 ile bu klasörde `python -m http.server 8765 --bind 127.0.0.1` çalıştırın ve `http://127.0.0.1:8765/index.html` açın. Bu adres yalnızca aynı bilgisayardadır. Telefon bağlantısı için bu dosyanın güvenilir bir sunucuda yayımlanması ve ortak veri deposu gerekir; bu paket bunları yapmaz.

Kayıtlar tek tarayıcının localStorage alanında tutulur. Demo için uygundur; gerçek kullanıcılar arası veri paylaşımı, güvenilir yetkilendirme, sunucu taraflı tekilleştirme ve veri saklama/silme politikası sonraki pilotun işleridir. Tarayıcı verileri temizlenirse kayıtlar silinir. CSV/JSON dışa aktarımı alın.

## Canlı demo (5 dakika içinde)

1. Mutfak ekranında **Yeni öğün kaydı aç**. Aynı tepsinin yeni kullanımı için bu işlemi tekrarlayın.
2. Öğrenci ekranında porsiyon, kalan oranı ve neden seçip yanıt gönderin. Tekrar gönderimin engellendiğini gösterin.
3. Mutfak ekranında **Örnek veriyi yükle**. 20 başlangıç ve 20 deneme kaydı açıkça sentetik olarak yüklenir; düğme tekrar basılınca kopya oluşmaz.
4. Öneriyi inceleyip **Kontrollü denemeyi onayla**. Üretim planı ve zaman damgalı işlem kaydı oluşur. Bu plan gerçekleşmiş tasarruf değildir.
5. Etki hesabında gün sayısını 180'den 36'ya değiştirin: aynı yemek daha seyrek sunulunca sonuç 1080'den 216 kg'a düşer. Sistemin yıllık çalışma günü ayrı kaldığından sistem yükü otomatik düşmez.
6. Sonraki artığı 80 g yapın: olumsuz sonuç saklanmaz. Gıda karbon katsayısı boşken net karbon sayısı uydurulmadığını gösterin.
7. Bir sayfa raporunu yazdırın; yanıt CSV'sini ve işlem kaydını dışa aktarın.

QR'ın sabit adresi tepsiyi tanımlar, kişiyi tanımlamaz. Örnekteki `qr_ornek.svg`, aynı bilgisayardaki yerel sunucu adresine yönelir. Yeni öğün oturumu servis tarafında açılır. Dosya açıldığında tarayıcı adresi de `?tray=T025` parametresiyle sınanabilir. Gerçek telefon taraması için erişilebilir sunucu adresine göre QR yeniden üretilmelidir; bilgisayarın localhost adresi telefonda çalışmaz.

## Veri ve tekrar üretim

- `sources.json`: yayıncı, tarih, veri ailesi ve kapsam sınırları.
- `data/source_indicators.csv`: resmi yayınların elle aktarılmış, yuvarlanmış özet göstergeleri. Tam veri tabanı veya mikro veri indirmesi değildir.
- `data/scenario_assumptions.csv`: tüm kampüs senaryosu varsayımları; resmi verilerle karıştırılmaz.
- `analysis.py`: Python 3 standart kütüphanesiyle gösterge değişimlerini ve senaryoları yeniden hesaplar: `python analysis.py`.
- `core.js`: tarayıcıdaki hesapların bağımsız çekirdeği. `node test_core.js` ile hesap/negatif sonuç/eksik değer/tekilleştirme kontrolleri çalışır.
- `DengeTabak-Proje-Raporu.pdf`: proje, veri analizi, pilot tasarımı, etki hesabı, kriter eşleştirmesi ve kaynakça.
- `Etki-Hesabi-Tek-Sayfa.pdf`: örnek senaryonun tek sayfalık raporu.

Kendi özgün uygulama kodu ve rapor metni MIT lisansıyla sağlanır. Harici veri setlerinin hakları yayıncılarına aittir; kaynak gösterimi yeniden kullanım lisansını kendiliğinden vermez. Kamuya açık yeniden yayımlama öncesi ilgili yayıncının lisans koşulları kontrol edilmelidir.

## Hesap kapsamı

Yıllık gıda etkisi = (önceki g/öğün − sonraki g/öğün − henüz içerilmeyen ek atık g/öğün) / 1000 × kapsamdaki öğün/kullanıcı sayısı × bu yemeğin yıllık servis sıklığı.

Gönüllü yanıtlar tartım değildir; temsili örneklem sayılmaz. Miktarlar reçete ve beyana dayalı tahmindir. Yanıt vermeyenleri sıfır atık saymayın. Yalnızca ankete katılanları tüm yemekhaneye otomatik genellemeyin. Ek servis sonraki miktara zaten dahilse tekrar düşmeyin. Üretim azaltımı, tabak artığı azaltımı ve maliyet karşılığını aynı fiziksel kazanç olarak iki kez toplamayın.

## Yarışma için kalan dış adımlar

Paket yerel Git geçmişinin taşınabilir kopyası olan `dengetabak-history.bundle` dosyasını içerir. `git clone dengetabak-history.bundle dengetabak-repo` komutuyla depo ve gerçek commit geçmişi açılabilir. Bu dosya bir GitHub/GitLab teslim bağlantısı değildir.

Kodun zamanını geriye çekmeyin; sahte commit geçmişi üretmeyin. Bu tarihteki gerçek çalışmayı ilk commit olarak kaydedin. Ekip bir Git barındırma hesabında depo oluşturup kodu göndererek jüriye depo bağlantısı sunmalıdır. README ve prototip teslim bağlantısı, en çok 8 slayt ve 1 sayfa etki hesabı hazırlanmalıdır. Yönergedeki son teslim 23 Kasım 2026 saat 11.15'tir. Jüri onayı, gerçek pilot verisi ve dış Git bağlantısı bu paketin oluşturulmasıyla gerçekleşmiş olmaz.
