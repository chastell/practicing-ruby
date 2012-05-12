Persisting Relations in a Polyglot World
========================================

Hi, my name is Piotr Szotkowski and I was invited by Greg to write a
_Practicing Ruby_ article after he saw my RubyConf 2011 talk _Persisting
Relations Across Time and Space_
([slides](http://persistence-rubyconf-2011.heroku.com),
[video](http://confreaks.net/videos/657)). This is not a one-to-one text
version of that talk; rather, these are some thoughts on the topics of
[polyglot
persistence](http://architects.dzone.com/articles/polyglot-persistence-future)
and modelling relations between objects.

Persistence: Your Objects’ Time Travel
--------------------------------------

The first thing we need to ask ourselves when thinking about object persistence
is how can we dehydrate an object into a set of simple values – usually
strings, numbers, dates and boolean ‘flags’ – in a way that will let us
rehydrate it back at some point in the future, often on a completely unrelated
run of our application. With the bulk of contemporary Ruby programs being Rails
webapps this is so obvious we usually don’t even think about it; the
persistence is conviniently taken care of by ActiveRecord and we often actually
_start_ writing the application by defining database-oriented models of our
objects: `rails generate model person name bio:text height:float born:date
vip:boolean`, followed by `rake db:migrate`, takes care of any
behind-the-scenes machinery required to persist instances of our `Person`
class.

The main problem with the above (other than the fact that starting designing
your app by writing `rails` is [considered _losing the architecture game_ by
some](http://confreaks.com/videos/759)) is that it puts us into a tight tunnel
of relational database-driven design – and while many came back saying that the
light at the end is one of a truly glorious meadow and we should speed up to
get there faster, our actual options of taking detours, driving on the
shoulders and stopping for a bit to take a high-altitute view of the road ahead
are even more limited than the run of this metaphor. ActiveRecord’s handling of
model relations (`belongs_to`, `has_many`, `has_and_belongs_to_many`, etc.)
sometimes furthers the problem by giving us seemingly fit-all solutions which
are often quite useful, but end up requiring just-this-little-bit-more tweaking
– tweaking which accumulates over time.

Persistence in Practice
-----------------------

As mentioned above, persisiting an object means dehydrating it into a set of
simple values – but they way we do this depends heavily on the database backend
being used.

When it comes to the most popular case of relational databases (such as MySQL,
PostgreSQL or SQLite), we use tables for classes, rows for objects and columns
to hold a given object property across all instances of the same class. To
persiste an object, we serialise the given object’s properties down into table
cells with column types supported by the underlying database – but even in this
seemingly obvious case it’s worth to stop for a second and think.

Should we go for the lowest common denominator (strings, numbers and dates –
even booleans are not really cross-engine: e.g. MySQL presents them as one-bit
integers, `0` and `1`), should we use a given ORM’s ‘common ground’ (here
booleans are usually fair game, and the ORM can take care of exposing them as
`true` and `false`), or should we actually limit the portability while
leveraging a given RDBMS’s features? For example, PostgreSQL not only exposes
‘real’ booleans, but [a lot of other very useful
types](http://www.postgresql.org/docs/9.1/static/datatype.html) – such as
gemometric points and paths, network addresses, XML (with XPath query support
to search and filter by!) or arrays (so we can actually store a given blog
post’s tags in a single column in the `posts` table and query by
inclusion/exclusion just as well as with a separate join table).

Persisting objects in document databases (such as CouchDB or MongoDB) is
somewhat similar, but often quite a bit different; classes are usually mapped
to collections, objects to documents and object properties to these documents’
fields. While strings, numbers and dates are serialised similarly to relational
databases, document databases also usually allow us to store properties that
are arrays or hashes, and allow easy storage of related objects as nested
documents (the canonical example being comments for a blog post, in cases when
they’re most often requested only in the context of the given post). This
allows for all sorts of tradeoffs (potentially many fewer joins, but at the
price of more costly joins when they’re still needed and quite a lot more
designing up-front).

Other kinds of databases have still other approaches for serialising objects.
Key-value stores (like Redis) usually need the objects to be in an
already-serialised form (e.g., represented as JSON strings), but then there are
gems like [ROC](https://github.com/benlund/roc) which map simple objects
directly to their canonical Redis representations; graph databases (like Neo4j)
are centred around object relations, and often allow persisting objects as
schema-less nodes, akin to document databases; other storage types (like
directory services, e.g. LDAP) have their own object/persistence mapping
specifics as well.

One of my personal favourite ways of persisting objects is the `PStore`
library (distributed with Ruby) coupled with YAML serialisation. While
highly inefficient (compared to powerhouses like relational or document
databases), it’s often more than good enough for small applications, and its
simplicity can be quite a benefit.

Let’s assume for a second that we want to write [a small application for
handling quotes](https://github.com/chastell/signore) – what would be the
simplest way to persist them? See for yourselves:

```ruby
require 'yaml/store'
store = YAML::Store.new 'quotes.yml'

# quotes are author + text structures
Quote = Struct.new :author, :text

store.transaction do   # a read/write transaction…
  store['db'] ||= []
  store['db'] << Quote.new('Charlie Gibbs',
    'A database is a black hole into which you put your data.')
  store['db'] << Quote.new('Will Jessop',
    'MySQL is truly the PHP of the database world.')
end                    # …is atomically committed here

# read-only transactions can be concurrent
# and raise when you try to write anything
store.transaction(true) do
  store['db'].each do |quote|
    puts quote.text
    puts '-- ' + quote.author
    puts
  end
end
```

Saving the above file and running it prints the two quotes just fine:

```
$ ruby quotes.rb
A database is a black hole into which you put your data.
-- Charlie Gibbs

MySQL is truly the PHP of the database world.
-- Will Jessop
```

…but a real treat awaits when we inspect the `quotes.yml` file:

```yaml
---
db:
- !ruby/struct:Quote
  author: Charlie Gibbs
  text: A database is a black hole into which you put your data.
- !ruby/struct:Quote
  author: Will Jessop
  text: MySQL is truly the PHP of the database world.
```

This allows us to have an automated way to persist and rehydrate our `Quote`
objects while also allowing us to easily edit them and fix any typos right
there in the YAML file. Is it scalable? Probably not really, but [my current
database of email signatures](https://github.com/chastell/dotfiles/blob/aee1d31618e2e4ea88186eda163f29ebd72702d1/.local/share/signore/signatures.yml)
consists of 4,000 entries and works fast enough.

Sweet Relations, How Do They Work?
----------------------------------

Now that we’ve covered the idea of object persistence using various backends
it’s time to finally talk about relations between objects. Quite often it’s the
relations that are the crux of our application (even when we’re not building
another social network…), and the problem of their persistence is usually
overlooked and simplified to ‘let’s just use foreign keys and join tables where
needed’.

The way relations are canonically persisted depends greatly on the type of the
database. Contrary to their name, relational databases are not an ideal
solution for storing relations: their name comes from relations between the
rows of a single table (which translates to the assumption that objects of the
same class have the same property types), not from relations between objects of
potentially different classes, which end up being rows in separate tables.

Modelling relations in relational databases is quite complicated and depends on
the type of relation, its directionality and whether it carries any
relation-specific data. For example, an object representing a person can have
the relations such as having a particular gender (one-to-many relation), having
a hobby (many-to-many), having a spouse (many-to-many, with the relation
carrying additional data, such as start date of the relationship),
participating in an event (many-to-many, with additional data such as
participation role), being on two different ends of parental relation (having
parents and children), etc. Some of these relations (gender) can be stored
right in the `people` table, some need to be represented by having a foreign
key, other require a separate join table (potentially carrying any
relation-specific data). Dereferencing such relations mean crafring and
executing (potentially complicated) SQL `JOIN` queries.

Modelling relations in document databases is quite different. Some of the
relations (like the above-mentioned post/comments example) are best modelled
using embedded documents; while very useful in certain scenarios (retrieving
post with all of its comments), this approach might cause problems when new
features require cross-cutting through all of such embedded documents –
retrieving all of the comments by a given person or the list of the most recent
comments means scanning through the whole `posts` collection.

While some document databases employ implicit, foreign-key-like references
(like MongoDB’s DBRefs, which are two-key documents of the form `{ $ref:
<collection>, $id: <object_id> }`), dereferencing relations is usually a bigger
problem (due to the lack of standard approaches like SQL `JOIN` queries) and is
often done on the client side (even if it’s simplified greatly by tools like
[MongoHydrator](https://github.com/gregspurrier/mongo_hydrator)).

Key-value stores are, by definition, the least relation friendly backends – any
modelling requires explicit foreign keys that need to be managed client-side.
On the other end of the spectrum are graph databases: relations (modelled as
edges) can usually carry any data required, can as easily point in either or
both directions and are represented in the same way regardless of whether they
model a one-to-one, one-to-many or many-to-many relation. Graph databases also
allow for all kinds of data analysis/querying based on the relations themselves
– things like graph traversal or proximity metrics are much easier done (and
much faster) than in relationship databases.

Modelling Relations as Proper Objects
-------------------------------------

Now that we know the different ways (and issues with) persisting objects and
relations between them – is there a way to model relations that could be deemed
‘persistence independent’, or at least ‘not persistence driven’? One of such
approaches would be to model relations as proper objects in the system, akin to
how they’re modelled in graph databases.

With this approach relations would be objects that reference two other objects
and carry any additional data particular to a given relation (such as
participation role in a relation between a person and an event, start/end dates
of the given relation, etc.). This approach is the most flexible in schema-less
databases – document databases could have a separate collection of relations,
and different relations could store different types of data. In relational
databases this could be modelled either by separate tables (one per relation
type) or a common `relations` table storing the references to the related
objects and a relation type pointing to a table holding data for all relations
of this particular type/schema.

The main drawback of this approach is dereferencing – getting other objects
related to a given object at hand would be a two-step process: getting all of
the object’s relations (potentially only of a certain type) and then getting
all of the ‘other’ objects referenced by these relations. Note, however, that
this is exactly what we do every day with join tables for many-to-many
relations – so the drawback is mostly that this approach would apply to all of
the relations in the given system, not only many-to-many ones.

The main advantages of this approach are its simplicity (everything is an
object, relations just happen to share certain properties, like the object
identifiers they reference) and potential higher portability (by not driving,
nor tying the way relations are modelled to the given persistence approach).
Having relations as proper objects can also help in producing aggregated
statistics about the system (like ‘what are the hubs of the system – the most
connected objects, regardless of relation type’).

Additionally, when all of the objects in the system have unique identifiers
([of course PostgreSQL have a native type for
UUIDs…](http://www.postgresql.org/docs/9.1/static/datatype-uuid.html)),
relations no longer need to carry the information about the table/collection of
the referenced object; assuming the system has a way to retrieve an object
solely based on its UUID, relations become – in their simplest form – just
triples of 128-bit UUIDs (one identifying the relation and the other two the
referenced objects).

Object Databases
----------------

_\[…possibly solving all of this, but still somewhat in the future…\]_

Not Your Usual Persistence
--------------------------

_\[…Candy/Ambition example of transparent persistence and Ruby-like
querying…\]_

← Models | Persistence →
------------------------

_\[…decoupling models from their persistence altogether – some highlights from
the ‘Decoupling Persistence (Like There’s Some Tomorrow)’ talk…\]_
