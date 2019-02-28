---
title: Breaking changes in EF Core 3.0 - EF Core
author: divega
ms.date: 02/19/2019
ms.assetid: EE2878C9-71F9-4FA5-9BC4-60517C7C9830
uid: core/what-is-new/ef-core-3.0/breaking-changes
---

# Breaking changes included in EF Core 3.0 (in preview)

> [!IMPORTANT]
> Please note that the feature sets and schedules of future releases are always subject to change, and although we will try to keep this page up to date, it may not reflect our latest plans at all times.

These are breaks in either API or behavior between the 2.2.x releases and the 3.0.0 release. Breaks in new features introduced from one 3.0 preview to another 3.0 preview are not documented here.

## High-impact breaks

TODO: Removal of automatic client-evaluation of queries

## Medium-impact breaks

### Query execution is logged at `Debug` level

[Tracking Issue #14523](https://github.com/aspnet/EntityFrameworkCore/issues/14523)

Prior to EF Core 3.0, execution of queries and other commands was logged at the `Info` level. This has been changed in EF Core 3.0 so that the logging is at the `Debug` level. This logging event is defined by `RelationalEventId.CommandExecuting` with event ID 20100.

This change was made to reduce the noise at the `Info` log level.

### Temporary key values are not set onto entity instances

[Tracking Issue #12378](https://github.com/aspnet/EntityFrameworkCore/issues/12378)

Prior to EF Core 3.0, temporary values were assigned to all key properties that would later have a real value generated by the database. These values were usually large negative numbers.

Starting with EF Core 3.0, the temporary key value is now stored with EF's tracking information while the value of the key property itself will not be changed.

This change was made to allow entities that have been previously tracked by some context instance to be moved to another instance without the temporary key values erroneously becoming permanent.

Applications may be using the temporary key values to form associations between entities. For example, the temporary primary key value may have been used to set an FK value. This can be avoided by:
* Not using store-generated keys.
* Not using primary key/foreign key values to associate entities, but instead use navigation properties. This is a best practice anyway since it uses only the object model without dependency on the underlying keys.
* Obtain the temporary values from EF's tracking information. For example, `context.Entry(blog).Property(e => e.Id).CurrentValue` will return the temporary value even though `blog.Id` itself has not been set.

### `DetectChanges` honors store-generated key values

[Tracking Issue #14616](https://github.com/aspnet/EntityFrameworkCore/issues/14616)

Prior to EF Core 3.0, an un-tracked entity found by `DetectChanges` would be tracked in the `Added` state. This means that it would be inserted as a new row when `SaveChanges` is called.

Starting with EF Core 3.0, if an entity is using generated key values and some key value is set, then the entity will be tracked in the `Modified` state. This means that a row for the entity is assumed to exist and it will be updated when `SaveChanges` is called. If the key value is not set, or if the entity type is not using generated keys, then the new entity will still be tacked as `Added` just as in previous versions.

This change was made to make it easier and more consistent to work with disconnected entity graphs while using store-generated keys.

This can break an application if an entity type is configured to use generated keys but then explicit key values are being set for new instances. The fix is to explicitly configure the key properties to not use generated values.  For example, with the fluent API:

```C#
modelBuilder
    .Entity<Blog>()
    .Property(e => e.Id)
    .ValueGeneratedNever();
```

Or with data annotations:

```C#
[DatabaseGenerated(DatabaseGeneratedOption.None)]
public string Id { get; set; }
```

### Cascade deletions now happen immediately by default

[Tracking Issue #10114](https://github.com/aspnet/EntityFrameworkCore/issues/10114)

Prior to EF Core 3.0, cascading actions (deleting dependent entities when a required principal is deleted or when the relationship to a required principal is severed) did not happen until SaveChanges was called.

Starting with EF Core 3.0, cascading actions happen immediately that the triggering condition is detected by EF. For example, calling `context.Remove()` to delete a principal entity will result in all tracked related required dependents also being set to `Deleted` immediately.

This change was made to improve the experience for data binding and auditing scenarios where it is important to understand which entities will be deleted _before_ `SaveChanges` is called.

The previous behavior can be restored through settings on `context.ChangedTracker`. For example:

```C#
context.ChangeTracker.CascadeDeleteTiming = CascadeTiming.OnSaveChanges;
context.ChangeTracker.DeleteOrphansTiming = CascadeTiming.OnSaveChanges;
```

### Query types are consolidated with entity types

[Tracking Issue #14194](https://github.com/aspnet/EntityFrameworkCore/issues/14194)

[Query types](xref:core/modeling/query-types) were introduced in 2.1 as a means to query data that doesn't contain a primary key in a structured way. However it caused some confusion when deciding whether to configure something as an entity type or a query type. Therefore we have decided to bring the two closer together in the API.

A query type now becomes just an entity type without a primary key, but will have the same functionality as in previous versions. The following parts of the API are now obsolete:
* **`ModelBuilder.Query<>()`** - Instead `ModelBuilder.Entity<>().HasNoKey()` needs to be called to mark an entity type as having no keys. This would still not be configured by convention to avoid misconfiguration when a primary key is expected, but doesn't match the convention.
* **`DbQuery<>`** - Instead `DbSet<>` should be used.
* **`DbContext.Query<>()`** - Instead `DbContext.Set<>()` should be used.

### Configuration API for owned type relationships has changed

[Tracking Issue #12444](https://github.com/aspnet/EntityFrameworkCore/issues/12444)
[Tracking Issue #9148](https://github.com/aspnet/EntityFrameworkCore/issues/9148)
[Tracking Issue #14153](https://github.com/aspnet/EntityFrameworkCore/issues/14153)

Prior to EF Core 3.0, configuration of the owned relationship was performed directly after the `OwnsOne` or `OwnsMany` call. 

Starting with EF Core 3.0, there is now fluent API to configure a navigation to the owner using `WithOwner()`. For example:

```C#
modelBuilder.Entity<Order>.OwnsOne(e => e.Details).WithOwner(e => e.Order);
```

The configuration related to the relationship between owner and owned should now be chained after `WithOwner()` similarly to how other relationships are configured. While the configuration for the owned type itself would still be chained after `OwnsOne()/OwnsMany()`. For example:

```C#
modelBuilder.Entity<Order>.OwnsOne(e => e.Details, eb =>
    {
        eb.WithOwner()
            .HasForeignKey(e => e.AlternateId)
            .HasConstraintName("FK_OrderDetails");
            
        eb.ToTable("OrderDetails");
        eb.HasKey(e => e.AlternateId);
        eb.HasIndex(e => e.Id);

        eb.HasOne(e => e.Customer).WithOne();

        eb.HasData(
            new OrderDetails
            {
                AlternateId = 1,
                Id = -1
            });
    });
```

Additionally calling `Entity()`, `HasOne()` or `Set()` with an owned type target will now throw an exception.

### The foreign key property convention no longer matches same name as the principal property

[Tracking Issue #13274](https://github.com/aspnet/EntityFrameworkCore/issues/13274)

Consider the following model
```C#
public class Customer
{
    public int CustomerId { get; set; }
    public ICollection<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
}

```
Prior to EF Core 3.0, the `CustomerId` property would be used for the foreign key by convention. However, if `Order` is an owned type, then this would also make `CustomerId` the primary key and this is usually not the expectation.

Starting with EF Core 3.0, EF  will not try to use properties for foreign keys by convention if they have the same name as the principal property. Principal type name + principal property name and navigation + principal property name patterns will still be matched. For example:

```C#
public class Customer
{
    public int Id { get; set; }
    public ICollection<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
}
```

```C#
public class Customer
{
    public int Id { get; set; }
    public ICollection<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int BuyerId { get; set; }
    public Customer Buyer { get; set; }
}
```

## Other known breaks

### Each property uses independent in-memory integer key generation

[Tracking Issue #6872](https://github.com/aspnet/EntityFrameworkCore/issues/6872)

Prior to EF Core 3.0, one shared value generator was used for all in-memory integer key properties.

Starting with EF Core 3.0, each integer key property gets its own value generator when using the in-memory database. Also, if the database is deleted, then key generation is reset for all tables.

This change was made to align in-memory key generation more closely to real database key generation and to improve the ability to isolate tests from each other when using the in-memory database.

This can break an application that is relying on specific in-memory key values to be set. Consider instead not relying on specific key values, or updating to match the new behavior.

### Backing fields are used by default

[Tracking Issue #12430](https://github.com/aspnet/EntityFrameworkCore/issues/12430)

Prior to EF Core 3.0, even if the backing field for a property was known, EF would still by default read and write the property value using the property getter and setter methods. The exception to this was query execution, where the backing field would be set directly if known.

Starting with EF Core 3.0, if the backing field for a property is known, then will always read and write that property using the backing field. This could cause an application break if the application is relying on additional behavior coded into the getter or setter methods.

This change was made to prevent EF from erroneously triggering buisness logic by default when performing database operations invoving the entities.

The pre-3.0 behavior can be restored through configuration of the property access mode in the modelBuilder fluent API. For example:

```C#
modelBuilder.UsePropertyAccessMode(PropertyAccessMode.PreferFieldDuringConstruction);
```

### Throw if multiple compatible backing fields are found

[Tracking Issue #12523](https://github.com/aspnet/EntityFrameworkCore/issues/12523)

Prior to EF Core 3.0, if multiple fields matched the rules for finding the backing field of a property, then one field would be chosen based on some precedence order. This could cause the wrong field to be used in ambiguous cases.

Starting with EF Core 3.0, if multiple fields are matched to the same property, then an exception is thrown.

This change was made to avoid silently using one field over the another when only one can be correct.

Properties with ambiguous backing fields must have the field to use specified explicitly. For example, using the fluent API:

```C#
modelBuilder
    .Entity<Blog>()
    .Property(e => e.Id)
    .HasField("_id");
```

### `DbContext.Entry` now performs a local `DetectChanges`

[Tracking Issue #13552](https://github.com/aspnet/EntityFrameworkCore/issues/13552)

Prior to EF Core 3.0, calling `DbContext.Entry` would cause changes to be detected for all tracked entities. This ensured that the state exposed in the `EntityEntry` was up-to-date.

Starting with EF Core 3.0, calling `DbContext.Entry` will now only attempt to detect changes in the given entity and any tracked principal entities related to it. This means that changes elsewhere may not have been detected by calling this method, which could have implications on application state.

Note that if `ChangeTracker.AutoDetectChangesEnabled` is set to `false` then even this local change detection will be disabled.

Other methods that cause change detection--for example `ChangeTracker.Entries` and `SaveChanges`--still cause a full `DetectChanges` of all tracked entities.

This change was made to improve the default performance of using `context.Entry`.

Call `ChgangeTracker.DetectChanges()` explicitly before calling `Entry` to ensure the pre-3.0 behavior.

### `String`/`byte[]` keys are not client-generated by default

[Tracking Issue #14617](https://github.com/aspnet/EntityFrameworkCore/issues/14617)

Prior to EF Core 3.0, `string` and `byte[]` key properties could be used without explicitly setting a non-null value. In such a case, the key value would be generated on the client as a GUID, serialized to bytes for `byte[]`.

Starting with EF Core 3.0 an exception will be thrown indicating that no key value has been set.

This change was made because client-generated `string`/`byte[]` values are generally not useful and this innapropriate default configuration was causing issues with reasoning about generated key values in a common way.

The pre-3.0 behavior can be obtained by explicitly specifying that the key properties should use generated values if no other non-null value is set. For example, with the fluent API:

```C#
modelBuilder
    .Entity<Blog>()
    .Property(e => e.Id)
    .ValueGeneratedOnAdd();
```

Or with data annotations:

```C#
[DatabaseGenerated(DatabaseGeneratedOption.Identity)]
public string Id { get; set; }
```

### `ILoggerFactory` is now a scoped service

[Tracking Issue #14698](https://github.com/aspnet/EntityFrameworkCore/issues/14698)

Prior to EF Core 3.0, ILoggerFactory was registered as a singleton service, whereas it is now registered as scoped. This change should not impact application code unless it is registering and using custom services on the EF internal service provider. This is not common. In these cases, most things will still work, but any singleton service that was depending on `ILoggerFactory` will need to be changed to obtain the `ILoggerFactory` in a different way.

This change was made to allow association of a logger with a `DbContext` instance which enables other functionality and removes some cases of pathalogical behavior such as an explosion of internal service providers.

If you run into situations like this, please file an issue at on the [EF Core GitHub issue tracker](https://github.com/aspnet/EntityFrameworkCore/issues) to let us know how you are using `ILoggerFactory` such that we can better understand how not to break this again in the future.

### `IDbContextOptionsExtensionWithDebugInfo` merged into `IDbContextOptionsExtension`

[Tracking Issue #13552](https://github.com/aspnet/EntityFrameworkCore/issues/13552)

Any implementations of `IDbContextOptionsExtension` will need to be updated to support the new member.

This change was made because the interfaces are conceptually one.

### Lazy-loading proxies no longer assume navigation properties are fully loaded

[Tracking Issue #12780](https://github.com/aspnet/EntityFrameworkCore/issues/12780)

Prior to EF Core 3.0, once a `DbContext` was disposed there was no way of knowing if a given navigation property on an entity obtained from that context was fully loaded or not. Proxies would instead assume that a reference navigation is loaded if it has a non-null value, and that a collection navigation is loaded if it is not empty. In these cases, attempting to lazy-load would be a no-op.

Starting with EF Core 3.0, proxies keep track of whether or not a navigation is loaded. These means attempting to access a navigation property that is loaded after the context has been disposed will always be a no-op, even when the loaded navigation is empty or null. Conversely, attempting to access a navigation property that is not loaded will throw an exception if the context is disposed even if the navigation property is a non-empty collection. If this situation arises, it means the application code is attempting to use lazy-loading at an invalid time and the application should be changed to not do this.

This change was made to make the behavior consistent and correct when attempting to lazy-load on a disposed `DbContext` instance.

### Excessive creation of internal service providers is now an error by default

[Tracking Issue #10236](https://github.com/aspnet/EntityFrameworkCore/issues/10236)

Prior to EF Core 3.0, a warning would be logged for an application creating a pathological number of internal service providers.

Starting with EF Core 3.0, this warning is now considered and error and an exception is thrown. 

This change was made to drive better application code through exposing this pathological case more explicitly.

The most appropriate cause of action on encountering this error is to understand the root cause and stop creating so many internal service providers. However, the error can be converted back to a warning (or ignored) via configuration on the `DbContextOptionsBuilder`. For example:

```C#
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .ConfigureWarnings(w => w.Log(CoreEventId.ManyServiceProvidersCreatedWarning));
}
```

### The `Relational:TypeMapping` annotation is now just `TypeMapping`

[Tracking Issue #9913](https://github.com/aspnet/EntityFrameworkCore/issues/9913)

This will only break applications that access the type mapping directly as an annotation, which is not common. The most appropriate action to fix is to use API surface to access type mappings rather than using the annotation directly.

### `ToTable` on a derived type throws an exception 

[Tracking Issue #11811](https://github.com/aspnet/EntityFrameworkCore/issues/11811)

Prior to EF Core 3.0, `ToTable()` called on a derived type would be ignored since only inheritance mapping strategy was TPH where this is not valid. 

Starting with EF Core 3.0 and in preparation for adding TPT and TPC support, `ToTable()` called on a derived type will now throw an exception to avoid an unexpected mapping change in the future.

### ForSqlServerHasIndex replaced with HasIndex 

[Tracking Issue #12366](https://github.com/aspnet/EntityFrameworkCore/issues/12366)

`ForSqlServerHasIndex().ForSqlServerInclude()` provided a way to configure columns used with `INCLUDE`, now the same can be accomplished with `HasIndex().ForSqlServerInclude()`

## Breaks that should only impact database providers

Changes that we expect to only impact database providers are documented under [provider changes](../../providers/provider-log.md).

## Breaks to the `Microsoft.Data.Sqlite` ADO.NET database provider

* Microsoft.EntityFrameworkCore.Sqlite now depends on SQLitePCLRaw.bundle_e_sqlite3 instead of SQLitePCLRaw.bundle_green. This makes the version of SQLite used on iOS consistent with other platforms.
* Removed SqliteDbContextOptionsBuilder.SuppressForeignKeyEnforcement(). EF Core no longer sends `PRAGMA foreign_keys = 1` when a connection is opened. Foreign keys are enabled by default in SQLitePCLRaw.bundle_e_sqlite3. If you're not using that, you can enable foreign keys by specifying `Foreign Keys=True` in your connection string.