# K6 lab results

## Part 0
- Step 0.1, http_req_duration: avg=271.64ms min=271.64ms med=271.64ms max=271.64ms p(90)=271.64ms p(95)=271.64ms
- Step 0.2, requests, rate and p(95), and why the rate stays below 5:
  Requests: 80, Rate: 3.82 req/s (3.815552/s), p(95): 271.81ms.
  Why the rate stays below 5: Each iteration consists of an HTTP request (~262ms) and a 1-second pause (`sleep(1)`), making each iteration take approximately 1.26 seconds. Therefore, each virtual user can perform at most ~0.79 requests per second (1 / 1.26s). With 5 virtual users, the theoretical maximum rate is 5 * 0.79 ≈ 3.95 req/s, keeping the actual rate below 5 req/s.
- Step 0.6, exit code of the normal run and of the run with P95=300:
  Normal run exit code: 0
  P95=300 run exit code: 99
- Step 0.7, rules that passed, p(95), p(99), exit code after your change:
  Rules that passed: All 3 rules passed (`http_req_duration` `p(95)<1000`, `p(99)<1500`, and `http_req_failed` `rate<0.01`).
  Measured p(95): 386.32ms, measured p(99): 394.83ms.
  Exit code after changing requirement to fail (e.g. p(95)<300): 99.
- Step 0.8, what was wrong, and how the checks showed it:
  What was wrong: QuickPizza's API is case-sensitive and expects capitalized names for ingredients and tools (`"Pepperoni"` and `"Knife"`). The original script supplied lowercase strings (`"pepperoni"`, `"knife"`), which the API ignored, causing it to randomly generate pizzas that still included Pepperoni or required a Knife.
  How the checks showed it: Under 5 VUs over 20s (75 requests), the checks revealed that 8% of responses contained Pepperoni (69 passed, 6 failed) and 30% required a Knife (53 passed, 22 failed). A single manual request might coincidentally avoid those items, but repeated checks under load exposed the stochastic failure rate.

## Part A results
| Step | Test type | Users | Req/s | p(95) | Max | Failed | Checks |
| ---- | ------------ | ----- | ----- | ----- | --- | ------ | ------ |
| A1 | Smoke | 1 | 0.49 | 343.69ms | 345.58ms | 0.00% | 100.00% |
| A2 | Average load | 3 | 1.33 | 362.38ms | 408.21ms | 0.00% | 100.00% |
| A3 | Stress | 5 | 2.20 | 368.12ms | 572.96ms | 0.00% | 100.00% |
| A4 | Spike | 5 | 1.05 | 362.57ms | 390.94ms | 0.00% | 100.00% |
| A5 | Soak | 2 | 0.99 | 363.31ms | 1.28s | 0.00% | 100.00% |

## Part B
1. Completed in the results table above.
2. In Step 0.2, each iteration takes the network round-trip time (~262ms) plus `sleep(1)` (1000ms), totaling ~1.26 seconds per iteration. Consequently, each VU executes approximately 0.79 requests per second. Five VUs running concurrently produce 5 × 0.79 ≈ 3.95 req/s, which remains strictly below 5 requests per second.
3. The response time grew slightly from the average-load test (p(95) = 362.38ms, Max = 408.21ms) to the stress test (p(95) = 368.12ms, Max = 572.96ms). In terms of the operational envelope, QuickPizza is built to handle substantially higher loads than 5 VUs. Thus, both 3 VUs and 5 VUs operate well inside the system's operational envelope, exhibiting linear throughput scaling without resource saturation or request degradation.
4. In the spike test, response times remained well within thresholds and normalized within 1 to 2 iterations (~5-10 seconds) after dropping back from 5 VUs to 1 VU, showing rapid recovery.
5. In the soak test, the response time and error rate did not degrade over the 10-minute hold; the error rate remained 0.00% and p(95) remained steady at 363.31ms. A multi-hour soak test could reveal defects such as memory leaks, database connection pool exhaustion, file descriptor leaks, or unbounded cache growth.
6. Thresholds use p(95) because an arithmetic average hides outliers; hundreds of fast responses can mask a significant fraction of unacceptably slow requests. Percentiles like p(95) evaluate the experience of the slowest 5% of users, which directly corresponds to service-level objectives (SLOs).
7. (1) Network variability: Internet routing introduces unpredictable network hops, latency jitter, and packet transmission delays compared to a local low-latency lab network. (2) Multi-tenancy and contention: QuickPizza is a shared public server serving global users concurrently, causing variable resource availability unlike an isolated, dedicated test system in a lab.
8. Running high-concurrency stress, spike, or breakpoint tests against an external server can overwhelm server resources, causing a Denial of Service (DoS) for legitimate users and incurring significant infrastructure or bandwidth costs for the owner. It is unethical and unauthorized without explicit written consent.
9. Vague requirement: "The search feature in our music streaming app should be fast and reliable under heavy traffic."
Testable requirement: "Under a sustained load of 50 concurrent users over 5 minutes, 95% of search requests must complete within 400ms, 99% within 800ms, and fewer than 1% of requests may fail."
K6 thresholds:
```javascript
thresholds: {
  http_req_failed: ['rate<0.01'],
  http_req_duration: ['p(95)<400', 'p(99)<800'],
}
```
