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

.. mermaid:: metadata_flow.mdd

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
the group_id will be the BLOCK-XXX number plus an incrementing integer number.
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
   :widths: 25 25 50 25 25 25 25
   :header-rows: 1

   * - FBS
     - JSON BLOCK
     - CSC
     - FITS header
     - ObservationInfo
     - Butler exposure records
     - ConsDB
   * - science_program
     - program
     - Camera.startIntegration.additionalValues
     - PROGRAM
     - science_program
     - science_program
     - science_program
   * - observation_reason
     - reason
     - Camera.startIntegration.additionalValues
     - REASON
     - observation_reason
     - observation_reason
     - observation_reason
   * - target_name
     - name
     - Ptg.currentTarget.targetName
     - OBJECT
     - target_name
     - target_name
     - target_name
   * - scheduler_note
     - note
     - Camera.imageReadoutParameters.daqAnnotation
     - OBSANNOT
     - N/A
     - N/A
     - N/A (yet)
   * - N/A
     - N/A
     - Camera.startIntegration.groupId
     - GROUPID
     - group_id
     - group_id
     - group_id
   * - target_id
     - N/A
     - ?
     - ?
     - N/A
     - N/A
     - N/A (yet?)



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
The target_name would be set for visits which have a named target (such as the DDF or ToO visits).
The observation_reason could be set to indicate WFD or DD or near-sun twilight microsurvey (etc).
The observation_reason could potentially be equivalent to the science_program.



Observatory Metadata Use-cases
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
PP can create a list of science_programs which will trigger prompt processing.


Calibration
-----------

Calibration processing needs a way to unique identify data within a single "run" of a calibration process.
This process will ideally be run by the FBS,
but will need to use several different JSON BLOCKs to acquire exposures using different scripts.
The science_program in the JSON BLOCKs could be overridden to match a single value if desirable.
The "run" identifier is not currently available and cannot easily be set without some additional machinery.
One option would be to track an incremental run ID in the FBS and store this in the observation_reason
or possibly scheduler_note.
A potential drawback might be if science_program is overriden in the BLOCK to be a single value for all
contributing content for the 'run', and observation_reason is only the incremental run id,
then the trace to the JSON BLOCK in use for a given observation could be lost.
