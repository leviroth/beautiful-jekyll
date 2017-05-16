---
layout: post
title: Testing Reddit bots with Betamax [draft]
---

When testing code that relies on a network API, such as Reddit's, network IO
itself is a significant bottleneck. This post has a dual aim: first, it provides
an introduction to
the [Betamax](http://betamax.readthedocs.io/en/latest/index.html) library, which
speeds up integration tests by recording and reusing HTTP interactions. Second,
through the running example of testing a Reddit bot, it provides boilerplate
code that can be used to get started with Betamax in
any [PRAW](http://praw.readthedocs.io/en/latest/) client application.

# The problem #

[Here](https://github.com/leviroth/bigbenbot/blob/d746271c79a7e82ea6e908188a4f363caccc4b05/bot.py) is
a simple Reddit bot. It monitors a subreddit (/r/BigBenBot) for comments that
start with the word `!time`, and posts the current time in response. The
connection to Reddit is provided by the excellent PRAW library.

The bot will reply to any (1) non-deleted (2) top-level comment that (3) begins
with the word `!time`. We implement these rules with the following method:

```python
def check_comment(self, comment):
    """Check if comment should trigger the bot."""
    return (comment.body.split(None, 1)[0] == "!time"
            and comment.is_root
            and comment.banned_by is None)
```

Let's write some integration tests to make sure that these rules work as
expected on actual comments:

```python
import praw

from bot import BigBenBot


SUBREDDIT = 'BigBenBot'


class IntegrationTest:
    def setup(self):
        self.reddit = praw.Reddit(user_agent='BigBenBot test suite')
        self.bot = BigBenBot(self.reddit, self.reddit.subreddit(SUBREDDIT))


class TestBigBenBot(IntegrationTest):
    def test_check_comment__valid(self):
        comment = self.reddit.comment('dhb7fxz')
        assert self.bot.check_comment(comment)

    def test_check_comment__no_command(self):
        comment = self.reddit.comment('dhb7goj')
        assert not self.bot.check_comment(comment)

    def test_check_comment__not_top_level(self):
        comment = self.reddit.comment('dhb7h42')
        assert not self.bot.check_comment(comment)

    def test_check_comment__removed(self):
        comment = self.reddit.comment('dhb7g3w')
        assert not self.bot.check_comment(comment)
```

And let's run them with pytest:

```
(bigbenbot) PS C:\Users\levim\src\bigbenbot> pytest
============================= test session starts =============================
platform win32 -- Python 3.6.1, pytest-3.0.7, py-1.4.33, pluggy-0.4.0
rootdir: C:\Users\levim\src\bigbenbot, inifile:
collected 4 items

test_bot.py ....

========================== 4 passed in 4.36 seconds ===========================
```

Fantastic, everything worked! But it's a bit slow. Running four tests took us
about four seconds, most of which time was spent waiting for API requests to
complete. If we have a lot of tests, this wait can become unacceptable. PRAW
itself, for example, has 531 tests at the time of writing, and they generate
1436 requests to Reddit's servers. As our four tests use eight requests,[^1] a
rough estimate of half a second per request puts us in the ballpark of twelve
minutes to run all of PRAW's tests. That's quite a wait, especially if you're
going to have to turn around and fix some bugs.

Furthermore, our tests depend on the reliability of Reddit's servers and of our
own Internet connection. If there's any hiccup, we'll have to re-run the tests -
an especially bad result if this takes twelve minutes. And if Reddit's servers
go down for maintenance, we have to wait before we can continue testing.

# The solution, part 1: Betamax #

Betamax provides a solution to these problems. With Betamax, we can record HTTP
interactions from our tests in files called *cassettes*. On subsequent runs of
the test suite, the cassettes are used to simulate the interactions instead of
relying on the network.

For example, each test that we've written so far fetches a comment. Each time we
run one of the tests:

1.  Our code calls one of PRAW's methods.
2.  PRAW sends an HTTP request to reddit.com.
3.  PRAW receives a response from reddit.com.
4.  PRAW converts that response into Python data.
5.  Our code makes use of that data.

Steps 2 and 3 are where we spend most of our time. With Betamax, however, we can
record the HTTP request in response. Then, on subsequent runs of the test, we go
through the following steps instead:

1.  Our code calls one of PRAW's methods.
2.  PRAW's request to reddit.com is intercepted by Betamax.
3.  PRAW receives a simulated response, corresponding to the previously recorded
    interaction.
4.  PRAW converts that response into Python data.
5.  Our code makes use of that data.

Because we've swapped out network IO for disk IO, the running time is much
shorter. And because we aren't actually using the network, our tests won't fail
if a Reddit server happens to go down.

Let's see how to implement this in actual code. First, we'll need a couple
imports:

```python
import betamax
from betamax_serializers import pretty_json
```

`betamax` is the only module we absolutely need here. However, the `pretty_json`
serializer will make our cassettes much more readable, and that's especially
helpful for a tutorial such as this.

Next, we perform a bit of configuration. Again, the lines concerning
`pretty_json` are strictly optional. But we do need to tell Betamax where the
cassettes will be stored. We also need to make sure we create this directory, as
with `mkdir cassettes`.

```python
betamax.Betamax.register_serializer(pretty_json.PrettyJSONSerializer)
with betamax.Betamax.configure() as config:
    config.cassette_library_dir = 'cassettes'
    config.default_cassette_options['serialize_with'] = 'prettyjson'
```

Now we've configured the Betamax library, but we haven't yet attached it to our
tests. We do this in two steps. First, we add a `recorder` to our
`IntegrationTest` class and point it toward the `requests.Session` instance that
underlies PRAW's network code. We also require this `Session` to use the
`identity` encoding, which ensures that the traffic we record is uncompressed
(hence, human-readable).

```python
http = self.reddit._core._requestor._http
http.headers['Accept-Encoding'] = 'identity'
self.recorder = betamax.Betamax(http,
                                cassette_library_dir='cassettes')
```

Our second and final step is simply to add a context declaration to each test,
specifying the file in which to store the cassettes for that test:

```python
def test_check_comment__valid(self):
    comment = self.reddit.comment('dhb7fxz')
    with self.recorder.use_cassette('test_check_comment__valid'):
        assert self.bot.check_comment(comment)
```

Because requests are only performed when we actually access a PRAW object's
attributes,[^2] we only need to add this context for the final line.

Now, if we run our tests once, the requests are recorded inside the `cassettes/`
directory. And subsequent runs are much faster:

```
============================= test session starts =============================
platform win32 -- Python 3.6.1, pytest-3.0.7, py-1.4.33, pluggy-0.4.0
rootdir: C:\Users\levim\src\bigbenbot, inifile:
plugins: betamax-0.8.0
collected 4 items

test_bot.py ....

========================== 4 passed in 3.75 seconds ===========================
(bigbenbot) PS C:\Users\levim\src\bigbenbot> pytest
============================= test session starts =============================
platform win32 -- Python 3.6.1, pytest-3.0.7, py-1.4.33, pluggy-0.4.0
rootdir: C:\Users\levim\src\bigbenbot, inifile:
plugins: betamax-0.8.0
collected 4 items

test_bot.py ....

========================== 4 passed in 0.22 seconds ===========================
```

Not a bad speedup. By the way, in the case of PRAW and its 1436 tests, the total
running time
is [about 17 seconds](https://travis-ci.org/praw-dev/praw/jobs/223832473#L792).

# The solution, part 2: security #

However, we have a security problem. Take a look at the first few lines of
`cassettes/test_check_comment__valid.json`:

```json
{
  "http_interactions": [
    {
      "recorded_at": "2017-05-09T02:33:16",
      "request": {
        "body": {
          "encoding": "utf-8",
          "string": "grant_type=password&password=a+trivial+password&username=bigbenbot"
        },
```

Not a great password, to be sure. But even if my bot had a strong password, it
wouldn't do much good if it were plastered all over GitHub. Likewise, the
cassettes record my bot's client ID, client secret, and the temporary access
token that it gets in response.

<!-- This is the client info in base64 -->
```json
{
"Authorization": [
  "Basic NkxEMXRpam56ZnEzX0E6LXBmdVVQQUVDc01iUi1sV3o2cHNnRlpuYU40"
],

...

"string": "{\"access_token\": \"Mp5vztYHAHXR_C-lGQBxsQafbrw\", \"token_type\": \"bearer\", \"expires_in\": 3600, \"scope\": \"*\"}"
}
```

A first line of defense is to use a separate account for testing. But we'd still
not like to have these accounts be susceptible to hijacking.

To protect sensitive information, we can add a *placeholder* that will replace a
given string when it's found in a request. For example, we can add a placeholder
that replaces a client ID like `6LD1tijnzfq3_A` with a mask like `<CLIENT_ID>`.

## Simple string replacement ##

To add a placeholder, we use the Betamax configuration's
[`define_cassette_placeholder` method](http://betamax.readthedocs.io/en/latest/api.html#betamax.configure.Configuration.define_cassette_placeholder).
We just need to provide the string that we want to filter and the placeholder
that will replace it. In this case, we can get the strings we want to replace by
checking the `Reddit` object's configuration after it's loaded[^3]:

```python
# (in IntegrationTest.setup)

# Prepare placeholders for sensitive information.
placeholders = {
    attr: getattr(self.reddit.config, attr)
    for attr in "client_id client_secret username password".split()}

# Add the placeholders.
with betamax.Betamax.configure() as config:
    for key, value in placeholders.items():
        config.define_cassette_placeholder('<{}>'.format(key.upper()),
                                            value)
```

To re-record the tests with these placeholders, simply delete the JSON files in
`cassettes/` and run `pytest` again. The result should look like this:

```json
{
  "http_interactions": [
    {
      "recorded_at": "2017-05-09T02:33:16",
      "request": {
        "body": {
          "encoding": "utf-8",
          "string": "grant_type=password&password=a+trivial+password&username=<USERNAME>"
        },
```

## More complex strings ##

As you can see, we've filtered the username, but we haven't yet succeeded in
filtering out the password. This is because Betamax is looking for the string
`"a trivial password"`, but the Reddit authentication scheme uses the
URL-escaped form: `"a+trivial+password"`. We can fix this by modifying
`placeholders['password']` before we pass it into the Betamax configuration. We
just need to use `urrlib`'s `quote_plus` function in order to get the
URL-escaped version of the password:

```python
from urllib.parse import quote_plus

# ...
# (in IntegrationTest.setup)

# Password is sent URL-encoded.
placeholders['password'] = quote_plus(placeholders['password'])
```

We also haven't tackled the following block from earlier:

```json
{
"Authorization": [
  "Basic NkxEMXRpam56ZnEzX0E6LXBmdVVQQUVDc01iUi1sV3o2cHNnRlpuYU40"
],
}
```

You can hardly tell just by looking at it, but this is a base-64 encoding of the
client ID and secret. Just as we filtered the password by deriving its
URL-escaped form, we can filter this string by deriving it from the client ID
and secret:

```python
from base64 import b64encode

# ...

def b64_string(input_string):
    """Return a base64 encoded string (not bytes) from input_string."""
    return b64encode(input_string.encode('utf-8')).decode('utf-8')
    
# ...
# (in IntegrationTest.setup)
    
# Client ID and secret are sent in base-64 encoding.
placeholders['basic_auth'] = b64_string(
    "{}:{}".format(placeholders['client_id'],
                   placeholders['client_secret']))
```

## Adding placeholders at recording time ##

In the previous two sections, we've been able to add placeholders by determining
the text to be replaced before we send a request. That strategy won't work if
the data we'd like to filter is only generated upon sending a request. In our
running example, we'd like to filter our access token, but we don't know what
that is until Reddit sends it to us. So, we instead have to add a filter *after*
the request is performed but *before* it's recorded to disk. This is made
possible by the Betamax
configuration's
[`before_record` method](http://betamax.readthedocs.io/en/latest/api.html#betamax.configure.Configuration.before_record).
This method takes a callback function that receives each interaction and
cassette before recording, and can make changes as needed. Here's how it's done:

```python
import json

# ...

def filter_access_token(interaction, current_cassette):
    """Add Betamax placeholder to filter access token."""
    request_uri = interaction.data['request']['uri']
    response = interaction.data['response']
    if ('api/v1/access_token' not in request_uri or
            response['status']['code'] != 200):
        return
    body = response['body']['string']
    try:
        token = json.loads(body)['access_token']
    except (KeyError, TypeError, ValueError):
        return
    current_cassette.placeholders.append(
            betamax.cassette.cassette.Placeholder(
                placeholder='<ACCESS_TOKEN>', replace=token))

betamax.Betamax.register_serializer(pretty_json.PrettyJSONSerializer)
with betamax.Betamax.configure() as config:
    config.cassette_library_dir = 'cassettes'
    config.default_cassette_options['serialize_with'] = 'prettyjson'
    config.before_record(callback=filter_access_token)
```

This callback watches for a call to the `api/v1/access_token` API endpoint. At
that point, it tries to find a new access token in the response body. Assuming
there is one, a new placeholder can be added that lasts until the cassette is
done recording a given test.

# Mocking out rate limits #

Now, let's enhance our bot's `post_time` method so that follows proper court
protocol if it's replying to a moderator:

```python
def post_time(self, parent):
    """Post the current time as a reply to the parent comment."""
    if parent.author in self.subreddit.moderator():
        template = "The current time is {}, your grace."
    else:
        template = "The current time is {}."
    body = template.format(self.get_time())
    parent.reply(body)
```

We can test as follows:

```python
def test_post_time(self):
    comment = self.reddit.comment('dhb7fxz')
    with self.recorder.use_cassette('test_post_time'):
        self.bot.post_time(comment)
```

And running the tests, before and after recording cassettes:

```
(bigbenbot) PS C:\Users\levim\src\bigbenbot\bigbenbot> pytest
============================= test session starts =============================
platform win32 -- Python 3.6.1, pytest-3.0.7, py-1.4.33, pluggy-0.4.0
rootdir: C:\Users\levim\src\bigbenbot\bigbenbot, inifile:
plugins: betamax-0.8.0
collected 5 items

test_bot.py .....

========================== 5 passed in 1.92 seconds ===========================
(bigbenbot) PS C:\Users\levim\src\bigbenbot\bigbenbot> pytest
============================= test session starts =============================
platform win32 -- Python 3.6.1, pytest-3.0.7, py-1.4.33, pluggy-0.4.0
rootdir: C:\Users\levim\src\bigbenbot\bigbenbot, inifile:
plugins: betamax-0.8.0
collected 5 items

test_bot.py .....

========================== 5 passed in 0.39 seconds ===========================
```

As expected, the second time around is fast. But it can be made even faster. If
we examine our cassettes, we'll find that the first four cassettes we built all
have two requests: one to retrieve an access token, and one to make an actual
API call. Our new cassette, however, makes three API calls after receiving an
access token: one to fetch a comment and determine its author, one to check if
the author is a moderator of the subreddit, and one to post a reply. Because
PRAW delays consecutive requests in order to comply with Reddit's API rate
limits, that means that our tests are also being artificially delayed, even when
we're just reading from a cassette.

We can skip the rate-limiting by stubbing out `time.sleep` so that it
immediately returns when PRAW calls it:

```python
import time

# ...

time.sleep = lambda x: None
```

```
(bigbenbot) PS C:\Users\levim\src\bigbenbot> pytest
============================= test session starts =============================
platform win32 -- Python 3.6.1, pytest-3.0.7, py-1.4.33, pluggy-0.4.0
rootdir: C:\Users\levim\src\bigbenbot, inifile:
plugins: betamax-0.8.0
collected 5 items

test_bot.py .....

========================== 5 passed in 0.15 seconds ===========================
```

The caveat here is that we could theoretically run afoul of Reddit's rate limits
by making lots and lots of requests very quickly. However, so long as we add a
few tests at a time, generating cassettes as we go, we should be safe, since
most requests in a given run of the test suite will never actually reach
Reddit's servers.

# Other points of interest #

  * If you want to be especially security-conscious, you should be wary of
    feeding untrusted data into your tests. Otherwise, secrets could be exposed
    by fishing for placeholders. For example, suppose Mallory wants to
    compromise your test account's password, and she knows that you write a lot
    of integration tests that pull comments from /r/all. Then she can start
    posting lists of potential passwords as comments. If she gets lucky, and one
    of her lists makes it into your cassettes, then she can figure out your
    password by seeing which line has been replaced by a placeholder.

    Perhaps this is only a remote possibility. However, if it concerns you, here
    are two ways to mitigate. First, make sure where possible that your tests
    interact only with a private test subreddit where data can only be
    manipulated by people you trust. Second, if you do have untrusted data,
    search for your placeholders to make sure that they don't show up where they
    aren't expected.

  * Besides security, placeholders are one way of reusing cassettes even when
    part of the request data can change. For example, PRAW has code from many
    different contributors. Without placeholders, it would be necessary to share
    one testing account; otherwise, if Alice recorded a test using her
    credentials, Betamax would complain that it can't find a matching cassette
    when Bob uses his credentials. But if usernames, passwords, and the like are
    filtered, then Betamax can match on the filtered versions, while
    re-inserting the tester's credentials when the cassette is read.
    
    This approach can be extended beyond login information, which has been the
    focus of this post. In the case of PRAW's test suite, a test subreddit is
    provided as an environment variable and filtered out of cassettes, which
    allows contributors to run tests without needing access to any particular
    subreddit.

[^1]: One to authenticate, and one to fetch a comment.

[^2]: For more on PRAW's use of lazy objects,
    see
    [PRAW's documentation](http://praw.readthedocs.io/en/latest/getting_started/quick_start.html#determine-available-attributes-of-an-object).

[^3]: This example makes use of some light metaprogramming. The first block of
    code is equivalent to
    
    ```python
    placeholders = {
        'client_id': self.reddit.config.client_id,
        'client_secret': self.reddit.config.client_secret,
        'username': self.reddit.config.username,
        'password': self.reddit.config.password,
    }
    ```
