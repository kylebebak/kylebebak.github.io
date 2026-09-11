---
layout: post
comments: true
title: "Clustering Location History Records: How and Why"
categories: code algorithms
tags: [algorithms, google, location-history, clustering, privacy]
---

## Motivation
If you have an Android phone with location services turned on, whenever your phone is connected to the Internet it sends __location history records__ to Google's servers, one per minute. This is true of Microsoft's and Apple's phones as well. These records contain three important fields: `latitude`, `longitude`, and `timestamp`. If your phone is usually with you, these records give a very complete picture of where you were at any time in the past.

Their usefulness, however, is limited by their number: location history is ___too detailed___. In a year your phone will send 500,000 records to be saved in a database. Executing queries on millions of records to uncover patterns is difficult for both users and hardware. What's worse is that the records are semantically identical. We know intuitively that 600 consecutive records with the same geographic coordinates represent something very different from a sequence of 30 records starting at home and ending at work, so we need a classification that makes this clear.

This is where clustering comes in. It accomplishes two important goals:

- compresses the data
- groups records into distinct, meaningful entities

I'll describe these entities to explain the basic idea, then I'll look at the algorithm to show that the clustering is simple, fast, and predictable. At the end I'll talk about some applications.

## Visits, Trips and Locations
We return to the intuition that a sequence of records in the same location are different from a sequence that starts at A and ends at B. Our first logical entity, then, will serve to group consecutive stationary records into visits. Specifically, a __`visit`__ is a sequence of `N` or more records separated by no more than a distance `R`. A visit knows its `lat`, `lon`, `start_time`, and `end_time`.

A sequence of records which is not stationary and cannot be grouped into a visit is necessarily grouped into a __`trip`__ between two visits. Every pair of consecutive visits is connected by a trip. A trip knows its `displacement`, `distance`, `start_time`, `end_time`, `start_visit`, and `end_visit`. By giving the trip pointers to the first and last location history records from which it was formed, these records can be used to reconstruct the trip's route if necessary.

The last entity we define allows us to cluster visits <em>(and hence trips, which are defined in terms of their start and end visits)</em> into groups. We define a __`location`__ as the average latitude and longitude of a group of visits separated by no more than a distance `R`, the same R that we used for clustering records into visits. The motivation is simple: locations allow us to see hundreds of otherwise distinct visits as belonging to the same equivalence class, e.g. visits to __home__, or visits to __work__. They also group trips into equivalence classes, e.g. trips __from home to work__, or __from work to home__.

Assigning locations to visits and trips allows us to answer aggregate questions, like:

- Where do I spend most of my time (top 5 places) on Saturdays?
- How many times have I been to my girlfriend's house, on the weekend?
- Over the past 6 months, at what time on average do I leave work on Friday?
- How long on average does it take me to get to work?

When run on 18 months (~750,000 records) of my location history, this process turned up 4,400 visits (and hence 4,400 trips) belonging to 750 locations. I used `N=6, R=10m` as clustering parameters, meaning that 6 minutes or more spent in the same 10 meter radius was considered a visit. 750,000 records were thus transformed into 10,000 vastly more expressive ones.

## Clustering Algorithm
Without constraints, clustering spatial data is a hard problem with imprecise solutions. Even using heuristics it's hard to beat quadratic time complexity, the intuition being that to assign each record to a cluster, you need to compare it with all the other records in the set to know which ones are nearby.

