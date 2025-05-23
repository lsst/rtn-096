##############################################
Observatory Metadata Flow and Usage Guidelines
##############################################

.. abstract::

   In this technote we describe the usage and propagation of the different Metadata keywords available for describing and identifying datasets. Metadata keywords are traced from data acquisition through to the different endpoints where they can be used and guidelines are provided for how to set and use the keywords.



Introduction
============

Metadata keywords are used to identify and describe datasets.
These keys need to be able to identify subsets of data for a range of purposes, including

- finding all data from a given test case
- indicating visits that may need special processing such as deep drilling fields
- triggering processing for the next visit in the nextVisit event

The needs for metadata may be different during commissioning than during operations.

Metadata for images can be considered to include information about the pointing location,
such as RA, Dec, altitude, azimuth, and band.


Available Observatory Metadata Keys
===================================

The details of available metadata varies slightly with where this metadata is retrieved from.
Options for retrieving information include the Consolidated Database (ConsDb), the Butler, and the EFD.
Inputs for the metadata mostly originate at the SchedulerCSC,
where information can be received from the Feature Based Scheduler (FBS)
or directly from JSON BLOCKs and scripts.

The metadata keys include:

- target_name: a particular name for the pointing(s)
- observation_reason: a reason for this particular observation(s)
- science_program: the general science program
- scheduler_note or note: commentary about a particular observation(s)
- img_type: the kind of image (OBJECT, BIAS, DARK, FLAT, ENGTEST, ACQ, CWFS ..)
- group_id: an identifier for a group of images which should be processed together in pipelines
- target_id: an integer identifying the id of the requested observation from the FBS

The img_type and group_id are set fairly deeply within scripts for obtaining images,
so are useful for identifying kinds of data but are fundamentally tied into data processing,
and should be set with that in mind.

The target_id is only set by the FBS and should (?) be 0 for all other observations.
This is just a counter to enable tracking from requested targets to completed observations.

The target_name passes through the pointing component of the CSCs,
and will only update when a script passes a new value for the target name.
Observations taken after a named target may retain the same target_name,
even if it was not relevant or even at the same pointing on the sky
(thus target_name by itself is not suitable for uniquely identifying observations for a particular purpose).

The observation_reason and science_program are the most flexible pieces of metadata.
In general, science_program should be linked to the JSON BLOCK used to acquire those visits.
By convention this is would then be the "biggest bucket" identifying visits linked to a particular program.
For visits acquired via the FBS, the science_program *must* match the JSON BLOCK.
This leaves observation_reason available to provide commentary about particular observations within
that program, such as marking an offset to the focus or identifying visits as reference for that
program.

For visits acquired using the FBS, the scheduler_note/note should generally be populated by the FBS directly.
The FBS uses this metadata for tracking purposes under its own schema.
For observations which are acquired using methods clearly separable from science observations with the FBS,
such as being obtained using a science_program or img_type that never is used by the FBS,
the scheduler_note/note field can also be used freely.
There is some unresolved ambiguity about the name of this metadata key,
since it has not arrived at the ConsDB yet.

To identify subsets of visits under a science program,
it is likely that a combination of target_name and observation_reason will be useful.


Observatory Metadata Flow
=========================

.. mermaid:: metadata_flow.mmd

science_program
---------------
For observations being run from the FBS,
the science_program must match the name of the JSON BLOCK being used to acquire the visit.
The value of the "program" within that JSON BLOCK could be overriden with a different string,
and program can also be overriden in the scripts called from the BLOCK.

Some existing values for science_program::

    BLOCK-T215, BLOCK-320, spec-survey


observation_reason
------------------
The observation_reason can be set from the FBS or can be overriden in the JSON BLOCK or script.

Some existing values for observation_reason::

    INTRA_SITCOM-1491v2, EXTRA_SITCOM-1491v2, INFOCUS_SITCOM-1491v2, 'Rotator_Check',
    x_offset, x_offset_-50, x_offset_50


target_name
-----------
The target_name can be set from the FBS but is also often set from the scripts run from the JSON BLOCK.

Some existing values for target_name::

    Fornax_dSph, NGC1097, BLOCK-T81, slew_icrs, FeatureSchedulerTarget


