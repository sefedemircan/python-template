# Meyer Excel → Aylık Puantaj Şablonu
## Hesaplama ve Kodlama Kural Seti

**Doküman amacı:** Meyer’den alınan kişi-gün bazlı Excel/CSV dosyasının yüklenmesi, puantaj hesaplarının yapılması ve sonucun aylık puantaj raporu veya kurumun mevcut bordro şablonu biçiminde üretilmesi için uygulanacak kuralları tanımlamak.  
**Kapsam:** Meyer puantaj dosyasının hesaplanması ve sonuçların hedef bordro şablonuna aktarılması.  
**Kapsam dışı:** Ücret, vergi, SGK primi ve net bordro tutarı hesabı.  
**Referans kod listesi:** `data/harf.jpg`.  
**Doküman statüsü:** Meyer geliştirme ekibiyle teknik ve fonksiyonel mutabakat için hazırlanmış uygulama kural seti.

---

## 1. Sistemden beklenen işlem

Kullanıcı Meyer’den aldığı Excel/CSV dosyasını sisteme yükler ve rapor dönemini seçer. Sistem:

1. Meyer dosyasındaki personel-gün kayıtlarını okur.
2. Tarih ve süre alanlarını standartlaştırır.
3. Her günü çalışma, izin, rapor, ücretsiz izin, devamsızlık, hafta tatili veya resmi tatil olarak sınıflandırır.
4. Kayıtları pazartesi-pazar ISO haftalarına ayırır.
5. Haftalık 45 saat kontrolünü ve gerekli `FM → NM` dönüşümünü yapar.
6. Mazeretsiz tam gün devamsızlığa göre pazar/hafta tatili hakkını belirler.
7. Her personel için günlük saat veya harf kodu üretir.
8. Aylık puantaj, personel özeti, haftalık kontrol ve günlük detay verilerini oluşturur.
9. Üretilen günlük değerleri kullanıcının yüklediği mevcut bordro şablonunun doğru gün ve personel hücrelerine yazar.
10. Resmi tatilde çalışılan saatleri haftalık 45 saati aşan/aşmayan bölümlere ayırır.
11. Doldurulmuş Excel dosyasını kullanıcıya sonuç olarak verir.

### 1.1. Girdi ve çıktı

| Girdi | Çıktı |
|---|---|
| Meyer Excel/CSV dosyası + rapor yılı/ayı + hedef bordro şablonu | Günlük kodları, haftalık hesapları ve manuel tatil alanları doldurulmuş bordro şablonu |

Günlük sınıflandırma ve haftalık hesaplama tamamlandıktan sonra sonuçlar mevcut şablondaki ilgili personel ve gün hücrelerine yerleştirilmelidir.

---

## 2. Meyer kaynak dosyası

### 2.1. Dosya formatı

- Desteklenen formatlar: `.xlsx`, `.xls` ve `;` ayraçlı `.csv`.
- Excel dosyasında ilk satır kolon başlıklarını içermelidir.
- Bir satır, bir personelin bir takvim gününü temsil etmelidir.
- CSV karakter kodlamasında Türkçe karakterler korunmalıdır.

### 2.2. Zorunlu alanlar

| Alan | Açıklama | Kural |
|---|---|---|
| `sicilno` | Personel sicil numarası | Metin olarak alınmalı; baştaki sıfırlar korunmalıdır. |
| `Ad` | Personel adı | Boş olmamalıdır. |
| `Soyad` | Personel soyadı | Boş olmamalıdır. |
| `mesaitarih` | Puantaj tarihi | Geçerli tarih olmalıdır. |
| `NM` | Normal mesai | Boşsa `0` kabul edilir. |
| `FM` | Fazla mesai | Boşsa `0` kabul edilir. |

### 2.3. Hesaplamada kullanılan diğer alanlar

| Alan | Kullanım amacı |
|---|---|
| `Firma`, `Bölüm`, `Pozisyon`, `Görev`, `Yaka` | Personel bilgisi ve raporlama |
| `Giriş`, `Çıkış` | Hareket kontrolü ve gerektiğinde resmi tatil çalışma süresi hesabı |
| `MS` | Günlük planlanan/beklenen mesai süresi |
| `IZS` | Ücretli izin süresi |
| `YIZS` | Yıllık izin süresi |
| `SGKIZS` | Sağlık raporu süresi |
| `UCZIZS` | Ücretsiz izin süresi |
| `RM` | Rapor/resmi tatil işareti olarak kullanılan Meyer alanı |
| `EM` | Eksik mesai/devamsızlık süresi |
| `İzin Açıklama` | İzin türünün metinden belirlenmesi |
| `RM Açıklama` | Resmi tatil ve dini bayram işaretlerinin belirlenmesi |
| `Mesai Açıklama` | Vardiya/mesai açıklaması ve kontrol bilgisi |
| `Kaynak Kod` | Varsa önceden onaylanmış puantaj kodu |
| `Hesaplanmış` | Aynı haftaya 45 saat dönüşümünün ikinci kez uygulanmasını önleyen işaret |

