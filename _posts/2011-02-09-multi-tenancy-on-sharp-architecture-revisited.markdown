---
title: Multi-tenancy on S#arp Architecture Revisited
date: 2011-02-09 00:00:00 Z
categories:
- Sharp Architecture
- Multi-tenancy
layout: post
comments: true
alias: "/2011/02/multi-tenancy-on-sharp-architecture-revisited/"
author: chrisr
---

Following my [previous post](/blog/multi-tenancy-on-sharp-architecture) some issues were pointed out with the implementation, the main one being that the correct repository implementation was not resolved from an IRepository&lt;T&gt; interface (see the Google Group <a title="use custom IRepository interface in SharpModelBinder Options " href="http://groups.google.com/group/sharp-architecture/browse_thread/thread/3d8b190ada63a06b" target="_self">discussion</a> for more details).<a id="more"></a><a id="more-62"></a>

As mentioned in the discussion I have made some minor modifications to <a title="S#arp Architecture" href="http://sharparchitecture.net/">S#arp Architecture</a> to solve this problem and better support multi-tenancy. You can see the changes in the <a title="Enabling multi-tenancy" href="https://github.com/sharparchitecture/Sharp-Architecture/pull/1" target="_self">pull request</a>, but it looks like they are now included in the <a title="1.9.5 Released" href="http://groups.google.com/group/sharp-architecture/browse_thread/thread/2091f202966654dc">latest release</a>.

The changes are very minor, and centre around the introduction of an interface, ISessionFactoryKeyProvider, so that it is possible to get the session factory key without having to use an attribute:
{% highlight csharp %}
namespace SharpArch.Data.NHibernate
{
  public interface ISessionFactoryKeyProvider
  {
    /// &lt;summary&gt;
    /// Gets the session factory key.
    /// &lt;/summary&gt;
    /// &lt;returns&gt;&lt;/returns&gt;
    string GetKey();

    /// &lt;summary&gt;
    /// Gets the session factory key.
    /// &lt;/summary&gt;
    /// &lt;param name="anObject"&gt;An optional object that may have an attribute used to determine the session factory key.&lt;/param&gt;
    /// &lt;returns&gt;&lt;/returns&gt;
    string GetKeyFrom(object anObject);
  }
}
{% endhighlight %}

I've created a default implementation of this interface that just delegates getting the key from the existing SessionFactoryAttribute.
{% highlight csharp %}
namespace SharpArch.Data.NHibernate
{
  /// &lt;summary&gt;
  /// Implementation of &lt;see cref="ISessionFactoryKeyProvider" /&gt; that uses
  /// the &lt;see cref="SessionFactoryAttribute" /&gt; to determine the session
  /// factory key.
  /// &lt;/summary&gt;
  public class DefaultSessionFactoryKeyProvider : ISessionFactoryKeyProvider
  {
    public string GetKey()
    {
      return NHibernateSession.DefaultFactoryKey;
    }

    /// &lt;summary&gt;
    /// Gets the session factory key.
    /// &lt;/summary&gt;
    /// &lt;param name="anObject"&gt;An object that may have the &lt;see cref="SessionFactoryAttribute"/&gt; applied.&lt;/param&gt;
    /// &lt;returns&gt;&lt;/returns&gt;
    public string GetKeyFrom(object anObject)
    {
      return SessionFactoryAttribute.GetKeyFrom(anObject);
    }
  }
}
{% endhighlight %}

I've also added a helper class SessionFactoryKeyHelper:
{% highlight csharp %}
using SharpArch.Core;

namespace SharpArch.Data.NHibernate
{
  public static class SessionFactoryKeyHelper
  {
    public static string GetKey()
    {
      var provider = SafeServiceLocator&lt;ISessionFactoryKeyProvider&gt;.GetService();
      return provider.GetKey();
    }

    public static string GetKey(object anObject)
    {
      var provider = SafeServiceLocator&lt;ISessionFactoryKeyProvider&gt;.GetService();
      return provider.GetKeyFrom(anObject);
    }
  }
}
{% endhighlight %}

