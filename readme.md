## Apache lucene; belge deposu ve tam bir metin arama motorudur,

    Çalışma mantığı;
    lucene dizinine bir belge eklediğinde sadece o belgeyi diske yazmaz, belgelerin hangi 
    değerleri içerdiğini izleyen bir veri yapısı olan "Lucane's inverted index" tutar.
    bir kelimeyi aradığınızda sadece o kelimeyi aramaz çok hızlı bir şekilde kaç belgenin
    eşleştiğini ve hatta hangi belirli belgelerin eşleştiğini söyleyebilir.

    Olayı şöyle düşünelim, bir belge yığınımız olsun, word,excel v.s. burada sadece word
    dosyalarını bulmak biraz zaman alacaktır. ancak eğer bu karışık yapıyı parçalara ayrılmış daha küçük gruplara 
    ayırırsak gerekli dosyaları daha hızlı buluruz.

    lucene, veri arama yapılarını sadece metin alanları için değil desteklediği tüm alan türleri için yapar.

    Lucene, çok hızlı bir şekilde çalışır ancak belli bir eşiğe kadar. Ancak bu eşik maalesef hesaplanabilir bir 
    değer değildir. Yarı güvenilir olarak test edilerek bulunabilir. Bunu anlamak için bir senaryo belirleyelim.

    Indexleme oranımız saniyede X belgeye yükselene kadar güzel çalışıyor olsun. X e gelince sorun olursa sınırımız 
    hakkında bir fikrimiz olur. Böyle bir durumda ilk seçenek daha fazla veri üretmekten ve arama yapmaktan vazgeçmek olabilir :)
    
    Ancak burada tercihlerimiz daha fazla işlem yapmak, bu nedenle yatay ya da dikey olarak genişleme yoluna gidebiliriz. 
    Burada arama yükünü dengelerken saniyede 2 kat belge depolayabilmek için lucene nin iki örneğini çalıştırıyoruz 
    ve her ikisine de yeniden dağıtım yapıyoruz. Böyle bir durumda iki fiziksel deponun veri kümesinin yarısını tuttuğu 
    bir mantıksal belge deposu elde ederiz. Burada bazı zorluklar vardır
    ilk olarak ölçek 3x,10x olduğunda lucene örneklerinde arama yaptığımızdan emin olmamız gereklidir ve bulunan her 
    bir surunu birleştirmemiz gereklidir. tüm bunlar olurken de hiç bir örneğin kesintiye uğamadığından emin olmalıyız. 
    İşte tam burada bir component yazarak bu işlemleri bizim içi yapmasını sağlarız böylece analiz işlemlerini başka bir 
    serviste okuma-yazma işlemlerini de componentte yapabiliriz. İŞTE, burada bu komponent yerine gelecek olan şey 
    Elasticsearh un ta kendisidir.

    Elasticsearch çalıştığında başlatılan işleme "Düğüm - Node" adı verilir. bir düğüm tarafından yönetilen
    Lucene örneklerine "shards - parça" denir.

    Node, Lucene' in işlevselliğini saran ve genişleten yeni arama yüzüdür. shards verilerimizin ölçeklendirme birimidir.

    Veri yükelemeye devam ettikçe elasticsearch daha fazla shard ekleyerek performansın mükemmel kalmasını sağlar. 
    Ancak bir süre sonra, elasticsearch ü barındıran makinelerdeki fiziksel kaynaklar tükenir. 
    Tek makinenin gerçekleştirebileceği iş sınırlıdır ve sonunda sunucunun belleği, CPU çekirdeği ve O/I 
    bant genişliği tükenir. ayrıca eslacticsearh üçalıştıran makine çevrimdışı olursa yerileri işlemek mümkün olamaz.

    Büyük sistemlerde hizetlerin ve tüm sunucuların çökmesi garanti edilir ve bunun uygulamayı etkilemeden 
    gerçekleşmesine izin vermemiz gerekir. buna "fault tolerace" denir.

    Bu sorunu çözmek için parçaların birden fazla sunucuya dağıtılması işi çözebilir. Yani elasticsearch birden fazla
    sunucuya kurulur ve çalışır ancak bu seferde sunucular bir birinden habersiz şekilde çalışacaklardır. 
    Bu nedenle Node ların ağ üzerinde bir birlerini otomatik olarak keşfedeceleri bir iletişim yoluna ihtiyaç vardır. 
    İşte bu yeni iletişim kanalı "inter-node communication" "transport layer" olarak adlandırılır. İşte böylece, 
    bir node diğer parçalarıda keşfederek onlarda da arama yapabiliyor. Birlikte çalışan ve birbirlerinden haberi olan 
    bu düğümlerin oluşturduğu yapıya "Cluster - Küme" denilir.

    tüm bu veriler, kümeyle(cluster) ilgili diğer meta verilerinin yanı sıra, her düğüm-node de "cluster state" te depolanır.

    Taşıma katmanı ve cluster state, veri kümesinin birden fazla sunucuya dağıtılmasına olanak tanır ve böylece 
    her bir makinedei yükü azaltır. Böylelikle hataya dayanıklı bir küme oluşturmak için temelimiz hazırdır.

    Ancak, bir düğüm-node çöktüğünde o düğümde depolanmış olan tüm veriler gider. Açık diğer düğümlerde arama 
    yapabilirsiniz ancak veriler kısıtlı gelecektir. Böyle bir durumda tam sonuç elde etmek için bir düğümde olan 
    her bir parçayı kümedeki farklı düğümlere yedeklemeniz gereklidir.

    "Replica shard - çoğaltma parçacığı" olarak adlandırılan bu yedek parça, her zaman "primary shard - orjinal parça"
    ile tamamen güncel olacaktır.

    Bunu anlamak için düşünelim;
    diyelim ki Node-0 da P0 parçamız olsun, ve Node-1 de P1 parçamız olsun. Eğer Node-1 çevrimdışı kalırsa 
    verilere ulaşılamaz. Bunu çözek için hiçbir zaman birincil parçalar ayın düğümde bulunamaz şeklinde bir kural 
    uyguluyoruz. Böylecek Node-0 da R1 Node-1 de R0 saklanıyor. Burada bir Node un ne zaman kapandığını bilmek için 
    Taşıma katmanını  ayarlayarak düğümleri-Node  sürekli olarak birbirleri ile iletişimde kalmaya zorlarız. 
    Bir Node a bilgi teslim edilmez ise cluster state, şuanda kullanılamayan birincil parçaların kopyalarını bulmak 
    için kullanılır ve bunlar birincil parçalara yükseltilir.

    Diyelim ki Node-1 düştü, böyle bir durumda Node-0 da bulunan P1 in Replica R1 i yükseltilerek yeni P1 haline gelir.

    Baktığımızda uygulamamız artık hızlı, ölçeklenebilir ve dayanıklıdır. Ancak havamız kısa ömürlüdür. :)

    Kapsam kayması;
    Diyelimki elasticsearch ün işlevselliği o kadar iyi halde ki geliştirme ekibimiz log ları tutmak ve aramak için 
    kullanmak istiyorlar. Şimdiye kadarki senaryolarda tek bir veri kaynağı ve veri tipi var idi. ancak artık farklı 
    bir veri kümesi tutmak istiyoruz.
    Böyle bir durumda log lar çok fazla hızlı büyüyecek ve okunduğundan daha fazla yazılacak. Bu nedenle 
    birincil ve kopya parçalaının sayısı için farklı gereksinimlere sahip bir yapıya ihtiyaç duyacağız.
    Bu durumda bir parçanın logları ya da anket gibi bir veri kümesini tuttuğunu belirtmek için her parçaya bir 
    "grup" etiketi eklememiz gerekiyor. Böylece elasticsearch tam olarak hangi veriyi sorgulayacağını bilecektir. 
    Ayrıca her bir belgenin alan adları, dizi, ip adresi, tam sayı v.s. oluş olmadığını da tutabiliriz.
    Her bir parça-shard ın aynı veri kümesinin bir bölümünü tuttuğu bir parça grubu olan bu gruplandırma kavramına 
    "index-dizin" denir. Alan düzeyindeki meta verilere "eşlemeler-mappings" adı verilir.

    Bir belgeyi saklarken index düzeyinde bir elasticsearch düğümüyle etkileşime girersiniz. depolamak için bir anket 
    yanıtınız mı var? anket yanıtları index ini belirtin:.

    Çıkarımlar;
    - Elasticsearch'ü bir sunucuda çalıştırmak nir süreç başlatır ve biz bu süreci Node-Düğüm olarak adlandırırız.
    - Bir Node, verileri her biri bir shard olarak adlandırılan Apache Lucene'nin birden çok örneğine depolar. 
    İk tür Shard vardır. Primary-Shard, Replica-Shard veriler sadece birincil shard a başarı ile yazıldıktan sonra 
    replica shard a yazılır.
    -  Bir veri kümesi , bir dizin oluşturan bir veya daha fazla parçada(shard) depolanır. Dizindeki parçaların(shards)
    sayısı birçok faktöre bağlıdır ve tümünü kapsayan doğru bir cevap yoktur.
    - Shard ın temel amacı index in ölçeklenmesine izin vermektir.
    - Ayrı sunucularda çalışan düğümler bir küme oluşturacak şekilde yapılandırılabilir. Hata töleransı sağlamak 
    için parçalar kümedeki düğümler arasında dağıtılabilir.

