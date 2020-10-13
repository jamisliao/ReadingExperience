## Redis資料存好存滿系列 Part 2 - Lists的使用

Lists是由字串組成的一個集合，會根據新增或插入的順序將集合中的元素排序，並且在操作上允許使用者將新增的元素放置在集合的頭或尾，也允許插入在任兩個元素中間。以效能面來說，Lists中就算已存在上百萬甚至上千萬筆資料，在頭尾新增資料還是相當有效率的，在官方的文件上也載明了這兩種操作的時間複雜度是**O(1)**。但是，在任兩個元素中插入新元素的效能就來得差了許多，時間複雜度為**O(N)**。所以在使用上，應該朝著以頭尾新增元素為主。附帶一提，Lists無法像Strings一樣設定資料存活時間，只能自行手動刪除。

### Redis原始命令

Redis有關Lists的命令並不少，可以在[這裡](https://redis.io/commands#list)看到官方有關Lists的命令說明。命令雖然不少，不過大致可以分成幾類

1. 新增元素 & 取得元素
	
	**新增元素**

	[`LPUSH`](https://redis.io/commands/lpush)與[`RPUSH`](https://redis.io/commands/rpush)分別是將元素新增到集合的頭尾，並且命令支援一次新增多個元素，先來看看新增一個元素時的指令

		127.0.0.1:6379>LPUSH Number 9527
		(integer) 1
		127.0.0.1:6379>LPUSH Number 2185
		(integer) 2
		127.0.0.1:6379>LRANGE Number 0 -1		
		1)"9527"
		2)"2185"	

	一次新增多個元素

		127.0.0.1:6379>LPUSH Number 9527 2185
		(integer) 2
		127.0.0.1:6379>LRANGE Number 0 -1		
		1)"2185"
		2)"9527"	
	Lists的陣列起算值與程式陣列一樣，都是由**0**開始起算。此外，Lists的陣列位置值支援負值，當輸入正整數時，是由陣列的左邊開始起算，而輸入負整數時，則是由陣列的右邊開始起算。

	**取得元素**
	
	取得Lists的元素有分為，**取得元素後會被移除**([`LPOP`](https://redis.io/commands/lpop), [`RPOP`](https://redis.io/commands/rpop))與**取得元素後依然存在**([`LRANGE`](https://redis.io/commands/lrange), [`LINDEX`](https://redis.io/commands/lindex))兩類，在命令上也有滿大的差別，取得後元素會被移除的命令有`LPOP`與`RPOP`，兩者的差別就只是從集合的頭或是從集合的尾巴取得，先來看看`LPOP`

		127.0.0.1:6379>LPUSH Number 9527 2185
		(integer) 2	
		127.0.0.1:6379>LPOP Number
		"2185"
		127.0.0.1:6379>LRANGE Number 0 -1
		"9527"

	可以從上面的執行結果發現，再經由`LPOP`取得元素"2185"後，該元素就在集合中被移除了。來看看另一種`LRANGE`與`LINDEX`的效果

		127.0.0.1:6379>LPUSH Number 9527 2185
		(integer) 2
		127.0.0.1:6379>LRANGE Number 0 -1		
		1)"2185"
		2)"9527"	
		127.0.0.1:6379>LRANGE Number 0 -1		
		1)"2185"
		2)"9527"

		127.0.0.1:6379>LINDEX Number 0		
		"2185"
		127.0.0.1:6379>LINDEX Number 1
		"9527"
	
	這邊連拿兩次`LRANGE`的原因是要展示`LRANGE`並不會刪除Lists中的元素。

