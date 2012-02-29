---
title: ql.io Baseline Benchmarks
layout: post
author: subbu
---

One of the key visible benefits of [ql.io](http://ql.io) is that it eliminates the code noise that
is common in writing HTTP client apps. As a DSL for writing HTTP client code, it focuses on
automating the task of making multiple HTTP requests and processing responses in the best order
possible taking care of paralleization, orchestration, projections and normalizations behind the
scenes.

In this post, I would like to present the baseline performance benchmarks of ql.io running on
node.js 0.4.12. Though I have done some ad hoc tests in the last 2-3 months for hardware sizing
purposes, this is my first systematic attempt.

<!-- more -->

Since this post is long, here is a quick summary for the "TL;DR".

* For simple scripts - such a query using a `select` statement to get data from an HTTP API, ql.io
  can handle 2400+ requests/sec at various concurrency levels ranging from 100 to 500.
* For scripts involving dependencies between statements (such as statement B needing input from
  results of statement A), thoughput drops almost linearly and proportionately. For instance, for
  scenario B below, ql.io can handle nearly 1000 requests/sec.
* The conventional wisdom of using `n` worker processes where `n` is the number of CPU threads
  provides a reasonable default for all practical purposes but tuning the number of worker processes
  is a good exericise to do. All the test scenarios below yielded better numebers with `5*n`
  workers. Scenario D, which involves a non-trivial amount of CPU bound work, benefitted the most
  from the increased number of worker processes.

You can find the raw output files of test runs on [github](https://github.com/ql-io/ql.io-perf).
The application used for these tests is on [github](https://github.com/ql-io/ql.io-site). All the
ql.io modules used by the app are on npmjs.org.

## Test Environment

The test environment is based on the folllowing, each running Ubuntu, sititng under my desk at
work.

* An Intel Xeon E5645 workstation with 6 cores (12 CPU threads) and 24GB RAM running the
  [ql.io-site](https://github.com/ql-io/ql.io-site) app on node.js 0.4.12 with 12 worker processes.
* An Intel Xeon E5507 workstation with 4 cores (8 CPU threads) with 12GB RAM running apachebench.
* An Intel Xeon E5630 workstation with 4 cores (8 CPU threads) with 24GB RAM running Apache
  Traffic Server (ATS) 3.0.1 as a forward proxy for all outgoing HTTP requests. The cache is primed
  before running benchmarks to avoid making requests to any other machines.

All these are running Ubuntu 11.04.

## Test Scenarios

These tests cover a range of aggregation and orchestration scenarios possible with ql.io and show
how ql.io behaves under varying loads.

### Scenario A

<pre class="brush: sql toolbar: false;">
select * from twitter.search where q = "ql.io"
</pre>

This scenario involves sending a HTTP GET request to `http://search.twitter.com/search.json`,
parsing the JSON response, and writing it back to the client's response.

### Scenario B

<pre class="brush: sql toolbar: false;">
select id as id, from_user_name as user_name, text as text from twitter.search where q = "ql.io";
</pre>

This scenario is similar to scenario A except the following:

* Extract `results` array from the response and extracts `id`, `from_user_name`, and `text` for
  each result.
* Assemble the projected fields into an object.
* Write all the objects as an array into the client's response.

### Scenario C

<pre class="brush: sql toolbar: false;">
select ItemID, ViewItemURLForNaturalSearch, Location from details where itemId in
    (select itemId from finditems where keywords='mini cooper');
</pre>

This scenario involves finding IDs of items from one API and sending those IDs to another API to
get details as follows:

* Send an HTTP request to `http://svcs.ebay.com/services/search/FindingService/`, parse the JSON
  response, and extract the array of items by selecting the
  `findItemsByKeywordsResponse.searchResult.item` field of the response.
* For each item in the array, project the item's ID. Collect the IDs into an array.
* Then send an HTTP request to `http://open.api.ebay.com/shopping` with all the item IDs.
* Parse the JSON response, select the `Item` array from the response, and project each `Item` to
  extract `ItemID`, `ViewItemURLForNaturalSearch`, and `Location` fields. Assemble the projected
  fields into an array.
* Write all the arrays to the client's response as an array or arrays.

### Scenario D

<pre class="brush: sql toolbar: false;">
prodid = select ProductID[0].Value from eBay.FindProducts where
    QueryKeywords = 'macbook pro';
details = select * from eBay.ProductDetails where
    ProductID in ('{prodid}') and ProductType = 'Reference';
reviews = select * from eBay.ProductReviews where
    ProductID in ('{prodid}') and ProductType = 'Reference';

return select d.ProductID[0].Value as id, d.Title as title,
    d.ReviewCount as reviewCount, r.ReviewDetails.AverageRating as rating
    from details as d, reviews as r
    where d.ProductID[0].Value = r.ProductID.Value
    via route '/myapi' using method get;
</pre>

The implementation details for this script are a bit more involved, but at a high level, here is
what happens under the hood:

* Find the script when the client submits a request to the script through a route `/myapi`.
* Send a HTTP request to `http://open.api.ebay.com/shopping?callname=FindProducts` with a keyword
  and extract product IDs from the response.
* Send `5` HTTP requests to `http://open.api.ebay.com/shopping?callname=FindProducts` with the
  product IDs found and extract the details.
* Send `5` HTTP requests to `http://open.api.ebay.com/shopping?callname=FindReviewsAndGuides` with
  the product IDs found and extract the reviews.
* Once the `10` requests complete, join details and reviews by matching responses by IDs, and
  extract the selected fields into an object.
* Return an array of objects with each object containing the selected fields.

This script covers most of the code paths of ql.io. See [Build an
App](http://ql.io/docs/build-an-app) for a step by step description of this scenario

### Differences Between Scenarios

* Both scenario A and B are mostly IO bound.
* Scenario C is also mostly IO bound, but it makes two HTTP requests in sequence as the outer
  `select` depends on the results of the `inner` select. The second request is made after the first
  one completes.
* Scenario D involves making `11` HTTP requests, parsing and projecting response fields, and joining
  members of responses of the second and third statements. These responses are unsorted, and joining
  them by a matching product ID takes O(n^2) steps - in this case 25. Yes - this can be improved -
  but [let's measure twice before cutting
  once](http://calendar.perfplanet.com/2011/measure-twice-cut-once/).

## Test Settings

All tests are done using `ab -k` to maintain persistent connections from the client to the server.

The ql.io app is run with `12` node.js worker processes managed by
[cluster](http://learnboost.github.com/cluster/).

## First Round Results

### Throughput

Here are the throughput results for concurrency ranging from 100 to 500.

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=A3%3AE8&gid=0&pub=1","options":{"reverseCategories":false,"curveType":"","titleX":"Concurrency","backgroundColor":"#FFFFFF","pointSize":0,"width":600,"lineWidth":2,"logScale":false,"hAxis":{"maxAlternations":1},"hasLabelsColumn":true,"vAxes":[{"title":"Requests/sec","minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null},{"viewWindowMode":"pretty","viewWindow":{}}],"title":"Throughput (12 workers)","height":371,"interpolateNulls":false,"legend":"right","reverseAxis":false},"state":{},"view":"{\"columns\":[0,1,2,3,4]}","chartType":"LineChart","chartName":"Throughput"} </script>

### Mean Response Times

The corresponding chart showing the mean response time for the same range of concurrency is below.

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=A27%3AE32&gid=0&pub=1","options":{"reverseCategories":false,"curveType":"","titleX":"Concurrency","pointSize":0,"backgroundColor":"#FFFFFF","lineWidth":2,"logScale":false,"hAxis":{"maxAlternations":1},"hasLabelsColumn":true,"vAxes":[{"title":"Time for 80% requests to complete","minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null},{"viewWindowMode":"pretty","viewWindow":{}}],"title":"Mean response time","interpolateNulls":false,"legend":"right","reverseAxis":false,"width":600,"height":371},"state":{},"view":"{\"columns\":[0,1,2,3,4]}","chartType":"LineChart","chartName":"Chart 2"} </script>

## Effect of Number of Workers

In these tests, scenario D fared badly as it includes a mixture of IO and CPU workloads. The CPU
workload is not predominant but is not insignificant either. Here is a chart of the CPU data
captured using `dstat` at a concurrency level of 200 for sceanrio D.

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=A7%3AG443&gid=5&pub=1","options":{"vAxes":[{"viewWindowMode":"pretty","viewWindow":{}},{"viewWindowMode":"pretty","viewWindow":{}}],"displayAnnotations":true,"height":371,"width":709,"displayRangeSelector":true,"displayZoomButtons":true,"hAxis":{"maxAlternations":1},"hasLabelsColumn":true,"wmode":"opaque"},"state":{},"view":"{\"columns\":[0,1,2,3,4,5,6]}","chartType":"AnnotatedTimeLine","chartName":"CPU Load for Scenario A with 12 workers"} </script>

This confirms that there is a fair bit of CPU bounded work going on. How does the number of
workers influence such a scenario? I repeated the tests varying the number of worker processes.

The chart below shows the number of requests per second for Scenario D as I changed the number of
workers from 12 to 96 in increments of 12. All the test runs were done at a concurrency level of
100.

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=A4%3AB12&gid=2&pub=1","options":{"reverseCategories":false,"curveType":"","titleX":"Number of workers","pointSize":0,"backgroundColor":"#FFFFFF","lineWidth":2,"logScale":false,"hasLabelsColumn":true,"hAxis":{"maxAlternations":1},"vAxes":[{"title":"Req/sec","minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null},{"viewWindowMode":"pretty","viewWindow":{}}],"title":"","interpolateNulls":false,"legend":"right","reverseAxis":false,"width":600,"height":371},"state":{},"view":"{\"columns\":[0,1]}","chartType":"LineChart","chartName":"Chart 3"} </script>

The number of requests per sec increase from 192 to 384 as I increased the number of workers from
12 to 96. The improvement is less significant after 60 workers.

Here is chart for the mean response time which shows a similar improvement.

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=D4%3AE12&gid=2&pub=1","options":{"series":{"0":{"color":"#6aa84f"}},"reverseCategories":false,"curveType":"","titleX":"Number of workers","pointSize":0,"backgroundColor":"#FFFFFF","lineWidth":2,"logScale":false,"hAxis":{"maxAlternations":1},"hasLabelsColumn":true,"vAxes":[{"title":"Mean response time (msec)","minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null},{"viewWindowMode":"pretty","viewWindow":{}}],"title":"","interpolateNulls":false,"legend":"none","reverseAxis":false,"width":600,"height":371},"state":{},"view":"{\"columns\":[0,1]}","chartType":"LineChart","chartName":"Chart 4"} </script>

The flatness of these charts with increased worker count can easily be explained by looking at the
CPU again. The chart below shows the CPU data at a cocurrency level of 200 for scenario D with a
worker count of 96.

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=A7%3AG288&gid=6&pub=1","options":{"displayAnnotations":true,"vAxes":[{"viewWindowMode":"pretty","viewWindow":{}},{"viewWindowMode":"pretty","viewWindow":{}}],"wmode":"opaque","hasLabelsColumn":true,"hAxis":{"maxAlternations":1},"width":742,"height":371},"state":{},"view":"{\"columns\":[0,1,2,3,4,5,6]}","chartType":"AnnotatedTimeLine","chartName":"Chart 9"} </script>

The chart below shows the effect of increasing the worker count from 12 to 96 across all test
scenarios.

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=A2%3AI7&gid=1&pub=1","options":{"vAxes":[{"title":"Requests/sec","minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null},{"viewWindowMode":"pretty","viewWindow":{}}],"reverseCategories":false,"title":"","titleX":"Concurrency","backgroundColor":"#FFFFFF","legend":"right","logScale":false,"reverseAxis":false,"hasLabelsColumn":true,"hAxis":{"maxAlternations":1},"isStacked":false,"width":1074,"height":340},"state":{},"view":"{\"columns\":[0,1,2,3,4,5,6,7,8]}","chartType":"ColumnChart","chartName":"Chart 5"} </script>

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=A34%3AI39&gid=1&pub=1","options":{"vAxes":[{"title":"Mean response time (msec)","minValue":null,"viewWindowMode":"pretty","viewWindow":{"min":null,"max":null},"maxValue":null},{"viewWindowMode":"pretty","viewWindow":{}}],"reverseCategories":false,"title":"","titleX":"Concurrency","backgroundColor":"#FFFFFF","legend":"right","logScale":false,"reverseAxis":false,"hAxis":{"maxAlternations":1},"hasLabelsColumn":false,"isStacked":false,"width":972,"height":402},"state":{},"view":"{\"columns\":[0,1,2,3,4,5,6,7,8]}","chartType":"ColumnChart","chartName":"Chart 6"} </script>

## What About Memory

Below is a chart of the memory usage with 96 workers for scenario D at a concurrency level of 200.

<script type="text/javascript" src="http://ajax.googleapis.com/ajax/static/modules/gviz/1.0/chart.js"> {"dataSourceUrl":"//docs.google.com/a/subbu.org/spreadsheet/tq?key=0ApntcBgpHeZldFREOFlGeXVWdzZZc3lNSld1aWZqdVE&transpose=0&headers=1&range=A299%3AE581&gid=6&pub=1","options":{"displayAnnotations":true,"vAxes":[{"viewWindowMode":"pretty","viewWindow":{}},{"viewWindowMode":"pretty","viewWindow":{}}],"wmode":"opaque","hasLabelsColumn":true,"hAxis":{"maxAlternations":1},"width":708,"height":371},"state":{},"view":"{\"columns\":[0,1,2,3,4]}","chartType":"AnnotatedTimeLine","chartName":"Chart 12"} </script>

The lines remained nearly flat for the duration of the test.

## Summary

The goal of this exercise is to set a baseline for future work. The scenarios I used show a range of
scripts that cover most of the current capabilities of ql.io.

Here are few key take-aways:

* ql.io is designed for IO bound workloads. However, data aggregation and orchestration often
  involves some CPU bound work such as projections and joins. This is unavoidable. I suspect that
  the same is the case with many typical uses of node.js.
* On commodity hardware with commodity network layer, my tests show that ql.io can do 400-2400
  requests/sec depending on the nature of the work involved. Your mileage may vary.
* Use of as many workers as there are CPU threads available is a good starting point, but tuning
  the number based on the characteristics of the app may yield better results.

We're currently working on upgrading ql.io to node.js 0.6.x. See the [0.4 branch on
github](https://github.com/ql-io/ql.io/tree/0.4). Watchout for a repeat of these tests on node.js
0.6.x.

