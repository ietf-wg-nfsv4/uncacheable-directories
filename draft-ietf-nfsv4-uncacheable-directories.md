---
title: Adding an Uncacheable Directory-Entry Metadata Attribute to NFSv4.2
abbrev: Uncacheable Dirents
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
  RFC8275:
  RFC8276:
  MS-ABE:
    title: Access-Based Enumeration (ABE) Concepts
    author:
      org: Microsoft
    target: https://techcommunity.microsoft.com/blog/askds/access-based-enumeration-abe-concepts-part-1-of-2/400435
    date: May 2009
  MS-SMB2:
    title: Server Message Block (SMB) Protocol Versions 2 and 3
    author:
      org: Microsoft Corporation
    seriesinfo:
      Microsoft: MS-SMB2
    target: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/
--- abstract

Network File System version 4.2 (NFSv4.2) clients may cache the
file attributes returned by READDIR alongside each directory
entry.  This caching is inherently best-effort: those attributes
belong to the underlying files and change when the files are
written, which the directory's change attribute does not track.
In some deployments the rate of file writes by other clients
makes such caching produce incorrect size and timestamp values
often enough to be a deployment problem.  This document introduces
an uncacheable dirent metadata attribute for NFSv4.2 that allows
a server to identify a directory for which an honoring client is
required to retrieve directory-entry metadata from the server on
each READDIR rather than serving the response from a local cache.

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
dirent metadata attribute applies to the caching of directory-entry
metadata, including names and associated file object metadata such
as size and timestamps.  It does not prohibit caching of the
directory object itself, nor does it affect caching of file data.

Client implementations may cache the file attributes returned by
READDIR alongside the dirents themselves.  This caching is
inherently best-effort: the file attributes can change at any time
as a result of writes to the underlying files, which the directory's
change attribute does not track.  When this caching returns stale
size and timestamp information for files that have been modified
concurrently, it also undermines the effectiveness of uncacheable
file data semantics ({{I-D.ietf-nfsv4-uncacheable-files}}) in the
same deployment.  This can lead to applications observing
inconsistent metadata and data views even when file data caching
is disabled.

This document introduces the uncacheable dirent metadata attribute
to NFSv4.2 to allow servers to advise clients that caching of
directory-entry metadata is unsuitable.  Using the process detailed
in {{RFC8178}}, the revisions in this document become an extension
of NFSv4.2 {{RFC7862}}.  They are built on top of the external data
representation (XDR) {{RFC4506}} generated from {{RFC7863}}.

# Deployment Motivation {#deployment-motivation}

A class of deployment exposes a single file namespace concurrently
through multiple file-access protocols, typically including
NFSv4.2, NFSv3, and the Server Message Block (SMB) protocol family
{{MS-SMB2}}.  The same directories are reachable from all of those
protocols and, in many such deployments, from server-side policy
components (placement, replication, archival, tiering) that act on
the namespace without a client connection.  Changes to a directory's
contents -- entries added, entries removed, entries whose attributes
have been updated -- may therefore originate at any time from any
of those sources.

NFSv4.2 client implementations typically cache READDIR responses for
a period bounded by either the directory's change attribute or a
heuristic timeout.  The cached file attributes returned by READDIR
are NOT invalidated by the directory's change attribute -- they are
only invalidated by writes to the underlying files, which the
directory's change attribute does not track.  In deployments of the
kind described above, the rate at which the underlying files are
written by other clients can exceed what such caches can track, so
that an NFSv4.2 client serving READDIR responses from its local cache
will, with some regularity, return file attribute values that no
longer reflect the current state of those files at the server.

The fattr4_uncacheable_dirent_metadata attribute is the server's
mechanism to identify a directory for which this risk is high
enough that client-side caching is not safe.  When the server sets
the attribute on a directory, an honoring client retrieves
directory-entry metadata from the server on each READDIR rather
than from a local cache.

