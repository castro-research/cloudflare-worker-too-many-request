# Introduction
A detailed account of how I solved a problem with an API that used the Cloudflare Browser Rendering Worker service.

TLDR: We have a Ruby on Rails API that receives a request with a body, evaluates it, and passes it to Cloudflare's Worker to generate a PDF. This API suddenly stopped working.

# Start
In the middle of the holidays, I received a notification:

<img src="https://github.com/user-attachments/assets/9e7219b4-3e9f-4627-a68b-1ecbf936590b" />

My first reaction: Let's check Sentry to see what's happening.

I saw 170 attempts to generate a page with Puppeteer.

<img src="https://github.com/user-attachments/assets/dd7d5cb1-3439-4e50-b6cc-60e62ef2bbe3" />

The first problem was that, since I was using TypeScript, I had the transpiled version but without [source maps](https://developers.cloudflare.com/workers/observability/source-maps/) (which are still in BETA), and the upload_source_maps flag was not active. (I think it should be default for new TypeScript projects or at least identified somehow.)

Without my MacBook at hand, I looked at the logs on my phone (which didn't say anything relevant) since I didn't have the stack trace or where the problem occurred.

# What I Thought Happened (In My Head)
I received many requests, it tried to scale to more workers than it should, resulting in an Out of Memory error, and stopped receiving requests for a while, also giving a 503 error.

# What I Assumed:
Since the paid plan limit is 500 workers, for each request, a Worker opens a headless browser, does what it needs to do, closes the browser, and ends.

# Discovering the Problem

With my MacBook in hand, knowing I didn't have a stack trace or logs, I converted the code from TypeScript to a JS Module to avoid dealing with source maps still in Beta.

Since the project has integration tests, I ran them, and everything passed.

I deployed a remote preview version, wrote a separate script to make 10 simultaneous requests (via callbacks), and voilà, error 429 appeared. Without transpiling, debugging became much easier.

I went to the documentation to look for [Rate limit](https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/), which mentioned the same thing in the plans section, and I could even edit rate limits based on the user, location, etc...

Then I thought there might be some Cloudflare security aspect where I didn't have access.

Navigating through the documentation, I found that the [Browser Rendering limit](https://developers.cloudflare.com/browser-rendering/platform/limits/) is different from the Worker limit.

>> Two new browsers per minute per account.
>> Two concurrent browsers per account

Eureka! I couldn't have more than 2 browser sessions at the same time.

# Digging Deeper

It wasn't just that; the information was scattered in small pieces of best practices. So, before diving into the code, the task was to piece these together:

1 - The new browser instance should be within a try/catch to avoid lost sessions/crashes.

I had already done this.

2 - Omitting browser.close was necessary to session `keep-alive`, allowing more requests to be handled by different workers without opening a new instance.

I had the opposite understanding, always closing it, so I never handled more than 2 requests at once.

3 - You have access to the workers of browser sessions.

I didn't know this either, and could check if one was already open and make a `connect` instead of launch. So I can implement something like a `round-robin` strategy

4 - You should close the browser WHEN an error occurs, but only `disconnect` the session if you can do what you need to do, freeing it up for another request.

5 - You can have a recursive function to retry the browser connection.

Up to point 4, you solve part of the problem of keeping 2 sessions open and getting stuck with `Too Many Requests` for a while, then blocking again after another request. But you'd still encounter some 429 errors because sometimes you'd have more concurrent requests, and the Workers would be busy.

This particular implementation isn't in the documentation and could be implemented by Rails. However, I implemented a simple retry where it would try to find another available browser, only returning an error when the worker truly couldn't perform the task or timed out.

Why didn't we encounter this problem before?

Because we were still testing a feature, and it was never necessary to make more than a few requests in a short time.

# Conclusion

I implemented a test that made several simultaneous requests, and all returned status 200:

<img src="https://github.com/user-attachments/assets/905fe43b-0e41-48c4-a20f-5d8fc17e40f0" />

# Other challenges

Since we have 2 workers limited to browser rendering, the content of the HTML matters a lot, as if it is too large, it will take longer to process, and then the timeout will be inevitable.
