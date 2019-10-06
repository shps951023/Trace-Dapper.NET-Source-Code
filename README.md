以下是繁體中文版本
簡體中文版本麻煩需要到博客園觀看 : [深入Dapper.NET源码 (文长) - 暐翰 - 博客园](https://www.cnblogs.com/ITWeiHan/p/11614704.html)

---

# 深入Dapper.NET源碼

## 目錄
1. [前言、目錄、安裝環境  ](#page1)
2. [Dynamic Query 原理 Part1 ](#page2)
3. [Dynamic Query 原理 Part2 ](#page3)
4. [ Strongly Typed Mapping 原理 Part1 : ADO.NET對比Dapper ](#page4)
5. [ Strongly Typed Mapping 原理 Part2 : Reflection版本  ](#page5)
6. [Strongly Typed Mapping 原理 Part3 : 動態建立方法重要概念「結果反推程式碼」優化效率 ](#page6)
7. [Strongly Typed Mapping 原理 Part4 : Expression版本 ](#page7)
8. [ Strongly Typed Mapping 原理 Part5 : Emit IL反建立C#代碼 ](#page8)
9. [Strongly Typed Mapping 原理 Part6 : Emit版本 ](#page9)
10. [Dapper 效率快關鍵之一 : Cache 緩存原理  ](#page10)
11. [錯誤SQL字串拼接方式,會導致效率慢、內存洩漏](#page11)
12. [Dapper SQL正確字串拼接方式 : Literal Replacement  ](#page12)
13. [Query Multi Mapping 使用方式 ](#page13)
14. [Query Multi Mapping 底層原理 ](#page14)
15. [QueryMultiple 底層原理 ](#page15)
16. [TypeHandler 自訂Mapping邏輯使用、底層邏輯 ](#page16)
17. [ CommandBehavior的細節處理 ](#page17)
18. [Parameter 參數化底層原理 ](#page18)
19. [ IN 多集合參數化底層原理 ](#page19)
20. [DynamicParameter 底層原理、自訂實作 ](#page20)
21. [ 單次、多次 Execute 底層原理 ](#page21)
22. [ ExecuteScalar應用](#page22)
23. [總結](#page23)

---


 　 
 　 
## 1.前言、目錄、安裝環境   <a name="page1"></a>


經過業界前輩、StackOverflow多年推廣,「Dapper搭配Entity Framework」成為一種功能強大的組合,它滿足`「安全、方便、高效、好維護」`需求。

但目前中文網路文章,雖然有很多關於Dapper的文章但都停留在如何使用,沒人系統性解說底層原理。所以有了此篇「深入Dapper源碼」想帶大家進入Dapper底層,了解Dapper的精美細節設計、高效原理,並`學起來`實際應用在工作當中。

---

#### 建立Dapper Debug環境

1. 到[Dapper Github 首頁](https://github.com/StackExchange/Dapper) Clone最新版本到自己本機端
2. 建立.NET Core Console專案
![20191003173131.png](https://i.loli.net/2019/10/03/z3ohdWlMpUscwbG.png)
3. 需要安裝NuGet SqlClient套件、添加Dapper Project Reference
![20191003173438.png](https://i.loli.net/2019/10/03/X6xYvkwandAMiRC.png)
4. 下中斷點運行就可以Runtime查看邏輯
![20191003215021.png](https://i.loli.net/2019/10/03/1yaCfgGsUmY7nX5.png)

#### 個人環境 
- 數據庫 : MSSQLLocalDB
- Visaul Studio版本 : 2019
- LINQ Pad 5 版本
- Dapper版本 : V2.0.30
- 反編譯 : ILSpy


 　 
 　 
## 2.Dynamic Query 原理 Part1  <a name="page2"></a>



在前期開發階段因為表格結構還在`調整階段`,或是不值得額外宣告類別輕量需求,使用Dapper dynamic Query可以節省下來回修改class屬性的時間。當表格穩定下來後使用POCO生成器快速生成Class轉成`強型別`維護。

#### 為何Dapper可以如此方便,支援dynamic?

追溯`Query`方法源碼可以發現兩個重點
1. 實體類別其實是`DapperRow`再隱性轉型為dynamic。
![20191003180501.png](https://i.loli.net/2019/10/03/wmCaByiPftj7V6W.png)
2. DapperRow繼承`IDynamicMetaObjectProvider`並且實作對應方法。

![20191003044133.png](https://i.loli.net/2019/10/03/Bt71kJc3dnOforN.png)

此段邏輯我這邊做一個簡化版本的Dapper dynamic Query讓讀者了解轉換邏輯 : 

1. 建立`dynamic`類別變量,實體類別是`ExpandoObject`
2. 因為有繼承關係可以轉型為`IDictionary<string, object>`
3. 使用DataReader使用GetName取得欄位名稱,藉由欄位index取得值,並將兩者分別添加進Dictionary當作key跟value。
4. 因為ExpandoObject有實作IDynamicMetaObjectProvider介面可以轉換成dynamic

```C#
public static class DemoExtension
{
  public static IEnumerable<dynamic> Query(this IDbConnection cnn, string sql)
  {
    using (var command = cnn.CreateCommand())
    {
      command.CommandText = sql;
      using (var reader = command.ExecuteReader())
      {
        while (reader.Read())
        {
          yield return reader.CastToDynamic();
        }
      }
    }
  }
  
  private static dynamic CastToDynamic(this IDataReader reader)
  {
    dynamic e = new ExpandoObject();
    var d = e as IDictionary<string,object>;
    for (int i = 0; i < reader.FieldCount; i++)
      d.Add(reader.GetName(i),reader[i]);
    return e;
  }
}
```

![20191003044145.png](https://i.loli.net/2019/10/03/7b3vFaEQpnSVj1y.png)

 　 
 　 
## 3.Dynamic Query 原理 Part2  <a name="page3"></a>



有了前面簡單ExpandoObject Dynamic Query例子的概念後,接著進到底層來了解Dapper如何細節處理,為何要自訂義DynamicMetaObjectProvider。

#### 首先掌握Dynamic Query流程邏輯 :  

假設使用下面代碼
```C#
using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
{
    var result = cn.Query("select N'暐翰' Name,26 Age").First();
    Console.WriteLine(result.Name);
}
```

取值的過程會是 : 建立動態Func > 保存在緩存 > 使用`result.Name` > 轉成呼叫 `((DapperRow)result)["Name"]` > 從`DapperTable.Values陣列`中以`"Name"欄位對應的Index`取值

接著查看源碼GetDapperRowDeserializer方法,它掌管dynamic如何運行的邏輯,並動態建立成Func給上層API呼叫、緩存重複利用。
![20191003190836.png](https://i.loli.net/2019/10/03/x2Pre7EDdTjUhXq.png)

此段Func邏輯 : 
1. DapperTable雖然是方法內的局部變量,但是被生成的Func引用,所以`不會被GC`一直保存在內存內重複利用。
![20191003182219.png](https://i.loli.net/2019/10/03/kmXF7f5v8wTNKib.png)

2. 因為是dynamic不需要考慮類別Mapping,這邊直接使用`GetValue(index)`向數據庫取值
```
var values = new object[select欄位數量];
for (int i = 0; i < values.Length; i++)
{
    object val = r.GetValue(i);
    values[i] = val is DBNull ? null : val;
}
```

3. 將資料保存到DapperRow內
```C#
public DapperRow(DapperTable table, object[] values)
{
    this.table = table ?? throw new ArgumentNullException(nameof(table));
    this.values = values ?? throw new ArgumentNullException(nameof(values));
}
```

4. DapperRow 繼承 IDynamicMetaObjectProvider 並實作 GetMetaObject 方法,實作邏輯是返回DapperRowMetaObject物件。

```C#
private sealed partial class DapperRow : System.Dynamic.IDynamicMetaObjectProvider
{
    DynamicMetaObject GetMetaObject(Expression parameter)
    {
        return new DapperRowMetaObject(parameter, System.Dynamic.BindingRestrictions.Empty, this);
    }
}
```

5. DapperRowMetaObject主要功能是定義行為,藉由override `BindSetMember、BindGetMember`方法,Dapper定義了Get、Set的行為分別使用`IDictionary<string, object> - GetItem方法`跟`DapperRow - SetValue方法`
![20191003210351.png](https://i.loli.net/2019/10/03/xn5YMfiXkAvBq2O.png)
![20191003210547.png](https://i.loli.net/2019/10/03/u45PmaRGMQBUz7k.png)

6. 最後Dapper利用DataReader的`欄位順序性`,先利用欄位名稱取得Index,再利用Index跟Values取得值

![20191003211448.png](https://i.loli.net/2019/10/03/cGNIzAjuMv4HkyF.png)


#### 為何要繼承IDictionary<string,object>?

可以思考一個問題 : 在DapperRowMetaObject可以自行定義Get跟Set行為,那麼不使用Dictionary - GetItem方法,改用其他方式,是否代表不需要繼承`IDictionary<string,object>`?

Dapper這樣做的原因之一跟開放原則有關,DapperTable、DapperRow都是底層實作類別,基於開放封閉原則`不應該開放給使用者`,所以設為`private`權限。
```
private class DapperTable{/*略*/}
private class DapperRow :IDictionary<string, object>, IReadOnlyDictionary<string, object>,System.Dynamic.IDynamicMetaObjectProvider{/*略*/}
```

那麼使用者`想要知道欄位名稱`怎麼辦?   
因為DapperRow實作IDictionary所以可以`向上轉型為IDictionary<string, object>`,利用它為`公開介面`特性取得欄位資料。
```C#
public interface IDictionary<TKey, TValue> : ICollection<KeyValuePair<TKey, TValue>>, IEnumerable<KeyValuePair<TKey, TValue>>, IEnumerable{/*略*/}
```

舉個例子,筆者有做一個小工具[HtmlTableHelper](https://github.com/shps951023/HtmlTableHelper)就是利用這特性,自動將Dapper Dynamic Query轉成Table Html,如以下代碼跟圖片
```C#
using (var cn = "Your Connection")
{
  var sourceData = cn.Query(@"select 'ITWeiHan' Name,25 Age,'M' Gender");
  var tablehtml = sourceData.ToHtmlTable(); //Result : <table><thead><tr><th>Name</th><th>Age</th><th>Gender</th></tr></thead><tbody><tr><td>ITWeiHan</td><td>25</td><td>M</td></tr></tbody></table>
}
```

![20191003212846.png](https://i.loli.net/2019/10/03/ke5Ty8lVGJCQjKX.png)


 　 
 　 
## 4. Strongly Typed Mapping 原理 Part1 : ADO.NET對比Dapper  <a name="page4"></a>


接下來是Dapper關鍵功能 `Strongly Typed Mapping`,因為難度高,這邊會切分成多篇來解說。

第一篇先以ADO.NET DataReader GetItem By Index跟Dapper Strongly Typed Query對比,查看兩者IL的差異,了解Dapper Query Mapping的主要邏輯。

有了邏輯後,如何實作,我這邊依序用三個技術 :`Reflection、Expression、Emit` 從頭實作三個版本Query方法來讓讀者漸進式了解。

----


### ADO.NET對比Dapper

首先使用以下代碼來追蹤Dapper Query邏輯
```C#
class Program
{
  static void Main(string[] args)
  {
    using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
    {
      var result = cn.Query<User>("select N'暐翰' Name , 25 Age").First();
      Console.WriteLine(result.Name);
      Console.WriteLine(result.Age);
    }
  }
}

public class User
{
  public string Name { get; set; }
  public int Age { get; set; }
}
```

這邊需要重點來看`Dapper.SqlMapper.GenerateDeserializerFromMap`方法,它負責Mapping的邏輯,可以看到裡面大量使用Emit IL技術。

![20191004012713.png](https://i.loli.net/2019/10/04/DCVHTaOGBhcL59s.png)

要了解這段IL邏輯,我的方式 :`「不應該直接進到細節,而是先查看完整生成的IL」`,至於如何查看,這邊需要先準備 [il-visualizer](https://github.com/drewnoakes/il-visualizer) 開源工具,它可以在Runtime查看DynamicMethod生成的IL。

它預設支持vs 2015、2017,假如跟我一樣使用vs2019的讀者,需要注意
1. 需要手動解壓縮到
`%USERPROFILE%\Documents\Visual Studio 2019`路徑下面
![20191004005622.png](https://i.loli.net/2019/10/04/wMHI1xCPQfV2rZg.png)
2. `.netstandard2.0`專案,需要建立`netstandard2.0`並解壓縮到該資料夾
![20191003044307.png](https://i.loli.net/2019/10/03/TvctsVmLJFXh43O.png)


最後重開visaul studio並debug運行,進到GetTypeDeserializerImpl方法,對DynamicMethod點擊放大鏡 > IL visualizer > 查看`Runtime`生成的IL代碼
![20191003044320.png](https://i.loli.net/2019/10/03/wyrcNSdnOloFICQ.png)

可以得出以下IL
```C#
IL_0000: ldc.i4.0   
IL_0001: stloc.0    
IL_0002: newobj     Void .ctor()/Demo.User
IL_0007: stloc.1    
IL_0008: ldloc.1    
IL_0009: dup        
IL_000a: ldc.i4.0   
IL_000b: stloc.0    
IL_000c: ldarg.0    
IL_000d: ldc.i4.0   
IL_000e: callvirt   System.Object get_Item(Int32)/System.Data.IDataRecord
IL_0013: dup        
IL_0014: stloc.2    
IL_0015: dup        
IL_0016: isinst     System.DBNull
IL_001b: brtrue.s   IL_0029
IL_001d: unbox.any  System.String
IL_0022: callvirt   Void set_Name(System.String)/Demo.User
IL_0027: br.s       IL_002b
IL_0029: pop        
IL_002a: pop        
IL_002b: dup        
IL_002c: ldc.i4.1   
IL_002d: stloc.0    
IL_002e: ldarg.0    
IL_002f: ldc.i4.1   
IL_0030: callvirt   System.Object get_Item(Int32)/System.Data.IDataRecord
IL_0035: dup        
IL_0036: stloc.2    
IL_0037: dup        
IL_0038: isinst     System.DBNull
IL_003d: brtrue.s   IL_004b
IL_003f: unbox.any  System.Int32
IL_0044: callvirt   Void set_Age(Int32)/Demo.User
IL_0049: br.s       IL_004d
IL_004b: pop        
IL_004c: pop        
IL_004d: stloc.1    
IL_004e: leave      IL_0060
IL_0053: ldloc.0    
IL_0054: ldarg.0    
IL_0055: ldloc.2    
IL_0056: call       Void ThrowDataException(System.Exception, Int32, System.Data.IDataReader, System.Object)/Dapper.SqlMapper
IL_005b: leave      IL_0060
IL_0060: ldloc.1    
IL_0061: ret        
```

要了解這段IL之前需要先了解`ADO.NET DataReader快速讀取資料方式`會使用`GetItem By Index`方式,如以下代碼
```C#
public static class DemoExtension
{
  private static User CastToUser(this IDataReader reader)
  {
    var user = new User();
    var value = reader[0];
    if(!(value is System.DBNull))
      user.Name = (string)value;
    var value = reader[1];
    if(!(value is System.DBNull))
      user.Age = (int)value;      
    return user;
  }

  public static IEnumerable<User> Query<T>(this IDbConnection cnn, string sql)
  {
    if (cnn.State == ConnectionState.Closed) cnn.Open();
    using (var command = cnn.CreateCommand())
    {
      command.CommandText = sql;
      using (var reader = command.ExecuteReader())
        while (reader.Read())
          yield return reader.CastToUser();
    }
  }
}
```

接著查看此Demo - CastToUser方法生成的IL代碼  
```C#
DemoExtension.CastToUser:
IL_0000:  nop         
IL_0001:  newobj      User..ctor
IL_0006:  stloc.0     // user
IL_0007:  ldarg.0     
IL_0008:  ldc.i4.0    
IL_0009:  callvirt    System.Data.IDataRecord.get_Item
IL_000E:  stloc.1     // value
IL_000F:  ldloc.1     // value
IL_0010:  isinst      System.DBNull
IL_0015:  ldnull      
IL_0016:  cgt.un      
IL_0018:  ldc.i4.0    
IL_0019:  ceq         
IL_001B:  stloc.2     
IL_001C:  ldloc.2     
IL_001D:  brfalse.s   IL_002C
IL_001F:  ldloc.0     // user
IL_0020:  ldloc.1     // value
IL_0021:  castclass   System.String
IL_0026:  callvirt    User.set_Name
IL_002B:  nop         
IL_002C:  ldarg.0     
IL_002D:  ldc.i4.1    
IL_002E:  callvirt    System.Data.IDataRecord.get_Item
IL_0033:  stloc.1     // value
IL_0034:  ldloc.1     // value
IL_0035:  isinst      System.DBNull
IL_003A:  ldnull      
IL_003B:  cgt.un      
IL_003D:  ldc.i4.0    
IL_003E:  ceq         
IL_0040:  stloc.3     
IL_0041:  ldloc.3     
IL_0042:  brfalse.s   IL_0051
IL_0044:  ldloc.0     // user
IL_0045:  ldloc.1     // value
IL_0046:  unbox.any   System.Int32
IL_004B:  callvirt    User.set_Age
IL_0050:  nop         
IL_0051:  ldloc.0     // user
IL_0052:  stloc.s     04 
IL_0054:  br.s        IL_0056
IL_0056:  ldloc.s     04 
IL_0058:  ret         

```

跟Dapper生成的IL比對可以`發現大致是一樣的`(差異部分後面會講解),代表兩者在運行的邏輯、效率上都會是差不多的,這也是為何`Dapper效率接近原生ADO.NET`的原因之一。


 　 
 　 
## 5. Strongly Typed Mapping 原理 Part2 : Reflection版本   <a name="page5"></a>



在前面ADO.NET Mapping例子可以發現嚴重問題`「沒辦法多類別共用方法,每新增一個類別就需要重寫代碼」`。要解決這個問題,可以寫一個共用方法在Runtime時期針對不同的類別做不同的邏輯處理。

實作方式做主要有三種Reflection、Expression、Emit,這邊首先介紹最簡單方式:「Reflection」,我這邊會使用反射方式從零模擬Query寫代碼,讓讀者初步了解動態處理概念。(假如有經驗的讀者可以跳過本篇)

邏輯 : 
1. 使用泛型傳遞動態類別
2. 使用`泛型的條件約束new()`達到動態建立物件
3. DataReader需要使用`屬性字串名稱當Key`,可以使用Reflection取得動態類別的屬性名稱,在藉由`DataReader this[string parameter]`取得數據庫資料
4. 使用PropertyInfo.SetValue方式動態將數據庫資料賦予物件

最後得到以下代碼 : 

```C#
public static class DemoExtension
{
  public static IEnumerable<T> Query<T>(this IDbConnection cnn, string sql) where T : new()
  {
    using (var command = cnn.CreateCommand())
    {
      command.CommandText = sql;
      using (var reader = command.ExecuteReader())
        while (reader.Read())
          yield return reader.CastToType<T>();
    }
  }

  //1.使用泛型傳遞動態類別
  private static T CastToType<T>(this IDataReader reader) where T : new()
  {
    //2.使用泛型的條件約束new()達到動態建立物件
    var instance = new T();

    //3.DataReader需要使用屬性字串名稱當Key,可以使用Reflection取得動態類別的屬性名稱,在藉由DataReader this[string parameter]取得數據庫資料
    var type = typeof(T);
    var props = type.GetProperties();
    foreach (var p in props)
    {
      var val = reader[p.Name];

      //4.使用PropertyInfo.SetValue方式動態將數據庫資料賦予物件
      if( !(val is System.DBNull) )  
        p.SetValue(instance, val);
    }

    return instance;
  }
}
```

Reflection版本優點是代碼`簡單`,但它有以下問題
1. 不應該重複屬性查詢,沒用到就要忽略
舉例 : 假如類別有N個屬性,SQL指查詢3個欄位,土炮ORM每次PropertyInfo foreach還是N次不是3次。而Dapper在Emit IL當中特別優化此段邏輯 : `「查多少用多少,不浪費」`(這段之後講解)。
![https://ithelp.ithome.com.tw/upload/images/20191003/20105988Y7jmgF76Wd.png](https://ithelp.ithome.com.tw/upload/images/20191003/20105988Y7jmgF76Wd.png)
![https://ithelp.ithome.com.tw/upload/images/20191003/20105988nHZMb3Copc.png](https://ithelp.ithome.com.tw/upload/images/20191003/20105988nHZMb3Copc.png)
2. 效率問題 : 
  - 反射效率會比較慢,這點之後會介紹解決方式 : `「查表法 + 動態建立方法」`以空間換取時間。
  - 使用字串Key取值會多呼叫了`GetOrdinal`方法,可以查看MSDN官方解釋,`效率比Index取值差`。
![https://ithelp.ithome.com.tw/upload/images/20191003/20105988ABufu55xes.png](https://ithelp.ithome.com.tw/upload/images/20191003/20105988ABufu55xes.png)
![https://ithelp.ithome.com.tw/upload/images/20191003/20105988TqMlMbAIls.png](https://ithelp.ithome.com.tw/upload/images/20191003/20105988TqMlMbAIls.png)
  




 　 
 　 
## 6.Strongly Typed Mapping 原理 Part3 : 動態建立方法重要概念「結果反推程式碼」優化效率  <a name="page6"></a>


接著使用Expression來解決Reflection版本問題,主要是利用Expression特性 : `「可以在Runtime時期動態建立方法」`來解決問題。

在這之前需要先有一個重要概念 : `「從結果反推最簡潔代碼」`優化效率,舉個例子 : 以前初學程式時一個經典題目「打印正三角型星星」做出一個長度為3的正三角,常見作法會是迴圈+遞迴方式
```C#
void Main()
{
  Print(3,0);
}

static void Print(int length, int spaceLength)
{
  if (length < 0)
    return;
  else
    Print(length - 1, spaceLength + 1);
  for (int i = 0; i < spaceLength; i++)
    Console.Write(" ");
  for (int i = 0; i < length; i++)
    Console.Write("* ");
  Console.WriteLine("");
}
```

但其實這個題目在已經知道長度的情況下,可以被改成以下代碼
```C#
Console.WriteLine("  * ");
Console.WriteLine(" * * ");
Console.WriteLine("* * * ");
```

這個概念很重要,因為是從結果反推代碼,所以邏輯直接、`效率快`,而Dapper就是使用此概念來動態建立方法。

舉例 : 假設有一段代碼如下,我們可以從結果得出
- User Class的Name屬性對應Reader Index 0 、類別是String 、 預設值是null
- User Class的Age屬性對應Reader Index 1 、類別是int 、 預設值是0
```C#
void Main()
{
  using (var cn = Connection)
  {
    var result = cn.Query<User>("select N'暐翰' Name,26 Age").First();
  }
}

class User
{
  public string Name { get; set; }
  public int Age { get; set; }
}
```

假如系統能幫忙生成以下邏輯方法,那麼效率會是最好的
```C#
User 動態方法(IDataReader reader)
{
  var user = new User();
  var value = reader[0];
  if( !(value is System.DBNull) )
    user.Name = (string)value;
  value = reader[1];
  if( !(value is System.DBNull) )
    user.Age = (int)value;  
  return user;
}
```

另外上面例子可以看出對Dapper來說`SQL Select對應Class屬性順序很重要`,所以後面會講解Dapper在緩存的算法特別針對此優化。



 　 
 　 
## 7.Strongly Typed Mapping 原理 Part4 : Expression版本  <a name="page7"></a>


有了前面的邏輯,就著使用Expression實作動態建立方法。

#### 為何先使用 Expression 實作而不是 Emit ?
除了有能力動態建立方法,相比Emit有以下優點 :
- `可讀性好`,可用熟悉的關鍵字,像是變量Variable對應Expression.Variable、建立物件New對應Expression.New
![https://ithelp.ithome.com.tw/upload/images/20190920/20105988rkSmaILTw7.png](https://ithelp.ithome.com.tw/upload/images/20190920/20105988rkSmaILTw7.png)
- `方便Runtime Debug`,可以在Debug模式下看到Expression對應邏輯代碼
![https://ithelp.ithome.com.tw/upload/images/20190920/201059882EODD9OdnD.png](https://ithelp.ithome.com.tw/upload/images/20190920/201059882EODD9OdnD.png)
![https://ithelp.ithome.com.tw/upload/images/20190920/201059882gSYyfUduS.png](https://ithelp.ithome.com.tw/upload/images/20190920/201059882gSYyfUduS.png)

所以特別適合介紹動態方法建立,但Expression相比Emit無法作一些細節操作,這點會在後面Emit講解到。


#### 改寫Expression版本

邏輯 : 
1. 取得sql select所有欄位名稱
2. 取得mapping類別的屬性資料 >  將index,sql欄位,class屬性資料做好對應封裝在一個變量內方便後面使用
3. 動態建立方法 : 從數據庫Reader按照順序讀取我們要的資料,其中代碼邏輯 : 
```C#
User 動態方法(IDataReader reader)
{
  var user = new User();
  var value = reader[0];
  if( !(value is System.DBNull) )
    user.Name = (string)value;
  value = reader[1];
  if( !(value is System.DBNull) )
    user.Age = (int)value;  
  return user;
}
```

最後得出以下Exprssion版本代碼
```C#
public static class DemoExtension
{
  public static IEnumerable<T> Query<T>(this IDbConnection cnn, string sql) where T : new()
  {
    using (var command = cnn.CreateCommand())
    {
      command.CommandText = sql;
      using (var reader = command.ExecuteReader())
      {
        var func = CreateMappingFunction(reader, typeof(T));
        while (reader.Read())
        {
          var result = func(reader as DbDataReader);
          yield return result is T ? (T)result : default(T);
        }

      }
    }
  }

  private static Func<DbDataReader, object> CreateMappingFunction(IDataReader reader, Type type)
  {
    //1. 取得sql select所有欄位名稱
    var names = Enumerable.Range(0, reader.FieldCount).Select(index => reader.GetName(index)).ToArray();

    //2. 取得mapping類別的屬性資料 >  將index,sql欄位,class屬性資料做好對應封裝在一個變量內方便後面使用
    var props = type.GetProperties().ToList();
    var members = names.Select((columnName, index) =>
    {
      var property = props.Find(p => string.Equals(p.Name, columnName, StringComparison.Ordinal))
      ?? props.Find(p => string.Equals(p.Name, columnName, StringComparison.OrdinalIgnoreCase));
      return new
      {
        index,
        columnName,
        property
      };
    });

    //3. 動態建立方法 : 從數據庫Reader按照順序讀取我們要的資料
    /*方法邏輯 : 
      User 動態方法(IDataReader reader)
      {
        var user = new User();
        var value = reader[0];
        if( !(value is System.DBNull) )
          user.Name = (string)value;
        value = reader[1];
        if( !(value is System.DBNull) )
          user.Age = (int)value;  
        return user;
      }
    */
    var exBodys = new List<Expression>();
    {
      // 方法(IDataReader reader)
      var exParam = Expression.Parameter(typeof(DbDataReader), "reader");

      // Mapping類別 物件 = new Mapping類別();
      var exVar = Expression.Variable(type, "mappingObj");
      var exNew = Expression.New(type);
      {
        exBodys.Add(Expression.Assign(exVar, exNew));
      }

      // var value = defalut(object);
      var exValueVar = Expression.Variable(typeof(object), "value");
      {
        exBodys.Add(Expression.Assign(exValueVar, Expression.Constant(null)));
      }


      var getItemMethod = typeof(DbDataReader).GetMethods().Where(w => w.Name == "get_Item")
        .First(w => w.GetParameters().First().ParameterType == typeof(int));
      foreach (var m in members)
      {
        //reader[0]
        var exCall = Expression.Call(
          exParam, getItemMethod,
          Expression.Constant(m.index)
        );

        // value = reader[0];
        exBodys.Add(Expression.Assign(exValueVar, exCall));

        //user.Name = (string)value;
        var exProp = Expression.Property(exVar, m.property.Name);
        var exConvert = Expression.Convert(exValueVar, m.property.PropertyType); //(string)value
        var exPropAssign = Expression.Assign(exProp, exConvert);

        //if ( !(value is System.DBNull))
        //    (string)value
        var exIfThenElse = Expression.IfThen(
          Expression.Not(Expression.TypeIs(exValueVar, typeof(System.DBNull)))
          , exPropAssign
        );

        exBodys.Add(exIfThenElse);
      }


      // return user;  
      exBodys.Add(exVar);

      // Compiler Expression 
      var lambda = Expression.Lambda<Func<DbDataReader, object>>(
        Expression.Block(
          new[] { exVar, exValueVar },
          exBodys
        ), exParam
      );

      return lambda.Compile();
    }
  }
}
```

查詢效果圖 : 
![20191004205645.png](https://i.loli.net/2019/10/04/7Sbtr6gRdm4DyLs.png)


最後查看Expression.Lambda > DebugView(注意是非公開屬性)驗證代碼 : 
```C#
.Lambda #Lambda1<System.Func`2[System.Data.Common.DbDataReader,System.Object]>(System.Data.Common.DbDataReader $reader) {
    .Block(
        UserQuery+User $mappingObj,
        System.Object $value) {
        $mappingObj = .New UserQuery+User();
        $value = null;
        $value = .Call $reader.get_Item(0);
        .If (
            !($value .Is System.DBNull)
        ) {
            $mappingObj.Name = (System.String)$value
        } .Else {
            .Default(System.Void)
        };
        $value = .Call $reader.get_Item(1);
        .If (
            !($value .Is System.DBNull)
        ) {
            $mappingObj.Age = (System.Int32)$value
        } .Else {
            .Default(System.Void)
        };
        $mappingObj
    }
}
```

![20191005035640.png](https://i.loli.net/2019/10/05/3KVFvHyYS6kjDAs.png)

 　 
 　 
## 8. Strongly Typed Mapping 原理 Part5 : Emit IL反建立C#代碼  <a name="page8"></a>


有了前面Expression版本概念後,接著可以進到Dapper底層最核心的技術 : Emit。

首先要有個概念,MSIL(CIL)目的是給JIT編譯器看的,所以可讀性會很差、難Debug,但比起Expression來說可以做到更細節的邏輯操作。

在實際環境開發使用Emit,一般會`先寫好C#代碼後 > 反編譯查看IL > 使用Emit建立動態方法`,舉例 : 

1.首先建立一個簡單打印例子 : 
```C#
void SyaHello()
{
  Console.WriteLine("Hello World");  
}
```
2.反編譯查看IL
```
SyaHello:
IL_0000:  nop         
IL_0001:  ldstr       "Hello World"
IL_0006:  call        System.Console.WriteLine
IL_000B:  nop         
IL_000C:  ret  
```
3.使用DynamicMethod + Emit建立動態方法
```C#
void Main()
{
  // 1. 建立 void 方法()
  DynamicMethod methodbuilder = new DynamicMethod("Deserialize" + Guid.NewGuid().ToString(),typeof(void),null);

  // 2. 建立方法Body內容,藉由Emit
  var il = methodbuilder.GetILGenerator();
  il.Emit(OpCodes.Ldstr, "Hello World");
  Type[] types = new Type[1]
  {
    typeof(string)
  };
  MethodInfo method = typeof(Console).GetMethod("WriteLine", types);
  il.Emit(OpCodes.Call,method);
  il.Emit(OpCodes.Ret);
  
  // 3. 轉換指定類型的Func or Action
  var action = (Action)methodbuilder.CreateDelegate(typeof(Action));
  
  action(); 
}
```

![https://ithelp.ithome.com.tw/upload/images/20190924/20105988bD9GSXyjNt.png](https://ithelp.ithome.com.tw/upload/images/20190924/20105988bD9GSXyjNt.png)


但是對已經寫好的專案來說就不是這樣流程了,開發者不一定會好心的告訴你當初設計的邏輯,所以接著討論此問題。

#### 如果像是Dapper只有Emit IL沒有C# Source Code專案怎麼辦?
我的解決方式是 : `「既然只有Runtime才能知道IL,那麼將IL保存成靜態檔案再反編譯查看」`

這邊可以使用`MethodBuild + Save`方法將`IL保存成靜態exe檔案 > 反編譯查看`,但需要特別注意
1. 請對應好參數跟返回類別,否則會編譯錯誤。
2. netstandard不支援此方式,Dapper需要使用`region if 指定版本`來做區分,否則不能使用,如圖片
![20191004230125.png](https://i.loli.net/2019/10/04/ZUCKIcpnjmPB5Fb.png)

代碼如下 : 
```C#
  //使用MethodBuilder查看別人已經寫好的Emit IL
  //1. 建立MethodBuilder
  AppDomain ad = AppDomain.CurrentDomain;
  AssemblyName am = new AssemblyName();
  am.Name = "TestAsm";
  AssemblyBuilder ab = ad.DefineDynamicAssembly(am, AssemblyBuilderAccess.Save);
  ModuleBuilder mb = ab.DefineDynamicModule("Testmod", "TestAsm.exe");
  TypeBuilder tb = mb.DefineType("TestType", TypeAttributes.Public);
  MethodBuilder dm = tb.DefineMethod("TestMeThod", MethodAttributes.Public |
  MethodAttributes.Static, type, new[] { typeof(IDataReader) });
  ab.SetEntryPoint(dm);

  // 2. 填入IL代碼
  //..略

  // 3. 生成靜態檔案
  tb.CreateType();
  ab.Save("TestAsm.exe");
```
 
接著使用此方式在GetTypeDeserializerImpl方法反編譯Dapper Query Mapping IL,可以得出C#代碼 : 

```C#
public static User TestMeThod(IDataReader P_0)
{
  int index = 0;
  User user = new User();
  object value = default(object);
  try
  {
    User user2 = user;
    index = 0;
    object obj = value = P_0[0];
    if (!(obj is DBNull))
    {
      user2.Name = (string)obj;
    }
    index = 1;
    object obj2 = value = P_0[1];
    if (!(obj2 is DBNull))
    {
      user2.Age = (int)obj2;
    }
    user = user2;
    return user;
  }
  catch (Exception ex)
  {
    SqlMapper.ThrowDataException(ex, index, P_0, value);
    return user;
  }
}
```

![20191004230548.png](https://i.loli.net/2019/10/04/YBcnfCz2FtudaWX.png)

有了C#代碼後再來了解Emit邏輯會快很多,接著就可以進到Emit版本Query實作部分。

 　 
 　 
## 9.Strongly Typed Mapping 原理 Part6 : Emit版本  <a name="page9"></a>


以下代碼是Emit版本,我把C#對應IL部分都寫在註解。
```C#
public static class DemoExtension
{
  public static IEnumerable<T> Query<T>(this IDbConnection cnn, string sql) where T : new()
  {
    using (var command = cnn.CreateCommand())
    {
      command.CommandText = sql;
      using (var reader = command.ExecuteReader())
      {
        var func = GetTypeDeserializerImpl(typeof(T), reader);

        while (reader.Read())
        {
          var result = func(reader as DbDataReader);
          yield return result is T ? (T)result : default(T);
        }
      }

    }
  }

  private static Func<DbDataReader, object> GetTypeDeserializerImpl(Type type, IDataReader reader, int startBound = 0, int length = -1, bool returnNullIfFirstMissing = false)
  {
    var returnType = type.IsValueType ? typeof(object) : type;

    var dm = new DynamicMethod("Deserialize" + Guid.NewGuid().ToString(), returnType, new[] { typeof(IDataReader) }, type, true);
    var il = dm.GetILGenerator();

    //C# : User user = new User();
    //IL : 
    //IL_0001:  newobj      
    //IL_0006:  stloc.0         
    var constructor = returnType.GetConstructors(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic)[0]; //這邊簡化成只會有預設constructor
    il.Emit(OpCodes.Newobj, constructor);
    var returnValueLocal = il.DeclareLocal(type);
    il.Emit(OpCodes.Stloc, returnValueLocal); //User user = new User();

    // C# : 
    //object value = default(object);
    // IL :
    //IL_0007: ldnull
    //IL_0008:  stloc.1     // value  
    var valueLoacl = il.DeclareLocal(typeof(object));
    il.Emit(OpCodes.Ldnull);
    il.Emit(OpCodes.Stloc, valueLoacl);
    
    
    int index = startBound;
    var getItem = typeof(IDataRecord).GetProperties(BindingFlags.Instance | BindingFlags.Public)
            .Where(p => p.GetIndexParameters().Length > 0 && p.GetIndexParameters()[0].ParameterType == typeof(int))
            .Select(p => p.GetGetMethod()).First();

    foreach (var p in type.GetProperties())
    {
      //C# : value = P_0[0];
      //IL:
      //IL_0009:  ldarg.0      
      //IL_000A: ldc.i4.0
      //IL_000B: callvirt System.Data.IDataRecord.get_Item
      //IL_0010:  stloc.1     // value        
      il.Emit(OpCodes.Ldarg_0); //取得reader參數
      EmitInt32(il, index);
      il.Emit(OpCodes.Callvirt, getItem);
      il.Emit(OpCodes.Stloc, valueLoacl);


      //C#: if (!(value is DBNull)) user.Name = (string)value;
      //IL:
      // IL_0011:  ldloc.1     // value
      // IL_0012:  isinst      System.DBNull
      // IL_0017:  ldnull      
      // IL_0018:  cgt.un      
      // IL_001A:  ldc.i4.0   
      // IL_001B:  ceq         
      // IL_001D:  stloc.2    
      // IL_001E:  ldloc.2     
      // IL_001F:  brfalse.s   IL_002E
      // IL_0021:  ldloc.0     // user
      // IL_0022:  ldloc.1     // value
      // IL_0023:  castclass   System.String
      // IL_0028:  callvirt    UserQuery+User.set_Name      
      il.Emit(OpCodes.Ldloc, valueLoacl);
      il.Emit(OpCodes.Isinst, typeof(System.DBNull));
      il.Emit(OpCodes.Ldnull);
      
      var tmpLoacl = il.DeclareLocal(typeof(int));
      il.Emit(OpCodes.Cgt_Un);
      il.Emit(OpCodes.Ldc_I4_0);
      il.Emit(OpCodes.Ceq);
      
      il.Emit(OpCodes.Stloc,tmpLoacl);
      il.Emit(OpCodes.Ldloc,tmpLoacl);
      
      
      var labelFalse = il.DefineLabel();
      il.Emit(OpCodes.Brfalse_S,labelFalse);
      il.Emit(OpCodes.Ldloc, returnValueLocal);
      il.Emit(OpCodes.Ldloc, valueLoacl);
      if (p.PropertyType.IsValueType)
        il.Emit(OpCodes.Unbox_Any, p.PropertyType);
      else
        il.Emit(OpCodes.Castclass, p.PropertyType);
      il.Emit(OpCodes.Callvirt, p.SetMethod);
      
      il.MarkLabel(labelFalse);

      index++;
    }

    // IL_0053:  ldloc.0     // user
    // IL_0054:  stloc.s     04  //不需要
    // IL_0056:  br.s        IL_0058
    // IL_0058:  ldloc.s     04  //不需要
    // IL_005A:  ret         
    il.Emit(OpCodes.Ldloc, returnValueLocal);
    il.Emit(OpCodes.Ret);

    var funcType = System.Linq.Expressions.Expression.GetFuncType(typeof(IDataReader), returnType);
    return (Func<IDataReader, object>)dm.CreateDelegate(funcType);
  }

  private static void EmitInt32(ILGenerator il, int value)
  {
    switch (value)
    {
      case -1: il.Emit(OpCodes.Ldc_I4_M1); break;
      case 0: il.Emit(OpCodes.Ldc_I4_0); break;
      case 1: il.Emit(OpCodes.Ldc_I4_1); break;
      case 2: il.Emit(OpCodes.Ldc_I4_2); break;
      case 3: il.Emit(OpCodes.Ldc_I4_3); break;
      case 4: il.Emit(OpCodes.Ldc_I4_4); break;
      case 5: il.Emit(OpCodes.Ldc_I4_5); break;
      case 6: il.Emit(OpCodes.Ldc_I4_6); break;
      case 7: il.Emit(OpCodes.Ldc_I4_7); break;
      case 8: il.Emit(OpCodes.Ldc_I4_8); break;
      default:
        if (value >= -128 && value <= 127)
        {
          il.Emit(OpCodes.Ldc_I4_S, (sbyte)value);
        }
        else
        {
          il.Emit(OpCodes.Ldc_I4, value);
        }
        break;
    }
  }
}
```

這邊Emit的細節概念非常的多,這邊無法全部都講解,先挑出重要概念講解

#### Emit Label
在Emit if/else需要使用Label定位,告知編譯器條件為true/false時要跳到哪個位子,舉例 : 「boolean轉整數」,假設要簡單將Boolean轉換成Int,C#代碼可以用「如果是True返回1否則返回0」邏輯來寫:
```C#
  public static int BoolToInt(bool input) => input ? 1 : 0;
```

當轉成Emit寫法的時候,需要以下邏輯 :  
1. 考慮Label動態定位問題
2. 先要建立好Label讓Brtrue_S知道符合條件時要去哪個Label位子 `(注意,這時候Label位子還沒確定)`
3. 繼續按順序由上而下建立IL
4. 等到了`符合條件`要運行區塊的`前一行`,使用`MarkLabel方法標記Label的位子`。

最後寫出的C# Emit代碼 : 
```C#
public class Program
{
  public static void Main(string[] args)
  {
    var func = CreateFunc();
    Console.WriteLine(func(true)); //1
    Console.WriteLine(func(false)); //0
  }

  static Func<bool, int> CreateFunc()
  {
    var dm = new DynamicMethod("Test" + Guid.NewGuid().ToString(), typeof(int), new[] { typeof(bool) });

    var il = dm.GetILGenerator();
    var labelTrue = il.DefineLabel();

    il.Emit(OpCodes.Ldarg_0);
    il.Emit(OpCodes.Brtrue_S, labelTrue);
    il.Emit(OpCodes.Ldc_I4_0);
    il.Emit(OpCodes.Ret);
    il.MarkLabel(labelTrue);
    il.Emit(OpCodes.Ldc_I4_1);
    il.Emit(OpCodes.Ret);

    var funcType = System.Linq.Expressions.Expression.GetFuncType(typeof(bool), typeof(int));
    return (Func<bool, int>)dm.CreateDelegate(funcType);
  }
}
```

---

這邊可以發現Emit版本
優點 : 
1. 能做更多細節的操作
2. 因為細節顆粒度小,可以優化的效率更好

缺點 : 
1. 難以Debug
2. 可讀性差
3. 代碼量變大、複雜度增加

接著來看Dapper作者的建議,現在一般專案當中沒有必要使用Emit,使用Expression + Func/Action已經可以解決大部分動態方法的需求,尤其是Expression支援Block等方法情況。連結 [c# - What's faster: expression trees or manually emitting IL ](https://stackoverflow.com/questions/16530539/whats-faster-expression-trees-or-manually-emitting-il)

![20190927163441.png](https://i.loli.net/2019/09/27/Y4WeGHu8amVdIwz.png)

話雖如此,但有一些厲害的開源專案就是使用Emit管理細節,`如果想看懂它們,就需要基礎的Emit IL概念`。




 　 
 　 
## 10.Dapper 效率快關鍵之一 : Cache 緩存原理   <a name="page10"></a>


#### 為何Dapper可以這麼快?
前面介紹到動態使用 Emit IL 建立 ADO.NET Mapping 方法,但單就這功能無法讓 Dapper 被稱為輕量ORM效率之王。

因為動態建立方法是`需要成本、並耗費時間`的動作,單純使用反而會拖慢速度。但當配合 Cache 後就不一樣,將建立好的方法保存在 Cache 內,可以用`『空間換取時間』`概念加快查詢的效率,也就是俗稱`查表法`。

接著追蹤Dapper源碼,這次需要特別關注的是QueryImpl方法下的Identity、GetCacheInfo
![https://ithelp.ithome.com.tw/upload/images/20191005/20105988cCwaS7ejnY.png](https://ithelp.ithome.com.tw/upload/images/20191005/20105988cCwaS7ejnY.png)


#### Identity、GetCacheInfo

Identity主要封裝各緩存的比較Key屬性 : 
- sql : 區分不同SQL字串
- type : 區分Mapping類別
- commandType : 負責區分不同數據庫
- gridIndex : 主用用在QueryMultiple,後面講解。
- connectionString : 主要區分同數據庫廠商但是不同DB情況
- parametersType : 主要區分參數類別
- typeCount : 主要用在Multi Query多映射,需要搭配override GetType方法,後面講解

接著搭配GetCacheInfo方法內Dapper使用的緩存類別`ConcurrentDictionary<Identity, CacheInfo>`,使用`TryGetValue`方法時會去先比對HashCode接著比對Equals特性,如圖片源碼。
![https://ithelp.ithome.com.tw/upload/images/20191005/20105988tOgZiBCwly.png](https://ithelp.ithome.com.tw/upload/images/20191005/20105988tOgZiBCwly.png)


將Key類別Identity藉由`override Equals`方法實現緩存比較算法,可以看到以下Dapper實作邏輯,只要一個屬性不一樣就會建立一個新的動態方法、緩存。
```C#
public bool Equals(Identity other)
{
  if (ReferenceEquals(this, other)) return true;
  if (ReferenceEquals(other, null)) return false;

  int typeCount;
  return gridIndex == other.gridIndex
    && type == other.type
    && sql == other.sql
    && commandType == other.commandType
    && connectionStringComparer.Equals(connectionString, other.connectionString)
    && parametersType == other.parametersType
    && (typeCount = TypeCount) == other.TypeCount
    && (typeCount == 0 || TypesEqual(this, other, typeCount));
}
```


以此概念拿之前Emit版本修改成一個簡單Cache Demo讓讀者感受:
```C#
public class Identity
{
  public string sql { get; set; }
  public CommandType? commandType { get; set; }
  public string connectionString { get; set; }
  public Type type { get; set; }
  public Type parametersType { get; set; }
  public Identity(string sql, CommandType? commandType, string connectionString, Type type, Type parametersType)
  {
    this.sql = sql;
    this.commandType = commandType;
    this.connectionString = connectionString;
    this.type = type;
    this.parametersType = parametersType;
    unchecked
    {
      hashCode = 17; // we *know* we are using this in a dictionary, so pre-compute this
      hashCode = (hashCode * 23) + commandType.GetHashCode();
      hashCode = (hashCode * 23) + (sql?.GetHashCode() ?? 0);
      hashCode = (hashCode * 23) + (type?.GetHashCode() ?? 0);
      hashCode = (hashCode * 23) + (connectionString == null ? 0 : StringComparer.Ordinal.GetHashCode(connectionString));
      hashCode = (hashCode * 23) + (parametersType?.GetHashCode() ?? 0);
    }
  }

  public readonly int hashCode;
  public override int GetHashCode() => hashCode;
  
  public override bool Equals(object obj) => Equals(obj as Identity);
  public bool Equals(Identity other)
  {
    if (ReferenceEquals(this, other)) return true;
    if (ReferenceEquals(other, null)) return false;

    return type == other.type
      && sql == other.sql
      && commandType == other.commandType
      && StringComparer.Ordinal.Equals(connectionString, other.connectionString)
      && parametersType == other.parametersType;
  }
}

public static class DemoExtension
{
  private static readonly Dictionary<Identity, Func<DbDataReader, object>> readers = new Dictionary<Identity, Func<DbDataReader, object>>();

  public static IEnumerable<T> Query<T>(this IDbConnection cnn, string sql,object param=null) where T : new()
  {
    using (var command = cnn.CreateCommand())
    {
      command.CommandText = sql;
      using (var reader = command.ExecuteReader())
      {
        var identity = new Identity(command.CommandText, command.CommandType, cnn.ConnectionString, typeof(T), param?.GetType());
        
        // 2. 如果cache有資料就使用,沒有資料就動態建立方法並保存在緩存內
        if (!readers.TryGetValue(identity, out Func<DbDataReader, object> func))
        {
          //動態建立方法
          func = GetTypeDeserializerImpl(typeof(T), reader);
          readers[identity] = func;
          Console.WriteLine("沒有緩存,建立動態方法放進緩存");
        }else{
          Console.WriteLine("使用緩存");
        }


        // 3. 呼叫生成的方法by reader,讀取資料回傳
        while (reader.Read())
        {
          var result = func(reader as DbDataReader);
          yield return result is T ? (T)result : default(T);
        }
      }

    }
  }

  private static Func<DbDataReader, object> GetTypeDeserializerImpl(Type type, IDataReader reader, int startBound = 0, int length = -1, bool returnNullIfFirstMissing = false)
  {
    //..略
  }
}
```

效果圖 : 
![https://ithelp.ithome.com.tw/upload/images/20191005/20105988mKud6Ejzqe.png](https://ithelp.ithome.com.tw/upload/images/20191005/20105988mKud6Ejzqe.png)



 　 
 　 
## 11.錯誤SQL字串拼接方式,會導致效率慢、內存洩漏 <a name="page11"></a>


了解實作邏輯後,接著延伸一個Dapper使用的重要觀念,`SQL字串`為緩存重要Key值之一,假如不同的SQL字串,Dapper會為此建立新的動態方法、緩存,所以使用不當情況下就算使用StringBuilder`也會造成效率慢、內存洩漏問題`。

![https://ithelp.ithome.com.tw/upload/images/20190916/20105988lAFBAWbhS6.png](https://ithelp.ithome.com.tw/upload/images/20190916/20105988lAFBAWbhS6.png)

至於為何要以SQL字串當其中一個關鍵Key,而不是單純使用Mapping類別的Handle,其中原因之一是跟`查詢欄位順序`有關,在前面有講到,Dapper使用`「結果反推程式碼」`方式建立動態方法,代表說順序跟資料都必須要是`固定`的,避免SQL Select欄位順序不一樣又使用同一組動態方法,會有`A欄位值給B屬性`錯值大問題。

最直接解決方式,對每個不同SQL字串建立不同的動態方法,並保存在不同的緩存。

舉例,以下代碼只是簡單的查詢動作,查看Dapper Cache數量卻達到999999個,如Gif動畫顯示
```C#
using (var cn = new SqlConnection(@"connectionString"))
{
    for (int i = 0; i < 999999; i++)
    {
        var guid = Guid.NewGuid();
        for (int i2 = 0; i2 < 2; i2++)
        {
            var result = cn.Query<User>($"select '{guid}' ").First();
        }  
    }
}
```

![zeiAPVJ](https://i.loli.net/2019/10/03/R8piq1uN4UZWg6t.gif)

要避免此問題,只需要保持一個原則`重複利用SQL字串`,而最簡單方式就是`參數化`, 舉例 : 將上述代碼改成以下代碼,緩存數量降為`1`,達到重複利用目的 : 
```C#
using (var cn = new SqlConnection(@"connectionString"))
{
    for (int i = 0; i < 999999; i++)
    {
        var guid = Guid.NewGuid();
        for (int i2 = 0; i2 < 2; i2++)
        {
            var result = cn.Query<User>($"select @guid ",new { guid}).First();
        }  
    }
}
```

![4IR5M47](https://i.loli.net/2019/10/03/3ZhQa5dwAE9Cmty.gif)

 　 
 　 
## 12.Dapper SQL正確字串拼接方式 : Literal Replacement   <a name="page12"></a>


假如遇到必要拼接SQL字串需求的情況下,舉例 : 有時候值使用字串拼接會比不使用參數化效率好,特別是該欄位值只會有幾種`固定值`。

這時候Dapper可以使用`Literal Replacements`功能,使用方式 : 將要拼接的值字串以`{=屬性名稱}`取代,並將值保存在Parameter參數內,舉例 : 
```C#
void Main()
{
  using (var cn = Connection)
  {
    var result = cn.Query("select N'暐翰' Name,26 Age,{=VipLevel} VipLevel", new User{ VipLevel = 1}).First();
  }
}
```

#### 為什麼Literal Replacement可以避免緩存問題
首先追蹤源碼GetCacheInfo下GetLiteralTokens方法,可以發現Dapper在建立`緩存之前`會抓取`SQL字串`內符合`{=變量名稱}`規格的資料。

```C#
private static readonly Regex literalTokens = new Regex(@"(?<![\p{L}\p{N}_])\{=([\p{L}\p{N}_]+)\}", RegexOptions.IgnoreCase | RegexOptions.Multiline | RegexOptions.CultureInvariant | RegexOptions.Compiled);
internal static IList<LiteralToken> GetLiteralTokens(string sql)
{
  if (string.IsNullOrEmpty(sql)) return LiteralToken.None;
  if (!literalTokens.IsMatch(sql)) return LiteralToken.None;

  var matches = literalTokens.Matches(sql);
  var found = new HashSet<string>(StringComparer.Ordinal);
  List<LiteralToken> list = new List<LiteralToken>(matches.Count);
  foreach (Match match in matches)
  {
    string token = match.Value;
    if (found.Add(match.Value))
    {
      list.Add(new LiteralToken(token, match.Groups[1].Value));
    }
  }
  return list.Count == 0 ? LiteralToken.None : list;
}
```

接著在CreateParamInfoGenerator方法生成Parameter參數化動態方法,此段方法IL如下 : 
```
IL_0000: ldarg.1    
IL_0001: castclass  <>f__AnonymousType1`1[System.Int32]
IL_0006: stloc.0    
IL_0007: ldarg.0    
IL_0008: callvirt   System.Data.IDataParameterCollection get_Parameters()/System.Data.IDbCommand
IL_000d: pop        
IL_000e: ldarg.0    
IL_000f: ldarg.0    
IL_0010: callvirt   System.String get_CommandText()/System.Data.IDbCommand
IL_0015: ldstr      "{=VipLevel}"
IL_001a: ldloc.0    
IL_001b: callvirt   Int32 get_VipLevel()/<>f__AnonymousType1`1[System.Int32]
IL_0020: stloc.1    
IL_0021: ldloca.s   V_1

IL_0023: call       System.Globalization.CultureInfo get_InvariantCulture()/System.Globalization.CultureInfo
IL_0028: call       System.String ToString(System.IFormatProvider)/System.Int32
IL_002d: callvirt   System.String Replace(System.String, System.String)/System.String
IL_0032: callvirt   Void set_CommandText(System.String)/System.Data.IDbCommand
IL_0037: ret        
```

接著再生成Mapping動態方法,要了解此段邏輯我這邊做一個模擬例子方便讀者理解 : 
```C#
public static class DbExtension
{
  public static IEnumerable<User> Query(this DbConnection cnn, string sql, User parameter)
  {
    using (var command = cnn.CreateCommand())
    {
      command.CommandText = sql;
      CommandLiteralReplace(command, parameter);
      using (var reader = command.ExecuteReader())
        while (reader.Read())
          yield return Mapping(reader);
    }
  }

  private static void CommandLiteralReplace(IDbCommand cmd, User parameter)
  {
    cmd.CommandText = cmd.CommandText.Replace("{=VipLevel}", parameter.VipLevel.ToString(System.Globalization.CultureInfo.InvariantCulture));
  }

  private static User Mapping(IDataReader reader)
  {
    var user = new User();
    var value = default(object);
    value = reader[0];
    if(!(value is System.DBNull))
      user.Name = (string)value;
    value = reader[1];
    if (!(value is System.DBNull))
      user.Age = (int)value;
    value = reader[2];
    if (!(value is System.DBNull))
      user.VipLevel = (int)value;
    return user;
  }
}
```

看完以上例子,可以發現Dapper Literal Replacements底層原理就是`字串取代`,同樣屬於字串拼接方式,為何可以避免緩存問題?

這是因為取代的時機點在SetParameter動態方法內,所以Cache的`SQL Key是沒有變動過的`,可以重複利用同樣的SQL字串、緩存。

也因為是字串取代方式,所以`只支持基本Value類別`,假如使用String類別系統會告知`The type String is not supported for SQL literals.`,避免SQL Injection問題。


 　 
 　 
## 13.Query Multi Mapping 使用方式  <a name="page13"></a>


接著講解`Dapper Multi Mapping`(多對應)實作跟底層邏輯,畢竟工作當中不可能都是一對一概念。

使用方式 : 
- 需要自己編寫Mapping邏輯,使用方式 : `Query<Func邏輯>(SQL,Parameter,Mapping邏輯Func)`
- 需要指定泛型參數類別,規則為`Query<Func第一個類別,Func第二個類別,..以此類推,Func最後返回類別>` (最多支持六組泛型參數)
- 指定切割欄位名稱,預設使用`ID`,假如不一樣需要特別指定 (這段後面特別講解) 
- 以上順序都是`由左至右`

舉例 : 有訂單(Order)跟會員(User)表格,關係是一對多關係,一個會員可以有多個訂單,以下是C# Demo代碼 : 
```C#
void Main()
{
  using (var ts = new TransactionScope())
  using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
  {
    cn.Execute(@"
      CREATE TABLE [User]([ID] int, [Name] nvarchar(10));
      INSERT INTO [User]([ID], [Name])VALUES(1, N'大雄'),(2, N'小明');

      CREATE TABLE [Order]([ID] int, [OrderNo] varchar(13), [UserID] int);
      INSERT INTO [Order]([ID], [OrderNo], [UserID])VALUES(1, 'SO20190900001', 1),(2, 'SO20190900002', 1),(3, 'SO20190900003', 2),(4, 'SO20190900004', 2);
    ");

    var result = cn.Query<Order,User,Order>(@"
        select * from [order] T1
        left join [User] T2 on T1.UserId = T2.ID    
      ", (order, user) => { 
        order.User = user;
        return order;
      }
    );
    
    ts.Dispose();
  }
}

public class Order
{
  public int ID { get; set; }  
  public string OrderNo { get; set; }  
  public User User { get; set; }  
}

public class User
{
  public int ID { get; set; }
  public string Name { get; set; }
}
```

![20191001145311.png](https://i.loli.net/2019/10/01/vEXgOINbojLs6yT.png)


----

#### 支援dynamic Multi Mapping
在初期常變動表格結構或是一次性功能不想宣告Class,Dapper Multi Mapping也支援dynamic方式

```C#
void Main()
{
  using (var ts = new TransactionScope())
  using (var connection = Connection)
  {
    const string createSql = @"
            create table Users (Id int, Name nvarchar(20))
            create table Posts (Id int, OwnerId int, Content nvarchar(20))

            insert Users values(1, N'小明')
            insert Users values(2, N'小智')

            insert Posts values(101, 1, N'小明第1天日記')
            insert Posts values(102, 1, N'小明第2天日記')
            insert Posts values(103, 2, N'小智第1天日記')
    ";
    connection.Execute(createSql);

    const string sql =
      @"select * from Posts p 
      left join Users u on u.Id = p.OwnerId 
      Order by p.Id
    ";

    var data = connection.Query<dynamic, dynamic, dynamic>(sql, (post, user) => { post.Owner = user; return post; }).ToList();
  }
}
```

![20191002023135.png](https://i.loli.net/2019/10/02/cOP2mrlRTGgVit1.png)


### SplitOn區分類別Mapping組別 

Split預設是用來切割主鍵,所以預設切割字串是`Id`,假如當表格結構PK名稱為`Id`可以省略參數,舉例
![20191001151715.png](https://i.loli.net/2019/10/01/4UzltBks7ae3TDA.png)
```C#
var result = cn.Query<Order,User,Order>(@"
  select * from [order] T1
  left join [User] T2 on T1.UserId = T2.ID    
  ", (order, user) => { 
    order.User = user;
    return order;
  }
);
```


假如主鍵名稱是其他名稱,`請指定splitOn字串名稱`,並且對應多個可以使用`,`做區隔,舉例,添加商品表格做Join : 
```
var result = cn.Query<Order,User,Item,Order>(@"
  select * from [order] T1
  left join [User] T2 on T1.UserId = T2.ID  
  left join [Item] T3 on T1.ItemId = T3.ID
  "
  
  ,map :  (order, user,item) => { 
    order.User = user;
    order.Item = item;
    return order;
  }
  ,splitOn : "Id,Id"
);
```

 　 
 　 
## 14.Query Multi Mapping 底層原理  <a name="page14"></a>


### Multiple Mapping 底層原理

這邊先以一個簡單Demo帶讀者了解Dapper Multi Mapping 概念
1. 按照泛型類別參數數量建立對應數量的Mapping Func集合
2. Mapping Func建立邏輯跟Query Emit IL一樣
3. 呼叫使用者的Custom Mapping Func,其中參數由前面動態生成的Mapping Func而來

```C#
public static class MutipleMappingDemo
{
  public static IEnumerable<TReturn> Query<T1, T2, TReturn>(this IDbConnection connection, string sql, Func<T1, T2, TReturn> map)
    where T1 : Order, new() 
    where T2 : User, new() //這兩段where單純為了Demo方便
  {
    //1. 按照泛型類別參數數量建立對應數量的Mapping Func集合
    var deserializers = new List<Func<IDataReader, object>>();
    {
      //2. Mapping Func建立邏輯跟Query Emit IL一樣
      deserializers.Add((reader) =>
      {
        var newObj = new T1();
        var value = default(object);
        value = reader[0];
        newObj.ID = value is DBNull ? 0 : (int)value;
        value = reader[1];
        newObj.OrderNo = value is DBNull ? null : (string)value;
        return newObj;
      });

      deserializers.Add((reader) =>
      {
        var newObj = new T2();
        var value = default(object);
        value = reader[2];
        newObj.ID = value is DBNull ? 0 : (int)value;
        value = reader[4];
        newObj.Name = value is DBNull ? null : (string)value;
        return newObj;
      });
    }


    using (var command = connection.CreateCommand())
    {
      command.CommandText = sql;
      using (var reader = command.ExecuteReader())
      {
        while (reader.Read())
        {
          //3. 呼叫使用者的Custom Mapping Func,其中參數由前面動態生成的Mapping Func而來
          yield return map(deserializers[0](reader) as T1, deserializers[1](reader) as T2);
        }
      }
    }
  }
}
```

以上概念就是此方法的主要邏輯,接著講其他細節部分

#### 支持多組類別 + 強型別返回值

Dapper為了`強型別多類別Mapping`使用`多組泛型參數方法`方式,這方式有個小缺點就是`沒辦法動態調整`,需要以寫死方式來處理。 

舉例,可以看到圖片GenerateMapper方法,依照泛型參數數量,寫死強轉型邏輯,這也是為何Multiple Query有最大組數限制,只能支持最多6組的原因。   
![20191001173320.png](https://i.loli.net/2019/10/01/3BKcPCzdnMeGRV6.png)

#### 多類別泛型緩存算法
- 這邊Dapper使用`泛型類別`來`強型別`保存多類別的資料
![20191001175139.png](https://i.loli.net/2019/10/01/5eSfkoaQiIPXvz4.png)
- 並配合繼承共用Identity大部分身分驗證邏輯
- 提供可`override`的GetType方法,來客製泛型比較邏輯,避免造成跟Non Multi Query`緩存衝突`。

![20191001175600.png](https://i.loli.net/2019/10/01/ypEj12lzDFVsMuo.png)
![20191001175707.png](https://i.loli.net/2019/10/01/nIbHarVcyNxfpqj.png)


#### Dapper Query Multi Mapping的Select順序很重要
因為SplitOn`分組基礎依賴於Select的順序`,所以順序一錯就有可能`屬性值錯亂`情況。

舉例 : 假如上面例子的SQL改成以下,會發生User的ID變成Order的ID;Order的ID會變成User的ID。
```sql
select T2.[ID],T1.[OrderNo],T1.[UserID],T1.[ID],T2.[Name] from [order] T1
left join [User] T2 on T1.UserId = T2.ID  
```

原因可以追究到Dapper的切割算法
1. 首先`倒序`方式處理欄位分組(GetNextSplit方法可以看到從DataReader Index`大到小`查詢)
![20191002022109.png](https://i.loli.net/2019/10/02/p6mzVJEGZtL84C7.png)
2. 接著`倒序`方式處理類別的Mapping Emit IL Func
3. 最後反轉為`正序`,方便後面Call Func對應泛型使用
![20191002021750.png](https://i.loli.net/2019/10/02/RjEizNoSFBgHODr.png)
![20191002022208.png](https://i.loli.net/2019/10/02/Qu1MENwDGWi9cCS.png)
![20191002022214.png](https://i.loli.net/2019/10/02/IBWfdvzLwbqX1GP.png)


 　 
 　 
## 15.QueryMultiple 底層原理  <a name="page15"></a>


使用方式例子 : 
```C#
  using (var cn = Connection)
  {
    using (var gridReader = cn.QueryMultiple("select 1; select 2;"))
    {
      Console.WriteLine(gridReader.Read<int>()); //result : 1
      Console.WriteLine(gridReader.Read<int>()); //result : 2
    }
  }
```

使用QueryMultiple優點 : 
- 主要`減少Reqeust次數`
- 可以將多個查詢`共用同一組Parameter參數`

QueryMultiple的底層實作邏輯 : 
1. 底層技術是ADO.NET - DataReader - MultipleResult
2. QueryMultiple取得DataReader並封裝進GridReader
3. 呼叫Read方法時才會建立Mapping動態方法,Emit IL動作跟Query方法一樣
4. 接著使用ADO.NET技術呼叫`DataReader NextResult`取得下一組查詢結果
5. 假如`沒有`下一組查詢結果才會將`DataReader釋放`

----


#### 緩存算法
緩存的算法多增加gridIndex判斷,主要對每個result mapping動作做一個緩存,Emit IL的邏輯跟Query一樣。

![20190930183038.png](https://i.loli.net/2019/09/30/b1aKNx2qQ5lSufF.png)

#### 沒有延遲查詢特性
注意Read方法使用的是buffer = true = 返回結果直接ToList保存在內存,所以沒有延遲查詢特性。

![20190930183212.png](https://i.loli.net/2019/09/30/mIVDex8rqk5Z46E.png)
![20190930183219.png](https://i.loli.net/2019/09/30/CdqjTZyI1BnuDzi.png)

#### 記得管理DataReader的釋放
Dapper 呼叫QueryMultiple方法時會將DataReader封裝在GridReader物件內,只有當`最後一次Read`動作後才會回收DataReader

![20190930183447.png](https://i.loli.net/2019/09/30/eB9Z5dgCtF87PRI.png)

所以`沒有讀取完`再開一個GridReader > Read會出現錯誤:`已經開啟一個與這個 Command 相關的 DataReader，必須先將它關閉`。

![20190930183532.png](https://i.loli.net/2019/09/30/ByrTuOf6jzD7w21.png)

要避免以上情況,可以改成`using`區塊方式,運行完區塊代碼後就會自動釋放DataReader
```C#
using (var gridReader = cn.QueryMultiple("select 1; select 2;"))
{
  //略..
}
```

#### 閒話 : 
感覺Dapper GridReader好像有機會可以實作`是否有NextResult`方法,這樣就可以`配合while`方法`一次讀取完多組查詢資料`,等之後有空來想想有沒有機會做成。

概念代碼 : 
```C#
public static class DbExtension
{
  public static IEnumerable<IEnumerable<dynamic>> GetMultipleResult(this IDbConnection cn,string sql, object paramters)
  {
    using (var reader = cn.QueryMultiple(sql,paramters))
    {
      while(reader.NextResult())
      {
        yield return reader.Read();
      }
    }
  }
}
```

 　 
 　 
## 16.TypeHandler 自訂Mapping邏輯使用、底層邏輯  <a name="page16"></a>


遇到想要客製某些屬性Mapping邏輯時,在Dapper可以使用`TypeHandler`

使用方式 : 
- 建立類別繼承`SqlMapper.TypeHandler`
- 將要客製的類別指定給`泛型`,e.g : `JsonTypeHandler<客製類別> : SqlMapper.TypeHandler<客製類別> `
- `查詢`的邏輯使用override實作`Parse`方法,`增刪改`邏輯實作`SetValue`方法
- 假如多個類別Parse、SetValue共用同樣邏輯,可以將實作類別改為`泛型`方式,客製類別在`AddTypeHandler`時指定就可以,可以避免建立一堆類別,e.g : `JsonTypeHandler<T> : SqlMapper.TypeHandler<T> where T : class`

舉例 : 
想要特定屬性成員在數據庫保存Json,在AP端自動轉成對應Class類別,這時候可以使用`SqlMapper.AddTypeHandler<繼承實作TypeHandler的類別>`。

以下例子是User資料變更時會自動在Log欄位紀錄變更動作。
```C#
public class JsonTypeHandler<T> : SqlMapper.TypeHandler<T> 
  where T : class
{
  public override T Parse(object value)
  {
    return JsonConvert.DeserializeObject<T>((string)value);
  }

  public override void SetValue(IDbDataParameter parameter, T value)
  {
    parameter.Value = JsonConvert.SerializeObject(value);
  }
}

public void Main()
{
  SqlMapper.AddTypeHandler(new JsonTypeHandler<List<Log>>()); 

  using (var ts = new TransactionScope())
  using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
  {

    cn.Execute("create table [User] (Name nvarchar(200),Age int,Level int,Logs nvarchar(max))");

    var user = new User()
    {
      Name = "暐翰",
      Age = 26,
      Level = 1,
      Logs = new List<Log>() {
        new Log(){Time=DateTime.Now,Remark="CreateUser"}
      }
    };

    //新增資料
    {
      cn.Execute("insert into [User] (Name,Age,Level,Logs) values (@Name,@Age,@Level,@Logs);", user);

      var result = cn.Query("select * from [User]");
      Console.WriteLine(result);
    }

    //升級Level動作
    {
      user.Level = 9;
      user.Logs.Add(new Log() {Remark="UpdateLevel"});
      cn.Execute("update [User] set Level = @Level,Logs = @Logs where Name = @Name", user);
      var result = cn.Query("select * from [User]");
      Console.WriteLine(result);
    }

    ts.Dispose();

  }
}

public class User
{
  public string Name { get; set; }
  public int Age { get; set; }
  public int Level { get; set; }
  public List<Log> Logs { get; set; }

}
public class Log
{
  public DateTime Time { get; set; } = DateTime.Now;
  public string Remark { get; set; }
}
```

效果圖 : 
![20190929231937.png](https://i.loli.net/2019/09/29/F7xWKHPaSGuo3kB.png)

----


接著追蹤TypeHandler源碼邏輯,需要分兩個部份來追蹤 : SetValue,Parse

### SetValue底層原理
1. AddTypeHandlerImpl方法管理緩存的添加
2. 在CreateParamInfoGenerator方法Emit建立動態AddParameter方法時,假如該Mapping類別TypeHandler緩存內有資料,Emit添加呼叫SetValue方法動作。
```C#
if (handler != null)
{
  il.Emit(OpCodes.Call, typeof(TypeHandlerCache<>).MakeGenericType(prop.PropertyType).GetMethod(nameof(TypeHandlerCache<int>.SetValue))); // stack is now [parameters] [[parameters]] [parameter]
}
```
3. 在Runtime呼叫AddParameters方法時會使用LookupDbType,判斷是否有自訂TypeHandler
![20191006151723.png](https://i.loli.net/2019/10/06/Jq5gomaEnb7kRTO.png)
![20191006151614.png](https://i.loli.net/2019/10/06/kjlbsXBWAFoJ72I.png)
4. 接著將建立好的Parameter傳給自訂TypeHandler.SetValue方法
![20191006151901.png](https://i.loli.net/2019/10/06/xehnlaP4NDJ6wLg.png)

最後查看IL轉成的C#代碼
```C#
    public static void TestMeThod(IDbCommand P_0, object P_1)
    {
        User user = (User)P_1;
        IDataParameterCollection parameters = P_0.Parameters;
        //略...
        IDbDataParameter dbDataParameter3 = P_0.CreateParameter();
        dbDataParameter3.ParameterName = "Logs";
        dbDataParameter3.Direction = ParameterDirection.Input;
        SqlMapper.TypeHandlerCache<List<Log>>.SetValue(dbDataParameter3, ((object)user.Logs) ?? ((object)DBNull.Value));
        parameters.Add(dbDataParameter3);
        //略...
    }
```

可以發現生成的Emit IL會去從TypeHandlerCache取得我們實作的TypeHandler,接著`呼叫實作SetValue方法`運行設定的邏輯,並且TypeHandlerCache特別使用`泛型類別`依照不同泛型以`Singleton`方式保存不同handler,這樣有以下優點 : 
1. 只要傳遞泛型類別參數就可以取得同一個handler`避免重複建立物件`
2. 因為是泛型類別,取handler時可以避免了反射動作,`提升效率`

![https://ithelp.ithome.com.tw/upload/images/20190929/20105988x970H6xWXC.png](https://ithelp.ithome.com.tw/upload/images/20190929/20105988x970H6xWXC.png)
![https://ithelp.ithome.com.tw/upload/images/20190929/20105988S7VZLLXLZo.png](https://ithelp.ithome.com.tw/upload/images/20190929/20105988S7VZLLXLZo.png)
![https://ithelp.ithome.com.tw/upload/images/20190929/20105988Q1mWkL0GP6.png](https://ithelp.ithome.com.tw/upload/images/20190929/20105988Q1mWkL0GP6.png)


----

### Parse對應底層原理

主要邏輯是在GenerateDeserializerFromMap方法Emit建立動態Mapping方法時,假如判斷TypeHandler緩存有資料,以Parse方法取代原本的Set屬性動作。
![https://ithelp.ithome.com.tw/upload/images/20190930/20105988JvCw5z207s.png](https://ithelp.ithome.com.tw/upload/images/20190930/20105988JvCw5z207s.png)

查看動態Mapping方法生成的IL代碼 : 
```
IL_0000: ldc.i4.0   
IL_0001: stloc.0    
IL_0002: newobj     Void .ctor()/Demo.User
IL_0007: stloc.1    
IL_0008: ldloc.1    
IL_0009: dup        
IL_000a: ldc.i4.0   
IL_000b: stloc.0    
IL_000c: ldarg.0    
IL_000d: ldc.i4.0   
IL_000e: callvirt   System.Object get_Item(Int32)/System.Data.IDataRecord
IL_0013: dup        
IL_0014: stloc.2    
IL_0015: dup        
IL_0016: isinst     System.DBNull
IL_001b: brtrue.s   IL_0029
IL_001d: unbox.any  System.String
IL_0022: callvirt   Void set_Name(System.String)/Demo.User
IL_0027: br.s       IL_002b
IL_0029: pop        
IL_002a: pop        
IL_002b: dup        
IL_002c: ldc.i4.1   
IL_002d: stloc.0    
IL_002e: ldarg.0    
IL_002f: ldc.i4.1   
IL_0030: callvirt   System.Object get_Item(Int32)/System.Data.IDataRecord
IL_0035: dup        
IL_0036: stloc.2    
IL_0037: dup        
IL_0038: isinst     System.DBNull
IL_003d: brtrue.s   IL_004b
IL_003f: unbox.any  System.Int32
IL_0044: callvirt   Void set_Age(Int32)/Demo.User
IL_0049: br.s       IL_004d
IL_004b: pop        
IL_004c: pop        
IL_004d: dup        
IL_004e: ldc.i4.2   
IL_004f: stloc.0    
IL_0050: ldarg.0    
IL_0051: ldc.i4.2   
IL_0052: callvirt   System.Object get_Item(Int32)/System.Data.IDataRecord
IL_0057: dup        
IL_0058: stloc.2    
IL_0059: dup        
IL_005a: isinst     System.DBNull
IL_005f: brtrue.s   IL_006d
IL_0061: unbox.any  System.Int32
IL_0066: callvirt   Void set_Level(Int32)/Demo.User
IL_006b: br.s       IL_006f
IL_006d: pop        
IL_006e: pop        
IL_006f: dup        
IL_0070: ldc.i4.3   
IL_0071: stloc.0    
IL_0072: ldarg.0    
IL_0073: ldc.i4.3   
IL_0074: callvirt   System.Object get_Item(Int32)/System.Data.IDataRecord
IL_0079: dup        
IL_007a: stloc.2    
IL_007b: dup        
IL_007c: isinst     System.DBNull
IL_0081: brtrue.s   IL_008f
IL_0083: call       System.Collections.Generic.List`1[Demo.Log] Parse(System.Object)/Dapper.SqlMapper+TypeHandlerCache`1[System.Collections.Generic.List`1[Demo.Log]]
IL_0088: callvirt   Void set_Logs(System.Collections.Generic.List`1[Demo.Log])/Demo.User
IL_008d: br.s       IL_0091
IL_008f: pop        
IL_0090: pop        
IL_0091: stloc.1    
IL_0092: leave      IL_00a4
IL_0097: ldloc.0    
IL_0098: ldarg.0    
IL_0099: ldloc.2    
IL_009a: call       Void ThrowDataException(System.Exception, Int32, System.Data.IDataReader, System.Object)/Dapper.SqlMapper
IL_009f: leave      IL_00a4
IL_00a4: ldloc.1    
IL_00a5: ret        
```

轉成C#代碼來驗證 : 
```C#
  public static User TestMeThod(IDataReader P_0)
  {
    int index = 0;
    User user = new User();
    object value = default(object);
    try
    {
      User user2 = user;
      index = 0;
      object obj = value = P_0[0];
      //..略
      index = 3;
      object obj4 = value = P_0[3];
      if (!(obj4 is DBNull))
      {
        user2.Logs = SqlMapper.TypeHandlerCache<List<Log>>.Parse(obj4);
      }
      user = user2;
      return user;
    }
    catch (Exception ex)
    {
      SqlMapper.ThrowDataException(ex, index, P_0, value);
      return user;
    }
  }
```

 　 
 　 
## 17. CommandBehavior的細節處理  <a name="page17"></a>


這篇將帶讀者了解Dapper如何在底層利用CommandBehavior優化查詢效率,如何選擇正確Behavior在特定時機。

我這邊整理了各方法對應的Behavior表格 : 

|方法|Behavior|
|-|-|
|Query | CommandBehavior.SequentialAccess & CommandBehavior.SingleResult |
|QueryFirst | CommandBehavior.SequentialAccess & CommandBehavior.SingleResult & CommandBehavior.SingleRow |
|QueryFirstOrDefault | CommandBehavior.SequentialAccess & CommandBehavior.SingleResult & CommandBehavior.SingleRow |
|QuerySingle | CommandBehavior.SingleResult & CommandBehavior.SequentialAccess |
|QuerySingleOrDefault | CommandBehavior.SingleResult & CommandBehavior.SequentialAccess |
|QueryMultiple|CommandBehavior.SequentialAccess|

---

#### SequentialAccess、SingleResult優化邏輯

首先可以看到每個方法都使用`CommandBehavior.SequentialAccess`,該標籤主要功能 `使DataReader順序讀取行和列,行和列不緩衝,讀取一列後,它會從內存中刪除。`,有以下優點 :
1. 可按順序分次讀取資源,`避免二進制大資源一次性讀取到內存`,尤其是Blob或是Clob會配合GetBytes 或 GetChars 方法限制緩衝區大小,微軟官方也特別標註注意 : 
![20191003014421.png](https://i.loli.net/2019/10/03/PVSuO8xwfhzgaFH.png)
1. 實際環境測試,可以`加快查詢效率`

但它卻`不是`DataReader的預設行為,系統預設是`CommandBehavior.Default`
![20191003015853.png](https://i.loli.net/2019/10/03/23Zs87l9aGCnR4E.png)
CommandBehavior.Default有著以下特性 : 
1. 可傳回`多個`結果集(Multi Result)
2. 一次性讀取行資料到內存

這兩個特性跟生產環境情況差滿多,畢竟大多時刻是`只需要一組結果集配合有限的內存`,所以除了SequentialAccess外Dapper還特別在大多方法使用了`CommandBehavior.SingleResult`,滿足只需一組結果就好避免浪費資源。

這段還有一段細節的處理,查看源碼可以發現除了標記SingleResult外,Dapper還特別加上一段代碼在結尾`while (reader.NextResult()){}`,而不是直接Return(如圖片)

![20191003021109.png](https://i.loli.net/2019/10/03/2VSKACmruvobanI.png)

早些前我有特別發Issue([連結#1210](https://github.com/StackExchange/Dapper/issues/1210))詢問過作者,這邊是回答 : `主要避免忽略錯誤,像是在DataReader提早關閉情況` 

---

#### QueryFirst搭配SingleRow,

有時候我們會遇到`select top 1`知道只會讀取一行資料的情況,這時候可以使用`QueryFirst`。它使用`CommandBehavior.SingleRow`可以避免浪費資源只讀取一行資料。

另外可以發現此段除了`while (reader.NextResult()){}`外還有`while (reader.Read()) {}`,同樣是避免忽略錯誤,這是一些公司自行土炮ORM會忽略的地方。
![20191003024206.png](https://i.loli.net/2019/10/03/PRsrNoeJOInKB8u.png)

### 與QuerySingle之間的差別
兩者差別在QuerySingle沒有使用CommandBehavior.SingleRow,至於為何沒有使用,是因為需要有`多行資料才能判斷是否不符合條件並拋出Exception告知使用者`。

這段有一個特別好玩小技巧可以學,錯誤處理直接沿用對應LINQ的Exception,舉例:超過一行資料錯誤,使用`new int[2].Single()`,這樣不用另外維護Exceptiono類別,還可以擁有i18N多國語言化。
![20191003025631.png](https://i.loli.net/2019/10/03/S38mD9LElTgUYwd.png)
![20191003025334.png](https://i.loli.net/2019/10/03/9vMXH8D1muWdYzQ.png)



 　 
 　 
## 18.Parameter 參數化底層原理  <a name="page18"></a>



接著進到Dapper的另一個關鍵功能 : 「Parameter 參數化」

主要邏輯 : 
GetCacheInfo檢查是否緩存內有動態方法 > 假如沒有緩存,使用CreateParamInfoGenerator方法Emit IL建立AddParameter動態方法 > 建立完後保存在緩存內

接著重點來看CreateParamInfoGenerator方法內的底成邏輯跟「精美細節處理」,使用了結果反推代碼方法,忽略`「沒使用的欄位」`不生成對應IL代碼,避免資源浪費情況。這也是前面緩存算法要去判斷不同SQL字串的原因。

以下是我挑出的源碼重點部分 : 
```C#
internal static Action<IDbCommand, object> CreateParamInfoGenerator(Identity identity, bool checkForDuplicates, bool removeUnused, IList<LiteralToken> literals)
{
  //...略
  if (filterParams)
  {
    props = FilterParameters(props, identity.sql);
  }

  var callOpCode = isStruct ? OpCodes.Call : OpCodes.Callvirt;
  foreach (var prop in props)
  {
    //Emit IL動作
  }
  //...略
}


private static IEnumerable<PropertyInfo> FilterParameters(IEnumerable<PropertyInfo> parameters, string sql)
{
  var list = new List<PropertyInfo>(16);
  foreach (var p in parameters)
  {
    if (Regex.IsMatch(sql, @"[?@:]" + p.Name + @"([^\p{L}\p{N}_]+|$)", RegexOptions.IgnoreCase | RegexOptions.Multiline | RegexOptions.CultureInvariant))
      list.Add(p);
  }
  return list;
}
```

接著查看IL來驗證,查詢代碼如下
```C#
var result = connection.Query("select @Name name ", new { Name = "暐翰", Age = 26}).First();
```

CreateParamInfoGenerator AddParameter 動態方法IL代碼如下 : 
```C#
IL_0000: ldarg.1    
IL_0001: castclass  <>f__AnonymousType1`2[System.String,System.Int32]
IL_0006: stloc.0    
IL_0007: ldarg.0    
IL_0008: callvirt   System.Data.IDataParameterCollection get_Parameters()/System.Data.IDbCommand
IL_000d: dup        
IL_000e: ldarg.0    
IL_000f: callvirt   System.Data.IDbDataParameter CreateParameter()/System.Data.IDbCommand
IL_0014: dup        
IL_0015: ldstr      "Name"
IL_001a: callvirt   Void set_ParameterName(System.String)/System.Data.IDataParameter
IL_001f: dup        
IL_0020: ldc.i4.s   16
IL_0022: callvirt   Void set_DbType(System.Data.DbType)/System.Data.IDataParameter
IL_0027: dup        
IL_0028: ldc.i4.1   
IL_0029: callvirt   Void set_Direction(System.Data.ParameterDirection)/System.Data.IDataParameter
IL_002e: dup        
IL_002f: ldloc.0    
IL_0030: callvirt   System.String get_Name()/<>f__AnonymousType1`2[System.String,System.Int32]
IL_0035: dup        
IL_0036: brtrue.s   IL_0042
IL_0038: pop        
IL_0039: ldsfld     System.DBNull Value/System.DBNull
IL_003e: ldc.i4.0   
IL_003f: stloc.1    
IL_0040: br.s       IL_005a
IL_0042: dup        
IL_0043: callvirt   Int32 get_Length()/System.String
IL_0048: ldc.i4     4000
IL_004d: cgt        
IL_004f: brtrue.s   IL_0058
IL_0051: ldc.i4     4000
IL_0056: br.s       IL_0059
IL_0058: ldc.i4.m1  
IL_0059: stloc.1    
IL_005a: callvirt   Void set_Value(System.Object)/System.Data.IDataParameter
IL_005f: ldloc.1    
IL_0060: brfalse.s  IL_0069
IL_0062: dup        
IL_0063: ldloc.1    
IL_0064: callvirt   Void set_Size(Int32)/System.Data.IDbDataParameter
IL_0069: callvirt   Int32 Add(System.Object)/System.Collections.IList
IL_006e: pop        
IL_006f: pop        
IL_0070: ret            
```

IL轉成對應C#代碼: 
```C#
public class TestType
{
  public static void TestMeThod(IDataReader P_0, object P_1)
  {
    var anon = (<>f__AnonymousType1<string, int>)P_1;
    IDataParameterCollection parameters = ((IDbCommand)P_0).Parameters;
    IDbDataParameter dbDataParameter = ((IDbCommand)P_0).CreateParameter();
    dbDataParameter.ParameterName = "Name";
    dbDataParameter.DbType = DbType.String;
    dbDataParameter.Direction = ParameterDirection.Input;
    object obj = anon.Name;
    int num;
    if (obj == null)
    {
      obj = DBNull.Value;
      num = 0;
    }
    else
    {
      num = ((((string)obj).Length > 4000) ? (-1) : 4000);
    }
    dbDataParameter.Value = obj;
    if (num != 0)
    {
      dbDataParameter.Size = num;
    }
    parameters.Add(dbDataParameter);
  }
}
```

可以發現雖然傳遞Age參數,但是SQL字串沒有用到,Dapper不會去生成該欄位的SetParameter動作IL。這個細節處理真的要給Dapper一個讚!

 　 
 　 
## 19. IN 多集合參數化底層原理  <a name="page19"></a>



#### 為何ADO.NET不支援`IN 參數化`,Dapper支援 ?
原理
1. 判斷參數的屬性是否為IEnumerable類別子類別 
2. 假如是,以該參數名稱為主 + Parameter正則格式找尋SQL內的參數字串 (正則格式 : `([?@:]參數名)(?!\w)(\s+(?i)unknown(?-i))?`)
3. 將找到的字串以`()` + 多個`屬性名稱+流水號`方式替換
4. 依照流水號順序依序CreateParameter > SetValue 

關鍵程式部分
![https://ithelp.ithome.com.tw/upload/images/20190925/20105988ouMJ6GRB7F.png](https://ithelp.ithome.com.tw/upload/images/20190925/20105988ouMJ6GRB7F.png)

以下用sys.objects來查SQL Server的表格跟視圖當追蹤例子 : 
```C#
var result = cn.Query(@"select * from sys.objects where type_desc In @type_descs", new { type_descs = new[] { "USER_TABLE", "VIEW" } });
```

Dapper會將SQL字串改成以下方式執行
```C#
select * from sys.objects where type_desc In (@type_descs1,@type_descs2)
-- @type_descs1 = nvarchar(4000) - 'USER_TABLE'
-- @type_descs2 = nvarchar(4000) - 'VIEW'
```

查看Emit IL可以發現跟之前的參數化IL很不一樣,非常的簡短
```C#
IL_0000: ldarg.1    
IL_0001: castclass  <>f__AnonymousType0`1[System.String[]]
IL_0006: stloc.0    
IL_0007: ldarg.0    
IL_0008: callvirt   System.Data.IDataParameterCollection get_Parameters()/System.Data.IDbCommand
IL_000d: ldarg.0    
IL_000e: ldstr      "type_descs"
IL_0013: ldloc.0    
IL_0014: callvirt   System.String[] get_type_descs()/<>f__AnonymousType0`1[System.String[]]
IL_0019: call       Void PackListParameters(System.Data.IDbCommand, System.String, System.Object)/Dapper.SqlMapper
IL_001e: pop        
IL_001f: ret        
```

轉成C#代碼來看,會很驚訝地發現:`「這段根本不需要使用Emit IL簡直多此一舉」`
```C#
    public static void TestMeThod(IDbCommand P_0, object P_1)
    {
        var anon = (<>f__AnonymousType0<string[]>)P_1;
        IDataParameterCollection parameter = P_0.Parameters;
        SqlMapper.PackListParameters(P_0, "type_descs", anon.type_descs);
    }
```

沒錯,是多此一舉,甚至` IDataParameterCollection parameter = P_0.Parameters;`這段代碼根本不會用到。

Dapper這邊做法是有原因的,因為要`能跟非集合參數配合使用`,像是前面例子加上找出訂單Orders名稱的資料邏輯 
```C#
var result = cn.Query(@"select * from sys.objects where type_desc In @type_descs and name like @name"
    , new { type_descs = new[] { "USER_TABLE", "VIEW" }, @name = "order%" });
```
對應生成的IL轉換C#代碼就會是以下代碼,達到能搭配使用目的 : 
```C#
    public static void TestMeThod(IDbCommand P_0, object P_1)
    {
        <>f__AnonymousType0<string[], string> val = P_1;
        IDataParameterCollection parameters = P_0.Parameters;
        SqlMapper.PackListParameters(P_0, "type_descs", val.get_type_descs());
        IDbDataParameter dbDataParameter = P_0.CreateParameter();
        dbDataParameter.ParameterName = "name";
        dbDataParameter.DbType = DbType.String;
        dbDataParameter.Direction = ParameterDirection.Input;
        object obj = val.get_name();
        int num;
        if (obj == null)
        {
            obj = DBNull.Value;
            num = 0;
        }
        else
        {
            num = ((((string)obj).Length > 4000) ? (-1) : 4000);
        }
        dbDataParameter.Value = obj;
        if (num != 0)
        {
            dbDataParameter.Size = num;
        }
        parameters.Add(dbDataParameter);
    }
```

另外為何Dapper這邊Emit IL會直接呼叫工具方法`PackListParameters`,是因為IN的`參數化數量是不固定`,所以`不能由固定結果反推程式碼`方式動態生成方法。

該方法裡面包含的主要邏輯:
1. 判斷集合參數的類型是哪一種 (假如是字串預設使用4000大小)
2. 正則判斷SQL參數以流水號參數字串取代
3. DbCommand的Paramter的創建

![https://ithelp.ithome.com.tw/upload/images/20190925/20105988KgYZmlciZJ.png](https://ithelp.ithome.com.tw/upload/images/20190925/20105988KgYZmlciZJ.png)
SQL參數字串的取代邏輯也寫在這邊,如圖片
![https://ithelp.ithome.com.tw/upload/images/20190925/20105988Rhner7LZPA.png](https://ithelp.ithome.com.tw/upload/images/20190925/20105988Rhner7LZPA.png)



 　 
 　 
## 20.DynamicParameter 底層原理、自訂實作  <a name="page20"></a>


這邊用個例子帶讀者了解DynamicParameter原理,舉例現在有一段代碼如下 : 
```C#
using (var cn = Connection)
{
    var paramter = new { Name = "John", Age = 25 };
    var result = cn.Query("select @Name Name,@Age Age", paramter).First();
}
```

前面已經知道String型態Dapper會自動將轉成數據庫`Nvarchar`並且`長度為4000`的參數,數據庫實際執行的SQL如下 : 
```sql
exec sp_executesql N'select @Name Name,@Age Age',N'@Name nvarchar(4000),@Age int',@Name=N'John',@Age=25
```

這是一個方便快速開發的貼心設計,但假如遇到欄位是`varchar`型態的情況,有可能會因為隱性轉型導致`索引失效`,導致查詢效率變低。

這時解決方式可以使用Dapper DynamicParamter指定數據庫型態跟大小,達到優化效能目的
```C#
using (var cn = Connection)
{
    var paramters = new DynamicParameters();
    paramters.Add("Name","John",DbType.AnsiString,size:4);
    paramters.Add("Age",25,DbType.Int32);
    var result = cn.Query("select @Name Name,@Age Age", paramters).First();
}
```

----


接著往底層來看如何實現,首先關注GetCacheInfo方法,可以看到DynamicParameters建立動態方法方式代碼很簡單,就只是呼叫AddParameters方法
```C#
Action<IDbCommand, object> reader;
if (exampleParameters is IDynamicParameters)
{
    reader = (cmd, obj) => ((IDynamicParameters)obj).AddParameters(cmd, identity);
}
```

代碼可以這麼簡單的原因,是Dapper在這邊特別使用`「依賴於介面」`設計,增加`程式的彈性`,讓使用者可以客制自己想要的實作邏輯。這點下面會講解,首先來看Dapper預設的實作類別`DynamicParameters`中`AddParameters`方法的實作邏輯

```C#
public class DynamicParameters : SqlMapper.IDynamicParameters, SqlMapper.IParameterLookup, SqlMapper.IParameterCallbacks
{
    protected void AddParameters(IDbCommand command, SqlMapper.Identity identity)
    {
        var literals = SqlMapper.GetLiteralTokens(identity.sql);

        foreach (var param in parameters.Values)
        {
            if (param.CameFromTemplate) continue;

            var dbType = param.DbType;
            var val = param.Value;
            string name = Clean(param.Name);
            var isCustomQueryParameter = val is SqlMapper.ICustomQueryParameter;

            SqlMapper.ITypeHandler handler = null;
            if (dbType == null && val != null && !isCustomQueryParameter)
            {
#pragma warning disable 618
                dbType = SqlMapper.LookupDbType(val.GetType(), name, true, out handler);
#pragma warning disable 618
            }
            if (isCustomQueryParameter)
            {
                ((SqlMapper.ICustomQueryParameter)val).AddParameter(command, name);
            }
            else if (dbType == EnumerableMultiParameter)
            {
#pragma warning disable 612, 618
                SqlMapper.PackListParameters(command, name, val);
#pragma warning restore 612, 618
            }
            else
            {
                bool add = !command.Parameters.Contains(name);
                IDbDataParameter p;
                if (add)
                {
                    p = command.CreateParameter();
                    p.ParameterName = name;
                }
                else
                {
                    p = (IDbDataParameter)command.Parameters[name];
                }

                p.Direction = param.ParameterDirection;
                if (handler == null)
                {
#pragma warning disable 0618
                    p.Value = SqlMapper.SanitizeParameterValue(val);
#pragma warning restore 0618
                    if (dbType != null && p.DbType != dbType)
                    {
                        p.DbType = dbType.Value;
                    }
                    var s = val as string;
                    if (s?.Length <= DbString.DefaultLength)
                    {
                        p.Size = DbString.DefaultLength;
                    }
                    if (param.Size != null) p.Size = param.Size.Value;
                    if (param.Precision != null) p.Precision = param.Precision.Value;
                    if (param.Scale != null) p.Scale = param.Scale.Value;
                }
                else
                {
                    if (dbType != null) p.DbType = dbType.Value;
                    if (param.Size != null) p.Size = param.Size.Value;
                    if (param.Precision != null) p.Precision = param.Precision.Value;
                    if (param.Scale != null) p.Scale = param.Scale.Value;
                    handler.SetValue(p, val ?? DBNull.Value);
                }

                if (add)
                {
                    command.Parameters.Add(p);
                }
                param.AttachedParam = p;
            }
        }

        // note: most non-priveleged implementations would use: this.ReplaceLiterals(command);
        if (literals.Count != 0) SqlMapper.ReplaceLiterals(this, command, literals);
    }
}
```

可以發現Dapper在AddParameters為了方便性跟兼容其他功能,像是Literal Replacement、EnumerableMultiParameter功能,做了許多判斷跟動作,所以代碼量會比以前使用ADO.NET版本多,所以效率也會比較慢。

假如有效率苛求的需求,可以自己實作想要的邏輯,因為Dapper此段特別設計成`「依賴於介面」`,只需要實作`IDynamicParameters`介面就可以。

以下是我做的一個Demo,可以使用ADO.NET SqlParameter建立參數跟Dapper配合
```C#
public class CustomPraameters : SqlMapper.IDynamicParameters
{
  private SqlParameter[] parameters;
  public void Add(params SqlParameter[] mParameters)
  {
    parameters = mParameters;
  }

  void SqlMapper.IDynamicParameters.AddParameters(IDbCommand command, SqlMapper.Identity identity)
  {
    if (parameters != null && parameters.Length > 0)
      foreach (var p in parameters)
        command.Parameters.Add(p);
  }
}
```

![https://ithelp.ithome.com.tw/upload/images/20191005/20105988qzCAsa5KZu.png](https://ithelp.ithome.com.tw/upload/images/20191005/20105988qzCAsa5KZu.png)

 　 
 　 
## 21. 單次、多次 Execute 底層原理  <a name="page21"></a>


查詢、Mapping、參數講解完後,接著講解在`增、刪、改`情況Dapper我們會使用Execute方法,其中Execute Dapper分為`單次執行、多次執行`。

#### 單次Execute
以單次執行來說Dapper Execute底層是ADO.NET的ExecuteNonQuery的封裝,封裝目的為了跟Dapper的`Parameter、緩存`功能搭配使用,代碼邏輯簡潔明瞭這邊就不做多說明,如圖片
![20191002144453.png](https://i.loli.net/2019/10/02/DPUYt5mLzFvoaQw.png)


#### 「多次」Execute

這是Dapper一個特色功能,它簡化了集合操作Execute之間的操作,簡化了代碼,只需要 : `connection.Execute("sql",集合參數);`。

至於為何可以這麼方便,以下是底層的邏輯 :
1. 確認是否為集合參數  
![20191002150155.png](https://i.loli.net/2019/10/02/alGyqdNxSkg3EWz.png)
2. 建立`一個共同DbCommand`提供foreach迭代使用,避免重複建立浪費資源
![20191002151237.png](https://i.loli.net/2019/10/02/HfTID8jpE1nZyXF.png)
3. 假如是集合參數,建立Emit IL動態方法,並放在緩存內利用
![20191002150349.png](https://i.loli.net/2019/10/02/s1aNBUEfhnFcMbw.png)
4. 動態方法邏輯是`CreateParameter > 對Parameter賦值 > 使用Parameters.Add添加新建的參數`,以下是Emit IL轉成的C#代碼 : 
```C#
  public static void ParamReader(IDbCommand P_0, object P_1)
  {
    var anon = (<>f__AnonymousType0<int>)P_1;
    IDataParameterCollection parameters = P_0.Parameters;
    IDbDataParameter dbDataParameter = P_0.CreateParameter();
    dbDataParameter.ParameterName = "V";
    dbDataParameter.DbType = DbType.Int32;
    dbDataParameter.Direction = ParameterDirection.Input;
    dbDataParameter.Value = anon.V;
    parameters.Add(dbDataParameter);
  }
```
5. `foreach`該集合參數 > 除了第一次外,每次迭代清空DbCommand的Parameters > 重新呼叫`同一個`動態方法添加Parameter > 送出SQL查詢

---

實作方式簡潔明瞭,並且細節考慮共用資源避免浪費(e.g`共用同一個DbCommand、Func`),但遇到大量執行追求效率需求情況,需要特別注意此方法`每跑一次對數據庫送出一次reqesut`,效率會被網路傳輸拖慢,所以這功能被稱為`「多次執行」而不是「批量執行」`的主要原因。

舉例,簡單Execute插入十筆資料,查看SQL Profiler可以看到系統接到10次Reqeust:
```C#
using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=Northwind;"))
{
    cn.Open();
    using (var tx = cn.BeginTransaction())
    {
        cn.Execute("create table #T (V int);", transaction: tx);
        cn.Execute("insert into #T (V) values (@V)", Enumerable.Range(1, 10).Select(val => new { V = val }).ToArray() , transaction:tx);

        var result = cn.Query("select * from #T", transaction: tx);
        Console.WriteLine(result);
    }
}
```

![20191002151658.png](https://i.loli.net/2019/10/02/WZ5w1gMFG2ESzDm.png)


 　 
 　 
## 22. ExecuteScalar應用 <a name="page22"></a>


ExecuteScalar因為其`只能讀取第一組結果、第一筆列、第一筆資料`特性,是一個常被遺忘的功能,但它在特定需求下還是能派上用場,底下用「查詢資料是否存在」例子來做說明。

### 首先,Entity Framwork如何高效率判斷資料是否存在?

假如有EF經驗的讀者會答使用`Any`而不是`Count() > 1`。

使用Count系統會幫轉換SQL為 : 
```sql
SELECT COUNT(*) AS [value] FROM [表格] AS [t0]
```
SQL Count 是一個匯總函數,會迭代符合條件的資料行`判斷每列該資料是否為null`,並返回其行數。

而Any語法轉換SQL使用`EXISTS`,它只在乎`是否有沒有資料`,代表`不用檢查到每列`,只需要其中一筆有資料就有結果,所以效率快。
```sql
SELECT 
    (CASE 
        WHEN EXISTS(
            SELECT NULL AS [EMPTY]
            FROM [表格] AS [t0]
            ) THEN 1
        ELSE 0
     END) AS [value]
```

####  Dapper如何做到同樣效果?

SQL Server可以使用SQL格式`select top 1 1 from [表格] where 條件` 搭配 ExecuteScalar 方法,接著在做一個擴充方法,如下 : 
```C#
public static class DemoExtension
{
  public static bool Any(this IDbConnection cn,string sql,object paramter = null)
  {
    return cn.ExecuteScalar<bool>(sql,paramter);
  }
}
```
效果圖 : 
![20191003043825.png](https://i.loli.net/2019/10/03/iSzgBUmkuOFoa4w.png)

使用如此簡單原因,是利用Dapper ExecuteScalar會去呼叫ExecuteScalarImpl其底層Parse邏輯
```C#
private static T ExecuteScalarImpl<T>(IDbConnection cnn, ref CommandDefinition command)
{
    //..略
    object result;
    //..略
    result = cmd.ExecuteScalar();
    //..略
    return Parse<T>(result);
}

private static T Parse<T>(object value)
{
    if (value == null || value is DBNull) return default(T);
    if (value is T) return (T)value;
    var type = typeof(T);
    //..略
    return (T)Convert.ChangeType(value, type, CultureInfo.InvariantCulture);
}
```
使用 Convert.ChangeType 轉成 bool : `「0=false,非0=true」` 特性,讓系統可以簡單轉型為bool值。

#### 注意
不要QueryFirstOrDefault代替,因為它需要在SQL額外做Null的判斷,否則會出現「NullReferenceException」。
![20191003043931.png](https://i.loli.net/2019/10/03/XoRu7Y3nA58wKpe.png)

這原因是兩者Parse實作方式不一樣,QueryFirstOrDefault判斷結果為null時直接強轉型
![20191003043941.png](https://i.loli.net/2019/10/03/TjHpdwxDMWuVaA5.png)

而ExecuteScalar的Parce實作多了`為空時使用default值`的判斷
![20191003043953.png](https://i.loli.net/2019/10/03/hcoO1sEbaIzHlKS.png)


 　 
 　 
## 23.總結 <a name="page23"></a>


Dapper系列到這邊,重要底層原理差不多都講完了,這系列總共花了筆者連續25天的時間,除了想幫助讀者外,最大的收穫就是我自己在這期間更了解Dapper底層原理,並且學習Dapper精心的細節、框架處理。

另外想提Dapper作者之一Marc Gravell,真的非常熱心,在寫文章的期間有幾個概念疑問,發issue詢問,他都會熱心、詳細的回覆。並且也發現他對代碼的品質要求之高,舉例 : 在S.O發問,遇到他在底下留言 : `「他對目前Dapper IL的架構其實是不滿意的,甚至覺得粗糙,想搭配protobuf-net技術打掉重寫」` (謎之聲 : 真令人敬佩 )

連結 : [c# - How to remove the last few segments of Emit IL at runtime - Stack Overflow](https://stackoverflow.com/questions/58094242/how-to-remove-the-last-few-segments-of-emit-il-at-runtime)
![https://ithelp.ithome.com.tw/upload/images/20190925/201059884hfaioQATW.png](https://ithelp.ithome.com.tw/upload/images/20190925/201059884hfaioQATW.png)

最後筆者想說 : 
寫這篇的初衷,是希望本系列可以幫助到讀者
1. 了解底層邏輯,知其所以然,避免寫出吃掉效能的怪獸,更進一步完整的利用Dapper優點開發專案
2. 可以輕鬆面對Dapper的面試,比起一般使用Dapper工程師回答出更深層的概念
3. 從最簡單Reflection到常用Expression到最細節Emit從頭建立Mapping方法,帶讀者`漸進式`了解Dapper底層強型別Mapping邏輯
4. 了解動態建立方法的重要概念`「結果反推程式碼」`
5. 有基本IL能力,可以利用IL反推C#代碼方式看懂其他專案的底層Emit邏輯
6. 了解Dapper因為緩存的算法邏輯,所以`不能使用錯誤字串拼接SQL`



