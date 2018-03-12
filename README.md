PG Delta Stream
===============
Introduction
------------
PG Delta Stream uses PostgreSQL's logical decoding feature to stream table changes over websockets. 

PostgreSQL logical decoding allows the streaming of the Write Ahead Log (WAL) via an output plugin from a logical replication slot. An output plugin is required to decode the changes into a user consumable format. We use [wal2json](https://github.com/eulerto/wal2json) as the output plugin to get changes as JSON objects.

Overview of PGDeltaStream
-------------------------
When a replication slot is created, Postgres takes a snapshot of the current state of the database and records the consistent point from where streaming is supposed to begin. To facilitate retrieving data from the snapshot and to stream changes from then onwards, the workflow as been split into 3 phases:

1. Init: Create a replication slot

2. Snapshot data: Get data from the snapshot

3. Stream: Stream the WAL changes from the consistent point over a websocket connection

Requirements
------------
- PostgreSQL 10 running on port `5432` with the [wal2json](https://github.com/eulerto/wal2json) plugin

Configuration
-------------
To use the logical replication feature, set the following parameters in `postgresql.conf`:

```
wal_level = logical
max_replication_slots = 4
```

Further, add this to `pg_hba.conf`:

```
host    replication     all             127.0.0.1/32            trust
```

Restart the postgresql service.

Launching
---------
Launch the application server on localhost port 12312:
```
go run hasuradb localhost localhost:12312
```

Usage
-----

**Init**

Call the `/v1/init` endpoint to create a replication slot and get the slot name. 

```bash
$ curl localhost:12312/v1/init 
{"slotName":"delta_face56"}
```

Keep note of this slot name to use in the next phases.

**Get snapshot data**

To get data from the snapshot, make a POST request to the `/v1/snapshot/data` endpoint with the table name, offset and limit:
```
curl -X POST \
  http://localhost:12312/v1/snapshot/data \
  -H 'content-type: application/json' \
  -d '{"table": "test_table", "offset":0, "limit":5}'
```

The returned data will be a JSON list of rows:

```json
[
  {
    "id": 1,
    "name": "abc"
  },
  {
    "id": 2,
    "name": "abc1"
  },
  {
    "id": 3,
    "name": "abc2"
  },
  {
    "id": 4,
    "name": "val1"
  },
  {
    "id": 5,
    "name": "val2"
  }
]
```

Note that only the data upto the time the replication slot was created will be available in the snapshot. 



**Stream changes over websocket**

Connect to the websocket endpoint `/v1/lr/stream` along with the slot name to start streaming the changes:

```
ws://localhost:12312/v1/lr/stream?slotName=delta_face56
```

The streaming data will contain the operation type (create, update, delete), table details, old values (in case of an update or delete), new values and the `nextlsn` value. 

The query:

```
INSERT INTO test_table (name) VALUES ('newval1');
```
will produce the following change record:
```json
{
  "nextlsn": "0/170FCB0",
  "change": [
    {
      "kind": "insert",
      "schema": "public",
      "table": "test_table",
      "columnnames": [
        "id",
        "name"
      ],
      "columntypes": [
        "integer",
        "text"
      ],
      "columnvalues": [
        3,
        "newval1"
      ]
    }
  ]
}
```

The `nextlsn` is the Log Sequence Number (LSN) that points to the next record in the WAL. To update postgres of the consumed position simply send this value over the websocket connection:

```
{"lsn":"0/170FCB0"}
```

This will commit to Postgres that you've consumed upto the WAL position `0/170FCB0` so that in case of a failure of the websocket connection, the streaming resumes from this record.