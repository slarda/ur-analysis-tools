# Universal Recommender Analysis Tool V3

The primary change to V3 of the tool is to support the Harness version of the Universal Recommender (UR).

These scripts do analysis of input data and run cross-validation tests using the UR. The analysis shows information about usage data, and the cross-validation tests use a technique called MAP@k, a type of precision test, to give a predictiveness metric to various combinations  of *indicators* used by the Correlated Cross-Occurrence (CCO) algorithm of the UR.

Harness has a more streamlined workflow and has a single server, that takes input and responds to queries. It also responds to a CLI so information similar to that produced by the PIO CLI can be found.

## The Test Process

There are 2 Phases to the process that this tool supports and one that must be done using Harness.

 1. **Split the Dataset** into training and test splits using this tool.
 - **Create and Train the Recommender** by importing data and training the recommender using the Harness CLI.
 - **Analyze** with this tool.

**Note:** This toolset does #1 and #3 but does not do #2. The specific steps to execute all workflow are:

 1. **Install and Start the Harness Server** This process is described [here](https://github.com/actionml/harness/blob/release/0.4.0-snapshot/install.md). Notice that we are using the 0.4.0-SNAPSHOT version of the server which includes Universal Recommender.
 2. **Verify the Harness Server is running** `harness status` will do this.
 3. **Get the Full Dataset** We assume that the data exists in a single large file in the local filesystem. The data location is specified in the `config.json` file. Todo: since it is often desirable to use an existing data split there should be a CLI option to skip splitting to use the paths in the config if they exist.
 4. **Split** the data as defined in the config.json into a "train" and "test" split, usually about 80% of the data will go to "train", the rest is used to "test".
 5. **Train a Model** from the "train" split of the data:
    1. **`harness add <engine-json-config>`** This will create the Universal Recommender instance configured to take input from the specific dataset use and use the algorithm parameters configured. The most important thing to note in the Engine's config is the `enginId` which is the instance ID for the test Engine.
    2. **`harness import <path-to-train-split> <engine-id>`** which will import data (perhaps slowly) into the Engine Instance defined by the Engine's config file.
    3. **`harness train <engine-id>`** This will create a model with the UR from with the `engineId` defined in the Engine's JSON config file from step #5.1 above.
    4. **`harness status <engine-id>`** After `harness train` the model is calculated in Spark and once the trining Job is completed is immediately deployed. Check the Engine Instance status to see if the Job is running or the job status is empty, meaning it has completed. Check server logs to make sure there were no errors in training. If there are, then the analysis phase will be meaningless.
 6. **Analyze** : Run several forms of MAP@k predictiveness tests on the splits, collect stats, and create a report for Excel.

Since Splitting can take a significant amount of time, it is advised to run it once, then train and deploy once, and run #the analysis with different configs as needed. There is no need to retrain in most cases. This will create a completely reproducible split so you can safely compare results from any run of analysis.

## A Note About Changing Parameters

If any change is made to the Engine's JSON config file, the data may need to be re-imported (if different indicator names are used) or may only need to be re-trained (if algorithm params are changed). Using the identical dataset splits will allow analysis results to be compared across runs.

# Setup

 1. **[Install Harness](https://github.com/actionml/harness/blob/release/0.4.0-snapshot/install.md)**: This will result in the correct versions of Python and Spark being installed.

 2. **Check Python**

 	`python3 --v`

 	if the version is less than 3.7 upgrade to the most recent stable version of python using systems package management tools like `apt-get` for Ubuntu linux or `brew` for the Mac.

 2. Install Python libraries using the Python package manager found [here](https://pip.pypa.io/en/stable/installing/)

 	```
 	sudo pip3 install numpy scipy pandas ml_metrics predictionio tqdm click openpyxl
 	```

 3. Setup Spark and Pyspark paths in `~/.bashrc` (linux) or `~/.bash_profile` or `~/.profile` macOS.

 	```
 	export SPARK_HOME=/path/to/spark
 	# this may be /usr/local/spark or wherever the root directory of the Spark installation is located
	export PYTHONPATH=$SPARK_HOME/python/:$SPARK_HOME/python/build/:$PYTONPAHTH
	```

# Run the Analysis Script

The script uses two configuration files:

- **`<some-engine>.json`** the configuration of UR instance being used for the analysis. This file is used to determine the indicator list and primary indicator. This code often refers to the "indicator" as an "event". The `algorithm.indicators` part of the JSON will contain the event names used in analysis. 
- **`config.json`** contains all other configuration including the path to the above `<some-engine>.json`

## Configuration options

This tool can be used without HDFS for file storage by using the local file system for the input dataset and the output splits. **Note**: replace `hdfs` below with `file` below for `source_file`, `train_file`, and `test_file`. The config.json has the following structure:

```
{
  "engine_config": "/path/to/the/engine.json",

  "splitting": {
    "version": "1",
    "source_file": "hdfs:...<PUT SOME PATH>...",
    "train_file": "hdfs:...<PUT SOME PATH>...train",
    "test_file": "hdfs:...<PUT SOME PATH>...test",
    "type": "date",
    "train_ratio": 0.8,
    "random_seed": 29750,
    "split_event": "<SOME NAME>"
  },

  "reporting": {
    "file": "./report.xlsx"
  },

  "testing": {
    "map_k": 10,
    "non_zero_users_file": "./non_zero_users.dat",
    "consider_non_zero_scores_only": true,
    "custom_combos": {
      "event_groups": [["ev2", "ev3"], ["ev6", "ev8", "ev9"]]
    }
  },

  "spark": {
    "master": "spark://<some-url>:7077"
  }
}
```

- __engine_config__ - file to be used as the Engine's config JSON file (see configuration of UR)
- __splitting__ - this section is about splitting data into train and test sets
	- __version__ - version to append to train and test file names (may be helpful if different test with different split configurations are run)
	- __source_file__ - file with data to be split
	- __train_file__ - file with training data to be produced (note that version will be append to file name)
	- __test_file__ - file with test data to be produced (note that version will be append to file name)
	- __type__ - split type (can be time in this case eventTime will be used to make split or random)
	- __train_ratio__ - float in (0..1), share of training samples
	- __random_seed__ - seed for random split
	- __split_event__ - in case of __type__ = "date", this is event to use to look for split date, all events with this name are ordered by eventTime and time which devides all such events into first __train_ratio__ and last (1 - __train_ratio__) sets is used to split all the rest data (events with all names) into training set and test set
- __reporting__ - reporting settings
	- __file__ - excel file to write report to
  - __csv_dir__ - directory name for csv reporting
  - __use_uuid__ - append to csv file names unique uuid associated with script run (can be useful to manage different results and reports)
- __testing__ - this section is about different tests and how to perform them
	- __map_k__ - maximum map @ k to be reported
	- __non_zero_users_file__ - file to save users with scores != 0 after first run of test set with primary event, this set may be much smaller then initial set of users so saving it and reusing can save much time
	- __consider_non_zero_scores_only__ (default: true) whether take into account only users with non-zero scores (i.e. users for which recommendations exist)
	- __custom_combos__
		- __event_groups__ - groups of events to be additionally tested if necessary
- __spark__ - Apache Spark configuration
	- __master__ - for now only master URL is configurable

## Split the Data

Use this command to run split of data into "train" and "test" sets

```
SPARK_HOME=/usr/local/spark PYTHONPATH=/usr/local/spark/python:/usr/local/spark/python/lib/py4j-0.9-src.zip ./map_test.py split
```

Additional options are available:

- `--csv_report` - put report to csv file not excel
- `--intersections` - calculate train / test event intersection data (**Advanced**)

## Train a Model

The above command will create a test and training split in the location specified in config.json. Now you must import, setup engine.json, train and deploy the "train" model so the rest of the MAP@k tests will be able to query the model.

## Test and Analysis

To run tests
```
SPARK_HOME=/usr/local/spark PYTHONPATH=/usr/local/spark/python:/usr/local/spark/python/lib/py4j-0.9-src.zip ./map_test.py test --all
```

Additional options are available and may be used to run not all test:

- `--csv_report` -
- `--dummy_test` - run dummy test
- `--separate_test` - run test for each separate event
- `--all_but_test` - run test with all events and tests with all but each every event
- `--primary_pairs_test` - run tests with all pairs of events with primary event
- `--custom_combos_test` - run custom combo tests as configured in config.json
- `--non_zero_users_from_file` - use list of users from file prepared on previous script run to save time

## Generated Report



### Old ipython analysis script
This is not recommended old approach to run ipython notebook.
```
IPYTHON_OPTS="notebook" /usr/local/spark/bin/pyspark --master spark://spark-url:7077
```
