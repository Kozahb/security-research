# Methodology: how I reached duplicates, and what they changed in my process

> **About this document.** Portfolio piece. It describes technique and process, not targets: no
> program, company, host, real endpoint, report number or account appears here. Route names are
> generic (`/bff/<namespace>/...`) and serve only to show the shape of the problem. Every number
> cited is a real measurement from my own work, under authorised engagement and on accounts of mine.
>
> Scope: two of my findings were closed as **duplicate** - a real bug, already known to the program.
> This document explains (a) the method that produced each one, (b) why that method led to contested
> ground, and (c) the process change that followed, with the executable procedure.

---

## 1. What a duplicate means, technically

A duplicate is not a measurement error. It is confirmation that the bug exists, plus the information
that somebody got there first. The practical consequences are different:

- the **class** was validated by the triager, so the framing was right;
- the **root cause is burned**: under the usual "several issues with the same root cause = one
  payout" rule, sibling endpoints in the same family fall with it and no longer warrant a new report;
- the cost is only time. On the platform where I report, a duplicate neither adds to nor subtracts
  from reputation.

That makes a duplicate a cheap calibration signal: it does not say the technique is bad, it says the
**surface** was predictable. The right question after a duplicate is not "what did I do wrong", it is
**"why was this piece of surface obvious to other people too"**.

---

## 2. Case 1 - authorization inconsistency between two services over the same object

### The shape of the bug

An object identifier was read by two different services. The service that **owns** the feature
returned **404** for an id belonging to another account, correctly. A second service, handling bulk
metadata, returned **200** with the metadata for that same object, for the same session, without
checking the relationship. The inconsistency is unarguable precisely because the target demonstrates
the correct behaviour on one path and the incorrect one on another, for the same id.

### How it was found

1. **Inventory endpoints by identifier type, not by feature.** I grouped routes by the *type of id*
   they accept. If two endpoints accept the same type of id, they are comparable, even when they
   belong to different products.
2. **Replay the denied id.** I took an id that one endpoint rejected with 404 and replayed that same
   id against every other endpoint in the inventory that accepts that type. The 404 from one serves
   as a **baseline of correct behaviour**: any 200 from another service is a measured deviation, not
   an assumption.
3. **Enumerate by body parameter, not by path.** The vulnerable endpoint was a `:bulk`. There was a
   single path with that suffix, but the body parameters (lists of item ids) revealed the real
   family. Enumerating by the **shape of the body** found siblings that enumerating the path did not.
4. **Measure amplification, do not claim it.** The parameter accepted a comma-separated list, so one
   request tests many ids. That turns a point leak into a scanning primitive. I measured how many ids
   per request were accepted before writing the word "enumeration" in the report.

### Why it duplicated

A bulk endpoint taking an array of ids is **the first thing** anyone tests on a BFF: it advertises
IDOR in its own signature. The finding depended on careful observation, but the surface was among the
most visited on that target.

### A limit I documented myself before submitting

The leaked data was weak: a UUID and a user id, and that program lists user id as non-confidential. I
wrote this in my internal analysis **before** submitting, and ordered the report queue from strongest
to weakest. It is the same **impact delta** discipline: a field the legitimate feature already returns
to that same actor is not a leak.

---

## 3. Case 2 - a security control the public API respects and the internal API does not

### The shape of the bug

The application has an account setting that requires manual acceptance of collaboration invitations.
With it enabled, an invitation stays `pending` and grants no access at all. The internal endpoint that
the **invitee's** UI uses to accept pending invitations did not verify that the caller was the invitee.
Whoever sent the invitation could call that endpoint with the id of their own collaboration and mark
the invitation as accepted on the victim's behalf, with no action and no notification on her side.

### How it was found

This is the part that holds up as process, because the value is in the order of the steps:

1. **Start from the triager's response, not from the endpoint.** An earlier finding of mine had been
   closed as informative, on the grounds that the default behaviour is documented and that a report
   would need to show a **security control being bypassed**. That defines the next target: no longer
   the default behaviour, but the control the user explicitly turns on.
2. **Enable the control and attack the documented public API first.** Five variants, all of which
   upheld the control: invitation by email, invitation by numeric user id, invitation with
   notification suppressed, injecting `"status": "accepted"` into the creation body, and updating the
   collaboration status directly (which answered `400 invalid_status`). **The five negatives are the
   value of the test**: they prove the control is real and narrow the problem down to a single
   possible path.
