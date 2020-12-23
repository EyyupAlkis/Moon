---
layout: post
title: "Kotlin Coroutine"
date: 2019-12-03
excerpt: "Kotlin Coroutine"
tags: [Coroutine, Jetpack Components, Coroutine Nedir]
feature: https://miro.medium.com/max/4800/1*hOir8SBFtIAHTng6F-VxIA.png
comments: true
---
# Kotlin Coroutines
* Coroutine Nedir ? 

Coroutines = Co`operation` + Routines

> En basit tanımı: Çalışması askıya alınabilen birbirleri arasında  geçişleri seri olan fonksiyonlardır diyebiliriz. Bahsi geçen geçiş, thread üzerinde koşarken birinin askıya alınıp diğerinin çalıştırılması kastedilmektedir. Geliştirci ekibin `lightweight thread` olarak tanımlama sebebi; normalde bir fonksiyonu koşarken bir thread ihtiyacımız  olur o thread üzerinde koşarız  ama yüzlerce ayrı fonksiyonu yüzlerce ayrı thread'de koşmaya kalkarsak stack-over-flow (out of memory)hatası  alırız. Ama aynı şeyi Coroutine için düşündüğümüzde hata almayız ve coroutine'lerimiz çalışmaya devam eder. Suspend özelliğinden ötürü fonksiyon askıya alınıp başka fonksiyonları işletip tekrardan askıya alınan fonksiyon devam ettirilebilmekte. Bu nokta  Coroutine'i öne çıkaran kısımdır. Coroutine'nin sunduğu en önemli özellikler; asnyc (asenkron) ve concurency (eşzamanlılık). Yani bir işi veya iş dizisini çalıştırırken bunların çalışma şekil ve ve çalışacakları thread'leri esnek bir şekilde belirlememizde oldukça kolaylık sağlamaktadır.

Peki ama bunları yapabilen hali hazırda kullanılan yapılar bulunmakta. Mesela `AsyncTask` ve `RxJava`. Coroutines bu iki kütüphaneye karşı ayrı ayrı avantajları bulunmakta. AsyncTask'ta olduğu gibi boilerplate koda gerek olmaması, RxJava'da threading ve asenkron işlemleri yapabilmek için daha az karmaşıklık ve öğrenme eforu gerektirmesi. Fakat Coroutine, RxJava'nın reaktif programlama mantığına ve kullanışlı operatörlerine bir alternatif değildir. Eğer konu async işlemler ve threading ise coroutines bir adım öne çıkmaktadır.

Aşağıda bu üçlüye ait örnek kodlar bulunmaktadır.

```java
    private final class LongOperation extends AsyncTask<Void, Void, String> {

        @Override
        protected String doInBackground(Void... params) {
            for (int i = 0; i < 5; i++) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // We were cancelled; stop sleeping!
                }
            }
            return "Executed";
        }

        @Override
        protected void onPostExecute(String result) {
            TextView txt = (TextView) findViewById(R.id.output);
            txt.setText("Executed"); // txt.setText(result);
            // You might want to change "executed" for the returned string
            // passed into onPostExecute(), but that is up to you
        }
    }
```
```kotlin
getBooks().subscribeOn(Schedulers.io())
          .observeOn(AndroidSchedulers.mainThread())
          .subscribeWith(new DisposableObserver<Book>() {
              @Override
              public void onNext(@NonNull Book book) {
                  // You can access your Book objects here
              }

              @Override
              public void onError(@NonNull Throwable e) {
                  // Handler errors here
              }

              @Override
              public void onComplete() {
                  // All your book objects have been fetched. Done!
              }
          });
```
```kotlin
suspend fun doOperation1() {
    delay(1000L)
}

suspend fun getSomeResult(): String { 
    delay(2000L)
    return "hello"
}

fun main(){
  runBlocking {
    doOperation1()
    println("operation1 done after 1 second")
    val result = getSomeResult()
    println("operation2 done after 2 seconds and result: $result")
  }
}
```
* Coroutine ne sağlar ? 

