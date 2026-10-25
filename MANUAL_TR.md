# Better RGB — Kullanım Kılavuzu (Türkçe)

Tongfang/ITE8291 tabanlı klavyeler (VID_048D, PID_600B) için özel per-key RGB kontrol uygulaması.

---

## 1. Üretici yazılımıyla birlikte nasıl çalışır (cihaz kilidi)

Bu uygulama, klavyenin USB RGB arayüzüyle doğrudan konuşuyor. Üreticinin kendi servisi ("GCUBridge" / Kontrol Merkezi) de aynı arayüzü kullanıyor ve — zor yoldan keşfedildiği üzere — kendi seçili aydınlatma durumunu belirli aralıklarla EC'ye yeniden göndererek başka bir programın ayarladığı her şeyin üzerine sessizce yazıyor.

Uygulama bunu sürücü seviyesinde çözüyor: klavyenin RGB arayüzünü **paylaşımsız (exclusive)** açıyor. Uygulama çalıştığı sürece başka hiçbir process (üretici servisi dahil) o arayüzü açamıyor; dolayısıyla aydınlatma ayarlarının üzerine hiçbir şey yazamıyor.

Pratikte bunun anlamı:

- **Artık Kontrol Merkezi'nde herhangi bir aydınlatma ayarı değiştirmen gerekmiyor.** Eski sürümler önce orada klavye aydınlatmasını kapatmanı istiyordu — o adım kalktı.
- **Üretici servisi normal şekilde çalışmaya devam ediyor.** Fan eğrileri, güç limitleri ve performans modları hiç etkilenmiyor; bu işlevler bu arayüzü kullanmıyor.
- **Uygulama çalışırken Kontrol Merkezi'nin klavye aydınlatma sayfası yüklenmez.** Bu kasıtlı — kilidin çalıştığının göstergesi. Uygulamayı kapattığında üreticinin aydınlatma kontrolleri tekrar kullanılabilir hale gelir.

**İlk çalıştırma:** üretici servisi genellikle boot sırasında, sen daha bir şey başlatamadan arayüzü kapar. Uygulama açıldığında kilidin kendisinde olmadığını tespit ederse, düzeltme öneren bir diyalog gösterir: üretici servisi birkaç saniyeliğine yeniden başlatılır (diğer işlevlerine dokunulmaz) ve uygulama kilidi almış olarak yeniden açılır. Windows bir kez yönetici onayı ister.

**Bunu tamamen otomatik hale getirmek için** **Auto-Start at Login** düğmesini kullan (Bölüm 4). Her oturum açılışında uygulamayı yönetici haklarıyla başlatan bir Zamanlanmış Görev kurar — UAC sorusu yok, diyalog yok; uygulama her girişte kilidi sessizce alır.

---

## 2. Kaynak koddan derleme

İhtiyacın olanlar:
- MSYS2 ve `mingw-w64` araç zinciri (gcc, windres)
- CMake ve Ninja

`hidapi`, `third_party/hidapi/` altında bir git submodule'üdür ve küçük ama
kritik bir yama taşır: `open_device` cihazı önce `dwShareMode = 0` (exclusive)
ile açmayı dener, ancak bu başarısız olursa orijinal paylaşımlı moda düşer.
Bölüm 1'deki cihaz kilidi bu yama olmadan çalışmaz, bu yüzden submodule'lerle
klonla:

```
git clone --recurse-submodules https://github.com/EpicYamo/BetterRGB.git
cd BetterRGB
cmake -B build -G "Ninja"
cmake --build build
```

Sonuç: `build/better-rgb.exe` — statik bağlanmış olduğu için ayrı DLL
gerektirmez.

---

## 3. Uygulamayı çalıştırma

`build\better-rgb.exe` dosyasına çift tıkla (ya da terminalden çalıştır). Herhangi bir konsol penceresi açılmaz — bu normal, pencereli bir uygulama.

