# 目錄(for c#)
- <a href="#GC">GC運作原理</a>
- <a href="#Iinterface&abstract">Interface & Abstract class</a>
- <a href="#Thread&Task">Thread & Task</a>
- <a href="#Session&Cookie">Session & Cookie</a>
- <a href="#CORS">CORS解決的原理</a>
- <a href="#Enum&Collection">IEnumerable, ICollection, IList And List</a>

## <a name="GC">GC運作原理</a>
### Stack & Heap
在.NET執行程式的時候，CLR針對不同的型別在記體體中會分配不同的空間進行儲存。

對於是`Value Type`而言，在宣告一個變數的時候，CLR會在Stack分配一個空間儲存，而在賦予值的時候也會在同樣的空間儲存。其特性是後進先出(LIFO)，而且一旦function執行完畢，就會被自動回收，不需要擔心memory leak的發生。

對於`Reference Type`而言，CLR會在Stack創造一塊儲存記憶體位址的空間，並且在初始化的時候在Heap上配置該型別所需要的空間，並且將記憶體位置回傳給Stack儲存。

由CLR自動配置與管理的記憶體，被稱為Managed resource；而不受CLR管理被稱為Unmanaged resource。

> *最常見的 Unmanaged 資源類型就是包裝作業系統資源 (例如檔案控制代碼、視窗控制代碼或網路連接) 的物件。*

基本上如果該變數或參考**不再有效**的時候，CLR會判定其為垃圾資源，並等待回收。

對於`Value Type`，CLR會直接回收Stack的空間;`Reference Type`而言，CLR一樣會回後Stack上面儲存的記憶體位置，此時Heap上面的空間就會被標記為垃圾。若重新被初始化，因為記憶體位置不同，原本的也同樣被標記為垃圾。

GC的機制並不是即時的，回收的條件可能有下列的原因:

- 系統的實體記憶體不足。 這是由 OS 的低記憶體通知或主機所指示的記憶體不足通知所偵測到。
- Managed 堆積上設定物件所使用的記憶體超過可接受的臨界值。 這個臨界值會在處理序執行時持續調整。
- 已呼叫 GC.Collect 方法。

GC的演算法有幾個考慮：
- 壓縮部分受控堆積的記憶體比整個受控堆積更快。
- 較新物件的存留期較短，而較舊的物件則存留期較長。
- 較新的物件通常會彼此相關，並由應用程式在相同時間存取。

在Heap上面的物件會被CLR區分三個等級：0、1、2來代表存活的時間長度。

大部分的物件都會在層代0回收，以進行垃圾收集，而不會存活到下一代。

如果今天層代0已經滿了但又必須創建新的物件，則CLR會把層代0還在使用的物件提升至層代1。如果又滿了就會把層代1的物件提升至層代2，最高就是2。

而進行回收的時候，會一並把等級低的一起回收，例如執行層代2回收時，會把1與0的也一起回收。

為了讓使用者可以明確的選擇Unmanaged resource回收的時機點，C#提供了一個Dispose()方法，使用者可以藉由繼承IDispose來使用。

當物件呼叫Dispose()的時候，有可能還是會被GC的自動回收機制Finalize()，因為這是避免Dispose()失敗後資源還是在那邊。

但是如果確定Dispose()成功執行，那會建議在後面再加一個`GC.SuppressFinalize(this);`，告訴GC不需要再去呼叫這個物件的Finalize()

## <a name="Iinterface&abstract">Interface & Abstract class</a>

面試常常問的問題： Interface與Abstract Class有什麼區別？

> 除非需要為子類別提供公共功能，否則優先使用介面。

Interface的中心思想是「封裝隔離」，意思就是不需要知道裡面的實作方法，看到的只會是我提供了哪些方法或屬性。而且繼承了這個Interface之後，必須要實作裡面所有的方法以及屬性。在組合不同類別的時候可以使用Interface。

抽象類別是在整個繼承體系的最上層，裡面可以再宣告抽象的方法或是普通的方法，如果是抽象的方法就一定要override它。在繼承相關的類別的時候可以使用抽象類別。

除非要用到的類別他們關係真的很緊密，例如交通工具、汽車之間的關係，他們之間有一些共用的方法，不然使用Interface是較為恰當的。