> Main thread'ı meşgul edecek uzun süreli işler üstlenmesine

> Main safety: coroutine fonksiyonun main thread'ı tehdit etmeyecek şekilde yapılanmayı sağlar.

> Ve bunları sağlarken yukarda bahsi geçen avantajları beraberinde getirir.


* 1- Uzun Süreli İşler 

Bunlara örnek olarak, database'den uzun listeleri query etmeyi, yoğun cpu işlemleri gerektiren dosya ve data işleme işlemleri veya network üzerinden yapılacak data aktarımları sayılabilir. Bu işleri yaparken main thread'i es geçmek ve yükü diğer thread'lere dağıtmak hayati öneme sahiptir. 

Network çağrıları üzerinden örnekler üzerinden ilerlemek gerekirse;

Bu örnekte soru listesinin sunucudan çekildiği ve bir callback ile ui'ın beslenildiği görüyoruz.`get` metodu main thread'ten çağrılmasına rağmen işlerini arka planda yapmakta. Bu yapıya Retrofit ile aşinayız. 
```kotlin
class ViewModel: ViewModel() {
   fun getQuestions() {
       get("denizbank.fastpay.com") { result ->
           show(result)
       }
    }
}
```
Bu örnekte ise aynı işin Coroutine ile nasıl basitleştiğini görüyoruz. Callback'ler ile uğraşmadan daha okunabilir her açıdan efordan tasarruf edebileceğimiz bir yapı karşımıza çıkmakta. Ve bunu main thread'i bloklamadan yapmaktadır. Kod bloğunun ikinci kısmını daha sonra ele alacağız.

```kotlin
// Dispatchers.Main
suspend fun getQuestions() {
    // Dispatchers.Main
    val result = get("denizbank.fastpay.com")
    // Dispatchers.Main
    show(result)
}

suspend fun get(url: String) = withContext(Dispatchers.IO){/*...*/}
```
Peki Callback'ler olmadan Coroutine bu işi nasıl yapıyor. Bunun arka planına kısa bir bakış atalım.
Burdaki anahtar kelimeler `suspend` ve `resume`. Bunları eski yapılarla karşılaştırmak gerekirse suspend yerine invoke ve resume yerine return kelimelerini yerleştirebiliriz. 

> `suspend` anahtar kelimesini gördüğümüz yerde anlayacağımız şey; bu fonksiyonun suspend edilebilir olduğunu yani işi bitene kadar durucağını ama aynı zamanda var olduğu threadin (eğer biz isteyerek blocklamazsak) blocklanmayacağını belirtiyor.

> resume anahtar kelimesinde ise; mevcut Coroutine'in main thread'de sonucunun yansımasına olanak sağlar. 

Unutulmaması gerekenler:

i - Suspend fonksiyon demek main thread dışında çalışan bir fonksiyon demek değildir. Suspend olup main thread üzerinde de çalıştırabiliriz.

ii - Suspend bir fonksiyon ancak bir suspend fonksiyondan veya Coroutine bir block'tan çağrılabilir. [launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) bloğu bunlara örnek verilebilir.