## Otomasyon ile ElasticSearch ü kontrol etmek yapılandırmak. ANSIBLE

    Genel olarak elasticsearch tek node şeklinde kurmak ve kullanmak kolaydır ve önemsizdir. Büyük firmlar ve işlemler 
    için mutlaka otomsyonla ölçkelendirme yapmanız gereklidir.Bu konuda elasticsearch bize yardımcı oluyor ve ansible a 
    destek veriyor.

    İlk olarak, sistemi kontrol etmek üzere ibr ansible kontrol düğümü oluşturuyorum. Bununla sistemi kontrol edeceğim.

## Elastic Search Giriş

### Kavramlar

####    Index:

    bir biri ile iliişkili dokümanlara index adı verilir.
```
    {
        "id": 1,
        "title": "Java SE",
        "content":  "Java yazılım dili, platform bağımsız ilk yazılım dilidir."
    }
```

####    Data Stream:
    Zaman temelli data dır. her gün için ayrı ayrı index tablosu oluşturabilirsiniz. Bu datayı bir alias vererek isimlen
    direbilir ve yönetebiliriz.

####    Cluster:

####    Node:

####    Shard:

####    Replica:

####    Text Analyzing:

    Elsticsearch, gelen data nın içinde text temelli bir data geliyor ise bunu bir analiz sürecinden geçirir. Burada 
    mantık büyük montanlı datların hızlıca full text search yapılabilmesini sağlar. 
    Burada işlemler şöyle ilerler;
    Tokenization; ilk olarak gelen text boşluk karakterine baz alınarak javada ki String[] şeklinde kayıt edilir. Burada
    her bir dizi elemanı token olarak adlandırılır.
    Normalization; token haline getirilen datanını zenginleştirilmesi sağlanır. Belli kelimelerin kökleri alınır,
    Yukarıdaydı - Yukarı. ya da eş anlamlıları eklenir. İsim - ad gibi
    Google içinde yapılan aramaları örnek olarak gösterbiliriz.
    normalizasyon süreçlerini zenginleştirmek için biz belirleyebiliriz.

    NOT: Burada analiz sürecinin nasıl organize edebileceğimizi anlatalım.

    