- Uygulama cihaz kilidini zaten alabiliyorsa (örn. Auto-Start göreviyle başlatıldıysa ya da üretici servisi arayüzü bırakmışsa), sessizce açılır.
- Üretici servisi arayüzü elinde tutuyorsa, Bölüm 1'de anlatılan düzeltme diyaloğunu görürsün. **Evet** dersen bir UAC onayı, kısa bir servis yeniden başlatması olur ve uygulama kilitli şekilde yeniden açılır; UAC istemi iptal edilirse uygulama bir uyarı gösterip kapanır — kilitsiz devam etmez. **Hayır** dersen ikinci, açık bir uyarı çıkar: üretici servisiyle paylaşımlı modda çalışmak yalnızca aydınlatma ayarlarının üzerine yazılması riskini değil, yazma çakışması nedeniyle **klavyenin gömülü kontrolcüsünün (EC) kilitlenmesi** riskini de taşır — bu durumda klavye aydınlatması, tam bir güç sıfırlaması yapılana kadar hiçbir yazılıma yanıt vermez (bkz. Sorun Giderme). Uygulama, ancak bu uyarıyı da onaylarsan paylaşımlı modda devam eder; reddedersen kapanır.
- Uygulama yönetici haklarıyla başlatılmışsa ve arayüzün kapılmış olduğunu görürse, hiç diyalog göstermeden durumu kendisi düzeltir — Auto-Start görevinin her oturum açılışında yaptığı tam olarak budur.
- Auto-Start göreviyle başlatıldığında uygulama **sistem tepsisinde gizli** başlar — oturum açılışında pencere görünmez. Açmak için tray simgesine çift tıkla.

**Ayar hafızası:** tüm ayarlar — seçili mod (Stop dahil), parlaklık ve her efektin bütün parametreleri ile renkleri — exe'nin yanındaki `better_rgb.cfg` dosyasına otomatik kaydedilir (her mod değişiminde, Stop'a basılınca ve çıkışta/kapanışta). Bir sonraki açılışta her şey geri yüklenir ve son efekt kendiliğinden başlar; yani yeniden başlatma sonrası klavyen, oturum açtıktan saniyeler sonra dünkü düzeninle yanar. Dosyayı silmek uygulamayı varsayılanlara döndürür.

Uygulama ayrıca `rgb_engine_log.txt` adlı bir tanılama günlüğünü de yine exe'nin yanına yazar — bir sorun bildirirken bu dosyayı eklemek faydalıdır.

Kullanıcıya gösterilen tüm diyaloglar hem İngilizce hem Türkçe'dir.

---

## 4. Arayüz

- **Select Mode** — üstteki açılır kutu. Bir öğe seçmek o efekti anında başlatır ve altında sadece o efekte ait ayarları gösterir.
- **Stop (Turn Off Lights)** — mevcut efekti durdurur ve klavyeyi kapatır.
- **Brightness** — alttaki kaydırıcı, hangi mod aktif olursa olsun her zaman görünür. Genel çıktı parlaklığını (0-100%) ölçekler.
- **Active Mode** yazısı — şu an hangi efektin çalıştığını gösterir.
- **Auto-Start at Login** düğmesi — mevcut durumu (ON/OFF) gösterir ve değiştirir. AÇIK yapmak, her oturum açılışında uygulamayı yönetici haklarıyla başlatan bir Windows Zamanlanmış Görevi ("Better RGB by TheYamo") kurar; böylece cihaz kilidi her girişte UAC sorusu olmadan sessizce alınır, uygulama sistem tepsisinde gizli başlar ve son kullanılan ayarları uygular. Görev tam ayarlarla kaydedilir (pil gücünde de çalışır, çalışma süresi limiti yoktur). KAPALI yapmak görevi kaldırır. Görevin kurulması/kaldırılması işleminin kendisi bir kez yönetici onayı ister.

Pencereyi küçültmek, onu görev çubuğu yerine sistem tepsisine (saat yanındaki gizli simgeler alanı) gönderir — uygulama arka planda çalışmaya devam eder. Tray simgesine çift tıklayarak pencereyi geri getirebilir, ya da sağ tıklayarak Aç/Çıkış menüsüne ulaşabilirsin.

---

## 5. Modlar ve ayarları

### Breathing (Nefes Alma)
Tüm klavye tek bir renkte yanıp sönerek nefes alır.
- **Color** — "Pick Color" ile renk seç.
- **Speed** — nefes alma döngüsünün ne kadar hızlı olacağı.
- **Smoothness** — parlaklık geçişinin ne kadar kademesiz göründüğü. Yüksek değer = göz farkı algılamayacağın kadar pürüzsüz.