![Medium 1](https://miro.medium.com/max/1400/1*U24_ZyMJKI_c2qMspCXxZw.gif)

Gif'ten anlaşıldığı üzere suspend fonksiyon işleme noktasına geldiğinde thread üzerindeki değişkenleri arkada stack(memory) üzerinde kaydedilip mevcut thread bloklanmadan Coroutine işlenip daha sonra suspend edildiği thread'e aktarılmakta. Bu bahsi geçen ui thread ise kullanıcı bu sırada ekranı kullanmaya devam edebilmektedir. 

![Coroutine akışı](https://github.com/EyyupAlkis/coroutines/blob/main/coroutine1)

* 2 - Coroutine ile Main Thread Güvenliği

Kotlin Coroutine'de iyi yazılmış bir suspend fonksiyon main thread'i tehdit etmez. Ne yaptıklarından bağımsız olarak her thread'den çağrılabilirler.

Fakat yukarda da bahsedildiği gibi her iş main thread veya background thread için iyi bir seçenek olmayabilir. Network çağrıları, json parse edilmesi, database CRUD işlemleri veya uzun dataların işlenmesi gibi işler için farklı farklı stratejiler geliştiririz.

Kotlin Coroutine'de bunun için aynı şeyi düşünmüş ve bize thread switch edebileceğimiz `Dispatcher` diye bir yapı sunmakta. Ve switch edebileceğimiz üç adet thread sağlamaktadır. Bunlar `Main`, `IO` ve `Default`. Bunların kullanımlarını aşağıda detaylı inceleyeceğiz. 

Daha önce bahsi geçtiği üzere fonksiyonu suspend şeklinde tanımlamak onu main thread'ten uzaklaştırmaz. Aksine Dispatcher ile thread bilgisi sağlanmamış her suspend fonksiyon main thread'te koşacaktır. Hatta main thread'te yapacağımız çok küçük ve  acil işler için [Dispatchers.Main.immediate](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-main-coroutine-dispatcher/immediate.html) yapısını kullanmamız bize avantaj sağlayacaktır.



```kotlin
+-----------------------------------+
|         Dispatchers.Main          |
+-----------------------------------+
| Main thread on Android, interact  |
| with the UI and perform light     |
| work                              |
+-----------------------------------+
| - Calling suspend functions       |
| - Call UI functions               |
| - Updating LiveData               |
+-----------------------------------+

+-----------------------------------+
|          Dispatchers.IO           |
+-----------------------------------+
| Optimized for disk and network IO |
| off the main thread               |
+-----------------------------------+
| - Database*                       |
| - Reading/writing files           |
| - Networking**                    |
+-----------------------------------+

+-----------------------------------+
|        Dispatchers.Default        |
+-----------------------------------+
| Optimized for CPU intensive work  |
| off the main thread               |
+-----------------------------------+
| - Sorting a list                  |
| - Parsing JSON                    |
| - DiffUtils                       |
+-----------------------------------+
```

```kotlin
// Dispatchers.Main
suspend fun getQuestions() {
    // Dispatchers.Main
    val result = get("denizbank.fastpay.com")
    // Dispatchers.Main
    show(result)
}
// Dispatchers.Main
suspend fun get(url: String) =
    // Dispatchers.Main
    withContext(Dispatchers.IO) {
        // Dispatchers.IO
        /* perform blocking network IO here */
    }
    // Dispatchers.Main
    
```
Kod bloğunda `withContext`yapısı gözünüze çarpmıştır. Bu yapıyla thread değişimini seri şekilde yapıp aynı işin farklı kısımlarını farklı thread'ler ile yapabiliriz.



> Kotlin Coroutine'nin şimdiye kadar işleri kolaylaştırdığından suspend ve resume'ın nasıl bir arada ahenk içinde çalıştığını gördük. Fakat işlerin kolaylaşmasının yanında takip edilemeyecek kadar iş yığınlarını çalıştırırken kontrolüne değinmedik. Bundan sonra buraya parmak basacağız.

## Coroutine'lerin Takibi

Birbirine bağımlı işler, sıralı işlemler veya paralel çalışması gereken fonksiyonları göz önünde bulundurun ve bunlardan onlarcasını düşünün. Hata alınması durumunda geri kalan işlerin iptal edilip edilmemesine veya hatayı ele almak gibi işlemleri de düşündüğümüzde işler iyice içinden çıkılamaz hale geliyor. Potansiyel memory leak'leri önlemek, gereksiz cpu harcamak veya hata durumunda yapılmaması gereken network çağrılarını önlemek uygulamamızın daha sağlıklı bir yapıda kalkınmasını sağlar. Ve Coroutine bütün bunlar için çözüm sunmaktadır.

Coroutine leak'lerini önlemek için [structured concurrency](https://kotlinlang.org/docs/reference/coroutines/basics.html#structured-concurrency) ortaya konmuştur. Peki nedir bu? `Structured concurrency` dediğimiz yapı bize bütün koştuğumuz Coroutine'lerin kontrolü için en iyi pratikleri ve Kotlin özelliklerini sunmaktadır. Peki tam olarak ne işe yarar? 

> Belirli bir olgu sonucunda artık ihtiyacımız olmayan  Coroutine'lerin iptal edilmesine

> Koşan Coroutine'lerin durumlarını izlememize

> Herhangi bir hata alan Coroutine'in hata sinyallerini yakalamamıza olanak sağlar.

* 1 - Scope ile Coroutine İptali

Kotlin ile beraber, coroutine'leri `CoroutineScope` dediğimiz bir scope ile koşmak durumundayız. Bu scope ile koşan ve suspend edilmiş coroutine'lerimizi takipedebiliriz.Bu scope bir önceki bölümde bahsettiğimiz `Dispatchers`ın aksine coroutine'leri kontrol etmemize değil onları takip altında tutmamıza yarar.
Bütün Coroutine'lerin takip altında olduğundan emin olmak için Kotlin, bir coroutine'i `CoroutineScope` olmadan koşmamıza izin vermez. Daha iyi kavrayabilmemiz için bu scope'u extra güçlendirilmiş `ExecutorService` gibi düşünebiliriz. Örnek bir executorService java kodu aşağıdaki gibidir.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ThreadLesson {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 20; i++) {
            Thread thread = new Thread(new MyThread("thread" + i, 3));
            executor.execute(thread);
        }
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("Done");
    }
}
/***
ExecutorService bize belli bir anda en fazla kaç Thread çalıştırmak 
istediğimizi sormaktadır. Yukarıdaki örnek newFixedThreadPool metodu ile 5 
farklı iş parçasının aynı anda çalıştırılabileceği belirtilmiştir. Daha 
sonrasında for döngüsü içinde 20 adet Thread tanımlanmasına rağmen 
executor servisi gelen işleri düzene sokar ve 5 Thread üzerinde işlem 
gerçekleştirmez. Sonradan eklenen işlemler sıraya (queue) sokulur ve 
mevcut işlemler bitirildikçe çalıştırılır. Böylece sistem kaynakları işlem 
parçaları tarafından kontrolsüzce harcanamaz. shutdown metodu ise yeni 
işlem alımını durdurur ve mevcut işlemlerin bitirilmesini sağlar. 
awaitTermination ise mevcut işlemlerin bitirilmesi için belirli bir süre 
tanır ve bu sürenin sonunda ExecutorService tamamen kapatılır.
*/
```

CoroutineScope ile başlatılmış herhangi bir coroutine'i iptal edebiliriz. Böylelikle sık sık karşılaştığımız; kullanıcı işlem devam ederken ekrandan çıktığında çalışan iş parçacığını iptal etme sorununu kolaylıkla çözmüş oluyoruz. 

* 1.a - Yeni Coroutine başlatılması 
Daha önce bir suspend fonksiyounun sadece suspend bir fonksiyondan ya Coroutine  bloğunun içerisinden başlatılmak zorunda olduğunu söylemiştik. Bunun için aşağıdaki yapıları inceleyeceğiz. 

> launch : launch yapısı bize geriye değer döndürmeyen bir Coroutine bloğu başlatır. Fire-and-forget mantığı vardır.

> async : async yapısı ise bize geriye değer döndürebileceğimiz Coroutine  bloklarını başlatmaya yarar. 

Aşağıda launch ile oluşturulmuş bir örnek kod bulunmakta. async ile ilgili ilerleyen bölümlerde detaylı değinilecektir.

```kotlin
scope.launch {
    // This block starts a new coroutine 
    // "in" the scope.
    // 
    // It can call suspend functions
   getQuestions()
}
```

launch ve async sadece CoroutineScope varlığında başlatıldığını fark etmişsinizdir. Görüleceği üzere scope olmadan bunların  kullanımı mümkün olmaktadır.

launch ve async arasındaki en büyük farklardan biri de hatayı ele alma şekilleridir. async, nihayetinde bir sonuç ya da almak için `await`'i beklemek beklemek durumundadır. async default olarak hata fırlatmaz ve await handle edilmezse hata fark edilemez.

* ViewModel İçerisinde Scope
CoroutineScope'un yürütülen parçacıklarının kontrolünü bize verdiğinden bahsetmiştik. MVVM ile beraber hayatımıza giren viewModel içerisinde de tam kontrole sahip olmamıza yardım eden `viewModelScope`  bulunmaktadır.

`viewModelScope` bulunduğu viewModel'in yaşam döngüsünde hareket eder. OnCleared metoduna kadar çalışmasını sürdüren işler bu callback ile cancel durumuna geçer. Ve Kullanıcı ilgili ekranı terk  ettiğinde bu ekranda arka planda çalışan bu ekranın viewModeli scope'unda işleyen işlerin hepsi durdurulur. 

```kotlin
class MyViewModel : ViewModel() {
  
