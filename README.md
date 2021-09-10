# The Zed Project: Stumbling Upon a New Data Model for
  | Search and Analytics while Hacking on Packets

> This README comprises a presentation I gave at Sharkfest '21
> at 8-9am on September 17, 2021.  You can reproduce all the examples
> herein from the tools referenced and files in this repo.

## Abstract

If you've ever tried to assemble operational infrastructure for
search and analytics of network-oriented traffic logs, you know what a
daunting task this can be.  Roughly three years ago, we embarked upon a project
to explore search and analytics for Zeek and Suricata "sensors" running on live
network taps or over archived PCAP files.  Having experimented extensively
with well-known, open-source search and analytics systems and after talking
to a wide range of practitioners of such tech stacks, we noticed a recurring
_design pattern_: a search cluster is often deployed to hold recent logs
for interactive queries, while some sort of data lake is deployed
in parallel to hold historical data for batch analytics.

We wondered if this bifurcation between search and analytics was fundamental
or if we could tackle this problem with a different approach.
We eventually concluded these silos arise, at least in part,
from the bifurcation of the underlying data models themselves:
search systems typically use the schema-less model of JSON while
analytics systems use more structured formats like Parquet (or relational tables),
enforcing _one-or-the-other_ design decisions.

In this talk, I'll describe a new data model called Zed --- designed to
unify the document model of JSON with the relational model of databases --
where our ultimate goal is to converge search, analytics, and ETL.
I will then discuss how we've leveraged the Zed data model
in a new query engine that operates over Zed data instead of JSON objects
or relational tables, and admits and a new query language that is a superset
of SQL and log-search style languages.  Finally, I'll outline our
_"work in progress"_ adapting the Zed system to a Git-like data lake
for cloud storage -- called a Zed lake --- providing time travel,
live ingest, search indexes, and transactionally consistent views
across distributed workers.

## Introduction

* Thanks & Background
* Zed & Brim
    * (quick demo of pcap drag into Brim)
    * (pcaps/xxx.pcap)[pcaps/xxx.pcap]
* A pivot from Zeek/security to Zed/data
    * how we realized the data model inspired by Zeek TSV was so fundamental

## The Bifurcation of Search and Analytics

* Search: OpenSearch, Elastic, Splunk
* Analytics: A Data Lake
    * Log files (JSON or TSV) on a NFS cluster
    * Schema-siloed Parquet files on S3
    * "Cleaned-up data" in relational tables, e.g, ClickHouse, BigQuery, Snowflake

(picture of bifurcated pipeline)

## The Catch

* There's a catch
* ETL pipelines that route semi-structured events to "tables" are fragile
* _The downstream stuff_ breaks when upstream things change

And by the way, why do you want to manage two different systems?

(picture of breakage)

## Policy & Mechanism

* An old adage says _you should separate policy from mechanism_
* Yet, Parquet, Avro, JSON-schema, and RDBMS tables combine schema with format
* If schemas are your policy _and_ your mechanism for clean data,
this might just lead to problems...

_\<rant\> An aside: Parquet is based on the Dremel work from Google.
Have a look and please let me know how long it takes you to understand
the record shredding/re-assembly algorithm.  I'm usually a quick leaner,
but I found this stuff all very confusing. \</rant\>_

## Schema Declarations cause Cognitive Overload

Said another way, from a dev perspective...

* If you are declaring schemas before you can write data...
* If you are creating relational tables before you can query...
* If you are writing code to talk to a schema registry...

... then you are probably doing things the hard way.

Your extra effort comes from having to handle policy and mechanism _at the same time_.

## Zed: A Better Way

What if _the mechanism_ were _self-describing data_:

* A comprehensive type system
* First-class types
* Type-adaptive operators

And what if _the policy_ were enforced externally by the _type system_?

Then, a schema is simply a special case of a record type...

## Devil's in the Details

How do we get to this split of mechanism and policy?

Let's get down in the weeds...

## Composable Tools

Before talking about the data model, let me outline the tools.

(draw picture)

* Search/analytics engine
* Pcap to Zed converter/indexer
* Lake storage model (details later)
* Service endpoints - REST API to ingest, query, organize a lake

We have taken a very modular, "composable tools" approach
* fabulous for dev, test, debug
* bite-sized pieces for learning the system

> run `zed -h` and walk through commands briefly
> zed lake vs zed api

