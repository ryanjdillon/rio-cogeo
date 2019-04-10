=========
rio-cogeo
=========

Cloud Optimized GeoTIFF (COG) creation and validation plugin for Rasterio

.. image:: https://badge.fury.io/py/rio-cogeo.svg
    :target: https://badge.fury.io/py/rio-cogeo

.. image:: https://circleci.com/gh/cogeotiff/rio-cogeo.svg?style=svg
   :target: https://circleci.com/gh/cogeotiff/rio-cogeo

.. image:: https://codecov.io/gh/cogeotiff/rio-cogeo/branch/master/graph/badge.svg?token=zuHupC20cG
   :target: https://codecov.io/gh/cogeotiff/rio-cogeo


Cloud Optimized GeoTIFF
=======================

This plugin aim to facilitate the creation and validation of Cloud Optimized
GeoTIFF (COG or COGEO). While it respects the
`COG specifications <https://github.com/cogeotiff/cog-spec/blob/master/spec.md>`__, this plugin also
enforce several features:

- **Internal overviews** (User can remove overview with option `--overview-level 0`)
- **Internal tiles** (default profiles have 512x512 internal tiles)


Install
=======

.. code-block:: console

  $ pip install -U pip
  $ pip install rio-cogeo

Or install from source:

.. code-block:: console

   $ git clone https://github.com/cogeotiff/rio-cogeo.git
   $ cd rio-cogeo
   $ pip install -U pip
   $ pip install -e .


Usage
=====

.. code-block:: console

  $ rio cogeo --help
  Usage: rio cogeo [OPTIONS] COMMAND [ARGS]...

    Rasterio cogeo subcommands.

  Options:
    --help  Show this message and exit.

  Commands:
    create    Create COGEO
    validate  Validate COGEO

- Create a Cloud Optimized Geotiff.

.. code-block:: console

  $ rio cogeo --help
  Usage: rio cogeo [OPTIONS] INPUT OUTPUT

    Create Cloud Optimized Geotiff.

  Options:
    -b, --bidx BIDX                 Band indexes to copy.
    -p, --cog-profile [jpeg|webp|zstd|lzw|deflate|packbits|raw] CloudOptimized GeoTIFF profile (default: deflate).
    --nodata NUMBER|nan             Set nodata masking values for input dataset.
    --add-mask                      Force output dataset creation with an internal mask (convert alpha band or nodata to mask).
    --overview-level INTEGER        Overview level (if not provided, appropriate overview level will be selected until the
                                    smallest overview is smaller than the internal block size).
    --overview-resampling [nearest|bilinear|cubic|cubic_spline|lanczos|average|mode|gauss] Resampling algorithm.
    --overview-blocksize TEXT       Overview's internal tile size (default defined by GDAL_TIFF_OVR_BLOCKSIZE env or 128)
    --threads INTEGER
    --co, --profile NAME=VALUE      Driver specific creation options.See the documentation for the selected output driver for more information.
    -q, --quiet                     Suppress progress bar and other non-error output.
    --help                          Show this message and exit.

- Check if a Cloud Optimized Geotiff is valid.

.. code-block:: console

  $ rio cogeo validate --help
  Usage: rio cogeo validate [OPTIONS] INPUT

    Validate Cloud Optimized Geotiff.

  Options:
    --help  Show this message and exit.


Examples
========

.. code-block:: console

  # Create a COGEO with DEFLATE compression (Using default `Deflate` profile)
  $ rio cogeo create mydataset.tif mydataset_jpeg.tif

  # Validate COGEO
  $ rio cogeo validate mydataset_jpeg.tif

  # Create a COGEO with JPEG profile and the first 3 bands of the data and add internal mask
  $ rio cogeo create mydataset.tif mydataset_jpeg.tif -b 1,2,3 --add-mask --cog-profile jpeg


Default COGEO profiles
======================

**JPEG**

- JPEG compression
- PIXEL interleave
- YCbCr colorspace
- limited to uint8 datatype and 3 bands data

**WEBP**

- WEBP compression
- PIXEL interleave
- limited to uint8 datatype and 3 or 4 bands data
- Available for GDAL>=2.4.0

**ZSTD**

- ZSTD compression
- PIXEL interleave
- Available for GDAL>=2.3.0

*Note* in Nov 2018, there was a change in libtiff's ZSTD tags which create incompatibility for old ZSTD compressed GeoTIFF `link <https://lists.osgeo.org/pipermail/gdal-dev/2018-November/049289.html>`__

