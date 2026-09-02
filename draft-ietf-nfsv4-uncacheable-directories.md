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
file attributes returned by READDIR alongside each directory
entry.  Such a cache is not invalidated by the directory's change
attribute, which reflects changes to the directory and its entries
but not writes to the files those entries name, so it can become
stale when another client changes one of those files.  In some
deployments this produces incorrect size and timestamp values often
enough to be a deployment problem.  This document introduces an
uncacheable dirent metadata attribute for NFSv4.2 that allows a
server to identify a directory for which an honoring client goes to
the server for each enumeration, and does not report an entry's
attributes from a value it held before that READDIR.

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
This caching is inherently best-effort -- writes to the underlying
files can change those attributes at any time, and the directory's
change attribute does not track such writes.  In some deployments
the cost of that staleness is high enough to be a deployment
problem; the conditions are described in {{deployment-motivation}}.

In this document, the term directory is used to describe the
context in which directory entries are retrieved.  The uncacheable
dirent metadata attribute applies to the caching of dirent
metadata -- the file object attributes, such as size and timestamps,
returned alongside each entry.  It does not prohibit caching of the
directory object itself, nor does it affect caching of file data.

When this best-effort caching returns stale size and timestamp
information for concurrently modified files, it also undermines the
effectiveness of uncacheable file data semantics
({{I-D.ietf-nfsv4-uncacheable-files}}) in the same deployment:
applications can observe inconsistent metadata and data views even
when file data caching is disabled.

This document introduces the uncacheable dirent metadata attribute
to NFSv4.2 to allow servers to advise clients that caching of
dirent metadata is unsuitable.  Using the process detailed
in {{RFC8178}}, the revisions in this document become an extension
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

The staleness has correctness consequences, not merely cosmetic ones.
An incremental backup or an rsync scan that decides what to copy from
the size and time_modify reported for each entry will silently skip a
file whose cached metadata predates a concurrent write, leaving data
uncopied.  This attribute lets a server mark the directories where that
outcome is likely, so that an honoring client fetches current metadata
on each enumeration.

The fattr4_uncacheable_dirent_metadata attribute is the server's
mechanism to identify a directory for which this risk is high
enough that client-side caching is not safe.  When the server sets
the attribute on a directory, an honoring client retrieves
dirent metadata from the server on each READDIR rather
than from a local cache.

# Definitions

dirent

: A directory entry -- the (name, fileid) pair that names a file or
subdirectory within a directory.  This is what a client maintains for
an entry, whatever a given READDIR response carries on the wire; it is
the pair POSIX exposes as d_name and d_ino.  A dirent itself does not
include the file attributes returned alongside it.

dirent metadata

: The file attributes (size, mtime, ctime, atime, mode, owner, etc.)
returned in a READDIR response alongside each dirent.  These attributes
belong to the underlying file object, not to the directory; they
change when the underlying file is written, which is independent of
the directory's change attribute.  The term "dirent metadata" in this
document is a naming convenience for "the file attributes that arrive
in a READDIR response"; it does not assert that those attributes
inherit the directory's cache-coherence semantics.

dirent caching

: A client-side cache of the dirents themselves -- the (name, fileid)
pairs -- used to avoid repeated READDIR traffic.  Whether such a cache
remains valid is governed by the directory's change attribute: the
directory changes when an entry is created, removed, or renamed, and a
fileid is stable for as long as its entry names the same object.  A
fileid is therefore cached with the name rather than with the file
attributes: writes to a file change its size and timestamps without
touching either the name or the fileid.  This document does not change
what a client may hold in this cache or how it validates it; for a
directory on which the attribute is set, however, each enumeration is
satisfied by a READDIR, so for that directory the cache no longer
avoids that traffic.

dirent metadata caching

: A client-side cache of the dirent metadata returned alongside those
entries, used to avoid repeated GETATTR traffic.  Because those file
attributes are not invalidated by the directory's change attribute
(only by writes to the underlying files), this caching is inherently
best-effort and subject to staleness whenever the underlying files are
modified.  This is the caching that the attribute defined in this
document constrains.

uncacheable dirent metadata attribute

: An NFSv4.2 file attribute that advises clients not to cache
  dirent metadata associated with file objects, such as
  size and timestamps.

honoring client

: A client that implements this attribute and enforces the
always-refetch behavior it defines for a directory on which the
attribute is set.  The attribute is advisory: a client that does not
implement it, or that declines to enforce it, is non-honoring and may
continue to cache dirent metadata.

This document assumes familiarity with NFSv4.2 operations, attributes,
and error handling as defined in {{RFC8881}} and {{RFC7862}}.

# Requirements Language

{::boilerplate bcp14-tagged}

# Caching of Dirent Metadata

