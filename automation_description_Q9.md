# Automation of data loading process for Buienradar measurements and stations data

## Current state of the data loading process

Currently, the `./load_buienradar_data` script does the following:

1. Loads whatever most recent data is available from the Buienradar API in json format
2. Does some basic data transformation to convert the json data into a tabular format ready for the database
3. Loads the transformed data into a local SQLite database, using an upsert strategy to avoid duplicates

Since I ran `chmod +x load_buienradar_data` on the script, it can be ran directly
from the command line using `./load_buienradar_data`, and it will load the most
recent data available at the moment of running. It relies quite heavily on [uv](https://astral.sh/uv/) to manage
dependencies and virtual environments, which makes it very easy to run without having to worry about setting up
the environment first.

Since having data from just one snapshot is not very useful, automating this data load and running it regularly in order
to get the most recent data is a good idea, and historical data can also be built up that way for analysis
over time.

## Automating the data loading process

To automate the data loading process, the script (or rather a modified version of it) should be scheduled to run
regularly. On local machines this can be done e.g. using a cron job, but more likely it makes more sense to run this
somewhere in a cloud system, thus that is what I focus on here.

I am most familiar with GitHub Actions, for which another team in my company set up spot-priced runners in AWS for
very minimal costs, and the interface of GitHub Actions is very user friendly. Compute pricing of GitHub actions is
generally quite minimal, but other options like AWS Lambda or Azure Functions could also fit well depending on
requirements and existing infrastructure.

Already a few necessities for automating are implemented in the script, such as:

1. Idempotency:
    1. Database initialisation is ran if the database does not exist yet, but if it does exist, the script continues
    without error
    1. Upsert strategy is used to avoid duplicates, so if the script is ran multiple times, the same data will not
    be loaded multiple times, and only new data will be added. The created `measurementid` is an important part of
    this
1. Data load:
    1. The script already takes in whatever data is available at the moment of running, so as long as the script is
    ran often enough, all data will be loaded over time

This means the main job of automating the data loading process will be scheduling and triggering this script on a
regular basis. I have made some quick and dirty version of what a GitHub Actions workflow file could look like to
run this script every hour

## Other improvements

### Database choice

Sqlite is a great way to get a local database up and running in seconds/minutes, but accessing that data by multiple
users and not be limited to someone's local system, running a cloud database likely makes sense. A good option is
Postgres, as it is widely used and truly open source. Switching out the sqlite database in the script for a cloud
Postgres database is relatively straightforward, with secrets handling being the main thing to consider, and
initialising the database architecture outside of this script likely makes sense in that case, possiblky even with
IaC tools like Terraform.

### Data backfilling

The Buienradar API might make it possible to fetch historical data as well, meaning if an ingestion failed for
whatever reason and API already shows newer data, that the historical data can still be added to the database.
Adjusting the script to accept a timestamp or date parameter to be able to fetch data from a specific timestamp
makes this backfilling possible within the same script.

### Suppporting multiple scripts

It is likely that in a real world situation, there are many different data sources that data should be loaded from.
In that case, there will likely be quite a bit of overlap between the different scripts,
making it likely a good idea to split out the common code into a shared library of sorts.
That will make the code a lot more DRY (Do Not Repeat Yourself) and easier to maintain,
and optimizations immediately land everywhere.
This will make it necessary to have some good tests in place, to ensure that changes to the
shared code do not break any of the scripts that depend on it.
Of course a shared python library is one way of doing this, and is likely of great use here.

Another way to increase DRY-ness of code is to instrument different supporting functionalities
through something like GNU make. This allows installation of dependencies
(though this is light due to uv), running of tests, running of linters/formatters,
and running of the actual scripts as well, leaving e.g. GitHub Actions workflow files very clean and simple.

### Retry logic

Since it is an API we are fetching data from, it might not be available at all times, and it is good to have some
retry logic in place to make sure that temporary outages do not cause data loss. This can be implemented using a
simple retry mechanism with exponential backoff, which will try to fetch the data again after a certain amount of
time if the initial request fails. It should be taken into account though that every 20 minutes, the data is
updated and thus a retry after more than 20 minutes will not fetch the same data.

### Testing

I focused on having the basic functionality in here, as well as some reasoning for why certain choices were made.
I didn't have enough time to implement tests, but in a real world situation, it is very important to have good
test coverage, especially if there is shared code between different scripts. This will ensure that changes to the
code do not break any of the functionality, and that the data loading process continues to work as expected. It
even helps LLMs to make contributions as there is clear guardrails for what changes are allowed and what not, and
it is easier to understand the code with good tests in place as well.
