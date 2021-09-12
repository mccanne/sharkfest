# The Zed Project: Stumbling Upon a New Data Model
 while Hacking on Packets

> This README comprises a presentation I gave at Sharkfest '21
> at 8-9am on September 17, 2021.  You can reproduce all the examples
> herein from the tools referenced and files in this repo.
> In a sense, this is Part 2 of
> [my talk from 10 years ago](https://sharkfestus.wireshark.org/sf11)
> at Sharkfest '11.

## Abstract

If you've ever tried to assemble operational infrastructure for
search and analytics of network-oriented traffic logs, you know what a
daunting task this can be.  Roughly three years ago, we embarked upon a project
to explore search and analytics for Zeek and Suricata "sensors" running on live
network taps or over archived PCAP files.  Having experimented extensively
with well-known, open-source search and analytics systems and after talking
to a wide range of practitioners of such tech stacks, we noticed a recurring
and compelling _design pattern_: a search cluster is often deployed to hold recent logs
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
for cloud storage -- called a _Zed lake_ --- providing time travel,
live ingest, search indexes, and transactionally consistent views
across distributed workers.

## Introduction

* Some ancient history: PCAP, BPF, tcpdump
* Ten years ago: Stanford Sharkfest '11 and Riverbed
* Present: Up and at 'em with Brim and Zed!
    * Your 50s sure are different from your early 40s... :)
    * But I think I still got something left!

## Sharkfest '11

![Summary Slide from 2011](fig/summary-2011.png)

## Getting Back in the Game!

* Zed & Brim
    * (quick demo of pcap drag into Brim)
    * (pcaps/xxx.pcap)[pcaps/xxx.pcap]

![Brim App](fig/brim-grab.png)


## Our Research Team

Pretty early on, we realized we weren't a typical startup
maniacally focused on a go-to-market and instead embraced the idea that
we'd take our time as a "research project"... we are just now transitioning
from research to execution...

#### Front end
* James Kerr
* Mason Fish
#### Infrastructure
* Steve McCanne
* Noah Treuhaft
* Matt Nibecker
* Al Landrum (ex-Brim)
* Henri Dubois-Ferriere (ex-Brim)
#### Community + "Product" + Jack-of-all-trades
* Phil Rzewski
#### UC Berkeley Collaborators
* Amy Ousterhout
* Silvery Fu
* Sylvia Ratnasamy
* Joe Hellerstein

## Why not JSON + Elastic?

Douglas Crockford: JSON
* Just send a javascript data structure to a javascript entity
* _So much easier_ than XML, SOAP, RPC
* And node.js arrived on the backend and we had _full stack_

Shay Banon: Elastic
* Wrap Lucene Java libr in a REST API
* Post JSON docs to API
* Submit JSON search queries to API

_It's hard to make things easy._  They did it.

Brilliant, easy, simple.

## The Catch

Works great for many simple uses cases.

But the simplicity of JSON is a double-edged sword
* limited data types (object, array, string, number, bool, null)
* no schemas in JSON
* suboptimal format for scaleable analytics

For example, the Zeek TSV is rich and structured.
```
#fields	ts              uid                     id.orig_h       id.orig_p ...
#types  time            string                  addr            port      ...
1521911721.255387	C8Tful1TvM3Zf5x8fl	10.164.94.120	39681 ...
1521911721.411148	CXWfTK3LRdiuQxBbM6	10.47.25.80	50817 ...
```
But to get this structured data into Elastic...
* format as JSON and lose information
* configure extensive "mapping rules" in ingest pipeline
* mapping rules recover richness of data that was originally present

(TODO: figure of this process)