### 2.4. Tarih ve süre standardı

- Tarih `DD.MM.YYYY`, Excel tarihi veya ISO tarih olarak alınabilir; içeride gerçek tarih tipine çevrilmelidir.
- Saat/süre değerleri `HH:MM`, `HH:MM:SS`, Excel zaman değeri veya süre tipi olabilir.
- `null`, boş metin, `NaN` ve `#__#` sıfır/boş kabul edilir.
- Hesap içeride dakika veya ondalık saat üzerinden yapılabilir; çıktı en fazla iki ondalıkla yazılmalıdır.
- Negatif süre kabul edilmemelidir.
- Gece vardiyasında `Çıkış <= Giriş` ise çıkış ertesi gün kabul edilmelidir.

---

## 3. Veri hazırlama ve kontrol kuralları

### KRL-VERİ-001 – Geçersiz tarih

Geçerli tarihe çevrilemeyen satır hesaplamaya alınmaz ve “Geçersiz tarih” hatasıyla raporlanır.

### KRL-VERİ-002 – Boş sicil numarası

Sicil numarası bulunmayan satır hesaplamaya alınmaz ve “Boş sicil no” hatasıyla raporlanır.

### KRL-VERİ-003 – Mükerrer kişi-gün

Aynı `sicilno + mesaitarih` için birden fazla satır varsa:

- kayıt veri kalitesi raporunda mükerrer olarak gösterilir;
- mevcut sistem davranışında ilk satır esas alınır;
- süreler otomatik olarak toplanmaz;
- Meyer çıktısının kişi-gün bazında tekil kayıt üretmesi beklenir.

### KRL-VERİ-004 – Dönem seçimi

- Kullanıcı yıl ve ay seçer.
- Aylık matris ve personel toplamlarına yalnızca seçilen ayın günleri dahil edilir.
- Haftalık hesap için seçilen ayla aynı ISO haftasında bulunan önceki/sonraki ay günleri bağlam olarak kullanılabilir.
- Bağlam günleri aylık toplamları artırmaz.

### KRL-VERİ-005 – Personel anahtarı

- Hesaplamada ana personel anahtarı `sicilno`dur.
- Şablon doldurmada mevcut uygulama `Ad + Soyad` değerini hedef şablondaki `ADI SOYADI` alanıyla eşleştirir.
- Eşleştirme büyük/küçük harf, Türkçe karakter aksanı ve baştaki/sondaki boşluklardan bağımsız yapılır.
- Hedef şablonda bulunmayan Meyer personeli ve Meyer dosyasında bulunmayan şablon personeli ayrı listelenir.
- İsim benzerliğiyle yaklaşık eşleştirme yapılmaz.

> Kalıcı çözümde şablon eşleştirmesinin de sicil numarası üzerinden yapılması önerilir. İsimle eşleştirme aynı isimli personellerde risk oluşturur.

---

## 4. Günlük durum belirleme sırası

Bir gün için birden fazla bilgi bulunması halinde aşağıdaki öncelik sırası uygulanır:

1. İşe giriş öncesi veya işten çıkış sonrası `X`
2. Geçerli ve onaylı `Kaynak Kod` (özellikle `W`, `K`, `D` gibi birleşik/özel kodlar)
3. Resmi tatil/bayram
4. Sağlık raporu
5. Yıllık izin
6. Ücretsiz izin
7. Diğer ücretli izin
8. Fiili çalışma (`NM > 0` veya `FM > 0`)
9. Cumartesi/pazar hafta tatili
10. Hafta içi mazeretsiz devamsızlık

Çelişkili veri bulunan günler ayrıca uyarı olarak raporlanmalıdır. Örneğin aynı gün hem tam gün ücretsiz izin hem 9 saat çalışma varsa kayıt manuel incelemeye düşmelidir.

### 4.1. Otomatik sınıflandırma karar tablosu

| Öncelik | Koşul | Gün durumu | Günlük çıktı |
|---:|---|---|---|
| 1 | Tarih işe girişten önce/çıkıştan sonra | Sayılmayan gün | `X` |
| 2 | Onaylı kaynak kod mevcut | Kaynak kodun durumu | Kaynak kod veya çalışma saati |
| 3 | Sabit resmi tatil tarihi veya `RM Açıklama` tatil işareti | Resmi tatil | `B` |
| 4 | `SGKIZS > 0`, `RM > 0` veya açıklamada `RAPOR/SGK` | Rapor | `R` |
| 5 | `YIZS > 0` veya açıklamada `YILLIK` | Yıllık izin | `Y` |
| 6 | `UCZIZS > 0` veya açıklamada `ÜCRETSİZ` | Ücretsiz izin | `Z` |
| 7 | `IZS > 0` veya geçerli diğer izin açıklaması | Ücretli izin | `Ü` |
| 8 | `NM > 0` veya `FM > 0` | Çalışma | Ham `NM + FM` saat toplamı |
| 9 | Cumartesi, çalışma yok | Serbest zaman | `A3` |
| 10 | Pazar, çalışma yok ve hafta tatili hak edilmiş | Hafta tatili | `T` |
| 11 | Pazar, çalışma yok ve hafta tatili kesilmiş | Kesilen hafta tatili | `Z` |
| 12 | Hafta içi, çalışma/izin/tatil yok | Mazeretsiz | `M` |

