---
layout: post
author: Sean Hellebusch
title: "Dependency Injection > Proxy Injection"
categories: [journal]
tags: [opinion,tech]
---

## The Issue

A common issue I've experienced while developing Node servers is how to best inject mocks for tests. Libraries like [proxyquire](https://www.npmjs.com/package/proxyquire) override the `require` function and act as a pass-through proxy. This is achieved by overriding the `require` function and using a custom loader based on the injected module paths. Seems straightforward enough, but I've seen complicated examples with nested injections that invalidate the module cache from suite to suite, thus resulting in false positives. I'll admit I used to use proxy injection, but my opinion was quickly changed when I had to debug an integration test only to find out that a function called `isNumber` was being mocked out in a nested injection three levels deep. I spent _a couple of hours_ looking for something that shouldn't have been done in the first place. Previously, I would have just rolled my eyes at the developer, but that solves nothing. Rather, I reassessed my opinions and ultimately changed the way I look at Node servers. 

## Passing Dependencies From Top Down

I used to really dislike passing variables from the top down like this. I thought it was too verbose and would be a maintenence nightmare, but in the end it has made development easier because _tests became easier to write_. Instead of adding yet another dependency for testing, we can simply change the way we initialize applications by passing in common, application level dependencies at the entrypoint. To best understand what I mean, let's see what I believe an idiomatic express Node server ought to look like in code.

### server.js

This is the entrypoint for our server. It's here that we will initiate any top-level dependencies to be consumed throughout the application. In this example, I have a configuration and a logger objects.

```javascript
'use strict';

const config = require('config');
const initApp = require('./server/app');
const logger = require('./lib/logger');

const port = config.port || 3000;
let server = initApp({ config, logger }).listen(port, () => {
  let { address, port } = server.address();
  logger.info(`Server listening at ${address}:${port}`);
});

const shutdown = signal => () => {
  logger.info(`${signal} received, shutting down the server.`);
  server.close();
};

// listen for TERM signal .e.g. kill
process.on('SIGTERM', shutdown('SIGTERM'));

// listen for INT signal e.g. Ctrl-C
process.on('SIGINT', shutdown('SIGINT'));
```

### server/init-app.js

By using a function to initialize, we can continue to pass those dependencies mentioned above to any routers.

```javascript
'use strict';

const express = require('express');
const initStatusRouter = require('./route/status');

const initApp = ({ config, logger }) => {
  logger.info('Initializing server...');
  const app = express();
  app.use('/status', initStatusRouter({ config, logger }));
  return app;
};

module.exports = initApp;
```

### server/route/status.js

This is the simplest router that all servers should contain.

```javascript
'use strict';

const express = require('express');
const initStatusRouter = require('./route/status');

const initApp = ({ config, logger }) => {
  logger.info('Initializing server...');
  const app = express();
  app.use('/status', initStatusRouter({ config, logger }));
  return app;
};

module.exports = initApp;
```

## Testing

Now that we have the server setup and ready to go, we can see what testing using [supertest](https://www.npmjs.com/package/supertest) looks like. Now that we have a function to start our server, we can easily pass our own stubs into the initialization function. This makes dependency injection much simpler. It doesn't require any special modules to inject code code and is a more correct representation of how our code would be executed.

### test/server/init-app.js
```javascript
'use strict';

const config = require('config');
const supertest = require('supertest');
const sinon = require('sinon');
const initApp = require('../../server/init-app');

const loggerStub = {
  info: sinon.stub(),
  error: sinon.stub()
};

describe('the app', () => {
  let app = null;

  before(() => {
    app = initApp({ config, logger: loggerStub });
  });

  it('returns a healthy status upon request', () =>
    supertest(app)
      .get('/status')
      .expect(200, { status: 'ok' })
      .then(() => {
        sinon.assert.callCount(loggerStub.info, 2);
        sinon.assert.notCalled(loggerStub.error);
        sinon.assert.calledWith(loggerStub.info, 'Initializing server...');
        sinon.assert.calledWith(loggerStub.info, sinon.match('Status Check:'));
      }));
});

```

## The Code

The code to this example is [here](https://github.com/sahellebusch/examples/tree/master/idiomatic-express-server). Forgive me for not keeping up with it, but it does work!

### Cheers!

Thanks for reading, hope you enjoyed it. Comments or questions, feel free to email me at sahellebusch at gmail dot com.
