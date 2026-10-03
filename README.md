# İkinci Beyin V3.2 nasıl çalışır?

Bu rehber, İkinci Beyin V3.2'nin çalışma mantığını sesli sohbette sorularak öğrenilebilecek biçimde açıklar. Kişisel vault içeriği, kullanıcı kayıtları ve bilgisayara özgü yollar bu depoya alınmaz. Kod bağlantıları Taha'nın açık kaynak [avenoxbeyin v3.2.0](https://github.com/avenoxai/avenoxbeyin/tree/v3.2.0) sürümüne gider.

ChatGPT'ye bu README bağlantısını verip şöyle sorabilirsin: **"Bu rehberi kaynak al. Kütüphane benzetmesiyle, kısa cümlelerle anlat. Önce zil ve ajan ayrımını açıkla; sonra sorularıma göre ayrıntıya gir. Rehberde doğrulanmayan bir şeyi kesinmiş gibi söyleme."**

## Kütüphane benzetmesi ve gerçek dosyalar

```text
KUTUPHANEDEKI SEY                  V3.2'DEKI KARSILIGI
------------------                 -------------------
Bina ve raflar               -->    Vault klasoru ve Markdown notlari
Kullanici kimligi kitabi      -->    Companion/Core.md (veya mevcut Soul.md)
Onceki vardiya defteri        -->    Companion/Last-Session.md
Acik isler dosyasi            -->    Companion/Threads.md
Duzeltmeler defteri           -->    Companion/Kurallar.md
Gorevlinin talimat kitabi     -->    .claude/scripts/beyin_v3_*.py
Duvar yazisi                  -->    AGENTS.md / CLAUDE.md
Kapi zilleri                  -->    Uygulamanin hook ayarlari
Arka odadaki katalog         -->    Vault disindaki state/memory.sqlite3
Yapilan isin makbuzu          -->    receipts/*.md
```

Companion klasörünün adı kurulumdan kuruluma değişebilir. Bu örnekte `Companion/` açıklayıcı kısa addır. V3 motoru mevcut kullanıcı klasörünü bulup onu korur; ikinci bir kimlik klasörü açmaz. Gerçek başlangıç dosyaları ve bu davranış [companion kodunda](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3_companion.py) tanımlıdır.

Vault, asıl kitapların bulunduğu yerdir. `state` ise bilgisayara özgü, yeniden oluşturulabilir katalog ve olay fişlerini tutar. Bu yüzden state vault dışında olmalıdır; motor vault içine konursa çalışmayı reddeder. Kaynak: [hook](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3_hook.py), [hafıza deposu](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3.py).

## Bir mesaj geldiğinde

Örnek soru: “Bu projede en son hangi kararı almıştık?”

```text
KULLANICI MESAJI
       |
       v
Uygulama UserPromptSubmit olayini uretir
       |
       v
HOOK: beyin_v3_hook.py
       |
       +--> Olay fisi: hook-queue/*.json
       |       olay adi + uygulama + oturum ozeti + proje etiketi
       |       tam sohbet metni burada yok
       |
       +--> Arka plan iscisi
       |       vault'taki uygun Markdown dosyalarini tarar
       |       memory.sqlite3 katalogunu yeniler
       |       hook-health.json sonucunu yazar
       |       islenen fisi hook-done/ klasorune tasir
       |
       +--> Mesajin kelimeleriyle katalogda arar
               |
               +--> Eslesen kaynak -> sinirli baglam paketi
               +--> Eslesme yok    -> ilgisiz not eklemez
```

Fişin yazılması ve işlenmesi [hook kodunda](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3_hook.py) bulunur. Markdown taraması, dosya özetleri ve katalog yenilemesi [sync kodunda](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3_sync.py) bulunur. Hook ayarları otomatik kontrolü kapatmışsa veya kontrol aralığı dolmamışsa tarama ertelenebilir. Uygulamalar kapalıyken sürekli çalışan bir servis yoktur.

## Kaynak ajana nasıl ulaşır?

```text
ASIL MARKDOWN KAYNAGI
  Core.md / Last-Session.md / konu notu
              |
              v
KATALOG: memory.sqlite3
              |
              | soru ile ilgili kayitlari sec
              v
KISA KAYNAK PAKETI
  metin parcasi + dosya adi + kaynak kimligi + uyari
              |
              | hook ciktisi: additionalContext
              v
AJANIN O ANKI BAGLAMI
              |
              v
AJAN kaynagi yorumlayip cevap verir
```

Oturum başında Core, Kurallar, Last-Session, Threads ve Journal önceliklidir; varsa bilgi haritası ve son makbuz da sınıra sığdığı kadar gelir. Normal mesajda ilgili kaynak aranır. Paket kısaltılmış olabilir; ajan gerekli ayrıntı için asıl Markdown dosyasını açmalıdır. Kaynak: [companion bağlamı](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3_companion.py), [hook çıktısı](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3_hook.py).

`AGENTS.md` bir **kural metnidir**: ajana nasıl davranacağını söyler. Hook ise uygulamanın belli olayda **çalıştırdığı programdır**: ajanın önüne kaynak getirir. Kural okumak ve zili çalmak iki farklı mekanizmadır.

## Kalıcı hafızaya yazma

Bir mesajın aramada kullanılması, onun otomatik olarak bilgi notuna yazıldığı anlamına gelmez. Kullanıcı “Ben şu işi yürütüyorum” dediğinde, bunun kalıcı kullanıcı olgusu olup olmadığını ve nereye ait olduğunu ajan değerlendirir.

```text
KULLANICININ SOYLEDIGI YENI BILGI
               |
               v
AJAN anlamini, kaynagini ve kapsamini degerlendirir
               |
               +--> kalici tercih/olgu  -> Core.md
               +--> acik is/karar       -> Threads.md
               +--> oturum sonucu       -> Last-Session.md
               +--> tekrar kullanilan bilgi -> knowledge/concepts/*.md
               |
               v
AJAN ilgili Markdown kaynagini yazar
               |
               v
sync: katalog yeni kaynaga gore yenilenir
               |
               v
receipt: sonuc ve degisen kaynaklar makbuza baglanir
```

Ajanın kayıt protokolü [beyin skill'inde](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.agents/skills/beyin/SKILL.md) yazılıdır. Kullanıcının açık beyanı ile ajanın tahmini birbirine karıştırılmaz. `receipt`, ajanın yaptığı işe ilişkin kaynak bağlantılı **iddiasıdır**; her sonucu bağımsız olarak kanıtlayan bir denetçi değildir. `daily/v3/` ve `knowledge/v3/outcomes.md` makbuzlardan üretilen dizinlerdir; kendi başlarına yeni bilgi çıkarmazlar. Kaynak: [sync ve receipt](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3_sync.py), [makbuz görünümleri](https://github.com/avenoxai/avenoxbeyin/blob/v3.2.0/template/.claude/scripts/beyin_v3_projections.py).

Motor ilk kurulumda **boş başlangıç notlarını** oluşturabilir. Ayrıca olay fişi, katalog, sağlık kaydı ve otomatik makbuz dizinleri yazar. Buradaki ayrım şudur: **Kullanıcı hakkında anlamlı yeni bir cümleyi doğru kalıcı nota yazan taraf ajandır.**

## Sonraki oturumda hatırlama

```text
OTURUM 1                              OTURUM 2
--------                              --------
Kullanici bir tercih soyler           Yeni oturum acilir
        |                                     |
Ajan Core.md'ye kaydeder              SessionStart zili calar
        |                                     |
sync katalogu yeniler                 Core + son oturum baglami gelir
        |                                     |
receipt sonucu baglar                 Ajan kaynagi kontrol ederek devam eder
```

Kısacası: **Zil olayı yakalar. Görevli kaynakları bulur ve kataloglar. Ajan anlamlandırır, konuşur ve gerektiğinde yazar.**

## 3 Ekim 2026 tarihli yerel kurulumun farkları

Bu bölüm, V3.2'nin genel kodundan değil, kurulumun yerel envanterinden özetlenmiştir. Kişisel ayar dosyaları bu rehbere kopyalanmamıştır.

```text
RESMI V3.2 MOTORU              YEREL EK / BAGLANTI
----------------              -------------------
HS1 oturum basi baglami   -->   Claude: vault icindeki hook ayarlari
HS2 mesaj basi arama      -->   Codex: global ikinci-beyin koprusu
HS4-HS7 yasam dongusu    -->   Eski Codex proje hook'u yedekte
                            HS3: mesaj basi yerel saat, ozel ek
```

Saat ekleyen HS3, resmi V3.2 motorunun parçası değildir. Claude için V3 zilleri vault içindedir; başka bir projedeki Claude oturumu bu bağlantıyı kendiliğinden almaz. Codex için global köprü vardır. Ancak 2 Ekim tarihli bir Codex testinde köprü yalnız saati getirmiş, Companion bağlamını getirmemiştir; nedeni kesinleşmemiştir. Dolayısıyla “hook tanımlı” ile “bu oturumda bütün notlar geldi” aynı iddia değildir. Yerel hook sözleşmesini bütün uygulamalara yayma işi de bu tarihte **plan** durumundadır.

## Bu rehberin sınırı

Bu metin V3.2 motorunun kaynak koduyla doğrulanan genel akışı anlatır. Her istemcinin hook kurulumu ve güven onayı ayrıdır. Bir istemcinin belirli bilgisayardaki bağlantısının bugün çalıştığını, yalnız bu rehberden çıkaramayız. Kodun daha yeni sürümleri farklı davranabilir; buradaki kod bağlantıları özellikle `v3.2.0` etiketine sabitlenmiştir.
