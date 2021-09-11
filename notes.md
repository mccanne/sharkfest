## NOTES

Backstory is interesting: Stumbling upon a new data model while pcap hacking to converge search and analytics
-> will try to tell our story by cutting across the landscape (e.g., from proto to zeek to suricata.... the schema story... relational databases vs document-model query systems)

Why it's nice not having protos every...
 - You can still enforce schemas if you want but this policy is not dictated by the data architecture

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