3. **Ask how the legitimate UI performs the privileged action.** I observed the invitee's flow when
   accepting an invitation and captured the internal route the interface calls, whose body is a list
   of collaboration objects.
4. **Verify who can reach the identifier.** The inviter reads the pending collaboration's id from
   their own collaborator list. This is neither guessing nor bruteforce: the attacker already holds
   the id legitimately. That check is what separates an exploited bug from a hypothetical one.
5. **Call the route from the wrong side.** Same request, **inviter's** session. 200, invitation
   accepted, access granted. Reproduced in both directions, with the two accounts swapping the victim
   role, to rule out state contamination.
6. **Clean up and restore.** Collaborations removed, folders deleted and purged on both accounts, and
   the sharing settings restored to their original values - including on one account that was already
   off-default before the test, which I recorded so as not to confuse the baseline with an effect of
   my own.

### Why it duplicated

Authz on a BFF endpoint **that has an equivalent button in the UI** is well-trodden surface. The
interface documents the route's existence to anyone who opens DevTools. The measurement was clean and
the framing cleared the bar the triager himself had set - and it was still contested ground.

---

## 4. The signal that predicts a duplicate

From those two, I extract a predictor I now apply before investing time:

> **If the action has an equivalent button in the interface, assume the surface is well-trodden.**

The corollary is operational, not philosophical: a button exists => the route shows up in the network
tab for any logged-in person, in ten seconds, with no tooling at all. The difficulty of the bug does
not protect against duplication; the **visibility of the surface** is what decides. The order of value
I adopted:

| priority | surface | why |
|---|---|---|
| high | a namespace my session reaches but my UI does not expose | nobody finds it without harvesting bundles |
| high | a service that answers for my plan, whose feature my plan does not have | mismatch between entitlement and authz |
| medium | legacy RPC / old route parameter still routed | no UI and no documentation |
| low | BFF endpoint with a matching button | everyone has already been through there |

---

## 5. The countermeasure: find the surface that has no button

After the second duplicate, I changed the target of the recon phase. In a micro-frontend application,
the bundles the page **loads** are a fraction of the code: the host loads *remotes* on demand, each
with its own chunks, and a good share of the API paths exist **only** in there.

### 5.1 Procedure - reconstruct the chunk URLs without executing the JS

1. **Detect federated modules** on the authenticated page: a manifest object injected into the HTML
   (`webpackRemotesManifest`, `__federation`, `mf-manifest.json`), the `__webpack_require__.u`
   function in the host bundles, a CDN segmented by product.
2. **Extract the manifest** from the HTML of a logged-in page. This is the only step that needs a
   session. Each entry gives hostname, path, version and the name of the remote's entry file.
3. **Download the entries** (anonymously, straight from the CDN). They are module federation runtime:
   they serve to find the **chunk map**, not to grep for endpoints. In my case the entries contained
   **zero** occurrences of the BFF prefix - anyone grepping only the entry concludes, wrongly, that
   there is no surface there.
4. **Reconstruct the URLs by parsing, never with `eval`.** This is third-party code downloaded from
   the internet; running `eval` on it means executing arbitrary code on your own machine. Three rules
   that cost me a round to discover:
   - the right function is found by **punctuation**, not by a regex on `u=`: the expression must
     contain `.js` **and** a map with at least 3 hexadecimal values of 6 to 24 chars;
   - prefix = the **last** literal ending in `/` before the first map (there are hardcoded URLs ahead
     of the function, and concatenating every literal produces garbage);
   - suffix = the **last** literal that starts with a dot and ends in `.js`.
5. **Validate 1 URL per remote with HEAD before downloading in bulk.** Layouts coexist within the same
   app (`.chunk.js`, `.bundle.js`, plain `.js`, with different prefixes per remote). In my run, 3 of
   26 remotes had the wrong suffix, and **only** this test revealed it.
6. **Filter out translations before downloading** (`locale|messages|lang|i18n|translation` against the
   name map): this cut 394 of 2494 chunks with nothing lost, because a translation chunk holds no
   endpoints.
7. **Download with throttling** (few connections, short pause). It is static content on the CDN, not
   on the target's app, but unnecessary volume is unnecessary noise. A chunk hash is immutable, so a
   local cache pays off: only re-download when the remote's version changes.