## <a name="Thread&Task">Thread & Task</a>

Thread 類和 Task 類都用於 C# 中的並行程式設計。

Thread在C#中建立實際的作業系統級別的執行緒。用Thread建立的執行緒會佔用堆疊記憶體等資源。任何用完的執行緒都會先回到執行緒池備用，不會立刻銷毀。執行緒池所有執行緒，都是屬於背景執行緒。

執行緒池的使用還是有些許缺點，例如並不知道操作什麼時候會結束，無法有回傳值。所以才會有Task出現。Task則是建立一個非同步的任務，可以知道什麼時間結束以及回傳值，一般來說會在thread pool執行，不需要任何額外的記憶體或CPU資源，也不能指定執行緒的優先順序。

如果Task執行的時間很長，為了避免佔用
thread pool裡面的資源，有提供一個LongRunning的option，此時將會提供一個新的執行緒來執行，而不會去thread pool取。

對於任何長時間執行的操作，應優先選擇執行緒，而對於任何其他非同步操作，應優先選擇任務。

## <a name="Session&Cookie">Session & Cookie</a>

Session與Cookie最大的不同在於，Session將資料以object的形式暫時性的儲存在sever裡面，用於跨頁面瀏覽時讀取內容。

步驟：

1. Client端發送Request給Server，Server產生一個SessionId。
2. 將SessionId、UserName、ExpireDate...等等資訊儲存在資料庫。
3. 以Cookie形式將SessionId送回Client儲存。
4. 每當有新的Request進來的時候就可以檢查這個SessionId跟User是否符合，以及是否還有效。

Cookie是一個小型的文檔儲存在Client端，最大的大小為4kb，主要的工作是記錄一些瀏覽的歷史資料，例如購物車。Cookie只能以string的形式儲存，正因為以上的特性，Cookie並不能算是一個安全的儲存形式，任何人都可以取得Cookie的資料。

## <a name="CORS">CORS解決的原理</a>
在同源政策下，非同源的request則會因為安全性的考量受到限制。所謂的同源，指的是相同的通訊協定、相同的網域、相同的通訊埠。

在 CORS 的規範裡面，跨來源請求有分兩種：「簡單」的請求和「非簡單」的請求。

- 簡單請求：只能是 HTTP GET, POST or HEAD 方法、自訂的 request header 只能是特定的幾種：Accept、Accept-Language、Content-Language 或 Content-Type（值只能是 application/x-www-form-urlencoded、multipart/form-data 或 text/plain）
- 一般跨網域請求：瀏覽器在發送請求之前會先發送一個 「preflight request（預檢請求）」，其作用在於先問伺服器：你是否允許這樣的請求？真的允許的話，我才會把請求完整地送過去。
- 
收到preflight request後：

Server必須告訴瀏覽器允許的方法和header有哪些。因此Server的回應必須帶有以下兩個 header:

>Access-Control-Allow-Methods: 允許的 HTTP 方法。

>Access-Control-Allow-Headers: 允許的非「簡單」header。

瀏覽器收到正確的preflight response，表示CORS的驗證通過，就可以送出跨來源請求了。

最後一步，server 還是要回應<code>Access-Control-Allow-Origin header</code>。瀏覽器會再檢查一次跨來源請求的回應是否帶有正確的<code>Access-Control-Allow-Origin header</code>

## <a name="Enum&Collection">IEnumerable, ICollection, IList And List</a>

### IEnumerable & IEnumerable<T>
在C#的library中定義了兩個IEnumerable的Interface:

IEnumerable: Namespace為System.Collections，而且只有一個實作的方法。
```
public interface IEnumerable
{
  IEnumerator GetEnumerator();
}
```
GetEnumerator方法必須回傳一個已經實作IEnumerator的class。

C#的foreach對於任何具有實作IEnumerator的class都有作用。

IEnumerable<T>: 除了namespace為System.Collections.Generic以外，其餘的與IEnumerable幾乎一樣，只是它是generic並為type-safe。
    
### ICollection & ICollection<T>

兩者namespace分別為<code>System.Collections.ICollection</code>
以及<code>System.Collections.Generic.ICollection<T></code>

