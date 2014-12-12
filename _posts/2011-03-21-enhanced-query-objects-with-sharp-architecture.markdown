---
layout: post
published: true
title: Enhanced query objects with S#arp Architecture
categories: [ Sharp Architecture, Enhanced Query Objects ]
comments: true
alias: /2011/03/enhanced-query-objects-with-sharp-architecture/
author: chrisr
---

I've been using <a href="http://www.sharparchitecture.net/">S#arp Architecture</a> (and the Who Can Help Me structure) on a few projects for a while now and on the whole I'm very happy with it. However, Ayende's recent post <a id="viewpost_ascx_TitleUrl" title="Title of this entry." href="http://ayende.com/Blog/archive/2011/03/16/architecting-in-the-pit-of-doom-the-evils-of-the.aspx">Architecting in the pit of doom: The evils of the repository abstraction layer</a> hit a nerve, and got me thinking that maybe I've been applying the services (or tasks) / repository abstraction a bit too liberally.<a id="more"></a><a id="more-78"></a> The following day I saw this <a href="https://twitter.com/#!/hotgazpacho/status/48381984371777536">tweet</a> mentioning <a href="http://twitter.com/fabiomaulo">@fabiomaulo</a>'s <a href="http://fabiomaulo.blogspot.com/2010/07/enhanced-query-object.html">Enhanced Query Object</a> which looks just the ticket to solving this issue. So, I've updated my <a href="https://github.com/yellowfeather/SharpArchitecture-MultiTenant">SharpArchitecture-MultiTenant</a> project on GitHub to use enhanced query objects.

In the SharpArchitecture.MultiTenant.Data project I've created an NHibernate folder (and namespace) and added a base class for queries that provides access to the ISession (similar to the existing code in the Repository base class):
{% highlight csharp %}
using NHibernate;
using SharpArch.Data.NHibernate;

namespace SharpArchitecture.MultiTenant.Data.NHibernate
{
  public class NHibernateQuery
  {
    protected virtual ISession Session
    {
      get
      {
        var factoryKey = SessionFactoryKeyHelper.GetKey(this);
        return NHibernateSession.CurrentFor(factoryKey);
      }
    }
  }
}
{% endhighlight %}

As this project is enabled for multi-tenants I've also created a marker interface to indicate if the query is tenant specific:
{% highlight csharp %}
namespace SharpArchitecture.MultiTenant.Framework.Contracts
{
  public interface IMultiTenantQuery { }
}
{% endhighlight %}

In MultiTenantSessionFactoryKeyProvider, I've updated the GetKeyFrom method to test for implementation of the IMultiTenantQuery interface:
{% highlight csharp %}
public string GetKeyFrom(object anObject)
{
  var type = anObject.GetType();
  var isMultiTenant = type.IsImplementationOf&lt;IMultiTenantQuery&gt;() ||
                      type.IsImplementationOf&lt;IMultiTenantRepository&gt;() ||
                      IsRepositoryForMultiTenantEntity(type);
  return isMultiTenant
    ? GetKey()
    : NHibernateSession.DefaultFactoryKey;
}
{% endhighlight %}

Now everything is in place to create some queries, so for the list of Customers I've created an interface for the query as below:
{% highlight csharp %}
public interface ICustomerListQuery : IMultiTenantQuery
{
  IPagination&lt;CustomerViewModel&gt; GetPagedList(int pageIndex, int pageSize);
}
{% endhighlight %}

and an implementation:
{% highlight csharp %}
public class CustomerListQuery : NHibernateQuery, ICustomerListQuery
{
  public IPagination&lt;CustomerViewModel&gt; GetPagedList(int pageIndex, int pageSize)
  {
    var query = Session.QueryOver&lt;Customer&gt;()
      .OrderBy(customer =&gt; customer.Code).Asc;

    var countQuery = query.ToRowCountQuery();
    var totalCount = countQuery.FutureValue&lt;int&gt;();

    var firstResult = (pageIndex - 1) * pageSize;

    CustomerViewModel viewModel = null;
    var viewModels = query.SelectList(list =&gt; list
                            .Select(mission =&gt; mission.Id).WithAlias(() =&gt; viewModel.Id)
                            .Select(mission =&gt; mission.Code).WithAlias(() =&gt; viewModel.Code)
                            .Select(mission =&gt; mission.Name).WithAlias(() =&gt; viewModel.Name))
      .TransformUsing(Transformers.AliasToBean(typeof(CustomerViewModel)))
      .Skip(firstResult)
      .Take(pageSize)
      .Future&lt;CustomerViewModel&gt;();

    return new CustomPagination&lt;CustomerViewModel&gt;(viewModels, pageIndex, pageSize, totalCount.Value);
  }
}
{% endhighlight %}

As you can see, this code is using NHibernate projections and transforms to get a list of the required view models, bypassing the need for a data transfer object (DTO) and mappers. This is a trivial example, but in reality the query would be more complex and likely to flatten the object structure.

Now the controller itself can use the query interface to get the paged list of view models:
{% highlight csharp %}
public ActionResult Index(int? page)
{
  var customers = _customerListQuery.GetPagedList(page ?? 1, DefaultPageSize);
  var viewModel = new CustomerListViewModel { Customers = customers };
  return View(viewModel);
}
{% endhighlight %}

I quite like this solution; the unnecessary layers of abstraction are removed and the queries are nicely encapsulated, if anything more complex is required (e.g. multiple data sources) then it is always possible to fall back to the services / repositories style as before.
