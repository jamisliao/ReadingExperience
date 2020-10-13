## Redis資料存好存滿系列 - 序章

現在的系統設計中，越來越多人會將Cache加入設計，尤其是在分散式系統架構下，獨立於Application外的Cache更是不可缺少的一環。在Cache的選擇上，許多人會使用NoSQL來當資料的儲存體，而Redis(REmote DIctionary Server)又是眾多NoSQL數據框架的首選。

Redis是一個Key-Value的儲存系統，將資料存放在記憶體中，並且有下列
	
- 性能高
- 豐富的資料型別

Redis支援的型別有

- Strings
- Lists
- Sets
- Sorted sets
- Hashes
- Bitmaps
- HyperLogLogs
- Geospatial

在這一系列中，主要針對Redis的資料型別做介紹，會透過小品文章說明與實作來介紹各種資料結構，以及適用的場景，希望可以讓大家不再只是將物件序列化後，以字串的方式儲存在 Redis中，實作將以C#搭配StackExchange.Redis來完成Demo，但因為StackExchange.Redis尚未支援Geospatial，所以將不會在這系列中介紹，StackExchange.Redis的Contributor已在[Github](https://github.com/StackExchange/StackExchange.Redis/issues/496)上回覆應該在不久之後會支援Geospatial。另外，Bitmaps與HyperLogLogs將暫不在本系列中介紹。

Redis除了性能高和豐富的資料型別外，Redis也支援橫向擴展(Cluster)與HA，不過這兩點並不會在這系列中討論。對於Redis安裝、橫向擴展(Cluster)與HA感興趣的朋友，可以在[軟體主廚的點部落](https://dotblogs.com.tw/supershowwei/series/1?qq=Redis)中找到相關文章。

### 資料型別

在進到各種資料型的小品介紹文之前，先在這讓大家對於每一種型別有初步的認識，以下將對各個型別簡單的說明

##### Strings

Strings是Redis資料型別中最基本的一個，可以存放任何用String表示的資料，可以是一段文字，也可以是物件序列化後的資料，甚至可以是二進位的0101資料。

##### Lists

Lists是一種有順序的字串集合，轉換成程式的資料結構大概是個Double linked list的概念。常用的操作將資料放到列表的頭尾或是取頭尾資料，有時候也會取一個範圍的資料。

##### Sets

Sets簡單來說就是字面上的意思，集合。在Sets中，資料是無序的並且必須是唯一，這是與Lists最大的不同點。

##### Sorted sets

Sorted sets就是有順序的Sets，有人可能會問那個Lists不就一樣了，Sorted sets是透過每一個儲存的元素附帶有一個分數，並且透過該分數來做排序，與Lists是用Double linked list的概念又顯著的不同。

##### Hashes

Hashes與第一種Strings的概念相似，但是在Value的部分是由Field與Value組成，非常適合存放物件，概念大致如下圖
![](https://az787680.vo.msecnd.net/user/jamis/bed6b6b4-9354-4483-ac33-932eb82499c9/1482397862_30636.jpg)

### Redis原始命令

雖然在實作上是使用StackExchange.Redis，但是對於原生的Redis command還是了解一下會有所幫助，並且在StackExchange.Redis的指令對照上也會比較有概念，所以在後續的介紹中，也會一併簡單說明Redis command是如何使用。

### StackExchange.Redis

在後續文章的實作中，將會使用StackExchange.Redis，由相當有名的Stack Exchange所以主導，一套在.NET上存取Redis的套件，如果你沒聽過Stack Exchange，那你應該會聽過工程師該擁有的兩本書其中一本Stack Overflow，是同一個老爸。

StackExchange.Redis是一套Open Source的套件，開源在[Github](https://github.com/StackExchange/StackExchange.Redis)上。使用Visual Studio開發的讀者也可以很容易透過[Nuget](https://www.nuget.org/packages/StackExchange.Redis/)加入到專案之中。

### 小結

在序章中，稍微介紹了Redis和可以使用的資料型別，在接下來的文章中，將會針對每一種資料型別說明，並且從合適的適用場景中，實作一種場景的需求，使用C#搭配StackExchange.Redis，讓大家能夠更了解如何使用Redis來當中繼資料的儲存體。