Now whenever S#arp Architecture requires a session factory key we can use the helper class, rather than using SessionFactoryAttribute e.g. in the Repository implementation the code is changed from:
{% highlight csharp %}
protected virtual ISession Session {
  get {
    string factoryKey = SessionFactoryAttribute.GetKeyFrom(this);
    return NHibernateSession.CurrentFor(factoryKey);
  }
}
{% endhighlight %}

to:
{% highlight csharp %}
protected virtual ISession Session {
  get {
    string factoryKey = SessionFactoryKeyHelper.GetKey(this);
    return NHibernateSession.CurrentFor(factoryKey);
  }
}
{% endhighlight %}

<em>Note: this change to the Repository implementation means that the MultiTenantRepository class from my <a title="Multi-tenancy on S#arp Architecture" href="http://www.yellowfeather.co.uk/blog/multi-tenancy-on-sharp-architecture/" target="_self">previous post</a> is no longer required.</em>

Similar changes are also made to TransactionAttribute and EntityDuplicateChecker.

If you do not need multi-tenancy, or are happy to use the existing TransactionAttribute to specify the session factory key, then you just need to register the DefaultSessionFactoryKeyProvider implementation in the container:
{% highlight csharp %}
container.AddComponent("sessionFactoryKeyProvider", 
  typeof(ISessionFactoryKeyProvider),
  typeof(DefaultSessionFactoryKeyProvider));
{% endhighlight %}

But if you want to provide the session factory key by any other means, it is just a case of implementing and registering your implementation of ISessionFactoryKeyProvider.

In my <a title="SharpArchitecture-MultiTenant" href="https://github.com/yellowfeather/SharpArchitecture-MultiTenant" target="_self">sample application</a> I've implemented it as below:
{% highlight csharp %}
using System;
using System.Linq;
using SharpArch.Data.NHibernate;
using SharpArchitecture.MultiTenant.Framework.Contracts;
using SharpArchitecture.MultiTenant.Framework.Extensions;
using SharpArchitecture.MultiTenant.Framework.Services;

namespace SharpArchitecture.MultiTenant.Framework.NHibernate
{
  public class MultiTenantSessionFactoryKeyProvider : ISessionFactoryKeyProvider
  {
    private readonly ITenantContext _tenantContext;

    public MultiTenantSessionFactoryKeyProvider(ITenantContext tenantContext)
    {
      _tenantContext = tenantContext;
    }

    public string GetKey()
    {
      var key = _tenantContext.Key;
      return string.IsNullOrEmpty(key) ? NHibernateSession.DefaultFactoryKey : key;
    }

    public string GetKeyFrom(object anObject)
    {
      var type = anObject.GetType();
      return IsMultiTenantRepository(type) || IsRepositoryForMultiTenantEntity(type)
        ? GetKey()
        : NHibernateSession.DefaultFactoryKey;
    }

    public bool IsMultiTenantRepository(Type type)
    {
      return type.IsImplementationOf&lt;IMultiTenantRepository&gt;();
    }

    public bool IsRepositoryForMultiTenantEntity(Type type)
    {
      if (!type.IsGenericType) {
        return false;
      }

      var genericTypes = type.GetGenericArguments();
      if (!genericTypes.Any()) {
        return false;
      }

      var firstGenericType = genericTypes[0];
      return firstGenericType.IsImplementationOf&lt;IMultiTenantEntity&gt;();
    }
  }
}
{% endhighlight %}

This makes use of a couple of marker interfaces IMultiTenantEntity and IMultiTenantRepository to decide whether we need to get the tenant, or the default, session factory key. Actually getting the tenant session factory key is accomplished by the implementation of ITenantContext.
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

I've update my <a title="SharpArchitecture-MultiTenant" href="https://github.com/yellowfeather/SharpArchitecture-MultiTenant" target="_self">sample application</a> on GitHub with these changes and to run against  S#arp Architecture v1.95.
