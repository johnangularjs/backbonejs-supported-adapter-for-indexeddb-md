[![build status](https://secure.travis-ci.org/superfeedr/indexeddb-backbonejs-adapter.png)](http://travis-ci.org/superfeedr/indexeddb-backbonejs-adapter)

#Backbonejs adapter for IndexedDB

# Browser support and limitations

In Firefox, `backbone-indexeddb.js` should work from 4.0 up; but it


Chrome 11 and later are supported. (Chrome 9 and 10 should also work but are untested.) In Chrome 11, `backbone-indexeddb.js`

* will work with `file:///` URLs, but
* poses some hard size limit (5MB? quantity untested) unless Chrome is started with `--unlimited-quota-for-indexeddb`, with apparently no way to request increasing the quota.

Other browsers implementing the Indexed Database API Working Draft should work, with some of these limitations possibly cropping up or possibly not. Support and ease of use is expected to improve in upcoming releases of browsers.

# Tests

Just open the <code>/tests/test.html</code> in your favorite browser. (or serve if from a webserver for Firefox, which can't run indexedDB on local file.)

# Node.js Support

This is quite useless to most people, but there is also an npm module for this. It's useless because IndexedDB hasn't been (yet?) ported to node.js.
It can be used in the context of [browserify](https://github.com/substack/node-browserify) though... and this is exactly why this npm version exists.

# Implementation

## Database & Schema

Both your Backbone.js model and collections need to point to a `database` and a `storeName` attributes that are used by the adapter.


### Example

```js
var database = {
	id: "my-database",
	description: "The database for the Movies",
	migrations : [
		{
			version: "1.0",
			before: function(next) {
			    // Do magic stuff before the migration. For example, before adding indices, the Chrome implementation requires to set define a value for each of the objects.
			    next();
			}
			migrate: function(transaction, next) {
				var store = transaction.db.createObjectStore("movies"); // Adds a store, we will use "movies" as the storeName in our Movie model and Collections
				next();
			}
		}, {
			version: "1.1",
			migrate: function(transaction, next) {
				var store = transaction.db.objectStore("movies")
				store.createIndex("titleIndex", "title", { unique: true});  // Adds an index on the movies titles
				store.createIndex("formatIndex", "format", { unique: false}); // Adds an index on the movies formats
				store.createIndex("genreIndex", "genre", { unique: false}); // Adds an index on the movies genres
				next();
			}
		}
	]
}
```

## Models

Not much change to your usual models. The only significant change is that you can now fetch a given model with its id, or with a value for one of its index.

For example, in your traditional backbone apps, you would do something like :

```js
var movie = new Movie({id: "123"})
movie.fetch()
```

to fetch from the remote server the Movie with the id `123`. This is convenient when you know the id. With this adapter, you can do something like

```js
var movie = new Movie({title: "Avatar"})
movie.fetch()
```

Obviously, to perform this, you need to have and index on `title`, and a movie with "Avatar" as a title obviously. If the index is not unique, the database will only return the first one.

## Collections

I added a lot of fun things to the collections, that make use of the `options` param used in Backbone to take advantage of IndexedDB's features, namely **indices, cursors and bounds**.

First, you can `limit` and `offset` the number of items that are being fetched by a collection.

```js
var theater = new Theater() // Theater is a collection of movies
theater.fetch({
	offset: 1,
	limit: 3,
	success: function() {
		// The theater collection will be populated with at most 3 items, skipping the first one
	}
});
```

You can also *provide a range* applied to the id.

```js
var theater = new Theater() // Theater is a collection of movies
theater.fetch({
	range: ["a", "b"],
	success: function() {
		// The theater collection will be populated with all the items with an id comprised between "a" and "b" ("alphonse" is between "a" and "b")
	}
});
```

You can also get *all items with a given value for a specific value of an index*. We use the `conditions` keyword.

```js
var theater = new Theater() // Theater is a collection of movies
theater.fetch({
	conditions: {genre: "adventure"},
	success: function() {
		// The theater collection will be populated with all the movies whose genre is "adventure"
	}
});
```



You can also *get all items for which an indexed value is comprised between 2 values*. The collection will be sorted based on the order of these 2 keys.

```js
var theater = new Theater() // Theater is a collection of movies
theater.fetch({
	conditions: {genre: ["a", "e"]},
	success: function() {
		// The theater collection will be populated with all the movies whose genre is "adventure", "comic", "drama", but not "thriller".
	}
});
```

You can also selects indexed value with some "Comparison Query Operators" (like mongodb)
The options are:
* $gte = greater than or equal to (i.e. >=)
* $gt = greater than (i.e. >)
* $lte = less than or equal to (i.e. <=)
* $lt = less than (i.e. <)

See an example.

```js
var theater = new Theater() // Theater is a collection of movies
theater.fetch({
	conditions: {year: {$gte: 2013},
	success: function() {
		// The theater collection will be populated with all the movies with year >= 2013
	}
});
```

You can also *get all items after a certain object (excluding that object), or from a certain object (including) to a certain object (including)* (using their ids). This combined with the addIndividually option allows you to lazy load a full collection, by always loading the next element.

```js
    var theater = new Theater() // Theater is a collection of movies
    theater.fetch({
    	from: new Movie({id: 12345, ...}),
    	after: new Movie({id: 12345, ...}),
    	to: new Movie({id: 12345, ...}),
    	success: function() {
    		// The theater collection will be populated with all the movies whose genre is "adventure", "comic", "drama", but not "thriller".
    	}
    });
```

You can also obviously combine all these.

## Optional Persistence
If needing to persist via ajax as well as indexed-db, just override your model's sync to use ajax instead.

```coffeescript
class MyMode extends Backbone.Model

  sync: Backbone.ajaxSync
```

Any more complex dual persistence can be provided in method overrides, which could eventually drive out the design for a multi-layer persistence adapter.
