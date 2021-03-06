# EkoQueue

[![Build Status](https://travis-ci.org/EkoCommunications/EkoQueue.svg?branch=master)](https://travis-ci.org/EkoCommunications/EkoQueue) [![Code Climate](https://codeclimate.com/github/EkoCommunications/EkoQueue/badges/gpa.svg)](https://codeclimate.com/github/EkoCommunications/EkoQueue) [![Test Coverage](https://codeclimate.com/github/EkoCommunications/EkoQueue/badges/coverage.svg)](https://codeclimate.com/github/EkoCommunications/EkoQueue/coverage)

EkoQueue is a queue manager that allows you to easily dispatch and process jobs with an easy to use interface. EkoQueue wraps the excellent [kue.js](https://github.com/Automattic/kue) and also comes with a UI to monitor and manage the queue jobs.

EkoQueue is meant to be fully scalable with no persistance on disk. The only dependency is a Redis server.

## Requirements

* Node.js 6.*
* Redis Server

## Installation

```
$ npm install eko-queue --save
```

## Configuration

The queue module exposes a constructor function that accepts a **job provider** and a hash of options to connect to redis.

* **jobProvider** (an object that exposes instances of jobs and optionally a job path for job classes)
* **options** (object/hash of configuration options for redis)
	* host (host for redis server)
	* port (port for redis server)
	* prefix (prefix to use for queue's jobs)


## Job provider

The job provider is a simple object that allows you to easily compose your queue jobs with dependency injection as well as provide a path for job classes that don't rely on dependency injection.

The job path must use the key `jobPath` and the keys of the initialized instances must match the name of the job in the dispatcher and processor.

Look inside the `examples` directory for more examples.

### Examples

A job provider can be as simple as returning an object with a `jobPath` key as below.

```javascript
const path = require('path');
const jobPath = path.join(path.dirname(__filename), 'jobs');

module.exports = { jobPath };
```

Or it can contain a mixture of the `jobPath` and depencency injected instances of job classes as seen below.

```javascript
const path = require('path');
const jobPath = path.join(path.dirname(__filename), 'jobs');

const Email = require('./services/email');
const EmailJob = require('./jobs/email-service-job');

/**
 * Wrap email job in function to inject dependencies.
 * These functions themselves cannot have any dependencies.
 * @return {EmailJob}
 */
const emailJob = () => {
  /**
   * Simulating instance of email client that sends email
   * @type {Object}
   */
  const emailClient = {
    send: (address, subject, body, callback) => {
      callback(null, `Email sent to ${address}`);
    },
  };

  /**
   * Email service
   * @type {Email}
   */
  const emailService = new Email(emailClient);

  /**
   * Finally instantiate and return email job with email service injected
   */
  return new EmailJob(emailService);
};

/**
 * Export the jobPath for job classes that require no dependency injection
 * and then export the jobs that have been initialized by their job name
 * @type {Object}
 */
module.exports = {
  jobPath,
  'email-service': emailJob(),
};

```

## Dispatching jobs

The queue dispatcher has a single method to dispatch jobs. The dispatch method takes the job name as the first paramater and following optional parameters act as the parameters for the handle method of the job class.

### Methods

* Dispatcher#dispatch(*string* job, *any* ...params)

### Examples

```javascript
const EkoQueue = require('eko-queue')(jobProvider, {});
const dispatcher = EkoQueue.dispatcher();

// Using additional parameters to pass through to the handle method
dispatcher.dispatch('email', 'andrew@ekoapp.com', 'Welcome Back!');

// Or you can just send an object through with your parameters
dispatcher.dispatch('email', {
  email: 'andrew@ekoapp.com',
  subject: 'Welcome Back!',
});
```

## Processing jobs

The queue processor has two (2) methods to process jobs. You can process a single job or process all the jobs. If running the `processAll` method it will also subscribe to dispatched jobs even after it is started.

### Methods

* Processor#process(*string* job)
* Processor#processAll()

### Examples

```javascript
const EkoQueue = require('eko-queue')(jobProvider, {});
const processor = EkoQueue.processor();

// Process individual job
processor.process('email');

// Process all jobs
processor.processAll();
```

## Job classes

The job classes are what the processor uses to process the job. Job classes must all live in the same directory and have a suffix of *job* in the filename. An example would be a job named **email** would have a filename of `email-job.js`.

The job class require a `handle` and `failed` method that both accept optional parameters that will be passed from the dispatcher. The job classes can also contain optional `concurrency` and `priority` properties.

The `handle` and `failed` methods also allow you to yield to promises so you can write your code in a more produral manner. You can yield to generators or promises.

### Example

```javascript
'use strict';

class EmailJob {
  /**
   * Number of jobs to run concurrently
   * @return {number}
   */
  get concurrency() {
    return 1;
  }

  /**
   * Priority of job 10 is lowest, -10 is highest
   * @return {number}
   */
  get priority() {
    return 0;
  }

  /**
   * Handle job
   * @param {string} email
   * @param {string} subject
   */
  * handle(email, subject) {
    console.log(`Sending email to ${email} with the subject ${subject}`);
  	const results = yield someEmailClient.send(email, subject, 'Hello from EkoQueue');
  }

  /**
   * Handle job failure
   * @param {string} email
   * @param {string} subject
   */
  * failed(email, subject) {
    console.error(`Failed sending email to ${email} with the subject ${subject}`);
  }
}

module.exports = EmailJob;
```
## Running tests, linter

* Run `npm test` to run the test suite
* Run `npm run lint` to run the linter

## Running examples

Checkout this repository and run the following in two (2) different terminal sessions. The order does not matter.

    $ DEBUG=eko* nodemon examples/process.js
    $ DEBUG=eko* nodemon examples/dispatch.js

## Running the server

**Todo:** Export server from constructor. In the mean time you can checkout this repository and run `npm run server`.
