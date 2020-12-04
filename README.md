Trace-Dapper.NET-Source-Code
----


## Table of contents

1. [Introduction & Installation Environment](#page1)
2. 

## Introduction <a name="page1"></a>

After years of promotion by Industry Veterans and StackOverflow, “Dapper with Entity Framework” is a powerful combination that meets the needs of “safe, convenient, efficient, maintainable” .

But the current network articles, although there are many articles on Dapper but stay on how to use, no one systematic explanation of the source code logic. So with this article “Trace Dapper Source Code” want to take you into the Dapper code, to understand the details of the design, efficient principles, and learn up practical application in the work.

## Installation Environment <a name="page2"></a>

1. Clone the latest version from [Dapper's Github](https://github.com/StackExchange/Dapper) 
2. Create .Net Core Console project
   ![](https://i.imgur.com/i97jmVQ.png)
3. Install the [NuGet SqlClient](https://www.nuget.org/packages/System.Data.SqlClient) and add the Dapper Project Reference.
   ![](https://i.imgur.com/inIOYfx.png)
4. Running console with breakpoint it allows runtime to view the logic.
   ![](https://i.imgur.com/iSgyvGb.png)

My Personal Environment   

- MSSQLLOCALDB
- Visaul Studio 2019
- LINQPad 5
- Dapper version: V2.0.30
- ILSpy
- Windows 10 pro

## Dynamic Query

With Dapper dynamic Query, you can save time in modifying class attributes in the early stages of development because the table structure is still `in tuning phase`, or it isn’t worth the extra effort to declare class lightweight requirements. 

Use POCO generator to quickly generate `
Strong type` Class to strongly maintain when the table is stable, e.g [PocoClassGenerator](https://github.com/shps951023/PocoClassGenerator).

### Why can Dapper be so convenient and support dynamic?

Going to the source of the Query method can reveals two important points.

1. The entity class is actually `DapperRow` that transformed implicitly to dynamic.
   ![](https://i.imgur.com/ca1QuS8.png)
2. DapperRow inherits `IDynamicMetaObjectProviderand` implements corresponding methods.
   ![](https://i.imgur.com/aBXnex4.png)

For this logic, I will make a simplified version of Dapper dynamic Query to let readers understand the conversion logic:

1. Create a `dynamic` type variable, the entity type is `ExpandoObject`.  
2. Because there’s an inheritance relationship that can be transformed`IDictionary<string, object>`
3. Use DataReader to get the field name using GetName, get the value from the field index, and add both to the Dictionary as a key and value, respectively
4. Because expandobject has the implementation IDynamicMetaObjectProvider interface that can be converted to dynamic

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

![](https://i.imgur.com/nndidpV.png)

Now that we have the concept of the simple expandobject Dynamic Query example, go to the deep  level to see how Dapper handles the details and why dapper customize the DynamicMetaObjectProvider.

First, learn the Dynamic Query process logic:
code: 

```C#
using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
{
    var result = cn.Query("select N'Wei' Name,26 Age").First();
    Console.WriteLine(result.Name);
}
```

The value of the process would be: 
Create Dynamic FUNC > stored in the cache > use `result.Name` >  transfer to call `((DapperRow)result)["Name"]` > from `DapperTable.Values Array` with `index value corresponding to the field "Name" in the Values array` to get value.

Then look at the source code of the GetDapperRowDeserializer method, which controls the logic of how dynamic runs, and is dynamically established as Func for upper-level API calls and cache reuse.

![](https://i.imgur.com/H5zJAlV.png)

This section of Func logic:

1. Although DapperTable is a local variable in the method, it is referenced by the generated Func, so it is `not saved by GC` and always stored in the memory and reused.
   ![](https://i.imgur.com/dhz0bND.png)
2. Because it is dynamic, there is no need to consider the type Mapping, here directly use `GetValue(index)` to get value from database.  

```C#
var values = new object[select columns count];
for (int i = 0; i < values.Length; i++)
{
    object val = r.GetValue(i);
    values[i] = val is DBNull ? null : val;
}
```

3. Save the data in DapperRow

```C#
public DapperRow(DapperTable table, object[] values)
{
    this.table = table ?? throw new ArgumentNullException(nameof(table));
    this.values = values ?? throw new ArgumentNullException(nameof(values));
}
```

4. DapperRow inherits IDynamicMetaObjectProvider and implements the GetMetaObject method. The implementation logic is to return the DapperRowMetaObject object.

```C#
private sealed partial class DapperRow : System.Dynamic.IDynamicMetaObjectProvider
{
    DynamicMetaObject GetMetaObject(Expression parameter)
    {
        return new DapperRowMetaObject(parameter, System.Dynamic.BindingRestrictions.Empty, this);
    }
}
```

5. DapperRowMetaObject main function is to define behavior, by override `BindSetMember、BindGetMember` method, Dapper defines Get, Set of behavior were used `IDictionary<string, object> - GetItem` with `DapperRow - SetValue`
   ![](https://i.imgur.com/LRuCkeB.png)
6. Finally, Dapper uses the DataReader `column order` , first using the column name to get Index, then using Index and Values.  
   ![](https://i.imgur.com/W2SzdOF.png)

## Why inherit IDictionary<string,object>?

One question to consider: If you can define the Get and Set behaviors on your own at Dapperrowmetaobject, does using the Dictionary-GetItem method in some other way mean that you don’t need to inherit the `idictionary<string, object>` ?

One of the reasons Dapper does this has to do with the open principle. Dappertable and Dapperrow are both low-level implementation and `should not be open to users` based on the open-closed principle, so they are `private`.

```C#
private class DapperTable{/*...*/}
private class DapperRow :IDictionary<string, object>, IReadOnlyDictionary<string, object>,System.Dynamic.IDynamicMetaObjectProvider{/*...*/}
```

What if the user wants to know the field name? Because of Dapperrow’s implementation of Idictionary, you can move up to `IDictionary<string, object>` and use it to get field data for the public interface feature.

```C#
public interface IDictionary<TKey, TValue> : ICollection<KeyValuePair<TKey, TValue>>, IEnumerable<KeyValuePair<TKey, TValue>>, IEnumerable{/*..*/}
```

For example, I’ve created a small tool called [HtmlTableHelper](https://github.com/shps951023/HtmlTableHelper) that takes advantage of this feature to automatically convert Dapper Dynamic Query to Table Html, like the following code and images

```C#
using (var cn = "Your Connection")
{
  var sourceData = cn.Query(@"select 'ITWeiHan' Name,25 Age,'M' Gender");
  var tablehtml = sourceData.ToHtmlTable(); //Result : <table><thead><tr><th>Name</th><th>Age</th><th>Gender</th></tr></thead><tbody><tr><td>ITWeiHan</td><td>25</td><td>M</td></tr></tbody></table>
}
```

![](https://i.imgur.com/D8es51O.png)

## 3. Principle of Strongly Typed Mapping Part1: ADO.NET vs. Dapper

Next is the key function of Dapper `Strongly Typed Mapping` . Because of the high difficulty, it will be divided into multiple articles for explanation.

In the first article, compare ADO.NET DataReader GetItem By Index with Dapper Strongly Typed Query, check the difference between the IL and understand the main logic of Dapper Query Mapping.

With the logic, how to implement it, I use three techniques in order: `Reflection、Expression、Emit` implement three versions of the Query method from scratch to let readers understand gradually.

### ADO.NET vs. Dapper

First use the following code to trace the Dapper Query logic

```C#
class Program
{
  static void Main(string[] args)
  {
    using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
    {
      var result = cn.Query<User>("select N'Wei' Name , 25 Age").First();
      Console.WriteLine(result.Name);
      Console.WriteLine(result.Age);
    }
  }
}

public  class  User
{
  public string Name { get; set; }
  public int Age { get; set; }
}
```

Here we need to focus on the `Dapper.SqlMapper.GenerateDeserializerFromMap` method, it is responsible for the logic of Mapping, you can see that a large number of Emit IL technology is used inside.

![20191004012713.png](https://camo.githubusercontent.com/84c032f520434c337ee31c415bc9c27b931b88f9e0d705af5b977a8f15fe1791/68747470733a2f2f692e6c6f6c692e6e65742f323031392f31302f30342f4443564854614f474268634c3539732e706e67)



To understand this IL logic, my way: `「Don’t go straight to the details, but first look at the fully generated IL. 」` As for how to view it, you need to prepare the [il\-visualizer](https://github.com/drewnoakes/il-visualizer) open source tool first, which can view the IL generated by DynamicMethod at Runtime.



It supports vs 2015 and 2017 by default. If you use vs2019 like me, you need to pay attention

1. Need to manually extract the `%USERPROFILE%\Documents\Visual Studio 2019`path below
2. `.netstandard2.0` The project needs to be created `netstandard2.0` and unzipped to this folder
   ![image](https://user-images.githubusercontent.com/12729184/101119088-88b22c00-3625-11eb-9904-2d0033113522.png)

Finally reopen visaul studio and run debug, enter the GetTypeDeserializerImpl method, click the magnifying glass> IL visualizer> view the `Runtime` generated IL code for the DynamicMethod

![image](https://user-images.githubusercontent.com/12729184/101119149-aaabae80-3625-11eb-8d99-06002b720da1.png)

The following IL can be obtained

```C#
IL_0000 : LDC . i4 .0    
IL_0001 : stloc .0     
IL_0002 : newobj      Void . ctor () / Demo . Customer 
IL_0007 : stloc .1     
IL_0008 : ldloc .1     
IL_0009 : after         
IL_000a : LDC . i4 .0    
IL_000b : stloc .0     
IL_000c : ldarg .0     
IL_000d : LDC . i4 .0   
IL_000e: callvirt   System.Object get_Item(Int32)/System.Data.IDataRecord
IL_0013: dup        
IL_0014: stloc.2    
IL_0015: dup        
IL_0016: isinst     System.DBNull
IL_001b: brtrue.s   IL_0029
IL_001d: unbox.any  System.String
IL_0022: callvirt   Void set_Name(System.String)/Demo.User
IL_0027: br.s IL_002b
IL_0029: pop        
IL_002a: pop        
IL_002b: dup        
IL_002c: ldc.i4.1   
IL_002d: stloc.0    
IL_002e: ldarg. 0    
IL_002f: ldc.i4.1   
IL_0030: callvirt   System.Object get_Item(Int32)/System.Data.IDataRecord
IL_0035: dup        
IL_0036: stloc.2    
IL_0037: dup        
IL_0038: isinst     System.DBNull
IL_003d: brtrue.s IL_004b
IL_003f: unbox.any  System.Int32
IL_0044: callvirt   Void set_Age(Int32)/Demo.User
IL_0049: br.s IL_004d
IL_004b: pop        
IL_004c: pop        
IL_004d: stloc.1    
IL_004e: leave IL_0060
IL_0053: ldloc.0    
IL_0054: ldarg. 0    
IL_0055: ldloc.2    
IL_0056: call       Void ThrowDataException(System.Exception, Int32, System.Data.IDataReader, System.Object)/Dapper.SqlMapper
IL_005b: leave IL_0060
IL_0060: ldloc.1    
IL_0061: ret        
```

To understand this IL, you need to understand how it `ADO.NET DataReader fast way to read data` will be used `GetItem By Index` , such as the following code

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

Then look at the IL code generated by this Demo-CastToUser method

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

It can be compared with the IL generated by Dapper shows that it is `roughly the same` (the differences will be explained later), which means that the logic and efficiency of the two operations will be similar, which is `Dapper efficiency is close to the native ado.net` one of the reasons why.

##  Principle of Strongly Typed Mapping Part2: Reflection version

In the previous ado.net Mapping example, we found a serious problem with `there is no way to share multiple classes of methods, and each new class requires a code rewrite`. To solve this problem, write a common method that does different logical processing for different classes during the Runtime.

There are three main implementation methods: Reflection, Expression, and Emit. Here, I will first introduce the simplest method: "Reflection". I will use reflection to simulate Query to write code from scratch to give readers a preliminary understanding of dynamic processing concepts. (If experienced readers can skip this article)

Logic:

1.  Use generics to pass dynamic categories
2.  Use `Generic constraints new()` to create objects dynamically
3.  DataReader need to use `The attribute string name is used as Key` , you can use Reflection acquire property name of the dynamic type in by `DataReader this[string parameter]` obtaining data from database
4.  Use PropertyInfo.SetValue to dynamically assign database data to objects

Finally got the following code:

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

  // 1. Use generics to pass dynamic categories 
  private  static  T  CastToType < T >( this  IDataReader  reader ) where  T : new ()
  {
    // 2. Use generic conditions to constrain new() to dynamically create objects 
    var  instance  =  new  T ();

    // 3.DataReader needs to use the property string name as the Key. You can use Reflection to get the property name of the dynamic category. Use the DataReader this[string parameter] to get the database data 
    var  type  =  typeof ( T );
     var  props  =  type . GetProperties ();
     foreach ( var  p  in  props )
    {
      var val = reader[p.Name];

      // 4. Use PropertyInfo.SetValue to dynamically assign database data to objects 
      if ( ! ( Val  is  System . DBNull ))  
         p . SetValue ( instance , val );
    }

    return instance;
  }
}
```

The advantage of the Reflection version is the code `simple`, but it has the following problems

1. The property query should not be repeated. If it is not used, it should be ignored. Example: If the class has N properties, SQL means to query 3 fields, and the ORM PropertyInfo foreach N times is not 3 times each time. And Dapper specially optimized this logic in Emit IL: `「Check how much you use, not waste」` (explained after this paragraph ).
   ![image](https://user-images.githubusercontent.com/12729184/101120154-dcbe1000-3627-11eb-8e1b-79a6e3777dbc.png)

2. Efficiency issues:

- The reflection efficiency will be slower. After this point, the solution will be introduced: `「Key Cache + Dynamic Create Method」` exchange space for time.

- Using the string Key value will call more `GetOrdinal` methods, you can check the official MSDN explanation `its efficiency is worse than Index value` .

  ![image](https://user-images.githubusercontent.com/12729184/101120383-75549000-3628-11eb-9c8b-be3c3e654c46.png)

## Principle of Strongly Typed Mapping Part3: The important concept of dynamic create method "result inversion code" optimizes efficiency

Then use Expression to solve the Reflection version problem, mainly using Expression features: `「Methods can be dynamically created during Runtime」` to solve the problem.



Before this, we need to have an important concept: `「Reverse the most concise code from the result」` optimizing efficiency. For example: In the past, a classic topic of "printing regular triangle stars'' when learning a program to make a regular triangle of length 3, the common practice would be loop + recursion the way

```C#
void  Main ()
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

But in fact, this topic can be changed to the following code when the length is already known

```C#
Console.WriteLine("  * ");
Console.WriteLine(" * * ");
Console.WriteLine("* * * ");
```

This concept is very important, because the code is reversed from the result, so the logic is straightforward and `efficient` , and Dapper uses this concept to dynamically build methods.

Example: 

- The Name property of User Class corresponds to Reader Index 0, the type is String, and the default value is null

- The Age attribute of User Class corresponds to Reader Index 1, the type is int, and the default value is 0

```C#
void  Main ()
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

If the system can help generate the following logical methods, then the efficiency will be the best

```C#
User dynamic method ( IDataReader  reader )
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

In addition, the above example can be seen for Dapper `SQL Select corresponds to the Class attribute order is very important` , so the algorithm of Dapper in the cache will be explained later, which is specifically optimized for this.

## Principle of Strongly Typed Mapping Part4: Expression version

With the previous logic, we use Expression to implement dynamic creation methods, and then we can think `Why use Expression implementation first instead of Emit?`

In addition to the ability to dynamically build methods, compared to Emit, it has the following advantages:

- `Readadable`, You can use familiar keywords, such as the Variable corresponds to Expression.Variable, and the creation of object New corresponds to Expression.New
  ![image](https://user-images.githubusercontent.com/12729184/101126581-c5d2ea00-3636-11eb-9b4e-bdab6ff2a4d7.png)

- `Easy Runtime Debug`, You can see the logic code corresponding to Expression in Debug mode
  ![image](https://user-images.githubusercontent.com/12729184/101126670-ef8c1100-3636-11eb-82dd-fac4f9d205c7.png)
  ![image](https://user-images.githubusercontent.com/12729184/101126691-f7e44c00-3636-11eb-93b2-6dc1ff20f548.png)

So it is especially suitable for introducing dynamic method establishment, but Expression cannot do some detailed operations compared to Emit, which will be explained by Emit later.

## Rewrite Expression version

Logic:

1.  Get all field names of sql select
2.  Obtain the attribute data of the mapping type > encapsulate the index, sql field and class attribute data in a variable for later use
3.  Dynamic create method: Read the data we want from the database Reader in order, where the code logic:

```C#
User dynamic method ( IDataReader  reader )
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

Finally, the following Exprssion version code 

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
    // 1. Get all field names in sql select 
    var  names  =  Enumerable . Range ( 0 , reader . FieldCount ). Select ( index  =>  reader . GetName ( index )). ToArray ();

    // 2. Get the attribute data of the mapping type > encapsulate the index, sql fields, and class attribute data in a variable for later use 
    var  props  =  type . GetProperties (). ToList ();
     var  members  =  names . Select (( columnName , index ) =>
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

    // 3. Dynamic creation method: read the data we want from the database Reader in order 
    /*Method logic: 
      User dynamic method (IDataReader reader) 
      { 
        var user = new User(); 
        var value = reader[0]; 
        if( !(value is System.DBNull)) 
          user.Name = (string)value; 
        value = reader[1]; 
        if( !(value is System.DBNull)) 
          user.Age = (int)value;   
        return user; 
      } 
	*/ 
    var exBodys = new List < Expression >();    
        
    {
      // method(IDataReader reader)
      var exParam = Expression.Parameter(typeof(DbDataReader), "reader");

      // Mapping category object = new Mapping category(); 
      var  exVar  =  Expression . Variable ( type , " mappingObj " );
       var  exNew  =  Expression . New ( type );
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

![image](https://user-images.githubusercontent.com/12729184/101126967-822cb000-3637-11eb-8f1f-4b2194951181.png)

Finally, check Expression.Lambda > DebugView (note that it is a non-public attribute) verification code:

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

![image](https://user-images.githubusercontent.com/12729184/101127037-a2f50580-3637-11eb-863e-3fa558e19998.png)

## Principle of Strongly Typed Mapping Part5: Emit IL convert to C# code

With the concept of the previous Expression version, we can then enter the core technology at the deep of Dapper: Emit.

First of all, there must be a concept, MSIL (CIL) is intended to be seen by the JIT compiler, so the readability will be poor and difficult to debug, but more detailed logical operations can be done compared to Expression.

In the actual environment development and use Emit, usually `c# code > Decompilation to IL > use Emit to build dynamic methods` , for example:

\1. First create a simple printing example:

```C#
void SyaHello()
{
  Console.WriteLine("Hello World");  
}
```

\2. Decompile and view IL

```C#
SyaHello:
IL_0000:  nop         
IL_0001:  ldstr       "Hello World"
IL_0006:  call        System.Console.WriteLine
IL_000B:  nop         
IL_000C:  ret 
```

\3. Use DynamicMethod + Emit to establish a dynamic method

```C#
void Main()
{
  // 1. create void method()
  DynamicMethod methodbuilder = new DynamicMethod("Deserialize" + Guid.NewGuid().ToString(),typeof(void),null);

  // 2. Create the content of the method body by Emit
  var il = methodbuilder.GetILGenerator();
  il.Emit(OpCodes.Ldstr, "Hello World");
  Type[] types = new Type[1]
  {
    typeof(string)
  };
  MethodInfo method = typeof(Console).GetMethod("WriteLine", types);
  il.Emit(OpCodes.Call,method);
  il.Emit(OpCodes.Ret);
  
  // 3. Convert the specified type of Func or Action 
  var action = (Action)methodbuilder.CreateDelegate(typeof(Action));
  
  action(); 
}
```

But this is not the process for a project that has been written. Developers may not kindly tell you the logic of the original design.  

#### How to check like Dapper, only Emit IL doesn’t have C# Source Code Project

My solution is: `「Since only Runtime can know IL, save IL as a static file and decompile and view」`

You can use the `MethodBuild + Save` method here `Save IL as static exe file > Decompile view` , but you need to pay special attention

1. Please correspond to the parameters and return type, otherwise it will compile error.
2. netstandard does not support this method, Dapper needs to be used `region if yourversion to distinguish, otherwise it cannot be used, such as picture![image](https://user-images.githubusercontent.com/12729184/101128488-c1a8cb80-363a-11eb-8f35-38fafa1ccf8d.png)

code show as below :

```C#
  //Use MethodBuilder to view Emit IL that others have written 
  //1. Create MethodBuilder 
  AppDomain ad = AppDomain.CurrentDomain;
  AssemblyName am = new AssemblyName();
  am.Name = "TestAsm";
  AssemblyBuilder ab = ad.DefineDynamicAssembly(am, AssemblyBuilderAccess.Save);
  ModuleBuilder mb = ab.DefineDynamicModule("Testmod", "TestAsm.exe");
  TypeBuilder tb = mb.DefineType("TestType", TypeAttributes.Public);
  MethodBuilder dm = tb.DefineMethod("TestMeThod", MethodAttributes.Public |
  MethodAttributes.Static, type, new[] { typeof(IDataReader) });
  ab.SetEntryPoint(dm);

  // 2. the IL code 
  //..

  // 3. Generate static files 
  tb.CreateType();
  ab.Save("TestAsm.exe");
```

Then use this method to decompile Dapper Query Mapping IL in the GetTypeDeserializerImpl method, and you can get the C# code:

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

![image](https://user-images.githubusercontent.com/12729184/101128692-25cb8f80-363b-11eb-8673-8a6a0ea8ba3d.png)



After having the C# code, it will be much faster to understand the Emit logic, and then you can enter the Emit version Query implementation part.



## Strongly Typed Mapping Principle Part6: Emit Version

The following code is the Emit version, I wrote the corresponding IL part of C# in the comments.

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
    var constructor = returnType.GetConstructors(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic)[0]; 
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
      il.Emit(OpCodes.Ldarg_0); 
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
    // IL_0054:  stloc.s     04  
    // IL_0056:  br.s        IL_0058
    // IL_0058:  ldloc.s     04  
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

There are many detailed concepts of Emit here. First pick out the important concepts to explain.

## Emit Label

In Emit if/else, you need to use Label positioning, tell the compiler which position to jump to when the condition is true/false, for example: `boolean to integer`, assuming that you want to simply convert Boolean to Int, C# code can use `If it is True Return 1 otherwise return 0` logic to write:

```C#
public static int BoolToInt(bool input) => input ? 1 : 0;
```

When converting to Emit, the following logic is required:

1.  Consider the label dynamic positioning problem
2.  The label must be established first to let Brtrue\_S know which label position to set when the conditions are met `(Note:at this time the label position has not been determined yet)`
3.  Continue to build IL from top to bottom in order
4.  Wait until `match condition` you want to run the block `previous line` , use it `MarkLabel to position Label` .

The final c # Emit Code:

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

Here you can find the Emit version, which has the advantage of:

\1. To do more detailed manipulations

\2. Because the detail granularity is small, the efficiency that can be optimized is better

Disadvantages:

1.  Difficult to debug
2.  Poor readability
3.  The amount of code becomes larger and the complexity increases

Then look at the suggestions of the author of Dapper. Now there is no need to use Emit in general projects. Using Expression + Func/Action can solve most of the needs of dynamic methods, especially when Expression supports Block and other methods. Link [c#\-What's faster: expression trees or manually emitting IL](https://stackoverflow.com/questions/16530539/whats-faster-expression-trees-or-manually-emitting-il)

![image](https://user-images.githubusercontent.com/12729184/101129368-67a90580-363c-11eb-84c4-019c290fc8d0.png)

Having said that, there are some powerful open source projects that use Emit to manage details `If you want to understand them, you need the basic Emit IL concept` .

## One of the keys to Dapper's fast efficiency: Cache principle

Why can Dapper be so fast?  

I introduced the dynamic use of Emit IL to establish the ADO.NET Mapping method, but this function alone cannot make Dapper the king of lightweight ORM efficiency.

Because the dynamic create method is `Cost and time consuming` action, simply using it will slow down the speed. But when it cooperates with Cache, it is different. By storing the established method in Cache, you can use the `『Space for time』` concept to speed up the efficiency of the query .

Then trace the Dapper source code. This time, we need to pay special attention to Identity and GetCacheInfo under the QueryImpl method.

![image](https://user-images.githubusercontent.com/12729184/101129638-ef8f0f80-363c-11eb-8463-73a9ee081b1b.png)

### Identity、GetCacheInfo

Identity mainly encapsulates the comparison Key attribute of each cache:

*   sql: distinguish different SQL strings
*   type: distinguish Mapping type
*   commandType: Responsible for distinguishing different databases
*   gridIndex: Mainly used in QueryMultiple, explained later.
*   connectionString: Mainly distinguish the same database manufacturer but different DB situation
*   parametersType: Mainly distinguish parameter types
*   typeCount: Mainly used in Multi Query multi\-mapping, it needs to be used with the override GetType method, which will be explained later

Then match the cache type used by Dapper in the GetCacheInfo method. When `ConcurrentDictionary<Identity, CacheInfo>` using the `TryGetValue` method, it will first compare the HashCode and then compare the Equals features, such as the image source code.

![image](https://user-images.githubusercontent.com/12729184/101129762-2ebd6080-363d-11eb-898c-8fa84b7205fa.png)

Using the Key type Identity to `override Equals` implement the cache comparison algorithm, you can see the following Dapper implementation logic. As long as one attribute is different, a new dynamic method and cache will be created.

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

With this concept, the previous Emit version is modified into a simple Cache Demo :

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
        
        // 2. If the cache has data, use it, and if there is no data, create a method dynamically and save it in the cache 
        if ( ! Readers . TryGetValue ( identity , out  Func < DbDataReader , object > func ))
        {
          //The dynamic creation method 
          func  =  GetTypeDeserializerImpl ( typeof ( T ), reader );
           readers [ identity ] =  func ;
           Console . WriteLine ( " No cache, create a dynamic method and put it in the cache " );
        } else {
           Console . WriteLine ( " Use cache " );
        }


        // 3. Call the generated method by reader, read the data and return 
        while ( reader . Read ())
        {
          var result = func(reader as DbDataReader);
          yield return result is T ? (T)result : default(T);
        }
      }

    }
  }

  private static Func<DbDataReader, object> GetTypeDeserializerImpl(Type type, IDataReader reader, int startBound = 0, int length = -1, bool returnNullIfFirstMissing = false)
  {
    // .. slightly
  }
}
```

![image](https://user-images.githubusercontent.com/12729184/101130013-a5f2f480-363d-11eb-8e1d-1b9851c6e5b2.png)
