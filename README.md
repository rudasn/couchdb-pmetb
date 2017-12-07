# Querying CouchDB

> An exploration on high performance querying with big enough CouchDB databases.

This document describes common problems encountered when working with medium-sized CouchDB databases and how to overcome them. The intended audience is developers with some experience of CouchDB (particularly views) and NodeJS (particularly promises).

Note that some of the techniques described in this document can potentially be applied in situations where CouchDB is not used but we still need to retrieve and process a lot of data on a single machine.

---

<!-- MarkdownTOC autolink="true" bracket="round" -->

- [Introduction](#introduction)
- [Querying CouchDB](#querying-couchdb)
- [Limitations](#limitations)
- [Concurrent Querying](#concurrent-querying)
- [Memory Management](#memory-management)
- [Error handling](#error-handling)
- [Throttling](#throttling)
- [Batch Requests](#batch-requests)

<!-- /MarkdownTOC -->

---

## Introduction

Before we go into the details Lets recap what CouchDB is:

- CouchDB is a clustered NoSQL document database.
- CouchDB can be querying using Map/Reduce functions (called `views`).
- CouchDB views are indexed (updated) against new or changed documents since the most recent query against that view.
- View querying can be extremely fast and view indexing can be very slow (if there are a lot of documents to go through).

Lets assume we have a CouchDB cluster that only contains documents representing events (things that have happened in the past). Our cluster is hosted remotely on a number of nodes (think Cloudant).

Each event has a **type** (eg. `click`) and a **timestamp** (eg. `2018-01-01`) along with some optional meta-data (eg. `{ user: 'rudasn' }`).

Our goal is to retrieve and process these documents so that we get insights on what's happening, how often, etc etc. Even though there may be tens of millions of these documents we don't want to spend days or weeks doing this.

So how do we:

- Make it fast and reliable?
- Manage system resources?
- Handle network interruptions and other exceptions?

> **Note**: Part of what we want to do can be achieved by using the built-in reduce functions of CouchDB. For the sake of this experiment, we'll assume that using the reducer does not cover our needs.

Ready? Grab a cup of coffee and Lets get started.

---

## Querying CouchDB

Data can be retrieved from CouchDB in three ways:

1. Using the **primary index** (_allDocs) which allows us to fetch a number of documents using their `_id`.
2. Using a **mango index** which allows us to retrieve certain documents that match a set of criteria.
3. Using **views** (Map/Reduce functions).

In this scenario `views` better fit our needs and allow us to keep things simple.

So Lets go ahead and create a simple view that emits the `timestamp` for each event.

```js
function(doc) {
    emit(doc.timestamp)
}
```

Once the view is indexed we can query our database for events by date.

```js
// define the end-point of our query for future reference
const view = `https://my-remote-couchdb-instance.com/_design/events/view/events-by-timestamp`
```

Fetch all events:

```js
fetch(view).then(
    // log the total number of rows emitted from our view
    response => console.log(response.rows.length)
)
```

Events during a specific year (2018):

```js
fetch(`${ view }?startkey="2018-"&endkey="2018-\u9999"`)
```

Events during a specific month (January 2018):

```js
fetch(`${ view }?startkey="2018-01-"&endkey="2018-01-\u9999"`)
```

Each of these queries issues a single GET request to CouchDB. Internally, CouchDB goes through the pre-built view index to find rows between the provided `startkey` and `endkey`.

## Limitations

But what happens if our query has potentially millions of returned results?
What if our system cannot load all these results into memory without crashing? 
What if our network connection is interrupted in the middle of the request (or worse, at the end)?

Do we just sit around and hope all goes well?

No. Time is of the essence and hope is not enough.

First, lets reconsider and expand our goal:

> How can we retrieve and process millions of documents at once...
>
> 1. without wasting time waiting for the request to finish before processing results?
> 2. without needed to start afresh if something goes wrong?
> 3. without maxing out our system's and/or our cluster's resources?

The simplest answer is almost always the best one, so here we go:

> 1. Don't waste that time.
> 2. You can't avoid starting afresh, but you can make it less painful.
> 3. Don't retrieve millions of documents if you can't handle it.

There is one common theme that stands out: **break the one huge task into many smaller and more manageable tasks**. This is all this document is about so Lets go ahead and do just that.

## Concurrent Querying

We'll assume we want to fetch all events occurring in January 2018. Our previous query looked like this:

```js
fetch(`${ view }?startkey="2018-"&endkey="2018-\u9999"`)
```

How can this be broken up into more manageable queries? Well, we know that January always has 31 days so we can issue one query for each day:

```js
const queries = [
    '2018-01-01',
    '2018-01-02',
    '2018-01-03',
    '2018-01-04',
    // ... etc
].map(date =>
    fetch(`${ view }?startkey="${ date }"&endkey="${ date }\u9999"`)
)

Promise.all(queries).then(responses =>
    responses.map(response =>
        processRows(response.rows)
    )
).then(() => {
    console.log('All done!')
})
```

Nice! Now we are receiving data for the whole month much faster since we don't have to wait for CouchDB to retrieve the events for the first day, then retrieve the events for the second day, then for the third day, and so on, as it would if we only issued a single request starting at `2018-01-`.

The cluster would spread these 31 requests across the nodes and each node can do it's job individually without affecting the performance of other nodes.

## Memory Management

But we are still waiting for all requests to complete before we begin processing the data, and we still have to restart the whole process if, for example, the query for `2018-01-30` fails. Not so nice!

So how can we fetch and process the events for a single day regardless of the state of the queries for the other days? Lets see..

```js
const queries = [
    '2018-01-01',
    '2018-01-02',
    '2018-01-03',
    '2018-01-04',
    // ... etc
].map(date =>
    fetch(`${ view }?startkey="${ date }"&endkey="${ date }\u9999"`).then(
        response => processRows(response.rows)
    )
)

Promise.all(queries).then(() => {
    console.log('All done!')
})
```

Nice! Now we are handling the response of each query individually. While we are waiting for responses to complete before processing it, we are now able to process already completed responses. Yay for not wasting our time doing nothing!

This also means better memory management as we are not keeping all the data for the duration of the whole process -- we only keep the data required for each step (day) of the process.

But what happens when we change our query to retrieve data for the whole year and not just January? Instead of 31 queries we now have 365 (or 366) queries! And what's most likely to happen if you try to simultaneously execute so many queries? 

Things *will* break.

## Error handling 

Lets introduce some error handling in our code to take care of that. We want to find out which of our requests have failed and try to run them again but without affecting the execution of the rest of the queries.

As we increase the complexity of our process we want to refactor our code to accommodate for that complexity. Lets do that first..

```js
const dates = [
    '2018-01-01',
    '2018-01-02',
    '2018-01-03',
    '2018-01-04',
    // ... etc
]

const runQuery = date => 
    fetch(`${ view }?startkey="${ date }"&endkey="${ date }\u9999"`).then(
        response => processRows(response.rows),
        error => retryQuery(date, error)
    )

const retryQuery = (date, error) => {
    // magic
}

const queries = dates.map(date => runQuery(date))

Promise.all(queries).then(() => {
    console.log('All done!')
})
```

Nice! Now it's easier to see how each query gets built and handled. So Lets see what kind of magic we need in `retryQuery` to make this work.

The most naive approach would be to retry the query as soon as it fails. The most bullet-proof approach would be to check what type of error we received and act accordingly.

For the purposes of this experiment we're going with an in-between approach: just wait for a little while before retrying failed requests.

```js
// everything as before except...

const retryQuery = (date, error) => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(runQuery(date))
        }, 5000)
    })
}
```

At this point is it important to understand how javascript promises work.

You may have noticed that our `runQuery` method returns a promise whose error handler calls `retryQuery` without throwing an exception. `retryQuery` returns a promise that resolves with the result of an additional call to `runQuery` without ever triggering the `reject` callback.

Here's why this works:

The promise returned from `runQuery` will always be successful *because* we defined an error handler and **never throw** an exception. Since our error handler returns the promise from `retryQuery`, `runQuery` will eventually resolve to whatever `retryQuery` resolves. We don't have to handle errors within `retryQuery` since that's done in `runQuery`.

In other words:

```
runQuery(1) -> [error] -> retryQuery(1) -> [wait] -> runQuery(2) | repeat while [error
]
```

The process repeats while `runQuery` rejects. The process ends once `runQuery` resolves.

Nice! Now we are capturing errors and trying our best to never break the promise chain - our process will resolve, eventually.

## Throttling

But what if the cause of the errors is too many concurrent queries? We are only making things worse by trying again and again. We want a way to manage how many active queries we have at any given time.

Lets see how we can do that.

```js
const throttle = (fn, max) => { /* magic */ }

const runQueryNow = date => 
    fetch(`${ view }?startkey="${ date }"&endkey="${ date }\u9999"`).then(
        response => processRows(response.rows),
        error => retryQuery(date, error)
    )

const runQuery = throttle(runQueryNow, 10)

// same as before..

```

Nice! Now we have a maximum of 10 concurrent requests at any given time, regardless if they are retries or not.

We are managing our system resources by **splitting our workload** to the best of our knowledge (by day) and processing data as it comes in. We are **not under-utilizing** our system resources by idling while responses are pending and we are **not overloading** our system by asking it to do too many things at once.

But what happens if there are millions of events occurring in a single day? We would still need to load all that data in memory, our system would probably crash, and we're back to square one.

## Batch Requests

So how can we create even smaller chunks of work and keep our cool new features?

We can take our approach of chunking by date to the next logical level: chunk by hour. Our `dates` array would look like this:

```js
const dates = [
    '2018-01-01T00',
    '2018-01-01T01',
    '2018-01-01T02',
    '2018-01-01T03',
    // ... etc
]
```

We would end up with approximately 8,760 requests to go through and we would still not solve the problem: during certain hours (eg. morning) it is highly likely we'll have a lot more events than others (eg. midnight). We'll end up processing events during off-peak hours but fail to process events during peak hours.

Not good.

A better approach would allow us to be **explicit** on how much work our systems can handle and control the flow of data more efficiently. 

The simplest way to do that is to specify how much data we want to receive per individual request beforehand. Once a (partial) request is complete we would ask for the next batch, starting from the point we left off in the previous request. We would do this until there's no more data to fetch.

Luckily we can easily do that by using the `limit` and `startkey_docid` query parameters to CouchDB. `startkey_docid` specifies the starting point of the request (the document id of the first row) and `limit` specifies how many rows we want to retrieve after that.

Lets see that in practice:

```js
// A helper method to build the right query path depending on options passed.
const getViewPath = ({ date, limit, start }) => {
    let params = `${ view }?startkey="${ date }"&endkey="${ date }\u9999`
    if (limit) {
        params += `&limit=${ limit }`
    }
    if (start) {
        params += `&startkey_docid=${ start }&skip=1`
    }
    return params
}

// `runQueryNow` is replaced by `runQueryBatch`
const runQueryBatch = ({ date, limit, start }) =>
    fetch(`${ getViewPath({ date, limit, start }) }`).then(
        response => {
            const { rows } = response

            if (!rows.length) {
                return []
            } else {
                const processed = processRows(rows)
                const lastRowId = rows[ rows.length - 1 ].id

                return runQuery({
                    date,
                    limit,
                    start: lastRowId,
                }).then(nextResponse =>
                    processed.concat(processRows(nextResponse.rows))
                )
            }
        },
        error => retryQuery({ date, limit, start }, error)
    )

const runQuery = throttle(runQueryBatch, 10)

// ...as before
```

What happened here?! Where's `runQueryNow`? Why is `runQueryBatch` calling `runQuery`? Why are results of `processRows` being concatenated? Most importantly, **why is this working**?

Lets take a step back and think about how promises work.

When a promise is created a promise chain is also created. You can think of a promise chain as a series of nested promises. Nested promises are created when any of the handlers (`promise.then(successHandler, errorHandler)`) return a promise. For the original (parent) promise to resolve all nested promises need to be resolved first. 

In our old `runQueryNow` method we returned a promise (`fetch`). That promise had two handlers, one to `processRows` if the fetch was successful and one to `retryQuery` if the fetch was unsuccessful.

`retryQuery` returned a promise that resolved after an additional call to `runQueryNow` resolved successfully. 

Our promise chain looked like this:

`runQueryNow(1)` -> `fetch(1)` -> [error] -> `retryQuery(1)` -> `runQueryNow(2)`

The promise returned from `runQueryNow(1)` would resolve when `fetch(1)` resolved. In case of a failure, `fetch(1)` would resolve when `retryQuery(1)` would resolve. `retryQuery(1)` would resolve when 5 seconds have passed *and* `runQueryNow(2)` also resolved. The result of `runQueryNow(1)` would be whatever any of those promises resolved.

Likewise, `runQueryBatch` takes advantage of this feature to create a chain of promises for when the `fetch` is successful by recursively calling `runQuery`. 

The main difference however is that we want the parent promise to resolve the results of **all** the nested promises, not just the last one. That's why we use `concat`.


... TBC


