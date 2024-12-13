title: Compute Resource Sharing Overview

Compute Resource Sharing Overview
=================================

!!! info "Schedule a consultation"
    To ensure that we can meet your research computing goals needs, please contact us at <support@osg-htc.org> to schedule a consultation.

OSG uses a resource-overlay ("pilot") model to share resources from your local cluster:
compute resources are added to a large central resource pool in the OSG through the use of a bootstrap process, often called a
_pilot_ or a _glidein_.
These pilots, in turn, download and execute user jobs from an OSG community (also known as "payloads") from the resource
pool to run within the pilots.

On OSG, there are several resource pools, one for each large community (such as ATLAS or CMS) and the special-purpose
_Open Science Pool_.
The latter focuses on aggregating resources together for small researcher-driven groups and is operated by the OSG
itself.

There are several ways pilots can join a resource pool:

* Submitted to your local batch system by a [*compute entrypoint*](../compute-element/htcondor-ce-overview.md) (CE).
  These jobs are created by an external entity, a *pilot factory* based on observed demand in the pool.
  The CE is the most common way to receive pilot jobs since they integrate with automated processes that are responsive
  to existing demand.
* Sites can launch [*pilot containers*](os-backfill-containers.md) when they have local resources they
  would like to contribute directly to a specific OSG pool.
  The site-launched *pilot container* method is useful for backfilling resources without the need for a batch system;
  however, at times these pilots may stay idle because there is insufficient demand within the resource pool.
* Users can launch *personal pilot containers* within a site's batch system so they can use an existing share or
  allocation at a site through the open science pool.
