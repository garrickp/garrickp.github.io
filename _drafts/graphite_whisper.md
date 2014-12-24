---
layout: post
title: "Talking about Graphite: Whisper"
---

## Talking about Graphite ##

This post is the first in a short series which talks about the
[Graphite](http://graphite.readthedocs.org/en/latest/) graphing utility. This
post will cover Whisper, the default storage engine built for and used by
Graphite.  It will also be the most technically in-depth article; I have been
devoting a lot of time to researching the Whisper storage engine and its file
format, in an effort to overcome the shortcomings I will detail below.
Questions and comments are always welcome; find my email address on the
[about](/about/) page.

## The Whisper Storage Engine ##

The Graphite utility uses the
[Whisper](https://github.com/graphite-project/whisper) storage engine by
default for storing time series data. The Whisper storage engine was
more-or-less custom built just for Graphite, and provides long term storage of
time series data at multiple time resolutions.  For example, the Whisper
storage engine easily supports keeping six hours of ten second intervals, six
days of one minute intervals, and six months of one hour intervals, and the act
of updating one of the ten second intervals will automatically populate the
lower resolution archives, using an aggregation method of your choosing.

All of the archives for a given data stream are stored in a single file, which
can be easily shared among multiple processes by letting the OS handle the
details of sharing data between those processes, and can be easily parsed by
any program which wants to use the data.

### Whisper File Format ###

A Whisper file consists of a header structure, a variable length array of
archive metadata (the array lenth is stored in the header structure), and the
same number of archives which consist of a list of time-series data. All
numbers are stored in the
[big-endian](http://en.wikipedia.org/wiki/Endianness#Big-endian) format.

One quick implementation note: The Whisper storage engine uses the Python
[struct](https://docs.python.org/2/library/struct.html) library for packing
these values onto disk, using the generic (and platform dependent) 'long' and
'float' number lengths. I will instead be using explicit length identifiers
uint32, float32, and float64 to identfy these lengths as they exist on Linux in
this post, as Linux is the most widely used platform for hosting Graphite.

#### Header ####

The header structure consists of four packed values:

* Aggregation Type (uint32)
  * This is an enumeration, as follows

{% highlight python %}
aggregation_method = {1: 'average',
                      2: 'sum',
                      3: 'last',
                      4: 'max',
                      5: 'min'}
{% endhighlight %}

* Maximum Retention (uint32)
  * The maximum amount of time series data which can be stored in any archive,
    measured in seconds
* X-Files Factor(XFF) (float32)
  * If the ratio of populated (meaning not `None`) values to the total number
    of values is less than XFF, then the aggregation method will not be run
    against the data, resulting in no value being stored for that time period.
* Archive Count (uint32)
  * The number of archives stored in this whisper file.

#### Archive List #####

There are `Archive Count` number of archive headers, and each archive header
consists of the following:

* Offset (uint32)
  * Location in the file of the start of this particular archive
* Seconds Per Point(SPP) (uint32)
  * How many seconds worth of data is encapsulated per data point. This is used
    to help calculate the retention capabilities of a given archive when
    combined with: 
* Points (uint32)
  * How many actual data points are stored in the archive. This is also used to
    determine the length of the archive on disk.

#### Archive _N_ ####

For each archive, there is a list of data points. Each data point consists of
two numbers, the timestamp (stored as a uint32), and the value (stored as a
float64).

### Format Example ###

Here is an example empty Whisper file, broken down using `hexdump`.

{% gist garrickp/9191287c36b69ed59d58 %}

### Rules for a Valid Archive List ###

Whisper provides a number of rules which guide the creation of a set of
archives.

* The "seconds per point" for all lower precision archives must be a multiple
  of the precision of all of the higher precision archives.
  * eg. If your high precision archive is 10 SPP, your next lower precision
    must be a multiple of that, like 60, and the next archive must be something
    like 300.
  * You could not have archives with 10, 30, and 75 SPP precision.
* There must be more than enough points in a higher precision archive to fill a
  complete lower precision archive point.
  * In other words, you can not have just enough points of data in a higher
    precision archive to fill a lower precision archive, you must have more.
  * i.e. you must have at least 7 points in a 10 SPP archive to fill a 60 SPP
    archive.
* Two archives can not have the same precision

#### Pecularities ####

The file format is fairly straightforward and relatively easy to understand, but there are two pecularities to note:

* The archives wrap around themselves, and have no useful ordering.
  * This is not entirely true; the ordering is based on on the first value ever
    written, and ensures that any value written is a computable distance away
    from that first value, based on a modulous of the time distance between
    that value and the length of the archive. Useful for writing to the file,
    less useful for any other purpose.
* When reading values from the archive, time points which do not exist are
  considered to have no value (represented as `None` in Python).
  * Since there is no guarantee that there is a data point in place for every
    time slice, it is up to the reader to determine if a value should be
    populated or set to the language's equivelent of `None`.

### How Whisper updates ###

The Whisper file format lets a consumer read the header and access pretty much
any value within the structure using computed offsets and seeks.  Now let's
discuss briefly how the storage engine interacts with this file when a value is
updated.

For the example below, we have three archives with the following specifications:

* 10 seconds for 6 hours
* 1 minute for 6 days
* 1 hour for 6 months

First, truncate the incoming timestamp to the nearest "seconds per point" using
the following algorithm:

{% highlight python %}
myInterval = timestamp - (timestamp % archive['secondsPerPoint'])
{% endhighlight %}

Second, find the highest precision archive which covers the timestamp being
filled. If the timestamp is less than 6 hours old, the value will go into the
first archive. If it is between 6 hours and 6 days, the second archive, and so
on.

Next, look at the first value in the selected archive, and determine its
timestamp. Use this as a base interval, and use these two values in association
with the size of the archive to determine where in the file it will go, using
the following algorithm:

{% highlight python %}
timeDistance = myInterval - baseInterval # Base interval is the timestamp located in the record at the archive's offset
pointDistance = timeDistance / archive['secondsPerPoint']
byteDistance = pointDistance * pointSize # pointSize is 12, the size of a packed uint32 and float64
myOffset = archive['offset'] + (byteDistance % archive['size'])
{% endhighlight %}

Write the record to the location determined by myOffset

Following that, propagate the values down into the lower precision archives,
starting with the next lowest precision archive:

* Repeat the myInterval and myOffset logic above to find the value which will
  be updated in the lower precision archive
* Find and read the appropriate cells in the higher precision archive which
  will be aggregated into the new lower precision location
  * So for our example archive, we will identify and read the appropriate 10
    points in the higher precision archive which will be used to compute the
    value which goes into the lower precision archive
* If the ratio of populated points to all points is higher than the `X Files
  Factor`, aggregate those values & write the result to the lower precision
  archive.
* Go back to the first step, using the current archive and the next lowest
  precision archive

I'll be honest, this process took me awhile to fully grok. The fact that the
writes rotate through an archive using some nice pointer math, and the fact
that missing values are implemented as gaps in the data containing old points,
is both clever and confusing the first time through.

## Whisper Utilities ##

Graphite installs come with a number of supporting tools for managing Whisper
files.  Of particular note are:

* whisper-set-aggregation-method.py
  * Helps you modify existing files to match new settings in carbon_cache.py
* whisper-fill.py
  * Nice utility for merging multiple statistics into a single file while
    minimizing data loss. Note, this does not merge the values (there is no
    utility I'm aware of which does that), it just fills in gaps.
* whisper-dump.py
  * Useful for getting  a manual look into the data stored in a whisper file.
    I've used this for troubleshooting problems in the past.

## Whisper's Weaknesses ##

For all that Whisper is a simple purpose built storage engine, it has two major
flaws and a number of other minor annoyances.

The first major flaw is simply that the first update which is made is
destructive. If you update a point, you overwrite whatever was there
previously. The lower precision archives can also loose data, as they
re-calculate their values based on what is in the higher precision archives.
This isn't a major problem, so long as you are able to write to the file
exactly once per time period and do your own aggregation within that time
period; but any deviation from this will result in intermittent data loss. This
is particularly problematic given that Graphite is written in Python and uses
the twisted networking library, and is thus subject to all of the limitations
and slowdowns.

The second major flaw is more subtle (and only readily visible after you've
invested into the technology): Whisper is surprisingly inefficient with its
disk operations. As an example, here's the sequence of reads and writes
associated with a single update to our example file:

{% highlight python %}
open()
seek(0)
read(16) # header
read(12) # Archive info[0]
read(12) # Archive info[1]
read(12) # Archive info[2]
seek(0) # Technically to the point in the file where you were prior to reading,
        # but with a freshly opened file this will be 0
# Write to the first archive
seek(archiveOneOffset)
read(12) # First data point
seek(archiveOnePointOffset)
write(12)
# Propagate to the next archive
seek(archiveOneOffset)
read(12)
seek(coveringPointsOffset)
read(72)
# Optional seek and read of part of that 72, if we had to wrap around the higher precision archive
seek(archiveTwoOffset)
read(12)
seek(archiveTwoPointOffset)
write(12)
# Propagate to the third archive
seek(archiveTwoOffset)
read(12)
seek(coveringPointsOffset)
read(4320)
# Optional seek and read of part of that 4320, if we had to wrap around the higher precision archive
seek(archiveThreeOffset)
read(12)
seek(archiveThreePointOffset)
write(12)
close()
{% endhighlight %}

Even with the OS helping using file buffers and caches... this is really nasty
stuff. Combine that with the fact that there is is effectively no memory
caching of data, one file per metric, and hundreds or thousands of active
metrics, and it can very easily overwhelm just about any disk setup.

My first minor gripe has to do with Whisper's file format - the files are of a
fixed size, and thus consume a fixed amount of space on disk for each file (the
actual size depends on the number of points you save, but our sample format
above consumes around 30mb of space per file).  Not a huge problem with a few
metrics, but if you are constantly adding and removing hosts with hundreds of
individual data points, you will find yourself quickly running out of space.

Another problem is that this is, after all, a file based system. Files are just
plain slow, when compared to storing data in memory. Sure, this has gotten
better with the advances in solid state drives and OS level caching, but again
with the disk unfriendliness of the update activity.

My final minor gripe has to do with how `carbon_cache.py` (the daemon which
listens for incoming metrics and writes the Whisper files) manages the files. A
whisper file can have its aggregation method and even archive values mutated,
however `carbon_cache.py` will not handle this for you; instead you have to go
back and use the command line tools to manually merge and mutate the files.
Carbon also does not delete files, which can lead to a buildup of useless files
on disk. Not the most user friendly method of management.

## In Conclusion ##

This was a fairly long and technical dive into the Whisper storage engine used
by Graphite, but I hope it is of some use when you're out there managing the
Whisper files on your own.

Good luck!