#### Inverted Index

    Text analizlerinden sonra datanın kayıt edildiği yerdir. Her bir full-text field için incerted-index oluşturulur. As
    lında olay bir kitabın içinde geçen kelimelerin nerelerde geçtiğini gösreten index i gibi düşünebiliriz.
````
    KeyWord - Document Index Number
    Yukarı  |   1,43,677,5678

````
    bir inverted index oluşumu için sıralama şöyledir; mevcut metinde gereksiz özel karakterler filtrelenir, kalan
    kelimeler ayrıştırılarak dizi haline getirilir. tüm keywordler küçük hale dönüştürülür.
    * strip filter
    * whitespace tokenizer
    * lowercasing tokenizer

    Bir örnek vermek gerekirse, inverted index tablosu şöyle görünür;
    - Bu Akşam Gidelim  
    - Akşam Gidelim
    Keyword     |   Frequency   | DocumentId
    Bu              1             1
    Akşam           2             1,2
    Gidelim         2             1,2  


#### Relevancy

    Elasticsearch için ilişki skoru değeridir. Bulunan sonuçlar en alakalıdan en alakasıza doğrur sıralanır. Bu score
    belirlenirken. Best Match relevancy algorithm (BM25) kullanılır.
    

### Kurulum ve Yapılandırma

    1- Docker Üzerinde ayağa kaldırma
    2- Kibana UI
    3- Örnek Data Yükleme

