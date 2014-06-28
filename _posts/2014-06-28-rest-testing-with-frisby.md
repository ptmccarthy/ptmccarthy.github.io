---
layout: post
title: Testing REST APIs with Node, Jasmine, and Frisby
---

At work, one thing we are building is a web application that depends on polling a data service through a REST API. It's nothing terribly complex, but I picked up the task of testing that the service provides what it is expected to, especially as further development continues. It's a tedious and mundane task to do manually, so I turned to developing a test suite built on [node.js](http://nodejs.org/) along with two excellent testing libraries, [Jasmine.js](http://jasmine.github.io/) and [Frisby.js](http://frisbyjs.com/). I had some experience with Jasmine, but I had never used Frisby before. Within a couple of hours I was up and running with a decent test suite and overall I am quite pleased with how slick the whole system is.

I did run into some snags when I tried to implement some slightly more complex tests. There is quite a bit of "hidden" functionality in Frisby that is not in the official documentation but that you can find trolling through the library's source. A little later I will share some tips and tricks I stumbled across, but first a quick introduction to Frisby.


### Setting Up a New Frisby Test Suite

I'm going to assume a very basic familiarity with node.js, Javascript, and the terminal, but I will walk through the most important steps. If you don't already have node.js and its package manager, [npm](https://www.npmjs.org/), installed go do that first.

With node and npm installed, begin by creating a new directory and editing a new package.json file. Fill in a couple of important details, like so:

```json
{
  "name": "rest_tester",
  "author": "Patrick McCarthy",
  "main": "rest_service_spec.js",
  "version": "0.0.1",
  "dependencies": {
    "frisby": "latest",
    "jasmine-node": "latest"
  }
}
```

Here we have defined our project and included our two top-level dependencies, Frisby and Jasmine-Node. With package.json created, on the command line (in your project directory) run `npm install` to have npm fetch and install all the dependencies.

Now we can begin the fun part of writing Frisby tests!

### Managing Expectations

Like most testing frameworks, test cases are defined in spec files. Create a new file and name it something like `rest_service_spec.js`. You can name it whatever you want, but note that the `_spec` ending is important.

The gist of a Frisby test case is that you first describe the test, create some form of HTTP request (GET, POST, etc.), define some expectations about the response, and then "toss the frisby" to execute the test.

One of the really nice things Frisby provides is the ability to chain multiple expectations about a response together into one overall test case. Frisby also provides [several nice built-in matchers](http://frisbyjs.com/docs/api/) for things you will commonly want to test, all built on Jasmine. You can also use most of Jasmine's matcher functionaliy, but more on that later.

Let's add a simple test to our spec:

```javascript
frisby.create('GET JSON data from an endpoint')
  .get('http://httpbin.org/get')
  .expectStatus(200)
  .expectHeader('Content-Type', 'application/json')
  .expectJSON({ 'url': 'http://httpbin.org/get' })
.toss();
```

Here we have created a test case and told Frisby to expect a 200 (OK) response status and that the response header content type is application/json. Furthermore, we can tell Frisby things we expect the JSON response body to contain with the `.expectJSON()` matcher.

We can execute this test on the command line with `jasmine-node rest_service_spec.js`. In the results we can see how these multiple assertions roll up into "one" test. If any of the assertions fail, the whole test will fail.

```
.

Finished in 0.257 seconds
1 test, 3 assertions, 0 failures, 0 skipped
```

Speaking of failed assertions, let's quickly examine a failing test by modifying the previous test case a little bit.

```javascript
frisby.create('GET JSON data with parameters')
  .get('http://httpbin.org/get?param1=hello')
  .expectStatus(200)
  .expectHeader('Content-Type', 'application/json')
  .expectJSON({ 'args': {
    'param1': 'goodbye'
  }})
.toss();
```

```
F

Failures:

  1) Frisby Test: GET JSON data with parameters 
  [ GET http://httpbin.org/get?param1=hello ]
   Message:
     Error: Expected string 'hello' to match string 'goodbye' on key 'param1'
   Stacktrace:
      ...
```

Which can of course be easily fixed by correcting our .expectJSON assertion.

```javascript
  ...
  .expectJSON({ 'args': {
    'param1': 'hello'
  }})
```

```
.

Finished in 0.262 seconds
1 test, 3 assertions, 0 failures, 0 skipped
```

The other neat built-in matcher provided by Frisby allows you to test the types of keys in JSON responses, which is quite useful if you are getting data where the values in the response are unknown or arbitrary, but you do know what *types* those values should be.

```javascript
frisby.create('GET JSON and validate types')
  .get('http://httpbin.org/ip')
  .expectJSONTypes({ 'origin': String })
.toss();
```
