---
layout: post
title: "Winter Driving: A Programmer's Prospective"
---

## A Talk About Uncertainty ##

This came to mind while listening to a (fantastic) talk about how to address the
uncertainties inherent in creating automated driving protocols. The context was
around building automatic driving trucks and the challenges associated with that
in the absence of broadly available LIDAR sensors.

One of the questions which was raised (and upon which I expounded) was the
effects of ice and other winter driving obstacles. The answer boiled down to
"that's an engineering problem". And while there is certainly room for the
engineering around traction to become better and more reliable - there's also
room for adjusting your plan based on the road conditions. I'd even go so far as
to assert that the perceived road conditions must be included in your computer
vision identification and planning stages of automation.

To that end, I'd like to call out a few points which I believe are not well
understood by programmers and engineers who don't regularly drive in areas with
hard winters.

## Myths About Winter Driving ##

### Anti-Lock Brakes Will Stop You on Ice ###

Anti-lock brakes are a compensation for average or worse driving skills, and
increase the braking distance well over the minimum distance required; a skilled
driver with normal brakes can out-perform that same driver using anti-lock
brakes. They also provide no guarantee of stopping your vehicle, let alone
maintaining control of that vehicle. Specific to the trucking industry,
air-brake powered trailers do not typically have ALB systems installed; so while
the tractor will stop, the trailer will continue on its merry way, causing a
jackknife.

The underlying truth, as it applies to an automated driving system, is that
stopping is a function of your speed much more than it is a function of either
your weight or tires/brakes/etc. Back-of-the-napkin physics shows that the
energy to be dissipated by stopping a 60kph truck is over 4x that of the same
truck going 30kph. On compromised roads, this increase in demand from the brakes
is not something that can be ignored.

If you take nothing else away from this list - take this away: Slow speeds
matter more to winter driving than any other factor.

### 4WD/AWD/Traction Control Will Prevent Slides on Ice ###

On dry pavement, such systems will keep you from sliding. Traction Control
systems can reliably detect slips and reduce power to the drivetrain in
fractions of a second - faster than your average driver can. The problem is,
when you're on ice, that fraction of a second is enough to send your vehicle
out of control; and once it's started sliding, it's hard to stop. You have to
take action to give your tires sufficient traction to regain directional
control before you travel too far out of your lane.

Safe recovery from a slide depends upon the tires traveling at the same speed
and direction as the slide. I don't know of a traction control mechanism which
automatically manages this properly; mostly because they are unable to detect
sideways motion or the car's actual speed at the granularity required. Of
course, there is something which could have the proper granularity: the
positioning sensors of the automated driving system.

How do you prevent slides? Identify an icy intersection and accelerate slowly,
and use the information available to recover when you do slide. Also, do not
attempt to accelerate up an icy overpass.

### A Paradoxical Case of When Speeding is Important on Ice ###

In many cases, slow counters the challenges raised by ice. However there exists
an important edge case where you need to speed up on icy surfaces: when coming
up on a hill.  If a vehicle does not have the necessary momentum to crest a
hill, it will end up stuck on that hill, and short of adding chains, the vehicle
will have to go back down the hill and retry building sufficient momentum to
crest the hill. 

If you've ever seen a car on fire in the middle of a snow storm, that's probably
what happened. Why the fire? Well, the "planning" part of the driver said "we
need more power to get up the hill" and applied power. Making no progress, they
applied more power. The engine overheats and melts its seals. Oil spreads out on
the engine and is ignited by the (likely glowing red) surface of the exhaust
manifold. Fire.

### Roads are Only Icy if the Temperature is Below 0 C ###

Perhaps ironically, ice the most slippery on a sunny day following a freeze. The
air temperature gets above freezing, and melts the very top of a patch of ice,
leaving behind a thin layer of water. Any friction the ice would have naturally
provided, or that the siping on tires could have given on that ice is almost
completely negated by the thin layer of water.

You can try this yourself: Grab an ice cube fresh out of the freezer, and
compare your ability to hold on to it to one that's been out in the air for a
half hour or so; the second will be much more slippery.

If the ground is thoroughly frozen, such ice patches can persist across multiple
sunny days, despite the air temperatures reaching upwards of 10-15 degrees above
freezing. As a result, you can not rely solely on the air temperature provided
by weather reports to identify if a road will be hazardous; you have to look at
the roads themselves and test your brakes.

### Roads Conditions are Fairly Uniform ###

In the northern part of the US, especially the northwest, there are patches of
road which will receive no direct sunlight all day long. In more of the
mid-west, you'll run across patches which are either shaded by trees or by
drifted snow (snow being an amazing insulator).

This means that 90% of the highway could be completely clear, with only periodic
patches of ice. And so long as that icy portion of road is straight, and you
don't take any actions which cause your tires to slip, there will be few
repercussions of hitting those ice patches. However, if that ice is on a corner,
and you're traveling at the speed limit, you will end up in the ditch. This is
especially a problem in the north-west, since most mountain-shaded roads are in
passes, with frequent curves and constant elevation changes.

Human drivers will identify these patches at a distance, and slow to an
appropriate speed before actually hitting the patch. Automated driving
mechanisms will have to do the same, if they want to avoid costly crashes.

### Loss of Control is Limited to Sliding in a Straight Line ###

Let me introduce you to this driver's worst nightmare: slush. Slush is particles
of ice suspended in water, just like your favorite Frappe. Slush is effectively
a non-Newtonian fluid, where the speed of interaction matters. It is capable of
pushing your car sideways even at low speeds (under 10kph); at high speeds, it
will throw a vehicle off the road in less than a dozen meters.

How do you handle slush? You go slow. While under the effect of slush, your car
is not under your control, it can not (for most practical purposes) be recovered
from. So, put your controls in an attitude which lets you move in a correct
direction when you regain control of your vehicle, and go slow to both minimize
the chance of slush affecting you and to maximize your chances of recovering
prior to a crash.

How can you identify slush? By it's look, by it's feel and sound as you drive
through it. Slush sucks.

### Road Crews Keep the Highways Clear ###

Those (dramatically underpaid) folks do their best, but in the winter they're
all too often fighting a losing battle. A very recent example:

Tuesday day: Sunny and warm, with a bit of rain coming later in the day.

Tuesday night: Temperatures drop as the storm rolls in, turning wet roads to
ice, and snowing on the top of that ice.

Wednesday morning: Road crews plow off the snow, revealing ice. They sand and
apply de-icer, but the sand is used sparingly (it's hard on cars, and doesn't
"stick"), and the de-icer takes time to work. Cars and trucks are going into the
ditch on a regular basis, with towing crews roaming the highway to make a few
(thousand) bucks. It gets so bad that traffic is being stopped by trucks who are
unable to make it up slopes of overpasses without stopping and putting on
chains.

The final result? Traffic at a standstill for hours, and for over 10 miles on the
interstate. And most of these drivers were used to driving in winter conditions,
on properly maintained roads.

### Winter Driving Conditions are Limited to Certain Months ###

In Montana, there is not a single day of the year that has not received snow.
What we're receiving of climate change has been making the winters more
unpredictable, not less. And Montana's one of the tamer northern states, all
things considered.

## What's the Solution? ##

None of these obstacles are insurmountable for an automated driving system; but
only if they are taken into account not only in the design of the braking
and powerplant systems, but also the vision, location, and planning systems.

To put it more bluntly, especially for a system that wants to take over driving
semi trucks: the excuse of "it's an engineering problem" won't last on the 5th
call for help by the truck in the last two hours, or especially when your truck
is in the ditch because it went the speed limit onto a patch of ice, destroying
a few million dollar in goods in the process.

