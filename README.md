Trace Dapper.NET Source Code
English : Link  
Traditional Chinese : Link  
Simplified Chinese : Link  

1. Introduction
After years of promotion by Industry Veterans and StackOverflow, “Dapper with Entity Framework” is a powerful combination that deal the needs of “safe, convenient, efficient, maintainable”.

But the current network articles, although there are many articles on Dapper but stay on how to use, no one systematic explanation of the source code logic. So with this article “Trace Dapper Source Code” want to take you into the Dapper code, to understand the details of the design, efficient principles, and learn up practical application in the work.

2. Installation Environment
Clone the latest version from Dapper's Github 

Create .Net Core Console project


Install the NuGet SqlClient and add the Dapper Project Reference.


Running console with breakpoint it allows runtime to view the logic.


My Personal Environment   

MSSQLLOCALDB

Visaul Studio 2019

LINQPad 5

Dapper version: V2.0.30

ILSpy

Windows 10 pro

3. Dynamic Query
With Dapper dynamic Query, you can save time in modifying class attributes in the early stages of development because the table structure is still in the adjustment stage, or it isn’t worth the extra effort to declare class lightweight requirements. 

When the table is stable, use the POCO generator to quickly generate the Class and convert it to strong type maintenance, e.g PocoClassGenerator..

Why can Dapper be so convenient and support dynamic?
Two key points can be found by tracing the source code of the Query method

The entity class is actually DapperRow that transformed implicitly to dynamic.


DapperRow inherits IDynamicMetaObjectProviderand & implements corresponding methods.


For this logic, I will make a simplified version of Dapper dynamic Query to let readers understand the conversion logic:

Create a dynamic type variable, the entity type is ExpandoObject.  

Because there’s an inheritance relationship that can be transformed to IDictionary<string, object>

Use DataReader to get the field name using GetName, get the value from the field index, and add both to the Dictionary as a key and value

Because expandobject has the implementation IDynamicMetaObjectProvider interface that can be converted to dynamic

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
Now that we have the concept of the simple expandobject Dynamic Query example, go to the deep level to see how Dapper handles the details and why dapper customize the DynamicMetaObjectProvider.

First, learn the Dynamic Query process logic:
code: 

using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
{
    var result = cn.Query("select N'Wei' Name,26 Age").First();
    Console.WriteLine(result.Name);
}
The value of the process would be: 
Create Dynamic FUNC > stored in the cache > use result.Name >  transfer to call ((DapperRow)result)["Name"] > from DapperTable.Values Array with index value corresponding to the field "Name" in the Values array to get value.

Then look at the source code of the GetDapperRowDeserializer method, which controls the logic of how dynamic runs, and is dynamically created as Func for upper-level API calls and cache reuse.



This section of Func logic:

Although DapperTable is a local variable in the method, it is referenced by the generated Func, so it will not be GC and always stored in the memory and reused.


Because it is dynamic, there is no need to consider the type Mapping, here directly use GetValue(index) to get value from database.  

var values = new object[select columns count];
for (int i = 0; i < values.Length; i++)
{
    object val = r.GetValue(i);
    values[i] = val is DBNull ? null : val;
}
Save the data in DapperRow

public DapperRow(DapperTable table, object[] values)
{
    this.table = table ?? throw new ArgumentNullException(nameof(table));
    this.values = values ?? throw new ArgumentNullException(nameof(values));
}
DapperRow inherits IDynamicMetaObjectProvider and implements the GetMetaObject method. The implementation logic is to return the DapperRowMetaObject object.

private sealed partial class DapperRow : System.Dynamic.IDynamicMetaObjectProvider
{
    DynamicMetaObject GetMetaObject(Expression parameter)
    {
        return new DapperRowMetaObject(parameter, System.Dynamic.BindingRestrictions.Empty, this);
    }
}
DapperRowMetaObject main function is to define behavior, by override BindSetMember、BindGetMember method, Dapper defines Get, Set of behavior were used IDictionary<string, object> - GetItem , DapperRow - SetValue


Finally, Dapper uses the DataReader column order , first using the column name to get Index, then using Index and Values.  


Why inherit IDictionary<string,object>?
There is a question to think about: In DapperRowMetaObject, you can define the Get and Set behaviors by yourself, so instead of using the Dictionary-GetItem method, instead of using other methods, does it mean that you don't need to inherit IDictionary<string,object>?

One of the reasons for Dapper to do this is related to the open principle. DapperTable and DapperRow are all low-level implementation class. Based on the open and closed principle, they should not be opened to users, so they are set as private.

private class DapperTable{/*...*/}
private class DapperRow :IDictionary<string, object>, IReadOnlyDictionary<string, object>,System.Dynamic.IDynamicMetaObjectProvider{/*...*/}
What if the user wants to know the field name? 

Because DapperRow implements IDictionary, it can be upcasting to IDictionary<string, object>, and use it to get field data by public interface.

public interface IDictionary<TKey, TValue> : ICollection<KeyValuePair<TKey, TValue>>, IEnumerable<KeyValuePair<TKey, TValue>>, IEnumerable{/*..*/}
For example, I’ve created a tool called HtmlTableHelper to use this feature to automatically convert Dapper Dynamic Query to Table Html, such as the following code and picture

using (var cn = "Your Connection")
{
  var sourceData = cn.Query(@"select 'ITWeiHan' Name,25 Age,'M' Gender");
  var tablehtml = sourceData.ToHtmlTable(); //Result : <table><thead><tr><th>Name</th><th>Age</th><th>Gender</th></tr></thead><tbody><tr><td>ITWeiHan</td><td>25</td><td>M</td></tr></tbody></table>
}


4. Strongly Typed Mapping Part1: ADO.NET vs. Dapper
Next is the key function of Dapper, Strongly Typed Mapping. Because of the difficulty, it will be divided into multiple parts for explanation.

In the first part, compare ADO.NET DataReader GetItem By Index with Dapper Strongly Typed Query, check the difference between the IL and understand the main logic of Dapper Query Mapping.

With the logic, how to implement it, I use three techniques in order: Reflection、Expression、Emit implement three versions of the Query method from scratch to let readers understand gradually.

ADO.NET vs. Dapper

First use the following code to trace the Dapper Query logic

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
​
public  class  User
{
  public string Name { get; set; }
  public int Age { get; set; }
}
Here we need to focus on the Dapper.SqlMapper.GenerateDeserializerFromMap method, it is responsible for the logic of Mapping, you can see that a large number of Emit IL technology is used inside.

20191004012713.png

To understand this IL logic, my way: "You should not go directly to the details, but check the complete IL first" As for how to view it, you need to prepare the il-visualizer open source tool first, which can view the IL generated by DynamicMethod at Runtime.

It supports vs 2015 and 2017 by default. If you use vs2019 like me

Need to manually extract the %USERPROFILE%\Documents\Visual Studio 2019 path below

.netstandard2.0 The project needs to be created netstandard2.0 and unzipped to this folder
image

Finally reopen visaul studio and run debug, enter the GetTypeDeserializerImpl method, click the magnifying glass> IL visualizer> view the Runtime generated IL code for the DynamicMethod

