
:tocdepth: 1

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**


Introduction
============

This technical note describes series of tests performed with a Cassandra-based
APDB implementation on Google Cloud Platform (GCP). Previous round of APDB
tests with Apache Cassandra performed with a three-node cluster at NCSA
(`DMTN-156`_) showed promising results but it was not possible to evaluate
scaling behavior from those tests. Next series of tests was necessary at larger
scale and a plan for that test was outlined in `DMTN-162`_. Actual testing was
performed using GCP resources which allowed us to chose optimal configuration
for each test. Cluster setup and all tests are described in more detail in JIRA
tickets under `DM-27785`_ epic.


Cluster Configuration
=====================

GCP was used to run both Cassandra server cluster and client software. On
client side we are running 189 processes executing ``ap_proto`` application
which does not require significant resources. To avoid over-subscribing CPU
resources we were allocating one vCPU per process, the cluster that was used to
run client software was comprised of 6 nodes, each has e2-custom type, with 32
vCPUs and 16GB RAM. The nodes were using ``lsst-w-2020-49`` image with LSST
software pre-installed. Client process execution mas orchestrated using MPI
mechanism, similarly to how it was done at NCSA.

Cassandra cluster is more resource-demanding, from our previous tests at NCSA
it was clear that fast SSD storage is critical for performance and Cassandra in
general needs abundant amount of RAM, though too much RAM can cause issues with
garbage collection. For these reasons the configuration for a single node in
Cassandra cluster was selected as:

- n2-custom VM type
- 32 vCPUs
- 64 GB RAM
- locally attached SSD storage, 4 or 8 partitions, 375 GB each
- Ubuntu-based image which is recommended for optimized nVME drivers

GCP locally attached SSDs are non-persistent (scratch) type, they are
re-created on each VM reboot, so special care is needed in planning the tests.
Cassandra cluster size varied between 3 and 12 nodes to evaluate scaling
behavior.

Cassandra configuration needs a usual update for some of its parameters, copy
of the `configuration files
<https://github.com/lsst-dm/l1dbproto/tree/u/andy-slac/cassandra-2/misc/gcp-test/cassandra-config>`_
is archived in ``l1dbproto`` package for future reference.


Three-node Setup
================

The very first test (`DM-28136`_) that was done with the new platform was a
repeat of the setup used for APDB tests at PDAC. The goals of this test were to
verify that things work as expected, that performance is not worse than what
was observed in PDAC tests, and to set a baseline for scalability testing. For
this and all following tests we used replication factor 3 and consistency level
``QUORUM`` for both reading and writing.

In total 50k visits were generated in this test. Performance looked similar to
what was observed at PDAC with same `linear growth
<_static/apdb-gcp1-nb-time-select-fit.png>`_ of reading time with the number of
visits. Actual reading time has improved slightly compared to `PDAC
<https://dmtn-156.lsst.io/#three-replica-cassandra-test>`_, at 50k visits total
select time for all three tables is below 5 seconds, while PDAC performance
was closer to 7 seconds for the same 50k visits.


Six-node Setup
==============

Next test (`DM-28154`_) was to check how does performance improve with
increased cluster size by doubling the number of nodes. Ideally Cassandra
performance should scale linearly with the number of nodes and reading time
should reduce by 50%.

On the first iteration 75k visits were generated but `performance
<_static/apdb-gcp2-nb-time-select-fit.png>`_ did not improve as expected, total
read time at 50k visits was around 3.1 sec which is about 35% improvement
compared to three-node case. Profiling showed that significant fraction of time
on client side was spent on converting the data into ``afw.Table`` format which
was used as a return data type for APDB API in these tests.

As AP pipeline already decided to switch to ``pandas`` instead of ``afw.Table``
for its internal presentation we decided to switch to ``pandas`` as well and
repeated six-node test using that data format. There were 100k visits generated
in second iteration of this test, with ``pandas`` total read time reduced to
1.67 seconds at 50k (and 3.26 sec at 100k visits) and CPU time on client side
reduced dramatically. We did not repeat three-node test with ``pandas``,
instead we proceeded to a larger scale test.


Twelve-node Setup
=================

Next test (`DM-28172`_) increased scale of Cassandra cluster to 12 nodes. The
goal for this test was to observe scaling behavior but also to extend the test
beyond 12 months of visits to see how performance behaves when history size for
DiaSources and DiaForcedSources stops growing. For this test we increased SSD
storage size to 3TB (8 partitions) on each node. In total there were 450k
visits generated in this test.

Scaling Behavior
----------------

Compared to previous six-node test performance improved significantly and total
read time dropped to 1.65 sec at 100k visits, which is about 50% improvement
compared to six-node setup.

:numref:`apdb-gcp-scaling-plot.png` shows summary of scaling behavior for all
above tests. After fixing excessive CPU usage by ``afw.Table`` conversion code
scaling behavior practically matches ideal `1/N` curve.