### 4.2. Çalışma gününe yazılan değer

- Çalışılan bir günde aylık puantaj hücresine harf yerine **ham `NM + FM` saat toplamı** yazılır.
- Örnek: `NM = 09:00`, `FM = 02:30` ise hücre değeri `11,5` olur.
- Haftalık `FM → NM` dönüşümü hücredeki toplam saati değiştirmez; yalnızca personel özeti ve haftalık kontrol tablosundaki NM/FM dağılımını değiştirir.
- `N` kodu kod sözlüğünde “normal çalışma” karşılığıdır; otomatik günlük matriste fiili saat bulunduğunda sayısal saat tercih edilir.

---

## 5. Harf ve puantaj kodları

Aşağıdaki tablo `data/harf.jpg` referans alınarak hazırlanmıştır.

| Kod | Puantaj karşılığı | Açıklama | Üretim kuralı |
|---|---:|---|---|
| `R` | 7,50 saat | Raporlu | `SGKIZS`, `RM` veya rapor açıklamasından otomatik üretilebilir. |
| `N` | 9,00 saat | Normal çalışma | Kod sözlüğünde bulunur; otomatik matriste fiili `NM + FM` saati yazılır. |
| `K` | 3,75 saat | Yarım gün resmi tatil | Kaynak kod/tatil takvimi yarım gün olarak işaretliyse kullanılır. |
| `Ü` | 7,50 saat | Ücretli izin | `IZS` veya ücretli izin açıklamasından otomatik üretilir. |
| `X` | 9,00 saat | Sayılmayan gün | İşe girişten önce veya işten çıkıştan sonra otomatik üretilir. |
| `C` | 7,50 saat | Ücretsiz hafta tatili | Kaynakta/onaylı puantajda bu durum açıkça varsa kullanılır. |
| `D` | 7,50 saat | Dini bayram | Tatil takviminde dini bayram olarak tanımlı gün için kullanılır. |
| `B` | 7,50 saat | Resmi tatil – hafta içi | Tam gün resmi tatilde otomatik üretilir. |
| `T` | 7,50 saat | Hafta tatili / resmi tatil | Hak edilmiş pazar/hafta tatili için otomatik üretilir. |
| `V` | 3,75 saat | Yarım gün | Kaynak izin kodu veya manuel tanım yarım günse kullanılır. |
| `Y` | 7,50 saat | Yıllık izin | `YIZS` veya yıllık izin açıklamasından otomatik üretilir. |
| `M` | 7,50 saat | Mazeretsiz/devamsız | Hafta içi çalışma, izin ve tatil bulunmazsa otomatik üretilir. |
| `A1` | 1 gün | Serbest zaman – maktu tam gün | Kaynak kod veya manuel tanımdan gelir. |
| `A2` | 0,5 gün | Serbest zaman – maktu yarım gün | Kaynak kod veya manuel tanımdan gelir. |
| `A3` | 0 saat | Serbest zaman – saatlik tam gün | Mesaisiz cumartesi günü otomatik üretilir. |
| `A4` | 3,75 saat | Serbest zaman – saatlik yarım gün | Kaynak kod veya manuel tanımdan gelir. |
| `U` | 9,00 saat | Uzaktan çalışma | Kaynak kod/çalışma tipi uzaktan ise kullanılır. Fiili saat varsa matriste saat yazılabilir. |
| `Z` | 7,50 saat | Ücretsiz izin | `UCZIZS` veya ücretsiz izin açıklamasından otomatik üretilir. Kesilen pazar için de `Z` kullanılır. |
| `W` | 7,50 saat | Tatilde raporlu | Resmi/hafta tatili ile rapor aynı güne denk geliyorsa kaynak kod/tanımlı birleşik kural ile kullanılır. |

### 5.1. Kodların otomatik veya kaynak kontrollü olması

| Kod grubu | Davranış |
|---|---|
| `R`, `Ü`, `Y`, `Z`, `M`, `B`, `T`, `A3`, `X` | Meyer alanlarından veya takvimden otomatik üretilebilir. |
| `K`, `C`, `D`, `V`, `W`, `A1`, `A2`, `A4`, `U` | Kaynak kod, tatil türü veya açık iş kuralı gerektirir; tahmin yoluyla üretilmemelidir. |
| Sayısal saat | Fiili çalışma gününde ham `NM + FM` toplamıdır. |

### 5.2. `Z` kodunun iki farklı kullanımı

`Z` kodu iki durumda kullanılır:

1. Personelin ücretsiz izinli olduğu gün.
2. Mazeretsiz tam gün devamsızlık nedeniyle kesilen pazar/hafta tatili.

Bu iki durum günlük detayda mutlaka ayrıştırılmalıdır:

- ücretsiz izin gününde durum `UCRETSIZ_IZIN`;
- kesilen pazarda durum `HAFTA_TATILI`, `Pazar Durumu = Kesildi`.

Ay başındaki bağlam gününden kaynaklanan pazar kesintisi kontrol ekranında `Z*` olarak renklendirilebilir. Nihai hedef şablona `Z` yazılır.

### 5.3. Eski şablon kodlarının uyumluluğu

Eski Sakra/OTOM dosyalarından aşağıdaki kodlar gelebilir:

| Eski kod | Yeni/standart karşılık |
|---|---|
| `S` | `Y` – yıllık izin |
| `I/İ` | `Ü` – ücretli izin |
| `E` | `M` – mazeretsiz/devamsız |
| `H` | `T` – hafta tatili |

Eski kodlar okunabilir; yeni üretilen çıktıda standart kodlar kullanılmalıdır.

---

## 6. İzin ve rapor kuralları

### KRL-İZN-001 – Sağlık raporu

Aşağıdaki koşullardan biri varsa gün `R` olur:

- `SGKIZS > 0`;
- `RM > 0` ve kayıt resmi tatil olarak işaretli değilse;
- `İzin Açıklama` içinde “RAPOR” veya “SGK” bulunması;
- onaylı kaynak kodun `R` olması.

Raporlu gün pazar kesintisi oluşturmaz.

### KRL-İZN-002 – Yıllık izin

Aşağıdaki koşullardan biri varsa gün `Y` olur:

- `YIZS > 0`;
- `İzin Açıklama` içinde “YILLIK” bulunması;
- onaylı kaynak kodun `Y` veya eski kod `S` olması.

Yıllık izin pazar kesintisi oluşturmaz.

### KRL-İZN-003 – Ücretli izin

Aşağıdaki koşullardan biri varsa gün `Ü` olur:

- `IZS > 0`;
- yıllık izin veya rapor olmayan geçerli bir ücretli izin açıklaması;
- onaylı kaynak kodun `Ü` veya eski kod `I/İ` olması.

Ücretli izin pazar kesintisi oluşturmaz.

### KRL-İZN-004 – Ücretsiz izin

Aşağıdaki koşullardan biri varsa gün `Z` olur:

- `UCZIZS > 0`;
- `İzin Açıklama` içinde “ÜCRETSİZ” bulunması;
- onaylı kaynak kodun `Z` olması.

Hafta içinde en az bir ücretsiz izin (`Z`) varsa aynı ISO haftasının çalışılmayan pazarı da `Z` olur (pazar kesilir). Meyer kaynaktan `C` gelirse pazar ayrıca ücretsiz hafta tatili olarak kabul edilir.

### KRL-İZN-005 – Yarım günler

- Yarım gün izin `V`, yarım gün resmi tatil `K` olarak gösterilir.
- Yarım günün puantaj karşılığı 3,75 saattir.
- Aynı gün yarım gün izin/tatil ve çalışma varsa iki bileşenin toplamı günlük detayda ayrı gösterilmelidir.
- `K` ve `V` kodlarının otomatik üretilebilmesi için Meyer kaynağında yarım gün bilgisinin açıkça bulunması gerekir.

---

## 7. Haftalık 45 saat hesabı

### KRL-45-001 – Hafta tanımı

- Hafta pazartesi başlar ve pazar biter.
- Hesap personel + ISO yıl + ISO hafta bazında yapılır.
- Ay başı veya ay sonuna taşan haftalar tam hafta bağlamıyla değerlendirilir.

### KRL-45-002 – Dönüşüm hesabı

Mevcut sistemde 45 saat kontrolü şu şekilde yapılır:

1. Pazartesi-cuma günlerinin `NM` toplamı alınır.
2. `45 - hafta içi NM` ile eksik normal süre bulunur.
3. Eksik süre sıfırdan büyükse cumartesi ve ardından pazar `FM` süreleri kronolojik sırayla `NM`ye aktarılır.
4. Aktarım, eksik normal süre veya mevcut hafta sonu `FM` tutarından küçük olan kadardır.
5. Aktarılan süre `FM→NM Aktarım` alanında ayrıca gösterilir.
6. Ham `NM` ve `FM` değiştirilmez; `NM Güncel` ve `FM Güncel` alanları üretilir.

```text
Eksik NM = max(0, 45 - Hafta İçi NM)
Aktarım  = min(Eksik NM, Hafta Sonu FM)
NM Güncel = NM Ham + Aktarım
FM Güncel = FM Ham - Aktarım
```

### KRL-45-003 – Örnekler