image

The following IL can be obtained

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
To understand this IL, you need to understand how it ADO.NET DataReader fast way to read data will be used GetItem By Index , such as the following code

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
​
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
Then look at the IL code generated by this Demo-CastToUser method

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
It can be compared with the IL generated by Dapper shows that it is roughly the same (the differences will be explained later), which means that the logic and efficiency of the two operations will be similar, which is Dapper efficiency is close to the native ado.net the reasons why.

5. Strongly Typed Mapping Part2: Reflection version
In the previous ado.net Mapping example, we found a serious problem with there is no way to share multiple classes of methods, and each new class requires a code rewrite. To solve this problem, write a common method that does different logical processing for different classes during the Runtime.

There are three main implementation methods: Reflection, Expression, and Emit. Here, I will first introduce the simplest method: "Reflection". I will use reflection to simulate Query to write code from scratch to give readers a preliminary understanding of dynamic processing concepts. (If experienced readers can skip this article)

Logic:

Use generics to pass dynamic class

Use Generic constraints new() to create objects dynamically

DataReader need to use  attribute string name is used as Key , you can use Reflection to get the attribute name of the dynamic type and get the database data through the DataReader this[string parameter]

Use PropertyInfo.SetValue to dynamically assign database data to objects

Finally got the following code:

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
​
  // 1. Use generics to pass dynamic class 
  private  static  T  CastToType < T >( this  IDataReader  reader ) where  T : new ()
  {
    // 2. Use `Generic constraints new()` to create objects dynamically
    var  instance  =  new  T ();
​
    // 3.DataReader need to use  `attribute string name is used as Key` , you can use Reflection to get the attribute name of the dynamic type and get the database data through the `DataReader this[string parameter]`
    var  type  =  typeof ( T );
     var  props  =  type . GetProperties ();
     foreach ( var  p  in  props )
    {
      var val = reader[p.Name];
​
      // 4. Use PropertyInfo.SetValue to dynamically assign database data to objects 
      if ( ! ( Val  is  System . DBNull ))  
         p . SetValue ( instance , val );
    }
​
    return instance;
  }
}
The advantage of the Reflection version is that the code is simple, but it has the following problems

The attribute query should not be repeated, and it should be ignored if it is not used. Example: If the class has N properties, SQL means to query 3 fields, and the ORM PropertyInfo foreach N times is not 3 times each time. And Dapper specially optimized this logic in Emit IL: 「Check how much you use, not waste」.
image

Efficiency issues:

The reflection efficiency will be slower.  the solution will be introduced later: 「Key Cache + Dynamic Create Method」 exchange space for time.

Using the string Key value will call more GetOrdinal methods, you can check the official MSDN explanation its efficiency is worse than Index value .

image

6. Strongly Typed Mapping Part3: The important concept of dynamic create method "code from  result" optimizes efficiency
Then use Expression to solve the Reflection version problem, mainly using Expression features: 「Methods can be dynamically created during Runtime」 to solve the problem.

Before this, we need to have an important concept: 「Reverse the most concise code from the result」 optimizing efficiency. For example: In the past, a classic topic of "printing regular triangle stars'' when learning a program to make a regular triangle of length 3, the common practice would be loop + recursion the way

void  Main ()
{
  Print(3,0);
}
​
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
But in fact, this topic can be changed to the following code when the length is already known

Console.WriteLine("  * ");
Console.WriteLine(" * * ");
Console.WriteLine("* * * ");
This concept is very important, because the code is reversed from the result, so the logic is straightforward and efficient , and Dapper uses this concept to dynamically build methods.

Example: 

The Name property of User Class corresponds to Reader Index 0, the type is String, and the default value is null

The Age attribute of User Class corresponds to Reader Index 1, the type is int, and the default value is 0

void  Main ()
{
  using (var cn = Connection)
  {
    var result = cn.Query<User>("select N'Wei' Name,26 Age").First();
  }
}
​
class User
{
  public string Name { get; set; }
  public int Age { get; set; }
}
If the system can help generate the following logical methods, then the efficiency will be the best

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
In addition, the above example can be seen for Dapper SQL Select corresponds to the Class attribute order is very important , so the algorithm of Dapper in the cache will be explained later, which is specifically optimized for this.

7. Strongly Typed Mapping Part4: Expression version
With the previous logic, we use Expression to implement dynamic creation methods, and then we can think Why use Expression implementation first instead of Emit?

In addition to the ability to dynamically build methods, compared to Emit, it has the following advantages:

Readadable, You can use familiar keywords, such as the Variable corresponds to Expression.Variable, and the creation of object New corresponds to Expression.New
image

Easy Runtime Debug, You can see the logic code corresponding to Expression in Debug mode
image
image

So it is especially suitable for introducing dynamic method establishment, but Expression cannot do some detailed operations compared to Emit, which will be explained by Emit later.

Rewrite Expression version
Logic:

Get all field names of sql select

Obtain the attribute data of the mapping type > encapsulate the index, sql field and class attribute data in a variable for later use

Dynamic create method: Read the data we want from the database Reader in order,  the code logic:

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
Finally, the following Exprssion version code 

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
​
      }
    }
  }
​
  private static Func<DbDataReader, object> CreateMappingFunction(IDataReader reader, Type type)
  {
    // 1. Get all field names in sql select 
    var  names  =  Enumerable . Range ( 0 , reader . FieldCount ). Select ( index  =>  reader . GetName ( index )). ToArray ();
​
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
​
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
​
      // Mapping class object = new Mapping class(); 
      var  exVar  =  Expression . Variable ( type , " mappingObj " );
       var  exNew  =  Expression . New ( type );
      {
        exBodys.Add(Expression.Assign(exVar, exNew));
      }
​
      // var value = defalut(object);
      var exValueVar = Expression.Variable(typeof(object), "value");
      {
        exBodys.Add(Expression.Assign(exValueVar, Expression.Constant(null)));
      }
​
​
      var getItemMethod = typeof(DbDataReader).GetMethods().Where(w => w.Name == "get_Item")
        .First(w => w.GetParameters().First().ParameterType == typeof(int));
      foreach (var m in members)
      {
        //reader[0]
        var exCall = Expression.Call(
          exParam, getItemMethod,
          Expression.Constant(m.index)
        );
​
        // value = reader[0];
        exBodys.Add(Expression.Assign(exValueVar, exCall));
​
        //user.Name = (string)value;
        var exProp = Expression.Property(exVar, m.property.Name);
        var exConvert = Expression.Convert(exValueVar, m.property.PropertyType); //(string)value
        var exPropAssign = Expression.Assign(exProp, exConvert);
​
        //if ( !(value is System.DBNull))
        //    (string)value
        var exIfThenElse = Expression.IfThen(
          Expression.Not(Expression.TypeIs(exValueVar, typeof(System.DBNull)))
          , exPropAssign
        );
​
        exBodys.Add(exIfThenElse);
      }
​
​
      // return user;  
      exBodys.Add(exVar);
​
      // Compiler Expression 
      var lambda = Expression.Lambda<Func<DbDataReader, object>>(
        Expression.Block(
          new[] { exVar, exValueVar },
          exBodys
        ), exParam
      );
​
      return lambda.Compile();
    }
  }
}
image