### Wave (Dalga)
Klavye üzerinde hareket eden bir renk geçişi.
- **Angle** — 0-359°. 0° soldan sağa, 90° yukarıdan aşağıya akar, diğer değerler çapraz akar. Sürüklerken tam dereceyi gösteren canlı bir balon çıkar.
- **Speed** — geçişin ne kadar hızlı hareket edeceği.

### Sparkle (Rastgele Parıltı)
Rastgele tuşlar kısa süreliğine parlar.
- **Speed** — her parıltının ne kadar sürede söneceği (yüksek hız = daha kısa ömürlü, daha çabuk parıltılar).
- **Density** — aynı anda kaç tuşun yanık olacağı (3-20), speed'den bağımsız.

### Reactive (Tuşa Basınca Tepki)
Bastığın tuşlar yanar, bıraktığında söner.
- **Color** — sabit bir renk seç, ya da "Random" ile her basışta farklı rastgele renk.
- **Duration** — Short / Medium / Long: tuşu bıraktıktan sonra ışığın ne kadar sürede (sönerek) kapanacağı. Not: tuş basılı tutulduğu sürece bu ayardan bağımsız olarak tam parlaklıkta kalır — Duration sadece bırakma sonrası sönme süresini etkiler.

### Wheel (Çark)
Klavyenin merkezi etrafında dönen bir gökkuşağı.
- **Reverse Direction** — işaretlenince saat yönü yerine tersine döner.
- **Speed** — dönüş hızı.

### Lightning (Şimşek)
Klavyenin üstünden altına doğru düşen parlak bir "şimşek".
- **Speed** — şimşeğin ne kadar hızlı aşağı ineceği.
- **Smoothness** — şimşeğin arkasında bıraktığı kuyruğun ne kadar uzun ve yumuşak söneceği. Yüksek değerler daha uzun, daha yumuşak sönümlenen bir kuyruk oluşturur (ve tamamen kaybolması da buna bağlı olarak biraz daha uzun sürer).
- **Strike Width** — 1 ila 4 tuş genişliğinde.
- **Concurrent Strikes** — aynı anda kaç bağımsız şimşeğin düşebileceği (1-4). Tam sayı, kaydırıcının yanında canlı olarak gösterilir.

### Flame (Alev)
Sürekli gezinen/kayan bir ateş simülasyonu, alt kısımlar daha sıcak.
- **Flicker Speed** — ısı deseninin ne kadar hızlı değişip yeniden rastgeleleşeceği.
- **Smoothness** — renklerin bir durumdan diğerine ne kadar kademesiz geçeceği (ani sıçrama yerine).

### Rain (Yağmur)
Klavyeden aşağı düşen renkli damlalar, kısa bir kuyruk izi bırakarak.
- **Fall Speed** — damlaların ne kadar hızlı düşüp animasyonunu tamamlayacağı.
- **Density** — 1 ila 5 kademe; her kademe 3 eş zamanlı damla ekler (1. kademe = 3 damla, 5. kademe = 15 damla).
- **Color Count** — 1 ila 5. Bir sayıya tıklayarak paletteki renk sayısını belirle, sonra beliren her renk kutusuna tıklayıp rengini ata. Her damla, doğduğu anda paletten rastgele seçilen tek bir renkte kalır (yaşam süresi boyunca değişmez).

### Matrix
Style açılır kutusundan seçilebilen 4 farklı animasyon stili:
- **Diagonal Streaks** — rastgele çapraz açılarla klavye üzerinde akan renkli bantlar.
- **Vertical Columns** — klasik "dijital yağmur" görünümü, her sütunda düz aşağı akış.
- **Laser Sweep** — 2-3 keskin, hızlı yatay/dikey çizgi geçişi.
- **Ripple** — rastgele noktalardan dışa doğru genişleyen halkalar (su dalgası gibi).

Ayarlar:
- **Streak Speed** — genel animasyon hızı.
- **Density** — Diagonal Streaks ve Laser Sweep stillerini etkiler: aynı anda kaç çizginin aktif olacağı.
- **Color Count** — 1 ila 3, Rain'deki ile aynı tıkla-ata sistemi.

