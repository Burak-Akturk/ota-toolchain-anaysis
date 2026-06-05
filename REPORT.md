# Araştırma ve ELF Analizi Raporu

Bu bölümde, proje kapsamında oluşturulan ve kablosuz ağda aktarılan `new-firmware.z1` imaj dosyasının msp430 araç zincirleri (toolchain) ile yapılan yapısal analiz sonuçları ve terminal çıktıları sunulmaktadır.

## 1. Dosyanın ELF Sınıfı, Mimarisi ve Giriş Adresi

`msp430-readelf -h new-firmware.z1` komutu ile yapılan başlık (header) analizinde aşağıdaki sonuçlar elde edilmiştir:

* **Sınıf (Class):** `ELF32` (İmaj 32-bit bellek adresleme formatındadır).
* **Mimari (Machine):** `Texas Instruments msp430 microcontroller` (Derleyici, hedef donanım olarak MSP430 mimarisini baz almıştır).
* **Tip (Type):** `EXEC (Executable file)` (Doğrudan çalıştırılabilir uygulamadır).
* **Giriş Adresi (Entry Point):** `0x3100`. Cihaza enerji verildiğinde veya donanım resetlendiğinde işletim sisteminin okumaya başlayacağı ilk makine kodunun bellekteki tam başlangıç adresidir.

## 2. Temel Bölümlerin (Sections) Varlığı ve İfade Ettiği Anlam

`msp430-readelf -S new-firmware.z1` komutu ile incelenen imajda, belleğin farklı işlevler için yapısal olarak şu şekilde bölümlendirildiği tespit edilmiştir:

* **`.text` (Adres: `0x00003100`, Boyut: `0x976e`):** Cihazın doğrudan çalıştıracağı salt okunur ve çalıştırılabilir makine kodlarını barındırır. Giriş adresiyle tam olarak aynı noktadan başlamaktadır.
* **`.rodata` (Adres: `0x0000c870`):** Koddaki değiştirilemez string sabitlerini ("Dogrulama Basarili" vb.) tutan alandır.
* **`.data` (Adres: `0x00001100`):** Kod yazılırken ilk değeri verilmiş global değişkenlerin tutulduğu bölümdür. Hem ROM'da hem de RAM'de yer kaplar.
* **`.bss` (Adres: `0x00001250`):** İlk değeri atanmamış değişkenlerin bulunduğu alandır. Çıktıda `NOBITS` bayrağına sahiptir; yani ELF dosyasında disk alanı kaplamaz, cihaz çalıştığında RAM üzerinde yer tahsis edilir.
* **`.vectors` (Adres: `0x0000ffc0`):** Donanımsal kesmelerin (interrupt) bellekte nereye yönlendirileceğini belirten kritik tablodur.

## 3. Kod ve Veri Boyutları

İmajın terminal üzerinde `msp430-size new-firmware.z1` komutuyla yapılan fiziksel boyut analizi şu şekildedir:
`text: 71715    data: 336    bss: 5706    dec: 77757    hex: 12fbd`

* **Flash (Kalıcı Bellek) Tüketimi:** `.text` (71715 bayt) + `.data` (336 bayt) = **72.051 Bayt (~70.3 KB)**. İşlemcinin ROM'una yazılacak asıl kalıcı yük budur.
* **SRAM (Uçucu Bellek) Tüketimi:** `.bss` (5706 bayt) + `.data` (336 bayt) = **6.042 Bayt (~5.9 KB)**. Dosyanın çalışma zamanındaki (runtime) anlık RAM ihtiyacını ifade eder.

> **Not:** İmajın diske yazılan ham boyutu yaklaşık 130 KB olmasına rağmen, donanımsal karşılığının 78 KB çıkmasının sebebi; cihaza yüklenmeyen ve sadece debug/sembol eşleştirme amaçlı kullanılan `.symtab` ve `.debug_info` bölümleridir.

## 4. Sembol Tablosu ve Anlamlı Semboller

İmajın bölüm başlıkları incelendiğinde, `19` numaralı indekste `0x004770` (yaklaşık 18 KB) boyutunda devasa bir `.symtab` (Sembol Tablosu) barındırdığı görülmüştür.

* **İfade Ettiği Anlam:** Sembol tablosu, makine dilindeki onaltılık (hex) bellek adreslerinin, programcının yazdığı C fonksiyonlarıyla eşleşmesini sağlar. Bu tablo sayesinde cihazın içerisinde `udp_rx_callback`, `calculate_checksum` ve `cfs_write` gibi fonksiyonların bellek adresleri çözümlenebilmektedir. Hata ayıklama (debugging) için kritik bir yapıdır.

## 5. Kesme Vektörleri ve Başlangıç Adresi İlişkisi

`readelf -S` çıktısında `7` numaralı indeks olan `.vectors` tablosunun bellekte tam olarak `0x0000ffc0` adresinde yer aldığı tespit edilmiştir. 

MSP430 mimarisinde zamanlayıcı ve radyo alıcısı gibi donanımsal kesmeler (Interrupts) her zaman belleğin sonundaki (`0xFFC0 - 0xFFFF`) aralığa yazılmak zorundadır. OTA güncellemesi sırasında bu kesme vektörleri atlanır veya bellekte yanlış bir adrese kaydırılırsa, düğüm ağdan bir paket aldığı an (RX interrupt tetiklendiğinde) işletim sistemi ne yapacağını bilemez ve donanım Hard Fault vererek çöker.

## 6. Dosyanın Neden "Ham Binary (.bin)" Değil de "ELF Executable" Olarak Değerlendirildiği

OTA üzerinden cihaza sadece `101010` şeklindeki ham makine kodlarının (`.bin` formatında) gönderilmesi hedeflenen bellek yönetimi için yeterli değildir. 

Eğer cihaza sadece `.bin` dosyası yollansaydı, işletim sisteminin ara yükleyicisi (Bootloader) gelen 130 KB'lık verinin hangi baytının kod alanına (`0x3100`), hangi baytının RAM'e tahsis edileceğini ve hangi baytının kesme adreslerine (`0xFFC0`) yazılacağını bilemezdi. **ELF (Executable and Linkable Format)** dosyası, barındırdığı **Section Headers (Bölüm Başlıkları)** sayesinde bootloader'a dinamik bir **"Yükleme Haritası"** sunar. Güvenli bir bellek yerleşimi ve hatasız bir OTA aktarımı için Contiki-NG donanımının bu ELF hiyerarşisine yapısal olarak ihtiyacı vardır.