    /**
     * Heavy operation that cannot be done in the Main Thread
     */
    fun launchDataLoad() {
        viewModelScope.launch {
            sortList()
            // Modify UI
        }
    }
  
    suspend fun sortList() = withContext(Dispatchers.Default) {
        // Heavy work
    }
    
    fun runForever() {
    // start a new coroutine in the ViewModel
    viewModelScope.launch {
        // cancelled when the ViewModel is cleared
        while(true) {
            delay(1_000)
            // do something every second
        }
    }
}
}
```

* 2 - İşlerin Takibi
Bir Coroutine başlatıp bunun takibini yapmanın kolay olduğunu söylemiştik. Ama gerçek dünyada işler bu kadar basit değil. Birden fazla Coroutine'i aynı anda ya da ardışık koşup; birbirinin geri dönüşlerine göre bir sonraki Coroutine'i başlatmak isteyebiliriz. Bu durumda işler biraz kafa karıştırıcı olabilmektedir. Aynı anda birden fazla işi koşmamıza yardımcı ve bünyesindeki coroutine'lerden biri hata aldığında diğerlerini de iptal eden [`coroutineScope`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) karşımıza çıkmaktadır. Arada sadece bir harfin büyük olduğu `CoroutineScope` ile karıştırılmaması gerekir. Ki yukarda CoroutineScope'a değindik. Bu noktada `coroutineScope`'un kuzeni olan [`supervisorScope`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html)'a da değinmek gerekir. Bunların arasındaki en büyük fark `supervisorScope` child coroutine'lerinden biri iptal edilirse ya da fail olursa diğer child coroutine'leri iptal etmez ve koşmalarını sağlar. 

Bütün bunlar potansiyel memory leak'lerin önüne geçmek işleri daha iyi yapılandırmak ve üzerlerindeki kontrolümüzü arttırmak için gereklidir.

Kotin coroutineScope ile herhangi bir sızıntıya sebep olmayacağından emin olmamızı sağlıyor. 

![`coroutineScope`](https://miro.medium.com/max/1400/1*OeFUzbFsbBz034aZB4fbAw.gif)

Yukardaki gif'te coroutineScope altında launch ile çok sayıda network çağrısı yapıldığını görüyoruz. Bu noktada `coroutineScope`'un üzerinde koştuğu scope'a dair bir fikrimiz yok. CoroutineScope da olabilir viewModelScope'da olabilir. Hangi scope olursa olsun `coroutineScope` builder'ı onu yeni oluşturduğu scope'un parenti olarak kullanacaktır. Böylelikle sahip olduğumuz bir ana scope ile hem kendi işlerimiz hem de `coroutineScope`'un child işleri üzerinde tam kontrole sahip oluyoruz.

```kotlin
suspend fun showSomeData() = coroutineScope {
    val data = async(Dispatchers.IO) { // <- extension on current scope
     ... load some UI data for the Main thread ...
    }

    withContext(Dispatchers.Main) {
        doSomeWork()
        val result = data.await()
        display(result)
    }
}
```

*  3 - Coroutine Hatalarının işlenmesi

Coroutine'de hatalar tıpkı bildiğimiz düz fonksiyonlar ile exception fırlatmasıyla olur. Ve normal fonksiyonlarda olduğu gibi coroutine'lerde de  try/catch/finally yapısıyla hatayı işlememiz gerekir. 

Fakat iyi yapılandırılmamış bir blokta hatalarımız uzay boşluğuna doğru savrulabilir :) . Aşağıdaki örnekle inceleyelim:

```kotlin
val unrelatedScope = MainScope()
// example of a lost error
suspend fun lostError() {
    // async without structured concurrency
    unrelatedScope.async {
        throw InAsyncNoOneCanHearYou("except")
    }
}
```

Alakasız bir scope'la ilişkilendirilmiş bu coroutine'de esas sorunumuz `async` ile bereber sonucu ve hatayı gözlememizi  sağlayan `await`'in çağrılmamasıdır. Bir visor(coroutineScope ve superVisorScope) ile ilişkilendirilmediği için await'in çağrılmamasıyla var olan hata hafızada sonsuza kadar varlığını sürdürebilir. await ile bu hatanın handle edilip logic'in ona göre şekillendirilmesi gerekir. 

İyi yapılandırılmamış'tan kastımız sadece await'i çağırmamak gibi basit bir gözden  kaçırma değil.Bunları;

> Scope'unu bulunduğumuz konumla ilişkilendirmek: CoroutineScope veye viewModelScope. 

> visor'ını(gözetmenini) belirlemek: coroutineScope ve superVisorScope. 

> Ve son olarak da kotlin best practice'leri uygulamak. 

Aşağıda bulunan kodu inceleyelim

```kotlin
suspend fun foundError() {
    coroutineScope {
        async { 
            throw StructuredConcurrencyWill("throw")
        }
    }
}
```

Burda gözetmen coroutineScope builder'i child coroutine'lerin çalışmasını gözetleyecek ve hata almaları  durumunda diğerlerini iptal edebilecek.Tamamlandığında uyarılacak. Aynı şekilde bulunduğu scope'u parent olarak kullandığı için parentScope'unu da uyarabilecektir.

```kotlin
class ProductsViewModel(val productsRepository: ProductsRepository): ViewModel() {
   private val _sortedProducts = MutableLiveData<List<ProductListing>>()
   val sortedProducts: LiveData<List<ProductListing>> = _sortedProducts
  
