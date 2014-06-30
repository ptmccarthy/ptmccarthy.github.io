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

One of the really nice things Frisby provides is the ability to chain multiple expectations about a response together into one overall test case. Frisby also provides [several nice built-in matchers](http://frisbyjs.com/docs/api/) for things you will commonly want to test, all built on Jasmine. You can also use most of Jasmine's built-in matcher functionality, but more on that later.

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

### Custom Tests and Jasmine Matchers

The built-in Frisby tests are great and useful for a certain set of tasks, but to do more interesting, complex, or unusual tasks you'll have to write your own. This is where I found the official Frisby documentation lacking, but found some very useful functionality tucked away in the source.

#### The .after() and .afterJSON() Callbacks

These are two *very* useful methods for extending and customizing your Frisby test cases, and I'm disappointed that the official API documentation doesn't call them out. I was clued into them by an example test I found on GitHub, so I dug a little deeper to figure out what exactly they provide. Here's the "missing" documentation for these valuable methods, as far as I can tell:

*.after(function (error, response, body) {})*

Callback that is executed once the request has been executed and a response received.

*.afterJSON(function (body) {})*

Callback that is executed once the request has been executed and a response has been received. For convenience, this method will parse a body containing a JSON string into a Javascript object for you.

#### Using Native Jasmine Matchers

Since Frisby is built on top of Jasmine, you can actually use most of [Jasmine's built-in matchers or write your own custom matchers](https://github.com/pivotal/jasmine/wiki/Matchers) within the `.after()` or `.afterJSON()` callback.

A primitive example using `.after()` and the built-in Jasmine matcher `expect(foo).toMatch(bar)`:

```javascript
frisby.create('GET something and use the after callback')
  .get('http://httpbin.org/get?param1=hello')
  .after(function (err, res, body) {
    expect(JSON.parse(body).args.param1).toMatch('hello')
  })
.toss();
```

Note that because the response body is JSON, we could clean this up and instead use the `.afterJSON()` callback instead. This example is functionally the same thing as above:

```javascript
frisby.create('GET something and use the afterJSON callback')
  .get('http://httpbin.org/get?param1=hello')
  .afterJSON(function (body) {
    expect(body.args.param1).toMatch('hello')
  })
.toss();
```

### Storing Response Data For Later

One of the cool things about node and javascript in general is their asynchronous nature. When we run our test suite, there is no guarantee of the order in which test cases execute or finish because they are essentially running simultaneously. On one hand this great since it means the test suite will run *fast*. But what if we need to enforce some order of operations in our test? The aforementioned `.after()` callback allows us to do exactly that.

#### Chaining Test Cases

For example, imagine that you need to test a REST service that uses an x-access-token in the request header to authenticate clients. So first you have to POST valid login details via JSON, store the authentication token, and then include it in the headers of all your subsequent requests. Now, hopefully your production server doesn't have insecure test credentials, but let's just assume we're running this in a dev or test environment that does.

httpbin.org unfortunately does not support token authentication, so you'll just have to use your imagination for these examples.

```javascript
frisby.create('POST login details')
  .post('https://some.url/rest/login',
    { username: 'a_test_user', password: 'a_test_password' },
    { json: true },
    { headers: { 'Content-Type': 'application/json' }})
  .expectStatus(200)
  .expectHeader('Content-Type', 'application/json')
  .expectJSONTypes({
    token: String
  })
  .afterJSON(function (res) {
    /* include auth token in the header of all future requests */
    frisby.globalSetup({
      request: { 
        headers: { 'x-access-token': res.token }
      }
    });

    frisby.create('GET data requiring token authentication')
      .get('https://some.url/rest/data')
      .expectStatus(200)
      .expectHeader('Content-Type', 'application/json')
      .expectJSON({ /* whatever data you expect */ })
    .toss();

    /*
      however many more test cases you want to run using
      the authentication token
    */
  })
.toss();
```

A lot more is going on here than in any previous examples, but I hope it's easy enough to follow along. Note that when sending raw JSON data in a POST request it is important to include `{ json: true }` and to add `{ 'Content-Type', 'application/json' }` in the request headers. Otherwise Frisby will default to using `application/x-www-form-urlencoded` which likely will not work.

After we post the login request, we expect the response to have a status of 200, a JSON Content-Type, and that the JSON body has a token string. If all of that is true, in the `.afterJSON()` callback we take the token from the response and use `frisby.globalSetup()` to add it to the request header of all subsequent requests.

From there we can make as many additional test cases, still within the `.afterJSON()` callback, as we want and they will all have the authentication token available to them.

### Putting It All Together: Handling Data Encoding and Gzip

It's common for REST data services to compress their responses with gzip in order to speed up transfers and save bandwidth. Some services may allow you to request that your data *not* be gzip-encoded, or you may have control over this setting in your test environment. But let's say that either you don't have control over it, or that you specifically want to test that a REST service is appropriately encoding data (or you just like a challenge). Frisby does not handle gzip-encoded messages by default, but it gives us a foundation to implement a solution.

The secret sauce that I discovered in working through how to handle gzip with Frisby is that you __must__ specify in your request to use `{ encoding: null }` or else your gzipped response body will be cast from a binary object to a String - and good luck trying to unzip that! This is not a functionality documented by Frisby, but Frisby is using the node `request` module so we can pass in relevant parameters via Frisby just as if we were using `request` directly. Unfortunately, for reasons unknown to me it seems that you *cannot* put this request parameter into the global setup. If you do that, it breaks the entire test suite so you must add it to each GET request.

To actually handle the decompression we need to add the `zlib` module to our test spec to provide the gunzip functionality. Conveniently, this module is included with node so we don't need to add it to our `package.json` file; a simple require will do. Additionally we can write an async helper function to gunzip objects to use in our tests. With these components we can unzip encoded responses in the `.after()` callback of a Frisby test case and use Jasmine matchers to test things about our unzipped response!

```javascript
var frisby = require('frisby');
var zlib = require('zlib');

var unzipResponse = function(body, cb) {
  zlib.gunzip(body, function(err, result) {
    cb(result.toString());
  });
}
frisby.create('POST login details')
  .post('https://some.url/rest/login',
    { username: 'a_test_user', password: 'a_test_password' },
    { json: true },
    { headers: { 'Content-Type': 'application/json' }})
  .expectStatus(200)
  .expectHeader('Content-Type', 'application/json')
  .expectJSONTypes({
    token: String
  })
  .afterJSON(function (res) {
    frisby.globalSetup({
      request: { 
        headers: {
          'x-access-token': res.token,
          /* explicity add a header to accept gzip-encoding */
          'Accept-Encoding': 'gzip' 
        }
      }
    });

    frisby.create('GET data requiring token authentication')
      /* notice the { encoding: null } added to the request */
      .get('https://some.url/rest/data', { encoding: null })
      .expectStatus(200)
      .expectHeader('Content-Type', 'application/json')
      .expectHeader('Content-Encoding', 'gzip')
      .after(function(err, res, body) {
        var unzipped;
        unzipResponse(body, function(result) {
          unzipped = result;
        });

        waitsFor(function() {
          return unzipped;
        });

        runs(function() {
          /* Jasmine matchers here, for example: */
          expect(unzipped).toMatch(/* expected data */)
        });

      })
    .toss();
  })
.toss();
```

One last thing of note about this test is that because the `unzipResponse()` and `zlib.gunzip()` functions are asyncronous, we have to tell Jasmine to wait for `unzipped` to be defined. Frisby uses Jasmine 1.x which uses the older waitsFor/runs syntax for async control flow. Jasmine 2.x looks a bit different. Refer to the [Jasmine documentation](http://jasmine.github.io/) for a detailed explanation.

### Conclusion

I've enjoyed hacking on test cases similar to these examples using the conveniences of Frisby and digging deeper to use the underlying Jasmine and Request components to extend functionality. The power and ease of these tools makes creating a comprehensive REST test suite a quick and mostly-painless process.
