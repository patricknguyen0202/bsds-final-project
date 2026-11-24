# Known Issues

## 429 errors during a large join

**Problem.** The Directory API stage throttle is about 2,000 requests per second. If 30,000 players call `GET /games/{pin}` at once, requests over the limit return `429 THROTTLED`. At that rate the whole burst takes about 15 seconds to get through.

**Mitigation (decided with Jared).** The player client shows a loading state and retries on 429. After a few failed retries it returns an error to the player.

**For the experiments.** Join time and join failure rate in the 10,000 to 30,000 player tests depend on the throttle and on the retry settings (number of retries, delay). Record both with each result. Decide with Quang whether those runs keep the 2,000 req/s throttle or raise it.
