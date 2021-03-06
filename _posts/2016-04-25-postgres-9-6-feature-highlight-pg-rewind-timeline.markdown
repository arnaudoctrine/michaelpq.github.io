---
author: Michael Paquier
lastmod: 2016-04-25
date: 2016-04-25 12:45:32+00:00
layout: post
type: post
slug: postgres-9-6-feature-highlight-pg-rewind-timeline
title: 'Postgres 9.6 feature highlight - pg_rewind and timeline switches'
categories:
- PostgreSQL-2
tags:
- postgres
- postgresql
- 9.6
- pg_rewind
- timeline

---

[pg_rewind](https://www.postgresql.org/docs/devel/static/app-pgrewind.html),
a utility introduced in PostgreSQL 9.5 allowing one to reconnect a master
to one of its promoted standbys, making unnecessary the need of a new base
backup and hence saving a huge amount of time, has gained something new
in Postgres 9.6 with the following commit:

    commit: e50cda78404d6400b1326a996a4fabb144871151
    author: Teodor Sigaev <teodor@sigaev.ru>
    date: Tue, 1 Dec 2015 18:56:44 +0300
    Use pg_rewind when target timeline was switched

    Allow pg_rewind to work when target timeline was switched. Now
    user can return promoted standby to old master.

    Target timeline history becomes a global variable. Index
    in target timeline history is used in function interfaces instead of
    specifying TLI directly. Thus, SimpleXLogPageRead() can easily start
    reading XLOGs from next timeline when current timeline ends.

    Author: Alexander Korotkov
    Review: Michael Paquier

[The following document](/content/materials/20130713_pgunconf_pg_rewind.pdf)
gives a short introduction about pg\_rewind. You may want to look at it
before moving on with this post and understand why here is described a damn
cool feature. First, before entering in the details of this feature, let's
take the example of the following cluster:

                                Node 1
                                /    \
                               /      \
                              /        \
                           Node 2   Node 3
                           /
                          /
                         /
                      Node 4

Node 1 is a master node, with two standbys directly connected to it, presented
here as nodes 2 and 3. Finally node 4 is a cascading standby connected to node
2. Then say that all the nodes have been successively promoted. At promotion,
a standby finishes recovery and jumps to a new
[timeline](https://www.postgresql.org/docs/devel/static/continuous-archiving.html#BACKUP-TIMELINES)
to identify a new series of WAL record generated by what is now a master
node to avoid overlapping WAL records from another node. The promotion of
each node in the cluster is done in the following order, and results in
the following timelines being taken by each node:

  * Node 2 promotes, at the end of recovery it begins to use timeline 2.
  Node 4 jumps as well to timeline 2, and continues to follow node 2.
  * Node 3 promotes, it switches to timeline 3.
  * Node 4 promotes, it switches to timeline 4.

At this point all the nodes have become master nodes, and are now generating
their own history and making their own way in life. In terms of WAL history,
the cluster becomes as follows:

      ----------------------------------> Node 1 (timeline 1)
          \        \
           \        \
            \        \------------------> Node 3 (timeline 3)
             \
              \
               \------------------------> Node 2 (timeline 2)
                    \
                     \
                      \-----------------> Node 4 (timeline 4)

With 9.5's pg\_rewind, when trying to rewind a node, or to put it in other
words to synchronize a target node with a source node so as the target node
can become a standby of the target node, it is not possible to perform
complicated operations like in the case of the cluster described above.
For example, one can use node 1 as target and node 2 as source so as
node 1 is recycled as a standby of node 2. Same for node 1 as source and
node 3 as target, and similarly for node 4 as source and node 2 as target.
Also, as a timeline history cannot be looked up backwards, it is not
possible to rewind a promoted standby to its former master. In short,
when trying more complicated operations, the following error will be
quickly found out:

    could not find common ancestor of the source and target cluster's timelines
    Failure, exiting

Finally, what does the feature aimed at being described in this blog post
do? Well, to put it simply pg\_rewind has been extended so as it can have
a look at the timeline history graph like the one above. And it is able to
find out the most recent shared point in the timeline history between a target
and a source node, and then it performs a rewind operation from this point. In
even shorter words, do you remember the four nodes of the same cluster promoted
one after the other? In PostgreSQL 9.6, pg\_rewind can allow one to rebuild
a complete cluster using all the existing nodes, without limitation, even
if they completely forked off. Any node can become the master, and have as
standbys the other nodes. This abscence of limitation is what makes this
feature a really cool thing, and something to count on.