In the deployments that motivate this work, no attribute values
returned for a given dirent vary across users.  The server returns
the same set of entries with the same attribute values to all NFSv4
clients of a given directory regardless of the requesting user's
identity.

The SMB protocol family includes a related concept called
Access-Based Enumeration (ABE) {{MS-ABE}} {{MS-SMB2}}, in which an
SMB server filters the visible set of directory entries by the
requesting user's access rights.  ABE is named here as background
for readers familiar with the SMB ecosystem.  The NFSv4.2 attribute
defined in this document is not an NFS surface for ABE: it does not
require, specify, or depend on per-user directory filtering, and
the servers that motivate this work do not apply such filtering to
NFSv4.2 READDIR responses.

# Definitions

Access Based Enumeration (ABE)

: A Server Message Block (SMB) {{MS-SMB2}} feature in which the
server filters the visible set of directory entries by the
requesting user's access rights {{MS-ABE}}.  Used in this document
only as background for readers familiar with the SMB ecosystem;
the NFSv4.2 attribute defined here is not an NFS surface for ABE.

dirent

: A directory entry returned by READDIR -- the (name, file handle)
pair that names a file or subdirectory within a directory.  A dirent
itself does not include the file attributes returned alongside it.

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

: A client-side cache of READDIR results -- the (name, file handle,
file attributes) tuples -- used to avoid repeated READDIR and GETATTR
traffic.  Because the file attributes in a cached READDIR response are
not invalidated by the directory's change attribute (only by writes to
the underlying files), this caching is inherently best-effort and
subject to staleness whenever the underlying files are modified.

uncacheable dirent metadata attribute

: An NFSv4.2 file attribute that advises clients not to cache
  directory-entry metadata associated with file objects, including
  names, size, and timestamps.

This document assumes familiarity with NFSv4.2 operations, attributes,
and error handling as defined in {{RFC8881}} and {{RFC7862}}.

# Requirements Language

{::boilerplate bcp14-tagged}

# Caching of Directory-Entry Metadata

The uncacheable dirent metadata attribute enables servers to identify
directories where the staleness of cached READDIR attributes is
particularly likely and particularly damaging.  It is an OPTIONAL
attribute to implement for NFSv4.2.  If both the client and the
server support this attribute, and the attribute is set on a
directory, the client MUST retrieve directory-entry metadata from
the server on each READDIR rather than serving the response from
a local cache.

This document specifies the required observable behavior rather
than mandating a particular internal implementation strategy.
Clients MAY employ more sophisticated mechanisms, such as
time-limited caches that revalidate against the server on each
READDIR, provided that the externally visible behavior is
equivalent to retrieving directory-entry metadata from the
server on each READDIR.

Allowing clients to set this attribute provides a portable mechanism
to request that directory-entry metadata not be cached, without
requiring changes to application behavior or out-of-band administrative
configuration.

A client can determine whether the uncacheable dirent metadata attribute
is supported for a given directory by examining the supported_attrs
attribute for that directory's filesystem or by probing support using
the procedures described in {{RFC8178}}.

The only way that the server can determine that the client supports
the attribute is if the client sends either a GETATTR or a SETATTR
with the uncacheable dirent metadata attribute.

The uncacheable dirent metadata attribute governs caching behavior of
directory-entry metadata returned by READDIR and related operations,
not the directory object itself.

The uncacheable dirent metadata attribute addresses a different
aspect of client-side caching than fattr4_uncacheable_file_data
({{I-D.ietf-nfsv4-uncacheable-files}}). The file data attribute
governs caching of file contents, while the dirent metadata attribute
governs caching of directory-entry metadata returned by READDIR and
related operations. The attributes are independent and may be used
separately.

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

The attribute does NOT make READDIR-attr caching reliable for
directories where it is not set.  Clients MUST NOT interpret the
absence of fattr4_uncacheable_dirent_metadata, or its value being
false, as a guarantee that cached READDIR attributes are
authoritative.  As stated in {{RFC8881}} Section 10.3.2, all
client-cached attributes are subject to staleness; the attribute
defined in this document only identifies directories for which
staleness is particularly likely and particularly damaging.

