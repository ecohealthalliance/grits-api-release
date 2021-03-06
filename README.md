# grits-api

This project provides the backend for the GRITS [diagnostic-dashboard](https://github.com/ecohealthalliance/diagnostic-dashboard). The main API which it furnishes, accessible at `/diagnose`, takes an incoming document and returns a differential disease diagnosis and numerous extracted features for that document.

This project also provides resources for training the classifier model used to make disease predictions, and for managing long-running classification tasks over large corpora.

# Dependencies

Aside from the requirments noted in [requirements.txt](requirements.txt) which may be installed as usual with `pip install -r requirements.txt`, this project also relies on the annotation library [annie](https://github.com/ecohealthalliance/annie).

# Installation and set-up

## As part of total GRITS deployment

You may elect to install all GRITS components at once (this backend, the front-end [diagnostic-dashboard](https://github.com/ecohealthalliance/diagnostic-dashboard), and the [girder](https://github.com/ecohealthalliance/girder) database) by following the instructions in the [grits-deploy-scripts](https://github.com/ecohealthalliance/grits-deploy-scripts) project.

The provided deploy script [deploy.sh](deploy.sh) will fetch all dependencies, include nltk data and annie, and use `supervisorctl` to launch the API server and celery processes for managing diagnoses. There are 3 celery task queues, `priority`, `process` and `diagnose`. The process queue is for scraping and extracting articles prior to diagnosis. We recommend running a single threaded worker process on the process queue because it primarily makes http requests, so it spends most of it's time idling. The diagnose queue should have several worker processes as it is very CPU intensive. The priority queue is for both processing and diagnosing articles and should have a dedicated worker process for immediatly diagnosing individual articles. See the supervisor config in the grits-deploy-scripts for examples of how to initialize the various types of workers.

This deploy script relies on config files and other steps performed by [grtis-deploy-scripts](https://github.com/ecohealthalliance/grits-deploy-scripts), so should not be used in isolation.

## Standalone

To run this project in isolation, without deploying the entire GRITS suite, clone this repository:

    $ git@github.com:ecohealthalliance/grits-api.git

Copy the default config to the operative version and edit it to suit your environment:

    $ cp config.sample.py config.py

Install the pip requirements:

    $ sudo pip install -r requirements.txt

Get the [annie](https://github.com/ecohealthalliance/annie) project and make sure it's in your pythonpath.

Start a celery worker:

	$ celery worker -A tasks -Q priority --loglevel=INFO --concurrency=2

Start the server:

	# The -debug flag will run a celery worker synchronously in the same process,
	# so you can debug without starting a separate worker process.
	$ python server.py

## Full setup with virtualenv

These instructions will get `grits-api` working under a Python virtualenv.

First, choose a `WORKSPACE` for your Git checkouts.

    export WORKSPACE=~

Checkout grits-api and copy the default configuration.

    cd $WORKSPACE
    git clone git@github.com:ecohealthalliance/grits-api.git
    cd grits-api
    cp config.sample.py config.py

If you do not have `virtualenv`, first install it globally.

    sudo pip install virtualenv

Now create and enter the virtual environment. All `pip` and `python` commands from here should be run from within the environment. Leave the environment with the `deactivate` command.

    virtualenv venv
    source venv/bin/activate

Install `grits-api` dependencies and `nose`.

    pip install -r requirements.txt
    pip install nose

Checkout and install `annie`.

    cd $WORKSPACE
    git clone git@github.com:ecohealthalliance/annie.git
    cd annie
    pip install -r requirements.txt
    python setup.py install

Import Geonames data into Mongo and prepare the training data.

    cd $WORKSPACE/grits-api
    ./import_geonames.sh
    python train.py
    
Start a celery worker.

    celery worker -A tasks -Q priority --loglevel=INFO --concurrency=2

Start the server.

    python server.py

# Testing

To run the tests:

    git clone -b fetch_4-18-2014 git@github.com:ecohealthalliance/corpora.git
    cd test
    python -m unittest discover

Many tests are based on the comments in this document:
https://docs.google.com/document/d/12N6hIDiX6pvIBfr78BAK_btFxqHepxbrPDEWTCOwqXk/edit


# Classifier Data

## Using existing classifier data

Three files are required to operate the classifier: training.p, validation.p, ontologies.p
These will be downloaded from our S3 bucket by default, however that bucket might not
be available to you, or it might no longer exist. In that case, ontologies.p can be
generated by running the mine_ontologies.py but the training and validation pickles
are generated from HealthMap metadata which we do not have permission to distribute.
The corpora directory includes code for iterating over HealthMap data stored
in a girder database, scraping and cleaning the content of the linked source articles,
and generating pickles from it.

# Training the classifier

New data may be generated using the `train.py` script:

    $ python train.py

This script relies on having the [corpora](https://github.com/ecohealthalliance/corpora) data available.