Finally, check Expression.Lambda > DebugView (note that it is a non-public attribute) :

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
image

8. Strongly Typed Mapping Part5: Emit IL convert to C# code
With the concept of the previous Expression version, we can then enter the core technology of Dapper: Emit.

First of all, there must be a concept, MSIL (CIL) is intended for JIT compiler, so the readability will be poor and difficult to debug, but more detailed logical operations can be done compared to Expression.

In the actual environment development and use Emit, usually c# code > Decompilation to IL > use Emit to build dynamic methods , for example:

First create a simple printing example:

void SyaHello()
{
  Console.WriteLine("Hello World");  
}
Decompile and view IL

SyaHello:
IL_0000:  nop         
IL_0001:  ldstr       "Hello World"
IL_0006:  call        System.Console.WriteLine
IL_000B:  nop         
IL_000C:  ret 
Use DynamicMethod + Emit to create a dynamic method

void Main()
{
  // 1. create void method()
  DynamicMethod methodbuilder = new DynamicMethod("Deserialize" + Guid.NewGuid().ToString(),typeof(void),null);
​
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
But this is not the process for a project that has been written. Developers may not kindly tell you the logic of the original design.  

How to check like Dapper, only Emit IL doesn’t have C# Source Code Project
My solution is: 「Since only Runtime can know IL, save IL as a static file and decompile and view」

You can use the MethodBuild + Save method here Save IL as static exe file > Decompile view , but you need to pay special attention

Please correspond to the parameters and return type, otherwise it will compile error.

netstandard does not support this method, Dapper needs to be used region if yourversion to distinguish, otherwise it cannot be used, such as pictureimage

code show as below :

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
​
  // 2. the IL code 
  //..
​
  // 3. Generate static files 
  tb.CreateType();
  ab.Save("TestAsm.exe");
Then use this method to decompile Dapper Query Mapping IL in the GetTypeDeserializerImpl method, and you can get the C# code:

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
image

After having the C# code, it will be much faster to understand the Emit logic.

9. Strongly Typed Mapping Principle Part6: Emit Version
The following code is the Emit version, I wrote the corresponding IL part of C# code

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
​
        while (reader.Read())
        {
          var result = func(reader as DbDataReader);
          yield return result is T ? (T)result : default(T);
        }
      }
​
    }
  }
​
  private static Func<DbDataReader, object> GetTypeDeserializerImpl(Type type, IDataReader reader, int startBound = 0, int length = -1, bool returnNullIfFirstMissing = false)
  {
    var returnType = type.IsValueType ? typeof(object) : type;
​
    var dm = new DynamicMethod("Deserialize" + Guid.NewGuid().ToString(), returnType, new[] { typeof(IDataReader) }, type, true);
    var il = dm.GetILGenerator();
​
    //C# : User user = new User();
    //IL : 
    //IL_0001:  newobj      
    //IL_0006:  stloc.0         
    var constructor = returnType.GetConstructors(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic)[0]; 
    il.Emit(OpCodes.Newobj, constructor);
    var returnValueLocal = il.DeclareLocal(type);
    il.Emit(OpCodes.Stloc, returnValueLocal); //User user = new User();
​
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
​
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
​
​
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
​
      index++;
    }
​
    // IL_0053:  ldloc.0     // user
    // IL_0054:  stloc.s     04  
    // IL_0056:  br.s        IL_0058
    // IL_0058:  ldloc.s     04  
    // IL_005A:  ret         
    il.Emit(OpCodes.Ldloc, returnValueLocal);
    il.Emit(OpCodes.Ret);
​
    var funcType = System.Linq.Expressions.Expression.GetFuncType(typeof(IDataReader), returnType);
    return (Func<IDataReader, object>)dm.CreateDelegate(funcType);
  }
​
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
There are many detailed of Emit here. First pick out the important concepts to explain.

Emit Label

In Emit if/else, you need to use Label positioning, tell the compiler which position to jump to when the condition is true/false, for example: boolean to integer, assuming that you want to simply convert Boolean to Int, C# code can use If it is True Return 1 otherwise return 0 logic to write:

public static int BoolToInt(bool input) => input ? 1 : 0;
When converting to Emit, the following logic is required:

Consider the label dynamic positioning problem

The label must be established first to let Brtrue_S know which label position to set when the conditions are true (Note:at this time the label position has not been determined yet)

Continue to build IL from top to bottom in order

Wait until match condition you want to run the block previous line , use it MarkLabel to position Label .

The final c # Emit Code:

public class Program
{
  public static void Main(string[] args)
  {
    var func = CreateFunc();
    Console.WriteLine(func(true)); //1
    Console.WriteLine(func(false)); //0
  }
​
  static Func<bool, int> CreateFunc()
  {
    var dm = new DynamicMethod("Test" + Guid.NewGuid().ToString(), typeof(int), new[] { typeof(bool) });
​
    var il = dm.GetILGenerator();
    var labelTrue = il.DefineLabel();
​
    il.Emit(OpCodes.Ldarg_0);
    il.Emit(OpCodes.Brtrue_S, labelTrue);
    il.Emit(OpCodes.Ldc_I4_0);
    il.Emit(OpCodes.Ret);
    il.MarkLabel(labelTrue);
    il.Emit(OpCodes.Ldc_I4_1);
    il.Emit(OpCodes.Ret);
​
    var funcType = System.Linq.Expressions.Expression.GetFuncType(typeof(bool), typeof(int));
    return (Func<bool, int>)dm.CreateDelegate(funcType);
  }
}
Here you can find the Emit version, which has the advantage of:

Can do more detailed operations

Because the detail granularity is small, the efficiency that can be optimized is better

Disadvantages:

Difficult to debug

Poor readability

The amount of code becomes larger and the complexity increases

Then look at the suggestions of the author of Dapper. Now there is no need to use Emit in general projects. Using Expression + Func/Action can solve most of the needs of dynamic methods, especially when Expression supports Block and other methods. Link c#-What's faster: expression trees or manually emitting IL

image

Having said that, there are some powerful open source projects that use Emit to manage details If you want to understand them, you need the basic Emit IL concept .

10. One of the keys to Dapper's fast efficiency: Cache principle
Why can Dapper be so fast?  

I introduced the dynamic use of Emit IL to establish the ADO.NET Mapping method, but this function cannot make Dapper the king of lightweight ORM efficiency.

Because the dynamic create method is Cost and time consuming action, simply using it will slow down the speed. But when it cooperates with Cache, it is different. By storing the established method in Cache, you can use the 『Space for time』 concept to speed up the efficiency of the query .

Then trace the Dapper source code. This time, we need to pay special attention to Identity and GetCacheInfo under the QueryImpl method.

image

Identity、GetCacheInfo

Identity mainly encapsulates the comparison Key attribute of each cache:

sql: distinguish different SQL strings

type: distinguish Mapping type

commandType: Responsible for distinguishing different databases

gridIndex: Mainly used in QueryMultiple, explained later.

connectionString: Mainly distinguish the same database manufacturer but different DB situation