   private val _sortButtonsEnabled = MutableLiveData<Boolean>()
   val sortButtonsEnabled: LiveData<Boolean> = _sortButtonsEnabled
  
   init {
       _sortButtonsEnabled.value = true
   }

   /**
    * Called by the UI when the user clicks the appropriate sort button
    */
   fun onSortAscending() = sortPricesBy(ascending = true)
   fun onSortDescending() = sortPricesBy(ascending = false)

   private fun sortPricesBy(ascending: Boolean) {
       viewModelScope.launch {
           // disable the sort buttons whenever a sort is running
           _sortButtonsEnabled.value = false
           try {
               _sortedProducts.value =
                       productsRepository.loadSortedProducts(ascending)
           } finally {
               // re-enable the sort buttons after the sort is complete
               _sortButtonsEnabled.value = true
           }
       }
   }
}
```

Günlük hayatta karşılaştığımız:

> Şuanki işi iptal etme

> Bir sonraki işi sıraya koyma

> Bir sonraki işle birleştirmek 

[`Faydalı bir Helper Sınıfı`](https://gist.github.com/objcode/7ab4e7b1df8acd88696cb0ccecad16f7#file-concurrencyhelpers-kt-L124)

[`Mutex`](https://paste.ofcode.org/SauWtUMyMpWhQvxUyYCKcM)



Dinlediğiniz için Teşekkürler 




