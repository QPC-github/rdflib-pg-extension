# RDFLib.js Pointed Graph extension

This is a repository with a PointedGraph extension of RDFLib.js.
RDFLib.js is not always easy to deal with, not really documented... 
This library will try to make your life easier with RDFLib.

It is in active development and will provide soon different ways to edit RDF graphs.

## What is a Pointed Graph?

- A pointed graph is a pointer in a named graph.
- A named graph is an http resource/document which contains an RDF graph (a list of RDF triples)
- A pointer is a particular node in this graph.


This API permits to easily navigate through RDFLib graphs in an RDFLib store.
Inspired by what is done in [banana-rdf](https://github.com/w3c/banana-rdf) but for the browser.

#### Additional vocabulary:

A PG is an abbreviation for a pointed graph.

A document / named graph / http resource means the same thing here.

A rel / relation / RDF predicate is the "middle" part of a triple: `subject / predicate / object`.

#### Navigate in the RDF graphs

From a given pointed graph, you can navigate to two different kind of pointed graphs:

- **localPg**: this is a pointed graph in the same http document: no additional http request is needed to obtain it.
- **remotePg**: this is a pointed graph that belong to another http document. This means that it is probable that additional http requests have to be made to access this pointed graph.

#### Rel operation

We describe the **rel** operation as being the operation that permits to stay in the same document but move the pointer to a different node (Yes, it can also be a blank/literal node!)
For exemple if you points to a `foaf:Person` you can follow the rel `FOAF("name")`. It will give you a list of **localPg**. You may wonder why it is a list? Because one person can have multiple triples with the same relation. This is why we have a **relFirst** operator which will be useful for most cases line a person name, age...

#### Jump operation

We describe the **jump** operation as being the operation that permits to keep the same pointer but changes the underlying document / named graph / http resource. It is the **jump** operation that triggers the fetching of remote documents. If the current PG pointer is an URL of another document, this will fetch this document and so give you a **remotePg**. If the current pointer is a blank node, literal or local named node, this will do nothing and return the current PG because all the data is already defined locally.


TODO: define an unique ubiquitous language because the RDF world is not easy...

## Dependencies:
- [Q](https://github.com/kriskowal/q)
- [Underscore](https://github.com/jashkenas/underscore)
- [JQuery](https://github.com/jquery/jquery) 
- [RxJs](https://github.com/Reactive-Extensions/RxJS) (rx.js + rx.binding.js are required)

## Exemple

We will show how from a `foaf:Person` url, we can print the friend names of this person.

#### Fetching the person

```javascript
var FOAF = $rdf.Namespace("http://xmlns.com/foaf/0.1/");

// We don't use the RDFLib original proxy but this one instead
$rdf.Fetcher.fetcherWithPromiseCrossSiteProxyTemplate = "https://www.stample.io/srv/cors?url=";

var fetcherTimeout = 3000;
var store = $rdf.PG.createNewStore(fetcherTimeout);

// You can use an URL with or without a fragment, it will give you different
// pointer graphs with the same underlying document/namedGraph.
var henryFoafProfileUrl = "http://bblfish.net/people/henry/card#me";

// You can fetch it, it returns a Q promise of pointed graph.
// The pointer will be Henry's foaf:Person subject
store.fetcher.fetch(henryFoafProfileUrl)
        .then(function(henryPg) {
            printPersonFriendNames(henryPg);
        })
        .fail(function(fetchError) {
            console.warn("Can't fetch the pointed graph at",henryFoafProfileUrl,fetchError);
        });
```


#### With JumpAsync

```javascript
function printPersonFriendNames(personPg) {
    // get the list of pointed graphs that points to henry's friend in the local document
    var localFriendPgs = personPg.rel( FOAF("knows") );
    _.each(localFriendPgs, function(localFriendPg) {
        printFriendName(localFriendPg);
    });
}

function printFriendName(localFriendPg) {
    // Generally, each foaf profile is described in separate http resources / rdf graphs.
    // So you can't get much data by staying in the local graph,
    // you have to jump to the remote graph that is hosted at your friend's foaf profile URL.
    // For that you can use jumpAsync: this will fetch the remote URL
    // and return a promise of pointed graph. The PG pointer will be the same node
    // (the friend url symbol) but the underlying document will be changed
    localFriendPg.jumpAsync()
            .then(function(remoteFriendPg) {
                // Finally we can print the name of all Henry's friends
                var friendName = remoteFriendPg.relFirst(FOAF("name")).pointer.toString();;
                $("#friendList").append("<li>"+friendName+"</li>")
            }).fail(function (jumpError) {
                console.warn("Error during jump from ",localFriendPg.printSummary(),jumpError);
            });
}
```

#### Using RxJs Observable streams

If you are using RxJs it's even simpler to get a stream of friends: you'll receive the friends pointed graphs as the requests come back.

We will try to print the name of the persons we find. This code will permit to subscribe to the name stream and print it to our page:

```javascript
    function handleNameStream(nameStream) {
        nameStream.subscribe(
                function(namePg) {
                    var friendFriendName = namePg.pointer.toString();
                    $("#friendList").append("<li>"+friendFriendName+" ("+namePg.getCurrentDocumentUrl()+")</li>")
                },
                function(error) {
                    console.error("Unexpected end of stream",error, error.stack);
                },
                function() {
                    $("#end").html(" -> End");
                }
        );
    }
```


Now see how easy it is to create interesting pointed graph streams.



```javascript
    function printFriendNames(personPg) {
        var personFriendStream = personPg.jumpRelObservable( FOAF("knows") );
        var personFriendNameStream = personFriendStream.flatMap(function(friendPg) {
            return Rx.Observable.fromArray( friendPg.rel(FOAF("name")) ).take(1);
        });
        handleNameStream(personFriendNameStream);
    }
    
    printFriendNames(henryPg);
```

We have also made some recursive code to make it easier to use.
You can follow a path of RDF predicates:

```javascript
    function printFriendFriendName(personPg) {
        var personFriendFriendNameStream = personPg.followPath( [FOAF("knows"), FOAF("knows"), FOAF("knows"), FOAF("name") ]);
        handleNameStream(personFriendFriendNameStream);
    }
```

Cool isn't it?


Now let's try an advanced exemple: we want to build a very simple friend recommender engine.
If a friend of you has another friend and this friend knows you, it is very probable that you also know him. So if he's not your friend yet, you might want to add him as a friend to your foaf profile.
Here is the code to get a stream on that potential friend list:

```javascript
    function printPotentialFriendNames(personPg) {

        var personDoesntKnowHimFilter = function(himPg) {
            var personKnowHim = personPg.hasPointerTripleMatching( FOAF("knows") , himPg.pointer );
            return !personKnowHim;
        };

        var heKnowsPersonFilter = function(hePg) {
            return hePg.hasPointerTripleMatching( FOAF("knows") , personPg.pointer);
        };

        var potentialFriendStream = personPg.followPath( [FOAF("knows"), FOAF("knows")])
                .filter($rdf.PG.Filters.isSymbolPointer)
                .distinct(function (pg) {
                    return $rdf.PG.Transformers.symbolPointerToValue(pg);
                })
                .filter(personDoesntKnowHimFilter)
                .filter(heKnowsPersonFilter);

        var potentialFriendNameStream = potentialFriendStream.flatMap(function(potentialFriendPg) {
            return potentialFriendPg.followPath( [FOAF("name")] ).take(1);
        });

        handleNameStream(potentialFriendNameStream);
    }
```

As you can see, we just have to filter the initial stream obtained from the path [FOAF("knows"), FOAF("knows")].
We do not want to have duplicate suggestions and we only want to display one name per user (ignoring people having no name). 

Running this exemple with `henryPg` we can find out that he may want to add 2 friends.


**Getting started with RxJs**

To use RxJs, you'll have to understand some functional operators on streams like the flatMap operator to be able to use it efficiently.
You will find great help in the Coursera course: [Functional Reactive Programming in Scala](https://www.coursera.org/course/reactive).


#### Getting Metadatas

rdflib doesn't make it really easy to understand how to get simple metadatas after a document has been fetched.
Actually fetch metadatas are stored as RDF triples into the store, under a request and response BNode that are not obvious to retrieve.

So we added some helper code to make it easier.

```javascript
    store.fetcher.fetch(henryFoafProfileUrl)
            .then(function(henryPg) {
                var metadataHelper = henryPg.currentDocumentMetadataHelper();
                console.debug(henryFoafProfileUrl);
                console.debug(metadataHelper.getResponseStatus());
                console.debug(metadataHelper.getResponseStatusText());
                console.debug(metadataHelper.getResponseHeaderValue("Content-Type"));
            })
            .fail(function(fetchError) {
                console.warn("Can't fetch the pointed graph at",henryFoafProfileUrl,fetchError);
            });
```


#### Other exemples?

You can find how we use our own library in this [react-foaf](https://github.com/stample/react-foaf) project.

Look at the code documentation, there's not much more methods but the it's just a preview of what you'll be able to do. Coming soon: how to edit graphs for the RWW world.


## Try it

These working exemples can be found [here](https://github.com/stample/rdflib.js/tree/master/pointedgraph/exemples).
Just clone the repo and open the html files in your browser.

There's also an exemple of [using requireJS](https://github.com/stample/rdflib.js/blob/master/pointedgraph/exemples/getFriendsRequire.html
).

You can also play with this prebuilt [JsFiddle](http://jsfiddle.net/D5Gqs/4/)


## CORS Proxy

Here are some CORS proxy that you can use:

```javascript
$rdf.Fetcher.fetcherWithPromiseCrossSiteProxyTemplate = "https://www.stample.io/srv/cors?url=";
$rdf.Fetcher.fetcherWithPromiseCrossSiteProxyTemplate = "http://data.fm/proxy?uri=";
```
There may not always be online. 
Stample.io proxy (rww-play/Netty/Scala based) is a lot faster than data.fm proxy (PHP).


You can also run your own local proxy thanks to [rww-play](/stample/rww-play)

```javascript
$rdf.Fetcher.fetcherWithPromiseCrossSiteProxyTemplate = "http://localhost:9000/srv/cors?url=";
```

##### What about original RDFLib proxy?

We usually can register a proxy for RDFLib:

```javascript
$rdf.Fetcher.CrossSiteProxyTemplate = "http://data.fm/proxy?uri=";
```

We do not recommend using it. 
Actually the integration of our lib with RDFLib is not perfect and it doesn't work with the regular RDFLib proxy.

## Changelog

### 0.1.1

- Preregister some commonly used namespaces in `$rdf.PG.Namespaces`
- Add possibility to easily get PG's underlying document metadatas like http status code, response headers...

### 0.1.0

- The initial version


## Download

You can find versioned releases in the [releases](https://github.com/stample/rdflib.js/tree/master/releases) folder.

We recommend using our rdflib-stample.js version because it may include pull requests
that have not yet been merged to the official rdflib.js master.


## Install
 
 Run `make` to generate the dist directory
 This is like the original RDFLib Makefile but it generates an additional rdflib-pg-extension.js
 
 
## Help

`#stample` on Freenode (IRC)

Stample google group: https://groups.google.com/forum/#!forum/stample

## Licence: 
 
 MIT
