---
title: Multi-tenancy on S#arp Architecture
date: 2011-01-11 12:00:00 Z
categories:
- Sharp Architecture
- Multi-tenancy
layout: post
comments: true
alias:
- "/2011/01/multi-tenancy-on-sharp-architecture/"
- "/blog/2011/01/multi-tenancy-on-sharp-architecture/"
author: chrisr
---

<strong>Update</strong>: the code below is out of date, please see [Multi-tenancy on S#arp Architecture Revisited](/blog/multi-tenancy-on-sharp-architecture-revisited/) for a better solution

Recently I've been working on adding multi-tenancy to a web application based on the excellent <a title="S#arp Architecture" href="http://sharparchitecture.net/">S#arp Architecture</a> and thought I'd share what I have so far.<a id="more"></a><a id="more-24"></a>

This implementation is mainly based on this <a title="Implementing A Multi-Tenant ASP.NET MVC Application with S#arp Architecture" href="http://thatstoday.com/a/557835">post</a> from Robert Johnson, and uses a separate database for each tenant, along with a master database to hold the details of the tenants (and anything else you wish). The main drawback with using separate databases is handling the migrations, I haven't yet got a solution to this but will look at a way of automating it using something like <a title="Migrator.Net" href="http://code.google.com/p/migratordotnet/">Migrator.Net</a> in a future post.

All of the domain entities inherit from SharpArch.Core.DomainModel.Entity, and I use a marker interface to indicate which entities are part of the tenant domain model.
{% highlight csharp %}
namespace SharpArchitecture.MultiTenant.Core.Contracts
{
  /// &lt;summary&gt;
  /// Marker interface for multi tenant entities.
  /// &lt;/summary&gt;
  public interface IMultiTenantEntity { }
}
{% endhighlight %}

We can now create our domain entities for the tenants:
{% highlight csharp %}
using NHibernate.Validator.Constraints;
using SharpArch.Core.DomainModel;
using SharpArchitecture.MultiTenant.Core.Contracts;

namespace SharpArchitecture.MultiTenant.Core
{
  public class Customer : Entity, IMultiTenantEntity
  {
    [DomainSignature]
    [NotNullNotEmpty]
    public virtual string Code { get; set; }

    [DomainSignature]
    [NotNullNotEmpty]
    public virtual string Name { get; set; }
  }
}
{% endhighlight %}

And we create an entity, to be stored in the master database, to represent each tenant:
{% highlight csharp %}
using SharpArch.Core.DomainModel;

namespace SharpArchitecture.MultiTenant.Core
{
  public class Tenant : Entity
  {
    public virtual string Name { get; set; }

    [DomainSignature]
    public virtual string Domain { get; set; }

    public virtual string ConnectionString { get; set; }
  }
}
{% endhighlight %}

The application will make use of a separate NHibernate session for each tenant, identified by a key. For the master database the default session will be used. So, we create an interface that will provide access to the key:
{% highlight csharp %}
namespace SharpArchitecture.MultiTenant.Framework.Services
{
  public interface ITenantContext
  {
    string Key { get; set; }
  }
}
{% endhighlight %}

And we implement this interface based on the method we choose to identify tenants, in this case based on the subdomain:
{% highlight csharp %}
using System.Web;
using SharpArchitecture.MultiTenant.Framework.Services;

namespace SharpArchitecture.MultiTenant.Web.Services
{
  public class TenantContext : ITenantContext
  {
    private const string DefaultStorageKey = "tenant-context-key";

    public string Key
    {
      get
      {
        if (string.IsNullOrEmpty(StoredKey)) {
          StoredKey = KeyFromRequest;
        }
        return StoredKey;
      }

      set { StoredKey = value; }
    }

    public string KeyFromRequest
    {
      get
      {
        var host = HttpContext.Current.Request.Headers["HOST"];
        var domains = host.Split('.');
        return domains.Length &gt;= 3 ? domains[0] : string.Empty;
      }
    }

    protected string StoredKey
    {
      get { return HttpContext.Current.Items[DefaultStorageKey] as string; }
      set { HttpContext.Current.Items[DefaultStorageKey] = value; }
    }
  }
}
{% endhighlight %}

If a different method of identifying tenants is required, say by a query string parameter, then it is just a case of providing a different implementation of ITenantContext.

We can now create a multi-tenant repository that uses ITenantContext to select the NHibernate session based on the key:
{% highlight csharp %}
using NHibernate;
using SharpArch.Data.NHibernate;
using SharpArchitecture.MultiTenant.Framework.Services;

namespace SharpArchitecture.MultiTenant.Data.Repositories
{
  public class MultiTenantRepository&lt;T&gt; : Repository&lt;T&gt;
  {
    private readonly ITenantContext _tenantContext;

    public MultiTenantRepository(ITenantContext tenantContext)
    {
      _tenantContext = tenantContext;
    }

    protected override ISession Session
    {
      get
      {
        var key = _tenantContext.Key;
        return string.IsNullOrEmpty(key) ? base.Session : NHibernateSession.CurrentFor(key);
      }
    }
  }
}
{% endhighlight %}

Next we need to create a repository interface:
{% highlight csharp %}
using MvcContrib.Pagination;
using SharpArch.Core.PersistenceSupport;

namespace SharpArchitecture.MultiTenant.Core.RepositoryInterfaces
{
  public interface ICustomerRepository : IRepository&lt;Customer&gt;
  {
    IPagination&lt;Customer&gt; GetPagedList(int pageIndex, int pageSize);
  }
}
{% endhighlight %}

and implementation for our multi-tenant entities:
{% highlight csharp %}
using MvcContrib.Pagination;
using NHibernate.Criterion;
using SharpArchitecture.MultiTenant.Core;
using SharpArchitecture.MultiTenant.Core.RepositoryInterfaces;
using SharpArchitecture.MultiTenant.Framework.Services;

namespace SharpArchitecture.MultiTenant.Data.Repositories
{
  public class CustomerRepository : MultiTenantRepository&lt;Customer&gt;, ICustomerRepository
  {
    public CustomerRepository(ITenantContext tenantContext) : base(tenantContext)
    {
    }

    public IPagination&lt;Customer&gt; GetPagedList(int pageIndex, int pageSize)
    {
      var firstResult = (pageIndex - 1) * pageSize;
      var customers = Session.QueryOver&lt;Customer&gt;()
        .OrderBy(customer =&gt; customer.Code).Asc
        .Skip(firstResult)
        .Take(pageSize)
        .Future&lt;Customer&gt;();

      var totalCount = Session.QueryOver&lt;Customer&gt;()
        .Select(Projections.Count&lt;Customer&gt;(customer =&gt; customer.Code))
        .FutureValue&lt;int&gt;();

      return new CustomPagination&lt;Customer&gt;(customers, pageIndex, pageSize, totalCount.Value);
    }
  }
}
{% endhighlight %}

Our controllers can now make use of ICustomerRepository without having to worry about any of the multi-tenancy issues.

We also need to update the TransactionAttribute so that it makes use of the appropriate NHibernate session:
{% highlight csharp %}
using Microsoft.Practices.ServiceLocation;
using SharpArchitecture.MultiTenant.Framework.Services;

namespace SharpArchitecture.MultiTenant.Framework.NHibernate
{
  public class TransactionAttribute : SharpArch.Web.NHibernate.TransactionAttribute
  {
    public TransactionAttribute()
      : base(FactoryKey)
    {
    }

    protected static string FactoryKey
    {
      get
      {
        var tenantContext = ServiceLocator.Current.GetInstance&lt;ITenantContext&gt;();
        return tenantContext.Key;
      }
    }
  }
}
{% endhighlight %}

Next up, we need to update the initialisation in Global.asax.cs so that we create a session factory for the master database and also for each tenant:
{% highlight csharp %}
        private void InitializeNHibernateSession()
        {
          var mappingAssemblies = new [] { Server.MapPath("~/bin/SharpArchitecture.MultiTenant.Data.dll") };

          var configFile = Server.MapPath("~/NHibernate.config");
          NHibernateSession.Init(
                webSessionStorage,
                mappingAssemblies,
                new AutoPersistenceModelGenerator().Generate(),
                configFile);

            var tenantConfigFile = Server.MapPath("~/NHibernate.tenant.config");
            var multiTenantInitializer = ServiceLocator.Current.GetInstance&lt;IMultiTenantInitializer&gt;();
            multiTenantInitializer.Initialize(mappingAssemblies, new MultiTenantAutoPersistenceModelGenerator(),  tenantConfigFile);
        }
{% endhighlight %}

In the code above, the standard NHibernate.config file is used to configure the master database. Whilst NHibernate.tenant.config, along with the connection string provided by the Tenant, is used to configure the tenant databases:
{% highlight csharp %}
using SharpArch.Data.NHibernate.FluentNHibernate;

namespace SharpArchitecture.MultiTenant.Framework.Services
{
  public interface IMultiTenantInitializer
  {
    void Initialize(string[] mappingAssemblies, IAutoPersistenceModelGenerator modelGenerator, string tenantConfigFile);
  }
}
{% endhighlight %}

{% highlight csharp %}
using System.Collections.Generic;
using NHibernate.Cfg;
using SharpArch.Core.PersistenceSupport;
using SharpArch.Data.NHibernate;
using SharpArch.Data.NHibernate.FluentNHibernate;
using SharpArchitecture.MultiTenant.Core;
using SharpArchitecture.MultiTenant.Framework.Services;

namespace SharpArchitecture.MultiTenant.Framework.NHibernate
{
  public class MultiTenantInitializer : IMultiTenantInitializer
  {
    private readonly IRepository&lt;Tenant&gt; _tenantRepository;

    public MultiTenantInitializer(IRepository&lt;Tenant&gt; tenantRepository)
    {
      _tenantRepository = tenantRepository;
    }

    public void Initialize(string[] mappingAssemblies, IAutoPersistenceModelGenerator modelGenerator, string tenantConfigFile)
    {
      var tenants = _tenantRepository.GetAll();
      foreach (var tenant in tenants) {
        Initialize(mappingAssemblies, modelGenerator, tenantConfigFile, tenant);
      }
    }

    private static void Initialize(string[] mappingAssemblies, IAutoPersistenceModelGenerator modelGenerator, string tenantConfigFile, Tenant tenant)
    {
      var properties = new Dictionary&lt;string, string&gt;
                         {
                           { "connection.connection_string", tenant.ConnectionString }
                         };
      AddTenantConfiguration(tenant.Domain, mappingAssemblies, modelGenerator, tenantConfigFile, properties);
    }

    private static Configuration AddTenantConfiguration(string factoryKey, string[] mappingAssemblies, IAutoPersistenceModelGenerator modelGenerator, string cfgFile, IDictionary&lt;string, string&gt; cfgProperties)
    {
      return NHibernateSession.AddConfiguration(factoryKey,
        mappingAssemblies,
        modelGenerator.Generate(),
        cfgFile,
        cfgProperties,
        null, null);
    }
  }
}
{% endhighlight %}

This code iterates through the list of tenants from the master database, setting the connection string and session factory key from the tenant properties and adds the configuration to NHibernate.

All that is left is to generate the mappings. In AutoPersistenceModelGenerator we move the creation of the standard mapping configuration into a method so that it can be overridden:
{% highlight csharp %}
    protected virtual IAutomappingConfiguration GetAutomappingConfiguration()
    {
      return new AutomappingConfiguration();
    }
{% endhighlight %}

And derive from AutoPersistenceModelGenerator to create our multi-tenant mapping configuration:
{% highlight csharp %}
using FluentNHibernate.Automapping;

namespace SharpArchitecture.MultiTenant.Data.NHibernateMaps
{
  public class MultiTenantAutoPersistenceModelGenerator : AutoPersistenceModelGenerator
  {
    protected override IAutomappingConfiguration GetAutomappingConfiguration()
    {
      return new MultiTenantAutomappingConfiguration();
    }
  }
}
{% endhighlight %}

Then we add a method to AutomappingConfiguration to determine if a type is a multi-tenant entity (by checking to see if the type implements our IMultiTenantEntity interface):
{% highlight csharp %}
    public bool IsMultiTenantEntity(Type type)
    {
      return type.GetInterfaces().Any(x =&gt; x == typeof(IMultiTenantEntity));
    }
{% endhighlight %}

and update the ShouldMap method to only map entities that are not multi-tenant:
{% highlight csharp %}
    public override bool ShouldMap(Type type)
    {
      var isMultiTenantEntity = IsMultiTenantEntity(type);
      var shouldMap = type.GetInterfaces().Any(x =&gt;
                                      x.IsGenericType &amp;&amp; x.GetGenericTypeDefinition() == typeof (IEntityWithTypedId&lt;&gt;) &amp;&amp;
                                      !isMultiTenantEntity);
      return shouldMap;
    }
{% endhighlight %}

It is then a case of deriving from AutomappingConfiguration to map the multi-tenant entities:
{% highlight csharp %}
using System;

namespace SharpArchitecture.MultiTenant.Data.NHibernateMaps
{
  /// &lt;summary&gt;
  ///
  /// &lt;/summary&gt;
  public class MultiTenantAutomappingConfiguration : AutomappingConfiguration
  {
    public override bool ShouldMap(Type type)
    {
      var shouldMap = IsMultiTenantEntity(type);
      return shouldMap;
    }
  }
}
{% endhighlight %}

A <a title="sample application" href="https://github.com/yellowfeather/SharpArchitecture-MultiTenant">sample application</a> is available on GitHub.
