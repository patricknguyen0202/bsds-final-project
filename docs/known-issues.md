# Known Issues

## 429 errors during a large join

**Problem.** The Directory API stage throttle is about 2,000 requests per second. If 30,000 players call `GET /games/{pin}` at once, requests over the limit return `429 THROTTLED`. At that rate the whole burst takes about 15 seconds to get through.

**Mitigation (decided with Jared).** The player client shows a loading state and retries on 429. After a few failed retries it returns an error to the player.

**For the experiments (decided with Quang).** Keep the throttle at 2,000 req/s for the 10,000 to 30,000 player join tests and measure:
- joining time
- percent of players who saw a 429 at least once
- percent of players who could not join (still failing after the retries)

These results depend on the retry settings (number of retries, delay), so record them with each run.
