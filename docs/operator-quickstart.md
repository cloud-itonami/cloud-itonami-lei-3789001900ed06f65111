# Operator quickstart — cloud-itonami-lei-3789001900ed06f65111

This repository is a read-only archive (see `README.md`): Eskom Holdings SOC Ltd's
published Website Terms and Conditions, plus GLEIF registry facts about the
company. It has no service to start. An operator's job here is to **check
that the archive still agrees with its sources**. The steps below do that.

Every command and every output quoted here was run on 2026-09-25 against
commit `6706650`, with Node v26.7.0 and `kbb` from `kotoba-lang/kotoba/bin/kbb`.

## Prerequisites

- `kbb`, the workspace script host (`orgs/kotoba-lang/kotoba/bin/kbb` in the
  `com-junkawasaki` superproject). Nothing else: `scripts/verify-facts.cljk` is a
  vendored, dependency-free copy, so a plain `git clone` of this repo is enough.
- Outbound HTTPS to `api.gleif.org` (step 2) and `www.eskom.co.za` (step 4).

All commands run from the repository root.

## 1. Read what is recorded

```bash
kbb --backend sci -e '(require (quote [clojure.edn :as edn]) (quote ["fs" :as fs]))
(let [f (edn/read-string (fs/readFileSync "facts.edn" "utf8"))]
  (doseq [x f] (prn (:fact/id x) (:source/url x))))'
```

`facts.edn` is tx-data: one map per registry page that was fetched, each carrying
`:source/url`, `:source/http-status` and `:source/retrieved-at`. It is generated
(see the header). Do not edit it by hand.

## 2. Check the facts against the live registry

```bash
kbb --backend sci scripts/verify-facts.cljk; echo "exit=$?"
```

Output on 2026-09-25:

```
CHECKED	11
ENTITIES	9
NO-PARENT	https://api.gleif.org/api/v1/lei-records/3789001900ED06F65111/direct-parent	404 -- GLEIF publishes the other side of this pair for this entity
NO-PARENT	https://api.gleif.org/api/v1/lei-records/3789001900ED06F65111/ultimate-parent	404 -- GLEIF publishes the other side of this pair for this entity
OK	all 9 recorded fact(s) still match the live sources
exit=0
```

The two `NO-PARENT` lines are expected. For each consolidation level GLEIF publishes
either a parent or a reporting exception, and returns 404 for the other one.

The script has three exit codes. **Read the code, not just the last line:**

| exit | meaning | what to do |
|---|---|---|
| `0` | every cited URL answered and every recorded fact still matches | nothing |
| `1` | a citation broke, or a fact **drifted** from the live record | read the `DRIFT` lines (step 3) |
| `3` | the check **could not run**, e.g. `facts.edn` missing or empty. This is not a pass | restore `facts.edn` from git; do not treat it as green |

To see what exit 1 looks like, break a copy (not the repository):

```bash
S=$(mktemp -d) && cp -R . "$S" && cd "$S"
sed -i '' 's|"2002/015527/30"|"2002/015527/31"|' facts.edn   # GNU sed: sed -i
kbb --backend sci scripts/verify-facts.cljk; echo "exit=$?"
```

```
DRIFT	gleif-lei-record	:company/registered-as
  recorded: "2002/015527/31"
  live:     "2002/015527/30"
...
exit=1
```

Removing `facts.edn` from the copy gives `exit=3` with
`INCONCLUSIVE facts.edn is missing or holds no facts -- ... Refusing to report a pass.`

## 3. When the registry really changed (exit 1 on an unmodified checkout)

GLEIF records change legitimately. As of 2026-09-25 the recorded
`:registration/status` is `"LAPSED"`, and a renewal would change it. When a
`DRIFT` line shows a real change upstream, regenerate the file. Do not patch it:

```bash
kbb --backend sci scripts/verify-facts.cljk --write
git diff facts.edn        # review: every changed value should match a DRIFT line
```

Do not edit `scripts/verify-facts.cljk` in this repository. It is a vendored copy
of `scripts/lei-verify-facts.cljs` in `com-junkawasaki/root`. Fix the canonical
there and re-vendor it. The superproject's `scripts/verify-vendored-copies.cljs`
reports an edit made only here as a fork.

## 4. Check the archived Terms and Conditions text

The text lives in `80-data/public/tos.journal.edn` as quads under entity
`"eskom-tos-1"`: `:tos/full-text`, `:tos/source-url`, `:tos/retrieved-at`,
`:tos/sha256`, `:tos/doc-type`. `:tos/sha256` is the SHA-256 of the stored
**text**. To confirm the text has not changed since it was archived:

```bash
kbb --backend sci -e '(require (quote [clojure.edn :as edn]) (quote ["fs" :as fs]) (quote ["crypto" :as c]))
(let [m (into {} (map (fn [[_ a v]] [a v]))
              (edn/read-string (fs/readFileSync "80-data/public/tos.journal.edn" "utf8")))]
  (prn :recorded (:tos/sha256 m)
       :computed (.digest (.update (.createHash c "sha256") (:tos/full-text m) "utf8") "hex")))'
```

The two values must be equal. On 2026-09-25 both were
`6c0dd99af41c77614ab65a36f50edceec264ef7890ae2bbcd9c3eed157bdfde1`.

To check that the source is still published:

```bash
curl -sS -o /dev/null -w '%{http_code} %{content_type}\n' -L \
  'https://www.eskom.co.za/wp-content/uploads/2021/10/WEBSITE-TERMS-AND-CONDITIONS_Sep2021.pdf'
```

On 2026-09-25 this printed `200 application/pdf`.

⚠ **Do not compare the hash against a fresh PDF-to-text extraction.** The hash is
over the text as it was extracted in July 2026, with whatever extractor was used
then. `pdftotext` (with or without `-layout`) on the same, still-published PDF gives
a different byte count (24,075 / 21,706 vs 24,005 characters stored) and a different
hash. A mismatch there says nothing about whether the archive or the source changed.
To see whether the *source* changed, compare the live PDF's own bytes over time.
This repository does not record them yet.

## What this repository does not do

- It is not a governed Advisor/Governor actor, and it runs nothing on a schedule.
  Nothing re-verifies it unless an operator (or a loop) runs step 2.
- It does not publish the facts to the shared query plane (see the `facts.edn`
  header).