[Corelight's ECS mapping repo](https://github.com/corelight/ecs-mapping)

Hmm, _it's hard to make things easy_

This all creates complexity.

_It's hard to make things easy_

## The Bifurcation of Search and Analytics

Moreover, search is often not enough... need historical analytics

Leads to a bifurcation

* *Search*: OpenSearch, Elastic, Splunk
    * Unstructured logs or semistructured JSON
* *Analytics*: An Analytics Lake
    * "Cleaned-up data" in relational tables, e.g, ClickHouse, BigQuery, Snowflake
    * "Schema-siloed" Parquet files on S3

(TODO: figure of bifurcated pipeline)

Each piece is
* easy enough and sensible by itself,
* but when you assemble the pieces, things get complex fast!

_It's hard to make things easy._

## Schemas to the Rescue

Make sure all data conforms to pre-design set of schemas
* Elastic can recover rich structure even with JSON itermediary
* Analytics organized around relational table with schemas
* Parquet files with schemas for efficient columnar analytics

But there's another catch...
* It's all quite fragile
* _The downstream stuff_ breaks when upstream things change
* And why do you want to manage two different systems?

(TODO: figure of breakage)

## Policy & Mechanism

* An old adage says _you should separate policy from mechanism_
* Yet, Parquet and RDBMS tables combine schema with format
* If schemas are your policy for clean data _and_ tables/filters are your mechanism,
then these approaches might just lead to headaches...

## Schema Declarations cause Cognitive Overload

Said another way, from a dev perspective...

* If you are declaring schemas before you can write data...
* If you are creating relational tables before you can query...
* If you are writing code to talk to a schema registry...

... then you are probably doing things the hard way.

Your extra effort comes from having to handle policy and mechanism _at the same time_.

_It's hard to make things easy._

## A Concrete Example: Avro

register type Foo with the a schema registry get back a global ID.  Encode semi-structured into an efficient binary format and Bar-encoding tag it with ID.  Receiver fetches the Schema from the registry (and caches the binding) and decodes the Bar-encoding back to a Foo.

Alternatively, send a verbose schema definition with every record...

There's a better (and seemingly obvious) approach.  Put the type system in the data stream but do so efficiently...

Devil's in the details.  Getting type contexts right while being efficient was tricky but an elegant solution emerged Type Context

## The "A Ha" Moment

When we really started working on this problem, we realized we were working
not on a security app, per se, but a fundamental data model problem.

It's hard to make things easy...

* A _gentle slope_ throughout
* Lightweight, desktop-scale to large and more complex cloud deployment
* Things just work

We spend a lot of time fussing over the details, so if you find something
complicated or surprising, let us know and we'll try to fix!

ERGONOMICS

* "Once my data is in ZNG, everything is easy..."

* I can't explain it with words
* You just have to dip your toes in and try it out...


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

We wanted Zed to be a superset of JSON... ergonomics!
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

## Zed is statically typed

Unlike JSON, Zed is statically type and _comprehensive_
```
{
        v1: 1,
        v2: 1.5,
        v3: 1 (uint8),
        v4: 192.168.1.1,
        v5: 192.168.1.0/24,
        v6: [1,2,3],
        v7: [1(uint32),2(uint32),3(uint32)],
        v8: |["HEADS","TAILS"]|,
        v9: |{"key1":"value1","key2":"value2"}|,
        //XXX more
}
```
_It's hard to make things easy._

But, let's follow Crockford's lead!

Implied types!  Yeah!

Then decorators.

Data can be self-describing.  

No need to define a schema a shoehorn it all in.

E.g., what is the "type" of this object or "record" in Zed terminology:
```
echo '{"s":"hello","val":1,"a":[1,2],"b":true}' | zq -Z "cut typeof(this)" -
```
`this` refers to each input record in sequence.
This creates a new record with one field `TYPE` whose value is a
_type value_ indicating the type signature of the input.

The type of a type value is type _type_:
```
echo '{"s":"hello","val":1,"a":[1,2],"b":true}' | zq -Z "cut typeof(typeof(this))" -
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

The Zed language is a superset of SQL... ergonomics!

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
You want columnar, no problem!
```
zq -f parquet -o tables.parquet tables.zng
```
Oops, that didn't work
* Policy and mechanism intermixed again
* Have to specify schema before you can write to the format
* Schema same for all rows

Ok, we can fuse...
```
zq -f parquet -o tables.parquet "fuse" tables.zng
```
But _I had to change the data_ to shoehorn it into Parquet's assumption.

ZST is different...
* separation of schema-silo from data...
* heterogeneous sequence of records _of any type_
* columns self-organized around record types

(picture of columns)

```
zq -f zst -o tables.zst tables.zng
hexdump -C tables.zst
```
You can see the string values of each column are stored sequentially.

And the data didn't have to change to go into column format even
retaining the orginal order of records...
```
 zq -i zst tables.zst
 ```

## Zed Type Context

Show how type context works in ZNG

Show how type context drives columnar structure in ZST.

(contrast with systems that clean up data and store as parquet)

## Zed Inspired by Zeek

(this section subsumed by Zeek motivation from above?)

(Type back the relational muck to relateable pcap stuff.)

Why is all this type stuff important?

Compare/contrast Zeek and Suricata.

(This is where the Z comes from.)

Vision: clients will someday somehow express their data in
a format like Zed (like Zeek does!)

But we need to deal with messy data today...
* Zed types tame the problem
* Zed types make shaping easier
* Show how we shape suricata...

Show type context...

## Zed lake

put it all together in a lake

composable tools and search indexes... just a zng file.

xxx

built on a cloud storage model...
- everything is write-once immutable (no appends)
- everything is named with a globally unique ID
- transaction log has logical appends (as a new cloud object)
- garbage collect unreachable objects

Easy caching.  Can cache *everything* since everything is immutable and has a globally unique name.

no leader election because of cloud storage semantics.
state is detemined by XXX.

all lake state is stored in the cloud
  (user model still under dev)

now show some lake use cases that leverage Zed...

## threat intel join example / workflow

## main/live

main/live branching model for streaming pipelines
(work in progress, but power of approach is illustrated here)
90% of clean up happens on the live to main branch, then kick off index job

## Derived analytics

edge graph... from beacons work.
show how this query is not easily done with SQL

relate to an orchestration agent.
(could be triggered off commits...?)

Introspection using meta-queries... the power of Zed in such an approach.
Compare to database systems that have to design and implement internal fixed schemas for exporting introspection as relational tables.

## pcap lake example

Use brimcap to generate flow IDs

XXX brimcap join on flow (you can create a community ID but you don't have
to ... you just use the flow ID)
Zed can join on record values so you can just form a flow ID record and
do joins on that...

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

## threat intel join example / workflow

## join on "this" example...?

## .

End with pitch for help... we're focused on data platform and modular tools.
We'd love for community to get involved.

## Wrap Up

It's hard to make things easy ...

* Separate of policy/mechanism in data engineering
* Superset of JSON, relational tables
* Intuitive data shaping with first-class types
* Leverages the familiar Git design pattern
* Nice, intuitive UX in App, in API, in CLI commands

* Committed to open source
    * [github.com/brimdata/brim](http://github.com/brimdata/brim)
    * [github.com/brimdata/zed](http://github.com/brimdata/zed)
* Public slack
* Follow us on Twitter

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
