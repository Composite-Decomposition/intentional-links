# Intentional Links Specification

* **Version:** `0.1-draft`
* **Status:** Community Draft (Open for RFCs and Implementations)
* **Compatibility:** Backward-compatible extension of the Robust Links specification [1]
* **Maintainers:** [Composite Decomposition](https://github.com/Composite-Decomposition)

---

## 1. Abstract & Motivation

Standard hyperlinks bind a human-readable anchor to a network location (`href`), while severing the author's semantic reason for creating the reference. Automated link checkers validate transport mechanics (`HTTP 200 OK`), not meaning. When an external resource is replaced by a generic landing page, moved behind an unrelated redirect, or repurposed over time, transport checks continue to pass while the reader loses the intended context—a silent failure mode termed **content drift**.

Existing archival specifications such as Robust Links solve *link rot* (`404 Not Found`) by decorating anchor elements with snapshot metadata (`data-versionurl`, `data-versiondate`, `data-originalurl`). However, Robust Links do not express *what* the target resource is expected to convey, nor can they detect when a live `200 OK` target has semantically betrayed the linking document.

**Intentional Links** extend Robust Links with two declarative HTML attributes—`data-intention` and `data-action`—enabling asynchronous, build-time semantic validators to verify whether a target resource still fulfills the author's intent and deterministically heal the DOM when drift occurs.

---

## 2. Conformance & Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

* **Intentional Link:** An HTML `<a>` element containing a non-empty `href` attribute and a non-empty `data-intention` attribute.
* **Target Resource:** The final representation retrieved after following HTTP redirects (`301`, `302`, `307`, `308`) from the URI specified in `href`.
* **Semantic Intent:** The concise, natural-language condition declared in `data-intention` that the Target Resource must satisfy to remain valid within the context of the linking document.
* **Content Drift:** A state in which the Target Resource resolves over HTTP (`2xx`) yet fails to satisfy the declared Semantic Intent.
* **Evaluator:** Any compliant build-time CLI, CI/CD step, static site generator plugin, or automated AI agent that validates Intentional Links and applies DOM transformations.

---

## 3. Relationship to Robust Links (Backward Compatibility)

Intentional Links are designed as a **strictly backward-compatible superset** of the Robust Links specification:

1. **Valid HTML5 & Robust Links:** Every Intentional Link is a valid HTML5 `<a>` element. Authors **SHOULD** pair `data-intention` with standard Robust Links attributes (`data-versionurl`, `data-versiondate`, and/or `data-originalurl`) whenever archival fallback is desired.
2. **Lifecycle Separation:** Robust Links define *runtime and archival decoration* for browsers, crawlers, and client-side scripts (`robustlinks.js`). Intentional Links define *authoring-time and build-time semantic validation*.
3. **Zero Runtime Overhead:** Evaluators **SHOULD** execute asynchronously during build or CI/CD pipelines rather than intercepting user clicks at runtime.
4. **Drop-In Output Compatibility:** Once an Evaluator applies a healing action (`switch-to-snapshot`, `switch-to-robust`, `unlink`, or `keep-live`), the resulting HTML conforms directly to standard Robust Links or plain HTML and works without modification alongside existing Robust Links client-side libraries.

---

## 4. Attribute Specification

An Intentional Link is expressed on an HTML `<a>` element using the following attributes besides those inherited from Robust Links:

| Attribute | Requirement | Specification | Value Format |
| :--- | :--- | :--- | :--- |
| `href` | **MUST** | HTML | Valid URL of the primary target resource. |
| `data-intention` | **REQUIRED** | Intentional Links | Non-empty natural-language assertion describing the required semantic content of the target. |
| `data-action` | **OPTIONAL** | Intentional Links | Case-insensitive fallback directive: `keep-live` (default), `unlink`, `switch-to-snapshot`, or `switch-to-robust`. |

### 4.1 `data-intention`

The `data-intention` attribute **MUST** contain a self-contained statement describing what the Target Resource must explain, define, or provide.

* Authors **SHOULD** write concrete, falsifiable assertions (e.g., `"Must explain configuring evarga/jenkins-slave Docker image as template for Jenkins slaves"`) rather than vague labels (e.g., `"Documentation"`).
* Evaluators **MUST** ignore `<a>` elements where `data-intention` is absent or contains only whitespace.

### 4.2 `data-action`

The `data-action` attribute declares the deterministic DOM mutation an Evaluator **MUST** execute when semantic validation evaluates to `FAIL`.

* Valid values are `keep-live`, `unlink`, `switch-to-snapshot`, and `switch-to-robust`.
* Values **MUST** be matched case-insensitively after trimming leading and trailing whitespace.
* If `data-action` is omitted, empty, or unrecognized, the Evaluator **MUST** default to `keep-live`.

---

## 5. Evaluation & Validation Semantics

A compliant Evaluator processing a document **MUST** adhere to the following validation rules:

1. **Redirect Traversal:** The Evaluator **MUST** follow standard HTTP redirects (`301`, `302`, `307`, `308`) to inspect the final Target Resource. HTTP `4xx` or `5xx` responses and network timeouts **MUST** be treated as validation failures (`FAIL`).
2. **Boilerplate Isolation:** Before semantic evaluation, the Evaluator **SHOULD** strip non-content structural elements (`<script>`, `<style>`, `<nav>`, `<header>`, `<footer>`, `<aside>`, `<noscript>`) so navigation links or site-wide footers do not produce false-positive matches on drifted pages.
3. **Closed-World Assessment:** Semantic judgment **MUST** rely solely on the extracted content of the Target Resource compared against `data-intention`, without assuming the page still contains historical content based on its URL or domain reputation.
4. **Conservative Failure Bias:** If the extracted content is empty, behind an authentication wall, ambiguous, or lacks clear evidence satisfying `data-intention`, the Evaluator **MUST** return `FAIL`.

---

## 6. Normative DOM Transformation Rules

When an Intentional Link passes validation (`PASS`), the Evaluator **MUST NOT** alter `href` or Robust Links attributes.

When an Intentional Link fails validation (`FAIL`), the Evaluator **MUST** transform the `<a>` element according to `data-action`:

### 6.1 `keep-live` (Default)
* **Behavior:** Leave only `href` and remove all Robust Links attributes.
* **Signaling:** The Evaluator **SHOULD** emit a non-blocking warning in build logs for human review.

### 6.2 `unlink`
* **Behavior:** Remove the `href` attribute from the `<a>` element, preserving its inner text and surrounding markup so readers are not routed to a misleading destination.

### 6.3 `switch-to-snapshot`
* **Precondition:** Requires a non-empty `data-versionurl` attribute. If `data-versionurl` is missing, the Evaluator **MUST NOT** modify `href` and **SHOULD** emit a warning.
* **Behavior:**
  1. Set `href` to the exact value of `data-versionurl`.
  2. Remove all Robust Links attributes.

### 6.4 `switch-to-robust`
* **Precondition:** Requires non-empty `data-originalurl` and `data-versiondate` attributes. If any is missing, the Evaluator **MUST NOT** modify `href` and **SHOULD** emit a warning.
* **Behavior:** None.

### 6.5 Production Attribute Stripping (Optional)
An Evaluator **MAY** offer an option to strip `data-intention` and `data-action` from final production HTML artifacts after transformation, leaving behind pure Robust Links or standard HTML anchors.

---

## 7. Worked Examples

### 7.1 Missing Resource
An author cites a historic CloudBees engineering guide and binds both a Web Archive snapshot and a semantic contract.

```html
<a href="https://www.cloudbees.com/blog/docker-jenkins-plugin"
   data-versionurl="https://web.archive.org/web/20210415/https://www.cloudbees.com/blog/docker-jenkins-plugin"
   data-intention="Must explain configuring evarga/jenkins-slave Docker image as template for Jenkins slaves"
   data-action="switch-to-snapshot">
   CloudBees Docker Jenkins Guide
</a>
```

### 7.2 Drift Event with Redirection
IEEE Computer Society restructures its site and redirects `www.computer.org/education/registered-education-providers` (`301` → `200 OK`) to a generic education related homepage. A traditional link checker passes; an Intentional Links Evaluator detects that the extracted homepage text does not list registered education providers (`FAIL`).

```html
<a href="https://www.computer.org/education"
   data-versionurl="https://web.archive.org/web/20190430175337/https://www.computer.org/education/registered-education-providers"
   data-versiondate="2019-04-30"
   data-originalurl="https://www.computer.org/education/registered-education-providers"
   data-intention="Must list IEEE CS registered education providers"
   data-action="switch-to-robust">
   IEEE CS registered education providers
</a>
```
### 7.3 Drift Event without Redirection
Over time, ACME Corporation modified its tutorial to the point that non-experts could no longer understand it.

```html
<a href="https://example.com/tutorial"
   data-versiondate="2023-01-01"
   data-originalurl="https://example.com/tutorial"
   data-intention="The Gunning fog index of the English text shall be less than 13."
   data-action="switch-to-robust">
   ACME tutorial
</a>
```
---

## 8. Related Work

The problem of link integrity has been addressed from several angles, but no existing
specification combines **author-declared, natural-language intent** with **build-time,
deterministic remediation**. This section positions Intentional Links relative to the
most relevant prior art and clarifies the gap this specification fills.

### Link Rot and Robust Links

The most common response to link degradation is **link rot** (HTTP 404 or 410), and the
**Robust Links** specification [1] is the closest existing standard. Robust Links
addresses rot by decorating `<a>` elements with snapshot metadata (`data-versionurl`,
`data-versiondate`), enabling a client or archive to redirect to a preserved copy when
the live target disappears. Intentional Links is designed as a **backward-compatible
superset** of Robust Links: an element may carry both the Robust Links attributes and
the `data-intention`/`data-action` attributes defined here. The critical distinction is
that Robust Links responds to a target that is *gone*, whereas Intentional Links
responds to a target that is *still reachable but no longer says what the author
claimed it would say* — the content-drift problem.

### Semantic Drift and Automated Verification

**Content drift** (sometimes called *semantic drift*) — a live, HTTP 200 target whose
content no longer matches the context in which it was cited — is a recognized research
problem. Recent academic work such as **SemLink** [2] proposes an automated test oracle
for semantic hyperlink verification, using sentence-embedding models to detect when a
target's content diverges from its anchor text or surrounding context. That line of work
validates the importance of the problem and offers ML-based *detection*. Intentional
Links is complementary rather than competing: it does not specify how the Evaluator must
detect drift, but it standardizes **how the author declares the intent to be
checked** and **what deterministic action follows a failed check**. In other words,
SemLink and similar systems provide the *oracle*; Intentional Links provides the
*declarative contract and remediation protocol* around that oracle.

### Historical Precedent: Web Intents and `rel="intent"`

The idea of attaching intent semantics to a hyperlink is not without precedent. The W3C
**Web Intents** effort explored adding `rel="intent"` to links so that a
link could declare the *service action* it intended to invoke (e.g. share, edit, view)
rather than merely its destination [3]. Web Intents was oriented toward **service
discovery and interoperability** — routing a URL to a handler — and did not attempt to
validate the *content* of the target or define fallback behavior on failure. Intentional
Links differs in purpose (content-contract validation), syntax (`data-intention` holding
a falsifiable natural-language assertion), and lifecycle (build-time evaluation by an
Evaluator, not runtime routing by a user agent).

### Existing Link Semantics: `rel` and Typed Links

HTML's `rel` attribute and the IETF **Web Linking** framework (RFC 5988 and its
successor RFC 8288) [4] already allow an author to declare the *relationship type*
between the current document and the target — values such as `author`, `license`,
`alternate`, or `nofollow`. The Semantic Web community describes these as **typed
links**, where the edge itself carries meaning. Intentional Links operates at a
different level of granularity: `rel` categorizes the *kind* of relationship, while
`data-intention` states a *falsifiable proposition about the target's content*. The two
are orthogonal and may be combined on the same element. Nothing in this specification
depends on, or conflicts with, the `rel` registry.

