ServiceStack provides a number of ways of changing the database connection used at runtime based on an incoming Request. You can use a [Request Filter](https://github.com/ServiceStack/ServiceStack/wiki/Request-and-response-filters#global-request-filters), use the `[ConnectionInfo]` [Request Filter Attribute](https://github.com/ServiceStack/ServiceStack/wiki/Filter-attributes#request-filter-attributes), use the `[NamedConnection]` attribute on [[Auto Query]] Services, access named connections in Custom Service implementations or override `GetDbConnection(IRequest)` in your AppHost.

### Change Database Connection at Runtime

The default implementation of `IAppHost.GetDbConnection(IRequest)` includes an easy way to change the DB Connection that can be done by populating the 
[ConnectionInfo](https://github.com/ServiceStack/ServiceStack/blob/master/src/ServiceStack/ConnectionInfo.cs) 
POCO in any
[Request Filter in the Request Pipeline](https://github.com/ServiceStack/ServiceStack/wiki/Order-of-Operations):

```csharp
req.Items[Keywords.DbInfo] = new ConnectionInfo {
    NamedConnection  = ... //Use a registered NamedConnection for this Request
    ConnectionString = ... //Use a different DB connection for this Request
    ProviderName     = ... //Use a different Dialect Provider for this Request
};
```

To illustrate how this works we'll go through a simple example showing how to create an AutoQuery Service 
that lets the user change which DB the Query is run on. We'll control which of the Services we want to allow 
the user to change the DB it's run on by having them implement the interface below:

```csharp
public interface IChangeDb
{
    string NamedConnection { get; set; }
    string ConnectionString { get; set; }
    string ProviderName { get; set; }
}
```

We'll create one such AutoQuery Service, implementing the above interface:

```csharp
[Route("/rockstars")]
public class QueryRockstars : QueryBase<Rockstar>, IChangeDb
{
    public string NamedConnection { get; set; }
    public string ConnectionString { get; set; }
    public string ProviderName { get; set; }
}
``` 

For this example we'll configure our Database to use a default **SQL Server 2012** database, 
register an optional named connection looking at a "Reporting" **PostgreSQL** database and 
register an alternative **Sqlite** RDBMS Dialect that we also want the user to be able to use:

#### ChangeDB AppHost Registration

```csharp
container.Register<IDbConnectionFactory>(c => 
    new OrmLiteConnectionFactory(defaultDbConn, SqlServer2012Dialect.Provider));

var dbFactory = container.Resolve<IDbConnectionFactory>();

//Register NamedConnection
dbFactory.RegisterConnection("Reporting", ReportConnString, PostgreSqlDialect.Provider);

//Register DialectProvider
dbFactory.RegisterDialectProvider("Sqlite", SqliteDialect.Provider);
```

#### ChangeDB Request Filter

To enable this feature we just need to add a Request Filter that populates the `ConnectionInfo` with properties
from the Request DTO:

```csharp
GlobalRequestFilters.Add((req, res, dto) => {
   var changeDb = dto as IChangeDb;
   if (changeDb == null) return;

   req.Items[Keywords.DbInfo] = new ConnectionInfo {
       NamedConnection = changeDb.NamedConnection,
       ConnectionString = changeDb.ConnectionString,
       ProviderName = changeDb.ProviderName,
   };
});
```

Since our `IChangeDb` interface shares the same property names as `ConnectionInfo`, the above code can be 
further condensed using a 
[Typed Request Filter](https://github.com/ServiceStack/ServiceStack/wiki/Request-and-response-filters#typed-request-filters)
and ServiceStack's built-in [AutoMapping](https://github.com/ServiceStack/ServiceStack/wiki/Auto-mapping)
down to just:

```csharp
RegisterTypedRequestFilter<IChangeDb>((req, res, dto) =>
    req.Items[Keywords.DbInfo] = dto.ConvertTo<ConnectionInfo>());
```

#### Change Databases via QueryString

With the above configuration the user can now change which database they want to execute the query on, e.g:

```csharp
var response = client.Get(new QueryRockstars()); //SQL Server

var response = client.Get(new QueryRockstars {   //Reporting PostgreSQL DB
    NamedConnection = "Reporting"
}); 

var response = client.Get(new QueryRockstars {   //Alternative SQL Server Database
    ConnectionString = "Server=alt-host;Database=Rockstars;User Id=test;Password=test;"
}); 

var response = client.Get(new QueryRockstars {   //Alternative SQLite Database
    ConnectionString = "C:\backups\2016-01-01.sqlite",
    ProviderName = "Sqlite"
}); 
```

### ConnectionInfo Attribute

To make it even easier to use we've also wrapped this feature in a simple
[ConnectionInfoAttribute.cs](https://github.com/ServiceStack/ServiceStack/blob/master/src/ServiceStack/ConnectionInfoAttribute.cs)
which allows you to declaratively specify which database a Service should be configured to use, e.g we can
configure the `Db` connection in the Service below to use the PostgreSQL **Reporting** database with:

```csharp
[ConnectionInfo(NamedConnection = "Reporting")]
public class ReportingServices : Service
{
    public object Any(Sales request)
    {
        return new SalesResponse { Results = Db.Select<Sales>() };
    }
}
```

### [Auto Query Named Connection](https://github.com/ServiceStack/ServiceStack/wiki/Auto-Query#named-connection)

[[Auto Query]] can also easily be configured to query any number of different databases registered in your AppHost. 

In the example below we configure our main RDBMS to use SQL Server and register a **Named Connection** 
to point to a **Reporting** PostgreSQL RDBMS:

```csharp
var dbFactory = new OrmLiteConnectionFactory(connString, SqlServer2012Dialect.Provider);
container.Register<IDbConnectionFactory>(dbFactory);

dbFactory.RegisterConnection("Reporting", pgConnString, PostgreSqlDialect.Provider);
```

Any normal AutoQuery Services like `QueryOrders` will use the default SQL Server connection whilst 
`QuerySales` will execute its query on the PostgreSQL `Reporting` Database instead:

```csharp
public class QueryOrders : QueryBase<Order> {}

[NamedConnection("Reporting")]
public class QuerySales : QueryBase<Sales> {}
```

### Resolving Named Connections in Services

Whilst inside a Service you can change which DB connection to use by passing in the NamedConnection when opening a DB Connection. E.g. The example below allows the user to change which database to retrieve all sales records for otherwise fallbacks to "Reporting" database by default:

```csharp
public class SalesServices : Service
{
   public IDbConnectionFactory ConnectionFactory { get; set; } 

   public object Any(GetAllSales request)
   {
       var namedConnection = request.NamedConnection ?? "Reporting";
       using (var db = ConnectionFactory.Open(namedConnection)) 
       {
           return db.Select<Sales>();
       }
   }
}
```

### Override Connection used per request at Runtime

All built-in dependencies available from `Service` base class, AutoQuery, Razor View pages, etc are resolved 
from a central overridable location in your `AppHost`. This lets you control which pre-configured 
dependency gets used based on the incoming Request for each Service by overriding any of the `AppHost` methods below: 

```csharp
public virtual IDbConnection Db
{
    get { return db ?? (db = HostContext.AppHost.GetDbConnection(Request)); }
}

public virtual ICacheClient Cache
{
    get { return cache ?? (cache = HostContext.AppHost.GetCacheClient(Request)); }
}

public virtual MemoryCacheClient LocalCache
{
    get { return localCache ?? 
              (localCache = HostContext.AppHost.GetMemoryCacheClient(Request)); }
}

public virtual IRedisClient Redis
{
    get { return redis ?? (redis = HostContext.AppHost.GetRedisClient(Request)); }
}

public virtual IMessageProducer MessageProducer
{
    get { return messageProducer ?? 
              (messageProducer = HostContext.AppHost.GetMessageProducer(Request)); }
}
```

E.g. to change the DB Connection your Service uses you can override `GetDbConnection(IRequest)` in your `AppHost`.

### [Multi Tenancy Example](https://github.com/ServiceStack/ServiceStack/blob/master/tests/ServiceStack.WebHost.Endpoints.Tests/MultiTennantAppHostTests.cs)

To show how easy it is to implement a Multi Tenancy Service with this feature we've added a stand-alone
[Multi Tenancy AppHost Example](https://github.com/ServiceStack/ServiceStack/blob/master/tests/ServiceStack.WebHost.Endpoints.Tests/MultiTennantAppHostTests.cs) 
showing 2 different ways we can configure a Service to use different databases based on an incoming request.

In this example we've configured our AppHost to use the **master.sqlite** database as default and registered
3 different named connections referencing 3 different databases. Each database is then initialized with a 
different row in the `TenantConfig` table to identify the database that it's in.

```csharp
public class MultiTenantChangeDbAppHost : AppSelfHostBase
{
    public MultiTenantChangeDbAppHost()
        : base("Multi Tennant Test", typeof (MultiTenantChangeDbAppHost).Assembly) {}

    public override void Configure(Container container)
    {
        container.Register<IDbConnectionFactory>(new OrmLiteConnectionFactory(
            "~/App_Data/master.sqlite".MapAbsolutePath(), SqliteDialect.Provider));

        var dbFactory = container.Resolve<IDbConnectionFactory>();

        const int noOfTennants = 3;

        using (var db = dbFactory.OpenDbConnection())
            InitDb(db, "MASTER", "Masters inc.");

        noOfTennants.Times(i =>
        {
            var tenantId = "T0" + (i + 1);
            using (var db = dbFactory.OpenDbConnectionString(GetTenantConnString(tenantId)))
                InitDb(db, tenantId, "ACME {0} inc.".Fmt(tenantId));
        });

        RegisterTypedRequestFilter<IForTenant>((req,res,dto) => 
            req.Items[Keywords.DbInfo] = new ConnectionInfo { ConnectionString = GetTenantConnString(dto.TenantId)});
    }

    public void InitDb(IDbConnection db, string tenantId, string company)
    {
        db.DropAndCreateTable<TenantConfig>();
        db.Insert(new TenantConfig { Id = tenantId, Company = company });
    }

    public string GetTenantConnString(string tenantId)
    {
        return tenantId != null 
            ? "~/App_Data/tenant-{0}.sqlite".Fmt(tenantId).MapAbsolutePath()
            : null;
    }
}
```

This example uses only contains a single Service which returns the first result in the `TenantConfig` table:

```csharp
public interface IForTenant
{
    string TenantId { get; }
}

public class TenantConfig
{
    public string Id { get; set; }
    public string Company { get; set; }
}

public class GetTenant : IForTenant, IReturn<GetTenantResponse>
{
    public string TenantId { get; set; }
}

public class GetTenantResponse
{
    public TenantConfig Config { get; set; }
}

public class MultiTenantService : Service
{
    public object Any(GetTenant request)
    {
        return new GetTenantResponse
        {
            Config = Db.Select<TenantConfig>().FirstOrDefault(),
        };
    }
}
```

Calling this Service with a different `TenantId` value changes which database the Service is configured with:

```csharp
var client = new JsonServiceClient(Config.AbsoluteBaseUri);

var response = client.Get(new GetTenant()); //= Company: Masters inc. 

var response = client.Get(new GetTenant { TenantId = "T01" }); //= Company: ACME T01 inc.

var response = client.Get(new GetTenant { TenantId = "T02" }); //= Company: ACME T02 inc.

var response = client.Get(new GetTenant { TenantId = "T03" }); //= Compnay: ACME T03 inc.

client.Get(new GetTenant { TenantId = "T04" }); // throws WebServiceException
```

An alternative way to support Multitenancy using a Custom DB Factory is available in 
[MultiTennantAppHostTests.cs](https://github.com/ServiceStack/ServiceStack/blob/e392265456e12077087c1cd014c918913f409bc9/tests/ServiceStack.WebHost.Endpoints.Tests/MultiTennantAppHostTests.cs#L132).