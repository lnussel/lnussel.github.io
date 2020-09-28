---
layout: post
title:  "Zero copy RPM updates"
date: 2020-07-07
---
## Intro

A basic building block of (open)SUSE's operating systems are RPM packages.
The methods to install and update those packages were perfected over two
decades and are considered mature. With todays copy-on-write file systems,
their snapshotting capabilities and ubiquitous internet resp network access
there is potential to speed things up by rethinking and tweaking some of those
methods.

## Goals

For installing and updating packages this proposal wants to have

- Data only written once to conserve flash memory
- Less CPU cycles when applying package updates compared to delta
  RPMs
- No precomputed Deltas to be able to update from any version of
  installed packages
- Less bandwidth for downloads
- Means less overall wall clock time needed

At the same time data needs to be deliverable via HTTP with no
software on special server. This is required to be able to leverage
the existing CDN resp mirror infrastructure.

## Current State

### Storage on server side

Each new revision of an RPM package is stored as individual file on
server side. The payload of RPM packages is compressed.

That means

- Even packages that were only rebuild to sync the release counter
  are separate files that have to be stored and mirrored
  individually.
- Rsync does not help to reduce bandwidth when mirroring as files
  get new names and the payload is compressed, hiding equal blocks.

### Delta RPMs

In updated RPM packages, most of the data in the payload actually
does not change. For that reason delta RPMs were invented. The build
system precomputes the differences between two given versions of a
package to produce a binary delta. The binary delta can then be used
to reconstruct the original RPM by taking data from the installed
system. The result is an RPM that is 100% identical the the full
download. That RPM can then be processed by the regular rpm tool.

That means

- deltas can only be generated between predetermined versions of a
  package
- with the exception of config files, all data of the old RPM needs
  to exist unmodified on the target system to be able to reconstruct
  the new RPM. Means options like --exclude-docs break deltas.
- the computational effort to apply a delta and compress the new
  payload are high. The created RPM needs to be stored just be
  decompressed and unpackaged again.
- on build system and server creating the deltas also costs
  resources. They have to be stored in addition to several versions
  of the actual full RPMs.

So in other words. The build system takes file system content, compresses it to
produce an rpm package. The makedeltarpm tool takes two of those, decompresses
them, compares the content and produces a compressed delta rpm. The
applydeltarpm tool then takes the delta rpm, decompresses it,
applies the delta information with information based on files from
the system. The so created payload is compressed again to write an
rpm package. Then upon actual installation the rpm command
decompresses the rpm to write the content to the file system.
So payload gets compressed three times and decompressed three times.

### RPM installation

In order to install or update an RPM package, the target system has
to download the RPMs in question. In order to prevent
inconsistencies due to unreliable internet connections usually all
packages are downloaded prior to installing them.

Installing packages means the rpm tool decompresses the payload and
copies the data of the files parallel to the already existing ones.
After all files are placed on disk, they are renamed to their final
name.

That means:

- intermediate extra storage space is needed to store both the
  downloaded package
- storing the data after decompression also requires extra space and
  writes.
- rpm replacing files that didn't actually change increases write
  cycles on the storage. An upstream setting to mitigate that to
  some degree is already available in Factory though[^1].

## Ideas

### Optimizing Server Side Storage

RPM payload could be seen as a serialized representation of a file
system tree:

![rpm](/images/2020-07-07-rpm-delta-updates/rpm.png "RPM payload")

Every such represenation is stored seperately. Even if the data of
one file just changed slightly, the whole payload would have to be
stored:

![rpm2](/images/2020-07-07-rpm-delta-updates/rpm3.png "Two files with similar payload")

Instead of looking at the payload as a concatenation of files, one
could also look at it as a concatenation of data chunks. Instead of
a file name those chunks could be identified by a hash of the data
they contain. So the payload would turn into an index of data
chunks.

![rpm2](/images/2020-07-07-rpm-delta-updates/rpm2.png "Two files with similar payload")

