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

_\[…relations between objects are often the crux of our application and the
problem of their persistence is usually overlooked…\]_

_\[…the way relations are ‘canonically’ persisted depends greatly on the type
of the database backend: keys, embedding, graph edges…\]_

_\[…modelling relations as proper domain objects can yield surprising results,
and their persistence can be greatly simplified…\]_

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
