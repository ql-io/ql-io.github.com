---
title: Milestone 5
layout: post
author: subbu

---

- [OAuth example](https://github.com/ql-io/ql.io/issues/121) - OAuth2 is trivial as ql.io proxies
 headers from clients to servers. OAuth1 requires glue code to compute the Authorization header.
 See [http://ql.io/docs/oauth](http://ql.io/docs/oauth) for an example.

- Use npm installed modules for ql.io-site ([see issue 116](https://github.com/ql-io/ql.io/issues/116)).

- Handle empty response bodies gracefully ([see issue 98](https://github.com/ql-io/ql.io/issues/98)).

- Recover from partial failures in case of scatter-gather calls
 ([see issue 90](https://github.com/ql-io/ql.io/issues/90)) - some statements can result in multiple
 HTTP requests. When this happens, the engine used to fail the entire statement if any of those
 requests fail. The engine now looks for success responses and aggregates them.

- Update CodeMirror to support line-wrapping ([See issue
  11](https://github.com/ql-io/ql.io/issues/11)) - no need to split lines manually anymore.