---
title: Making Peace with HTTP APIs
layout: post
author: subbu
---

Once in a while you come across an HTTP API that uses HTTP in complicated and incorrect ways. There
are many examples of this on the Web today including those from
[eBay](http://developer.ebay.com/devzone/xml/docs/reference/ebay/GetMyeBayBuying.html),
[Amazon](http://docs.amazonwebservices.com/amazonswf/latest/developerguide/UsingJSON-swf.html),
[Google](http://code.google.com/apis/friendconnect/docs/opensocial_rest_rpc.html),
[Microsoft](http://www.bing.com/toolbox/bingdeveloper/) and many many others. These can be hard to
use as they require you to follow proprietary styles for constructing requests and parsing
responses. Some of those also don't work well with common HTTP infrastructure like caches.

In this post, I would like to show how you can, in four simple steps, use ql.io to hide the
complexity of such APIs.

To illustrate, let me take eBay's [PlaceOffer
API](http://developer.ebay.com/DevZone/XML/docs/Reference/eBay/PlaceOffer.html) that lets an eBay
buyer place an offer for an item listed on eBay. This example may be more complex than other similar
APIs that you have encoutered, but it helps me drive the point.

This API requires you to send a POST request with some custom headers and an XML document in the
body.

    POST /ws/api.dll HTTP/1.1
    Host: api.ebay.com/ws/api.dll
    Content-Type: application/xml; charset=UTF-8
    X-EBAY-API-DEV-NAME: developer ID
    X-EBAY-API-APP-NAME: app ID
    X-EBAY-API-CERT-NAME: cert ID,
    X-EBAY-API-CALL-NAME: PlaceOffer
    X-EBAY-API-COMPATIBILITY-LEVEL: version
    X-EBAY-API-SITEID: site ID


See [developer docs](http://tinyurl.com/76q6e7q) for more details of these headers.

The body of the request is an XML document. An example is below.

    <PlaceOfferRequest xmlns="urn:ebay:apis:eBLBaseComponents">
      <ErrorLanguage>en_US</ErrorLanguage>
      <EndUserIP>192.168.255.255</EndUserIP>
      <ItemID>110096039601</ItemID>
      <Offer>
        <Action>Bid</Action>
        <MaxBid currencyID="USD">20.00</MaxBid>
        <Quantity>1</Quantity>
      </Offer>
      <RequesterCredentials>
        <eBayAuthToken>ABC...123</eBayAuthToken>
      </RequesterCredentials>
      <WarningLevel>High</WarningLevel>
    </PlaceOfferRequest>

A response from this API looks like the following:

    <PlaceOfferResponse xmlns="urn:ebay:apis:eBLBaseComponents">
      <Timestamp>2012-02-03T18:06:51.230Z</Timestamp>
      <Ack>Success</Ack>
      <Version>757</Version>
      <Build>E757_CORE_BUNDLED_14364711_R1</Build>
      <UsageData>MTMyOTUyMTQ2LzE1MzczOw**</UsageData>
      <SellingStatus>
        <ConvertedCurrentPrice currencyID="USD">1.0</ConvertedCurrentPrice>
        <CurrentPrice currencyID="USD">1.0</CurrentPrice>
        <HighBidder>
          <UserID>testuser_bountifulbuyer</UserID>
        </HighBidder>
        <MinimumToBid currencyID="USD">1.25</MinimumToBid>
      </SellingStatus>
    </PlaceOfferResponse>

Here is how you can use ql.io to simplify this.

### Step 0: Create an App

Create a ql.io app.

    mkdir myapp
    cd myapp
    curl https://raw.github.com/ql-io/ql.io/master/modules/template/init.sh | bash
    bin/start.sh

See [docs](http://ql.io/docs) for more details.

### Step 1: Create a Table

Place the following in `tables/placeoffer.ql`.

<script src="https://gist.github.com/1886983.js?file=gistfile1.sql"></script>

This step binds the API into the ql.io runtime so that you can use ql.io's DSL to send requests
and process responses.

### Step 2: Describe the Shape of the Request Body

Place the following in `tables/placeoffer.xml.mu`.

<script src="https://gist.github.com/1886988.js?file=gistfile1.xml"></script>

This is just a mustache template. You can use [EJS](http://embeddedjs.com/) too if you like.

### Step 3: Create a Route

Place the following in `routes/placeoffer.ql`

<script src="https://gist.github.com/1886996.js?file=gistfile1.sql"></script>

### Step 4: Use the API

    POST /offers?siteId=0&itemId=your-item-id&offer=your-offer&action=your-action&quantity=your-quantity
    Host: api.ebay.com/ws/api.dll
    Authorization: your auth token

This request returns JSON.

### Step 5: Enjoy

No XML, no schemas, no SDKs. As an added benefit, you can combine this API with other APIs as you
like using ql.io's DSL.

Why does this matter? If you have a legacy API that you can not afford to rewrite to use HTTP
sanely, ql.io can help you hide it behind a saner interface.

Thanks to [Juan Rodriguez](https://github.com/jmrodriguez) for showing me this example.

