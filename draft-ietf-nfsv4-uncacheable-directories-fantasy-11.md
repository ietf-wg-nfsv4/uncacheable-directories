---
title: Adding an Uncacheable Dirent Metadata Attribute to NFSv4.2
abbrev: Uncacheable Dirent Metadata
docname: draft-ietf-nfsv4-uncacheable-directories-latest
category: std
date: {DATE}
consensus: true
ipr: trust200902
area: General
workgroup: Network File System Version 4
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping, comments]

author:
 -
    ins: T. Haynes
    name: Thomas Haynes
    organization: Hammerspace
    email: loghyr@gmail.com

normative:
  RFC4506:
  RFC7862:
  RFC7863:
  RFC8178:
  RFC8881:

informative:
  I-D.ietf-nfsv4-uncacheable-files:
  RFC7204:
--- abstract

Network File System version 4.2 (NFSv4.2) clients may cache the
file attributes that a READDIR returns alongside each directory
entry.  Such a cache is not invalidated by the directory's change
attribute, which reflects changes to the directory and its entries
but not writes to the files those entries name, so it can become stale
when another client changes one of those files.  In some deployments
this produces incorrect size and timestamp values often enough to
be a deployment problem.  This document introduces an uncacheable
dirent metadata attribute for NFSv4.2 that allows a server to
identify a directory for which an honoring client goes to the server
for each enumeration, and does not report an entry's attributes from a
value it held before that READDIR.  The attribute
places no requirement on the files the entries name; an honoring
client may continue to retain the entries themselves and use them to
resolve names, but may not satisfy a directory read from them.

--- note_Note_to_Readers

Note to RFC Editor: please remove this section prior to publication.