That's what
[casync](http://0pointer.net/blog/casync-a-tool-for-distributing-file-system-images.html)
does. It uses [content defined
chunking](https://moinakg.wordpress.com/2013/06/22/high-performance-content-defined-chunking/)
to determine the chunks dynamically.

That means if RPMs had uncompressed payload, one could actually
store only the data chunks, plus an index that tells casync how to
stitch the chunks together to recreate the original rpm.
So when the server has to keep several versions of an only slightly
changed package this method would save space.

Storing chunks in separate files still allows to fetch them via HTTP
GET request and allows mirroring of chunks.

### Reassembling an RPM on client side

With the aforementioned storage on the server, a client could
download the index file, followed by all required data chunks to
recreate the rpm locally. Since most files didn't change a lot
anyways, the majority of data is basically already available
locally. So the client can borrow those chunks without having to
download them. For this method it does not matter if a non-config
file got modified locally behind rpm's back. So the delta download
would be dynamic based on what data actually changed locally.

![rpm-rsyncfromserver](/images/2020-07-07-rpm-delta-updates/rpm-syncfromserver.png "Downloading from the server")

At this point the client would have a regular RPM file that could be
installed using rpm as usual. The download handling could be
implemented in the layer on top ie zypp or dnf.

A slight improvement to reduce disk writes could be made using the
[%_minimize_writes](https://github.com/openSUSE/rpm-config-SUSE/commit/8dcfe7b89abddeb2d3ed32046338c82cce9c306d)
setting.

On filesystems that support it, reflinks<sup>[1](#footnote1)</sup> could be used for
preparing the new rpm payload without actually copy data.

### Tuning rpm

Installing an rpm so far would still be the traditional way, ie at
least changed files would have to be copied.

On filesystems that support it, rpm could be extended to use
reflinks<sup>[1](#footnote1)</sup> preparing the new rpm payload without actually copying
data out of the rpm file. So basically the reverse operation of
assembling the new rpm.

![rpm-install1](/images/2020-07-07-rpm-delta-updates/rpm-install1.png "RPM installation using reflinks")

#### Dealing with the RPM header.

The RPM header can be quite big and wasn't discussed so far.
Normally the rpm header of installed packages is added to a
database. Means it's copied also. Database APIs abstract away where
the actual data is stored, so using the reflink method won't work to
avoid duplication.

However, the rpm header could simply be considered part of the
installed files of a package. For example by explicitly or
implicitly having a %ghost entry for the header in the file list. Eg
/usr/lib/sysimage/rpm/%{NEVRA}. The RPM package installation could
install the header of a package there. So presence of the header
would tell that a package is installed.

A side effect of that would be that no "database" is needed anymore.
The canonical source of truth would be installed headers.
For efficient rpm queries etc still some cached indexes are
required.

### Package manager integration

![zypp-integration](/images/2020-07-07-rpm-delta-updates/zypp-integration.png "file system with zypp")

Assuming the package manager is zypp, it would create the
to-be-installed rpm packages in /var/cache/zypp and then reflink the
content into the system. However, since /var/cache is normally a
separate subvolume, it would also be a mountpoint. Therefore
reflinks can't cross mountpoints.

To make that work, there are two options:

1. use a temporary mount of the top level subvolume and operate on
   it's subdirectories to avoid crossing mount point boundaries.
2. re-create the rpm directly in /usr/lib/sysimage/rpm/. So not only
   the header would be stored there but actually the full content.

Option 2) seems more straight forward. Also, all of the system would
be self contained in /usr.

![one-usr-tree.png](/images/2020-07-07-rpm-delta-updates/one-usr-tree.png "one file system tree with rpms integrated")

Now with this solution everything is in one place. The whole system would
unfold from /usr/lib/sysimage/rpm/. The data would have to written to disk only
once, installation is just metadata updates. All in-place updates are actually
safe on a transactional system where modifications are applied in a CoW
snapshot.

## Footnotes

* <a name="footnote1">1</a>: man ioctl_fideduperange(2)