2. 取得元素數量

	取得元素數量可以使用[`LLEN`](https://redis.io/commands/llen)

		127.0.0.1:6379>LPUSH Number 9527 2185
		(integer) 2
		127.0.0.1:6379>LLEN Number
		(integer) 2

3. 刪除元素

	刪除元素也可以分為兩類，**刪除指定範圍**([`LTRIM`](http://redisdoc.com/list/ltrim.html))與**刪除指定元素**([`LREM`](http://redisdoc.com/list/lrem.html))，先示範`LTRIM`

		127.0.0.1:6379>LPUSH Number 9527 2185 1234 1080
		(integer) 4
		127.0.0.1:6379>LRANGE Number 0 -1
		1)"1080"
		2)"1234"
		3)"2185"
		4)"9527"
		127.0.0.1:6379>LTRIM Number 1 2
		"OK"
		127.0.0.1:6379>LRANGE Number 0 -1
		1)"1234"
		2)"2185"
	原本Lists中有四個元素，分別是1080, 1234, 2185, 9527，使用`LTRIM`將頭尾的元素刪除後，就只剩下1234, 2185了。
	
	接著來說明`LREM`，在說明這一個命令的功能前，先來看看`LREM`的命令格式

	LERM Key Count Value

	`LREM`的主要功能是刪除與value相同的元素值，刪除的數量是Count，並且當Count是正整數時，由陣列的左邊開始刪除，當Count是負整數時，由陣列的右邊開始刪除。將上述的規則整理後，如下

	- Count > 0 由陣列的左邊開始掃描，刪除元素值與Value相等的元素，刪除的數量為Count
	- Count < 0 由陣列的右邊開始掃描，刪除元素值與Value相等的元素，刪除的數量為Count的絕對值
	- Count = 0 刪除陣列中所有元素值與Value相等的元素
	
	以下是`LREM`的操作

		127.0.0.1:6379>LPUSH Number 9527 2185 9527 2185 9527
		(integer) 5
		127.0.0.1:6379>LREM Number 1 9527
		(integer) 1
		127.0.0.1:6379>LRANGE Number 0 -1
		1)"2185"
		2)"9527"
		3)"2185"
		4)"9527"
		127.0.0.1:6379>LREM Number -2 9527
		(integer) 2
		127.0.0.1:6379>LRANGE Number 0 -1
		1)"2185"
		2)"2185"
		127.0.0.1:6379>LREM Number 0 2185
		(integer) 2
		127.0.0.1:6379>LRANGE Number 0 -1
		(empty list or set)

4. 指定位置的元素值

	Lists的操作除了新增與刪除元素外，也提供了更新的命令。可以透過[`LSET`](https://redis.io/commands/lset)更新指令位置的元素值，以下是`LSET`的操作

		127.0.0.1:6379>LPUSH Number 9527 2185 1234 1080
		(integer) 4
		127.0.0.1:6379>LSET Number 2 4321
		OK
		127.0.0.1:6379>LRANGE Number 0 -1
		1)"1080"
		2)"1234"
		3)"4321"
		4)"9527"
	可以看到在位置2的元素值由原本的2185更新成4321。

### 場景

一般來說，使用StackExchange.Redis透過`LPUSH`或`RPUSHS`存放Lists的資料時，無法同時設定TTL(Time To Live)，但是可以透過`EXPIRE`來設定Lists的TTL，雖然多了一個步驟，但是還是可以透過時間來控制資料的失效時間，所以在場景的適用上，並不會有所特別。

Lists的資料在存放的過程，大部分是一種順序式的存放，也就是說資料在存放的時候，就已經有了排序。所以，這樣的方式建議可以用來存放**連續性**或是**順序性**的資料。例如，每日溫度的歷史資料、降雨量的歷史資料、Email的收件夾。由於Lists在操作上，可以指定範圍的讀取，所以前述所提到的資料就很容易做到分群讀取。像是一次拿取一個月的歷史資料或是取得最近十封的Email。

另外，Lists的`LPOP`與`RPOP`取得資料後，會將資料從Lists中刪除，這樣的特性也適合用在**未讀訊息**或是**Email的未讀郵件**，這一類的需求都是當使用者讀取訊息，訊息就會從未讀的歸類中移置已讀的歸類中，也特別適合以Lists設定。

### .Net實作

在這Lists的實作上，這次選擇了歷史資料的呈現上實作。使用Lists來存放一整年的PM2.5的量測資料，但是由於資料內包含的量測點很多，所以只挑了一個量測點來呈現。

