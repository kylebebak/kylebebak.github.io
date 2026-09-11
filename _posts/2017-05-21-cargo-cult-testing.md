---
layout: post
comments: true
title: "Cargo Cult Testing"
categories: code testing
tags: [unit-testing, system-testing, requests, mock, tdd]
---


I was doing research on how to test an [HTTP client I wrote](http://requester.org) that uses the [Requests](http://docs.python-requests.org/en/master/) library and came upon [this question](http://stackoverflow.com/questions/9559963/unit-testing-a-python-app-that-uses-the-requests-library) on StackOverflow.

The OP wants a way to test client code that uses `requests` to consume an API. The answers are all toy examples of how to mock an endpoint so that when hit by `requests` it returns response X. Then their tests assert that hitting this endpoint with `requests` returns response X.

This post is written in two parts: a critique of shoddy testing, motivated by libraries that seem to encourage shoddy testing, and a summary of techniques for [effective testing](#tests-worth-running).


## Tautological Testing
Here's the code in the top answer, making use of `httmock`, which is described as "wonderfully simple and elegant".

~~~py
from httmock import urlmatch, HTTMock
import requests

# define matcher:
@urlmatch(netloc=r'(.*\.)?google\.com$')
def google_mock(url, request):
    return 'Feeling lucky, punk?'

# open context to patch
with HTTMock(google_mock):
    # call requests
    r = requests.get('http://google.com/')
print r.content  # 'Feeling lucky, punk?'
~~~

This answer is fine. It points to a library you can use to mock an API, and hence test an API client, which is what the OP wants to do. But it makes no mention of an actual API client. Scrolling down, I saw all the answers were like this. So I checked out the mocking libraries they mentioned, and what I found was disappointing.

The answer above, and all the rest, were basically copied from the READMEs of one of these API mockers:

- <https://github.com/getsentry/responses>
- <https://github.com/gabrielfalcao/httpretty>
- <https://github.com/patrys/httmock>

None of the examples in the READMEs tests what client code does with responses. They test the mockers themselves, and that `requests.get` sends GET requests.

My guess is the takeaway for the average coder is this: these libraries help you test another library called Requests, written by Kenneth Reitz. If your tests fail send him an email, he'll appreciate it.


## Confusion in Terms
So far there's nothing technically wrong. We have 3 similar libraries designed to test API clients, with very similar examples that make no allusion to API clients written by users of the libraries.

But if you look closer, things get weird. [Here's an example](https://github.com/getsentry/responses#response-body-as-string) from the `responses` library:

~~~py
import responses
import requests


def test_my_api():
    with responses.RequestsMock() as rsps:
        rsps.add(responses.GET, 'http://twitter.com/api/1/foobar',
                 body='{}', status=200,
                 content_type='application/json')
        resp = requests.get('http://twitter.com/api/1/foobar')

        assert resp.status_code == 200

    # outside the context manager requests will hit the remote server
    resp = requests.get('http://twitter.com/api/1/foobar')
    resp.status_code == 404
~~~

The name of the test is `test_my_api`. But `responses` is a substitute for an API. It patches `requests` so that no request it sends can ever hit an API. __By definition, it can't be used to test an API__. This library is useful, but not even the authors really understand what to use it for.

This name, `test_my_api`, is used in 3 examples, and the library has 40+ contributors. Still, maybe it's an innocent mistake. They could rename the test to `test_my_api_client`.

But it's not testing that either, because in the examples there is no API client. There's just the Requests library.

[HTTPretty](https://github.com/gabrielfalcao/httpretty) also seems confused about its purpose. Its repo description starts with "HTTP client mocking tool for Python".

HTTPretty mocks the API, not the API client. Mock means imitate. Postman and [Requester](https://github.com/kylebebak/Requester) mock the API client. HTTPretty __tests__ the API client. The author should have gotten this right; he says HTTPretty was inspired by [FakeWeb](https://github.com/chrisk/fakeweb), whose description is more wooden but more accurate: "__Ruby test helper for injecting fake responses to web requests__".

To be fair, a little later he almost does get it right:

>Once upon a time a python developer wanted to use a RESTful api, everything was fine but until the day they needed to test the code that hits the RESTful API: what if the API server is down? What if its content has changed ?

Cool, he's talking about testing the API client. And yeah, having your build fail because someone else's API is down for 5 minutes is probably not what you want. But __if its content has changed?__ If your API client expects response X but now gets Y instead, __your API client is broken, and needs to be updated__. You'd better __hope__ your build fails.


## Tests Don't Make Code Correct
At first glance, [this post](http://engineroom.trackmaven.com/blog/real-life-mocking/) makes more sense. It's a 15 minute read with 50+ lines of code to test the following method:

~~~py
def _get(self, url, retries=3):
    """Make a GET request to an endpoint defined by 'url'"""

    while retries > 0:
        try:
            response = requests.get(url=url)
            try:
                response.raise_for_status()
                return response.json()
            except requests.exceptions.HTTPError as e:
                self._handle_http_error(e)
        except (requests.exceptions.ConnectionError,
                requests.exceptions.Timeout) as e:
            retries -= 1
            if not retries:
                self._handle_connection_error(e)
~~~

But there are problems.

The method ignores best practices for using Requests.

>You can tell Requests to stop waiting for a response after a given number of seconds with the timeout parameter. Nearly all production code should use this parameter in nearly all requests. Failure to do so can cause your program to hang indefinitely.

It explodes if the response content isn't valid JSON, which could happen if the API server is configured to respond with HTML in the event of an error (a common mistake).

But there's something much worse. Look what happens if the API returns a 4XX or 5XX. `response.raise_for_status` raises an exception, which is caught and handled by `self._handle_http_error`. But there's no `return` statement, and `retries` isn't decremented unless there's a connection error or a timeout, so the request will just be sent again and again and again.

If you forget to add an API token to the headers, this method tries to DDoS the API. If your overeager client has been sending too many requests and the API tells it to fuck off for a while with a `429`, this method tries to DDoS the API. This second scenario is kind of funny.

The only non-trivial thing it does is retry the request N times before handling a connection error. Then again, this functionality [is provided by Requests](https://stackoverflow.com/questions/15431044/can-i-set-max-retries-for-requests-request), with support for exponential backoff, and it's not going to DDoS anyone's server.

---

So that's `_get`. As it's written, if you pass `retries=3` it sends a GET request to `url`. If the response is 4XX or 5XX, it resorts to anti-social behavior. If the connection is laggy, the request could hang for minutes before raising an error. It will then be retried immediately. Oh, and it will only be retried two times.

It's important to remember the author wrote 50+ lines of code to test this method. But his tests, like his method, are shoddy. Also important: the tests cover nearly "100%" of `_get`, __but they fail to catch an obvious, show-stopping bug__.

Why? They were written with __insufficient imagination__. The author didn't imagine all the things the method might have to deal with. He didn't put it through its paces. There are software shops where this level of test coverage would earn him a gold star or a bronze shield. But "100% coverage" isn't sufficient. Tests hitting every line of code in a method doesn't prove it's correct.

Also, if your code and tests are well written, "100%" is often overkill. Here's how I might have written `_get` with exponential backoff.

~~~py
def _get(self, url, retries=3, timeout=3, backoff=5):
    """Sends GET request to `url`."""
    
    backoff_factor = 1
    while True:
        try:
            response = requests.get(url=url, timeout=timeout)
            try:
                response.raise_for_status()
            except requests.exceptions.HTTPError as e:
                return self._handle_http_error(e)
            try:
                return response.json()
            except JSONDecodeError:
                return self._handle_json_error(response)
        except (requests.exceptions.ConnectionError, requests.exceptions.Timeout) as e:
            if retries <= 0:
                return self._handle_connection_error(e)
            time.sleep(backoff * backoff_factor)
            retries -= 1
            backoff_factor *= 2
~~~

This method isn't trivial, it probably warrants tests. That said, I tested it in the REPL, and it manages to avoid the bugs in the author's implementation.


## Tests Worth Running
The post above need not have been written, not because `_get` is buggy, but because using `mock` to monkey-patch Requests isn't the best way to test `_get`.

Use something like [httpbin](http://httpbin.org) or [json-server](https://github.com/typicode/json-server) instead. `httpbin` runs an HTTP server that responds to all kinds of requests, and you can specify the status code and content of the response. Make it a part of your build.

This way your `_get` method sends real requests and gets real responses. You don't have to mock anything, so your tests are shorter and easier to write, and they test more of the intended behavior of your method. Avoid monkey-patch mocking unless you really need it.

---

The first part of this post may seem like gratuitous criticism of libraries that, in the right hands, can be put to good use. It may even seem like a critique of testing in general. If you're feeling that way I've made a mistake, and I'd like to clarify.

__Tests are tools__. [Developer tools](https://en.wikipedia.org/wiki/Programming_tool), like version control and code linters. Your end users couldn't care less about tests.

But in many cases not writing tests is insane. Let's say you're working on something fairly large and complex. Without tests, you have __no way of knowing__ it still works after you make changes. Requester, for example, has [plenty of tests](https://github.com/kylebebak/Requester/tree/master/tests) that have failed plenty of times. You want your tests to fail occasionally. If they don't, they're not telling you anything. Your team is either so good that writing tests is redundant, or you aren't testing the actual behavior of your software.

Anyway, for Requester, some tests are end-to-end tests that depend on Sublime Text. Some are decoupled from Sublime Text and run in CI. And many of them cover __a lot__ of Requester's behavior.


### Integration Tests
You __need__ tests that touch many parts of your system. Let this sink in, because it goes against what you hear from a lot of the TDD and unit testing crowd. Why? For starters, in a complex system, testing everything as units requires more code than anyone could write.

Integration tests have big advantages. You get __way more bang__ for your buck. So much more that you can hit each piece of your system __with multiple tests__, testing how functions behave in different contexts. And you don't sacrifice specificity. If an integration test fails and you have a fair understanding of your system, you can trace the failure to its cause very quickly.

Try to write tests that cover as much behavior as possible, and test just the behavior, not the implementation. This is another issue with unit tests: they tend to test the implementation. Too many unit tests may __discourage__ you from refactoring your system, because this also means refactoring your tests. They may force you into radically decoupled architectures whose main purpose is to facilitate more unit tests.

On the point of architecture, however, a healthy focus on unit tests __can guide you towards a sweet spot__. Key algorithms or functions with complex behavior should be unit-tested. They should be decoupled from your system if possible.


### Case Study: Requests
My beef with these libraries is they don't encourage useful tests. Their examples all feature Requests and tests of its vanilla functionality. Requests is downloaded more than 13 million times a month; it's been tested to death.

The most damning thing is they seem unaware of how [Requests does its own tests](https://github.com/requests/requests/blob/master/tests/test_requests.py). It doesn't use [mock](https://docs.python.org/3/library/unittest.mock.html), and it monkey-patches almost nothing. It uses `httpbin` (also written by Kenneth Reitz) to send real requests to real endpoints; it uses integration tests.

As for the HTTP mocking libraries, I figure they owe their popularity to devs who heard unit testing is the __right way__ to test, that only "mocks" can give you that level of control, and they believed it.

To sum up:

- test behavior, not implementation
- there's no such thing as 100% coverage
- if you want good coverage, you'll need integration tests
- if you're afraid to refactor because of tests, use fewer unit tests
- use a judicious mix of both

Most importantly, it takes good engineers to write good tests. Anyone who believes otherwise is fooling themselves. I see these libraries and their toy examples, I imagine mountains of useless code they inspire, and I come to an unsurprising conclusion: many programmers are dogmatic, and critical thinking is hard. Feynman knew this about scientists. He called it cargo cult science.


## Suggested reading:
- <http://calteches.library.caltech.edu/51/2/CargoCult.htm>
- <http://rbcs-us.com/documents/Why-Most-Unit-Testing-is-Waste.pdf>.
- <http://david.heinemeierhansson.com/2014/tdd-is-dead-long-live-testing.html>
