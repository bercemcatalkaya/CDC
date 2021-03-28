## _CDC(Change Data Capture)_

- Kritik verilerin bulunduğu veritabanı uygulamalarında, veriler üzerindeki değişiklikleri veri güvenliği amacıyla kaydetmemiz gerekir. Bunun yapılması için çeşitli yöntemler bulunmaktadır: Tetikleyiciler(triggers), saklı yordamlar(stored procedures) vb. Bu yöntemlere ek olarak,SQL Server geliştiricileri bu yöntemlerden daha performanslı ve sisteme daha az yük getirmek amacıyla yeni bir çözüm ürettiler.

## CDC'yi Ne Amaçla Kullanabiliriz?

Örneğin, ihtiyacınız farklı sunucularda bulunan verileri tek bir sunucuda birleştirmek ve raporlama yapmak. Bunu sadece farklı verileri geçici tablolarda tutarak, yordam ile nihai sunucunuzdaki tablolarınıza alarak yapabilirsiniz. Ancak veri sayısı arttıkça, işlemlerinizin zamanı da artar.

## CDC Nasıl Çalışır?
CDC, verilerdeki değişikliği yakalamak için SQL Server T-Log alt yapısını kullanır. SQL Server, takip etmek istediğimiz tabloda bulunan verilerdeki değişikliği önce loga yazar. Eğer tablo üzerinde CDC yapısı aktif ise(nasıl aktif edileceği bir sonraki bölümde anlatılacaktır) Resim 2'de gösterildiği gibi, transaction log mekanizması loga yazılan verileri CDC sürecine giriş olarak gönderir. CDC süreci, transaction log mekanizması üzerinden değişiklikleri alıp tablo üzerinde kayıt oluşturur. Bu kayıt, izlenilmesi istenen tablonun aynı sütun yapısıyla ancak tablo üzerindeki değişikliğin ne olduğunu özetlemek için meta verileri içeren ek sütunlar bulundurur. Log alt yapısını kullandığı için daha performanslı, hızlı ve sisteme daha az yük getiriyor. Ayrıca sadece tablo üzerinde değil, tüm tablodaki değişikliğin algılanması yerine, değişikliğin algılanmasını istediğiniz sütunu seçebiliyorsunuz. Böylece, daha etkili bir yapı sunuyor.

![CDC](https://media2.picsearch.com/is?LtfQTyawFJHox31XVsnwzkV_-yr5xyCmdE1n5fHHdRA&height=319)
[Detaylı bilgi için tıklayınız.](https://docs.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver15) 

> Not: CDC, SQL Server Enterprise, Developer, SQL Server 2016 SP1 ve sonraki Standard sürümleri destekler. Fakat Web ve Express sürümlerini desteklemez. Örneklerimde SQL Server 2012 Developer sürümünü kullandım. Siz de uygulamadan önce SQL Server sürümünüzün CDC’yi destekleyip desteklemediğini kontrol etmelisiniz.

## CDC Kullanım Adımları
İki aşamadan oluşur:
1. Veritabanı üzerinde aktif etmek
2. Tablo üzerinde aktif etmek

1.İlk olarak değişikliği yakalamak istediğimiz veritabanı üzerinde CDC’nin aktif edilmesi gerekir. (Bunun için aktif edecek kişinin sunucu rolünün SYSADMIN olması gereklidir.)
Daha öncesinde yarattığım “CDC” isimli veritabanı üzerinde CDC’yi aşağıda yazılan betik ile aktif ettim.
```sh
USE CDC
GO
EXEC sys.sp_cdc_enable_db
```
Sunucuda hangi veritabanı üzerinde CDC’nin aktif olduğuna Resim 4'te gösterilen betiği yazarak bakabiliriz. Eğer “name” sütununda bizim veritabanımız var ise ve ona karşılık gelen is_cdc_enabled sütunu 1 ise CDC, veritabanımızda aktif olmuş demektir. Yolumuza temiz bir şekilde devam edebiliriz. :)
```sh
SELECT name,is_cdc_enabled
FROM sys.databases
WHERE is_cdc_enabled = 1
```
Yukarıdaki betik çalıştıktan sonra, veritabanı altında “cdc” isminde bir şema ve bu şema içerisinde bazı tablolar oluşacaktır. Bu tabloları CDC’nin aktif edildiği veritabanında(CDC), sistem tablolarının (System Tables) altında görebilirsiniz. Bu tabloların detaylı açıklamalarını aşağıdaki tabloda açıkladım.

| Tablo Adı | Açıklama |
| ------ | ------ |
| cdc.captured_columns |CDC ile değişikliklerin takip edildiği kolonların listesini tutar.|
| cdc.change_tables |CDC ile değişikliklerin takip edildiği tabloların listesini tutar.|
| cdc.ddl_history | CDC aktif edildikten sonra yapılan DDL işlemlerinin tarihini tutar. |
| cdc.index_columns | CDC ile değişikliklerin takip edildiği kolonlara ait indexlerdeki değişikliği tutar.|
| cdc.lsn_time_mapping | Tüm işlemlerin yaptığı değişikliği ve zamanı tutar.Lsn bilgisine göre de işlemin hangi sırada yapıldığı bilgisini tutar. |

2.Tablo üzerinde CDC’nin aktif olup olmadığına Resim 6'da gösterilen betiği yazarak bakabiliriz.
```sh
USE [CDC]
SELECT [name],is_tracked_by_cdc
FROM sys.tables
GO
```
“Name “ sütununda yazan bizim tablomuzun ismine karşılık gelen is_tracked_by_cdc sütunundaki değer 0 ise, tablomuzda CDC aktif değildir demektir. O zaman aktif edelim. :)
Burada dikkat etmemiz gereken konu; tablomuzda CDC’yi aktifleştirmeden önce SQL Server Agent servisinin çalıştığından emin olmalısınız. Çünkü CDC’ye ait değişikliği alan ve temizleme işini yapan işler(“jobs”) tablo aktif olduktan sonra tanımlanacak.
Aşağıdaki betik ile tablomuzda CDC’yi aktifleştirebiliriz.(Bknz: Resim 7) Daha önce oluşturduğum “cdc_test” tablosu üzerinde Resim 8'de gösterildiği gibi, iki iş oluşturduğunu görüyoruz.
```sh
USE [CDC]
GO
EXEC sys.sp_cdc_enable_table
@source_schema = N'dbo',
@source_name = N'cdc_test',
@role_name = NULL
```
Buradaki parametreleri açıklayalım:
|Parametre| Açıklama |
| ------ | ------ |
| @source_schema |Kaynak tablonun ait olduğu şema ismi|
| @source_name |Kaynak tablonun ismi|
| @role_name |Değişen verilere erişim sağlamak için kullanılan veritabanı rol ismi.|
| @supports_net_changes | 0 veya 1 değerini alır.Bir kayıtta gerçekleşen tüm değişikliklerin net değişikik şeklinde özetleneceği anlamına gelir. |
| @captured_column_list |Değişikliğini takip edeceğiniz kolon listesi |

Bu işlemlerden sonra son bir kontrolümüz kaldı. Veritabanı altında “cdc” isminde şemanın altında bulunan tablolara Resim 10'da gösterildiği gibi, “_CT” uzantılı yeni bir tablonun eklenmesi lazım. O da oluştuysa, CDC’nin tablomuzda aktif olduğuna ve artık yapacağımız değişiklikleri takip edeceğine emin olabiliriz. :)

Bu işlemlerden sonra son bir kontrolümüz kaldı. Veritabanı altında “cdc” isminde şemanın altında bulunan tablolara “_CT” uzantılı yeni bir tablonun eklenmesi lazım. O da oluştuysa, CDC’nin tablomuzda aktif olduğuna ve artık yapacağımız değişiklikleri takip edeceğine emin olabiliriz. :)

