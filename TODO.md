# dns-filter-bench — handover

Empty repo. This file is the brief for building it.

The work exists already as a one-off, run by hand: see
[bancuh-dns/COMPARISON.md](https://github.com/ragibkl/bancuh-dns/blob/master/COMPARISON.md).
The job here is to turn that into a harness anyone can run and CI can execute,
and to fix the fact that the original was measured by hand and had several
errors that were only caught by re-checking.

## What this benchmarks

Four DNS filtering engines given the same blocklist, on constrained hardware:

- **bancuh-dns** — Rust, RocksDB on disk
- **Pi-hole** — C (dnsmasq fork), SQLite on disk
- **Blocky** — Go, in memory
- **AdGuard Home** — Go, in memory

The interesting range is **large blocklists**: around 8 million entries, where
the engines diverge sharply. Every published comparison uses household-scale
lists of a few hundred thousand, where they all look roughly the same.

## Declared scope — read this before adding a "winner" column

These measurements reflect **one specific use case**, and the results change
completely if your constraints differ. State this prominently in the README;
an undeclared frame is what makes vendor benchmarks worthless.

| assumption | what it stops measuring |
|---|---|
| public, multi-tenant resolver | per-client policy, DHCP, LAN device identity |
| hands-off, auto-updating | dashboard UX, curation workflow, install experience |
| 1 GB RAM, 1 vCPU | the biggest lever — at 4 GB the memory column barely matters |
| privacy-focused, retains nothing | anything whose value *is* retained query history |
| list contains ~3.18M `*.domain` wildcards | the only reason list format matters |

**The fourth assumption is unfair to Pi-hole and the README must say so.**
Pi-hole gives up ~88% of its throughput to per-query logging, but that logging
*is* the product — its dashboards are why people run it. Measuring it with
logging disabled measures it doing something it was never designed for.

Report raw figures — throughput with and without logging, memory, accepted
entry counts, feature support — and **do not compute a composite score.**
Someone with different priorities should be able to re-weigh the same data.

## What to build

1. **A harness** taking the blocklist and engine set as inputs, so it can run
   with a 100k household list as easily as an 8M one.
2. **Per-engine adapters.** Each needs different config, a different list
   format, and a different way to report what it loaded. See the pitfalls below.
3. **CI.** The 8M run peaked around 6 GB and took over an hour of wall time,
   which will not fit a standard GitHub runner. Suggested split: a small list
   (~100k) on every push to prove the harness works, and the full run as a
   manual or self-hosted job. **Decide this early — it shapes everything.**
4. **Result output** as machine-readable JSON plus a rendered table, so results
   can be diffed across runs and engine versions.

## Pitfalls

Every one of these was hit during the original run. Most produce
plausible-looking numbers that are wrong, which is the dangerous kind.

**Measurement**

- `dig` reports whole milliseconds. Every engine answers in well under 1 ms on
  loopback, so `dig` cannot resolve the difference. Use `dnsperf`
  (`dnf install dnsperf`), not a shell loop.
- Pin every engine to the same CPU and memory (`--cpus`, `--memory`) or you are
  benchmarking the host, not the target hardware.
- Use Docker **host networking for all engines**. Bridge networking adds NAT
  overhead and penalises whichever engine gets it.
- Take RSS *after* the process settles. bancuh-dns reads 108 MB immediately
  after its compile and 41 MB once RocksDB releases memtables.
- Host memory pressure can OOM-kill a container mid-run and the result still
  looks like a bad score. Check `.State.OOMKilled` and exit code 137 after every
  measurement.

**Things that silently measure the wrong thing**

- **Rate limits.** AdGuard Home defaults to `ratelimit: 20` and Pi-hole to
  ~1000/60s. Saturating them measures the rate limiter. Disable both.
- **Logging defaults.** This was the largest single effect found — larger than
  any architectural difference. Pi-hole −88%, Blocky −30%, AdGuard −16%.
  Measure both with and without, and never compare engines at their defaults
  while claiming to compare engines.
- **Filter download caches.** Pi-hole and AdGuard Home cache lists on disk. A
  stale cached filter produced a completely wrong AdGuard result in the original
  run. **Verify the downloaded byte count matches the list you meant to serve.**
  Note that a container-written directory may resist `sudo rm -rf` under
  distrobox/rootless Docker — use a fresh directory per run instead.
- **Pi-hole's FTL caches the blocking decision per domain.** Repeating one
  domain does not re-evaluate rules, so regex cost measures as zero. Use unique
  query names when measuring anything rule-dependent.

**Semantics — the part most likely to produce an unfair comparison**

- **The engines accept different subsets of the same list.** Report what each
  actually loaded, not what you submitted. From 7,806,090 source entries:
  Pi-hole 4,630,065 (rejects `*` outright, silently), Blocky 6,117,020 (rejects
  ~1.7M entries that are not valid domains, logging each), AdGuard 4,632,419
  plus 3,173,671 rules when given its own syntax.
- **List format is not neutral.** AdGuard Home cannot act on `*.domain` in a
  hosts file but handles `||domain^` natively. Testing it only with hosts format
  wrongly records it as unable to do wildcards. Give each engine a list written
  for it, and report which format was used.
- **None of them do implicit suffix matching.** Verify with an entry that has no
  `*.` twin: all four block the exact name and return NXDOMAIN for a subdomain.
  Wildcard coverage comes from the list, never for free.
- **Distinguish blocked from nonexistent.** Blocked is `NOERROR` + `0.0.0.0`;
  unblocked-and-dead is `NXDOMAIN`. Both look like an empty answer to
  `dig +short`. Check the status code.

## Baseline to reproduce

From the original run: 7,806,090 entries (4.63M plain + 3.18M `*.domain`), 1
vCPU, host networking, logging and telemetry disabled everywhere.

| engine | format | accepted | wildcards | resident | qps |
|---|---|---|---|---|---|
| bancuh-dns | `domain` + `*.domain` | 7.81M submitted | yes | 41 MB | 40,323 |
| Pi-hole 6.4.3 | hosts | 4,630,065 | no | 10 MB | 54,680 |
| AdGuard Home | hosts, plain only | 4,632,419 | no | 646 MB | 12,916 |
| AdGuard Home | hosts + `\|\|domain^` | 4.63M + 3,173,671 | yes | 1025 MB | 7,574 |
| Blocky 0.34.0 | `domain` + `*.domain` | 6,117,020 | yes | 602 MB | 4,758 |

If the harness reproduces roughly these, it is working. Treat them as a sanity
check rather than as truth — they were measured once each, by hand, on a shared
desktop, and the point of this repo is to replace them with something repeatable.

Also worth reproducing: Pi-hole's regex table as an alternative to wildcards
scales badly enough to be worth documenting — 100k patterns cost +97 ms per
uncached query and 359 MB; 300k cost +2.35 s and 1054 MB.

## Source data

The list came from the 71 blocklist sources in
[adblock-dns-server/data/configuration.yaml](https://github.com/ragibkl/adblock-dns-server/blob/master/data/configuration.yaml).
The harness should fetch and compile these itself rather than depending on a
checked-in copy, but should pin a snapshot for reproducibility — upstream lists
change daily and results are not comparable across different fetch dates.

Note the source contains entries that are not valid domains (URLs with query
strings, and similar). Do not silently clean them: how each engine handles that
input is part of the result.
