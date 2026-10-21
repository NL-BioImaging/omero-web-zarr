.. image:: https://github.com/NL-BioImaging/omero-web-zarr/actions/workflows/provider.yml/badge.svg
    :target: https://github.com/NL-BioImaging/omero-web-zarr/actions/workflows/provider.yml

.. image:: https://badge.fury.io/py/omero-web-zarr.svg
    :target: https://badge.fury.io/py/omero-web-zarr

omero-web-zarr
==============

OMERO.web plugin for OME-Zarr.

NL-BioImaging performance fork
-----------------------------

This fork of `ome/omero-web-zarr <https://github.com/ome/omero-web-zarr>`_
starts from upstream commit ``4e1e685dff22136abfd2626f6fd6eaf985aba368``.
Its Python version is ``0.1.2.dev0+nlbi.2``; it is installed from a pinned
Git commit, not published as an upstream or PyPI release.
It serves ordinary OMERO images through authenticated virtual OME-Zarr HTTP
endpoints. No BIOMERO, Slurm, Analysis or exported Zarr store is required.
It produces OME-NGFF 0.3/0.4 with Zarr v2 arrays and optionally OME-Zarr 0.5
with unsharded Zarr v3 arrays, using separate versioned routes.
The separate `BIOMERO.ZarrViewer <https://github.com/NL-BioImaging/BIOMERO.ZarrViewer>`_
can also read physical OME-Zarr 0.4/0.5 stores without this optional provider.

Changes relative to upstream
~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Build array metadata directly and encode lossless Zstd level-1 chunks in
  memory, eliminating temporary Zarr directories and file reads. All-zero
  chunks remain valid encoded chunks.
* Read tiles up to 1024 by 1024 pixels and group up to eight Z planes per
  chunk, with a 2 MiB target. T and C chunks remain one. Partial edge chunks
  are zero padded, and singleton axes retain upstream conventions.
* Use one RawPixelsStore per chunk, select the actual native pyramid
  resolution, convert network byte order to the declared dtype, and always
  close pixel readers and metadata rendering engines.
* Expose ``CHUNK_METADATA_API=1`` for trusted in-process callers that already
  checked image permissions and metadata. Ordinary HTTP routes still resolve
  image access themselves. ``OPTIMIZED_LAYOUT=v2-z8-1024`` identifies the chunk
  addressing scheme; consumers must invalidate cached metadata when it changes.

The viewer's progress bar, browser caches, deferred contrast statistics and
non-binary pyramid display fixes live in ZarrViewer, not in this provider.
This fork remains an experimental HTTP provider under the upstream AGPL license.

Optional OME-Zarr 0.5
~~~~~~~~~~~~~~~~~~~~~

The existing 0.3/0.4 routes retain their Zarr v2 layout. Clients can explicitly
select the additional 0.5 representation::

    /zarr/v0.5/image/[ID].zarr/zarr.json
    /zarr/v0.5/image/[ID].zarr/0/zarr.json
    /zarr/v0.5/image/[ID].zarr/0/c/[chunk-coordinates]

The group metadata uses ``attributes.ome.version=0.5``. Arrays include named
dimensions, a regular chunk grid, little-endian bytes and lossless Zstd level 1
without a checksum. There is no sharding. Pixel access, chunk shapes and native
pyramid levels are shared with 0.4; a format switch alone promises no speedup.
Physical XYZ calibration is included in the 0.5 scale transformations.
Image permissions apply to every 0.5 metadata and chunk route.

ZarrViewer keeps HTTP NGFF 0.4 as its default. With a compatible viewer/provider
and Zarr-Python 3, explicitly select the optional format with::

    omero config set omero.web.zarr_viewer.http_ngff_version 0.5

Restart OMERO.web to apply this setting. Set it back to ``0.4`` to restore
the default representation. This setting affects ordinary virtual HTTP images;
physical OME-Zarr 0.4/0.5 and BIOMERO registrations retain their existing paths.

Installation with ZarrViewer
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Choose the Docker recipe for your deployment:

* **Standalone OMERO, without BIOMERO:** use the `standalone installation guide
  <https://github.com/NL-BioImaging/BIOMERO.ZarrViewer/blob/0.8.1-beta-standalone/docs/http-zarr.md#reproduce-the-standalone-docker-installation>`_
  and `Dockerfile.standalone
  <https://github.com/NL-BioImaging/BIOMERO.ZarrViewer/blob/0.8.1-beta-standalone/docker/Dockerfile.standalone>`_.
* **Existing BIOMERO stack:** use the `BIOMERO installation guide
  <https://github.com/NL-BioImaging/BIOMERO.ZarrViewer/blob/0.8.1-beta-standalone/docs/deployment-nl-biomero.md#optional-pinned-http-zarr-provider-in-an-existing-biomero-stack>`_
  and `Dockerfile.biomero
  <https://github.com/NL-BioImaging/BIOMERO.ZarrViewer/blob/0.8.1-beta-standalone/docker/Dockerfile.biomero>`_.
  This extends your combined web image, preserving BIOMERO, Analysis,
  WorkflowSkills, the installed viewer and existing storage configuration.

Both recipes install this optional provider with ``ENABLE_HTTP_ZARR=true``
and pin its commit and archive checksum. The guides cover build, configuration
and deployment checks. Leaving the option false in the BIOMERO recipe keeps
any provider inherited from the base image unchanged.