scheduler_note / note
---------------------
Set by the FBS or by a JSON BLOCK.

Some possible future values for scheduler_note::

    CANDIDATE:HD60753, 'blob_long, gr, a', 'DD:XMM_LSS, 314', 'pair_15, iz, b', 'pair_33, iz, b'


img_type
--------
Set from the scripts.

Some existing values for img_type::

    CWFS, BIAS, DARK, FLAT, OBJECT, ACQ, FOCUS, ENGTEST, STUTTERED


group_id
--------
Set by the ScriptQueue, potentially using additional code in the script.
For a limited set of calibration JSON BLOCKs run via the "add_block" command (not through the FBS),
the group_id will be the BLOCK-XXX number plus day_obs an incrementing integer number.
For most blocks,
the group_id is a string starting with the time that the observing script associated with that exposure reaches the top of the ScriptQueue.

Some existing values for group_id::

    2024-10-25T03:51:17.756#1, BT220_O_20241024_000001, 2024-11-06T04:40:51.280, 2024-10-25T03:36:54.070#2,




target_id
---------

Set by the FBS.


Requested targets from the FBS are retrievable in the EFD under the lsst.sal.Scheduler.logevent_target topic;
it is worth noting that only the "note" and "target_name" are saved in this topic (observation_reason and science_program are not).
Requested targets for which the entire JSON BLOCK scripts execute successfully are recorded in the
lsst.sal.Scheduler.logevent_observation topic;
information can be linked from target to observation using targetId.
Acquired visits (there may be several visits per target or observation) arrive in the Butler and ConsDb.
In the ConsDb, the targetId is (not yet) available to link to the original requested target.
I assume that targetId would become target_id at the ConsDB (?).


.. list-table:: Observatory Metadata
   :widths: 25 25 25 50 25 25 25 25
   :header-rows: 1

   * - FBS
     - JSON BLOCK
     - nextVisit
     - Final CSC
     - FITS header
     - ObservationInfo
     - Butler exposure records
     - ConsDB
   * - science_program
     - program
     - survey
     - Camera.startIntegration.additionalValues
     - PROGRAM
     - science_program
     - science_program
     - science_program
   * - observation_reason
     - reason
     - -- (needed)
     - Camera.startIntegration.additionalValues
     - REASON
     - observation_reason
     - observation_reason
     - observation_reason
   * - target_name
     - name
     - -- (needed)
     - Ptg.currentTarget.targetName
     - OBJECT
     - target_name
     - target_name
     - target_name
   * - scheduler_note
     - note
     - --
     - Camera.imageReadoutParameters.daqAnnotation
     - OBSANNOT
     - --
     - --
     - -- (needed)
   * - N/A
     - N/A
     - groupId
     - Camera.startIntegration.additionalValues
     - GROUPID
     - exposure_group
     - group
     - group_id
   * - N/A
     - N/A
     - -- (needed)
     - Camera.startIntegration.additionalValues
     - IMGTYPE (?)
     - observation_type
     - observation_type
     - img_type
   * - target_id
     - N/A
     - --
     - ?
     - ?
     - --
     - --
     - -- (needed)


All information available in the Butler exposure record::

    instrument, id, day_obs, group, physical_filter, obs_id,
    exposure_time, dark_time, observation_type, observation_reason,
    seq_num, seq_start, seq_end, target_name, science_program, tracking_ra,
    tracking_dec, sky_angle, azimuth, zenith_angle, has_simulated, can_see_sky,
    timespan (TAI)


Observatory Metadata Examples
=============================

Examples of some general uses of these metadata keys.

AOS triplets used observation_reason to indicate the extra-, intra-, and in-focus visits within their single JSON BLOCK.
The different JSON BLOCKs corresponded to different test cases.
Tests that were being run at a location on the sky that corresponded to a named target
(or acquired directly after a previous JSON BLOCK with a named target, without resetting the target_name)
also had target_name available.


