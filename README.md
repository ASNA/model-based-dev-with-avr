## Creating data models with ASNA Visual RPG


This article discusses a way to approach model-based development with ASNA Visual RPG. This type of coding has been around for a long time. Some of you may recognize some of the concepts presented here from our past advanced AVR classes. However, as you'll soon see, using data models usually requires tooling to generate the models--and that often limits its use with AVR. 

Generally, with AVR, you don't need to use a model-based approach (or a least a "pure" model-based approach). The AVR Memory File is an effective stand-in for pure modeling in many use-cases (and virtually all of them in a 100% AVR vacuum). Lately, though, we've helped several customers integrate AVR with C# (usually using .NET's MVC Web app model). When merging AVR and C# in the same project, the language boundaries start to reveal weaknesses with AVR Memory File-based IO. This is especially true when AVR programming team uses AVR to fetch data from the IBM i to pass it off to C# MVC coders. 

While the techniques presented here are especially effective in a [polyglot](https://www.techtarget.com/searchsoftwarequality/definition/polyglot-programming#:~:text=Polyglot%20programming%20is%20the%20practice,practice%20for%20enterprise%20application%20development.) programming environment, you may also want to use them in your 100% AVR projects.

---

AVR programmers, have for years, used the Visual RPG Memory File to populate user interfaces, especially grids, in both Windows fat client apps and in ASP.NET Web apps. Even beginning AVR coders are familiar with the general pattern of using a Memory File. To populate a customer grid (in either fat clients or Web apps), you read records from a file, write them to memory file, and bind the resulting DataTable as to a grid as a its DataSource.

For example, the code below populates a DataGridView in a fat client with a Memory File. 

```
DclDB pgmDB DBName("*Public/DG NET Local")

DclDiskFile  CustomerByName +
      Type(*Input) +
      Org(*Indexed) +
      File("Examples/CMastNewL2") +
      DB(pgmDB) +
      ImpOpen(*No)

DclMemoryFile CustomerByNameMF +
      DBDesc("*Public/DG NET Local") +
      FileDesc("Examples/CMastNewL2") +
      RnmFmt(RMemFile)

BegSr Form1_Load Access(*Private) Event(*this.Load)
    DclSrParm sender *Object
    DclSrParm e System.EventArgs
    
    datagridviewCustomers.AutoGenerateColumns = *False
    ReadRows()
EndSr

BegSr OpenData
    Connect pgmDB 
    Open CustomerByName 
EndSr 

BegSr CloseData
    Close *All
    Disconnect pgmDB 
EndSr

BegSr ReadRows
    OpenData()

    Do FromVal(1) ToVal(14)
        Read CustomerByName
        Write CustomerByNameMF
    EndDo 

    CloseData()

    datagridviewCustomers.DataSource = CustomerByNameMF.DataSet.Tables[0] 
EndSr
```
<small>Figure 1a. Code to populate a DataGridView using a Memory File</small>

>A GitHub repo with the example code this repo references [is available at this GitHub repo.](https://github.com/ASNA/model-based-dev-with-avr-examples)

The code above produces a Windows form that looks like this: 


![](https://asna.com/filebin/marketing/DataModelExamples_CPmVE7QKqs.png)

<small>Figure 1b. The Win form produced with the code in Figure 1a.</small>


The code above isn't complete. It arbitrarily reads 14 rows into the grid. A production-ready program needs logic to page through the data. However, the code above is quite representative of a basic Memory File pattern that hundreds of AVR coders have used in AVR apps for many, many years. 

A variation on the pattern above might use a [program-described Memory File](https://asna.com/us/tech/kb/doc/avr-memory-file). In this example, the Memory File is described by the file specified in its `FileDesc` keyword--which makes the underlying DataTable have exactly the same structure as the DclDiskFile's data file. 

The AVR Memory File is essentially a wrapper around the [System.Data.DataSet object](https://docs.microsoft.com/en-us/dotnet/api/system.data.dataset?view=netframework-4.7.1). The creamy nougat center of a DataSet is a [System.Data.DataTable](https://docs.microsoft.com/en-us/dotnet/api/system.data.datatable?view=netframework-4.7.1).The DataTable is analogous to a member in an IBM i data file. The DataTable is the data container--it is where rows written to the Memory File reside. 

The code below (from the example above) assigns the rows in the Memory File's DataSet's zeroth DataTable (which with the pattern above is the only DataTable in the DataSet) as the data source for the DataGateGridView. 

```
datagridviewCustomers.DataSource = CustomerByNameMF.DataSet.Tables[0] 
```

The strength of this Memory File/DataSet/DataTable approach is its simplicity and ease. With a 4-line Memory File declaration and about 20 lines of code, you can populate a grid in either a fat client or Web app. Let that soak in. How many lines of code does it take to populate a subfile with ILE RPG? Fuhgettaboutit! 

The schema shown below in Figure 2a defines the data file used in Figure 1a. 

```
Database Name.: *PUBLIC/DG NET Local
Library.......: Examples
File..........: CMastNewL2
File alias....: CustomerByName
Format........: RCMMastL2
Type..........: Simple logical
Base file.....: Examples/CMastNew
Description...: CustomerByName
Record length.: 151
Key length....: 45
Key field(s)..: CMName, CMCustNo

Field name           Data type   Length  Decimals  Description
----------------------------------------------------------------------------
 CMCustNo            Packed          9        0    Customer number 
 CMName              Char           40             Name 
 CMAddr1             Char           35             Address 
 CMCity              Char           30             City
 CMState             Char            2             State
 CMCntry             Char            2             Country
 CMPostCode          Char           10             Postal code
 CMActive            Char            1             1=active, 0=inactive
 CMFax               Packed         10        0    Fax number
 CMPhone             Char           20             Phone number
----------------------------------------------------------------------------

```
<small>Figure 2a. The file layout for the data file used in Figure 1a.</small>

The Memory File in Figure 1a automatically creates a buffer (the DataTable) based on the file layout. With field names in the buffer identical to that in the data file, this enables AVR to read from the data file and populate a Memory File easily:

```
Read CustomerByName
Write CustomerByNameMF
```

For all intents and purposes, the Memory File/DataSet/DataTable approach is a model-based approach, with the underlying DataTable being the model. The DataTable layout from Figure 1a is exactly as you see in Figure 2a's file layout.

Don't let anyone tell you that DataSet/DataTable-based coding is bad. It's not. You can create supremely clever business solutions with this model. 

### A model-based approach

That said, time, and technology, march onward. Nearly all Microsoft tutorials lately eschew the DataSet/DataTable-based model for a more pure, model-based approach. They did that because DataTables are not strongly typed--and therefore their contents are not discoverable at runtime by .NET. To get data out of a DataTable, you have to explicitly interrogate its `ItemArray` property and you have to cast the weakly typed value like this:

```
Value = dr.ItemArray[i] *As *Integer4 

or

Value = dr.ItemArray["cmcustno"] *As *Integer4 
```

A model-based approach is discoverable by .NET and is strongly-typed. Intellisense proves this; you can see and work with the contents of a strongly-type object at development time. 

![](https://asna.com/filebin/marketing/devenv_AGecjr4Fz2.png)

The MVC model does a similar discovery at development time and, perhaps even more importantly, at runtime ([using reflection](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/reflection)), for tasks like populating a Web page at runtime. 

Let's get a few more model basics under our belt. What does a model-based approach mean? It means that a "model," created with code, defines the data buffer needed. For example, a way to code the model for the file defined by Figure 2 with ASNA Visual, is shown below in Figure 2b. (Columns are aligned to make it easy to compare Figure 2a and Figure 2b.)

```
BegClass CustomerModel Access(*Public)
    DclProp CMCustNo   Type(System.Decimal) Access(*Public)
    DclProp CMName     Type(System.String)  Access(*Public)
    DclProp CMAddr1    Type(System.String)  Access(*Public)
    DclProp CMCity     Type(System.String)  Access(*Public)
    DclProp CMState    Type(System.String)  Access(*Public)
    DclProp CMCntry    Type(System.String)  Access(*Public)
    DclProp CMPostCode Type(System.String)  Access(*Public)
    DclProp CMActive   Type(System.String)  Access(*Public)
    DclProp CMFax      Type(System.Decimal) Access(*Public)
    DclProp CMPhone    Type(System.String)  Access(*Public)
EndClass
```

<small>Figure 2b. An AVR custom model to define the file shown in Figure 2a.</small>

Dispense for a few minutes with the realization that to manually create a buffer like this for a 100 (or 500! We've seen em!) field file would be an excruciating job. You need some kind of tooling (a fancy word for code generator) to produce accurate and reliable data models. For now, let's pretend like you had a magic wand that could create a buffer like this for any file of yours in about 2 seconds (no fair peeking ahead!)

>You'll probably quickly notice that the data types from Figure 2b are different from those in Figure 2a. Or are they? They are not, AVR's `*Packed` data type is simply an alias for .NET's `System.Decimal` type. You can use either in your AVR code. See the [table here for a list that maps](https://asna.com/us/tech/kb/doc/avr-data-types) AVR data types to .NET data types.

### Getting a list of models

A model-based approach needs two things: the model, as shown above. But it also needs a list of models, for data binding to a grid, passing many data rows from a Web service, or many other things. For that, a couple of approaches are possible. One thing you can do is create an array of the CustomerModel class, like this:

```
DclArray CustomerModelList Type(CustomerModel) Rank(1)
```

> [See this link if you aren't familiar with ranked arrays.](https://asna.com/us/tech/kb/series/avr-rpg-arrays/dynamic-arrays)

An array of models is often a good approach, but it's not easy to add elements to an array on the fly. Rather than use an array we'll use a .NET generic collection, in this case [we'll use a `List of <T>` also known as `List<T>`](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=net-6.0). (In your head say "List of tee"). Looks weird, right? Don't let it scare you, the `<T>` is shorthand for "a specific type." The `List<T>` object lives in .NET's System.Collection.Generic namespace. The following AVR code defines a list of CustomerModel objects:

```
using System.Collection.Generic 

// Instance the list with the declaration:
DclFld Customers Type(List(*Of CustomerModel)) New()

or 

using System.Collection.Generic 

// Declare the list.
DclFld Customers Type(List(*Of CustomerModel)) 

...
// Instance it later in code:
Customers = *New List(*Of CustomerModel)()
```
<small>Figure 3. Declaring a strongly typed list of the CustomerModel object</small>

> Watch your parentheses when using .NET's generic objects with AVR. They look a little obtuse at first.

You can only add instances of `CustomerModel` to this list. Attempting to add object but `CustomerModel` objects won't compile. 

In much older AVR code, you may be familiar with .NET's [Arraylist](https://docs.microsoft.com/en-us/dotnet/api/system.collections.arraylist?view=net-6.0) object. It is similar to the `List<T>` object, but `ArrayList` isn't strongly typed. The ArrayList is an array of System.Objects. Therefore, you can add objects of any type to the same ArrayList--and getting an element out of the ArrayList required casting the object (as shown below in Figure 4.) 

```
using System.Collections

DclArray CustomerList Type(ArrayList)

.. 
CustomerList.add(model)   // where model is an instance of CustomerModel

model = CustomerList[0] *As CustomerModel

```
<small>Figure 4. Using the weakly-typed ArrayList.</small>

Weakly-typed lists won't work with C# and its standard MVC databinding. For that we need the strong typing that `List<T>` provides.

### Putting it all together

How does it all work? The AVR code below uses the `CustomerModel` object as an alternative to the Memory File. It populates a Windows form and looks exactly like the form shown in Figure 1b.

```
DclDB pgmDB DBName("*Public/DG NET Local")

DclDiskFile  CustomerByName +
        Type(*Input) +
        Org(*Indexed) +
        File("Examples/CMastNewL2") +
        DB(pgmDB) +
        ImpOpen(*No)

DclFld Customers Type(List(*Of DataModels.CustomerModel)) 

BegSr Form2_Load Access(*Private) Event(*this.Load)
    DclSrParm sender Type(*Object)
    DclSrParm e Type(System.EventArgs)

    datagridviewCustomers.AutoGenerateColumns = *False
    ReadRows()
EndSr

BegSr OpenData
    Connect pgmDB 
    Open CustomerByName 
EndSr 

BegSr CloseData
    Close *All
    Disconnect pgmDB 
EndSr

BegSr ReadRows
    DclFld Customer Type(DataModels.CustomerModel) 

    // Declare a list of customers
    Customers = *New List(*Of DataModels.CustomerModel)()  

    OpenData()

    Do FromVal(1) ToVal(14)
        Read CustomerByName
        // Instance a customer model for the row just read. 
        Customer = *New DataModels.CustomerModel()
        // Populate fields 
        PopulateCustomerModelFromFormat(Customer)
        Customers.Add(Customer) 
    EndDo 

    CloseData()
    
    datagridviewCustomers.DataSource = Customers
EndSr

BegSr PopulateCustomerModelFromFormat Access(*Public)
    DclSrParm Model Type(DataModels.CustomerModel)

    Model.CMCustNo = CMCustNo
    Model.CMName = CMName
    Model.CMAddr1 = CMAddr1
    Model.CMCity = CMCity
    Model.CMState = CMState
    Model.CMCntry = CMCntry
    Model.CMPostCode = CMPostCode
    Model.CMActive = CMActive
    Model.CMFax = CMFax
    Model.CMPhone = CMPhone
EndSr
```
<small>Figure 5. AVR code using the CustomerModel objects</small>

While this is a 100% AVR example, it would be easy to pass its `Customers` list (declared locally in this example in the `ReadRows` method) to C#. That list would exactly like a C# would hope it would in an MVC application. 

The code in figure 5 further reveals the need for tooling. Producing the routine to populate a customer model (shown below in the Figure 6) is a pretty simple job for a demo file. But for complex, big production models, trying to create the code you need by hand is quite challenging (and probably not really doable).

```
BegSr PopulateCustomerModelFromFormat Access(*Public)
    DclSrParm Model Type(DataModels.CustomerModel)

    Model.CMCustNo = CMCustNo
    Model.CMName = CMName
    Model.CMAddr1 = CMAddr1
    Model.CMCity = CMCity
    Model.CMState = CMState
    Model.CMCntry = CMCntry
    Model.CMPostCode = CMPostCode
    Model.CMActive = CMActive
    Model.CMFax = CMFax
    Model.CMPhone = CMPhone
EndSr
```

### The tooling 

A model-based approach requires tooling to generate code for you. Microsoft provides such tooling in [].NET's Entity Framework](https://docs.microsoft.com/en-us/ef/). Many other popular Web frameworks in other languages also provide such tooling. 

To help bootstrap you into model-based coding, CreateDGDataModel is an AVR console application that creates a model class for a given table. To be clear, this is a hack tool of the highest order! It grew out of needing several models for a very specific customer project. Time was tight and we needed many large models to be consumed by C# and an MVC application. Despite its hacky nature, the tool makes possible what otherwise probably isn't a rational pursuit by hand. 

CreateDGDataModel is run from the command line.

![](https://asna.com/filebin/marketing/cmd_kX3lFSLVzS.png)

It requires four arguments: 

1. -d or --database-name 
2. -l or --library-name
3. -f or --file-name 
4. -c or --class-name

Using the same table as described in Figure 2a, CreateDGDataModel produces this source: 

```
Using System
Using System.Data
Using System.Reflection

DclNameSpace DataModels

// This class was created with the CreateDGModel utility.

BegClass CustomerModel Access(*Public)

    DclProp CMCustNo Type(System.Decimal) Access(*Public)
    DclProp CMName Type(System.String) Access(*Public)
    DclProp CMAddr1 Type(System.String) Access(*Public)
    DclProp CMCity Type(System.String) Access(*Public)
    DclProp CMState Type(System.String) Access(*Public)
    DclProp CMCntry Type(System.String) Access(*Public)
    DclProp CMPostCode Type(System.String) Access(*Public)
    DclProp CMActive Type(System.String) Access(*Public)
    DclProp CMFax Type(System.Decimal) Access(*Public)
    DclProp CMPhone Type(System.String) Access(*Public)

/*
These getter and setter methods won't compile because they rely
on global field names -- which generally come from a locally-scoped
DclDiskFile's record format. You can either build on this class by 
adding a DclDiskFile and correpsonding IO operations to it 
or cut and paste the getter and setter methods to a class that 
does have the necessary DclDiskFile.

In Figure 5, you can see that I copied the `PopulateCustomerModelFromFormat` method into that code. 

    BegSr PopulateCustomerModelFromFormat Access(*Public)
        DclSrParm Model Type(DataModels.CustomerModel)

        Model.CMCustNo = CMCustNo
        Model.CMName = CMName
        Model.CMAddr1 = CMAddr1
        Model.CMCity = CMCity
        Model.CMState = CMState
        Model.CMCntry = CMCntry
        Model.CMPostCode = CMPostCode
        Model.CMActive = CMActive
        Model.CMFax = CMFax
        Model.CMPhone = CMPhone
    EndSr


    BegSr PopulateFormatFromCustomerModel Access(*Public)
        DclSrParm Model Type(DataModels.CustomerModel)
        CMCustNo = Model.CMCustNo
        CMName = Model.CMName
        CMAddr1 = Model.CMAddr1
        CMCity = Model.CMCity
        CMState = Model.CMState
        CMCntry = Model.CMCntry
        CMPostCode = Model.CMPostCode
        CMActive = Model.CMActive
        CMFax = Model.CMFax
        CMPhone = Model.CMPhone
    EndSr

*/
EndClass
        
```
<small>Figure 6. Code produced by CreateDGDataModel</small>

CreateDGDataModel creates two important chunks of code: 

##### 1. It produces the model definition  

This provides the main body of the data model--the file buffer. 

```
DclProp CMCustNo Type(System.Decimal) Access(*Public)
DclProp CMName Type(System.String) Access(*Public)
DclProp CMAddr1 Type(System.String) Access(*Public)
DclProp CMCity Type(System.String) Access(*Public)
DclProp CMState Type(System.String) Access(*Public)
DclProp CMCntry Type(System.String) Access(*Public)
DclProp CMPostCode Type(System.String) Access(*Public)
DclProp CMActive Type(System.String) Access(*Public)
DclProp CMFax Type(System.Decimal) Access(*Public)
DclProp CMPhone Type(System.String) Access(*Public)
```
##### 2. It produces model "setter" and "getter" routines 
 
It's not enough to produce the buffer, you also need routines to move data to a model and move it from a model. These two routines do that. Alas, because of the global nature of an AVR DclDiskFile, there isn't an effective to pass reference of the record format to these routines. You've got two choices to resolve this: either copy the getter and settings to your AVR class where the DclDiskFile is declared or use this class as a bootstrap class and the DclDiskFile and the necessary file IO routines to the class. 

```
BegSr PopulateCustomerModelFromFormat Access(*Public)
    DclSrParm Model Type(DataModels.CustomerModel)

    Model.CMCustNo = CMCustNo
    Model.CMName = CMName
    Model.CMAddr1 = CMAddr1
    Model.CMCity = CMCity
    Model.CMState = CMState
    Model.CMCntry = CMCntry
    Model.CMPostCode = CMPostCode
    Model.CMActive = CMActive
    Model.CMFax = CMFax
    Model.CMPhone = CMPhone
EndSr

BegSr PopulateFormatFromCustomerModel Access(*Public)
    DclSrParm Model Type(DataModels.CustomerModel)
    CMCustNo = Model.CMCustNo
    CMName = Model.CMName
    CMAddr1 = Model.CMAddr1
    CMCity = Model.CMCity
    CMState = Model.CMState
    CMCntry = Model.CMCntry
    CMPostCode = Model.CMPostCode
    CMActive = Model.CMActive
    CMFax = Model.CMFax
    CMPhone = Model.CMPhone
EndSr
```

I told you it was hacky! But for AVR coders needing to move data to and from C# for MVC (and other) uses, it is quite serviceable. 

As I said at the beginning, a model-based isn't necessary to be highly productive with ASNA Visual RPG. It's proven, and relatively simple, Memory File/DataSet model does a great job within the boundaries of AVR. However, if you're crossing language barriers working with a C# team or using some C# yourself, sometimes you need to use a model-based approach. Now you can! 