Like the `docker` command, everything packaged under the `zed` command, though there
are two shortcuts:
* `zq` - `zed query` operates on files and streams
* `zapi`- `zed api` client to talk to service

## A Zed Primer

Let's start with JSON and we'll work our away over to relational tables.

We'll use `zq` to take an input, do no processing, and display it
in pretty-printed ZSON.

```
echo "..." | zq -Z -
```

We wanted Zed to be a superset of JSON and relational tables but let's
start with JSON:
* object
* array
* string
* number (float64)
* bool
* null

E.g., this command will convert the given object into "Zed":
```
echo '{"s":"hello","val":1,"a":[1,2],"b":true}' | zq -Z -
```
You will notice:
* don't need (but can have) quotes around field names
* otherwise, very familiar
* at the same time, very different

E.g., what is the "type" of this object or "record" in Zed terminology:
```
echo '{"s":"hello","val":1,"a":[1,2],"b":true}' | zq -Z "cut TYPE:=typeof(this)" -
```
`this` refers to each input record in sequence.
This creates a new record with one field `TYPE` whose value is a
_type value_ indicating the type signature of the input.

The type of a type value is type _type_:
```
echo '{"s":"hello","val":1,"a":[1,2],"b":true}' | zq -Z "cut TYPE:=typeof(typeof(this))" -
```

* These are _first-class types_
* A powerful means for data discovery and introspection

Zed also has all the expected data types, e.g., IP addresses, networks,
so we can cast the strings here into Zed native types...
```
echo '{"a":"128.32.1.1","n":"10.0.0.1/8"}' | zq -Z "a:=ip(a),n:=net(n)" -
```

## Union Types

The `typeof` operator can be applied to any field, e.g.,
```
echo '{"s":"hello","val":1,"a":[1,2],"b":true}' | zq -Z "cut TYPE:=typeof(a)" -
```
This gives `[int64]` i.e., an array of `int64`.

But JSON is dynamically typed whereas Zed is statically typed.
What if field `a` had mixed types?
```
echo '{"a":[1,"hello",true]}' | zq -Z "cut TYPE:=typeof(a)" -
```
This gives `(int64,string,bool)`!
* parentheses indicate a _union_ type
* so this is type array of a union of `int64`, `string`, and `bool`

Note: union types are essential in "shaping" and "fusing" values of different types.
e.g.,
```
echo '{a:1,b:1} {a:"hello",b:2}' | zq -Z fuse -
```
Note also that a sequence of records is valid ZSON so
* no need to put them in an array like JSON
* ZSON is also a _superset of NDJSON_

## Zed is a Superset of Relational Tables

Armed with a strong typing, we can tackle relational tables.

