--- 
title: "ql.io XML/Protobuf Performance"
layout: post
author: idralyuk
comments: true
categories: ql.io protobuf protocol buffers performance nodejs javascript
---

Should backend servers send XML or Protobuf responses to ql.io? That is the question this post addresses.  

I setup a ql.io application hitting a mock server and ran JMeter to generate load and collect results.

Source code for the app is on [Github](https://github.com/idralyuk/ql.io-protobuf-test).
Mock server is [here](https://github.com/idralyuk/ql.io-protobuf-test/tree/master/mock_server) and JMeter script is [here](https://github.com/idralyuk/ql.io-protobuf-test/tree/master/jmeter).

ql.io tables for eBay's Marketplaces APIs are [here](https://github.com/ql-io/ql.io-ebay-mp-apis).  

The table used for this test is [findItemsByKeywords.ql](https://github.com/ql-io/ql.io-ebay-mp-apis/blob/master/tables/finding/findItemsByKeywords.ql).

### Test Setup
**Server**: Dev workstation ca. 2011 (Dell Precision T5500 with Intel Xeon E5630 2.53GHz 24Gb RAM), Linux 3.0.0-20-generic #34-Ubuntu SMP Tue May 1 17:24:39 UTC 2012 x86_64 x86_64 x86_64 GNU/Linux  

**Client**: Dev workstation ca. 2007 (Dell Precision 690 Intel Xeon DP 5060 3.2GHz, 8GB RAM), SunOS 5.11 joyent_20120517T192048Z i86pc i386 i86pc  

The server ran node v0.6.18a and ql.io app 0.7.4. On the same box another node process was serving canned xml and protobuf responses from eBay's [FindingService](http://developer.ebay.com/devzone/finding/callref/findItemsByKeywords.html) on port 6000 (it was on the same box in order to take the network out of the equation).  

The test app was running on port 3000, with two route/table/patch combinations (one for XML, one for Protobuf) pointing to the above response server on localhost:6000.

The client was JMeter 2.7, running in server mode (jmeter-server), with HEAP="-Xms1024m -Xmx1024m".

See Appendix for implementation details.

###Test 1: 200 users, running for 1 hour, hitting the XML path
**Rate: 203.43 trans/sec. Average response time: 72 ms.**

	Number of Samples:           734,339
	Average Response Time:            72 ms
	Minimim Response Time:            25 ms
	Maximum Response Time:         1,002 ms
	Standard Deviation:               49 ms
	Error Percentage:               0.00 %
	Transaction rate:             203.43 trans/sec
	Throughput:                   12,248 KB/sec


###Test 2: 200 users, running for 1 hour, hitting the Protobuf path
**Rate: 213.91 trans/sec. Average response time: 24 ms.**
	
	Number of Samples:           772,200
	Average Response Time:            24 ms
	Minimim Response Time:             8 ms
	Maximum Response Time:           198 ms
	Standard Deviation:               26 ms
	Error Percentage:               0.00 %
	Transaction rate:             213.91 trans/sec
	Throughput:                   12,388 KB/sec


###Discussion

The results show that Protobuf is **_three_** times faster than XML!  

Is it surprising? Yes. Even with the optimize_for = SPEED option turned on in the .proto file, this is a bit extreme.

The exercise of comparing Protobuf to JSON is left to the reader. It is reasonable to assume that they should be on par, as any conversion step is going to be slower than the native format.

This test doesn't take into account network latency. Payload sizes play an important role when network is involved; for the resultset being used in the test (50 items) the sizes were as follows:

	mock.xml - 83 KB
	mock.protobuf - 27 KB

### Conclusion
According to the results of the test, it certainly makes sense to use Protobuf instead of XML.  
 
Further testing (involving the network), needs to be done in order to determine whether protobuf is a better solution than json, but given that there is a significant reduction in payload size, Protobuf is likely to come out a winner in that test as well.

##Appendix
### URLs hit by JMeter (client)
	http://10.xx.xx.xx:3000/ebay/finding/keywords/xml/ipad
	http://10.xx.xx.xx:3000/ebay/finding/keywords/protobuf/ipad

### Routes (server)
	return select searchResult.item, errorMessage
    from ebay.finding.findItemsByKeywordsXML where keywords = '{keywords}'
    via route '/ebay/finding/keywords/xml/{keywords}' using method get;

	return select searchResult.item, errorMessage
    from ebay.finding.findItemsByKeywordsProtobuf where keywords = '{keywords}'
    via route '/ebay/finding/keywords/protobuf/{keywords}' using method get;

### Tables (server)

	create table ebay.finding.findItemsByKeywordsXML
    on select post to 'http://localhost:6000/mock.xml'
         using headers 'X-EBAY-SOA-SECURITY-APPNAME'='{config.tables.ebay.finding.appname}',
                       'X-EBAY-SOA-OPERATION-NAME'='findItemsByKeywords'
         using defaults format = "JSON", limit = 5, offset = 0
         using patch 'findItemsByKeywordsXML.js'
         using bodyTemplate "findItemsByKeywords.ejs" type 'application/xml'
         resultset 'soapenv:Envelope.soapenv:Body.findItemsByKeywordsResponse'
	
    create table ebay.finding.findItemsByKeywordsProtobuf
    on select post to 'http://localhost:6000/mock.protobuf'
         using headers 'X-EBAY-SOA-SECURITY-APPNAME'='{config.tables.ebay.finding.appname}',
                       'X-EBAY-SOA-OPERATION-NAME'='findItemsByKeywords'
         using defaults format = "JSON", limit = 5, offset = 0
         using patch 'findItemsByKeywordsProtobuf.js'
         using bodyTemplate "findItemsByKeywords.ejs" type 'application/xml'
         resultset 'findItemsByKeywordsResponse'	
### Patches (server)

The following patch was used in the XML path: [findItemsByKeywordsXML.js](https://github.com/idralyuk/ql.io-protobuf-test/blob/master/tables/finding/findItemsByKeywordsXML.js).

This additional code was inserted into the above patch to decode Protobuf responses:

	var fs = require('fs'),
        _ = require('underscore'),
        Schema = require('protobuf').Schema,
        fis_schema = new Schema(fs.readFileSync(__dirname + '/util/FindItemsByKeywords.desc')),
        FindItemsByKeywordsResponse = fis_schema['com.ebay.marketplace.search.v1.services.finditemservice.FindItemsByKeywordsResponse'];

    exports['parse response'] = function(args) {
        var length = 0, idx = 0;
        _.each(args.body, function(b) {
            length += b.length;
        });
    
        var buf = new Buffer(length);
            _.each(args.body, function(b) {
                idx = idx + b.copy(buf, idx);
        });

        var fir = { 'findItemsByKeywordsResponse' : FindItemsByKeywordsResponse.parse(buf) };

        return {
            type: 'application/json',
            content: JSON.stringify(fir)
        };
    }

**Note: two optimizations can be made to the above code: a) allow the patch to return the json structure instead of a string that will need to be parsed again and b) receive buffer length as an argument in order to avoid looping through the data buffers twice.**

### Mock Server
    var _ = require('underscore'),
    fs = require('fs'),
    url = require('url'),
    util = require('util'),
    http = require('http');
    
    var port = 6000;
    
    function endsWith(str, suffix) {
        return str.indexOf(suffix, str.length - suffix.length) !== -1;
    }
    
    var server = http.createServer(function(req, res) {
        var file = __dirname + '/data/' + req.url

        var cType;

        if (endsWith(req.url, '.xml')) {
            cType = 'text/xml;charset=UTF-8';
        } else if (endsWith(req.url, '.json')) {
            cType = 'application/json;charset=UTF-8';
        } else if (endsWith(req.url, '.protobuf')) {
            cType = 'application/octet-stream;charset=UTF-8';
        }
        
        var stat = fs.statSync(file);
        res.writeHead(200, {
            'Content-Type' : cType,
            'Content-Length' : stat.size
        });
        
        var readStream = fs.createReadStream(file);
        util.pump(readStream, res, function(e) {
            if (e) {
                console.log(e.stack || e);
            }
            res.end();
        });
    });
    
    server.listen(port, function() {
        console.log('\nmock server listening on ' + port);
    });


Please send comments/suggestions to [ql.io Google Group](http://groups.google.com/group/qlio).