This attribute does not define behavior for positive or negative name
caching or for caching of LOOKUP results outside the scope of
directory-entry metadata returned by READDIR and related operations.

Directory delegations grant a client exclusive caching rights subject
to server recall.  In deployments where directory contents change at
a rate that makes per-change recall impractical, a directory delegation
does not provide the always-refetch semantics defined by the uncacheable
dirent metadata attribute.  These mechanisms are independent.

Clients MUST NOT assume that directory-entry metadata is valid beyond
the READDIR that produced it.

## Uncacheable Directory-Entry Metadata {#sec_dirents}

The fattr4_uncacheable_dirent_metadata attribute is a read-write boolean
attribute that applies to directory objects.
Authorization to query or modify this attribute is governed by
existing NFSv4.2 authorization mechanisms.

If a directory object has the uncacheable dirent metadata attribute
set, the client MUST retrieve directory-entry metadata from the
server on each READDIR rather than serving the response from a
local cache.  This ensures that the returned metadata reflects
the current state of the directory as determined by the server.

The uncacheable dirent metadata attribute does not modify the
semantics of the NFSv4.2 change attribute.  Clients MUST continue to
use the change attribute to detect directory modifications and to
determine when directory contents may have changed, even when
directory-entry metadata caching is suppressed.  Suppressing caching
of directory-entry metadata does not remove the need for change-based
validation.

Servers SHOULD assume that clients which do not query or set this
attribute may cache directory-entry metadata, and therefore SHOULD
NOT rely on this attribute for correctness unless client support
is confirmed.

A directory delegation grants a client the right to cache
directory-entry metadata until the server recalls the delegation.
The always-refetch rule of this attribute is incompatible with that
grant.  If a directory has both the uncacheable dirent metadata
attribute set and an outstanding directory delegation, the server
MUST recall the delegation, after which the client follows the
always-refetch rule on each subsequent READDIR.  A server MUST NOT
grant a new directory delegation on a directory while the
uncacheable dirent metadata attribute is set on that directory.

# Example: Directory Enumeration With and Without Dirent Metadata Caching

This example illustrates the difference in client-visible behavior when
directory-entry metadata caching is enabled versus when the uncacheable
dirent metadata attribute is set on a directory.  In both scenarios,
the set of entries in the directory does not change between the two
calls; an attribute value of one entry is updated at the server
between calls.  The difference is whether the second call observes
the updated attribute value.

## Classic Directory Enumeration (Directory-Entry Metadata Cached)

In this scenario, the client caches directory-entry metadata obtained
from the server and reuses it for the second readdir.

~~~
Application             NFSv4.2 Client        NFSv4.2 Server
-----------             --------------        --------------
readdir("/dir")
   |
   |                     READDIR
   |-------------------->------------------------>
   |                     entries:
   |                       a (size=100)
   |                       b (size=200)
   |                       c (size=300)
   |<--------------------<------------------------
   |
(entries cached in client)

                                            (concurrent writer extends
                                             a from size=100 to
                                             size=500)

readdir("/dir")
   |
   |                     (no network traffic)
   |                     entries returned from
   |                     client cache:
   |                       a (size=100)
   |                       b (size=200)
   |                       c (size=300)
