6.824 2015 Lecture 17: PNUTS
============================

**Note:** These lecture notes were slightly modified from the ones posted on the
6.824 [course website](http://nil.csail.mit.edu/6.824/2015/schedule.html) from 
Spring 2015.

Brian F. Cooper, Raghu Ramakrishnan, Utkarsh Srivastava, Adam
Silberstein, Philip Bohannon, Hans-Arno Jacobsen, Nick Puz, Daniel
Weaver and Ramana Yerneni. PNUTS: Yahoo!'s Hosted Data Serving
Platform. Proceedings of VLDB, 2008.

Why this paper?

 - same basic goals as Facebook/memcache paper, more principled design
 - multi-region is very challenging -- 100ms network delays
 - conscious trade-off between consistency and performance

What is PNUTS' overall goal?

Diagram:
    
    [world, browsers, data centers]

 - overall story similar to that of Spanner and Facebook/memcache
 - data centers ("regions") all over the world
 - web applications, e.g. mail, shopping, social net
   + each app probably runs at all regions
 - PNUTS keeps state for apps
   + per-user: profile, shopping cart, friend list
   + per-item: book popularity, user comments
 - app might need any piece of data at any data center
 - need to handle lots of concurrent updates to different data
   + e.g. lots of users must be able to add items to shopping cart at same time
     thus 1000s of PNUTS servers
 - 1000s of servers => crashes must be frequent

Overview
--------

Diagram:
    
    3 regions, browsers, web apps, tablet ctlrs, routers, storage units, MBs]

 - each region has all data
 - each table partitioned by key over storage units
   + tablet servers + routers know the partition plan

Why replicas of all data at multiple regions?

 - multiple regions -> each user's data geographically close to user
 - multiple complete replicas -> maybe survive entire region failure
 - complete replicas -> read anything quickly
   + since some data used by many users / many regions
   + once you have multiple regions, fast reads are very important

What are the drawbacks of a copy at each region?

 - updates will be slow, need to contact every region
 - local reads will probably be stale
 - updates from multiple regions need to be sorted out
   + keep replicas identical
   + avoid order anomalies
   + don't lose updates (e.g. read-modify-write for counter)
 - disk space probably not an issue for their uses

What is the data and query model?

 - basically key/value
 - reads/writes probably by column
   + so a write might replace just one column, not whole record
 - range scan for ordered tables

How do updates work?

 - app server gets web request, needs to write data in PNUTS
 - need to update every region!
 - why not just have app logic send update to every region?
   + what if app crashes after updating only some regions?
   + what if concurrent updates to same record?

PNUTS has a "record master" for each record

 - all updates must go through that region
   + each record has a hidden column indicating region of record master
 - responsible storage unit executes updates one at a time per record
 - tells MB to broadcast update to all regions
 - per-record master probably better than Facebook/memcache master region

So the complete update story (some guesswork):

App wants to update some columns of a record, knows key

  1. app sends key and update to local SU1
  2. SU1 looks up record master for key: SI2
  3. SU1 sends update request to router at SI2
  4. router at SI2 forwards update to local SU2 for key
  6. SU2 sends update to local Message Broker (MB)
  7. MB stores on disk + backup MB, sends vers # to original app
     how does MB know the vers #? maybe SU2 told it
     or perhaps SU2 (not MB) replies to original app
  8. MB sends update to router at every region
  9. every region updates local copy

Puzzles:

 - 3.2.1 says MB is commit point
   + i.e. MB writes to log on two disks, keeps trying to deliver
     why isn't MB disk a terrible bottleneck?
 - does update go to MB then SU2? or SU2 then MB? or SU2, MB, SU2?
   + maybe MB then SU2, since MB is commit point
   + maybe SU2 then MB, since SU2 has to check it's the record's master
     and perhaps pick the new version number, tho maybe not needed
 - who replies to client w/ new version #?

All writes are multi-region and thus slow -- why does it make sense?

 - application waits for MB commit but not propagation ("asynchronous")
 - master likely to be local (they claim 80% of the time)
   + so MB commit will often be quick
   + and app/user will often see its own writes soon
 - still, eval says 300ms if master is remote!
 - down side: readers at non-master regions may see stale data

How does a read-only query execute?

 - multiple kinds of reads (section 2.2) -- why?
 - application gets to choose how consistent
 - `read-any(k)`
   + read from local SU
   + might return stale data (even if you just wrote!)
   + why: app wants speed but doesn't care about freshness
 - `read-critical(k, required_version)`
   + maybe read from local SU if it has vers >= required_version
   + otherwise read from master SU?
   + why: app wants to see its own write
 - `read-latest(k)`
   + always read from master SU (? "if local copy too stale")
   + slow if master is remote!
   + why: app needs fresh data

What if app needs to increment a counter stored in a record?

 - app reads old value, increments locally, writes new value
 - what if the local read produced stale data?
 - what if read was OK, but concurrent updates?

`test-and-set-write(version#, new value)` gives you atomic update to one record
 - master rejects the write if current version # != version#
 - so if concurrent updates, one will lose and retry 

`TestAndSet` example:

      while(1):
        (x, ver) = read-latest(k)
        if(t-a-s-w(k, ver, x+1))
          break