Start with with a simple example:
```
cat employee.csv
zq -Z -i csv employee.csv
```
Note that `id` and `field` are floating point numbers by default
(CSV doesn't tell use the types of things).
```
zq -Z -i csv "by typeof(this)" employee.csv
```
Let's clean that up...
```
zq -Z -i csv "id:=int64(id),phone:=int64(phone)" employee.csv
```
Let's save in a new file...
```
zq -z -i csv "id:=int64(id),phone:=int64(phone)" employee.csv > employee.zson
```
Now we have our relational table:
```
zq -f table  employee.zson
```

## Relational Zed

Zed is a superset of SQL

> Note: SQL support is currently experimental and early

zq -f table "SELECT name WHERE salary >= 250000" employee.zson

A _table_ is just a Zed type.

Let's look at another table...
```
zq -Z -i csv deals.csv
```
This time we'll clean it up through _shaping_:
```
zq -Z -i csv "type deal = {id:int64,name:string,customer:string,forecast:float64}; this:=cast(this,deal)" deals.csv
```
Note importantly the `(=deal)` type definition.

Let's shape both of the CSVs into a new file "tables.zson"...
```
zq -z -i csv "type deal = {id:int64,name:string,customer:string,forecast:float64}; this:=cast(this,deal)" deals.csv > tables.zson
zq -z -i csv "type employee = {id:int64,name:string,city:string,phone:int64,salary:float64}; this:=cast(this,employee)" employee.csv >> tables.zson
```

## SQL Tables as Zed Types

```
cat tables.zson
```
We can simply add a "FROM" clause to refer to a table by its type name:
```
zq -f table "SELECT name FROM employee WHERE salary >= 250000" tables.zson
zq -f table "SELECT name FROM deal WHERE forecast >= 200000" tables.zson
```
Of course, there is an easier way given the mixed nature of the Zed data model...
```
zq -f table "salary >= 250000 or forecast >= 200000" tables.zson
```

## SQL/Zed Joins

Since tables are just types, you can do JOINs too!

```
zq -f table "SELECT e.name AS NAME, d.forecast AS FORECAST FROM employee e JOIN deal d ON e.name=d.name" tables.zson
```
* You can aggregate too of course.  
* Unlike SQL, Zed has sets and set operators
```
zq -f table "SELECT e.name AS NAME, union(d.forecast) AS FORECAST FROM employee e JOIN deal d ON e.name=d.name GROUP BY NAME" tables.zson
```
Here it is in ZSON...
```
zq -z "SELECT e.name AS name, union(d.forecast) AS forecast FROM employee e JOIN deal d ON e.name=d.name GROUP BY name" tables.zson
```
The `|[ ... |]` syntax indicates a set.

## Zed Format Family

* ZSON - human readable like JSON (what I've been showing here)
* ZNG - performant, binary, compressed row-based format
* ZST - performant, binary, compressed column-based format
```
zq -o tables.zng tables.zson
hexdump -C tables.zng
```

## Zed Type Context

Show how type context works in ZNG

Show how type context drives columnar structure in ZST.

(contrast with systems that clean up data and store as parquet)

## Zed Inspired by Zeek

Why is all this type stuff important?

(This is where the Z comes from.)

Vision: clients will someday somehow express their data in
a format like Zed (like Zeek does!)

But we need to deal with messy data today...
* Zed types tame the problem
* Zed types make shaping easier
* aShow how we shape suricata...

Show type context...

## Mechanism/Policy Revisited

Now you can see the separation:
* You don't have to think about types _before you put data in Zed_
* You can shape the data whenever however you want
* You can this with the same format, in the same place

Policy may then dictate:
* What types you "are allowed to" query
* What types you allow into the cleaned up data pool
* What to do with data that has "the wrong shape"
    * because something upstream changes
    * let it in?
    * put it in an "error" pool
    * raise an alert

## Zed in the App

## relational tables

SELECT s.District, avg(sc.AvgScrMath),min(sc.AvgScrMath),max(sc.AvgScrMath)
FROM school s
LEFT JOIN satscore sc ON s.District=sc.dname
GROUP BY s.District | fuse

## Bio

Steve McCanne is the "Coding CEO" at Brim, a small startup
working on the open-source Zed Project and a new application called "Brim"
that leverages Zed.  Back in the days before the Web, Steve worked at
the Lawrence Berkeley National Laboratory where he developed BPF,
libpcap, the PCAP file format, and the tcpdump language and compiler,
while also working on the Real-time Transport Protocol (RTP) for
Internet video when the telcos claimed that real-time Internet
communication was impossible without end-to-end virtual-circuit guarantees.
(Guess who was right?)  After a brief stint in academia in the late '90s,
Steve crossed over to the dark side, became a tech entrepreneur,
and never looked back.  He has founded several startups and
took his '02 company and Sharkfest's sponsor, Riverbed, public in '06.
After many years working in other areas of tech, Steve has
returned to his roots, dabbling again with PCAPs, leading him to Zed
and a whole new way to approach network and IT observability data.


Zed is comprised of a human-readable form called ZSON and two binary, performant
formats for row and columnar layouts (called ZNG and ZST respectively).

## Outline / Extended Abstract

## NOTES

XXX brimcap join on flow (you can create a community ID but you don't have
to ... you just use the flow ID)
Zed can join on record values so you can just form a flow ID record and
do joins on that...

no leader election because of cloud storage semantics

all lake state is stored in the cloud
  (user model still under dev)

Backstory is interesting: Stumbling upon a new data model while pcap hacking to converge search and analytics
-> will try to tell our story by cutting across the landscape (e.g., from proto to zeek to suricata.... the schema story... relational databases vs document-model query systems)

* Stonebraker quote

Why unifying transport with data model is powerful

Why it's nice not having protos every...
 - You can still enforce schemas if you want but this policy is not dictated by the data architecture

Dremel was a nice approach but it just has one little flaw... adjacent rows must all have the same schema, i.e., you must declare the schema before writing data to a parquet file

## .

End with pitch for help... we're focused on data platform and modular tools.
We'd love for community to get involved.

## .

JOIN on this...

## .

composable tools and search indexes... just a zng file.

Don't forget: ERGONOMICS

## .

CHALLENGE OF SCHEMA-LESS QUERIES:

x == 1 AND s == "hello"

Do not know ahead of time if x or s are present.  Typos mysteriously return nothing without error whereas a mis-typed column name of a table in a relational database returns an error immediately and the query never even runs.

## .

Build your system X out of system X.  WHat does this mean?

Create a small kernel of concepts that are reusable and modular, and build bigger abstractions based on this.  

So what are our building blocks?

- The data model: everything is a sequence of Zed values (as ZNG, ZST, or ZSON)

- mutators

- aggregators (with partials)

- filters

(hmm, what else, this might be hard at this point)

## .

STUMBLING UPON...

zeek TSV -> but world doesn't understand this nice structure

So, we dumb it down into JSON...  <ex>

But systems like OpenSearch (formerly known as elastic) want structure.  No problem!  Tell OpenSearch that when it sees a certain pattern to turn into back into what it was at the source...

<picture of Foo turned into Bar with signaling to turn the Bar back into a Foo>

That's really inefficient!  Avro to the rescue.

register type Foo with the a schema registry get back a global ID.  Encode semi-structured into an efficient binary format and Bar-encoding tag it with ID.  Receiver fetches the Schema from the registry (and caches the binding) and decodes the Bar-encoding back to a Foo.

Alternatively, send a verbose schema definition with every record...

There's a better (and seemingly obvious) approach.  Put the type system in the data stream but do so efficiently...

Devil's in the details.  Getting type contexts right while being efficient was tricky but an elegant solution emerged Type Context
<explain all this about zng>

## .

Introspection using meta-queries... the power of Zed in such an approach.
Compare to database systems that have to design and implement internal fixed schemas for exporting introspection as relational tables.

## .

built on a cloud storage model...
- everything is write-once immutable (no appends)
- everything is named with a globally unique ID
- transaction log has logical appends (as a new cloud object)
- garbage collect unreachable objects

Easy caching.  Can cache *everything* since everything is immutable and has a globally unique name.

## .

edge graph... from beacons work.
show how this query is not easily done with SQL

## .

main/live branching model for streaming pipelines
(work in progress, but power of approach is illustrated here)
90% of clean up happens on the live to main branch, then kick off index job

## .

Zed is like JSON with types but a bit more.
Not JSON schema... the whole idea of attaching schemas to data seemed weird to me.  Don't you just want a type system?  Then you don't need the clunkiness of out of band schema definition.  A value just is what is.

If you flip it upside down like this then a schema is just a uniform type across Zed records.  It's just a special case of the type system.

## .

The system that shark appliances built a bunch of indexes for packet fields.

But there's been a trend of index-free search.
There was a trend to do search grep-style instead of building big bulk indexes.

(In DB world this is called table scan vs index scan...)

We think a hybrid solution makes the most sense (show picture of partial index, then index scan feeding data scan). This is an old idea from databases and we like it better...

Advantage don't need to index to run and in particular don't have to wait for indexes to be computed before results are calculated.  

You still could have a policy to index everything and compute an inverted keyword index for all the string fields and the string field representations of the non-string fields.  Of course, this would be expensive but you could choose (full-text search indexing not yet implemented in Zed, though full-text search is with a pool scan)

In Zed we call it a pool scan instead of a table scan.

## .

Bookends of talk will be the idea to use zeek connection ("conn") logs to search your pcap.
Conn logs can be treated as pointers to your pcaps.

For 90% of use cases, don't need to index a bunch of fields of every packet.  Just use zeek conn logs as the index and do your searches at the log level.

composable tools approach: brimcap is a tool to current flow indexes of PCAP files.

(Show pictures.)

Brim workflow:
  - drag a pcap into the window
  - create a data pool to hold the zeek/suricata logs
  - run brimcap analyze to generate logs
  - run brimcap index to generate flow index for fast packet extraction

Come back at end

INTRO:
backstory... stanford talked about the history of pcap and tcpdump and the work I did designing the tcpdump and translating it into the BPF VM.
I had gone off and worked on many other things (video over IP, WAN optimization) but recently I've come back to my roots a bit and returned the world of packets and PCAPs.
In this talk I'll describe the recent work I've been doing on a new data model called Zed, a search and analytics system based on the Zed data model, and an app Brim that uses Zed system.

# Brimcap out of box

* there is no -h like zed
* built-in help is limited/missing
* binaries are put in build/dist, build/dist/suricata, build/dist/zeek
* no make install, instead make build