The uncacheable dirent metadata attribute constrains what a READDIR on
a particular directory does: it directs an honoring client to fetch the
entries' file attributes from the server rather than serve them from a
local cache.  It constrains dirent metadata caching only: an honoring
client may continue to cache the dirents themselves, validated by the
directory's change attribute as it would be for any other directory,
and refetch only their metadata.  This matches how clients are
typically built, with the dirent cache maintained separately from the
attributes obtained for the objects the entries name.
A server sets it on the directories where it knows the
staleness of cached READDIR attributes is particularly likely and
particularly damaging.  It is a RECOMMENDED
attribute for NFSv4.2, in the attribute-category sense of {{RFC8881}}
Section 5.2 and {{RFC7862}} Section 12 rather than the BCP 14 sense; a
server is not required to support it.  If both the client and the
server support this attribute, and the attribute is set on a
directory, an honoring client goes to the server for each enumeration
and does not report an entry's attributes from a value it held before
that READDIR.  {{sec_dirents}} states the requirement normatively.

Because the attribute governs a READDIR rather than the objects the
entries name, it makes no claim about those objects.  A file reached
through a directory on which the attribute is not set is unaffected,
including where the same file is linked into both a directory on which
it is set and one on which it is not.

This document specifies the required observable behavior rather
than mandating a particular internal implementation strategy.
Clients MAY employ more sophisticated mechanisms, such as
time-limited caches that revalidate against the server on each
READDIR, provided that the externally visible behavior is
equivalent to retrieving dirent metadata from the
server on each READDIR.

A client can determine whether the uncacheable dirent metadata attribute
is supported for a given directory by examining the supported_attrs
attribute for that directory's filesystem or by probing support using
the procedures described in {{RFC8178}}.

The uncacheable dirent metadata attribute governs the client's
caching of READDIR responses for the directory.  It does NOT govern:

* The client's per-file attribute cache for individual children of
  the directory, populated by direct GETATTR (for example, following
  a LOOKUP).  Such caches are governed by the attribute-cache
  mechanisms already defined by NFSv4.2 and are subject to the same
  staleness from concurrent writes; clients in deployments using
  this attribute may wish to apply correspondingly short cache
  lifetimes to per-file attributes for children of the directory,
  but the present attribute does not require this.

* The directory's own attribute cache.  The directory object's own
  attributes (mode, owner, etc.) can be cached normally and
  revalidated via the directory's change attribute as usual.

* Operations that do not return file attributes in their response
  (for example, LOOKUP without a following GETATTR, ACCESS).  These
  are unaffected.

The uncacheable dirent metadata attribute addresses a different
aspect of client-side caching than fattr4_uncacheable_file_data
({{I-D.ietf-nfsv4-uncacheable-files}}).  The file data attribute
governs caching of file contents, while the dirent metadata
attribute governs caching of file attributes returned by READDIR.
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
  instruct an honoring client to suppress the caching for those
  specific objects.

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
file attributes that arrive alongside them, and this attribute affects
only the latter.

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
many notifications to decline to delegate it); and the
dirent_notif_delay attribute lets a server bound or refuse
child-attribute notification, so a client cannot rely on notification
for freshness.

## Uncacheable Dirent Metadata {#sec_dirents}

The fattr4_uncacheable_dirent_metadata attribute is a read-write boolean
attribute that applies to directory objects.
Authorization to query or modify this attribute is governed by
existing NFSv4.2 authorization mechanisms.

The attribute applies only to directory objects.  A server that
receives a GETATTR requesting fattr4_uncacheable_dirent_metadata on an
object that is not a directory MUST return FALSE: support for an
attribute is advertised per file system ({{RFC8881}} Section 5.8.1.1),
so a server that supports this attribute supports it for every object
in that file system and owes a value for each ({{RFC8881}} Section
18.7.3).  As with rawdev ({{RFC8881}} Section 5.8.2.31), the value
SHOULD NOT be considered useful for an object the attribute does not
describe.  A server that receives a SETATTR requesting
fattr4_uncacheable_dirent_metadata on an object that is not a directory
MUST return NFS4ERR_WRONG_TYPE ({{RFC8881}} Section 15.1.2.9).

This attribute is set per directory.  This document does not define
propagation of the attribute to subdirectories created within a
directory on which it is set; any such inheritance is a matter of
local server policy.

If a directory object has the uncacheable dirent metadata attribute
set, an honoring client MUST NOT satisfy a readdir of that directory
from READDIR results obtained during a different enumeration, and MUST
NOT report, for an entry, a value of a dirent metadata attribute that
it received before the READDIR that most recently returned that entry.
An honoring client therefore either names the attributes it will report
in that READDIR's attr_request, or obtains them afterwards; a value it
held beforehand is not usable for that entry.