parametersType: Mainly distinguish parameter types

typeCount: Mainly used in Multi Query multi-mapping, it needs to be used with the override GetType method, which will be explained later

Then match the cache type used by Dapper in the GetCacheInfo method. When ConcurrentDictionary<Identity, CacheInfo> using the TryGetValue method, it will first compare the HashCode and then compare the Equals features, such as the image source code.

image

Using the Key type Identity to override Equals implement the cache comparison algorithm, you can see the following Dapper implementation logic. As long as one attribute is different, a new dynamic method and cache will be created.

public bool Equals(Identity other)
{
  if (ReferenceEquals(this, other)) return true;
  if (ReferenceEquals(other, null)) return false;
​
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
With this concept, the previous Emit version is modified into a simple Cache Demo :

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
​
    public readonly int hashCode;
    public override int GetHashCode() => hashCode;
​
    public override bool Equals(object obj) => Equals(obj as Identity);
    public bool Equals(Identity other)
    {
        if (ReferenceEquals(this, other)) return true;
        if (ReferenceEquals(other, null)) return false;
​
        return type == other.type
          && sql == other.sql
          && commandType == other.commandType
          && StringComparer.Ordinal.Equals(connectionString, other.connectionString)
          && parametersType == other.parametersType;
    }
}
​
public static class DemoExtension
{
    private static readonly Dictionary<Identity, Func<DbDataReader, object>> readers = new Dictionary<Identity, Func<DbDataReader, object>>();
​
    public static IEnumerable<T> Query<T>(this IDbConnection cnn, string sql, object param = null) where T : new()
    {
        using (var command = cnn.CreateCommand())
        {
            command.CommandText = sql;
            using (var reader = command.ExecuteReader())
            {
                var identity = new Identity(command.CommandText, command.CommandType, cnn.ConnectionString, typeof(T), param?.GetType());
​
                // 2. If the cache has data, use it, and if there is no data, create a method dynamically and save it in the cache 
                if (!readers.TryGetValue(identity, out Func<DbDataReader, object> func))
                {
                    //The dynamic creation method 
                    func = GetTypeDeserializerImpl(typeof(T), reader);
                    readers[identity] = func;
                    Console.WriteLine(" No cache, create a dynamic method and put it in the cache ");
                }
                else
                {
                    Console.WriteLine(" Use cache ");
                }
​
​
                // 3. Call the generated method by reader, read the data and return 
                while (reader.Read())
                {
                    var result = func(reader as DbDataReader);
                    yield return result is T ? (T)result : default(T);
                }
            }
​
        }
    }
​
    private static Func<DbDataReader, object> GetTypeDeserializerImpl(Type type, IDataReader reader, int startBound = 0, int length = -1, bool returnNullIfFirstMissing = false)
    {
        // ..
    }
}
image

11. Wrong SQL string contacting will cause slow efficiency and memory leaks
Here's an important concept used by Dapper. Its SQL string is one of the important key values to cache. If different SQL strings are used, Dapper will create new dynamic methods and caches for this, so even if you use StringBuilder improperly can also cause slow query & memory leaks .

image

Why the SQL string is used as one of keys, instead of simply using the Handle of the Mapping type, one of the reasons is order of query column . As mentioned earlier, Dapper uses the 「result convert to code」 method to create a dynamic method, which means that the order and data must be fixed , avoid using the same set of dynamic methods with different SQL Select column order, there will be a A column value to b column wrong value problem.

The most direct solution is to establish a different dynamic method for each different SQL string and save it in a different cache.

For example, the following code is just a simple query action, but the number of Dapper Caches has reached 999999, such as image display

using (var cn = new SqlConnection(@"connectionString"))
{
    for ( int  i  = 0; i  <  999999 ; i ++ )
    {
        var guid = Guid.NewGuid();
        for (int i2 = 0; i2 < 2; i2++)
        {
            var result = cn.Query<User>($"select '{guid}' ").First();
        }  
    }
}
image

To avoid this problem, you only need to maintain a principle Reuse SQL string , and the simplest way is parametrization , for example: Change the above code to the following code, the number of caches is reduced to 1 , to achieve the purpose of reuse:

using (var cn = new SqlConnection(@"connectionString"))
{
    for ( int  i  = 0; i  <  999999 ; i ++ )
    {
        var guid = Guid.NewGuid();
        for (int i2 = 0; i2 < 2; i2++)
        {
            var result = cn.Query<User>($"select @guid ",new { guid}).First();
        }  
    }
}
image

12. Dapper SQL correct string contacting method: Literal Replacement
If there is a need to splice SQL strings, for example: Sometimes it is more efficient to use string contacting than not to use parameterization, especially if there are only a few  fixed values .

At this time, Dapper can use the Literal Replacements function, how to use it: {=Attribute_Name} replace the value string to be contacted, and save the value in the Parameter, for example:

void Main()
{
  using (var cn = Connection)
  {
    var result = cn.Query("select N'Wei' Name,26 Age,{=VipLevel} VipLevel", new User{ VipLevel = 1}).First();
  }
}
13. Why Literal Replacement can avoid caching problems?
First, trace the GetLiteralTokens method under the source code GetCacheInfo, you can find that before cache Dapper will get the data SQL string  that match {=Attribute_Name} role  .