| Hafta içi NM | Hafta sonu FM | NM’ye aktarım | Kalan FM |
|---:|---:|---:|---:|
| 45 | 4 | 0 | 4 |
| 43 | 4 | 2 | 2 |
| 36 | 3 | 3 | 0 |
| 30 | 20 | 15 | 5 |

### KRL-45-004 – İzinlerin etkisi

Mevcut hesapta izin/rapor kodlarının 7,50 saatlik puantaj karşılığı, hafta içi `NM` toplamına otomatik eklenmez. 45 saat eksiği yalnızca Meyer `NM` alanı üzerinden belirlenir.

Bu kural iş birimi tarafından değiştirilirse Meyer tarafında açıkça versiyonlanmalıdır; aynı dönemde iki farklı yöntem uygulanmamalıdır.

### KRL-45-005 – Tekrar hesaplamayı önleme

Haftadaki herhangi bir kayıt `Hesaplanmış = true` olarak geliyorsa aynı haftaya ikinci kez `FM → NM` dönüşümü uygulanmaz.

---

## 8. Pazar/hafta tatili kuralı

### KRL-PZR-001 – Hak edilen pazar

Aynı ISO haftasının pazartesi-cuma günlerinde mazeretsiz tam gün devamsızlık yoksa çalışılmayan pazar `T` olur.

### KRL-PZR-002 – Kesilen pazar

Aynı ISO haftasının pazartesi-cuma günlerinden en az birinde mazeretsiz tam gün devamsızlık (`M`) **veya** ücretsiz izin (`Z`) varsa çalışılmayan pazar `Z` olur.

Tam gün devamsızlık kontrolü:

1. Gün durumu `DEVAMSIZ/M` olmalıdır.
2. `EM >= 9 saat` ise tam gün kabul edilir.
3. `EM` tam gün değilse ve `MS > 0` ise `MS` dikkate alınır.
4. `EM` ve `MS` yoksa devamsız gün 9 saat kabul edilir.
5. Kontrol eşiği 9 saattir.

> `M` kodunun bordro karşılığı 7,50 saat olsa da pazar kesintisi için kullanılan çalışma düzeni eşiği 9 saattir. Bunlar farklı amaçlarla kullanılan iki değerdir.

### KRL-PZR-003 – Pazar hakkını koruyan durumlar

Aşağıdaki günler mazeretsiz devamsızlık sayılmaz ve pazar kesintisi doğurmaz:

- fiili çalışma;
- yıllık izin;
- ücretli izin;
- sağlık raporu;
- resmi tatil;
- yarım gün;
- uzaktan çalışma;
- serbest zaman;
- sayılmayan gün.

Ücretsiz izin (`Z`) bu listede değildir; hafta içinde en az bir `Z` pazarı keser.

### KRL-PZR-004 – Kaynaktan ücretsiz hafta tatili

Pazar günü Meyer kaynağından `C` geliyorsa sistem pazarın ücretsiz hafta tatili olduğunu kabul eder. Bu durumda otomatik hak ediş kuralı `C` kodunu ücretli `T` koduna çevirmemelidir.

### KRL-PZR-005 – Ay sınırı

Örnek: Temmuz ayı çarşamba günü başlıyorsa ilk pazarın haftası 29-30 Haziran günlerini de kapsar.

- 29 veya 30 Haziran’da tam gün mazeretsiz devamsızlık varsa 5 Temmuz pazarı kesilir.
- Temmuz matrisinde pazar nihai olarak `Z` gösterilir.
- Kontrol ekranında kesinti kaynağı `Bağlam` ve geçici gösterim `Z*` olabilir.
- Haziran günleri Temmuz ayı çalışma/izin/devamsızlık toplamlarına eklenmez.

---

## 9. Cumartesi ve hafta sonu kuralları

### KRL-HS-001 – Mesaisiz cumartesi

Cumartesi günü çalışma, izin veya özel kaynak kod yoksa `A3` yazılır.

### KRL-HS-002 – Çalışılan cumartesi/pazar

- `NM + FM > 0` ise günlük hücreye toplam fiili saat yazılır.
- Bu çalışma saatinin NM/FM dağılımı haftalık 45 saat kuralından sonra haftalık kontrol ve personel özetinde gösterilir.
- Günlük matris hücresi dönüşümden önceki toplam fiili saati korur.

### KRL-HS-003 – Çalışılmayan pazar

- Hafta tatili hak edilmişse `T`.
- Tam gün mazeretsiz devamsızlık nedeniyle kesilmişse `Z`.
- Kaynaktan ücretsiz hafta tatili gelmişse `C`.

---

## 10. Resmi tatil ve tatilde çalışma kuralları

### KRL-RT-001 – Sabit resmi tatiller

Sistem en az aşağıdaki tam gün resmi tatilleri takvimden tanır:

- 1 Ocak
- 23 Nisan
- 1 Mayıs
- 19 Mayıs
- 15 Temmuz
- 30 Ağustos
- 29 Ekim

### KRL-RT-002 – Değişken ve yarım gün tatiller