### Static (Sabit Bölgeler)
Hareketsiz bir düzen: klavye, senin belirlediğin renkli bölgelere ayrılır.
- **Horizontal / Vertical** — bölme yönünü seç. Horizontal, yukarıdan aşağıya en fazla **6** yatay bant oluşturur; Vertical, yan yana en fazla **10** dikey bant yerleştirir.
- **Zones** — klavyenin kaç bölgeye ayrılacağını seçen kaydırıcı (üst sınır, seçilen yöne bağlıdır).
- **Bölge seçici + Pick Color** — açılır kutudan bir bölge seç, sonra rengini ata. Her bölge için tekrarla.

---

## 6. Sorun Giderme

**Windows uygulamayı engelliyor: "Windows protected your PC" ya da "This app may be unsafe".**
Çalıştırılabilir dosya imzasız (küçük bir açık kaynak proje, ücretli kod imzalama sertifikası yok) ve her derleme, hiçbir itibarı olmayan yepyeni bir dosya. **SmartScreen** uyarısında *More info → Run anyway* seçeneğine tıkla. Engelleyen **Smart App Control** ise (bazı yeni Windows 11 kurulumlarında varsayılan olarak açık gelir), maalesef uygulama bazlı istisna yok — SAC yalnızca genel olarak kapatılabilir (Ayarlar → Gizlilik ve güvenlik → Windows Güvenliği → Uygulama ve tarayıcı denetimi → Smart App Control → Kapalı; bir kez kapatılınca Windows yeniden kurulmadan tekrar açılamadığını unutma). Uygulamanın yaptığı servis yeniden başlatma, Zamanlanmış Görev kurma ve kendini yönetici olarak yeniden başlatma işlemleri, imzasız bir dosyayı bu filtrelere şüpheli gösteren şeylerin ta kendisi — bu davranışlar Bölüm 1'de belgelenmiştir ve kaynak kod açıktır; hazır exe'ye güvenmek istemezsen uygulamayı kendin de derleyebilirsin (Bölüm 2).

**Açılışta "Could not open the keyboard device" hatası.**
Desteklenen RGB arayüzü (VID_048D / PID_600B, ITE8291) bu makinede bulunamadı. Uygulama özel olarak bu kontrolcüyü hedefler; farklı bir klavye/kontrolcüde çalışamaz.

**Kontrol Merkezi'nin klavye aydınlatma sayfası yüklenmiyor.**
Bu uygulama çalışırken beklenen durum — cihaz kilidi (Bölüm 1), üretici yazılımının RGB arayüzünü açmasını engelliyor. Bu uygulamayı kapattığında üreticinin aydınlatma sayfası tekrar çalışır.

**Işıklar kendiliğinden sönüyor / yerine üreticinin rengi geliyordu (eski sürümler).**
Bu, üretici servisinin arayüze müdahale etmesinin tipik belirtisidir ve exclusive kilidin ortadan kaldırdığı şey tam olarak budur. Hâlâ görüyorsan, uygulama muhtemelen kilitsiz çalışıyordur (düzeltme diyaloğunu reddettin ya da servis arayüzü önce kaptı). Uygulamayı yeniden başlatıp düzeltmeyi kabul et, ya da kalıcı sessiz çözüm için Auto-Start at Login'i aç.

**Klavye ışıkları hem bu uygulamaya hem üretici yazılımına tepki vermemeye başlıyor.**
Klavyenin gömülü kontrolcüsü kilitlenmiş — bu bilinen bir donanımsal tuhaflık, veri bozulması ya da kalıcı hasar değil. Kurtarmak için:
1. Laptobu tamamen kapat.
2. Şarj kablosunu çıkar.
3. Güç düğmesine yaklaşık 30 saniye bas.
4. Şarjı tekrar tak, laptobu aç.

**Bir modun renk ayar kutuları, başka bir mod seçiliyken de ekranda görünüyor.**
Bu, önceki sürümlerde bir hataydı ve düzeltildi; hâlâ görüyorsan en güncel derlemeyi kullandığından emin ol.

**Reactive modda numpad Enter ile ana Enter tuşu birlikte yanıyor.**
Bilinen bir sınırlama — Windows ikisini de aynı sanal tuş koduyla bildiriyor, uygulama şu an bunları ayırt edemiyor.

**Fn tuşu ya da Copilot tuşu Reactive modda hiç yanmıyor.**
Bu da beklenen bir durum — bu tuşlar Windows'un algılayabileceği standart bir sinyal göndermiyor.