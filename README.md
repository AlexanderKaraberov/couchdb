### CouchDB with custom extensions

This fork is used mostly as a playground and a bridgehead to push custom CouchDB extensions to the upstream repo. Currently three majors changes have been implemented.



### 1. Support for `validate_doc_read` functions (JS and native Erlang ones).

Idea and core implementation were taken from the rcouch and ported to a new CouchDB 2.x/BigCouch codebase with some enhancements. This functionality was originally implemented for some internal needs of the [SpotMe cloud engagment platform](https://spotme.com/spotme-cloud/) with the hope it will be merged eventually to the Apache CouchDB or [IBM Cloudant platform](https://console.bluemix.net/docs/services/Cloudant/basics/index.html#ibm-cloudant-basics) which uses CouchDB under the hood.

Detailed description taken from the [commit message](https://github.com/AlexanderKaraberov/couchdb/commit/faff575ef5c23874d08cc23b145cabeeac02ae12):

`validate_doc_read` implement doc-level read security model in order to prevent invalid or unauthorized document reads from proceeding. Unfortunately vanilla Apache CouchDB doesn't provide fine-grained doc read security model and all existing mechanisms (roles, [couch_peruser](http://docs.couchdb.org/en/master/config/couch-peruser.html) and so on) are very limited. Specification and interface of the validate_doc_read similar to the ones of validate_doc_update with the only change that it is executed on every document read.

### Performance notes
These functions have to be implemented only in Erlang in order to be executed inside of `couch_native_process`, otherwise implemented in JS they will lay waste to overall performance. Currently CouchDB sends all JS map functions in the design documents to the `couch_query_server` which spawns a pool of external `couch_js` OS processes (by default `proc_manager` spawns one OS process per design doc). This involves serialising/deserialising and sending strings back and forth via a slow standard I/O. And we know that the biggest complain about standard I/O is the performance impact from double copy (standard IO buffer -> kernel -> IO buffer). On the contrary, Erlang implementation of the VDR functions have almost zero overhead, because now map function will directly go to the `couch_native_process` which will tokenise (`erl_scan`), parse (`erl_parse`), call `erl_eval` and run a function.


### 2. Support for an indexed view-filtered changes feed.

Changes feed with [_view filter](http://docs.couchdb.org/en/2.2.0/api/database/changes.html#view) allows to use existing map function as the filter, but unfortunately it doesn’t query the view index files. Fortunately enough CouchDB's `mrview` has unused internal implementation of two separate B+ tree indexes namely: `Seq` and `KeyBySeq` which are responsible for indexing of the changes feed with view keys. This functionality marked as `fast_view` was never exposed to `fabric` cluster layer and `chttpd` API, because it has its own problems. Long story short, it appears to work in some simple cases, but quickly goes south once keys in views starting to be removed and then reappear. For instance when the same keys appear in a view, then removed with docs updated or deleted and then reappearing again with new updates, the `seq`-index breaks pretty quickly. Also another problem is that unlike DB's b-tree keys that are always strings, views keys in seq b-tree could be anything, so sorting might potentiall yield unexpected results. `Seq` index currently looks like this: `{UpdSeq, ViewKey} -> {DocID, EmittedValue, DocRev}`. Where `UpdSeq` number is the sum of the individual update sequences encoded in the second part of update sequence as base64. And `ViewKey` can be any mapped from JSON data type: binary, integer, float or boolean. Currently `mrview` uses Erlang native collation to find first key to start from, so consistent low limit key couldn't be constructed.
According to discussions in the dev-mailing list, hidden `seq` and `keyseq` indexes support will be completely removed in future and will be reimplemented from scratch. But nevertheless `seq`/`keyseq`-indexed changes feed proved to work for simple controlled scenarios. As a result aforementioned functionality has been extended in this fork and some bugs have been fixed (compaction of `seq`-indexes, b-tree reduce, some replication scenarios) and new functionality has been implemented namely: view-filtered replication with rcouch, support for custom `/_last_seq` and `_view/_info` endpoints and `fabric` interface to the underlying `mrview`'s `seq`/`keyseq`-indexes.
Additional `KeyBySeq` index allows to apply familiar view query params to changes feed such as `key`, `start_key`, `end_key`, `limit` and so on which allow to have a somewhat limited [channel-like](https://developer.couchbase.com/documentation/mobile/current/guides/sync-gateway/channels/index.html#introduction-to-channels) functionality in vanilla Apache CouchDB.


### 3. Support for bulk get with multipart response

**UPD** This functionality has been submitted to the upstream repo in this [PR](https://github.com/apache/couchdb/pull/1195) and hopefully will be merged soon. PR also has a detailed description and motivation of the feature.