Dini bayram, arife ve yarım gün resmi tatiller Meyer’in resmi tatil takviminden veya kaynak kayıttaki açık tatil türünden gelmelidir. `RM Açıklama` içinde Ramazan, Kurban veya bilinen tatil adı bulunması da tatil işareti olarak değerlendirilebilir.

### KRL-RT-003 – Günlük kod

- Tam gün resmi tatil için `B`.
- Dini bayram için açık tür bilgisi varsa `D`.
- Yarım gün resmi tatil için `K`.
- Tatilde raporlu için birleşik durum açıkça geliyorsa `W`; açık `W` kodu genel `B` koduna çevrilmez.
- İşe giriş öncesi/işten çıkış sonrası `X`, tatil kodundan önceliklidir.

Sistem, tanıdığı tam gün tatilleri hedef hücreye `B` olarak yazar.

### KRL-RT-004 – Resmi tatilde çalışılan süre

Resmi tatil gününde çalışma varsa günlük hücre tatil kodunu korur. Çalışılan saat ayrıca haftalık manuel kolonlara dağıtılır:

```text
45 saate kalan = max(0, 45 - tatil dışındaki haftalık fiili çalışma)
45'i aşmayan tatil çalışması = min(tatil çalışması, 45 saate kalan)
45'i aşan tatil çalışması = max(0, tatil çalışması - 45'i aşmayan kısım)
```

Örnek: Tatil dışındaki haftalık çalışma 43 saat, tatilde çalışma 4 saat ise:

- `45'i aşmayan kısım = 2 saat`
- `45'i aşan kısım = 2 saat`

### KRL-RT-005 – Tatilde çalışma süresi kaynağı

Tatil günü çalışma süresi aşağıdaki sırayla belirlenir:

1. `NM + FM`
2. `Giriş/Çıkış` hareketinden hesaplanan net süre
3. Günlük hücredeki sayısal çalışma değeri

Giriş/çıkıştan hesap yapılırsa:

- çıkış girişten küçük/eşitse ertesi gün kabul edilir;
- brüt süre 6 saat veya üzerindeyse mevcut kurala göre 1 saat 15 dakika mola düşülür;
- sonuç sıfırdan küçük olamaz.

### KRL-RT-006 – İlk haftaya önceki aydan devir

Ay pazartesi dışında bir günde başlıyorsa, aynı haftanın önceki aya ait fiili çalışma saatleri “Önceki ay son hafta devreden fiili çalışma” alanına yazılır ve resmi tatil çalışmasının 45 saat ayrımında dikkate alınır.

- Yalnızca fiili çalışma saati devreder.
- İzin/tatil kodlarının puantaj karşılığı devre eklenmez.
- Personel ay ortasında işe girdiyse önceki ay devri `0` olmalıdır.

---

## 11. İşe giriş ve işten çıkış kuralları

Sistem, hedef şablondaki `Çalışan Bilgileri` sayfasından `ADI SOYADI`, `GİRİŞ TARİHİ` ve `ÇIKIŞ TARİHİ` alanlarını okur.

- İşe giriş tarihinden önceki tüm dönem günlerine `X` yazılır.
- İşten çıkış tarihinden sonraki tüm dönem günlerine `X` yazılır.
- İşe giriş günü ve işten çıkış günü kapsam içindedir.
- Personelin giriş ve çıkış tarihi yoksa bu kural uygulanmaz.
- `X`, otomatik resmi tatil `B` kodunun da önüne geçer.

Örnek: Giriş 16 Temmuz, çıkış 19 Temmuz ise 1-15 Temmuz ve 20-31 Temmuz `X`; 16-19 Temmuz normal kurallara göre değerlendirilir.

---

## 12. Kontrol ve raporlama çıktıları

Hesaplama sürecinde aşağıdaki kontrol ve raporlama tabloları oluşturulmalıdır:

### 12.1. Aylık Puantaj

- Her personel bir satırdır.
- Gün kolonları `01 Pt`, `02 Sa`, `03 Ça` biçimindedir.
- Çalışma gününde ham `NM + FM` saati yazılır.
- Diğer günlerde kod tablosundaki harf/kod yazılır.
- Hafta sonu kolonları görsel olarak ayrıştırılır.

### 12.2. Personel Özeti

Her personel için en az aşağıdaki toplamlar bulunur:

- çalışma günü;
- güncel normal çalışma;
- güncel fazla mesai;
- `FM → NM` aktarımı;
- yıllık izin günü;
- ücretli izin günü;
- rapor günü;
- ücretsiz izin günü;
- devamsızlık günü;
- hafta tatili günü;
- ücretsiz hafta tatili günü;
- resmi tatil günü;
- serbest zaman günü;
- uzaktan çalışma günü;
- yarım gün;
- sayılmayan gün;
- pazar kesintisi adedi.

### 12.3. Haftalık Kontrol

Her personel ve ISO hafta için:

- hafta içi NM;
- hafta sonu ham FM;
- FM’den NM’ye aktarılan süre;
- toplam güncel NM;
- kalan güncel FM;
- pazar durumu;
- pazar kesinti kaynağı.