~~~
{: #fig-cached-dirents title="Directory-Entry Metadata Cached"}

In this case, {{fig-cached-dirents}} shows that directory-entry
metadata retrieved by the first READDIR is reused to satisfy the
second.  The cached response reflects entry a's size as it was
at the time of the first call; it does not reflect the update
that occurred at the server between calls.  This behavior is
typical of legacy NFSv4.2 clients and maximizes performance, but
it can result in applications observing dirent attribute values
that do not reflect the current state of the server.

## Directory Enumeration With Uncacheable Dirent Metadata

In this scenario, the directory has the uncacheable dirent metadata
attribute set.  The client retrieves directory-entry metadata from
the server on each READDIR.

~~~
Application             NFSv4.2 Client        NFSv4.2 Server
-----------             --------------        --------------
readdir("/dir")
   |
   |                     READDIR
   |-------------------->------------------------>
   |                     entries:
   |                       a (size=100)
   |                       b (size=200)
   |                       c (size=300)
   |<--------------------<------------------------
   |
(no directory-entry metadata retained)

                                            (concurrent writer extends
                                             a from size=100 to
                                             size=500)

readdir("/dir")
   |
   |                     READDIR
   |-------------------->------------------------>
   |                     entries:
   |                       a (size=500)
   |                       b (size=200)
   |                       c (size=300)
   |<--------------------<------------------------
~~~
{: #fig-uncached-dirents title="Directory-Entry Metadata Not Cached"}

In this case, {{fig-uncached-dirents}} shows that each readdir
request results in a READDIR operation sent to the server, so
the second call observes the updated size of entry a.  The set
of entries returned is unchanged between calls; only the
attribute value differs.  The client may still cache other
information, provided the externally observable behavior is
equivalent to retrieving directory-entry metadata from the
server on each READDIR.

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
per-inode flag (nfsi->uncacheable).  The readdir path in fs/nfs/dir.c
consults this flag to skip the readdir cache and refetch from the
server on each readdir call.  Clients may employ more sophisticated
mechanisms, such as time-limited caches that revalidate against the
server on each READDIR, provided that the externally observable
behavior matches the always-refetch semantics described in this
document.

At the time of writing, the prototype encodes the attribute on the
wire alongside the companion file-data attribute defined in
{{I-D.ietf-nfsv4-uncacheable-files}} as a single shared flag.  A
follow-on prototype branch separates the two attributes as the
two documents specify.

Experience with the prototype indicates that the attribute enables
servers to identify directories whose contents change faster than
typical NFSv4.2 client cache lifetimes can track, while remaining
compatible with existing NFSv4.2 semantics.

# XDR for Uncacheable Dirents Attribute

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
of client-side caching when client-cached directory-entry metadata
can become stale relative to the current state of the directory at
the server.  It does not change NFSv4.2 authentication or authorization
semantics, and it does not impose access controls on the entries it
describes.

Authorization to set or modify the fattr4_uncacheable_dirent_metadata
attribute is governed by existing NFSv4.2 authorization mechanisms.
Servers MAY restrict modification of this attribute based on local
policy, file ownership, or access control rules.  This document does
not define a new authorization model.

If the client supports Labeled NFS (see {{RFC7204}} for background),
the client MUST locally enforce the MAC security policies defined by
NFSv4.2 ({{RFC7862}}, Section 9).  This obligation is independent of
whether directory-entry metadata is cached or refetched.

The uncacheable dirent metadata attribute allows servers to indicate
that directory-entry metadata should not be assumed to remain valid
beyond the READDIR that produced it.

# IANA Considerations

This document has no IANA actions.  Numbered FATTR4 attributes for
NFSv4 are assigned by WG coordination through IETF publication
rather than by an IANA registry, consistent with the assignment of
attributes 81 and 82 in {{RFC8275}} and {{RFC8276}}.  Attribute
number 88 was selected on the NFSv4 working group mailing list in
conjunction with the assignment of number 87 in
{{I-D.ietf-nfsv4-uncacheable-files}}.

--- back

# Acknowledgments
{:numbered="false"}

Trond Myklebust, Mike Snitzer, Jon Flynn, Keith Mannthey, and Thomas
Haynes all worked on the prototype at Hammerspace.

Rick Macklem, Chuck Lever, and Dave Noveck reviewed the document.

Chris Inacio, Chuck Lever, Brian Pawlowski, and Gorry Fairhurst
helped guide this process.