8. **Grep for the BFF prefix capturing the namespace**, not for `/api/`. The output of this phase is
   not a list of endpoints: it is the **diff of namespaces against the list I already knew**.

**Measured result:** 26 remotes, 2494 chunks reconstructed, 2100 after filtering, 1951 downloaded with
200 and 148 with 404 (ids that only carry CSS - normal). Five BFF namespaces that no previous session
had ever seen appeared **exclusively** in the remotes' chunks. One of them was the billing subsystem,
with 24 routes and zero matching buttons in my interface.

### 5.2 An honest blind spot in this phase

There are endpoints that live in no bundle at all. Three lock/save routes in an editor appeared
neither in the main bundle, nor in the host's 427 chunks, nor in the remote's own 27 chunks: they were
found through **traffic capture** while exercising the feature. Static analysis and capture are
complementary, not substitutes. If a feature matters, exercise it with the network recording.

---

## 6. Triage before testing: three cost decisions

A new namespace in the grep does not mean a live namespace, and new surface does not mean high
severity. Three filters I run before building any scenario:

### 6.1 Reachability - GET only, anonymous x session matrix, across every host

Learning the **target's error dialect** is what gives the status codes meaning:

| response | reading |
|---|---|
| 200 | alive and answering my session. Priority |
| 401/403 with a structured error body | the service exists and is authenticating. Worth an authz scenario |
| 401/403 **empty** | usually the namespace **does not exist** on that host. Dropped two candidates without spending a test |
| 404 on a route the bundle declares | it exists but a parameter is missing. Finding the param is the next step |
| 400 with a validator message | the validator is talking: use the message to extract the schema |
| 200 with an error in the body | *errors-as-data*. Never conclude "it worked" from the status - always read the body |

That last row has already cost me two false positives of my own: GraphQL mutations returning 200 with
"Authorization Failed" inside the payload, read as success because the query only asked for
`__typename`. These days every mutation asks for the errors field alongside.

### 6.2 Severity ceiling, calculated before investing

A measured example: a redirect parameter was accepted **with no validation whatsoever** in an OAuth
flow, which looked like a path to account takeover. But the `redirect_uri` registered with the
provider is **fixed**, so the code never passes through the attacker's domain, and changing it
requires the target's `client_secret`. Real ceiling: authenticated open redirect. Still worth
measuring because it is cheap - but not as the main bet, and that is decided **before** building the
setup.

### 6.3 Error branches before an expensive setup

In an OAuth/redirect flow, the `error=` branch or the invalid-code branch sometimes redirects already
and hands you the answer for free. I tested with a freshly captured valid `state`: 400 with no
`Location` header on every branch. That **negative** is what proved the expensive setup was
unavoidable, at a cost of 10 requests.

---

## 7. The checklist I use today

Before investing in an endpoint:

- [ ] Is there an equivalent button in the UI? If so, assume well-trodden surface and look for the variant without a button.
- [ ] What is the severity ceiling if the bug exists exactly as I imagine it? Is the setup worth it?
- [ ] Does the GET matrix (anonymous x session, every host) say this is alive?
- [ ] What is the **delta** over what the legitimate feature already returns to the same actor?
- [ ] Is the required identifier reachable legitimately, or am I assuming guesswork?
- [ ] Is the baseline isolated - a clean scenario, with no collaboration or residual state that would explain the 200?
- [ ] Do the error branches already hand over the answer without the full setup?

During:

- [ ] No automated scanners; narrow automation, low volume, a researcher identification header.
- [ ] Only my own accounts and objects; exploitation limited to the minimum that demonstrates impact.
- [ ] Write routes in billing are not exercised: map them, read the schema, measure authz on reads only, and state in the report that writes were not exercised as a scoping decision.
- [ ] Reproduce in both directions and restore the accounts' state at the end.

---

## 8. What this document is not

It is not a method for "avoiding duplicates" in any guaranteed sense - in a mature program, any visible
surface has already been looked at by hundreds of people. It is a change in **where** to spend time:
from surface the interface advertises to surface that only appears once you reconstruct the
application's module graph, with cost filters ahead of every investment. The two duplicates are the
evidence that the measurement side was working; what was missing was target selection.
