---
layout: post
title: "Talking about Graphite: Whisper"
---

## Talking about Graphite ##

This is the first in a short series talking about the
[Graphite](http://graphite.readthedocs.org/en/latest/) graphing utility. This
discussion will be about Whisper, the default storage engine used by Graphite.
It will also be the most technically in-depth article, since I have been
devoting a lot of time to researching the Whisper storage engine and its file
format, for reasons I will detail below. Questions and comments are always
welcome; find my email address on the [about](/about/) page.

### The Whisper Storage Engine ###

The Graphite utility uses the [Whisper storage
engine](https://github.com/graphite-project/whisper) by default for storing
points of time-series data. The Whisper storage engine was more-or-less custom
built just for Graphite, and provide for the long term storage of time-series
data at multiple resolutions.  For example, the format easily supports keeping
six hours of 10 second intervals, six days of 1 minute intervals, and 6 months
of 1 hour intervals, and the act of updating one of the 10 second intervals
will automatically populate the lower resolution indexes, using an aggregation
method of your choosing.

Each series of data is stored in a single file, which can be easily shared
among multiple processes (by letting the OS handle the details of sharing data
between multiple processes), and is easily parsed by any program which wants
the data.

#### Whisper File Format ####

The Whisper file consists of a header structure, a variable length array of
archive metadata, and a variable number of archives which consist of a list of
time-series data. All numbers are stored in the big-endian format.

Also, please note that Whisper uses the Python
[struct](https://docs.python.org/2/library/struct.html) package for packing
these values, specifying 'long' and 'float' number sizes, which are platform
dependent lengths. I will be using the explicit lengths as they exist on Linux
in this article wherever possible.

##### Header #####

The header consists of four values:

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
  * The maximum number of seconds worth of data which can be stored in any archive
* X Files Factor (float32)
  * If there are less than XFF records which are populated, then the
    aggregation method will not be run against the data, resulting in a None
    value being stored.
* Archive Count (uint32)
  * The number of archives stored in this whisper file.

##### Archive List ######

There are `Archive Count` number of archive headers, and each archive header
consists of the following:

* Offset (uint32)
  * Location in the file of the start of this particular archive
* Seconds Per Point (uint32)
  * How many seconds worth of data is encapsulated per data point. This is used
    to help calculate the retention capabilities of a given archive when
    combined with: 
* Points (uint32)
  * How many actual data points are stored in the archive. This is also used to
    determine the length of the archive on disk.

##### Archive _N_ #####

For each archive, there is a list of data points. Each data point consists of
two numbers, the timestamp (stored as a uint32), and the value (stored as a
float64).


#### Format Example ####

Here is a sample empty Whisper file, broken down using `hexdump`.

{% gist garrickp/9191287c36b69ed59d58 %}

#### Whisper Constraints ####

Whisper introduces a number of rules which guide the creation of a set of
archives.

* The "seconds per point" for all lower precision archives must be a multiple
  of the precision of all of the higher precision archives.
  * eg. If your high precision archive is 10 seconds, your next lower precision
    must be a multiple of that, like 60, and the next archive must be something
    like 300.
  * You could not have 10, 30, and 75 precision archives.
* There must be more than enough points in a higher precision archive to fill a
  complete lower precision archive point.
  * That is, you must have at least 7 points in a 10 second archive to fill a
    60 second resolution archive.
* Two archives can not have the same precision

### How Whisper updates ###

So the file format is pretty simple and straightforward. It lets a consumer
read the header and access pretty much any value within the structure using
easily computed offsets and seeks.  Now let's discuss briefly what happens when
a value is updated.

For the example below, we have three archives with the following specifications:

* 10 seconds for 6 hours
* 1 minute for 6 days
* 1 hour for 6 months

First, truncate the incoming timestamp to the nearest "seconds per point" using
the following equation:

{% highlight python %}
myInterval = timestamp - (timestamp % archive['secondsPerPoint'])
{% endhighlight %}

Second, find the highest precision archive which covers the timestamp being
filled. If the timestamp is less than 6 hours old, the value will go into the
first archive. If it is between 6 hours and 6 days, the second archive, and so
on.

Next, look at the first value in the archive, and determine its timestamp. Use
this, combined with the incoming value's timestamp, and determine where in the
file it will go, using the following equation:

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
* If the ratio of populated points to all points is higher than the `X Files
  Factor`, aggregate those values & write the result to the lower precision
  archive.
* Go back to the first step, using the current archive and the next lowest
  precision archive

I'll be honest, this process took me awhile to fully grok. The fact that the
writes rotate through an archive using some nice pointer math is both clever
and confusing the first time through.

## Whisper's Weaknesses ##

For all that Whisper is a simple purpose built storage engine, it has two major
flaws and a number of minor flaws.

The first major flaw is simply that the first update which is made is
destructive. If you update a point, you immediately overwrite whatever was
there previously. The lower precision archives can also loose data, as they
re-calculate their values based on what is in the higher precision archives.
This isn't a major problem, so long as you are able to write to the file
exactly once per time period and do your own aggregation; but any deviation
from this will result in intermittent data loss. This is particularly
problematic, given that Graphite is written in Python using the twisted
networking library, and is thus subject to all of the limitations associated
with that combination.

The second major flaw is more subtle (and only readily visible after you've
invested into the technology): Whisper is really inefficient with its disk
operations. As an example, here's the sequence of reads and writes associated
with a single update to our example file:

{% highlight python %}
open()
seek(0)
read(16) # header
read(12) # Archive info[0]
read(12) # Archive info[1]
read(12) # Archive info[2]
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
stuff. Combine that with the fact that there is one file per metric, and
hundreds or thousands of active metrics, and it can very easily overwhelm just
about any disk setup.

The first minor problem has to do with Whisper's file format - the files are of
a fixed size, and thus consume a fixed amount of space on disk for each file
(the actual size depends on the number of points you save, but our sample
format above consumes around 300mb of space per file).  Not a huge problem with
a few metrics, but if you are constantly adding and removing hosts with
hundreds of individual data points, you will find yourself quickly running out
of space.

Another problem is that this is, after all, a file based system. Files are just
plain slow, when compared to storing data in memory. Sure, this has gotten
better with the advances in solid state drives and OS level caching, but again
with the disk unfriendliness of the update activity.

My final minor gripe has to do with how `carbon_cache.py` (the daemon which
listens for incoming metrics and writes the Whisper files) manages the files. A
whisper file can have its aggregation method and even archive values mutated,
however `carbon_cache.py` will not handle this for you; instead you have to go
back and use the command line tools to manually merge and mutate the files. Not
the most user friendly method of management.

## In Conclusion ##

This was a fairly long and technical dive into the Whisper storage engine used
by Graphite, but I hope it is of some use when you're out there managing the
whisper files on your own.

Good luck!


