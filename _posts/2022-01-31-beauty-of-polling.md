
# The Beauty of Polling

Consider this situation. You make an HTTP request to a web server to do something expensive. It could:

1. Keep the connection open until it's done and return the response.
2. Accept a callback URL and post the response to that URL when it's done.
3. Return an instant response that says, "when it's done, your response will be at this other URL." You poll that URL until you get the response.

**Option 1** has nice ergonomics as long as the task takes, like, a few seconds. If it takes longer, then you run into issues with open sockets to a single host, both on the client (limit of 8 in Chrome) and on the server (2048 is common, but you've gotta be careful if you open a lot more.) Also, there's not great support for updates while the connection is open. Yeah, there are plugins to support returning progress periodically, but _those_ ergonomics are not great. If you're doing that, you probably want to build a special library to wrap it.

**Option 2** will avoid the issue with open sockets, but it doesn't feel like the complexity tradeoff is worth it in a lot of cases. That is, the client needs to make a new HTTP endpoint, make it accessible to the public internet, and then try its best to avoid outages or bugs that end up dropping the response on the floor forever. The server needs to post to an untrusted HTTP server that might be down, or slow, so retrying is necessary. So now a single request might be outstanding for _days_, and we might even deliver the response multiple times. All that said, there's not really another option for _really_ long requests. If you're setting up a cron job or delivering a response that takes hours, this is still probably the best option, especially if you've got a good off-the-shelf webhooks solution.

**Option 3** doesn't get much love, but I think it's great. The ergonomics are not as good as Option 1, but they're close. You just write a loop to poll, time out if it takes too long, error if it reports an error. It avoids the open socket issue because the request/response cycle for this asset _should be_ incredibly fast and cheap. And crucially, it avoids the postback requirement. Without the postback, the server doesn't need to implement complex retry logic. The client doesn't need to add routes and signature verification to accept the postback. The client doesn't need to open its endpoints in dev or staging to the public internet. There's no possibility of missed delivery or dropped delivery. And you can use the response URL to include metadata about the job, including progress updates, partial results, and errors.

Of course, if you wanted to do that for really long waits, you'd need to set up some recursive background jobs, which makes it less appealing. And there's a latency tradeoff; you won't get the response _as fast_ as option 1. But if it's a difference of a request that takes 10.1 seconds or 11 seconds, does the user care at that point?

**Overall**, I guess I've made a case for using Option 3 for those middle cases. If you've got a request that's slow, but not so slow that the user won't wait around, you can do the whole client piece in the browser and avoid tons of complexity.

Polling gets a bad wrap. Use it, and you trade a little latency and efficiency for a much simpler and robust system that scales effortlessly. Often that can be a great tradeoff.