### Elasticsearch API Kullanımı

    API Türleri;    
    - Indexing API
    - Searcing API
    - Reading API
    - Updating API
    - Deleting API
    Bu apilerin 2 tür çalışma şekilleri var dır. Tek döküman ekeleme ve çokul döküman ekleme
    - Single Document APIs
    - Multi Document APIs


#### API Kullanım syntax i ve Create Document(indexing)

    [HTTP Method Type - KQL] [Server:Port] / [IndexName]/_doc/ Document Id

    PUT products/_doc/2
    {
        "kategori": "Telefon",
        "Marka": "Samsung",
        "Detay": "bu telefon yeni nesil AI kullanılarak işlerinizi kolaylaştırmaktadır."
    }

    REsponse :
    {
        "_index": "urunler",
        "_id": "1",
        "_version": 1,
        "result": "created",
        "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
        },
        "_seq_no": 0,
        "_primary_term": 1
    }

#### Retrieve Document - Datayı almak

    GET urunler/_doc/1
```
    {
      "_index": "urunler",
      "_id": "1",
      "_version": 1,
      "_seq_no": 0,
      "_primary_term": 1,
      "found": true,
      "_source": {
        "kategori": "Telefon",
        "Marka": "Samsung",
        "Detay": "bu telefon yeni nesil AI kullanılarak işlerinizi kolaylaştırmaktadır."
      }
    }
```
    GET urunler/_source/1
````
{
  "kategori": "Telefon",
  "Marka": "Samsung",
  "Detay": "bu telefon yeni nesil AI kullanılarak işlerinizi kolaylaştırmaktadır."
}

````
    GET _cat/shards/urunler
```
urunler 0 r STARTED 1 6.4kb 6.4kb 172.19.0.4 es02
urunler 0 p STARTED 1 6.3kb 6.3kb 172.19.0.5 es03
```

#### Identifier

    PUT urunler/_doc/1 -> İŞlemi temel olarak şöyle çalışır. Eğer eklenilen Id de bir kayıt yok ise yeni bir data yaratır, Ancak
    aynı Id ye sahip bir data var ise o datayı günceller.
    POST urunler/_doc -> Böyle kullanılır ise elastifcsearch datayı yeni bir UUID ile kayıt eder.
    PUT urunler/_create/1 -> kayıt yok ise ekler, ancak eğer katıt var ise exception fırlatır.

    Burada best practice olan id nin bizim tarafımızdan yönetilmesidir.


####    Peki Indexing nasıl meydana gelir?

        JVM üzerinden çalışan elasticsearch ilk olarak bizim index e gönderdiğimiz datayı öncelikle geçici hafızaya alır,
    (hafızaya alınma ve bekleme süremiz 1 saniyedir. istenirse bu değer değiştirilebilir.)
    sonra bu data primary shard içindeki segment lere atanır ve aramaya hazır hale gelir. Bir sonraki aşamada ise segment
    lerde bulunan data File Sistem e aktarılır.
    İŞte burada datanın eklenirken bekleme süresini belirtebiliyoruz.