For another OMERO.web environment, install a reviewed full commit::

    pip install "omero-web-zarr @ git+https://github.com/NL-BioImaging/omero-web-zarr.git@<full-commit>"
    omero config append omero.web.apps '"omero_web_zarr"'

To enable ZarrViewer's optional integration, also set::

    omero config set omero.web.zarr_viewer.omero_web_zarr_enabled true

Restart OMERO.web after changing its installed apps. Keep source data and the
OMERO database unchanged; the provider reads existing OMERO-managed pixels.

Validation and performance
~~~~~~~~~~~~~~~~~~~~~~~~~~

``test/check_provider.py`` exercises the installed functions with deterministic
pixel readers: black data, uint16 byte order, grouped T/C/Z, padded edges and
native pyramid resolutions. It compares every decoded pixel and verifies
reader closure. The Provider regression workflow runs this against OMERO.web
5.33.2 and Zarr-Python 3.4.1. This is a codec/reader regression check, not a live
server integration test. Upstream's OMERO infrastructure workflow is retained
for the upstream repository; this fork does not publish packages to PyPI.

Separately, local Docker validation on 20 October 2026 used OMERO.server
5.6.18 and OMERO.web 5.33.2 without BIOMERO. An Aperio CMU-1.svs image
(46000 x 32914 RGB; 0.499 micrometre pixels) rendered at overview and native
resolution. A calibrated confocal RGB stack (370 x 370 x 36 Z) produced an
exact maximum projection matching the source pixel hash. Its grouped layout
required 15 pixel HTTP requests instead of the previous 108; one local run
downloaded those chunks in approximately one second. These are local pixel
transfer measurements, not portable end-to-end loading benchmarks.
Physical OME-Zarr 0.4/0.5 regression reads in ZarrViewer also matched all pixels.

The `standalone 0.4 versus 0.5 comparison
<https://github.com/NL-BioImaging/BIOMERO.ZarrViewer/blob/0.8.1-beta-standalone/docs/http-zarr.md#standalone-comparison-of-virtual-04-and-05>`_
records the 21 October 2026 measurements and raw trials. SVS overview/native
pixels and the complete confocal MIP matched exactly, with the same request
counts, chunk shapes and compressed pixel byte counts. Pixel-transfer medians
were close; these results do not establish a large format-related speedup.

Upstream documentation
----------------------

Implementation of `OME-NGFF <https://ngff.openmicroscopy.org/latest/>`_ API with an omero-web app.

This plugin supports OME-NGFF v03 and v0.4.
For a given Image ID in OMERO, the following URL will refer to an OME-NGFF image::

    https://[omero-server]/zarr/v0.4/image/[ID].zarr/

Note: This app has not been extensively tested and should **not** be considered "production ready".

Currently supports `OME-NGFF v0.3 <https://ngff.openmicroscopy.org/0.3/index.html>`_ and
`OME-NGFF v0.4 <https://ngff.openmicroscopy.org/0.4/index.html>`_.

Development
-----------

Install with::

    $ pip install -e .

Configuration

::

    $ omero config append omero.web.apps '"omero_web_zarr"'

    # Allow to open-with Vizarr

    $ omero config append omero.web.open_with '["web_zarr_vizarr", "omero_web_zarr_index", {"supported_objects":["image"], "label": "Vizarr", "script_url": "omero_web_zarr/openwith.js"}]'

    # Open with ome-ngff-validator

    $ omero config append omero.web.open_with '["web_zarr_validator", "omero_web_zarr_index", {"supported_objects":["image"], "label": "NGFF validator", "script_url": "omero_web_zarr/openwith_validator.js"}]'


Then you will be able to access OMERO Images in OME-NGFF format v0.3 or v0.4 with a URLs like::

    # base URL for Image ID
    [omero-server]/zarr/v0.4/image/[ID].zarr

    # URLS for .zattrs, .zgroup
    [omero-server]/zarr/v0.4/image/[ID].zarr/.zattrs
    [omero-server]/zarr/v0.4/image/[ID].zarr/.zgroup

    # .zarray of the dataset at path '0'
    [omero-server]/zarr/v0.4/image/[ID].zarr/0/.zarray

    # first 3D chunk of the dataset at path '0'
    [omero-server]/zarr/v0.4/image/[ID].zarr/0/0/0/0


You can see this in action using the `Vizarr <https://github.com/hms-dbmi/vizarr/>`_ viewer.

This omero-web app self-hosts Vizarr to avoid CORS issues (delegating to https://hms-dbmi.github.io/vizarr/).

In the webclient UI you can use the context menu to `Open With > Vizarr`, or use your Image ID and go directly to::

    [omero-server]/zarr/vizarr/?source=[omero-server]/zarr/v0.4/image/[ID].zarr

Testing
-------

To run integration tests (in your omero-web conda environment above) with `pytest`.
See `OMERO testing docs <https://docs.openmicroscopy.org/latest/omero/developers/testing.html>`_
for setting `ICE_CONFIG` and dependencies etc., then::

    $ pytest test/integration/test_ngff.py

License
-------

The application is released under the AGPL.

Copyright
---------

2022-2023, The Open Microscopy Environment