In this document a readdir is a directory-read request made by an
application, however the client's interface batches entries, and an
enumeration is one pass over a directory: the readdirs of that pass and
the READDIRs the client issues to satisfy them.  A READDIR ordinarily
supplies enough entries for many readdirs, so the requirement above is
scoped to the enumeration a readdir belongs to and not to a READDIR per
readdir.

Entries and metadata carried by the READDIRs of a single enumeration
MAY be retained until that enumeration completes.  {{RFC8881}} Section
10.8.2 requires such a cache to be a consistent snapshot of directory
contents, validated by the directory's change attribute; because that
attribute does not move when a file the directory names is written, it
provides no corresponding guarantee for the entries' file attributes,
which are as of the READDIR that carried them.

The uncacheable dirent metadata attribute does not modify the
semantics of the NFSv4.2 change attribute.  Clients MUST continue to
use the change attribute to detect directory modifications and to
determine when directory contents may have changed, even when
dirent metadata caching is suppressed.  Suppressing caching
of dirent metadata does not remove the need for change-based
validation.

This attribute is advisory, so servers SHOULD NOT rely on it for
correctness: a client that does not implement it, or that declines to
enforce it, may continue to cache dirent metadata.  A server cannot
distinguish those clients from honoring ones.  Observing a GETATTR or
a SETATTR of the attribute shows only that a client knows the
attribute exists, not that it enforces the always-refetch rule, so
such a request is not a basis for treating that client's caching as
suppressed.

A directory delegation would let a client serve dirent
metadata from its cache without refetching, which is incompatible with
the always-refetch rule this attribute defines.  Accordingly, if a
directory has the uncacheable dirent metadata attribute set and an
outstanding directory delegation, the server MUST recall the
delegation, after which the client follows the always-refetch rule on
each subsequent readdir.  A server MUST NOT grant a new directory
delegation on a directory while the uncacheable dirent metadata
attribute is set on that directory.

# Example: Directory Enumeration With and Without Dirent Metadata Caching

This example illustrates the difference in client-visible behavior when
dirent metadata caching is enabled versus when the uncacheable
dirent metadata attribute is set on a directory.  In both scenarios
each readdir("/dir") is a separate enumeration -- the application opens
the directory, reads it to end-of-file and closes it, and opens it
again for the second.  The set of entries does not change between the
two; an attribute value of one entry is updated at the server between
them.  The difference is whether the stat after the second readdir
observes the updated value.

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
attribute set.  The client retrieves dirent metadata from
the server on each READDIR.

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
it does require the always-refetch behavior specified in
{{sec_dirents}}.  The attribute ensures that NFSv4.2 clients observe
file attribute values reflecting the current state of the server in
deployments where staleness of READDIR-returned attributes is known
to be a recurring problem.

# Implementation Status

Note to RFC Editor: please remove this section prior to publication.

There is a prototype Hammerspace server which implements the
uncacheable dirent metadata attribute and a prototype Linux client
which treats the attribute as an indication to retrieve directory-
entry metadata from the server on each READDIR rather than from a
local cache.

In the prototype, directories whose contents change at the server
at a rate exceeding typical client cache lifetimes are marked with
the fattr4_uncacheable_dirent_metadata attribute.

The Linux client decodes the attribute in fs/nfs/nfs4xdr.c into a
per-inode flag (nfsi->uncacheable_dirent_metadata, declared in
include/linux/nfs_fs.h).  The readdir path in fs/nfs/dir.c consults
this flag to skip the readdir cache and refetch from the
server on each readdir call.  Clients may employ more sophisticated
mechanisms, such as time-limited caches that revalidate against the
server on each READDIR, provided that the externally observable
behavior matches the always-refetch semantics described in this
document.

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
can become stale relative to the current state of the directory at
the server.  It does not change NFSv4.2 authentication or authorization
semantics, and it does not impose access controls on the entries it
describes.

Authorization to set or modify the fattr4_uncacheable_dirent_metadata
attribute is governed by existing NFSv4.2 authorization mechanisms.
Servers MAY restrict modification of this attribute based on local
policy, file ownership, or access control rules.  This document does
not define a new authorization model.

Because the attribute is visible to and affects the caching behavior
of all honoring clients, servers should consider the implications of
allowing unprivileged users to set or clear it.  Setting the attribute
on a directory forces honoring clients to abandon READDIR caching and
refetch dirent metadata on every enumeration, which can
increase load on the server and on other clients.  A server MAY
restrict modification of the attribute based on administrative
configuration, export policy, or ownership.

This attribute does not change the semantics of sec_label or the
enforcement of MAC security policies.  A client's obligations under
Labeled NFS (see {{RFC7204}} for background, and {{RFC7862}} Section 9
for the NFSv4.2 mechanism) are the same whether dirent metadata is
refetched or served from a cache.

The uncacheable dirent metadata attribute allows servers to indicate
that dirent metadata should not be assumed to remain valid
beyond the READDIR that produced it.

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