### 12.4. Günlük Detay

Ham ve hesaplanan değerler birlikte gösterilmelidir:

- tarih, giriş, çıkış;
- ham `MS/NM/FM`;
- güncel `NM/FM`;
- `FM → NM`;
- izin ve rapor süreleri;
- gün durumu;
- kaynak kod;
- sonuç kodu;
- pazar durumu.

### 12.5. Veri Kalitesi ve Kodlar

- Geçersiz tarih, boş sicil ve mükerrer kayıt sayıları gösterilir.
- Harf/kod açıklamaları ayrı sayfada bulunur.

---

## 13. Hedef şablon doldurma kuralları

### KRL-ŞBL-001 – Şablonun bulunması

Hedef çalışma sayfasında en az:

- `ADI SOYADI` personel kolonu;
- `Pt1, Sa1, Ça1, Pe1, Cu1, Ct1, Pz1` biçimindeki haftalık gün kolonları

bulunmalıdır. Bu yapı yoksa dosya geçersiz şablon olarak reddedilir.

### KRL-ŞBL-002 – Takvim hizalama

- Ayın 1. gününün haftanın hangi gününe geldiği hesaplanır.
- Gün numaraları doğru hafta/gün kolonuna yazılır.
- Örnek: 1 Temmuz çarşamba ise gün 1 `Ça1`, gün 2 `Pe1`, gün 3 `Cu1` kolonuna yazılır; `Pt1` ve `Sa1` boş kalır.
- Ayda bulunmayan veya ilgili dönemde kullanılmayan eski hücre değerleri temizlenir.

### KRL-ŞBL-003 – Personel eşleştirme

- Meyer’deki `Ad + Soyad`, şablondaki `ADI SOYADI` ile normalize edilerek eşleştirilir.
- Eşleşmeyen şablon ve kaynak personeller çıktı istatistiğinde ayrı gösterilir.
- Eşleşmeyen personele ait satıra tahmini veri yazılmaz.

### KRL-ŞBL-004 – Yazılan alanlar

Sistem aşağıdaki alanları doldurur:

- günlük saat/harf kodları;
- önceki aydan devreden ilk hafta fiili çalışma saati;
- her haftanın resmi/hafta tatili çalışmasında 45 saati aşmayan manuel süre;
- her haftanın resmi/hafta tatili çalışmasında 45 saati aşan manuel süre.

Şablondaki ücret ve bordro formülleri yeniden yazılmaz; mevcut formüllerin günlük kodlardan sonuç üretmesi beklenir.

### KRL-ŞBL-005 – `Z*` dönüşümü

Kontrol tablosunda bağlam kaynaklı pazar kesintisini göstermek için kullanılan `Z*`, hedef şablon hücresine `Z` olarak yazılır.

### KRL-ŞBL-006 – Sonuç istatistiği

İşlem sonunda en az aşağıdaki bilgiler dönmelidir:

- eşleşen personel sayısı;
- şablonda olup kaynakta bulunmayan kişiler;
- kaynakta olup şablonda bulunmayan kişiler;
- doldurulan günlük hücre sayısı;
- devir saati yazılan personel sayısı;
- tatil manuel alanı doldurulan kayıt sayısı;
- doldurulan sayfa adı, yıl ve ay.

---

## 14. Hesaplama örnekleri ve kabul senaryoları

| No | Senaryo | Girdi | Beklenen sonuç |
|---:|---|---|---|
| 1 | Normal çalışma | `NM 9`, `FM 0` | Günlük hücre `9` |
| 2 | Çalışma + FM | `NM 9`, `FM 2,5` | Günlük hücre `11,5` |
| 3 | Yıllık izin | `YIZS 7,5` | `Y` |
| 4 | Ücretli izin | `IZS 7,5` | `Ü` |
| 5 | Sağlık raporu | `SGKIZS 7,5` | `R` |
| 6 | Ücretsiz izin | `UCZIZS 7,5` | `Z`, durum ücretsiz izin |
| 7 | Mazeretsiz hafta içi | Çalışma/izin/tatil yok, `MS 9` | `M` |
| 8 | Mesaisiz cumartesi | `NM 0`, `FM 0` | `A3` |
| 9 | Hak edilmiş pazar | Hafta içinde tam gün mazeretsiz yok | `T` |
| 10 | Kesilen pazar | Hafta içinde en az bir 9 saat mazeretsiz veya ücretsiz izin (`Z`) | `Z`, pazar durumu kesildi |
| 11 | İzinli haftada pazar | Hafta içi bir gün yıllık/ücretli izin veya rapor | Pazar `T` |
| 11b | Ücretsiz izinli haftada pazar | Hafta içi en az bir `Z` | Pazar `Z` |
| 12 | 45 saat tamam | Hafta içi NM `45`, hafta sonu FM `4` | Aktarım `0`, kalan FM `4` |
| 13 | 45 saat eksik | Hafta içi NM `43`, hafta sonu FM `4` | Aktarım `2`, kalan FM `2` |
| 14 | Ay sınırında kesinti | Önceki ay son hafta içi gününde 9 saat mazeretsiz, pazar yeni ayda | Yeni ay pazarı nihai `Z`, kaynak `Bağlam` |
| 15 | Resmi tatil | 15 Temmuz, çalışma yok | `B` |
| 16 | Tatilde 4 saat çalışma | Tatil dışı hafta toplamı `43` | Gün `B`; aşmayan `2`, aşan `2` |
| 17 | Tatilde giriş/çıkış | `NM/FM 0`, giriş `07:40`, çıkış `18:15` | 1:15 mola sonrası net süre tatil manuel hesabında kullanılır |
| 18 | Ay ortası işe giriş | Giriş 16 Temmuz | 1-15 `X`; 16’dan sonrası günlük kurala göre |
| 19 | Ay içinde işten çıkış | Çıkış 19 Temmuz | 20’den ay sonuna kadar `X` |
| 20 | Şablon hizalama | Ay çarşamba başlıyor | Gün 1 `Ça1`; `Pt1/Sa1` boş |
| 21 | İsim eşleşmiyor | Meyer personeli şablonda yok | Veri yazılmaz, `unmatched_source` listesine eklenir |
| 22 | Mükerrer kişi-gün | Aynı sicil ve tarih iki kez | İlk kayıt kullanılır, kalite uyarısı oluşur |