private static readonly Regex literalTokens = new Regex(@"(?<![\p{L}\p{N}_])\{=([\p{L}\p{N}_]+)\}", RegexOptions.IgnoreCase | RegexOptions.Multiline | RegexOptions.CultureInvariant | RegexOptions.Compiled);
internal static IList<LiteralToken> GetLiteralTokens(string sql)
{
  if (string.IsNullOrEmpty(sql)) return LiteralToken.None;
  if (!literalTokens.IsMatch(sql)) return LiteralToken.None;
​
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
Then generate Parameter parameterized dynamic method in the CreateParamInfoGenerator method. The method IL of this section is as below:

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
​
IL_0023: call       System.Globalization.CultureInfo get_InvariantCulture()/System.Globalization.CultureInfo
IL_0028: call       System.String ToString(System.IFormatProvider)/System.Int32
IL_002d: callvirt   System.String Replace(System.String, System.String)/System.String
IL_0032: callvirt   Void set_CommandText(System.String)/System.Data.IDbCommand
IL_0037: ret        
Then generate the Mapping dynamic method. To understand this logic, I will make a simulation example here:

public  static  class  DbExtension
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
​
  private static void CommandLiteralReplace(IDbCommand cmd, User parameter)
  {
    cmd.CommandText = cmd.CommandText.Replace("{=VipLevel}", parameter.VipLevel.ToString(System.Globalization.CultureInfo.InvariantCulture));
  }
​
  private  static  User  Mapping ( IDataReader  reader )
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
After reading the above example, you can find that the underlying principle of Dapper Literal Replacements is string replace that it also belongs to the string contacting way. Why can the cache problem be avoided?

This is because the replacement timing is in the SetParameter dynamic method, so the Cache SQL Key is unchanged can reuse the same SQL string and cache.

Also because it is a string replace method, only support basic value type if you use the String type, the system will inform you The type String is not supported for SQL literals.to avoid SQL Injection problems.

How to use Query Multi Mapping
Then explain the Dapper Multi Mapping(multi-mapping) implementation and the underlying logic. After all, there can not always one-to-one relation in work.

How to use:

You need to write your own Mapping logic and use it: Query<Func>(SQL,Parameter,Mapping Func)

Need to specify the generic parameter type, the rule is Query<Func first type,Func second type,..,Func last type> (supports up to six sets of generic parameters)

Specify the name of the cutting field ID , it is used by default , if it is different, it needs to be specified .

The order is from left to right

For example: There is an order (Order) and a member (User) form, the relationship is a one-to-many relationship, a member can have multiple orders, the following is the C# Demo code:

void Main()
{
    using (var ts = new TransactionScope())
    using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
    {
        cn.Execute(@"
      CREATE TABLE [User]([ID] int, [Name] nvarchar(10));
      INSERT INTO [User]([ID], [Name])VALUES(1, N'Jack'),(2, N'Lee');
​
      CREATE TABLE [Order]([ID] int, [OrderNo] varchar(13), [UserID] int);
      INSERT INTO [Order]([ID], [OrderNo], [UserID])VALUES(1, 'SO20190900001', 1),(2, 'SO20190900002', 1),(3, 'SO20190900003', 2),(4, 'SO20190900004', 2);
    ");
​
        var result = cn.Query<Order, User, Order>(@"
        select * from [order] T1
        left join [User] T2 on T1.UserId = T2.ID    
      ", (order, user) =>
        {
            order.User = user;
            return order;
        }
        );
​
        ts.Dispose();
    }
}
​
public class Order
{
    public int ID { get; set; }
    public string OrderNo { get; set; }
    public User User { get; set; }
}
​
public class User
{
    public int ID { get; set; }
    public string Name { get; set; }
}
image

14. Support dynamic Multi Mapping
In the initial stage, the table structure is often changed or the one-time function and does not want to declare the class. Dapper Multi Mapping also supports the dynamic method.

void  Main ()
{
  using (var ts = new TransactionScope())
  using (var connection = Connection)
  {
    const string createSql = @"
            create table Users (Id int, Name nvarchar(20))
            create table Posts (Id int, OwnerId int, Content nvarchar(20))
​
            insert Users values(1, N'Jack')
            insert Users values(2, N'Lee')
​
            insert Posts values(101, 1, N'Jack's first day diary') 
            insert Posts values(102, 1, N'Jack's second day diary') 
            insert Posts values(103, 2, N'Lee's first day diary ') 
" ;
     connection . Execute ( createSql );    
​
    const string sql =
      @"select * from Posts p 
      left join Users u on u.Id = p.OwnerId 
      Order by p.Id
    ";
​
    var data = connection.Query<dynamic, dynamic, dynamic>(sql, (post, user) => { post.Owner = user; return post; }).ToList();
    }
}
15. SplitOn distinguish type Mapping group
Split Default is used to cut the primary key, so default cut string is ID, if the table structure PK name is ID can omit parameters, for example

var result = cn.Query<Order,User,Order>(@"
  select * from [order] T1
  left join [User] T2 on T1.UserId = T2.ID    
  ", (order, user) => { 
    order.User = user;
    return order;
  }
);
If the primary key name is another name, specify the splitOn string name and it corresponds to multiple names, it can be used ,as a segmentation. For example, add a product table as Join:

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
16. Query Multi Mapping underlying principle
First, a simple Demo.  

Create a Mapping FUNC collection corresponding to the number of generic class parameters

The Mapping FUNC setup logic is the same as Query Emit IL

Call the user's Custom Mapping Func, where the parameters are derived from the previously dynamically generated Mapping Func

public static class MutipleMappingDemo
{
    public static IEnumerable<TReturn> Query<T1, T2, TReturn>(this IDbConnection connection, string sql, Func<T1, T2, TReturn> map)
       where T1 : Order, new()
       where T2 : User, new() 
    {
        // 1. Create a Mapping FUNC collection corresponding to the number of generic class parameters
        var deserializers = new List<Func<IDataReader, object>>();
        {
            // 2. The Mapping FUNC setup logic is the same as Query Emit IL
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
​
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
​
​
        using (var command = connection.CreateCommand())
        {
            command.CommandText = sql;
            using (var reader = command.ExecuteReader())
            {
                while (reader.Read())
                {
                    // 3. Call the user's Custom Mapping Func, where the parameters are derived from the previously dynamically generated Mapping Func
                    yield return map(deserializers[0](reader) as T1, deserializers[1](reader) as T2);
                }
            }
        }
    }
}
Support multiple groups of type + strongly typed return values
Dapper using multiple generic parameter methods for strongly typed multi-class Mapping has disadvantage that it can not be dynamically adjusted and needs to be fixed.

For example, you can see that the image GenerateMapper method fix the strong transition logic in terms of the number of generic arguments, which is why Multiple Query has a maximum number of groups and can only support up to six.

image

Multi-Class generic caching algorithm
Dapper use Generic Classto save multiple types of data by strong-type
20191001175139.png

And cooperate with inheritance to share most of the identity verification logic

Provide available override GetType method to customize generic comparison logic to avoid non-multiple query Cache conflict.

image

image

Select order of Dapper Query Multi Mapping is important
Because of SplitOn group logic depend on Select Order, it is possible that attribute value wrong when sequence is wrong .

Example: If the SQL in the above example is changed to the following, the ID of User will become the ID of Order; the ID of Order will become the ID of User.

select T2.[ID],T1.[OrderNo],T1.[UserID],T1.[ID],T2.[Name] from [order] T1
left join [User] T2 on T1.UserId = T2.ID  
The reason can be traced to Dapper's cutting algorithm

First, the field group by reverse order, the GetNextSplit method can be seen DataReader Index  from large to small.  
image

Then process the Mapping Emit IL Func of the type in reverse order

Finally, it is reversed to positive order, which is convenient for the use of Call Func corresponding to generics later.

image

image

image

17. QueryMultiple underlying logic
Example:

  using (var cn = Connection)
  {
    using (var gridReader = cn.QueryMultiple("select 1; select 2;"))
    {
      Console.WriteLine(gridReader.Read<int>()); //result : 1
      Console.WriteLine(gridReader.Read<int>()); //result : 2
    }
  }
Advantages of using QueryMultiple:

Mainly reduce the number of Reqeust

Multiple queries can share the same set of parameter 

The underlying implementation logic of QueryMultiple:

The underlying technology is ADO.NET-DataReader-MultipleResult

QueryMultiple gets DataReader and encapsulates it into GridReader

The Mapping dynamic method is only created when the Read method is called, and the Emit IL action is the same as the Query method.

Then call ADO.NET DataReader NextResult to get the next set of query result

DataReader will be released if there is no next set of query results

Cache algorithm
The caching algorithm adds more gridIndex judgments, mainly for each result mapping action as a cache.

image

No delayed query feature
Note that the Read method uses buffer = true the returned result is directly stored in the ToList memory, so there is no delayed query feature.

image

image

Remember to manage the release of DataReader
When Dapper calls the QueryMultiple method, the DataReader is encapsulated in the GridReader object, and the DataReader will be recycled only after the last Read action. 

image

Therefore, if you open a GridReader > Read before finishing reading, an error will show: a DataReader related to this Command has been opened, and it must be closed first.

To avoid the above situation, you can change to the using block, and the DataReader will be automatically released after running the block code.

18. TypeHandler custom Mapping logic & its underlying logic
When you want to customize some attribute Mapping logic, you can use TypeHandler in Dapper

Create a class to inherit SqlMapper.TypeHandler

Assign the class to be customized to the generic, e.g: JsonTypeHandler<custom_type>: SqlMapper.TypeHandler<custom_type>

Override Parse method to custom Query logic, and override  SetValue method to custom Create,Delete,Updte  logic

If there are multiple class Parse and SetValue share the same logic, you can change the implementation class to a generic method. The custom class can be specified in AddTypeHandler, which can avoid creating a lot of class, eg: JsonTypeHandler<T>: SqlMapper.TypeHandler< T> where T: class

Example : when User level is changed, the change action will be automatically recorded in the Log field.

public class JsonTypeHandler<T> : SqlMapper.TypeHandler<T> 
  where T : class
{
  public override T Parse(object value)
  {
    return JsonConvert.DeserializeObject<T>((string)value);
  }
​
  public override void SetValue(IDbDataParameter parameter, T value)
  {
    parameter.Value = JsonConvert.SerializeObject(value);
  }
}
​
public void Main()
{
  SqlMapper.AddTypeHandler(new JsonTypeHandler<List<Log>>()); 
​
  using (var ts = new TransactionScope())
  using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=master;"))
  {
​
    cn.Execute("create table [User] (Name nvarchar(200),Age int,Level int,Logs nvarchar(max))");
​
    var user = new User()
    {
      Name = "Wei",
      Age = 26,
      Level = 1,
      Logs = new List<Log>() {
        new Log(){Time=DateTime.Now,Remark="CreateUser"}
      }
    };
​
    // add
    {
      cn.Execute("insert into [User] (Name,Age,Level,Logs) values (@Name,@Age,@Level,@Logs);", user);
​
      var result = cn.Query("select * from [User]");
      Console.WriteLine(result);
    }
​
    // Level up
    {
      user.Level = 9;
      user.Logs.Add(new Log() {Remark="UpdateLevel"});
      cn.Execute("update [User] set Level = @Level,Logs = @Logs where Name = @Name", user);
      var result = cn.Query("select * from [User]");
      Console.WriteLine(result);
    }
​
    ts.Dispose();
​
  }
}
​
public class User
{
  public string Name { get; set; }
  public int Age { get; set; }
  public int Level { get; set; }
  public List<Log> Logs { get; set; }
​
}
public class Log
{
  public DateTime Time { get; set; } = DateTime.Now;
  public string Remark { get; set; }
}
image

Then trace the TypeHandler source code logic, which needs to be traced in two parts: SetValue, Parse

The underlying logic of SetValue
AddTypeHandlerImpl method to manage the addition of cache  

When creating a dynamic AddParameter method in the CreateParamInfoGenerator method Emit, if there is data in the TypeHandler cache of the Mapping type, Emit adds an action to call the SetValue method.

if (handler != null)
{
  il.Emit(OpCodes.Call, typeof(TypeHandlerCache<>).MakeGenericType(prop.PropertyType).GetMethod(nameof(TypeHandlerCache<int>.SetValue))); // stack is now [parameters] [[parameters]] [parameter]
}
LookupDbType will be used when calling the AddParameters method at Runtime to determine whether there is a custom TypeHandler

image

image

Then pass the created Parameter to the custom TypeHandler.SetValue method

image

Finally, the C# code converted from IL

    public static void TestMeThod(IDbCommand P_0, object P_1)
    {
        User user = (User)P_1;
        IDataParameterCollection parameters = P_0.Parameters;
        //...
        IDbDataParameter dbDataParameter3 = P_0.CreateParameter();
        dbDataParameter3.ParameterName = "Logs";
        dbDataParameter3.Direction = ParameterDirection.Input;
        SqlMapper.TypeHandlerCache<List<Log>>.SetValue(dbDataParameter3, ((object)user.Logs) ?? ((object)DBNull.Value));
        parameters.Add(dbDataParameter3);
        //...
    }
It can be found that the generated Emit IL will get our implemented TypeHandler from TypeHandlerCache, and then call the implemented SetValue method to run the set logic, and TypeHandlerCache uses generic type to save different handlers in Singleton mode according to different generics. This has the following advantages :

Same handler can be obtained to avoid repeated creation of objects

Because it is a generic type, reflection actions can be avoided when the handler is taken, and efficiency can be improved

image
image
image

Parse corresponds to the underlying principle
The main logic is when the GenerateDeserializerFromMap method Emit establishes the dynamic Mapping method, if it is judged that the TypeHandler cache has data, the Parse method replaces the original Set attribute action.

image

View the IL code generated by the dynamic Mapping method:

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
Convert it into C# code to verify:

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
      //..
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
19. Detailed processing of CommandBehavior
This article will take readers to understand how Dapper uses CommandBehavior to optimize query efficiency, and how to choose the correct Behavior at a specific time.

I have compiled the Behavior table corresponding to each method here:

METHOD	BEHAVIOR
Query	CommandBehavior.SequentialAccess & CommandBehavior.SingleResult
QueryFirst	CommandBehavior.SequentialAccess & CommandBehavior.SingleResult & CommandBehavior.SingleRow
QueryFirstOrDefault	CommandBehavior.SequentialAccess & CommandBehavior.SingleResult & CommandBehavior.SingleRow
QuerySingle	CommandBehavior.SingleResult & CommandBehavior.SequentialAccess
QuerySingleOrDefault	CommandBehavior.SingleResult & CommandBehavior.SequentialAccess
QueryMultiple	CommandBehavior.SequentialAccess
SequentialAccess, SingleResult optimization logic
First, you can see that each method uses CommandBehavior.SequentialAccess. The main function of this tag is to make the DataReader read rows and columns sequentially without buffering. After reading a column, it will be deleted from memory.  It has the following advantages:

Resources can be read in order to avoid binary large resources from being read into memory at one time, especially Blob or Clob will cooperate with GetBytes or GetChars methods to limit the buffer size, Microsoft officials also have special attention:

image

Actual environment testing show it can speed up query efficiency. But it is not the default behavior of DataReader, the system default is CommandBehavior.Default

image

CommandBehavior.Default has the below behaviors:

Can return multiple result sets 

Read row data to memory at once

These two features are much different from the production environment. After all, most of the time,only a set of result sets are needed with limited memory, so in addition to SequentialAccess, Dapper also uses CommandBehavior.SingleResult in most methods, so that only one set of results is required. To avoid wasting resources.

There is also a detailed processing of this paragraph. Looking at the source code, you can find that in addition to marking SingleResult, Dapper also specially adds code at the end while(reader.NextResult()){} instead of directly Return (such as the picture)

image

Earlier, I specifically posted an Issue (Link #1210) to ask author, here is the answer: mainly to avoid ignoring errors, such as when the DataReader is closed early.

QueryFirst with SingleRow
Sometimes we will encounter a situation where select top 1 knows that only one row of data will be read. At this time, QueryFirst can be used. It uses CommandBehavior.SingleRow to avoid wasting resources and only read one row of data.

In addition, it can be found that in addition to while (reader.NextResult()){}, Dapper also has while (reader.Read()) {}, which is also to avoid ignoring errors. This is something that some companies’ self-make ORMs will ignore.

image

Differences with QuerySingle
The difference between the two is that QuerySingle does not use CommandBehavior.SingleRow. As for why it is not used, it is because multiple rows of data are needed to determine whether the conditions are not met and an Exception is thrown to inform the user.

There is a particularly fun trick to learn in Dapper. The error handling directly uses the Exception corresponding to LINQ. For example: more than one line of data is wrong, use new int[2].Single(), so you don't need to maintain the Exception class separately, and you can also have more i18N language message.

image

image

20. The underlying logic of Parameter parameterization
One key function of Dapper: "Parameterization"

Main logic: GetCacheInfo checks whether there is a dynamic method in the cache >  If there is no cache, use the CreateParamInfoGenerator method Emit IL to create the AddParameter dynamic method >  Save it in the cache after creation

Next, we will focus on the underlying logic and "exquisite detail processing" in the CreateParamInfoGenerator method, using the result reverse code method, ignoring the "unused fields" and not generating the corresponding IL code to avoid resource waste. This is also the reason why the previous caching algorithm has to check different SQL strings.

The following are the key parts of the source code I picked:

internal static Action<IDbCommand, object> CreateParamInfoGenerator(Identity identity, bool checkForDuplicates, bool removeUnused, IList<LiteralToken> literals)
{
  //...
  if (filterParams)
  {
    props = FilterParameters(props, identity.sql);
  }
​
  var callOpCode = isStruct ? OpCodes.Call : OpCodes.Callvirt;
  foreach (var prop in props)
  {
    //Emit IL action
  }
  //...
}
​
​
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
Then check IL to verify, the query code is as follows

var result = connection.Query("select @Name name ", new { Name = "Wei", Age = 26}).First();
The IL code of the CreateParamInfoGenerator AddParameter dynamic method is as below:

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
IL converted to C# code:

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
It can be found that although the Age parameter is passed, the SQL string is not used, and Dapper will not generate the SetParameter action IL for this field. This detail processing really needs to give Dapper a thumbs up!

21. The underlying logic of IN multi-set parameterization
Why ADO.NET does not support IN parameterization, but Dapper does?

Check whether the attribute of the parameter is a subclass of IEnumerable 

If yes, use the parameter name + regular format to find the parameter string in SQL (regular format: ([?@:]Parameter name)(?!\w)(\s+(?i)unknown(?- i))?)

Replace the found string with () + multiple attribute names + serial number

CreateParameter> SetValue in order of serial number

Key Code part

image

The following uses sys.objects to check SQL Server tables and views as an example of tracking:

var result = cn.Query(@"select * from sys.objects where type_desc In @type_descs", new { type_descs = new[] { "USER_TABLE", "VIEW" } });
Dapper will change the SQL string to the following sql to execute

select * from sys.objects where type_desc In (@type_descs1,@type_descs2)
-- @type_descs1 = nvarchar(4000) - 'USER_TABLE'
-- @type_descs2 = nvarchar(4000) - 'VIEW'
Looking at Emit IL, you can find that it is very different from the previous parameterized IL, which is very clean.

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
Turning to C# code, you will be surprised to find: This code does not need to use Emit IL at all. It is simply unnecessary.

    public static void TestMeThod(IDbCommand P_0, object P_1)
    {
        var anon = (<>f__AnonymousType0<string[]>)P_1;
        IDataParameterCollection parameter = P_0.Parameters;
        SqlMapper.PackListParameters(P_0, "type_descs", anon.type_descs);
    }
That's right, it is unnecessary, even IDataParameterCollection parameter = P_0.Parameters; this code will not be used at all.

There is a reason for Dapper, because it can be used with non-collective parameters, such as the previous example and the data logic to find the name of the order.

var result = cn.Query(@"select * from sys.objects where type_desc In @type_descs and name like @name"
    , new { type_descs = new[] { "USER_TABLE", "VIEW" }, @name = "order%" });
The corresponding generated IL conversion C# code will be the following code, which can be used together:

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
In addition, why does Emit IL directly call the tool method PackListParameters on Dapper? Because the number of IN parameters is not fixed, the method cannot be dynamically generated from the fixed result.

The main logic contained in this method:

Determine the type of the set parameter (if it is a string, the default size is 4000)

Regular role of SQL parameters are replaced with serial number parameter strings

Creation of DbCommand Paramter

image

The replacement logic of the SQL parameter string is also written here, such as the picture

image

22. DynamicParameter underlying logic and custom implementation
For example:  

using (var cn = Connection)
{
    var paramter = new { Name = "John", Age = 25 };
    var result = cn.Query("select @Name Name,@Age Age", paramter).First();
}
We already know that String type Dapper will automatically convert to database Nvarchar and a parameter with a length of 4000. The SQL actually executed by the database is as below:

exec sp_executesql N'select @Name Name,@Age Age',N'@Name nvarchar(4000),@Age int',@Name=N'John',@Age=25
This is an intimate design that is convenient for rapid development, but if you encounter a situation where the field is of varchar type, it may cause the index to fail due to the implicit transformation, resulting in low query efficiency.

At this time, the solution can use Dapper DynamicParamter to specify the database type and size to achieve the purpose of optimizing performance

using (var cn = Connection)
{
    var paramters = new DynamicParameters();
    paramters.Add("Name","John",DbType.AnsiString,size:4);
    paramters.Add("Age",25,DbType.Int32);
    var result = cn.Query("select @Name Name,@Age Age", paramters).First();
}
Then go to the source to see how to implement it. First, pay attention to the GetCacheInfo method. You can see that DynamicParameters create a dynamic method. The code is very simple, just call the AddParameters method.

Action<IDbCommand, object> reader;
if (exampleParameters is IDynamicParameters)
{
    reader = (cmd, obj) => ((IDynamicParameters)obj).AddParameters(cmd, identity);
}
The reason why the code can be so simple is that Dapper uses an "interface-dependent" design here to increase the flexibility of the program and allow users to customize the implementation logic they want. This point will be explained below. First, let's look at the implementation logic of the AddParameters method in Dapper's default implementation class DynamicParameters.

public class DynamicParameters : SqlMapper.IDynamicParameters, SqlMapper.IParameterLookup, SqlMapper.IParameterCallbacks
{
    protected void AddParameters(IDbCommand command, SqlMapper.Identity identity)
    {
        var literals = SqlMapper.GetLiteralTokens(identity.sql);
​
        foreach (var param in parameters.Values)
        {
            if (param.CameFromTemplate) continue;
​
            var dbType = param.DbType;
            var val = param.Value;
            string name = Clean(param.Name);
            var isCustomQueryParameter = val is SqlMapper.ICustomQueryParameter;
​
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
​
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
​
                if (add)
                {
                    command.Parameters.Add(p);
                }
                param.AttachedParam = p;
            }
        }
​
        // note: most non-priveleged implementations would use: this.ReplaceLiterals(command);
        if (literals.Count != 0) SqlMapper.ReplaceLiterals(this, command, literals);
    }
}
It can be found that Dapper has made many conditions and actions in AddParameters for convenience and compatibility with other functions, such as Literal Replacement and EnumerableMultiParameter functions, so the amount of code will be more than the previous version of ADO.NET, so the efficiency will be slower.

If you have demanding requirements for efficiency, you can implement the logic yourself, because this section of Dapper is specially designed to "depend on the interface", and you only need to implement the IDynamicParameters interface.

The following is a demo I made, you can use ADO.NET SqlParameter to establish parameters to cooperate with Dapper

public class CustomPraameters : SqlMapper.IDynamicParameters
{
  private SqlParameter[] parameters;
  public void Add(params SqlParameter[] mParameters)
  {
    parameters = mParameters;
  }
​
  void SqlMapper.IDynamicParameters.AddParameters(IDbCommand command, SqlMapper.Identity identity)
  {
    if (parameters != null && parameters.Length > 0)
      foreach (var p in parameters)
        command.Parameters.Add(p);
  }
}
image

23. The underlying logic of single and multiple Execute
After the Query, Mapping, and Parameters are explained, we will then explain that use the Execute method in adding, deleting, and modifying by Dapper. Execute Dapper is divided into single execute andmultiple execute`.

Single Execute
In terms of a single execution, the logic of Dapper is the encapsulation of ADO.NET's ExecuteNonQuery. The purpose of encapsulation is to be used with Dapper's Parameter and caching functions. The code logic is concise and clear. There is no more explanation here, such as the picture

image

"Multiple" Execute
This is a characteristic feature of Dapper, which simplifies the operations between the collection operations Execute and simplifies the code. Only: connection.Execute("sql",collection parameters);.

Why it is so convenient, the following is the underlying logic:

Confirm whether it is a collection parameter

image

Create a common DbCommand to provide foreach iterative call to avoid repeated create and waste of resources

image

If it is a set of parameters, create an Emit IL dynamic method and put it in the cache for use

image

The dynamic method logic is CreateParameter> Assign Parameter> Use Parameters.Add to add a new parameter. The following is the C# code converted by Emit IL:

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
foreach the set of parameters> except for the first time, clear the DbCommand parameters in each iteration> re-call the same dynamic method to add parameters> reqeust SQL query

The implementation method is simple and clear, and the details consider sharing resources to avoid waste (eg sharing the same DbCommand, Func), but in the case of a large number of execution pursuit efficiency requirements, you need to pay special attention to this method every time you run to send a request to the database, the efficiency will be the network transmission is slow, so this function is called "multiple execution" instead of "batch execution".

For example, simple Execute inserts ten pieces of data, and you can see that the system has received 10 Reqeust when viewing SQL Profiler:

using (var cn = new SqlConnection(@"Data Source=(localdb)\MSSQLLocalDB;Integrated Security=SSPI;Initial Catalog=Northwind;"))
{
    cn.Open();
    using (var tx = cn.BeginTransaction())
    {
        cn.Execute("create table #T (V int);", transaction: tx);
        cn.Execute("insert into #T (V) values (@V)", Enumerable.Range(1, 10).Select(val => new { V = val }).ToArray() , transaction:tx);
​
        var result = cn.Query("select * from #T", transaction: tx);
        Console.WriteLine(result);
    }
}
image

24. ExecuteScalar
ExecuteScalar is an often forgotten function because it can only read the first set of results, the first row, and the first data. However, it can still come in handy under specific needs, like "Check Existence".

First, how does Entity Framwork efficiently check whether data exists?

If the reader with EF experience will answer to use Any instead of Count()> 1. Using the Count system will help convert SQL to:

SELECT COUNT(*) AS [value] FROM [Table] AS [t0]
SQL Count is a summary function that will iterate the qualified data rows to determine whether the data in each row is null and return the number of rows.

The Any syntax conversion SQL uses EXISTS. It only cares whether there is data or not. It means that there is no need to check each column, so the efficiency is fast.

SELECT 
    (CASE 
        WHEN EXISTS(
            SELECT NULL AS [EMPTY]
            FROM [Table] AS [t0]
            ) THEN 1
        ELSE 0
     END) AS [value]
How does Dapper achieve the same effect?
SQL Server can use the SQL format select top 1 1 from [table] where conditions with the ExecuteScalar method, and then make an extension method, as follow:

public static class DemoExtension
{
  public static bool Any(this IDbConnection cn,string sql,object paramter = null)
  {
    return cn.ExecuteScalar<bool>(sql,paramter);
  }
}
image

The reason for this simple use is that Dapper ExecuteScalar will call ExecuteScalarImpl and its underlying Parse logic

private static T ExecuteScalarImpl<T>(IDbConnection cnn, ref CommandDefinition command)
{
    //..
    object result;
    //..
    result = cmd.ExecuteScalar();
    //..
    return Parse<T>(result);
}
​
private static T Parse<T>(object value)
{
    if (value == null || value is DBNull) return default(T);
    if (value is T) return (T)value;
    var type = typeof(T);
    //..
    return (T)Convert.ChangeType(value, type, CultureInfo.InvariantCulture);
}
Use Convert.ChangeType to convert to bool: "0=false, non-0=true" logic, so that the system can simply convert to bool value.

Note: Don't replace by QueryFirstOrDefault, because it requires additional Null check in SQL, otherwise "NullReferenceException" will show.

The reason is that the two Parse implementations are different, and the QueryFirstOrDefault check the result to be null.

20191003043941.png

The Parce implementation of ExecuteScalar has more check to use the default value when it is empty

image

25. Summary
The Dapper series end here, and the important underlying logics are almost finished. This series took the me 25 consecutive days. In addition to helping readers, the biggest gain is that I understand the underlying logic of Dapper better during this period and learn Dapper details and processing.

In addition, I would like to mention that Marc Gravell, one of the authors of Dapper, is really very enthusiastic. During the writing of the article, there are a few conceptual questions. If you ask an issue, he will reply enthusiastically and in detail. And he also found that he has high requirements for the quality of the code.For example: I asked a question on SO, and he left a message below: "He is actually not satisfied with the current Dapper IL architecture, and even feels it rough, and wants to use protobuf-net technology to rewrite"(Respect!)

image

Finally, I would like to say: the original intention of writing this is to hope that this series can help readers

Understand the underlying logic, know why, avoid writing monsters that eat efficiency, and take full advantage of Dapper to develop projects

You can easily face Dapper interviews, and answer deeper concepts than ordinary engineers.

From the simplest Reflection to the commonly used Expression to the most detailed Emit builds the Mapping method from scratch, taking readers to gradually understand the underlying strong type Mapping logic of Dapper

Understand the important concept of dynamic creation method "Code Converted From Result".

With basic IL capabilities, you can use IL to reverse C# code to understand the underlying Emit logic of other projects

Understand that Dapper cannot use error strings to contact SQL because of the algorithm logic of the cache

Thanks :)