```
PUT urunler/_settings
  {
  
    "index": {
        "refresh_interval":"30s"
    }
  }
```
```
PUT urunler/_doc/1?refresh
// 3 kullanımı var 
// Refresh = true -> anlık armaya kat
// Refresh = false(default)
// Refresh = wait_for
```



#### Updating Documnets

```
POST urunler/_update/1
{
  "doc": {
      "kategori1": "Telefon",
      "mategori": "melefon",
      "Marka": "Samsung",
      "Detay": "bu telefon yeni nesil AI kullanılarak işlerinizi kolaylaştırmaktadır."
    }

}
```

#### Deleting Document

```
DELETE urunler/_doc/1
```

#### Retrieving documents with HEAD

```
HEAD urunler/_doc/1
// return 200 - OK
```

#### Retrieving multiple documents 

````
GET urunler/_mget
{
  "ids":[1,2]  // or ["1","2"]
}

````


```
GET urunler/_search
{
  "query": {
    "match_all": {}
  }
}
```

```
GET _mget
{
  "docs":[
  {
    "_index": "urunler",
    "_id": 2
  },
  {
    "_index": "products",
    "_id": 1
  }
  
  ]
}
```

#### Custom response - source includes & source excludes
    Belli alanları listelemek ya da listelememek için kullanıyoruz.
```
GET urunler/_doc/1?_source_includes=kategori
GET urunler/_doc/1?_source_exludes=kategori

```














