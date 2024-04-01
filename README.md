[![Website](https://img.shields.io/badge/dmtn--289-lsst.io-brightgreen.svg)](https://dmtn-289.lsst.io)
[![CI](https://github.com/lsst-dm/dmtn-289/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-dm/dmtn-289/actions/workflows/ci.yaml)

# Caching Database Content in Butler

## DMTN-289

There are dimension records and collection information that are used heavily when interacting with a Butler. This note describes a possible scheme for caching these records to reduce database access.

**Links:**

- Publication URL: https://dmtn-289.lsst.io
- Alternative editions: https://dmtn-289.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-289
- Build system: https://github.com/lsst-dm/dmtn-289/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

.. code-block:: bash

```sh
git clone https://github.com/lsst-dm/dmtn-289
cd dmtn-289
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://dmtn-289.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-289.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