General baseline survey observations will use science_program to indicate the JSON BLOCK to use for the observations
(which take all parameters for a single slew + track + acquire visit as configuration parameters)
and scheduler_note as a value to indicate which Survey within the FBS CoreScheduler the visit should be 'credited'  as.
The target_id will be set as a running integer to link requested Targets to EFD Observations (and then to ConsDB Visits).
The target_name would be set according to the corresponding region(s) in the survey footprint.
The observation_reason could be set to indicate low dust WFD, NES, or DD or near-sun twilight microsurvey (etc),
matching a generalized description of the observing mode.
This would leave science_program as either an additional spare key to indicate "visits within the LSST"
(as opposed to engineering visits) or could be used to convey some additional higher level information.


Observatory Metadata Users
==============================

Add specific requirements here.

Survey scheduling
-----------------

The survey scheduling team needs:

- a way to uniquely identify all visits acquired with the FBS that were acquired with the "baseline survey" survey strategy configuration (vs. FBS for other uses or visits acquired for engineering via BLOCKs).
- to recapture the scheduler_note content, as it was originally set by the FBS (and not overriden in a JSON BLOCK or script)
- to be able to query this information from the ConsDB visits table
- a way to tell the Scheduler which JSON BLOCK to use for a given visit (currently uses science_program)
- a way to link acquired visits in the ConsDB with requested targets in the EFD (expect to use targetId)

Additional nice things for the survey scheduling team include use of the target_name.
This is a compact way for us to pull out some of the content of the scheduler_note without having to do string subsetting.


Prompt processing
-----------------

Prompt processing needs a way to identify which visits to trigger prompt processing,
using the 'nextVisit' event captured in lsst.sal.ScriptQueue.logevent_nextVisit.
Currently the only metadata key propagated to nextVisit is a "survey" key,
which corresponds to the science_program.
Potentially, the contents of the "survey" key could be modified separately from the science_program value,
by (for example) adding observation_reason or target_name.
PP can create a list of science_programs which will trigger prompt processing.


Calibration
-----------

Calibration processing needs a way to unique identify data within a single "run" of a calibration process.
If the data generated from a "run" is the result of running a single JSON BLOCK,
the group_id could be set as it currently is for some of the calibration scripts.
Specifically, for these calibration data taking scripts,
there is a feature that allows the group_id for all images taken together in a single JSON BLOCK to be set to match the unique observation id that is assigned to the JSON BLOCK execution.
This observation id is generated by the image naming service, and has the format BT220_O_20241024_000001,
which corresponds to the first execution 001 of BLOCK-T220 on 2024-10-24."
This allows users to uniquely identify calibration runs.

The science_program would then identify the JSON BLOCK used to generate the set of data,
with observation_reason identifying subsets of data within the block.
The group_id would identify the unique run of the block.

A potential drawback is that if the JSON BLOCK involves obtaining a significantly large amount of data,
and the BLOCK is interrupted (but cannot be paused),
then the BLOCK will have to be restarted from the beginning, starting another "run".  


Data Management
---------------

`DMTN-065 <https://dmtn-065.lsst.io>`_ :cite:`DMTN-065` lays out some requirements from Data Management for metadata applied to visits and catalogs in order to support general and special processing campaigns.
DMTN-065 requires a "region" label based on RA/Dec as well as an "observing mode" label to be applied to all visits and catalogs.
Examples in the technote for region labels include low-dust WFD, galactic plane WFD, dusty plane,
North Ecliptic Spur, Deep Drilling, and South Celestial Pole.
These generally match different areas on the sky defined in the baseline survey footprint.
As the region label is based solely on RA/Dec, it could be generated at any point
(and should be generated later, per-object, for catalog purposes).
A visit may overlap two different regions on the sky.
The observing model label is intended to be generated by the Feature Based Scheduler at the time of the requested target.
Examples for observing mode could include DD, ToO, near-sun twilight, triplet or engineering.
A visit would be acquired via a single primary mode.

Mapping the region and observing mode into the available metadata labels could be done in several ways.
Given that region labels could be multiple values (e.g. WFD + NES), this should not be the science_program,
but could be placed into either the target_name or the observing_reason.
Potentially, placing these labels for the target_name would collide with ToO target_names;
a solution would be to use the ToO name in the observation_reason and then simply label the target_name as usual.
This would then convey information about what part of the survey footprint the ToO landed within.
The observing mode could be placed into any of the metadata keys,
although would make most sense to be placed into observation_reason.

Requirements beyond DMTN-065?

References
==========

.. bibliography::