### Data Types

    text: String values - logs, blog content, product descriptions, product title ... (unstructured text and analysis process)
    byte/short/integer/long: number
    float/double: floating-point number
    boolean: true/false
    object: Json object
    keyword: analiz yapmak itemediğim alanlar, email,phone,number, Tc(structured text and don't analysis process)


####    Date type: yyyy-MM-dd ->ISO 8601
    tarih datasıdır, formatı değiştirilebilir,. mappings ile özel tanımlanır.

####    Numeric types:
    - byte 8bit
    - short 16bit
    - integer 32bit
    - long 64bit
    - float 32bit
    - double 64bit
    - half_float 16bit precision
    - half_double 32bit precision

####    Boolean Type:

####    range type / date_range

    belli bir data aralığı ya da tarih aralığı verilebilir.
    25-59
    15/06/2030 - 16/06/2031

####    Object data type
    aslında json data type dır.

####    ** Fattened data type
    özellikle performans odaklı kullanımlara açık olan bir veri tipidir. aslında bir ekleme işlemlerinde gönderilecek 
    alanların sayısı belli bir kritere uymuyor ise bu şekilde tanımlarız. burada dikkat etmemiz gereken husus flatten 
    olarak işaretlediğimiz alan keyowrd olarak işlem görecektir. bu sefer analiz süreci gerçekleşmeyecektir.

    DİKKAT!!!!
    Object olarak alanları belirlenerek tanımlanan bir index in analiz süreci olacakken, flatten olarak işaretlenen
    alanlar analiz süecinden geçmeyecek ve arama yetenekleri kısıtlı olacaktır.

#### Nested data type - Complex and Relational Type

    özelleştirilmiş Object gibi düşünelim. 
```
{
    "id": 1,
    "kullanici": [
        {
            "name": "murat",
            "surname: "taş"
        },
        {
            "name": "canan",
            "surname: "baş"
        }
    ]
}
```
    Bu kullanımda farklılık aslında aram tarafında belli oluyor. Şöyleki;
    kullanici.name = "murat";
    kullanici.surname= "baş";
    şeklinde arama yaparsam iki nesneyide bana getirir. ancak doğru olan aslına adı murat soyadı baş olanı getirmeli
    burada her bir alanı bağımsız arıyor. İşte tam burada biz kullanıcı tipini Nested olarak ayarlarsak arama yaparken
    verdiğim dataların büünlüğüne göre arama yapacaktır.
    *** DİKKAT!! burada yapılan işlem ilişkilendirmedir bu nedenle maliyetli bir işlemdir. gerekli ise kullanın.

#### Join data type

    eklenen satırların bir birleri ile ilişkilendirilmesi için kullanılır. yani DB de ki ilişkilere benzer şekilde 
    aynı tablodaki verilerin ilişkilerini ayarlıyor.



### Mapping 

    kayıt edeceğimiz datayı tüm alanlarını belirleyerek tanımlama işlemidir. SQL deki crete table ile aynı şeydir.

#### Dynamic Mapping 
    kayıtlarını eklemeden önce bir tip belirlemez iseniz, elasticsearch sizin için tipleri otomatik belirler.
```
PUT kategori/_doc/1
{
  "id": 2,
  "ad": "Bilgisayar",
  "parentId": 0
}

GET kategori/_mapping
//results
{
  "kategori": {
    "mappings": {
      "properties": {
        "ad": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "id": {
          "type": "long"
        },
        "parentId": {
          "type": "long"
        }
      }
    }
  }
}
```
    Burada dikkat etmemiz gereken bir husus var, ad alanı için hem text hemde keyword tanımı yapılmıştır. multi types
    kullanmış, hem tüm kelimeler ayrı ayrı indexlenirken hemde kelimenin bütünü indexlenecektir.

#### Explicit Mapping 
    kendi şemamızı belirlemek için istediğimiz index tablosunu belirliyoruz.
```
PUT auth
{
  "mappings": {
    "properties": {
      "id": {"type": "text"   },
      "user_name": {"type": "text"},
      "password": {"type": "text"},
      "photos":{
        "properties": {
          "thumpnail":{"type":"text"},
          "image_url": {"type": "text"}
        }
      }
      
    }
  }
}
```
```
GET auth/_mapping

PUT auth/_doc/1
{
  "id": "11",
  "user_name": "muhammedhoca",
  "password": "Aa123456***",
  "photos":{
    "thumpnail": "https://",
    "image_url": "http://"
  }
}
GET auth/_doc/1

```

    var olan bir indexi içinde yeni property ler ekleyebilirsiniz.
```

PUT auth/_mapping
{
  "properties":{
    "create_at": {"type": "long"}
  }
}

```

#### re-index

    sistemi tasarladık ve dataları kayıt ettik ama baktık ki alan yanlış tanımlanmış. böyle bir durumda alanda değişiklik 
    yapamazsınız. İşte böyle bir sorunu çözmek için kullanılır.
    Bu işlem şu adımlarla yapılır;
    1- yanlış olduğunu düşündüğünüz index in doğru yapılandırılmış halini içeren yeni bir index oluşturun.
    2- diğer hatalı kayıt içindeki dataları yeni index in içine taşıyacağız.
    3- re-index yaparız
```
POST /_reindex
{
    "source": {
        "index": "kategori"
        },
    "dest": {
        index: "kategori2"
    }
}
```


### Searching Types

    İlk olarak searching nasıl çalışır bir bakalım. ilk olarak aram geldiğinde master shard sorguyu alır ve kendi dataları
    içerisinde aramaya başladıktan sonra diğer shardlarda da aramayı başlatır. böylelikle işlemleri başlatan node a 
    Active Coordinator node adı verilir. bu node tüm shardlardan gelen datları birleştirerek sonuç döner.

#### Rest API - Query DSL(Best practice), Query Request

```
GET auth/_search?q=user_name:muhammet

GET auth/_search
{
  "query": {
    "match": {
      "user_name": "muhammedhoca"
    }
  }
}
```

#### Search Context

    1- Query context: çıkan sonuçların bir score değeri vardır

```
GET auth/_search
{
  "query": {
    "match": {
      "user_name": "muhammedhoca"
    }
  }
}
```

    2- Filter context: çıkan sonuçların score değeri yoktur, yapılan sorgular cache lenir ve performansı iyidir.


```
GET auth/_search
{
  "query": {
    "bool": {
        "filter": [
            "match": {"user_name": "muhammedhoca"}
        ]      
    }
  }
}
```

### Term-level queries

####    structured datalar üzerinde arama yapmayı ifade eder.
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "term": {
      "customer_first_name.keyword":{
        "value": "yasmine",
        "case_insensitive": true
      }
    }
  }
}
```
####    terms çoklu arama yapmak
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "terms": {
      "customer_first_name.keyword":["Yasmine","Pia"]
    }
  }
}
```
####    ids ile arama yapmak
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "ids": {
      "values":["BmBLSY0BHkxOm1sws1WB","NGBLSY0BHkxOm1sws1WB"]
    }
  }
}
```
####    exists bir alanın olup olmadığını aramak için
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "exists": {
      "field":"order_id"
    }
  }
}
```
#### prefix ile belli bir ifade ile başlayanları aramak
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "prefix": {
      "customer_first_name.keyword":"Son"
    }
  }
}
```
#### range ile aralık sorgulamak.
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "range": {
      "taxful_total_price":{
        "lte": 50,
        "gte": 20
      }
    }
  }
}
```
#### Wildcard ile özel arama yapmak (*, ?)
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "wildcard": {
      "customer_first_name.keyword":{
        "value": "So*"
      }
    }
  }
}
```
#### Fuzzy ile hata toleransı girerek arama yapmak. tolerans  aralığına bakalım.
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "fuzzy": {
      "customer_first_name.keyword":{
        "value": "ony",
        "fuzziness": 2
      }
    }
  }
}
```
#### pagination ile sayfalama yapmak
    size: kaç kayıt getireceğini belirtirken
    from: bulunan kayıtların hangisinden itibaren başlayacağını belirtir. 