.. figure:: /_static/apdb-gcp-scaling-plot.png
   :name: apdb-gcp-scaling-plot.png
   :target: _static/apdb-gcp-scaling-plot.png

   Summary of reading performance as a function of number of nodes. Performance
   is given as a total read time at 100k visits, with three-node and six-node
   ``afw.Table`` cases extrapolated to 100k. Curves represent ideal ``1/N``
   scaling behavior.

Twelve-month Plateau
--------------------

As seen on all previous plots select time is growing linearly with the number
of DiaSources and DiaForcedSources. Those numbers are determined by the size of
the history read from database which is set to 12 months. We expect the numbers
to plateau after 12 months and reading time to stabilize with it. To
demonstrate this we generated 450k of visits which corresponds approximately to
18 month of date. ``ap_proto`` generates 800 visits per night which translates
into 288k visits for twelve 30-night "months".

:numref:`apdb-gcp4-nb-time-select-real.png` shows select time behavior for the
whole range of generated visits. It is clear that after 300k visits select time
stabilizes at the level of 4.5 seconds per visit. The sawtooth-like
fluctuations after 300k visits are related to the time partitioning scale which
is 30 days in this case. This plot shows select time which is averaged over
multiple of visits, there are of course significant visit-to-visit
fluctuation. :numref:`apdb-gcp4-nb-time-scatter.png` shows scatter plot for
select and insert times for individual visits without averaging, visit-to-visit
fluctuations are clearly visible but they stay in reasonable range.

.. figure:: /_static/apdb-gcp4-nb-time-select-real.png
   :name: apdb-gcp4-nb-time-select-real.png
   :target: _static/apdb-gcp4-nb-time-select-real.png

   Time to read as a function of visit for all three tables, ``select_real`` is
   a sum ot three other values. Total time plateaus after approximately 300k
   visits, small fluctuations are due to granularity of time partitioning.
 
.. figure:: /_static/apdb-gcp4-nb-time-scatter.png
   :name: apdb-gcp4-nb-time-scatter.png
   :target: _static/apdb-gcp4-nb-time-scatter.png

   Scatter plot for select and insert time showing times for individual visits.
   Blue markers correspond to averaged green markers on the above plot.
 

Partitioning Options
====================

For all of the above test we used identical partitioning options:

- MQ3C(10) spatial partitioning
- 30 day time partitioning for DiaSource and DiaForcedSource
- time partition is not using Cassandra partitioning but separate
  per-partition tables instead

Optimal partition sizes should provide a balance between number of partitions
queried and the size of the data returned. Smaller partition size will reduce
overhead in the size of the returned data but will increase the number of
queries needed to select the data. Time partitioning is implemented using
separate per-month tables, this is done to simplify management of the data
beyond 12 month. Older data that will not be queried after 12 months can be
moved to slower storage or archived to save on SSD storage cost, that process
would be easier to implement with the data in separate tables.

Part of epic was devoted to testing possible options for partitioning that
could potentially improve performance which are described below.

Partitioning Granularity
------------------------

Reducing partition granularity decreases number of partitions and consequently
the number of separate queries that need to be executed to get the same data
which could have an impact on server performance. To check that we reduced the
size of the timing partitions from 30 days to 60 days and re-ran the test
(`DM-28467`_). There was no visible change in timing for select queries on
client side while server side monitoring showed some moderate improvement in
resource use. Given that overall performance does not improve it makes sense to
keep granularity at 1 month to limit the overhead in the size of the data
returned to clients.

Native Time Partitioning
------------------------

While using separate-table partitioning for time dimension has management
benefits it could also have some performance impact. To quantify it we
performed a test where separate-table partitioning mechanism was replaced with
the native Cassandra partitioning (`DM-28522`_).

As before no significant difference in select time was observed after this
change.

Query Format
------------

Cassandra query language is limited in what it can do but there is some freedom
in how queries can be formulated to select data from multiple partitions:

- execute single query specifying all partitions in ``IN()`` expression, e.g.
  ``SELECT ... WHERE partition IN (...)``
- execute multiple queries, one query per partition, e.g. ``SELECT ... WHERE
  partition = ...``

Difference between these two options is where the merging of the results
happens, in former case merge is done on server side by coordinator node, in
latter case client is responsible for merging.

We have tested both options for time partition (when time was natively
partitioned) and did not find significant difference in performance between
them. Queries cover only 13 time partitions, for spatial index number of
partitions per visit is higher. When we tried extreme case with individual
queries for each temporal and spatial partition then total number of separate
queries grew to more than 200. Client side performance in this case was
significantly worse with client spending significant CPU time on processing
of the multiple results.


Packed Data
===========

Schema of the Cassandra tables follows definition outlined in DPDD. DiaObject
and DiaSource tables are very wide and have large number of columns. Most of
those columns are never used by Cassandra, there are no indices defined for
them and queries do not use them. Management overhead for the schema could be
reduced if bulk of that data is stored in some opaque form. Packing most
columns in a BLOB-like structure on client side could have some benefits but
may also have some serious drawbacks:

- server-side operations may become faster if server does not need to care
  about individual columns
- potential schema change management may be simplified
- if packing format is dynamic it needs extra space for column mapping
- significantly more work needed on client side to pack/unpack the data

A simple test was done to check how this might work (`DM-28820`_). For
serialization of records we used `CBOR <https://cbor.io/>`_ which is a compact
binary JSON-like format. CBOR structure is dynamic and needs to pack all column
names with the data thus inflating the size of the BLOB. Cassandra uses
compression for the data saved on disk which could offset some of that inflated
size.

The results from this test show that performance is slower in this case and it
is caused by significantly higher CPU time on client side spent on query
result conversion. Attempts to optimize conversion were only partially
successful, improvements may be possible in general but would require doing
much of the conversion in C++.

Disk usage in Cassandra has increased by factor of two even if compression
ratio for the data increased. Given all these observation our simple approach
clearly does not result in improvement. It may still be possible to achieve
some gains with packing but it would require significant effort to use fixed
schema on client side and optimizing conversion performance.

Pandas Performance
------------------

The results of this test also show a potential for improvement. Converting
query results to ``pandas`` format requires significant effort on client side.
The main reason for this is a mismatch between data representation used by
Cassandra client and ``pandas``. Cassandra client produces result data as a
sequence of tuples which is a close match to wire-level protocol. ``pandas`` on
the other hand keeps the data in memory as a set of two-dimensional arrays.
Transformation of tuples to arrays involves a lot of iterations that all happen
in Python level. If further improvements for conversion is necessary one could
think of either replacing ``pandas`` with a format that better matches
Cassandra representation or rewriting expensive parts of conversion in C++.


High Availability
=================

One unplanned test happened by accident but allowed us to check how does high
availability feature of Cassandra perform (`DM-28522`_). One of the eight
Cassandra nodes was misconfigured and its server became unavailable for several
hours. Despite that cluster continued functioning normally without much of
impact on performance. Both read and write latencies stayed at the same level,
though obviously timeouts did happen when some clients that connected to that
particular instance had to wait for response before cluster declared that node
as dead.

After the instance was reconfigured and re-joined the cluster all operations
continued and monitoring showed that data recovery on the temporary unavailable
node worked as expected. This incident shows that Cassandra can function
without service degradation when one replica becomes inaccessible. Cassandra
has a flexible consistency model which can be tuned for particular operation
model.


Other observations
==================

High CPU Usage
--------------

Monitoring Cassandra cluster showed that occasionally one or two servers could
start showing high CPU usage compared to all other servers. It does not seem to
affect performance very much, a noticeable effect is seen only on write latency
which still stays reasonably low. It seems that the issue can be mitigated by
restarting that particular instance, after restart CPU usage returns to normal.
It may be related to how cluster is initialized as it was only seen when
cluster was re-initialized from scratch. We tried to get some advice from
Cassandra developers but none of the suggestion we received helped to
understand the cause of the issue.

What Was Not Tested
-------------------

The tests with AP prototype represent just a basic part of AP pipeline
operation. Some more complicated options are not implemented in the prototype,
in particular:

- Day-time re-association of DiaSources to SSObjects is not implemented and
  was not tested. Due to Cassandra architecture update operations are not
  trivial and may have some impact on later read requests. It may be possible
  to avoid it completely by splitting table schema, and it clearly deserves
  separate test.
- Possible concurrent access to APDB data by other clients was not tested, at
  this point it is not clear what these other clients could be.
- Variability of DiaSources density, ``ap_proto`` currently uses uniform
  distribution for DiaSources. It would be interesting to see the effect of
  non-uniformity on performance.
- Data management aspect of the operations was not tested. Such operations
  could include archiving or removal of older data and cleanup of the tables.
  This aspect will need to be understood and tested as well.


Conclusion
==========

We tested the APDB prototype against Cassandra cluster running on GCP using
different options for cluster size and its operating parameters. Twelve-node
Cassandra cluster seem to provide performance that can be adequate for AP
pipeline operation for the scale of one year and beyond. The tests also provide
valuable insight into operation of Cassandra cluster and potential for
performance improvements on client side.



.. _DMTN-156: https://dmtn-156.lsst.io/
.. _DMTN-162: https://dmtn-162.lsst.io/
.. _DM-27785: https://jira.lsstcorp.org/browse/DM-27785
.. _DM-28136: https://jira.lsstcorp.org/browse/DM-28136
.. _DM-28154: https://jira.lsstcorp.org/browse/DM-28154
.. _DM-28172: https://jira.lsstcorp.org/browse/DM-28172
.. _DM-28467: https://jira.lsstcorp.org/browse/DM-28467
.. _DM-28522: https://jira.lsstcorp.org/browse/DM-28522
.. _DM-28820: https://jira.lsstcorp.org/browse/DM-28820
