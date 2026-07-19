---
marp: true
theme: default
paginate: true
size: 16:9
footer: "@@FOOTER@@"
style: |
  section { font-size: 24px; }
  section table { font-size: 0.85em; }
  section.big { font-size: 35px; }
  section.big table { font-size: 1em; }
  section.shrink85 pre { font-size: 0.85em !important; }
  section.shrink85 pre code { font-size: inherit !important; }
  section.dense table { font-size: 0.72em; }
  section.tight { font-size: 21px; }
  section.tight table { font-size: 0.78em; }
---

<!-- _footer: "" -->
<!-- Slide role: SEPARATOR (title) -->
<!-- _class: big -->

# Uncacheable Directories: WGLC status

## IETF 126, NFSv4 WG

[draft-ietf-nfsv4-uncacheable-directories](https://datatracker.ietf.org/doc/draft-ietf-nfsv4-uncacheable-directories/)

&nbsp;

&nbsp;

Tom Haynes &mdash; @@DATE@@

<!-- Presenter note: this is a WGLC report-out, not a redesign
     discussion.  The room's decision path is on slide 4.  Do not
     re-litigate the design space from the front of the room.
     Do not name reviewers from the podium; frame all WGLC content
     as "WGLC raised" / "the proposed -08 text answers". -->

---

<!-- Slide role: CONTENT
     Must-deliver:
     -07 addressed the chairs' -06 readiness punch list and the
     earlier reviewer rounds' motivation defects.  The -07 review then
     raised a philosophically distinct concern in three parts: the
     caching pattern is presented as valid, the "dirent metadata"
     categorization conflates two cache-coherence stories, and the
     principled framework already exists in RFC 5661. -->
<!-- _class: tight -->
# What WGLC raised

**Prior rounds &mdash; settled in -07:**

- Chairs' post-WGLC readiness assessment on -06 &mdash; two publication blockers, four IESG-friction items, one procedural non-consensus finding
- Earlier reviewer rounds &mdash; motivation, per-user angle, concrete-deployment example
- All addressed in the -07 text

**-07 review then raised a philosophically distinct concern:**

1. **Caching mutable attributes from READDIR is unsound in principle.**  Writes to underlying files can occur at any time and are not tracked by the directory's change attribute, so cache reuse can return stale size / change / mtime.  An advisory attribute reads as legitimizing that practice.
2. **"Directory-entry metadata" conflates two cache-coherence stories.**  The constraints on directory content vs. attributes of files listed inside it are not the same and should not be folded into one category.
3. **The principled framework already exists.**  Directory delegations + `CB_NOTIFY` (**RFC 5661 &sect;18.39**) are the right tool for cache coherence over directory state.

---

<!-- Slide role: CONTENT
     Must-deliver:
     -08 does three things, one per WGLC point.  (1) Does not claim
     the caching is safe; explicit section 10.3.2 citation; the
     attribute is a per-directory suppress, not a blessing.  (2)
     Rewords intro and definitions to keep file attributes and
     directory contents distinct.  (3) Cites the framework; does not
     propose the attribute as a substitute.  Close with the
     pattern-mirror to fattr4_uncacheable_file_data. -->
# How -08 accommodates

1. **Does not claim the caching is safe.**  -08 cites RFC 8881 &sect;10.3.2 explicitly.  The draft describes a per-directory server signal that **suppresses** the caching for a class where it is known to be wrong &mdash; not a blessing.
2. **Keeps file attributes and directory contents distinct.**  -08 rewords the introduction and definitions so the attribute name and semantics target the deployment-visible symptom (stale attribute values served from cache); the categorization no longer folds file attributes into directory state.
3. **Cites the framework; does not propose the attribute as a substitute.**  Directory delegations + `CB_NOTIFY` remain the principled answer for the workloads they fit; the attribute is the lightweight signal for workloads whose cost model the framework does not fit &mdash; cost model in the backups.

Design-pattern mirror: `fattr4_uncacheable_file_data` (**draft-ietf-nfsv4-uncacheable-files**).  Protocol permits caching, cache can go stale, per-object server signal suppresses it where the deployment knows it is wrong.  Same shape at the directory layer.

---

<!-- Slide role: CONTENT
     Must-deliver:
     -08 does not adjudicate the caching-is-unsound position; it
     accommodates the specific text asks.  The remaining question is
     a WG-level scoping decision: (a) deprecate, (b) per-deployment
     opt-out (authors' position), (c) require delegations + CB_NOTIFY.
     Frame this as a WG choice, not as adjudication of one reviewer's
     objection. -->
<!-- _class: tight -->
# The choice on the table

The -08 accommodations settle the text-level concerns.  What remains is a **WG-level scoping decision** on whether the mechanism should exist at all:

| Option | What it says | Where it stands |
|---|---|---|
| **(a)** Deprecate READDIR-attribute caching across the board | Retract the caching pattern as unsound in principle. | Contradicts NFS client implementation for **25+ years**: Linux `nfs_inode->cache_validity`, FreeBSD `nfs_node_getattrcache`, illumos, AIX, macOS.  READDIRPLUS (RFC 1813, 1995) was designed to enable it. |
| **(b)** Keep the attribute as a **per-deployment opt-out** for cases where the deployment knows the caching is wrong | Protocol continues to permit caching (as it does today).  The attribute is the per-directory signal that suppresses it for a specific class. | **Authors' position.**  Consistent with the pattern already adopted at the file-data layer (`fattr4_uncacheable_file_data`). |
| **(c)** Require the deployments in &sect;2 to use directory delegations + `CB_NOTIFY` instead | Route the described HPC-ingest class through the notification framework. | Directory delegations are shipping in **Linux and FreeBSD**.  The framework and the attribute answer different workload shapes &mdash; cost model on slide 4. |

**Authors' recommendation: (b).**

<!-- Presenter note: this is the deterministic question for the WG.
     Frame it as a WG-level choice, not as adjudicating a single
     reviewer's objection.  Authors' proposed shape for a consensus
     writeup if the room lands on (b): a one-paragraph "the WG
     considered the question of whether READDIR-attribute caching
     should be deprecated, and declined to adopt that position; the
     attribute is retained as a per-deployment opt-out consistent
     with the pattern at the file-data layer."  This is authors'
     proposed language, not chair-role guidance. -->

---

<!-- Slide role: CONTENT
     Must-deliver:
     -08 addresses the WGLC comments (slide 3).  The chair's call to
     make.  Authors' preference: a confirming call to the list
     (silence = consensus).  A focused re-review of the -07 -> -08
     delta is a fine fallback if anyone wants one more pass.  Full
     new WGLC would be heavier than the delta warrants.  Aim is WGLC
     re-closes this cycle so publication can be requested. -->
# What we're asking for procedurally

**WGLC comments addressed in -08** (see slide 3).

- **Preferred:** confirming call to the list &mdash; silence = consensus on the -08 changes
- **Fallback:** focused re-review of the -07 &rarr; -08 delta
- **Aim:** WGLC re-closes this cycle &rarr; publication request

Chair's call &mdash; flagging preference, not dictating.

---

<!-- Slide role: SEPARATOR (Backups) -->
<!-- _class: big -->
# Backups

---

<!-- Slide role: CONTENT
     Must-deliver:
     Delegations are shipping in Linux and FreeBSD.  At scale
     (10^3 clients x 10^3 directories) the framework carries ~10^6
     delegation records + CB_NOTIFY fan-out on writes; the attribute
     carries ~10^3 static records and no callback traffic.  Point at
     the "per-caller policy variance" row - it's the deployment
     axis the framework structurally cannot serve. -->
<!-- _class: tight -->
# Delegations vs the attribute &mdash; cost shape at scale

Directory delegations + `CB_NOTIFY` are shipping in Linux and FreeBSD.  That is good &mdash; but the framework and the attribute answer **different workload shapes**.

*Illustrative scale: 10^3 clients &times; 10^3 directories.*

| Cost dimension | Directory delegations + `CB_NOTIFY` | Uncacheable-directories attribute |
|---|---|---|
| Server state | 1 record per (client, directory) &rarr; **~10^6 records** | 1 attribute per directory &rarr; **~10^3 records** |
| Callback traffic on a directory-change event | `CB_NOTIFY` fan-out to every client holding the delegation | **zero** &mdash; no callback channel used |
| Steady-state READDIR load | amortized: reads served from client cache under delegation | full: every client's READDIR reaches the server |
| Per-caller policy variance in the response | not supported (the delegation is one shared view) | supported (server generates the response per call) |

<!-- Presenter note: shipping-in-Linux-and-FreeBSD strengthens
     option (c), NOT weakens option (b).  With delegations coming to
     the field, the honest comparison is on cost shape, not on
     "delegations don't ship."  This slide moves the argument off the
     15-years-undeployed frame and onto the workload-shape frame,
     which is the durable argument. -->

---

<!-- Slide role: CONTENT
     Must-deliver:
     Delegations fit read-heavy stable directories - the framework
     earns its state cost via dedup'd reads.  The attribute fits
     high-churn or per-caller-varying directories - the framework's
     fan-out compounds against those workloads; the attribute's
     zero-state / zero-callback shape does not.  Both mechanisms
     coexist; the attribute is not a substitute for the framework. -->
# Delegations vs the attribute &mdash; where each fits

- **Delegations** &mdash; long-lived directories with low churn and high read rates (home directories, project trees, shared libraries).  The framework earns its state cost via dedup'd reads.
- **The attribute** &mdash; directories where per-second churn exceeds any cache TTL (&sect;2's HPC ingest class), or where the response varies per caller.  The framework's callback fan-out compounds against the workload; the attribute's zero-state, zero-callback shape does not.

**Both mechanisms coexist.**  The attribute is not a substitute for the framework; it covers a workload class the framework's cost model does not fit.

<!-- Presenter note: this is the argument that carries the room past
     "shipping delegations obsoletes the attribute" - they answer
     different workload shapes, and both will exist in real
     deployments.  Do not concede that (c) obsoletes (b). -->