```
POST kibana_sample_data_ecommerce/_search
{
  "size": 2, 
  "from": 2, 
  "query": {
    "multi_match": {
      "query": "Men's Clothing",
      "fields": ["category"]
    }
  }
}

```
#### response da include,exclude
```
POST kibana_sample_data_ecommerce/_search
{
  "_source": {
    "includes": ["_id","customer_full_name","email"]
    }, 
  "size": 2, 
  "from": 2, 
  "query": {
    "multi_match": {
      "query": "Men's Clothing",
      "fields": ["category"]
    }
  }
}

```
#### sort ile datalarımızı sıralamak
```
POST kibana_sample_data_ecommerce/_search
{
  "_source": {
    "includes": ["_id","customer_full_name","email","customer_full_name"]
    }, 
  "size": 20, 
  "from": 2, 
  "query": {
    "multi_match": {
      "query": "Men's Clothing",
      "fields": ["category"]
    }
  },
  "sort": [
    {
      "customer_first_name.keyword": {
        "order": "desc"
      }
    }
  ]
}

```


### Full text Queries

    unstructure datalar üzerinde arama yapma,
    bu datalarda artık match query ile arama yapacağız. Burada davranışımız değişiyor.

#### Match Query
```
POST kibana_sample_data_ecommerce/_search
{
  
  "query": {
    "match": {
      "category": "Men's" //or "men's"
    }
  }
}
// DİKKAT!! kelime ararken kelimenin tam ismi olmalı.
// aramalar en alakalıdan en alakasıza devam eder. customer_full_name= Yahya Goodwin
```

```
POST kibana_sample_data_ecommerce/_search
{
  
  "query": {
    "match": {
      "customer_full_name": {
        "query": "yahya goodwin",
        "operator": "and",
        "fuzziness": 2
      }
      
    }
  }
}
```
#### Multi Match Query
```
POST kibana_sample_data_ecommerce/_search
{
  
  "query": {
    "multi_match": {
      "query": "Yahya",
      "fields": ["customer_first_name","customer_last_name","customer_full_name"]
    }
  }
}
```

#### Match Boolean Prefix Query

```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "match_bool_prefix": {
      "customer_full_name": "Sultan al Moran"
    }   
  }
}
// son kelime önce alınarak sorgulama yapılır. ancak tüm kelimeler aramaya dahil edilir.

```

#### Match Phrase Query
    verilen sıralamaya bağlı kalarak tam ifade geçen liste getirir. Yani and  de kelimeler arasına kelime gire bilir
    ancak bunda tam eşleşme arar.
```
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "match_phrase": {
      "customer_full_name": "Sultan al Moran"
    }   
  }
}
```

#### Match Phrase Prefix Query