Meyer geliştirmesi, bu senaryoların tamamında referans sistemle aynı sonucu üretmelidir.

---

## 15. İşlem sonunda zorunlu kontroller

Aşağıdaki kontroller başarılı olmadan çıktı “bordroya hazır” kabul edilmemelidir:

- Kaynak personel sayısı ile eşleşen personel sayısı mutabakatı.
- Seçilen ayın tüm günlerinin doğru şablon kolonuna yazılması.
- Aynı kişi-gün için tek sonuç bulunması.
- Negatif `NM`, `FM` veya aktarım oluşmaması.
- `FM → NM + kalan FM = ham FM` eşitliğinin korunması.
- Ay dışı bağlam saatlerinin aylık toplama eklenmemesi.
- Pazar kesintilerinin kaynak gününün izlenebilmesi.
- `X` günlerinin resmi tatil/izin koduyla ezilmemesi.
- Resmi tatil çalışma saatlerinin aşan/aşmayan kolon toplamıyla eşleşmesi.
- Şablondaki mevcut formüllerin ve biçimlendirmenin korunması.

---

## 16. Meyer ekibiyle kesinleştirilecek maddeler

Bu bölüm hesaplama kuralını değiştirmek için değil, Meyer çıktısının doğru alanları sağlamasını garanti etmek için kullanılmalıdır:

1. `RM` alanının rapor ve resmi tatil kullanımının kesin ayrımı.
2. Dini bayram (`D`), yarım gün resmi tatil (`K`) ve tatilde raporlu (`W`) bilgisinin Meyer’den hangi alan/kodla geleceği.
3. Ücretsiz iznin pazar hakkına etkisi ve `C` kodunun Meyer tarafından hangi koşulda üretileceği.
4. İzin/rapor saatlerinin haftalık 45 saat hesabına dahil edilip edilmeyeceği. Mevcut sistem dahil etmemektedir.
5. Resmi tatilde giriş-çıkıştan süre üretirken uygulanacak mola süresinin sabit mi vardiya bazlı mı olduğu.
6. Personel eşleştirmesinde hedef şablondaki sicil numarasının güvenilir anahtar olarak kullanılıp kullanılamayacağı.
7. Aynı gün birden fazla vardiya veya hareket varsa Meyer’in tek kişi-gün kaydını nasıl oluşturacağı.
8. Onaylı kaynak kodun ham sürelerle çelişmesi halinde hangi alanın esas alınacağı.

---

## 17. Teslim ve onay kriteri

Geliştirme tamamlandığında Meyer ekibi:

1. Kullanılan alan ve kod sözlüğünü paylaşmalıdır.
2. Bu dokümandaki kabul senaryolarını içeren örnek Meyer dosyası üretmelidir.
3. Aynı dosya referans sistemde ve Meyer çözümünde çalıştırılmalıdır.
4. Günlük kodlar, haftalık NM/FM dağılımı, pazar durumu ve hedef şablon hücreleri birebir karşılaştırılmalıdır.
5. Fark bulunan kayıtlar kapatılmadan geliştirme kabul edilmemelidir.
6. Kural değişiklikleri sürüm numarası ve değişiklik tarihiyle yönetilmelidir.

| Onay tarafı | Ad / rol | Tarih | Onay |
|---|---|---|---|
| İnsan Kaynakları |  |  |  |
| Bordro |  |  |  |
| Meyer Geliştirme |  |  |  |
| Proje Sorumlusu |  |  |  |
