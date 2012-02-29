---
title: Milestone 6
layout: post
author: subbu
---

- Clients can occasionally get socket hangup errors when origin servers close connections without
  sending a `Connection: close` header. See
  [https://github.com/joyent/node/issues/1135](https://github.com/joyent/node/issues/1135) for some
  background. To avoid such errors, http.request.js now automatically retries the request once
  provided the statement that caused the HTTP request is a `select`.

- The engine can now consume CSV response in addition to XML and JSON.

- Fixed request body processing for routes (see [issue 161](https://github.com/ql-io/ql.io/pull/161)).
