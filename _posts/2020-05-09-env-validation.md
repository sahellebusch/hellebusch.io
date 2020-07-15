---
layout: post
author: Sean Hellebusch
title: Environment Variable Validation
categories: [tech]
tags: [opinion,tech]
---

### The Problem

I've worked at two orgnizations now that have have tripped over themselves in their efforts to more closely follow the [12 Factor App](https://12factor.net). The journey to following those rules takes time and is well worth it. One of the biggest mistakes I see is when applying [factor III](https://12factor.net/config): storing config in the environment. This provides many benefits, however without actually validating the config it's easy to run into runtime errors. Runtime errors are arguably the most expensive on average for any organization and can have severe impacts on the service's ability to complete basic tasks or, in extreme cases, service availability.

### The Solution

The good thing is, this is easily preventable! Hooray, programming! 

I encountered this same issue myself when I was leading the development of a data pipieline at an urban mobility company. As you can imagine, there were multiple services and a pretty involved set of configuration options to tweak the system's performance. During development, we saw a number of crashes due to the problem oultlined above. Not only that, it was obvious early on that maintaining all the configuration options, including their documentation, was increasingly difficult. In hindsight, the solution was simple: create a configuration class that validated the environment _before_ the service even started up. Modern day container tooling often has the ability to juggle service deployments for you, so I decided that if validation failed, the service should die and the previous should continue to run. To sovle this problem, we implemented kubernetes along with helm, which allowed us to package services and configuration together. This prevented downtime due to a misconfiguration and reduced the stress and cost of changing configurations. 

### Factor X 🤝

I'm also fond of [factor X](https://12factor.net/dev-prod-parity), keeping environment parity as close as possible, because of how complimentary it is to factor III. In classic tech shops, where operations teams productionized code, it was common for environments to drift substantially. This is problematic in a number of ways - including a potential increase in bugs, tension between organizations and a lack of developer confidence. 

Luckily, tools like Docker along with docker-compose and the like have made it much easier because they provide an isolated runtime environment that should replicate production. Being able to replicate environments locally makes this factor complimentary to factor III.

### Example Solutions

I primarily write TypeScript services, so that's what my two examples are written in. I'm also a big fan of [dependency injection](https://hellebusch.io/journal/dependency-injection-for-testing.html), so I prefer to pass these objects top down.

This example is a simple version of a config class I wrote for a Fastify service. I've also done the same for a Hapi service. You can check out that example [here](https://github.com/sahellebusch/hapi-graphql-ts/blob/master/src/lib/config.ts)

```typescript
import envSchema from 'env-schema';
import S, { ObjectSchema } from 'fluent-schema';

// fastify uses pino as it's logger solution
const pinoLogLevels = ['fatal', 'error', 'warn', 'debug', 'info', 'trace'];
const nodeEnvs = ['production', 'development'];

type EnvironmentVar = string | number;

interface EnvironmentVars {
  [key: string] : EnvironmentVar
}

export default class Config {
  static readonly schema: ObjectSchema = S.object()
    .prop('NODE_ENV', S.string().enum(nodeEnvs).required())
    .prop('LOG_LEVEL', S.string().enum(pinoLogLevels).required());

  private env: EnvironmentVars;

  // for supporting dependency injection
  public static init(injectedEnv?: EnvironmentVars): Config {
    const options = {
      schema: this.schema,
      data: injectedEnv || process.env,
      dotenv: true,
    };

    return new Config(envSchema(options));
  }

  private constructor(env: EnvironmentVars) {
    this.env = env;
  }

  public has(prop: string): boolean {
    return !!this.env[prop];
  }

  public get(prop: string): EnvironmentVar {
    if (!this.env[prop]) {
      throw new Error(`Property "${prop}" does not exist `);
    }

    return this.env[prop];
  }
}
```