### Interaction Intent: Open UI `interesttarget`

The Open UI community group has discussed an `interesttarget` attribute (and related
"interest" events) for buttons and links, which signals a user's intent to trigger a
particular interaction — typically revealing a popover or preview [5]. This is a
**runtime, user-facing interaction hint**; it is not a content assertion and is not
evaluated against the target's substance. It is mentioned here only to distinguish
*interaction intent* (what the user wants to do) from *content intention* (what the
author asserts the target will say), which is the concern of this specification.

---

## 9. Community Implementations & Collaboration

This repository serves as the canonical specification hub for **Intentional Links**. Rather than locking the standard to a single programming language or LLM vendor, we encourage community-maintained evaluators, CI actions, and static site generator plugins across ecosystems.

You can participate in the following ways:
* **Propose a Specification Refinement (RFC):** Open a specification proposal to discuss new `data-action` strategies, Markdown syntax conventions, or caching metadata.
* **Report an Edge Case:** Found an ambiguous redirect, SPA hydration issue, or Robust Links interaction? File an edge case.
* **Build & Register an Evaluator:** Implement the spec in your preferred language or SSG and submit an issue to list it in the table above.

---

## 10. References

[1]: Robust Links specification. https://robustlinks.org/spec/

[2]: SemLink: A Semantic-Aware Automated Test Oracle for Hyperlink Verification using Siamese Sentence-BERT. https://arxiv.org/abs/2604.05711

[3]: W3C Web Intents. https://www.w3.org/TR/web-intents/

[4]: M. Nottingham, "Web Linking," IETF RFC 8288 (successor to RFC 5988).
    https://www.rfc-editor.org/rfc/rfc8288

[5]: Open UI Community Group, `interesttarget` attribute / interest invokers proposal.
    https://open-ui.org/