However, when clustering location history records we exploit the fact that the __distance__ metric by which we define __nearness__ includes ___time___. We only group records that occur within a small span of time, and because we can sort the records by timestamp (indeed, they're already sorted for us), independent of geographic coordinates, we can get the clustering done in ___linear___ time.

We also exploit our classification scheme: we are looking for an alternating sequence of visits and trips. We know what the visits look like, and <em>we don't even need to find the trips.</em> Every trip is bookended by a pair of visits, so once we have the visits we simply read through the unclustered records that connect them and instantiate our trips.

Here's how it works. In the first pass through the data, each record is compared to the most recently instantiated __potential visit__, whose latitude and longitude are the average coordinates of the records it contains. If the record is within a distance `R` of the visit, it gets added to the visit, and the visit's latitude and longitude are recalculated to reflect the addition of the record. If the record is not within `R` of the visit, a new potential visit is instantiated containing only this record. In Python:

~~~py
visits = [] # potential visits
visit = Visit(records[0])

for record in records[1:]:
    # records sorted from oldest to newest
    if visit.distance_to(record) <= R:
        visit.add(record)
    else:
        visits.append(visit)
        visit = Visit(record) # instantiate new potential visit

visits.append(visit) # append last visit
~~~

Next, potential visits with fewer than `N` constituent records are discarded. The remaining visits have pointers to their first and last records, which allows the second pass through the data to focus only on the unclustered records. From each sequence of records linking one visit to another, a trip is instantiated.

A third pass, this time through the visits, generates locations. We compare each visit to all existing locations, and add it to the first location within distance `R` of the visit. We recompute this location's latitude and longitude as the average coordinates of the visits it contains, ___weighted by the duration of these visits___. If no nearby location exists, we instantiate a new location containing only this visit.

## Discussion
Depending on implementation, the clustering is deterministic: in any case, the only way to significantly change the results is to tweak the parameters `N` and `R`. Fortunately, the effects of these parameters are predictable. Increasing `N` causes shorter visits to disappear, and increasing `R` makes some visits slightly longer, and causes some adjacent locations to be merged.

Exploiting structure and constraints inherent in the data, we avoid the guesswork used in algorithms like __k-means__, resulting in a simple, predictable algorithm that produces clusters whose meaning is clear. As for time complexity, any conceivable clustering will have to read each record at least once, so our linear implementation, at least for clustering visits, is <em>as fast as possible</em>.

An aside: it seems plausible that a sequence of unfortunately placed records, moving slowly but surely in one direction, could __"stretch out"__ a visit so that it contains pairs of records that are separated by a distance much greater than `R`. [I've shown here]({{ site.baseurl }}/sub-pages/visit-logarithmic-growth) that for the most pathological sequence of records, the __"diameter"__ of the visit grows as `R * ln(n)`, where `n` is the number of records in the visit. With real data, stretched visits occur with vanishing probability. Still, it's reassuring that even in the worst case they grow slowly.

Another aside: because locations have no temporal component, we can't exploit time in our distance metric to decrease the number of comparisons we make to instantiate them. However, these comparisons are made between visits and locations, not between pairs of records. Comparing 4,400 visits to 750 locations is ___much less expensive___ than doing pairwise comparison between 750,000 records. If these comparisons become costly, we can always consider other techniques for finding neighbors, like [geohashing](http://www.bigfastblog.com/geohash-intro).


## Applications
Visits and trips give a good account of your movements, and even some of your habits, but this story can be enriched by knowing ___what___ you're doing and ___with whom___, instead of just where you're going. Google, for example, is in a good position to generate these stories. With the [Geocoding API](https://developers.google.com/maps/documentation/geocoding/intro#ReverseGeocoding), addresses can be looked up for geographical coordinates, so that automatic descriptions and photos are generated for your locations.

If you take pictures and use Google Photos, these could be linked to the visits or trips on which they were taken. If you go to a store, or a restaurant, or a museum, you are probably visiting a location for which Google has records and metadata, and your location instance could be a given a pointer to theirs. Suppose you use Google Wallet, or have your bank account configured to send you transaction alerts via text message. Financial transactions could be attached to the visits on which they occur.

Most importantly, visits, trips and locations point to the users that generate them, and this is where social networking comes into play. Imagine you __"friend"__ another user. Your visits that __"overlap"__ spatially and temporally with theirs could automatically be __"shared"__ between you, so that you could look at photos, comments, or whatever else your friends add to these visits. You could look at a location and see the list of friends who shared a visit there with you, then select one of these friends and see the list of visits you shared with them. And so on.

## Privacy
For allowing users to track themselves in the privacy of their own phones, it's easy to write a client-side application to cluster and store location history records. But part of the tradition of the web is users trading privacy for convenience, or simply for the opposite of privacy. I think the networking features of location history, which can't be implemented client-side, will be embraced by users in the future. I wrote this clustering program, and built a front end for [exploring the results](https://dronfelipe.onrender.com/location_history), in 2014. Google probably thought of it before that, but didn't release [Timeline](https://www.google.com/maps/timeline) until 2015, because it's an idea that [gives us pause](http://venturebeat.com/2015/07/27/hands-on-with-google-maps-your-timeline-fascinating-but-freaky/). We need to get comfortable tracking ourselves before we are tracked by our friends.

