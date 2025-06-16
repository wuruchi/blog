+++
date = '2025-06-16T10:28:30+02:00'
draft = false
title = 'The Mystery of the Random Test Failures'
+++

What happens when your test suite starts to throw some random unexpected results only when running locally that seem to increase or decrease depending on random circumstances? Some days you get higher chances of error, other days you have to try many times to get one. It seems that if you want to look at them they don't show up, but when you need to actually test your code, there they are. Is it perhaps your lack of care for **proper mocking** and **cleaning up** after each test? It might be all those **objects defined at the test file level** that are then mutated inside tests. No, it might be the **debugger** you are attaching to the terminal. Is it the _node version_? Or the fact that you are using `nvm`. What if all the hints you get from your debugging effort tell you that the root cause is _something_ and the more you try to fix that _something_ the more it seems to break everything? Let me share with you a story about **Software Development**.

We have a `node@18` project that implements an `express` `API` where we use `jest`, `nock` and `supertest` to implement more than `200` tests. Each test file groups the tests for one of the `API`'s endpoints, if the endpoint covers many use-cases, we divide the tests into two or more files. Most of the tests are `end-to-end`, we hit the endpoint running under `supertest` and we mock the external requests that the endpoint's workflow consumes, then we assert the response and the `mocks`' call count and/or arguments. The tests are executed automatically before we **push** to our `git` respository and they are also executed as part of the `GitLab` pipeline we have defined as part of the respository. The tests are run in `isolated` (`jest`) mode. Your **typical** setup.

Some weeks ago we started to notice an **increased number of random failures when running the complete test suite locally**. Everything was fine when running them as part of the pipeline, i.e. inside a container. The failures were in the form of `500` responses from the `API` endpoints running under `supertest`, after inspecting the logs, the `500` responses **appeared** to be caused by **missing mocks** or **mocks returning wrong data** that was set up for another test case. It used to happen very rarely before, when we had less tests. After some observation, I noticed that the larger the test file, the higher the chances of getting a random error while running the tests on that file. Moreover, **when one test failed, it also affected other tests**. However, there were times where the tests ran fine most of the time, locally. **Puzzling**, it was.

I have to admit that most the test files **were not set up carefully**. There was some shared state, `nock` mocks were not set up as cautiously as recommended by their documentation, some objects where defined at the `file` level and then shared between `describe` sections where some `test` cases could **mutate** them. There was some mock cleanup `afterEach` test but it could be improved, some operations were not mocked and they actually hit real services but it was not noticed because the response of those services was not consumed as part of the workflow. Nonetheless, **they used to work Ok before**, **what had changed?** What was the **breaking change** and when was it introduced? Was there a breaking change? If yes, then why it only affects **local execution**?

Another curious observation was that if I run **one test file at the time**, the chances of getting an error decreased. That raised the suspicion that there might some kind of shared state that was causing race conditions or clashes between test files if they run concurrently. In consequence, I added the `--runInBand` `jest` flag in an attempt to **decrease the probability** of "clashes" between test files. It seemed to help, there were **less random errors**, but still there was no certainty about this shared state, _what was it?_ After careful inspection of the code base, I couldn't find anything. I thought it was _Ok as long as the random failures happened rarely_. Some time later, even when using the `--runInBand` flag, test failures happened often.

**On the internet** it was heavily suggested that the reason of the random failures in `jest` tests was some mismanagement on how mocks or shared objects are set up in the tests definitions. It was also suggested that our code introduced some **race conditions** or, generally speaking, it had some **loose ends**, e.g. a non-awaited async call. The `AI LLM`s had similar suggestions, which is completely expected because that's usually what happens in these sort of situations. Anyhow, I'd dare say that the `AI LLM` proved to be a little **misleading** because it insisted and closed itself into a set of possible causes; while searching on the internet and looking at other humans errors, suggestions, observations, etc. proved to at least hint into another root cause for the issue.

I decided to implement all the `jest` and `nock` **good practices** that I could find and that seemed that could help on this kind of issue. The start of the body of the main `describe` block on each test file looks like:

```typescript
    beforeAll(() => {
      if (!nock.isActive()) nock.activate();
      nock.disableNetConnect();
      nock.enableNetConnect("127.0.0.1");
    });

    beforeEach(() => {
        jest.clearAllMocks();
        jest.resetModules();
        nock.cleanAll();
        pdwAccessMock = nock("https://access.pdw.company.io");
        amundsenMock = nock(
            "https://amundsen.discovery.company.io"
        );
        accessMock = nock("https://api.company.io");
        oauthMock = nock("https://oauth.company.io");
        contractMock = nock("https://contracts.company.io/v1");
        streamMock = nock("https://stream.company.io/v2");
    });

    afterEach(async () => {
        // Uncomment the following lines if you want to check for pending mocks
        try {
            expect(nock.isDone()).toBeTruthy(); // Ensure all mocks were used
        } catch (err) {
            const currentTestName = expect.getState().currentTestName; // Get the current test name
            // eslint-disable-next-line no-console
            console.error(`Test failed in afterEach: ${currentTestName}`);
            // eslint-disable-next-line no-console
            console.error('Pending mocks:', nock.pendingMocks());
            throw err;
        }
        nock.abortPendingRequests();
        nock.cleanAll(); // Clean up all mocks after each test
        jest.clearAllMocks();
        jest.resetModules();
    })

    afterAll(async () => {
        await new Promise(resolve => setTimeout(resolve, 1500)); // avoid jest open handle error
        nock.restore();
    });
```