![Imgur](http://i.imgur.com/7n6uSiD.jpg)

透過Table來呈現每個月永和量測站的PM2.5數值，下方的分頁可以切換月份。接著將針對與Redis有關的程式來說明

在透過Repository取得PM2.5的歷史資料時，會先判斷Redis中是否已經有存放著資料，再決定是由CSV檔讀取資料或是向Redis取得資料。

	public List<Station> GetDataByMonth(int month)
    {
        var result = default(List<Station>);
        if(RedisListExist(2015))
        {
            result = ReadHistoryDataFromRedis(2015, month);
        }
        else
        {
            var totalData = ReadHistoryDataFromFile(@"./wwwroot/data/2015-HistoryData.csv");
            PutToRedis(totalData, 2015);
            CreateMonthIndex(totalData, 2015);

            var date = new DateTime(2015, month, 1);
            date = date.AddMonths(1);
            result = totalData.Where(p => p.Date < date).ToList();
        }

        return result;
    }

如果是由CSV檔讀取資料時，則會在資料讀取完成後，將資料存放到Redis中，並且這邊還會將一年來的資料按照月份製作Index。判斷Key是否已經存在Redis，可以透過StackExchange.Redis所提供的Method `KeyExists`來判斷。

	private bool RedisListExist(int year)
    {
        var key = $"{year}-pm2point5history";
        var isExist = this._db.KeyExists(key);

        return isExist;
    }

要將資料以Lists的形式存放到Redis中，可以透過`ListRightPush`將資料存放到Redis。
	
	private void PutToRedis(List<Station> entities, int year)
    {
        if (RedisListExist(year))
        {
            return;
        }

        foreach (var entity in entities)
        {
            var jsonStr = JsonConvert.SerializeObject(entity);
            this._db.ListRightPush($"{year}-pm2point5history", jsonStr);
        }
    }

這邊為了之後方便取得每個月的資料範圍，所以先將一整年的資料按月分群，並且製作Index以Strings的形式存放在Redis，方便之後要取得每個月的資料

	private void CreateMonthIndex(List<Station> entities, int year)
    {
        var startDate = new DateTime(year, 1, 1);
        var startIndex = 0;
        var endIndex = 0;
        for (int i = 1; i <= 12; i++)
        {
            startDate = startDate.AddMonths(1);
            endIndex = entities.Count(p => p.Date < startDate);

            var key = $"{year}-{i}-pm2point5history";
            var value = $"{startIndex}-{endIndex}";
            this._db.StringSet(key, value, new TimeSpan(3650, 0, 0, 0));
            startIndex = endIndex;
        }
    }

但是，如果資料已經存在Redis中時，可以透過`ListRange`來取得指定範圍的資料

	private List<Station> ReadHistoryDataFromRedis(int year, int month)
    {
        var result = new List<Station>();

        var key = $"{year}-{month}-pm2point5history";
        var listRange = this._db.StringGet(key).ToString();
        var tmp = listRange.Split('-');
        var startIndex = Convert.ToInt64(tmp[0]);
        var endIndex = Convert.ToInt64(tmp[1]);

        var listKey = $"{year}-pm2point5history";
        var value = this._db.ListRange(listKey, startIndex, endIndex-1);
        foreach (var item in value)
        {
            var entity = JsonConvert.DeserializeObject<Station>(item);
            result.Add(entity);
        }

        return result;
    }

> StackExchange.Redis的`ListLeftPush`與`ListRightPush`皆沒有提供設定TTL的參數，如果需要替資料設定TTL，可以透過`KeyExpire`來設定

### 小結

在這一篇文章中，介紹了Redis中的Lists型別，這比起一般使用者最常用的Strings型別來說，能夠更有效存放有集合特性與順序性的資料。也讓大家在Redis的資料存放的設計上，能夠有不一樣的想法，來更有效地利用不同的Redis資料型別來增進資料存取的效能。

隨文附上範例程式 : [Github](https://github.com/jamisliao/RedisSample)，這邊稍微說明一下，範例程式是由Asp.net Core所撰寫，請執行前先安裝好.Net Core的環境(可以到此下載[安裝檔.Net Core](https://www.microsoft.com/net/core#windowsvs2015))。

### 參考資料

- [redis.io](https://redis.io/)
- [StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis)