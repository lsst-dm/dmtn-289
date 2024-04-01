# Caching Database Content in Butler

```{abstract}
There are dimension records and collection information that are used heavily when interacting with a Butler.
This note describes a possible scheme for caching these records to reduce database access.
```

## Dimension Records and the Data Coordinate Container

We currently cache all records of all "governor" dimensions (`instrument` and `skymap`) and anything marked as using a particular record storage manager class in the dimensions configuration (`physical_filter`, `detector`, and I think `band`).

The dimension records that we cache are added very rarely, modified in place extremely rarely, and never deleted (except through manual database surgery by admins) - if needed for cache correctness I think we could require modifications and deletions to require downtime.
If they are in a client-side cache, it is entirely reasonable to expect the user to manually refresh if they think their cache may be out of date.
It also should be fine to do a full refresh on cache misses (on client or server), since those will extremely rare and we don’t (I think) rely on cache misses meaning a record does not exist anywhere.

It would be better for butler startup time to not pull down the complete dimension cache at startup (as we do today), but we can probably pull the entire cache down the first time it is needed; I don’t think the size of the cache will ever be large enough that it will be worthwhile for us to do more fine-grained fetches.

The dimension record cache is used to expand data IDs, and I suspect the cache benefits here are pretty huge given how frequently the same `instrument` or `skymap` record appears in typical sets of data IDs.
There is at least one important case (raw ingest) where the caching avoids database calls while expanding data IDs entirely (because all dimensions needed other than `exposure` is cached, and the `exposure` records are being created).

Data ID expansion also plays a role in validation of new datasets; usually this is backstopped by foreign key constraints, but see {jira}`DM-40818` for an example where it isn’t (but probably should be).
Data ID expansion failure will also occur before datastore artifacts are written, while foreign key constraint failures will happen later (at least in the DMTN-249 future), so they’re easier to recover from.

A lesson recently learned from QG generation optimization is that dimension record "keys" (in caches and elsewhere) should be the tuple of required-only data ID values; these are many orders of magnitude faster to hash and compare than `DataCoordinate` instances.
We should optimize extracting this tuple from `DimensionRecord` and define the cache data structure to be something like:

```python
DimensionRecordCache: TypeAlias = dict[str, dict[tuple[DataIdValue, ...], DimensionRecord]]
```

where the first `str` key is the dimension element name.
I envision our new `DataCoordinate` container class being something like this internally:

```python
class DataCoordinateContainerGuts:
    dimensions: DimensionGraph
    values: Iterable[tuple[DataIdValue, ...]]
    records: DimensionRecordCache
```

with some of the per-dimension-element mappings nested in `records` being the complete client-side cache of all records for that element (the data ID container doesn’t care if it has extra records) while others are non-cached records fetched directly for these data IDs.

I also envision having the `DataCoordinate` implementation yielded by iterating over this container deferring the actual lookups of `DimensionRecords` until someone actually does e.g., `data_id.records["exposure"]`, since most of the time people iterate over the data IDs without actually accessing any records.


```{warning}
I’m a bit worried that we’re wasting a ton of database space by stuffing `instrument` and `skymap` names directly into a lot of tables (especially `dataset_tags_*`) where an autoincrement `uint16` would be perfectly adequate.
Changing this would be a big pain (in migration and code), and I’m raising it here only because if we go that route we’d really want a bidirectional mapping from ID to name to be part of any caching we do.
Doing the same for cached non-governor dimensions might help a bit, but I think would make less of a difference in practice, since our most numerous datasets are `{instrument, visit, detector}` or `{instrument, exposure, detector}` and we don’t put the implied `{physical_filter, band}` dimensions in `dataset_tags_*`.
```

## Dataset Types

We currently aggressively cache all dataset types defined in the repository.
This cache is actually a fundamental part of the manager instance structure: we have one `DatasetRecordStorage` instance for each `DatasetType`.
This is an implementation choice the `DatasetRecordStorageManager` ABC was designed to not assume (i.e., it should be possible to make `DatasetRecordStorage` instances on-the-fly instead), but as an ABC that has only ever had one real implementation it may not hold up.

Dataset types are usually identified by name, but we use an autoincrement integer ID for the primary and foreign keys in the database.
Our current caching allows either to be used as a lookup key.

Dataset types are rarely added, only very rarely removed, and never modified in place.

If dataset types are held are in a client-side cache, it is entirely reasonable to expect the user to manually refresh if they think their cache may be out of date, as users can be expected to know if they’re looking at something that might be changed by another client.
A server-side of cache would probably need to rely on certain dataset types being marked as frozen somehow, and limit its caching to just these.

Caching all dataset type names on the client currently lets us implement regex/glob matching in Python there.
We probably can’t have that kind of cache on the server, since we’d have no way of detecting cache misses and we can’t leave refreshes to the user.
We’d need to move our pattern-matching logic to the database instead.

The set of all dataset types will be larger than a the set of all cacheable dimension records but much smaller than the set of all collections.
We should try to use actual numbers when considering the cost of complete vs. partial caching here.

We need the dimensions associated with each dataset type name to do even basic aspects of dataset query construction.
This is simpler if all dataset types are cached up front, but that doesn’t make it necessary; we could pull down dataset type definitions when a dataset type is added to a query (but before the query is executed) instead, and this is probably better for butler startup time.
Dataset queries are actually only ever executed for one dataset type at a time, because different dataset types can have different dimensions and hence different columns; the chaining together of queries over multiple dataset types is purely a Python/client-side thing, and a major question for the `RemoteButler` query system is whether this aggregation over dataset types remains fully client-side or moves to the server (more on this later when we cover collection summaries).

## Collections and Collection Summaries

**IN PROGRESS**

We currently aggressively cache all collections in the repository.
This includes:

* the types of all collections
* the members of `CHAINED` collections
* some never-used metadata specific to `RUN` collections
* "summaries" of all `RUN`, `TAGGED`, and `CALIBRATION` collections:
    * the names of all dataset types with datasets in the collection
    * the values of all governor dimensions used in datasets in the collections.

Collection summaries are not updated when datasets are deleted; they always represent what may exist, not what does exist.

Collections are usually identified by name.
The manager system can be configured to use an autoincrement integer ID for the primary and foreign keys in the database, but while this is the default for new repositories it was not the configuration used when making our current major repos (first at NCSA, and then when replicating those to USDF and IDF).
Our current caching allows either to be used as a lookup key when the autoincrement integers are in play.

Collections are added and deleted frequently.
`CHAINED` collection memberships can be modified in place.
Collection summaries are modified whenever datasets are inserted, associated, or certified.
The vast majority of the collections in a DR database will be completely frozen, but we don’t have a way of marking those right now.

I think we could get away with a non-complete client-side cache populated incrementally on cache misses, and leave clearing or refreshing the cache as a manual user-called method.

Similar caching on the server would be fine when the collections are frozen, and probably impossible when they are not.

Like dataset types, caching all collection names on the client also lets us implement regex/glob matching in Python there, and again not having that kind of cache means we’d have to move the pattern matching to the DB.

But there will definitely be too many collections for all clients to aggressively fetch all collection information in the future.
There will almost definitely be too many collections for all clients to aggressively fetch all information about `CHAINED` collections the user can access (it might be possible, but we’d at least have to lean on Campaign Management to do things certain ways), so we’re best off planning to have a only incomplete caches of collections.

`CHAINED` collection memberships are used to "flatten" any collection search path into a sequence of `RUN`, `TAGGED`, and `CALIBRATION`  collections that can actually be queried for datasets.
This is another fundamental part of dataset query construction.
One advantage of caching all `CHAINED` collection memberships up front is that we can do this without any additional database queries; the bulk query to fetch all memberships is much faster than a single query to fetch the members of just one `CHAINED` collection.
If we do not have a complete `CHAINED` membership cache, an alternative would be to write some moderately complex SQL that could fetch the membership of all `CHAINED` collections nested under sequence of collection names, to avoid having many tiny queries to resolve highly-nested `CHAINED` collections.
I gather that sort of thing is possible with CTEs, but I’ve never done it.

Collection summaries are also a part of dataset query construction: after flattening the collection search path, we filter that path down to just the collections that have the dataset type we’re querying for (recall that we always query for datasets one dataset type at a time) and have governor dimensions consistent any data ID constraint provided by the user.
In the vast majority of cases this collapses long collection search paths down to a single RUN collection.
This is a massive simplification of the SQL query we end up passing to the DB, especially when doing a find-first search, and it definitely helps the DB answer those queries more quickly.

Collection summaries are also used to set butler "defaults": when a butler is initialized with a collection search path, and that search path has only one value for a particular governor dimension in all of its collections, that value is used as a default in all data IDs that require a value for that dimension but do not provide one.

## Ideas for the future

* We should think about a way to mark collections as frozen so that we can know when they’re safe to cache (especially on the server).
  This could involve a new column in the DB or some combination of configuration and pattern-matching.
* We may or may not want the same thing for dataset types -- those cannot be modified in place, and we may be able to use their autoincrement IDs to prevent deletion+re-creation masquerading as in-place modification.
* We should reimplement our dataset type and collection pattern-matching in the database.
  I know PostgreSQL supports regular expressions, and it seems SQLite can with some plugin work.
  We may have to limit pattern-matching support to some minimal common subset.
* I think we should keep dimension record caching on the client, and keep caching all records of the elements we’ve marked as cacheable.
  Given that we need to have essentially the same data structures on the client anyway just to represent query results, I don’t think we gain anything in versioning/maintenance by moving those caches to the server.
  I would like to defer fetching those until first use, instead of doing the fetches at construction like we do now.
* I’m not sure whether the dataset type and collection caches should stay on the client or go to the server or both, and I could imagine that we’ll want client-side caching of user collections but server-side caching of fixed DR collections.
    * It depends a lot on where query construction happens, and especially that aggregation over dataset queries when there are multiple dataset types -- is there one REST server call for each dataset type, or one REST server call for all dataset types?
    * Server-side caching of common frozen collections is probably important even if there is also client-side caching of those, since almost every client is going to want them.
    * In many contents, we probably want to identify the dataset types to cache as those referenced by the summaries of the collection’s we’re caching.
    * I remain unconvinced that there’s much to be gained in practice in terms of versioning/maintenance by keeping caching out of the client, again because I think the data structures that need to be on the client don’t change much (e.g., maybe we could keep `CollectionSummary` and `CollectionRecord` off the client, but as those are very closely tied to the SQL table definitions they’re already hard to evolve).
* Instead of caching dataset type and collection information directly in the Butler client, I could imagine those caches being scoped to a context manager that’d be fundamental part of the new query interface.
    * I’ve always intended to include a context manager there somewhere in that new interface; it solves a lot of problems in `DirectButler` w.r.t. open connections/transactions, and it might help with `RemoteButler` problems like cleaning up server-side parquet pagination files, too.
    * How much do we need caches for operations other than the query system?
