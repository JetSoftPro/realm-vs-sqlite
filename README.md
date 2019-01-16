![Realm vs SQLite](https://github.com/lHubertl/realm-vs-sqlite/blob/master/Matterials/Databases.jpg)

# Realm vs SQLite: the Battle of Databases

Storing mobile application data is the task we face in almost every project here at [JetSoftPro](http://www.jetsoftpro.com/). That’s why we know that choosing a database management system is vital. Some developers prefer **Realm**, while others stick to **SQLite**. But which one is better for mobile apps? We’ve conducted several tests to compare these two database management systems. Read on to discover our results and find out whether Realm or SQLite suits your business goals better.  

### Meet the Contestants
**Realm** is an open-source database management system designed specifically for mobile apps. Realm is a non-relational database which means it stores data as a collection of JSON objects. It’s fast and simple, has extensive documentation and support. Today, Realm is not just a database - it has become a full-fledged solution thanks to many additional features like API management, encryption support and background synchronization.

**SQLite** is an open-source database which is extremely popular around the world. That’s because SQLite is very lightweight, easy-to-deploy and requires minimal configuration. On top of that, it is free to use and is not owned by any company. SQLite is a relational database, which means it stores data as rows in the tables.

## JetSoftPro’s Database Test 
To explore the differences between Realm and SQLite, we’ve created a simple test application using Xamarin Forms and used iOS (iPhone 8) to test it. We tested both databases against the following criteria:
- Performance;
- Concurrency;
- Filtering and fetching;
- CRUD per second;
- Encryption;
- Foreign key;
- Maintainable by other developers.

After that, we’ve compared the results to find out who’s the winner.

## Our Application’s Architecture
Before getting to the test results, let’s talk about the architecture of our test application. We’ve made it easy to switch between database structures. To use another type of database, all you had to do was register another data repository in your dependency injection service. That’s why we needed different repositories for different databases in a single interface. 
 
This is what our [apps’ structure](https://github.com/JetSoftPro/realm-vs-sqlite/tree/master/mobileDbs) looked like. 

- **Domain** stores all models and services, being the bridge between the client app and the databases;
- **Infrastructure** stores all interfaces and different databases;
- **MobileDbs.Domain.Models** are the models used by client projects;
- **MobileDbs.Domain.Services** are the services used by clients. They provide a resolved data repository through dependency injection;
- **MobileDbs.Domain.Infrastructure** stores all interfaces;
- **MobileDbs.Domain.Realm** and **MobileDbs.Domain.SQLite** projects store all data repositories and data transfer objects. They also include Realm and SQLite managers to store the basic configuration like encryption;
- The others are shared and client projects.

## CRUD Operations 
Now, let’s look at the implementation of CRUD (Create, Read, Update, Delete) operations. Create, Read, Update, Delete are the four basic things you can do with any database. Here are the differences between implementing them in Realm and SQLite.

### SQLite: Create
```csharp
public async Task<IResponse> CreateAsync(CustomerModel customer)
{
    var customerDto = customer.ToDto();

    if (_sqliteManager.DataBase == null)
        return new Response(false);

    await _sqliteManager.DataBase.InsertAsync(customerDto);
   
    return new Response(true);
}
```

### Realm: Create 
```csharp
public async Task<IResponse> CreateAsync(CustomerModel customer)
{
    var customerDto = customer.ToDto();

    var realmInstance = await _realmManager.GetInstanceAsync();
    if (realmInstance == null)
        return new Response(false, "Realm instance can't be null");

    await realmInstance.WriteAsync(realm =>
    {
       realm.Add(customerDto, false);
       
    });
    return new Response(true);
}
```

### SQLite: Read
```csharp
public async Task<IDataResponse<IEnumerable<CustomerModel>>> ReadAsync()
{
    if (_sqliteManager.DataBase == null)
        return null;

    List<CustomerModelDto> resultDto;

    resultDto = await _sqliteManager.DataBase.Table<CustomerModelDto>().ToListAsync();

    var result = resultDto.ToModel();
    return new DataResponse<IEnumerable<CustomerModel>>(result, true);
}
```

### Realm: Read  
```csharp
public async Task<IDataResponse<IEnumerable<CustomerModel>>> ReadAsync()
{
    IEnumerable<CustomerModelDto> resultDto;
    IEnumerable<CustomerModel> result;

    var realmInstance = await _realmManager.GetInstanceAsync();
    if (realmInstance == null)
        return new DataResponse<IEnumerable<CustomerModel>>(null, false, "Realm instance can't be null");

    resultDto = realmInstance.All<CustomerModelDto>();
    result = resultDto.ToModel();

    return new DataResponse<IEnumerable<CustomerModel>>(result, result != null);
}
```

### SQLite: Update
```csharp
public async Task<IResponse> UpdateAsync(CustomerModel customer)
{
    var customerDto = customer.ToDto();

    if (_sqliteManager.DataBase == null)
        return new Response(false);

    await _sqliteManager.DataBase.UpdateAsync(customerDto);
    
    return new Response(true);
}
```

### Realm: Update 
```csharp
public async Task<IResponse> UpdateAsync(CustomerModel customer)
{
    var customerDto = customer.ToDto();

    var realmInstance = await _realmManager.GetInstanceAsync();
    if (realmInstance == null)
        return new Response(false, "Realm instance can't be null");

    await realmInstance.WriteAsync(realm =>
    {
        realm.Add(customerDto, true);

    });
    return new Response(true);
}
```

### SQLite: Delete
```csharp
public async Task<IResponse> DeleteAsync(CustomerModel customer)
{
    var customerDto = customer.ToDto();

    if (_sqliteManager.DataBase == null)
        return new Response(false);

    await _sqliteManager.DataBase.DeleteAsync(customerDto);
    
    return new Response(true);
}
```

### Realm: Delete 
```csharp
public async Task<IResponse> DeleteAsync(CustomerModel customer)
{
    var realmInstance = await _realmManager.GetInstanceAsync();
    if (realmInstance == null)
        return new Response(false, "Realm instance can't be null");

    using (var trans = realmInstance.BeginWrite())
    {
        var customerRealm = realmInstance.Find<CustomerModelDto>(customer.Guid); // do not repeat it at home
        realmInstance.Remove(customerRealm);
        trans.Commit();
    }

    return new Response(true);
}
```
**Realm has one major issue with Delete.** We can’t remove an object by an ID. To delete an object, Realm has to get the object from the database and then delete it, meaning that this function would affect the performance, which is bad. That’s because Realm tries to follow the View Model pattern of the client application and provide a direct connection of the object displayed in View with the database. 

Each Realm object has its own connection to the Realm database. So, if we try to delete the customer model after mapping, the application will crash. In a real-world application, this wouldn’t be an issue if the solution had been architected to use Realm from the start. In our case, though, we have to compromise performance to let the app switch between two databases. 
</aside>

## Performance
To evaluate Realm and SQLite performance, we operated 30 times with 1,000 records in different CRUD operations.

```csharp
private async Task TestInOrder()
{
    //...
    await CreateCustomers();
    await CreateEmployees();
    await CreateCompanies();

    await ReadAllCompaniesAsync();
    await ReadAllCustomersAsync();
    await ReadAllEmployeesAsync();

    await UpdateAllCompanies();
    await UpdateAllCustomers();
    await UpdateAllEmployees();

    await DeleteAllCustomers();
    await DeleteAllEmployees();
    await DeleteAllCompanies();
    //...
}
```
As you can see, we were executing CRUD operations for three different repositories. 
 
**In order** (Operations are performed serially) 

![Sqlite vs Realm in order](https://github.com/lHubertl/realm-vs-sqlite/blob/master/Matterials/Sqlite_vs_Realm_in_order.PNG)

**Not In Order** (Operations are performed in parallel with one another) 

![Sqlite vs Realm not in order](https://github.com/lHubertl/realm-vs-sqlite/blob/master/Matterials/Sqlite_vs_Realm_not_in_order.PNG)

*General time* - is the time of the last iteration. <br />
*Average time* - is the average of 30 iterations.

**The verdict:** In this round, Realm is the winner. It scored 57% in the serial test and 62% in the parallel test, which means that Realm’s performance is better than SQLite’s. Realm is simply faster.

[Click here for more](https://github.com/JetSoftPro/realm-vs-sqlite/blob/master/mobileDbs/MobileDbs/MobileDbs/ViewModels/PerformancePageViewModel.cs)

## Concurrency
The main idea of the concurrency test was to check the security of creating records in different threads. We ran 10 threads that were creating 10k records. After that, we checked if everything was created safely. Then, we calculated the total time.
```csharp
//Some code was removed to save space
private async Task ExecuteTest()
{
    int iterations = 10;

    // Generate 10 000 customers in 10 threads
    for (int i = 0; i < iterations; i++)
    {
        var threadStart = new ThreadStart(CreateCustomers);
        Thread thread = new Thread(threadStart);
        thread.Start();
    }
    
    // Wait while customers will be created
    
    //Get all customers from db
    var customersFromDb = await ReadAllCustomersAsync();

    //Compare all customers
    CheckIfAllCustomersHasRightData();
}
```

![Sqlite vs Realm Cuncurrency](https://github.com/lHubertl/realm-vs-sqlite/blob/master/Matterials/Sqlite_vs_Realm_concurrency.PNG)

**The verdict:** In this test, Realm wins once again. It’s faster than SQLite by 34%, meaning that Realm manages parallel calls better. Needless to say, it kept the data intact and unchanged.

[Click here for more](https://github.com/JetSoftPro/realm-vs-sqlite/blob/master/mobileDbs/MobileDbs/MobileDbs/ViewModels/ConcurrencyPageViewModel.cs)

## Filtering and Fetching 
The filtering and fetching records are necessary to get the exact data we want. To test them, we generated 10k records randomly with the Age property between 30 and 65. After that, by using a simple expression, we got all the customers whose age is below 35 years. 
```csharp
var filteringCustomers = (await _customerService.ReadByPredicate(item => item.Age < 35)).Data;
```
![Sqlite vs Realm Filtering and Fetching](https://github.com/lHubertl/realm-vs-sqlite/blob/master/Matterials/Sqlite_vs_Realm_filtering_fetching.PNG)

**The verdict:** The winner? Realm again, with a difference of 87%. Realm is almost 8 times faster than SQLite in filtering and fetching.  

[Click here for more](https://github.com/JetSoftPro/realm-vs-sqlite/blob/master/mobileDbs/MobileDbs/MobileDbs/ViewModels/FilteringPageViewModel.cs)

## CRUD Per Second 
In this test, we launched each of the operations as many times as possible within one second. We repeated this 10 times. 

*Average* is an average number of all the completed operations in 10 iterations.<br />
*Processed records* is a calculation of all operations per the last iteration. 

![Sqlite vs Realm CRUD operation per second](https://github.com/lHubertl/realm-vs-sqlite/blob/master/Matterials/Sqlite_vs_Realm_crud_per_second.PNG)

On the screen above, the average number is broken down by an operation. 
The logs show the results for the last iteration. 
 
**The verdict:** Now, it’s SQLite’s time to shine. It finally showed better results than Realm. The difference was 10%. So, SQLite can actually manage more requests per second than Realm. 
 
It’s worth adding that for the current implementation of Delete, Realm has to perform a Read operation before every Delete operation. So, even with this unoptimized approach, Realm has a rather solid performance. 

[Click here for more](https://github.com/JetSoftPro/realm-vs-sqlite/blob/master/mobileDbs/MobileDbs/MobileDbs/ViewModels/TestPerSecondPageViewModel.cs)

## Encryption
All of the previous tests were run with encryption. And because of that, the performance could differ from an encrypted database. Let’s look at how both databases handle encryption. 
 
The Realm database provided its own encryption based on the 64-byte array key. 
```csharp
var encryptKey = new byte[64] // key MUST be exactly this size
{
     0x02, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
     0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18,
     0x21, 0x22, 0x23, 0x24, 0x25, 0x26, 0x27, 0x28,
     0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38,
     0x41, 0x42, 0x43, 0x44, 0x45, 0x46, 0x47, 0x48,
     0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57, 0x58,
     0x61, 0x62, 0x63, 0x64, 0x65, 0x66, 0x67, 0x68,
     0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77, 0x78
};

var config = new RealmConfiguration($"{dbName}.realm")
{
    EncryptionKey = encryptKey
};
```

[Click here for more](https://github.com/JetSoftPro/realm-vs-sqlite/blob/master/mobileDbs/MobileDbs.Infrastructure.Realm/RealmManager.cs)

You will have to use the *config* variable to create each Realm instance. According to their documentation, there’s always a small performance hit (typically less than 10% slower) when using encrypted Realm. Our tests confirm this. 

For SQLite, we used SQL Cipher. It’s better to use the *sqlite-net-sqlcipher* package because it’s compliant with the .NET Standard 2.0. By using Cipher, it’s easy to pass a password as a query. But if the password is wrong, the next access to the database will throw an exception.
```csharp
await DataBase.QueryAsync<int>($"PRAGMA key={password}");
```

[Click here for more](https://github.com/JetSoftPro/realm-vs-sqlite/blob/master/mobileDbs/MobileDbs.Infrastructure.SQLite/SQLiteManager.cs)

Be careful: SQLite encryption will hit application performance! On average, the performance hit was 26%. 

**The verdict:** We can’t tell which encryption method is safer, but Realm’s 10% performance hit sounds better than SQLite’s 26%. 

## Foreign Key 
Since Realm isn’t a relational database, it doesn’t officially support the concept of foreign keys. Instead, Realm offers an ability to link objects as the children of a parent object directly. For one-to-one relationships, the child object’s type can be used as a property of the parent object. And for one-to-many relationships, a *Realm List* object can be used to store multiple child objects of the same type.

To build relationships in SQLite, you have to use additional libraries like SQLite Extension. But for now, this library doesn’t work with the .NET Standard 2.0 at all. Which means it’s not really reliable.

## AutoMapper 
Let’s set something straight: Automapper 6.2.X doesn’t work properly on iOS with the .NET Standard. But here’s a solution. 

In the case with SQLite 2.8.x and Realm 4.4.x, don’t forget to create a second map for two-way mapping. Otherwise, you’ll get a performance penalty.
```csharp
cfg.CreateMap<CustomerModel, CustomerModelDto>();
cfg.CreateMap<CustomerModelDto, CustomerModel>();
```
![Sqlite vs Realm automapper performance hit sqlite](https://github.com/lHubertl/realm-vs-sqlite/blob/master/Matterials/Sqlite_vs_Realm_automapper_performance_hit_sqlite.PNG) <br/>
**SQLite performance hit: 1.6x** 

![Sqlite vs Realm automapper performance hit realm](https://github.com/lHubertl/realm-vs-sqlite/blob/master/Matterials/Sqlite_vs_Realm_automapper_performance_hit_realm.PNG) <br/>
**Realm performance hit: 2.5x**

**The verdict:** Because Realm objects inherit from the *RealmObject* class, Automapper resolves models for Realm slower than for SQLite. Another fact is that Automapper’s initialization can last up to two seconds, which is terrible for mobile applications. So, we don’t recommend using Automapper for mobile apps. 

## Maintainability by Other Developers 
Realm models should be inherited from the *RealmObject* class, and all objects should be created by a *Realm.CreateObject()* method. You have to create a transaction before creating any objects, which is inconvenient for executing mapping. On the other hand, Realm allows you to “listen” to object changes, which is a very handy way to update the UI if you use mapping.

Realm is simple to set up. It doesn’t require the plumbing SQLite does, like getting a location for a file to store your DB in and creating *SqlAsyncConnection.* In Realm, we can just call the *GetInstance()* method and that’s it. 

**The verdict:** SQLite can be quite fast if you know what you’re doing. But Realm seems to do the job well without you having to think how to do things like transactions.

## The Benefits of Realm and SQLite 
Our tests reveal that Realm is faster than SQLite (applause!). Realm showed better results in terms of performance, concurrency, encryption, filtering and fetching. Also, it’s simple to set up and maintain by other developers. But SQLite is better at managing many requests per second. Now, what exactly does this mean? Let’s find out what benefits Realm and SQLite can bring to business projects. 

Because Realm is simple and easy to work with, it allows fast development. This makes application deployment and delivery to market quick. Moreover, the Realm Platform has a lot of additional features which can save you time and money. It seems like Realm is a perfect tool for small or medium-sized projects.

Although according to our tests, SQLite didn’t win, it’s still an excellent database management solution. True, SQLite is not as fast as Realm, but it can manage more requests per second. This feature makes SQLite perfect for large projects with a complicated structure. Also, SQLite is essentially a single file, which means it’s highly portable and can be easily transferred or integrated into other systems. This allows saving time on the system’s configuration. 

## Conclusion
Realm and SQLite are very different. To decide which one suits your project best, first, you have to know what goals you want to achieve. Realm allows fast development and delivery to market. SQLite is easy to deploy and requires minimal configuration. The Realm Platform is perfect for small projects while SQLite is more beneficial for larger ones. 

>*"No matter which of the two you choose, remember that the project’s success depends on the team that implements it."*

Contact our professionals at [JetSoftPro](http://www.jetsoftpro.com/) to achieve great results together.

*Created by Valerii Sovytskyi, Xamarin Developer*