Where the `mock` objects are initialized at the beginning of the file, for example `let oauthMock = null`, don't mind the names. Notice `nock.enableNetConnect("127.0.0.1");` to enable connection to the `supertest` test instance. Also, notice the shared statements between `beforeEach` and `afterEach` and the block of code to **catch unused mocks**, I was a little bit _paranoid_ about mocks at this point. Then, I took some time to **fix any possible source of shared state** between `describe` sections or `tests` on each test file. If a `test` needed some object, it would get `deepClone`d copy, even if it was only to return a `GET` response. I didn't want any possible **source of doubt** with respect to our mocks and shared state.

After these changes, the number of random failures seemed to **decrease but it was not reduced to zero**. Worse, the root cause was still unclear. However, I was fairly confident at this point that _there was no mocks or shared state source of error_. It **had to be something else**.

**At this point there were random test failures that happened more or less often depending on some random factor**. The errors manifested as `500` response from our `API` under test. The error logs mentioned missing mocks errors or mocks returning the wrong data. I decided to take a closer look at those error responses by using this function:

```typescript
export const assertWithLogging = (assertionCallback: () => void, response: any) => {
    try {
        assertionCallback();
    } catch (error) {
        // eslint-disable-next-line no-console
        console.error('Test failed. Response details:', {
            statusCode: response.statusCode,
            body: response.body,
            headers: response.headers,
        },
        response);
        throw error; // Re-throw the error to ensure the test fails
    }
};
```

Usage example:

```typescript
test("Fails if support email is not valid", async () => {
    const response = await request(app)
        .post("/v0/dataproducts")
        .set('authorization', `Bearer ${someToken}`)
        .send(Object.assign({}, cloneDeep(minDataProductPayload), { support: { email: 'foobar' } }));
    assertWithLogging(() => {
        expect(response.statusCode).toBe(400);
        expect(response.body.message).toBe('request.body.support.email should match format "email"');
    }, response);
});
```

If one of the assertion fails, we print the full response object. When one of these random `500` responses happened, `response.body` was empty: `{}`. By looking at the response contents I found the `text` property with value **'This is a proxy server. Does not respond to non-proxy requests.\n'**. Now that was _even more puzzling_ but at least it hinted into a different direction: The test was not hitting the `API` `supertest` implementation, **it was hitting something** else which was in turn returning that `500` response. Moreover, one of the other properties of the `response` object revealed that the request was sent to the IP `127.0.0.1:59475`. Notice the `port` number.

During my internet searching I ran into a response to the topic [Node+supertest flakes with "Client network socket disconnected before secure TLS connection was established"](https://stackoverflow.com/questions/63343123/nodesupertest-flakes-with-client-network-socket-disconnected-before-secure-tls/63343124#63343124) which led to [Ephemereal porst and SO_REUSEADDR](https://gavv.net/articles/ephemeral-port-reuse/). That gave me some intuition into what could be happening. _That and my notions of networking_. Then, I decided to look at **what was exactly running on that port**:

```bash
sudo lsof -i :59475
```

Returned:

```bash
COMMAND     PID          USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
snyk-maco 73241 wilmer.uruchi  180u  IPv4 0x1a64fb275cdfc9f7      0t0  TCP localhost:59475 (LISTEN)
```

__Hmmm__... `snyk`, that's a familiar name. The complete name of that process is **snyk-macos-arm64**. If I killed the process, a replacement was spawned. There were `6` processes with the same name in my computer and that number **corresponded exactly** with... the number of `VSCode` instances running at that time in my computer (I do some multi-tasking). That was Ok, I installed the `snyk` extension for `VSCode` because that's what we use and it is useful in the security sense. I proceeded to uninstall the `extension`, temporarily, from `VSCode` and restart it. Then, I ran the `tests`. **No errors, flawless**. I even removed `--runInBand` from the command and suprised me not: **no errors**.

From what I understand, the sockets created by `supertest` use the `SO_REUSEADDR` flag and the `Snyk` extension is doing the same which means that there is a chance that both could end up **consuming the same port**. That's the reason I got the random `500` responses from `supertest`, it was actually **the snyk process who was responding**. That also explains why the probability of these errors appeared to increase or decrease at random times, it depended on the number of `VSCode` instances I had running at that time. It also explains why there was a higher chance of getting random errors when running files that contained large numbers of test cases. Generally speaking, **the probability of getting one of these random errors was proportional to the number of** `VSCode` **instances running at the time and, obviously, the number of tests executed.**

However, **why one of these random errors would also cause errors for other test cases in the form of missing mocks or wrong mock data?** Here I don't have a clear answer. I know for certain that since the request didn't reach the test endpoint, all the mocks set for that test where not consumed, but they should've been removed by the `beforeEach` or `afterEach` sections. Perhaps that `instance` of `supertest` **that was not consumed needed to be also explicitly cleaned** `afterEach` test and because it was not cleaned, the next test was instead consuming the incorrect instance. I guess that sounds like the most plausible explanation. I must stress that **the error propagation caused by these random errors was the most misleading part of debugging this whole situation**, it was actively taking me into the wrong direction.

One more note about the `LLM` models. Although they proved to be somewhat misleading on this sceneario, they provided tools that helped me debug and discard possible root causes. Also, this is an unusual case, and probably it is the type of data that is treated as an outlier and not included in the training data. As a regular user, if I mention `node`, `express`, `jest`, `supertest`, `nock` and "random test errors", the first and most likely response is that there is a problem with my test setup.

Now that the root cause is clear, I can start **thinking about a solution**. Considering that the `Snyk` extension for `VSCode` has to stay, I will have to re-think how the `supertest` library is set up. From a quick look on the internet, there doesn't appear to be a straightforward way to force `supertest` to consume a pre-defined port (or range of ports) for testing. On the other hand, I am sure there are other libraries that serve the same purpose as `supertest` and that might provide the functionality that we need or at least that do not cause the port collisions. I will leave that solution for another article.

It has been a frustrating but instructing experience. I have added another dimension to the way I debug.