The Question
------------

 - how does PNUTS cope with Example 1 (page 2)
 - Initially Alice's mother is in Alice's ACL, so mother can see photos
   1. Alice removes her mother from ACL
   2. Alice posts spring-break photos
 - could her mother see update #2 but not update #1?
   + esp if mother uses different region than Alice
     or if Alice does the updates from different regions
 - ACL and photo list must be in the same record
   + since PNUTS guarantees order only for updates to same record
 - Alice sends updates to her record's master region in order
   + master region broadcasts via MB in order
   + MB tells other regions to apply updates in order
 - What if Alice's mother:
   - reads the old ACL, that includes mother
   - reads the new photo list
   - answer: just one read of Alice's record, has both ACL and photo list
     + if record doesn't have new ACL, order says it can't have new photos either
 - How could a storage system get this wrong?
   + No ordering through single master (e.g. Dynamo)

How to change record's master if no failures?

 - e.g. I move from Boston to LA
 - perhaps just update the record, via old master?
   + since ID of master region is stored in the record
 - old master announces change over MB
 - a few subsequent updates might go to the old master
   + it will reject them, app retries and finds new master?

What if we wanted to do bank transfers?
 - from one account (record) to another
 - can `t-a-s-w` be used for this?
 - multi-record updates are not atomic
   + other readers can see intermediate state
   + other writers are not locked out
 - multi-record reads are not atomic
   + might read one account before xfer, other account after xfer

Is lack of general transactions a problem for web applications?

- maybe not, if programmers know to expect it

What about tolerating failures?

App server crashes midway through a set of updates

 - not a transaction, so only some of writes will happen
 - but master SU/MB either did or didn't get each write
   + so each write happens at all regions, or none

SU down briefly, or network temporarily broken/lossy

 - (I'm guessing here, could be wrong)
 - MB keeps trying until SU acks
   + SU shouldn't ACK until safely on disk

SU loses disk contents, or doesn't automatically reboot 

 - can apps read from remote regions?
   + paper doesn't say
 - need to restore disk content from SUs at other regions
   1. subscribe to MB feed, and save them for now
   2. copy content from SU at another region
   3. replay saved MB updates
 - Puzzle: 
   + how to ensure we didn't miss any MB updates for this SU?
     - e.g. subscribe to MB at time=100, but source SU only saw through 90?
   + will replay apply updates twice? is that harmful?
   + paper mentions sending checkpoint message through MB
     - maybe fetch copy as of when the checkpoint arrived
     - and only replay after the checkpoint
     - BUT no ordering among MB streams from multiple regions

MB crashes after accepting update

 - logs to disks on two MB servers before ACKing
 - recovery looks at log, (re)sends logged msgs
 - record master SU maybe re-sends an update if MB crash before ACK
   + maybe record version #s will allow SUs to ignore duplicate

MB is a neat idea

 - atomic: updates all replicas, or none
   + rather than app server updating replicas (crash...)
 - reliable: keeps trying, to cope with temporarily SU/region failure
 - async: apps don't have to wait for write to complete, good for WAN
 - ordered: keeps replicas identical even w/ multiple writers

Record's master region loses network connection

 - can other regions designate a replacement RM?
   + no: original RM's MB may have logged updates, only some sent out
 - do other regions have to wait indefinitely? yes
   + this is one price of ordered updates / strict-ish consistency

Evaluation
----------

Evaluation focuses on latency and scaling, not throughput

5.2: time for an insert while busy

 - depends on how far away Record Master is
 - RM local: 75.6 ms
 - RM nearby: 131.5 ms
 - RM other coast: 315.5 ms

What is 5.2 measuring? from what to what?

 - maybe web server starts insert, to RM replies w/ new version?
 - not time for MB to propagate to all regions
   + since then local RM wouldn't be `< remote`

Why 75 ms?

Is it 75 ms of network speed-of-light delay?

 - no: local

Is the 75 ms mostly queuing, waiting for other client's operations?

 - no: they imply 100 clients was max that didn't cause delay to rise

End of 5.2 suggests 40 ms of 75 ms in in SU

 - how could it take 40 ms?
   + each key/value is one file?
   + creating a file takes 3 disk writes (directory, inode, content)?
 - what's the other 35 ms?
   + MB disk write?

But only 33 ms (not 75) for "ordered table" (MySQL/Innodb)

 - closer to the one or two disk write we'd expect

5.3 / Figure 3: effect of increasing request rate

 - what do we expect for graph w/ x-axis req rate, y-axis latency?
   + system has some inherent capacity, e.g. total disk seeks/second
   + for lower rates, constant latency
   + for higher rates, queue grows rapidly, avg latency blows up
 - blow-up should be near max capacity of h/w
   + e.g. # disk arms / seek time
 - we don't see that in Figure 3
   + end of 5.3 says clients too slow
   + at >= 75 ms/op, 300 clients -> about 4000/sec
 - text says max possible rate was about 3000/second
   + 10% writes, so 300 writes/second
   + 5 SU per region, so 60 writes/SU/second
   + about right if each write does a random disk I/O
   + but you'll need lots of SUs for millions of active users

Stepping back, what were PNUTS key design decisions?

  1. replication of all data at multiple regions
     - fast reads, slow writes
  2. relaxed consistency -- stale reads
     - b/c writes are slow
  3. only single-row transactions w/ test-and-set-write
  4. sequence all writes thru master region
     + pro: keeps replicas identical,
       enforces serial order on updates,
       easy to reason about
     + con: slow, no progress if master region disconnected

Next: Dynamo, a very different design

 - async replication, but no master
 - eventual consistency
 - always allow updates
 - tree of versions if network partitions
 - readers must reconcile versions