**LZW**

- LZW compression
- PIXEL interleave

**DEFLATE**

- DEFLATE compression
- PIXEL interleave

**PACKBITS**

- PACKBITS compression
- PIXEL interleave

**RAW**

- NO compression
- PIXEL interleave

Default profiles are tiled with 512x512 blocksizes.

Profiles can be extended by providing '--co' option in command line

.. code-block:: console

    # Create a COGEO without compression and with 1024x1024 block size and 256 overview blocksize
    $ rio cogeo create mydataset.tif mydataset_raw.tif --co BLOCKXSIZE=1024 --co BLOCKYSIZE=1024 --cog-profile raw --overview-blocksize 256


Overview levels
===============

By default rio cogeo will calculate the optimal overview level based on dataset
size and internal tile size (overview should not be smaller than internal tile
size (e.g 512px). Overview level will be translated to decimation level of
power of two:

.. code-block:: python

  overview_level = 3
  overviews = [2 ** j for j in range(1, overview_level + 1)]
  print(overviews)
  [2, 4, 8]

Internal tile size
==================

By default rio cogeo will create a dataset with 512x512 internal tile size.
This can be updated by passing `--co BLOCKXSIZE=64 --co BLOCKYSIZE=64` options.

**Web tiling optimization**

if the input dataset is aligned to web mercator grid, the internal tile size
should be equal to the web map tile size (256 or 512px). Dataset should be compressed.

if the input dataset is not aligned to web mercator grid, the tiler will need
to fetch multiple internal tiles. Because GDAL can merge range request, using
small internal tiles (e.g 128) will reduce the number of byte transfered and
minimized the useless bytes transfered.


GDAL configuration to merge consecutive range requests

.. code-block:: console

    GDAL_HTTP_MERGE_CONSECUTIVE_RANGES=YES
    GDAL_HTTP_MULTIPLEX=YES
    GDAL_HTTP_VERSION=2


GDAL Version
============

It is recommanded to use GDAL > 2.3.2. Previous version might not be able to
create proper COGs (ref: https://github.com/OSGeo/gdal/issues/754).


Nodata, Alpha and Mask
======================

By default rio-cogeo will forward any nodata value or alpha channel to the
output COG.

If your dataset type is **Byte** or **Unit16**, you could use internal bit mask
(with the `--add-mask` option) to replace the Nodata value or Alpha band in
output dataset (supported by most GDAL based backends).

Note: when adding a `mask` with an input dataset having an alpha band you'll
need to use the `bidx` options to remove it from the output dataset.

.. code-block:: console

  # Replace the alpha band by an internal mask
  $ rio cogeo mydataset_withalpha.tif mydataset_withmask.tif --cog-profile raw --add-mask --bidx 1,2,3

**Important**

Using internal nodata value with lossy compression (`webp`, `jpeg`) is not
recommanded. Please use internal masking (or alpha band if using webp).


Statistics
==========

Some libraries might request to use COGs with statistics written in the internal
metadata. **rio-cogeo** doesn't calculate nor copy those when creating the output
dataset (because statistics may change due to lossy compression).
To add the statistics to the output dataset you could use the code above:

.. code-block:: python

  import rasterio

  with rasterio.open("my-data.tif", "r+") as src_dst:
      for b in src_dst.indexes:
          band = src_dst.read(indexes=b, masked=masked)
          stats = {
              'min': float(band.min()),
              'max': float(band.max()),
              'mean': float(band.mean())
              'stddev': float(band.std())
          }
          src_dst.update_tags(b, **stats)


Contribution & Development
==========================

The rio-cogeo project was begun at Mapbox and has been transferred to the
CogeoTIFF organization in January 2019.

Issues and pull requests are more than welcome.

**dev install**

.. code-block:: console

  $ git clone https://github.com/cogeotiff/rio-cogeo.git
  $ cd rio-cogeo
  $ pip install -e .[dev]

**Python3.6 only**

This repo is set to use `pre-commit` to run *flake8*, *pydocstring* and *black*
("uncompromising Python code formatter") when commiting new code.

.. code-block:: console

  $ pre-commit install

Extras
======

Blog post on good and bad COG formats: https://medium.com/@_VincentS_/do-you-really-want-people-using-your-data-ec94cd94dc3f

Checkout `**rio-glui** <https://github.com/mapbox/rio-glui/>__` rasterio plugin to explore COG locally in your web browser.