並繼承了IEnumerable和IEnumerabl<T>

```
public interface ICollection : IEnumerable
{
  int Count { get; }  
  bool IsSynchronized { get; }
  Object SyncRoot { get; }
  void CopyTo(Array array, int index);
}
```
```
public interface ICollection<T> : IEnumerable<T>, IEnumerable
{
  int Count { get; }
  bool IsReadOnly { get; }
  void Add(T item);
  void Clear();
  bool Contains(T item);
  void CopyTo(T[] array, int arrayIndex);
  bool Remove(T item);
}
```
泛型的集合看上去與非泛型的不太一樣，多了一些實作的的方法，這是因為NET2.0之後才加入了泛型，而非泛型的1.1就出現了。
    
### IList & IList<T>
    
IList 繼承了ICollection與IEnumerable

```
public interface IList : ICollection, IEnumerable
{
  bool IsFixedSize { get; }
  bool IsReadOnly { get; }
  Object this[int index] { get; set; }
  int Add(Object value);
  void Clear();
  bool Contains(Object value);
  int IndexOf(Object value);
  void Insert(int index, Object value);
  void Remove(Object value);
  void RemoveAt(int index);
}
```
    
IList<T>則是繼承了ICollection<T>, IEnumerable<T>以及IEnumerable
    
```
public interface IList<T> : ICollection<T>, IEnumerable<T>, IEnumerable
{
  T this[int index] { get; set; }
  int IndexOf(T item);
  void Insert(int index, T item);
  void RemoveAt(int index);
}
```

### 使用的時機


|類型|使用時幾|
|---|---|
|IEnumerable|只需要迭代這個集合而且只是要讀取這個集合中的元素(因為他沒有Add或Remove這種異動的功能)|
|ICollection|除了可以迭代集合以外你還在乎他的大小|
|IList|除了可以迭代集合、控制大小以外，你還在乎他的排序以及位置|
|List|根據依賴反轉的原則，不建議直接使用List進行實作，除非必須用到List實作時的額外功能(ex: Sort()、Find()...)|
    
基本上如果使用的集合需要實作的越少(例如IEnumerable)，使用者在傳入參數的時候可以有比較大的彈性(因為即使是自定義的Collection，幾乎都會去實作IEnumeralbe);反之如果傳入的參數限定在IList，如果呼叫者的實作只有IEnumerable，就會出錯。
    
在使用LinQ的時候返回的都是IEnumerable，這時候只會把查詢的條件保留起來，並不會將結果存入記憶體中，但是一但ToList()的時候會迫使直接實例化，將結果存入記憶體。
    
所以一般來說，在查詢的結果到最後時，才去使用ToList()，避免資源的浪費。在這之前建議使用IEnumerable做為參數傳輸。
    
### 有關IEnumerable與IQueryable<T>

兩者皆為延遲啟動，IQueryable是LinQ-to-SQL，IEnumerable是LinQ-to-object。
以<a href="https://stackoverflow.com/questions/2876616/returning-ienumerablet-vs-iqueryablet">Stackoverflow</a>上的解釋：
```
IQueryable<Customer> custs = ...;
// Later on...
var goldCustomers = custs.Where(c => c.IsGold);
    
IEnumerable<Customer> custs = ...;
// Later on...
var goldCustomers = custs.Where(c => c.IsGold);
```
IQueryable的程式會驅動SQL的查詢並且只查出IsGold的結果。
    
IEnumerable的程式則是會先把SQL查詢的資料拿出來，然後再記憶體中剔除不是IsGold的資料。
    
看似IQueryable是一個很好的選擇，但是在repository layer或service layer盡量避免當作參數傳遞(或通過)，可以保護資料庫避免因為堆疊LinQ表達式而產生較高的資源花費。
    
不過大部分都還是使用IEnumerable的原因是，當資料不大而且經常需要篩選的時後，其實記憶體做這件事情速度會比較快。
    
其次是並不是所有的LinQ提供者(LINQ2SQL, EF, NHibernate, MongoDB etc.)都會支援LinQ所有的操作，反而是至少如果是Collection大家都應該都會去實作IEnumerable，所以會比較安全。