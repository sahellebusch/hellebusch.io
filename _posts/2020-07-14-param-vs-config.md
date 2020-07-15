---
layout: post
author: Sean Hellebusch
title: Service Configuration vs. Paramaterization
categories: [tech]
tags: [tech,service]
---

### An Easy Mistake To Make

There is a difference between servince configuration and paramaterization that is often ignored or goes unnoticed, which ultimately leads two a conflation of the two. When your service configuration is stored in a format like JSON or YAML, this often means that the tructure becomes deeply nested and untangling the mess is an cumbersome, and expensive, task. That's why I think it's important to understand the difference between service configuration and paramaterization.

### The Difference

A service configuration is typically a specification that defines how your service is _arranged_. Some typical examples that come to mind are database, logging and cache configurations. In other words, the configuration does not drive expected outcomes. You should be able to theoretically change the configuration, but given the same inputs (for example, if two databases had the same data), then the outcome should not change. Parameterization, on the other hand, _does_ drive expected outcomes.  It's not all that common for services to have paramaterizations, however it's likely most software developers will encounter them at some point of their careers.

### Example

I once was the lead for a data pipeline project. We followed strict SOC II guidelines and therefore had to be very careful when it came to our governance engine. That engine was a process that consumed some structured parameterizations as a string in the environment, which then drove how we scrubbed the data.  Here's a contrived, but siccunct, example of how what I mean.

```javascript
const filtersParams = JSON.parse(process.argv[2]);

const user = {
    firstName: 'Howard',
    lastName: 'Langston',
    ssn: '123456789',
    dob: new Date('July 30, 1947')
}

const filtersFunctions = {
    firstName: user => user.firstName = '[REDACTED]',
    lastName: user => user.lastName = '[REDACTED]',
    dob: user => user.birthday = '[REDACTED]',
    ssn: user => user.ssn = '[REDEACTED]'
}

const gobernate = () => {
    console.log('\n### User before gobernation\n', JSON.stringify(user, null, 2));
    
    Object.entries(filtersParams).forEach(([filter, shouldFilter]) => {
        if (filter && shouldFilter) {
            filtersFunctions[filter](user);
        }
    })
     
    console.log('### User after gobernation\n', JSON.stringify(user, null, 2));
}

gobernate();
```

Running this script with two sets of parameters, but with the same data, will result in two different outputs.

```bash
$ node test.js '{"firstName":true,"lastName":false,"ssn":true,"dob":false}' 

### User before gobernation
 {
  "firstName": "Howard",
  "lastName": "Langston",
  "ssn": "123456789",
  "dob": "1947-07-30T08:00:00.000Z"
}
### User after gobernation
 {
  "firstName": "[REDACTED]",
  "lastName": "Langston",
  "ssn": "[REDEACTED]",
  "dob": "1947-07-30T08:00:00.000Z"
}

$ node test.js '{"firstName":true,"lastName":true,"ssn":true,"dob":true}' 

### User before gobernation
{
  "firstName": "Howard",
  "lastName": "Langston",
  "ssn": "123456789",
  "dob": "1947-07-30T08:00:00.000Z"
}
### User after gobernation
 {
  "firstName": "[REDACTED]",
  "lastName": "[REDACTED]",
  "ssn": "[REDEACTED]",
  "dob": "[REDACTED]"
}
```