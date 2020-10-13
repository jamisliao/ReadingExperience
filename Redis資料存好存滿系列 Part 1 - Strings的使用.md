## Redis資料存好存滿系列 Part 1 - Strings的使用

Redis的眾多資料型別中，最基本也是最多人使用的就是Strings。雖然很多的場景可以使用，也很多人用，但不代表每一個可以使用的場景都是恰當的。簡單舉個例子來說，當存放一個物件時，許多使用者會將物件序列化成字串後存進Redis。但是，在後續的使用上就必須一口氣將整個字串拿出來反序列化後使用。如此一來，無法針對場景的需求，只拿取需要的物件屬性，這在考量系統與程式的合理性上都會來的薄弱一些。

> 注意:Value的最大值為512MB

### Redis原始命令

`Set`和`Get`可以說是Redis命令中最基本的兩個，分別是將指定的Key-Value放進到Redis與從Redis中取出指定的Key-Value，使用的格式為

- Set Key Value

- Get Key

而實際在Redis Cli上執行的結果會像這樣

	127.0.0.1:6379>Set 9527 Jamis
	OK

	127.0.0.1:6379>Get 9527
	"Jamis"

當Get的Key不存在時，則會回傳空值。這邊在介紹一個比較特別的指令是`INCR`。主要的功用是如果當儲存的Value為整數時，則會回傳**整數+1**的值，來看一下實際的例子

	127.0.0.1:6379>Set number 1
	OK

	127.0.0.1:6379>Get number
	"1"

	127.0.0.1:6379>incr number
	(integer) 2

當使用的Key不存在時，預設值為0，所以當遞增後則會回傳1。

> 有關Strings的命令還有很多，有興趣的讀者可以到[Redis](https://redis.io/commands#string)的官方網站查詢

### 場景

Strings這樣的型別可以說什麼場景都適用，但是可能每個場景也都有比Strings更合適的資料型別來取代Strings。所以，這邊沒有特別推薦的場景，當你想要簡單、快速地先完成一個Prototype或是POC(Proof Of Concept)時，可以直接先用Strings，後續進入開發週期後，再調整為適合的型別。但是，另一個`INCR`的指令就滿多適用的場景，尤其拿來取代資料庫資料的自動產生流水號，或是商品庫存就會很好用。


### .Net實作

這裡將示範幾種Strings的用法，在範例程式中將不會特別介紹**StackExchange.Redis**的安裝與基本設定的寫法，只會關注在與Strings有關的相對應方法。

>有興趣想知道StackExchange.Redis安裝與基本使用可以在[軟體主廚的文章](https://dotblogs.com.tw/supershowwei/2015/12/23/171448)中找到相關資訊

1. **Set**

		IDatabase db = conn.GetDatabase();
	        
    	var ts = new TimeSpan(0, 0, 10);
        db.StringSet("9527", "Jamis", ts);
        var data = db.StringGet("9527");
很簡單與直觀的用法，除了`Key`與`Value`兩個參數外，還可以再傳進一個`Expire`作為這一筆資料的生命週期。StringSet Method除了上述三個參數外，另外還支援`When`, `CommandFlags`兩個列舉參數。

	![](https://az787680.vo.msecnd.net/user/jamis/0d452c61-833f-45d9-bdeb-21741f754186/1482484306_09732.jpg)

	在`When`的部分，是在決定執行的條件，總共有三種情境

	1. Always - 永遠執行
	2. Exists - 當Key存在時執行
	3. NotExists - 當Key不存在時執行

	這個部分在Redis [Set](https://redis.io/commands/set)命令中，就包含了這個參數。而`CommandFlags`就相對來的複雜一些，這是用來決定Command在Redis上執行的行為，可以選擇的選項有許多，部分選項又牽扯到Redis的Cluster，這邊就不多做說明，一般使用上選擇使用預設值即可，有興趣的讀者可以到[Github](https://github.com/StackExchange/StackExchange.Redis/blob/master/StackExchange.Redis/StackExchange/Redis/CommandFlags.cs)看作者寫在程式碼上的說明。

	在常見的實務需求其中之一，新增資料的時候除了將資料存進資料庫外，還可以順便將資料放進快取中，之後如果有需要這筆資料時，就可以先看看資料是否還在快取中。
	
		public void AddProduct(ProductEntity entity)
        {
            entity.Id = Guid.NewGuid();
            this._productRepo.AddProduct(entity);

            var productStr = JsonConvert.SerializeObject(entity);
            var ts = new TimeSpan(0, 8, 0);
            this._db.StringSet($"Product_{entity.Id}", productStr, ts);
			....
			/* 後續省略 */
        }

2. **Get**

		ConnectionMultiplexer conn = ConnectionMultiplexer.Connect("127.0.0.1:6379");
    	IDatabase db = conn.GetDatabase();
	        
    	var data = db.StringGet("9527");

	在`Get`的使用上就來得更簡單了，只要透過IDatabase物件的`StringGet`傳入Key值，就能取得在Redis中相對應的Value。

	實務上，普遍就是在從資料庫拿資料前先看看資料是否有存在快取中，如果有就直接從快取中取出回傳，增進系統效能。

		public List<Product> GetProducts()
        {
            var products = default(List<Product>);
            var productsStr = this._db.StringGet("Products");
            if(productsStr.IsNull)
            {
                products = this._productRepo.GetProducts();
            }
            else{
                products = JsonConvert.DeserializeObject<List<Product>>(productsStr);
            }
        
            return products;
        }

3. **INCR** & **DECR**

	    IDatabase db = conn.GetDatabase();
	    
	    db.StringSet("Temp", "1");
	    db.StringIncrement("Temp");
	    var number = db.StringGet("Temp");
	    Console.WriteLine(number);
	
	    db.StringDecrement("Temp");
	    number = db.StringGet("Temp");
	    Console.WriteLine(number);
	這兩個指令在StackExchange.Redis上的Method對應也是相對的直覺，對應到的Method是`StringIncrement`與`StringDecrement`，分別是加一與減一。因為是Strings型別的指令，所以對應到的Method名稱中會帶有**String**。

	值得一提的是，**StackExchange.Redis**在提供`StringIncrement`這個Method時，不只是能夠達到`INCR`加一的功能，還允許使用者傳入想要加的數值，說白話一點就是允許使用者自訂**加數**的部分，但是，在預設值的部分預設就是加一，所以當不傳入加數的時候，加數的值就是一。

	![](http://i.imgur.com/U7fhNyG.jpg)

	在實務上的應用大部分就是拿來當計數器，或者可以拿來當**庫存數量**。當庫存數量可以不依賴在關聯式資料庫上，透過`Set`與`DECR`來設定與改變庫存數量，在系統效能上是能夠有一定程度的提升，但是後續需要透過其他方式將資料同步回關聯式資料庫。另外，如果某個需求需要去計算資料庫中某個Table的資料筆數時，也建議可以在Redis建立一個該Table資料的計數器，在新增資料時，也透過使用`INCR`來更新計數器上的數值，當需要資料筆數的總和時，直接使用該計數器的數值即可。
	

### 小結

在這個部分中，介紹了Strings型別的Redis命令、StackExchange.Redis對應的方法，試用的場景以及比較特別但好用的指令。Strings型別在Redis的應用中，算是最基本及簡單的，透過Redis命令與對應的Library method更了解到如何使用Strings型別。