Discussion of this draft takes place
on the NFSv4 working group mailing list (nfsv4@ietf.org),
which is archived at
[](https://mailarchive.ietf.org/arch/search/?email_list=nfsv4). Source
code and issues list for this draft can be found at
[](https://github.com/ietf-wg-nfsv4/uncacheable-directories).

Working Group information can be found at [](https://github.com/ietf-wg-nfsv4).

--- middle

# Introduction

Clients of remote filesystems may cache the file attributes returned
by READDIR alongside each directory entry, to reduce the volume of
follow-on GETATTR traffic for entries the client has already seen.
That cache is not invalidated by the directory's change attribute,
which reflects changes to the directory and its entries but not
writes to the files those entries name; it can become stale when
another client changes one of those files.  In some deployments the
cost of that staleness
is high enough to be a deployment problem; the conditions are
described in {{deployment-motivation}}.

In this document, the term directory is used to describe the
context in which directory entries are retrieved.  The uncacheable
dirent metadata attribute applies to the caching of dirent
metadata -- the file object attributes, such as size and timestamps,
returned alongside each entry.  It does not change how a client
validates cached dirents, which remains governed by the directory's
change attribute, nor does it constrain caching of the directory object
or of file data.  For a directory on which it is set, however, an
enumeration is satisfied by a READDIR rather than from a cache.

Stale dirent metadata for concurrently modified files also undermines
the effectiveness of uncacheable file data semantics
({{I-D.ietf-nfsv4-uncacheable-files}}) in the same deployment:
applications can observe inconsistent metadata and data views even
when file data caching is disabled.

This document introduces the uncacheable dirent metadata attribute
to NFSv4.2 to allow servers to mark the directories for which reusing
previously obtained dirent metadata is unsuitable.  Using the process
detailed in {{RFC8178}}, the revisions in this document become an
extension
of NFSv4.2 {{RFC7862}}.  They are built on top of the external data
representation (XDR) {{RFC4506}} generated from {{RFC7863}}.

# Deployment Motivation {#deployment-motivation}

A class of deployment uses NFSv4.2 to serve a shared directory to
many concurrent NFSv4.2 client writers, each writing files within
the directory.  Workloads of this kind are typical of
High-Performance Computing (HPC) environments, where a single
output directory may receive results from hundreds or thousands of
compute nodes simultaneously, and of large-scale data-ingest
pipelines where many producers append to a common landing
directory.  The files within such a directory have their attributes
-- size and timestamps in particular -- modified at a high rate by
clients other than the one performing READDIR.

{{RFC8881}} Section 10.6 permits a client to cache the file attributes
returned by READDIR on the same basis as attributes obtained by GETATTR:
cached per file, bounded by an upper time boundary, and revalidated
against that file's change attribute.  In a directory receiving writes
from thousands of compute nodes, any nonzero cache lifetime yields stale
size and time_modify for most entries most of the time, and revalidating
each entry individually costs one GETATTR per entry -- the very traffic
that requesting attributes in READDIR exists to avoid.  NFSv4.2 gives a
server no in-band way to tell a client that the acceptable attribute
cache lifetime for the children of a particular directory is zero; mount
options express this out of band and per client, not per directory.
What is missing is not a way to bound that lifetime, but a way to mark
a directory so that a client reading it goes to the server, and does
not report for those entries values it held beforehand.

The staleness has correctness consequences, not merely cosmetic ones.
An incremental backup or an rsync scan that decides what to copy from
the size and time_modify reported for each entry will silently skip a
file whose cached metadata predates a concurrent write, leaving data
uncopied.  This attribute lets a server mark the directories where that
outcome is likely, so that an honoring client fetches current metadata
on each enumeration.

The fattr4_uncacheable_dirent_metadata attribute is the server's
mechanism to identify a directory for which this risk is high
enough that reusing previously obtained values is not safe.  When the
server sets
the attribute on a directory, an honoring client goes to the server
for each enumeration, and does not report an entry's attributes
from a value it held before that READDIR.

# Definitions

dirent

: A directory entry -- the (name, fileid) pair that names a file or
subdirectory within a directory.  This is what a client maintains for
an entry, whatever a given READDIR response carries on the wire; it is
the pair POSIX exposes as d_name and d_ino.  A dirent itself does not
include the file attributes returned alongside it.

: On the wire only the name is structural: {{RFC8881}} Section 18.23.2
gives entry4 a name and an attrs, and every other value -- fileid
included -- reaches the client through attrs, and only if the client
named it in attr_request.  A client that wants the fileid of an entry
therefore requests it like any other attribute, and the rule in
{{sec_dirents}} applies to it as to any other.  For clients that
request it anyway -- a POSIX client must, to fill in d_ino -- applying
the rule to fileid adds no traffic.  A client that does not request it,
or a server that does not support it, has no value for the rule to
attach to.

dirent metadata

: The file attributes of the object an entry names (size, mtime, ctime,
atime, mode, owner, etc.), as a READDIR is able to return them
alongside the entry.  The term names the class of attributes, not the
subset a particular response happened to carry: an attribute is dirent
metadata whether or not the client named it in that READDIR's
attr_request.  These attributes belong to the underlying file object,
not to the directory; they change when the underlying file is written,
which is independent of the directory's change attribute.  The term is
a naming convenience and does not assert that those attributes inherit
the directory's cache-coherence semantics.

dirent caching

: A client-side cache of the dirents themselves -- the (name, fileid)
pairs -- used to resolve names locally and to avoid repeated READDIR
traffic.  Whether such a cache remains valid is governed by the
directory's change attribute: the
directory changes when an entry is created, removed, or renamed, and a
fileid is stable for as long as its entry names the same object.  A
fileid is therefore cached with the name rather than with the file
attributes: writes to a file change its size and timestamps without
touching either the name or the fileid.  This document does not change
what a client may hold in this cache, or how it validates it.  For a
directory on which the attribute is set, however, each enumeration is
satisfied by a READDIR ({{sec_dirents}}), so for that directory the
cache no longer avoids that traffic.

dirent metadata caching

: A client-side cache of the dirent metadata returned alongside those
entries, used to avoid repeated GETATTR traffic.  Because those file
attributes are not invalidated by the directory's change attribute
(only by writes to the underlying files), this caching is inherently
best-effort and subject to staleness whenever the underlying files are
modified.  This is the caching that the attribute defined in this
document constrains.

uncacheable dirent metadata attribute

: An NFSv4.2 file attribute that advises clients not to serve dirent
  metadata, such as size and timestamps, from a local cache when
  enumerating a directory on which it is set.

readdir

: A directory-read request made by an application, however the client's
interface batches entries -- one entry per call, as POSIX readdir()
presents it, or many, as a getdents-style interface does.  This document
writes that operation in lower case and the NFSv4.2 protocol operation
as READDIR.  The two do not correspond one to one: a single READDIR
ordinarily supplies enough entries for many readdirs, and the
requirement in {{sec_dirents}} is stated in terms of the enumeration a
readdir belongs to, not in terms of a READDIR per readdir.

enumeration

: One pass over a directory by an application: the sequence of readdir
calls, and the READDIRs the client issues to satisfy them, beginning at
an opendir or a rewinddir, and ending at whichever comes first of
end-of-file, a further rewinddir, or a closedir.  An enumeration
abandoned before end-of-file is still an enumeration; it simply covers
fewer entries.  A directory
large enough to need several READDIRs is enumerated by all of them
together.  Because a
READDIR carries the attributes of the entries in that READDIR alone,
what a client learns during an enumeration is current as of the READDIR
that carried each entry, not as of the point the enumeration finished.

honoring client

: A client that implements this attribute and enforces the rule it
defines for a directory on which it has observed the attribute set: go
to the server for each enumeration, and report no value received before
the READDIR that returned the entry ({{sec_dirents}}).  The attribute is
advisory: a client that does not implement it, or that declines to
enforce it, is non-honoring and may
continue to cache dirent metadata.

This document assumes familiarity with NFSv4.2 operations, attributes,
and error handling as defined in {{RFC8881}} and {{RFC7862}}.

# Requirements Language

{::boilerplate bcp14-tagged}

# Caching of Dirent Metadata

The uncacheable dirent metadata attribute constrains what an honoring
client may do when it enumerates a particular directory: it goes to the
server, and does not report for the entries returned any value it held
before that request.  {{sec_dirents}} states this normatively.  A
server sets the attribute on the directories where it knows the
staleness of cached READDIR attributes is particularly likely and
particularly damaging.

The extension defined in this document is OPTIONAL to implement.  The
attribute itself falls in the RECOMMENDED category of {{RFC8881}}
Section 5.2, the taxonomy every NFSv4.2 file attribute is placed in;
that is a statement about the attribute's category and not a BCP 14
keyword, and {{RFC8178}} Section 2.2 records that this use of the word
is a known inconsistency in the base specifications.  A server is not
required to support the attribute.

What the attribute constrains is the freshness of dirent metadata.  An
honoring client may continue to cache the dirents themselves, validated
by the directory's change attribute as it would be for any other
directory, and continue to use them for purposes such as name lookup;
this document does not change how a client validates cached dirents,
though it does withdraw, for a marked directory, the permission in
{{RFC8881}} Section 10.8.2 to use a cached READDIR result in place of a
new one.  For a directory on which it has observed the attribute set,
an honoring client does not use a cached READDIR result to satisfy a
new enumeration.  This matches how clients are typically
built, with the dirent cache maintained separately from the attributes
obtained for the objects the entries name.

Because the attribute governs a READDIR rather than the objects the
entries name, it makes no claim about those objects.  A file reached
through a directory on which the attribute is not set is unaffected,
including where the same file is linked into both a directory on which
it is set and one on which it is not.

This document specifies the required observable behavior rather
than mandating a particular internal implementation strategy.
Clients MAY employ more sophisticated mechanisms, such as
time-limited caches that revalidate against the server on each
READDIR, provided that no value is reported for an entry that the
client received before the READDIR that most recently returned it.

A client can determine whether the uncacheable dirent metadata attribute
is supported for a given directory by examining the supported_attrs
attribute for that directory's filesystem or by probing support using
the procedures described in {{RFC8178}}.

The value of the attribute is itself a directory attribute, cached and
revalidated like any other.  A server that sets or clears it modifies
the directory, so the directory's change attribute moves ({{RFC8881}}
Section 5.8.1.4 defines change as detecting that "file data, directory
contents, or attributes of the object have been modified"), and a
client revalidating the directory observes the new value on its next
fetch of the directory's attributes.  Until it does, it behaves as it
did before: a client that has cached the attribute as false continues to
serve enumerations from its cache, and one that has cached it as true
continues to go to the server after the server has cleared it.  A server
therefore cannot assume that setting the attribute takes effect at any
particular moment, which is consistent with the attribute being
advisory.
{: #sec_activation}

The uncacheable dirent metadata attribute governs the client's
caching of READDIR responses for the directory.  It does NOT govern:

* The lifetime of the client's per-file attribute cache for children
  of the directory.  What the attribute governs is that cache's
  contents relative to a READDIR, not how long those contents remain
  valid afterwards.  A client that holds a single attribute cache per
  file object, populated by READDIR and by GETATTR alike, satisfies
  the requirement by not serving from that cache, for an entry a
  READDIR returned, any value it received before that READDIR;
  nothing here requires a separate cache for the attributes that
  arrived in a READDIR.  Between READDIRs the cache's lifetime remains
  governed by the mechanisms already defined by NFSv4.2 ({{RFC8881}}
  Section 10.6) and is subject to the same staleness from concurrent
  writes.  Clients in deployments using this attribute may wish to
  apply correspondingly short lifetimes, but the present attribute
  does not require this.

* Objects the client reaches without enumerating the directory, and
  operations that return no file attributes (for example, LOOKUP
  without a following GETATTR, or ACCESS).  The requirement attaches
  to entries a READDIR returned and to the metadata reported for them,
  so a value obtained by GETATTR after that READDIR is usable, while
  one obtained before it is not -- however the client came by it.

* The directory's own attribute cache.  The directory object's own
  attributes (mode, owner, etc.) can be cached normally and
  revalidated via the directory's change attribute as usual.

The uncacheable dirent metadata attribute addresses a different
aspect of client-side caching than fattr4_uncacheable_file_data
({{I-D.ietf-nfsv4-uncacheable-files}}).  The file data attribute
governs caching of file contents, while the dirent metadata
attribute governs caching of file attributes returned by READDIR.
The principal motivating changes are the same in both cases -- writes,
which move size, time_modify, and change -- though the rule here applies
to every file attribute a READDIR returns, including ones changed by
SETATTR.  The two attributes govern
different caches: a client honoring fattr4_uncacheable_file_data may
still use validated cached file data, and may still report cached size
and timestamps for that file, subject to the revalidation that document
requires.
The attributes are independent and may be used separately.

This attribute follows the same pattern as
fattr4_uncacheable_file_data ({{I-D.ietf-nfsv4-uncacheable-files}})
applied at the file-data layer.  In both cases:

* The underlying NFSv4.2 protocol permits client-side caching that
  can become stale.

* Client caching of the relevant data is widely implemented in
  practice and reduces network traffic for stable objects.

* For specific objects where the deployment knows the caching will
  produce incorrect results, the server requires a mechanism to
  instruct an honoring client not to reuse previously obtained values
  for those specific objects.

* The attribute does not redefine the legality of caching in the
  general case.  It is a per-object server-side signal that the
  caching is known to be unsuitable for that object.

The attribute does NOT make dirent metadata caching reliable for
directories where it is not set.  Clients MUST NOT interpret the
absence of fattr4_uncacheable_dirent_metadata, or its value being
false, as a guarantee that cached READDIR attributes are
authoritative.  As stated in {{RFC8881}} Section 10.6, all
client-cached attributes are subject to staleness; the attribute
defined in this document only identifies directories for which
staleness is particularly likely and particularly damaging.  The base
specification separates the two concerns this attribute is often accused
of conflating: {{RFC8881}} Section 10.8.2 governs caching of the
directory entries themselves, while Section 10.6 governs caching of the
file attributes that arrive alongside them.  What this attribute
constrains is the freshness of the latter; it reaches the former only to
the extent that obtaining fresh attributes requires a READDIR, which is
why an enumeration may not be served from cache.

This attribute does not define behavior for positive or negative
name caching or for caching of LOOKUP results outside the scope of
file attributes returned by READDIR.

A directory delegation ({{RFC8881}} Section 10.9) lets a client cache a
directory's entries and the directory's own attributes until the server
recalls the delegation.  It is not recalled when the attributes of an
entry within the directory change ({{RFC8881}} Sections 10.9.2 and
10.9.4), so a directory delegation does not, by itself, keep the file
attributes returned by READDIR fresh.  NOTIFY4_CHANGE_CHILD_ATTRS,
requested through GET_DIR_DELEGATION, can deliver changed child
attributes to a delegated client, but it is not a substitute for this
attribute in the deployments of {{deployment-motivation}}:
GET_DIR_DELEGATION is OPTIONAL and is not implemented by the clients and
servers those deployments use; notification cost scales with the number
of delegated clients times the number of changes, which a directory
written by thousands of clients makes prohibitive (and {{RFC8881}}
Section 10.9.4 permits a server that finds a directory is causing too
many notifications to decline to delegate it); and a server may decline
child-attribute notification altogether through the GET_DIR_DELEGATION
response, while the dirent_notif_delay attribute
({{RFC8881}} Section 5.11.2) advertises the minimum delay before the
server will notify.  A client therefore cannot rely on notification for
freshness.

## Uncacheable Dirent Metadata {#sec_dirents}

The fattr4_uncacheable_dirent_metadata attribute is a read-write boolean
attribute that applies to directory objects.
Authorization to query or modify this attribute is governed by
existing NFSv4.2 authorization mechanisms.

The attribute applies only to directory objects.  Support for an
attribute is advertised per file system ({{RFC8881}} Section 5.8.1.1),
so a server that supports this attribute supports it for every object
in that file system.  A GETATTR requesting
fattr4_uncacheable_dirent_metadata for an object that is not a
directory MUST return FALSE; as with rawdev ({{RFC8881}} Section
5.8.2.31), the value SHOULD NOT be considered useful for such an
object.  A server that receives a SETATTR requesting
fattr4_uncacheable_dirent_metadata on an object that is not a directory
MUST return NFS4ERR_WRONG_TYPE.  A server that declines to set the
attribute for reasons of local policy returns NFS4ERR_PERM, or
NFS4ERR_ACCESS where the refusal follows from the caller's access
rights; both are valid SETATTR errors in {{RFC8881}} Section 15.2.

This attribute is set per directory.  This document does not define
propagation of the attribute to subdirectories created within a
directory on which it is set; any such inheritance is a matter of
local server policy.

A client observes the attribute when it receives a value for it: in a
GETATTR of the directory, in a READDIR of the parent that named the
attribute in attr_request, or in the result of its own SETATTR.  The
value a client has observed is the one it received most recently;
receiving TRUE begins the obligation below and receiving FALSE ends it.
An honoring client SHOULD name the attribute in attr_request whenever
it fetches or revalidates the attributes of a directory in a file
system that supports it, so that a change the server makes is picked up
within the ordinary attribute-cache lifetime rather than at an
unbounded later time.

An honoring client that has observed the uncacheable dirent metadata
attribute set on a directory MUST NOT satisfy a readdir of that
directory from READDIR results obtained during a different enumeration.
For each READDIR of that directory it issues after that observation, it
MUST NOT report, for an entry that READDIR returned, a value of a
dirent metadata attribute that it received before that READDIR.

The requirement is keyed on the value the client has observed, not on
the value the server currently holds.  A client that has not yet
refetched the directory's attributes since the server set them is not
in violation; it is not yet subject to the rule.  Nor does the
obligation reach backwards: values a client holds when it observes the
attribute remain reportable until a READDIR issued after the
observation returns the entry they belong to, which the first
enumeration after the observation supplies.  {{sec_activation}}
describes how a client comes to observe a change, and why a server
cannot assume one takes effect at a particular moment.

Two consequences follow.  An honoring client either names the attributes
it will report in that READDIR's attr_request, or obtains them
afterwards by other means; a value it held before that READDIR is not
usable for that entry.  And the guarantee is per READDIR, not per
enumeration: a directory enumerated over several
READDIRs yields, for each entry, metadata as of the READDIR that
returned it, not as of the moment the enumeration completed.

"Before" and "after" here are the order in which the client received
the responses, not the order in which it issued the requests.  A client
therefore needs no additional attribute in order to apply this rule,
and reordering of replies ({{RFC8881}} Section 2.10.6) cannot make it
ambiguous.  Reception order can be permissive: a reply the server built
before the READDIR may arrive after it, and the rule then does not
forbid reporting that value.  Because the rule is a prohibition rather
than a choice of value, a client remains free to prefer the value the
READDIR returned, or a later change attribute, and one that does so is
never worse off.

Entries and metadata carried by the READDIRs of a single enumeration MAY
be retained until that enumeration completes.  {{RFC8881}} Section
10.8.2 requires such a cache to be a consistent snapshot of directory
*contents*, validated by the directory's change attribute; because that
attribute does not move when a file the directory names is written, it
provides no corresponding guarantee for the entries' file attributes,
which are as of the READDIR that carried them.

The uncacheable dirent metadata attribute does not modify the
semantics of the NFSv4.2 change attribute.  Clients MUST continue to
use the change attribute to detect directory modifications and to
determine when directory contents may have changed, even for a
directory on which this attribute is set.  The rule in this section
does not remove the need for change-based validation.

This attribute is advisory, so servers SHOULD NOT rely on it for
correctness: a client that does not implement it, or that declines to
enforce it, may continue to cache dirent metadata.  A server cannot
distinguish those clients from honoring ones.  Observing a GETATTR or
a SETATTR of the attribute shows only that a client knows the
attribute exists, not that it enforces the rule, so such a request is
not a basis for assuming that client is honoring.

A directory delegation would let a client serve dirent metadata from
its cache without refetching, which is incompatible with the rule this
attribute defines.  Accordingly, if a directory has the uncacheable
dirent metadata attribute set and an outstanding directory delegation,
the server MUST recall the delegation.  A server MUST NOT grant a new
directory delegation on a directory while the uncacheable dirent
metadata attribute is set on that directory.

The recall restates behavior {{RFC8881}} Section 10.9.2 already
requires -- a delegation "covers directory attributes and all entries
in the directory", and "if either of these change, the delegation will
be recalled synchronously" -- rather than introducing a new trigger.
It does not, however, tell the client what changed: CB_RECALL carries a
stateid, a filehandle and a truncate flag, and no attribute value, and
a recall may equally follow a change to some other directory attribute.
A recalled client therefore becomes subject to the rule above when it
next observes the attribute set, not by virtue of the recall.

# Example: Directory Enumeration With and Without Dirent Metadata Caching

This example illustrates the difference in client-visible behavior when
dirent metadata caching is enabled versus when the uncacheable
dirent metadata attribute is set on a directory.  In both scenarios,
each readdir("/dir") is a separate enumeration -- the application opens
the directory, reads it to end-of-file and closes it, and opens it
again for the second -- so the retention permitted within a single
enumeration does not carry from the first to the second.  The set of
entries in the directory does not change between the two calls; an
attribute value of one entry is updated at the server between calls.
The difference is whether the stat after the second readdir observes
the updated attribute value.

## Classic Directory Enumeration (Dirent Metadata Cached)

In this scenario, the client caches dirent metadata obtained from the
server and reuses it after a second readdir.

~~~
Application             NFSv4.2 Client        NFSv4.2 Server
-----------             --------------        --------------
readdir("/dir")
   |
   |                     READDIR, size and
   |                     time_modify requested
   |-------------------->------------------------>
   |                     entries:
   |                       a (size=100)
   |                       b (size=200)
   |<--------------------<------------------------
   |<-- names a, b
                        (attributes retained per
                         entry, bounded by the
                         attribute cache lifetime)

                                          (concurrent writer extends
                                           a from size=100 to
                                           size=500)

readdir("/dir")
   |                     (served from the cached
   |                      READDIR result; no
   |                      network traffic)
   |<-- names a, b

stat("/dir/a")
   |                     (served from the retained
   |                      READDIR attributes)
   |<-- size=100
~~~
{: #fig-cached-dirents title="Dirent Metadata Cached"}

In this case, {{fig-cached-dirents}} shows a second readdir satisfied
from the cached result of the first.  No READDIR reaches the server, so
nothing refreshes what the client holds for entry a, and the stat that
follows reports the size as it was at the time of the first READDIR --
not the update the server took between the two.  This behavior
maximizes performance and is what {{RFC8881}} Sections 10.6 and 10.8.2
permit, but for the duration of the cache lifetime it can result in
applications observing dirent attribute values that do not reflect the
current state of the files the entries name.

## Directory Enumeration With Uncacheable Dirent Metadata

In this scenario, the directory has the uncacheable dirent metadata
attribute set.  The sequence is otherwise identical to
{{fig-cached-dirents}}.

~~~
Application             NFSv4.2 Client        NFSv4.2 Server
-----------             --------------        --------------
readdir("/dir")
   |
   |                     READDIR, size and
   |                     time_modify requested
   |-------------------->------------------------>
   |                     entries:
   |                       a (size=100)
   |                       b (size=200)
   |<--------------------<------------------------
   |<-- names a, b

                                          (concurrent writer extends
                                           a from size=100 to
                                           size=500)

readdir("/dir")
   |
   |                     READDIR, size and
   |                     time_modify requested
   |                     (cached result not used)
   |-------------------->------------------------>
   |                     entries:
   |                       a (size=500)
   |                       b (size=200)
   |<--------------------<------------------------
   |<-- names a, b

stat("/dir/a")
   |                     (the size held from the
   |                      first READDIR is older
   |                      than the second and is
   |                      not reported)
   |<-- size=500
~~~
{: #fig-uncached-dirents title="Dirent Metadata Not Cached"}

In this case, {{fig-uncached-dirents}} shows the second readdir going
to the server, and the stat that follows reporting what that READDIR
returned.  The set of entries is unchanged between the two calls; only
the attribute value differs.  The client may still cache other
information, provided it reports no value for these entries that it
received before the READDIR that most recently returned them.

## Discussion

This example demonstrates that the uncacheable dirent metadata
attribute does not mandate a particular client implementation, but
it does require the behavior specified in {{sec_dirents}}.  The
attribute ensures that clients observe, for each entry, the file
attribute values the server reported in the READDIR that returned it,
in deployments where staleness of READDIR-returned attributes is known
to be a recurring problem.

# Implementation Status {#implementation-status}

Note to RFC Editor: please remove this section prior to publication.

There is a prototype Hammerspace server which implements the
uncacheable dirent metadata attribute and a prototype Linux client
which treats the attribute as an indication to go to the server for
each enumeration rather than serving one from a local cache.

In the prototype, directories whose contents change at the server
at a rate exceeding typical client cache lifetimes are marked with
the fattr4_uncacheable_dirent_metadata attribute.

The Linux client decodes the attribute in fs/nfs/nfs4xdr.c into a
per-inode flag (nfsi->uncacheable_dirent_metadata, declared in
include/linux/nfs_fs.h).  The readdir path in fs/nfs/dir.c consults it
twice, once for each half of the rule in {{sec_dirents}}.

For the first half, nfs_readdir() takes the -EBADCOOKIE path rather
than searching the cached readdir folios, so each call on that kernel
directory-read path is served by uncached_readdir() and reaches the
server; the start-of-directory case is included, so the first such call
for a marked directory goes to the
server too.

For the second half, nfs_use_readdirplus() returns true for a marked
directory where readdirplus is enabled for the mount, which selects
the larger attribute set that client requests for readdirplus.  The
commit message notes why: a request that carries only names "refreshes
names but leaves the entries' attributes to the inode attribute
caches", and because the cache-bypassing path never accrues the cache
usage that would otherwise enable the larger set, the continuation
requests of a large directory would be left "refreshing names but
serving stale per-entry attributes".  Requesting the larger set makes
each request refresh the entries' attribute caches, through
nfs_prime_dcache() and nfs_refresh_inode(), so a following stat
observes what that request returned.  Where readdirplus is disabled for
the mount, that client bypasses the readdir cache without requesting
the entries' attributes, and so meets the first half of the rule but
not the second.

That client is a partial implementation of the second half in a further
respect.  The rule in {{sec_dirents}} reaches every file attribute a
READDIR returns, while the attributes the prototype sets out to keep
fresh are the per-entry size and timestamps, and the set it names in
attr_request is a fixed list rather than everything the server
supports.  An attribute outside that list, held from before the
READDIR and reported afterwards, would satisfy what the prototype sets
out to do and not the rule as written.

The prototype server predates the object-type rule in {{sec_dirents}}:
it rejects a GETATTR of the attribute on a non-directory with
NFS4ERR_INVAL rather than returning FALSE, and the prototype client is
written never to ask for it except on directories.  Experience with an
earlier revision of that pair illustrates why the rule reads as it
does: when the client's OPEN bitmap was not type-gated, every
regular-file OPEN queried a directory-only attribute and the server's
error failed those opens broadly.  Neither prototype yet exercises the
FALSE rule.

Where the client does request the attributes, it satisfies the second
half by naming them in attr_request.  A client that named fewer would
have to obtain them after the READDIR instead; {{sec_dirents}} requires
only that the reported value not predate it.  Clients may employ more
sophisticated mechanisms, such as time-limited caches that revalidate
against the server, provided that the externally observable behavior
matches what {{sec_dirents}} requires.

The Linux client implementation encodes this attribute as a flag
distinct from the companion file-data attribute defined in
{{I-D.ietf-nfsv4-uncacheable-files}}; the two attributes are separated
as the two documents specify.  That implementation is posted to
linux-nfs (patches 4-6 of
<https://lore.kernel.org/linux-nfs/cover.1785140181.git.snitzer@kernel.org/>).

Experience with the prototype indicates that the attribute enables
servers to identify directories whose contents change faster than
typical NFSv4.2 client cache lifetimes can track, while remaining
compatible with existing NFSv4.2 semantics.

# XDR for the Uncacheable Dirent Metadata Attribute

~~~ xdr
///
/// typedef bool            fattr4_uncacheable_dirent_metadata;
///
/// const FATTR4_UNCACHEABLE_DIRENT_METADATA   = 88;
///
~~~

# Extraction of XDR

This document contains the external data representation (XDR)
{{RFC4506}} description of the uncacheable dirent metadata attribute.  The XDR
description is presented in a manner that facilitates easy extraction
into a ready-to-compile format. To extract the machine-readable XDR
description, use the following shell script:

~~~ shell
#!/bin/sh
grep '^ *///' $* | sed 's?^ */// ??' | sed 's?^ *///$??'
~~~

For example, if the script is named 'extract.sh' and this document is
named 'spec.txt', execute the following command:

~~~ shell
sh extract.sh < spec.txt > uncacheable_prot.x
~~~

This script removes leading blank spaces and the sentinel sequence '///'
from each line. XDR descriptions with the sentinel sequence are embedded
throughout the document.

Note that the XDR code contained in this document depends on types from
the NFSv4.2 nfs4_prot.x file (generated from {{RFC7863}}).  This includes
both nfs types that end with a 4, such as offset4, length4, etc., as
well as more generic types such as uint32_t and uint64_t.

While the XDR can be appended to that from {{RFC7863}}, the code snippets
should be placed in their appropriate sections within the existing XDR.

# Security Considerations

This attribute is not a security mechanism.  It addresses correctness
of client-side caching when client-cached dirent metadata
can become stale relative to the current state of the files those
entries name.  It does not change NFSv4.2 authentication or
authorization semantics, and it does not impose access controls on
the entries it describes.

Authorization to set or modify the fattr4_uncacheable_dirent_metadata
attribute is governed by existing NFSv4.2 authorization mechanisms.
This document does not define a new authorization model.

Because the attribute affects every honoring client, the cost of
setting it is not borne by the principal who sets it.  Setting it on a
directory means each honoring client reads that directory from the
server on every enumeration, and obtains the entries' attributes rather
than reusing held values -- by naming them in attr_request, as the
implementation in {{implementation-status}} does, or by fetching them
afterwards.  The resulting load grows with the number of honoring
clients and the rate at which they enumerate, neither of which the
setter controls.  Servers SHOULD therefore restrict modification of this
attribute based on administrative configuration, export policy, or
ownership, rather than allowing any principal who can write the
directory to set it.

The value of the attribute is readable by any client that can GETATTR
the directory.  It discloses which directories an operator considers
subject to rapid concurrent modification, which is a coarse signal about
workload rather than about file contents; deployments for which that is
sensitive can restrict read access to the directory by the mechanisms
NFSv4.2 already provides.

The attribute is writable, so an attacker able to modify a SETATTR in
flight could set or clear it and, through the load described above,
affect every honoring client of that directory.  The attribute is
carried in ordinary GETATTR and SETATTR operations, and this document
adds no protection of its own; it relies on the
security services NFSv4.2 already makes available.  RPCSEC_GSS can
"perform integrity checking on the entire RPC message" ({{RFC8881}}
Section 2.2.1.1.1.1), and the session reply cache provides exactly-once
semantics ({{RFC8881}} Section 2.10.6).  NFSv4.2 requires
implementations to support RPCSEC_GSS but, in the words of {{RFC8881}}
Section 2.2.1.1, "this requirement to implement is not a requirement to
use", so a deployment needing the attribute's value protected in
transit selects a security flavor that provides integrity.  The same
applies to the replies a client acts on: a forged GETATTR or READDIR
reply carrying a true value imposes the additional traffic on that
client,
and one carrying false silently restores the caching this attribute
exists to prevent.

This attribute does not change the semantics of sec_label or the
enforcement of MAC security policies; where a READDIR returns
sec_label, the rule in {{sec_dirents}} governs when a held value may be
reported for an entry, as it does for any other attribute.  A client's
obligations under Labeled NFS (see
{{RFC7204}} for background, and {{RFC7862}} Section 9 for the NFSv4.2
mechanism) are the same whether dirent metadata is refetched or served
from a cache.

Setting or clearing the attribute changes how an honoring client
obtains the entries and their attributes, and nothing else.  It does
not alter which principals may read the directory or its entries, nor
what those entries reveal.

# IANA Considerations

This document has no IANA actions.

NFSv4.2 attribute numbers are assigned by working group coordination
rather than through an IANA registry.  This document uses attribute
number 88, chosen alongside attribute number 87 in
{{I-D.ietf-nfsv4-uncacheable-files}}.

--- back

# Acknowledgments
{:numbered="false"}

Trond Myklebust, Mike Snitzer, Jon Flynn, Keith Mannthey, and Thomas
Haynes all worked on the prototype at Hammerspace.

Rick Macklem, Chuck Lever, Dave Noveck, Sorin Faibish, Christoph
Hellwig, and Jeff Layton reviewed the document.

Chris Inacio, Brian Pawlowski, Chuck Lever, Zahed Sarker, and
Gorry Fairhurst helped guide this process.
