# @snrgy/api-perf

Reliable, unbiased HTTP API performance measurement with transparent methodology.

## Why this tool exists

The same philosophy as `web-speed-test` — but for HTTP APIs, without browser overhead. API performance testing tools tend to show "requests per second" without reporting the full statistical distribution, hide configuration details that affect results, or present scores instead of raw numbers.

This tool measures raw HTTP performance metrics with the same statistical rigor as `web-speed-test`: median, P10/P90, CV%, reliability ratings, cold-start analysis, and auto-generated methodology sections.

## Status

Not yet implemented. Coming soon.

## What will be measured

- TTFB (Time to First Byte)
- Total response time
- Response size (compressed and uncompressed)
- Status code consistency across runs
- Per-run statistical distribution (median, P10/P90, CV%, reliability rating)
- Cold-start vs. warm-run separation
- Network calibration pre-test

## Philosophy

- Raw metrics only — no scores, no grades
- Statistical rigor: every metric has a reliability rating
- Transparent methodology auto-generated in every report
- Honest about limitations (single geographic location, no real user simulation)

## License

MIT
