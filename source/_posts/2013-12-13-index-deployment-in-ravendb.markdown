---
layout: post
title: "Index Deployment in RavenDb"
date: 2013-12-13 08:15
comments: true
categories: [RavenDb]
published: true
---
As I've said before I'm a big fan of [RavenDb](http://ravendb.net). I remember playing around with Raven for the first time back in 2010 and sitting there with a big smile on my face, it was obvious that a huge amount of work had been put into the developer experience.

Compared to the relational databases I was working with at the time everything just seemed so friction free. That said while Raven is schema free you still can need to include specific steps for it in your deployment process as you would for a relational database. Namely this might include your index definitions or perhaps some kind of document migration.

<!--more-->

In Raven while the key/value store is ACID index queries are BASE. Raven uses Lucene for indexes which are pre-computed by background processes as documents arrive which means that query results can be stale until the indexing process catches up with the latest document write activity.

Raven has a lot of smarts built into its query system, you can just perform ad-hoc queries and let Raven determine the appropriate index to use. If an appropriate index doesn't exist then Raven will just go ahead and create one on the fly.

That said when you have a large production database you might not want Raven to trigger a huge index rebuild when it receives a new query from your freshly deployed application. Along with ad-hoc indexes Raven also offers the ability to create static indexes. 

Below is a simple static index definition.
``` c#
public class SampleStaticIndex : AbstractIndexCreationTask<SomeDocument>
{
    public SampleStaticIndex()
    {
        Map = docs => from someDoc in docs
                        select new {someDoc.Name};

        Index(x=>x.Name, FieldIndexing.Analyzed);
    }
}
```

We can then explicitly query on this index as follows.
``` c#
var results = session.Query<SomeDocument, SampleStaticIndex>
					 .Where(x => x.Name.StartsWith("Foo"))
					 .ToList();
```

Finally people then typically add a static index creation task to their Global.asax when the application starts up.
``` c#
protected void Application_Start()
{
	// ...
	IndexCreation.CreateIndexes(typeof(SampleStaticIndex).Assembly, documentStore);
	// ...
}
```

This is where I feel there is room for improvement, I'd rather not have index creation / re-builds which are potentially a slow expensive process tied to the start-up on my application. What I'd prefer is to separate it out into specific step in the deployment process which would be something along the lines of..

* Create new indexes across all Raven instances in production. Where an existing index has changed the original index should be left in place. In effect we will have two versions of the index, one that is currently in use by the application and another that will be used by the new version of the application when it is deployed.
* After the indexing process has completed deploy the new version of the application.
* Once we are happy with the release drop any indexes that are no longer in use.

A quick search for existing tooling to enable this process lead me to [markedup.com's](https://markedup.com/) [Hircine](http://blog.markedup.com/2012/09/introducing-hircine-a-stand-alone-index-builder-for-ravendb/) tool which is a stand alone index builder for RavenDb. This does almost everything I need however it seems that they have [moved on](http://blog.markedup.com/2013/02/cassandra-hive-and-hadoop-how-we-picked-our-analytics-stack/) from Raven so it hasn't been updated in quite some time. I thought this morning I'd do a [quick spike](https://github.com/ronnieDrew/Hircine) to add the functionality I'm looking for to Hircine. The following is a quick run through the changes I made.

The standard index & multi-map index creation tasks now have corresponding "Versioned" tasks. These versioned indexes can then be setup in the exact same way as standard static instances but instead you inherit from a different base class, other than that everything works the same from the developers point of view.

``` c#
public class SampleVersionedIndex : AbstractVersionedIndexCreationTask<SomeDocument>
{
    public SampleVersionedIndex()
    {
        Map = docs => from someDoc in docs
                        select new {someDoc.Name};

        Index(x=>x.Name, FieldIndexing.Analyzed);
    }
}
```

To make this work Hircine will have to do the following.

* Determine the indexes which requiring versioning during deployment.
* Check if these indexes have existing versions already deployed, if the index definitions differ a new index will be created leaving the existing index intact.
* On request be able to remove versioned indexes that are not currently in use in the application.

On the client side we need also to make sure the correct index version is specified when issuing queries.

To achieve this what I did was override the <code>IndexName</code> property in my <code>AbstractVersionedIndexCreationTask</code> & <code>AbstractMultiMapVersionedIndexCreationTask</code> classes. The index name now has the index definition hashcode appended to it so when a definition is changed we should get a new index name. This has the benefit that the Raven client can request the correct index version without having to do any further work.

The interface <code>IVersionedIndex</code> is just added to have an easy way to identify versioned indexes in an assembly and obtain the non-versioned index name. There are four different "versioned" creation tasks, each of these implement <code>IVersionedIndex</code>.

* <code>AbstractVersionedIndexCreationTask&#60;TDocument&#62;</code>
* <code>AbstractVersionedIndexCreationTask&#60;TDocument, TReduceResult&#62;</code>
* <code>AbstractMultiMapVersionedIndexCreationTask</code>
* <code>AbstractMultiMapVersionedIndexCreationTask&#60;TReduceResult&#62;</code>

``` c#
public interface IVersionedIndex
{
    string GetIndexNameWithoutVersion();
}

public abstract class AbstractVersionedIndexCreationTask<TDocument> : AbstractIndexCreationTask<TDocument>, IVersionedIndex
{
    string _indexName;

    public override string IndexName
    {
        get
        {
            if (string.IsNullOrEmpty(_indexName))
                _indexName = VersionIndexName.Create(base.IndexName, CreateIndexDefinition());

            return _indexName;
        }
    }
    public string GetIndexNameWithoutVersion() { return base.IndexName; } 
}

class VersionIndexName
{
    public static string Create(string indexName, IndexDefinition indexDefinition)
    {
        return indexName + "-" + Convert.ToBase64String(indexDefinition.GetIndexHash());
    }
}
```

The only change I need to make to Hircine is then to support the deletion of inactive versioned indexes which is fairly straight forward. In the <code>Hicine.Core</code> the <code>IndexBuilder.RunAsync</code> method compiles a list of tasks to create the various required indexes, I've implemented a <code>DropIndexAsync</code> task and when the build instructions <code>DropInactiveVersionedIndexes</code> flag is set to true a set of tasks will be added to delete any inactive indexes.

``` c#
public Task<IndexBuildReport> RunAsync(IndexBuildCommand buildInstructions, Action<IndexBuildResult> progressCallBack)
{
    //Load our indexes
    var indexes = GetIndexesFromLoadedAssemblies();
    var tasks = new List<Task<IndexBuildResult>>();

    foreach (var index in indexes)
    {
        var indexInstance = (AbstractIndexCreationTask)Activator.CreateInstance(index);
        tasks.Add(BuildIndexAsync(indexInstance, progressCallBack));
    }

    if (buildInstructions.DropInactiveVersionedIndexes)
    {
        var versionedIndexes = GetVersionedIndexesFromLoadedAssemblies();
        foreach (var indexType in versionedIndexes)
        {
            var index = (AbstractIndexCreationTask)Activator.CreateInstance(indexType);
            var nonVersionedIndexName = ((IVersionedIndex)index).GetIndexNameWithoutVersion();
            var inactiveIndexes = _documentStore.DatabaseCommands
                                    .GetStatistics()
                                    .Indexes
                                    .Where(x => x.Name.StartsWith(nonVersionedIndexName) &&
                                                x.Name.Equals(index.IndexName) == false
                                    ).ToList();

            foreach (var indexToDrop in inactiveIndexes)
            {
                tasks.Add(DropIndexAsync(indexToDrop.Name, progressCallBack));
            }
        }
    }

    return Task.Factory.ContinueWhenAll(tasks.ToArray(), cont =>
    {
        var buildReport = new IndexBuildReport()
        {
            BuildResults = tasks.Select(x => x.Result).ToList()
        };

        return buildReport;
    });
}
```

All in all I'm not sure how I feel about this spike. Including the index definition hash in the index name smells wrong. Firstly you have a bunch of gibberish appended to the index name and it wouldn't be obvious in the management studio which index is actually the active one. Secondly hash codes are not guaranteed to be unique so I guess there's the chance that an index definition could be changed but have the same hash as the existing version, in this case the index would get updated with the latest definition. 

That said where you have indexes that operate over large numbers of documents and take significant time to rebuild it would be nice to have something similar to this.

The code for this spike is here - [https://github.com/ronnieDrew/Hircine](https://github.com/ronnieDrew/Hircine)