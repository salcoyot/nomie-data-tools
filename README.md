![](http://snap.icorbin.com/Screen-Shot-2015-07-17-16-16-46/Screen-Shot-2015-07-17-16-16-46.png)

#Nomie 1.0 Sync
The purpose of this repo is to house various libraries, modules, snippets and knowledge about [Nomie's](http://nomie.io) syncing data model. Nomie (a life tracker for iOS and Android) supports data replication between the Nomie app and remote [CouchDB Servers](http://couchdb.apache.org/).

Nomie 1.0 will launch to the public with **one way replication only**. Meaning you can only push your data (and deletes) to your remote CouchDB but changes to your remote server will not be pushed back to your device (initially).

##Database Structure

Nomie expects your databases to be prefixed with your CouchDB username. For example, if you login to your CouchDB with **ThePickle**, then Nomie will use the database **ThePickle_events**.  

- `username_events` storage for all events (tracker taps, notes, and more coming soon) are stored in the this database. [see below for details on event documents]
- `username_trackers` storage for all trackers
- `username_meta` storage for group data, favorite comparisons, preferences, etc.

**Already have a CouchDB Server?**
Use the [Nomie Sync User Creator](https://github.com/happydata/nsync-createuser) to quickly create a new user, databases and security settings.

##Understanding the Events database

Each Nomie Event is stored as three documents (yes, three documents). This avoids indexes and dbsize issues that can occure on various devices. This method, while more verbose has worked signficantly more reliably both the server and the Nomie app. It also opens up three  ways of querying your Nomie data without ever using the performance robbing **include_docs**.

- **Time based** Get all events between date 1 and date 2
- **Tracker based** Get all events for this tracker between date 1 and date 2
- **Day based** Get all trackers that happened on this day at this time.

Events are stored in the `username_events` database.

Note: If you need GEO location, or Note data, you will need to use "include_docs:true" since GEO and note data is store in the document and not in the ID. Otherwise, stick to the insanely fast allDocs() w/out include_docs.


###By Time ID
Time based IDs are used to find events (for all trackers) that occurred between X and Y time.

`tick|tm|1437157872090|b547c3540eef451e66f42c0ff7648fa6|0`

**classifier** | **mode** | **javascript time** | **tracker id** | **charge**

```
var eventsDB = new PouchDB('http://[couchdb]/[username]_events');
eventsDB.allDocs({
	startkey : 'tick|tm|[start time]|0',
	endkey : 'tick|tm|[end time]|~'
}).then(function(results) { });
```

###By Tracker ID
Tracker based IDs are used to find events that happened for a specific tracker between X and Y time.

tick|pr|b547c3540eef451e66f42c0ff7648fa6|1437157872090|0

**classifier** | **mode** | **tracker id** | **javascript time** | **charge**

```
var eventsDB = new PouchDB('http://[couchdb]/[username]_events');
eventsDB.allDocs({
	startkey : 'tick|pr|[tracker id]|0',
	endkey : 'tick|tm|[tracker id]|~'
}).then(function(results) { });
```

###By Day ID
Day based IDs are used to find events that happened on a specific day, AND optionally for a specific tracker, AND optionally a time range.

`tick|dy|fri|01d0412cc9de43d9ec6c27a92e22770a|1425618843745|-1`

**classifier** | **mode** | **day** | **tracker id** | **javascript time** | **charge**

```
var eventsDB = new PouchDB('http://[couchdb]/[username]_events');
eventsDB.allDocs({
	startkey : 'tick|dy|fri|0',
	endkey : 'tick|dy|fri|~'
}).then(function(results) { });
```


##Decoding an ID
The following Javascript will decode an ID to a javascript object. Note: this code requires [Momentjs](http://momentjs.com).

```
var decodeNomieId = function (id) {
  var ida = id.split('|');
  var obj = {
    _id: id,
    type: ida[0]
  };
  var mode = ida[1];
  switch (mode) {
  case 'tm':
    obj.time = parseInt(ida[2]);
    obj.timeFormatted = moment(new Date(obj.time)).format('ddd MMM Do YYYY hh:mma');
    obj.parent = ida[3];
    obj.charge = parseInt(ida[4]);
    break;
  case 'dy':
    obj.time = parseInt(ida[4]);
    obj.timeFormatted = moment(new Date(obj.time)).format('ddd MMM Do YYYY hh:mma');
    obj.parent = ida[3];
    obj.charge = parseInt(ida[5]);
    break;
  case 'pr':
    obj.time = parseInt(ida[3]);
    obj.timeFormatted = moment(new Date(obj.time)).format('ddd MMM Do YYYY hh:mma');
    obj.parent = ida[2];
    obj.charge = parseInt(ida[4]);
    break;
  }
  return obj;

};

```