“cdc.dbo_cdc_test_CT” isimli değişikliklerin tutulduğu tablomuzun nihai halinde tablodaki kolonların yanı sıra ek kolonları tutmaktadır .__$operation sütununun aldığı değerlerin ne anlama geldiğini aşağıda açıkladım.
|.__$operation Değeri| Açıklama |
| ------ | ------ |
| 1 |Silinen kaydı gösterir.|
| 2 |Eklenen kaydı gösterir.|
| 3 |Güncellemeden önceki kaydı gösterir.|
| 4 |Güncellemeden sonraki kaydı gösterir.|
__$update_mask sütunu, DML işlemleri sırasında değiştirilen sütunları temsil eden “bit”lerden oluşur.
Değişiklik tablolarını sorgulamak yerine, 2 tane fonksiyon ile tüm değişiklikler sorgulanabilir. Bu fonksiyonları, fonksiyonlar(Functions) klasörünün altında, “Table-valued Functions” klasörünün altında bulabilirsiniz.
· cdc.fn_cdc_get_all_changes_<şema ismi_tablo ismi> : Belirtilen zaman aralığında(LSN aralığı) birden çok değişikliği görüntülemek için kullanılır.
· cdc.fn_cdc_get_net_changes_<şema ismi_tablo ismi>Belirtilen zaman aralığında(LSN aralığı) değiştirilen her kayıt için net değişikliği görüntülemek için kullanılır.
Belli bir zaman içindeki değişiklikleri de görebiliriz. LSN’yi belirtilen süre için “cdc.lsn_time_mapping” sistem tablosundan alabiliriz. Bu tablodan bilgileri almak için; “sys.fn_cdc_map_time_to_lsn” fonksiyonunu kullanabilirsiniz.

Daha detaylı bilgi için  [Medium hesabımda anlattığım yazıya](https://medium.com/@bercemcatalkaya/cdc-change-data-capture-88e9ec6f51d7) bakabilirsiniz.
## CDC Nasıl Pasif Hale Getirilir?
İki aşamadan oluşur:
1.Tablo üzerinde pasif hale getirmek
```sh
EXEC sys.sp_cdc_disable_db
GO
```
2.Veritabanı üzerinde pasif hale getirmek
```sh
EXEC sys.sp_cdc_disable_table
@source_name = N'table_name'
@source_schema=N'dbo'
@capture_instance=N'dbo_table_name'
GO
```
>Dezavantaj Olarak Görülebilecek Özellikleri Nelerdir?
· Büyük ölçüde bakım ve yönetici eforu gerektirir.
· Sürekli izlenecek veriler değişiklik tablosunda tutulacak, aynı veya farklı veri dosyasında saklanacaktır. Bunun için tabii ki alternatif çözümler var. Örneğin, değişiklik tablosundaki verileri belli aralıklarla bir yere alıp, tablodaki verileri silmek.
· SQL Server Agent çalışmıyorsa, CDC çalışmaz.

## SONUÇ
Her sistemde olduğu gibi, CDC’nin de avantajları ve dezavantajları var. Fakat hız ve performans açısından sizlere kefil olabilirim. :)
Konu hakkındaki sorularınızı [Linkedin](https://www.linkedin.com/in/bercemozcelik/)  aracılığıyla her zaman sorabilirsiniz.

## Lisans
Berçem Özçelik 
