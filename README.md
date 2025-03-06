# Lightcurve Pipeline

This repository contains a simple pipeline recipe for extracting lightcurves
from interferomter data. It is based on
https://github.com/ratt-ru/parrot-stew-recipes. The purpose of this recipe is
to provide a starting point for more sophisticated and potentially telescope
specific pipelines.

## 1GC

This recipe presumes that the data has already been had 1GC calibration
solutions applied i.e. it commences from the the standard self-cal loop.
Additionally, at present it requires the target to be extracted i.e. does
not handle field selection.

1GC can be accomplished using CARACal 1.0.6+
(https://github.com/caracal-pipeline/caracal).

## Installation

The pipline is implemented as a Stimela recipe (https://stimela.readthedocs.io),
and should be portable/reproducible given adequate hardware. Installation of
required depndencies can be accomplished by running the following (preferably
in a virtual environment):

```
git clone git@github.com:JSKenyon/lightcurve-pipeline.git
cd lightcuve-pipeline
pip install -e .  # Ensure you have updated pip first.
```

## Configuration

Some components of the pipeline will need to be configured for the data in
question. The `datasets.yml` file contains the majority of the regularly
changed parameters. However, any parameter can be adjusted by modifying the
contents of `lightcurve-pipeline.yml`.

## Running

In order to run the pipeline, ensure that you are in the `lightcurve_pipeline` 
directory.

The first step is to run the init steps of the pipeline. These steps are
typically run once and never again. This can be accomplished using:

```
stimela -C run lightcurve-pipline.yml -t init
```

Thereafter the remainder of the pipeline can be run using:
```
stimela -C run lightcurve-pipline.yml
```