````
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "match_phrase_prefix": {
      "customer_full_name": "Sultan ahmet M" -- eşleştiği kayıt Sultan ahmet Murat
    }   
  }
}
````


### Compound and Aggregation Queries

#### must query, sonuçlar mutlaka eşleşmeli
    
````
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "geoip.city_name": {
              "value": "New York"
            }
          }
        }
      ],
      "must_not": [
        {
          "range": {
            " taxful_total_price": {
              "lte": 45
            }
          }
        }
      ],
      "should": [
        {
          "term": {
            "category.keyword": {
              "value": "Women's Clothing"
            }
          }
        }
      ],
      "filter": [
        {
          "term": {
            "manufacturer.keyword": "Tigress Enterprises"
          }
        }
      ]
    }
  }
}
````

#### Compound Query

```
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "customer_full_name": "Morris"
          }
        },
        {
          "term": {
            "customer_gender": {
              "value": "FEMALE"
            }
          }
        }
      ],
      "must_not": [
        {
          "range": {
            " taxful_total_price": {
              "lte": 45
            }
          }
        }
      ],
      "should": [
        {
          "term": {
            "category.keyword": {
              "value": "Women's Clothing"
            }
          }
        }
      ],
      "filter": [
        {
          "term": {
            "manufacturer.keyword": "Tigress Enterprises"
          }
        }
      ]
    }
  }
}
```

#### Terms aggr

```
GET kibana_sample_data_ecommerce/_search
{
  "aggs": {
    "istedigini_ver": {
      "terms": {
        "field": "category.keyword"
      }
    }
  }
}

Results:
"aggregations": {
    "istedigini_ver": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "Men's Clothing",
          "doc_count": 2024
        },
        {
          "key": "Women's Clothing",
          "doc_count": 1903
        },
        {
          "key": "Women's Shoes",
          "doc_count": 1136
        },
        {
          "key": "Men's Shoes",
          "doc_count": 944
        },
        {
          "key": "Women's Accessories",
          "doc_count": 830
        },
        {
          "key": "Men's Accessories",
          "doc_count": 572
        }
      ]
    }
  }
}
```

```
GET kibana_sample_data_ecommerce/_search
{
  "_source": false, 
  "aggs": {
    "istedigini_ver": {
      "terms": {
        "field": "category.keyword",
        "size": 2,
        "order": {
            "_key": "asc" // or _count
        }
      }
    }
  }
}
// resuts
"aggregations": {
    "istedigini_ver": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 3482,
      "buckets": [
        {
          "key": "Men's Clothing",
          "doc_count": 2024
        },
        {
          "key": "Women's Clothing",
          "doc_count": 1903
        }
      ]
    }
  }
}
```

#### Range aggregate bucket

```
GET kibana_sample_data_ecommerce/_search
{
  "_source": false, 
  "aggs": {
    "istedigini_ver": {
      "range": {
        "field": "taxful_total_price",
        "ranges": [
          {"to": 50},
          {
            "from": 51,
            "to": 100
          },
          {"from": 101}
        ]
      }
    }
  }
}
//results
"aggregations": {
    "istedigini_ver": {
      "buckets": [
        {
          "key": "*-50.0",
          "to": 50,
          "doc_count": 1633
        },
        {
          "key": "51.0-100.0",
          "from": 51,
          "to": 100,
          "doc_count": 2011
        },
        {
          "key": "101.0-*",
          "from": 101,
          "doc_count": 944
        }
      ]
    }
  }
}
```

#### Metric aggregations

```
GET kibana_sample_data_ecommerce/_search
{
  "_source": false, 
  "aggs": {
    "ortalama": {
      "avg": {
        "field": "taxful_total_price"
      }
    }
  }
}
```
```
GET kibana_sample_data_ecommerce/_search
{
  "_source": false, 
  "query": {
    "term": {
      "category.keyword": {
        "value": "Men's Clothing"
      }
    }
  }, 
  "aggs": {
    "ortalama": {
      "avg": {
        "field": "taxful_total_price"
      }
    }
  }
}
```





