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

_\[…persisting objects in relational, document, key-value, graph, PStore etc.
databases…\]_

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
