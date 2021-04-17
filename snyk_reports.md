# Snyk dependency report with Python
## Problem
Provide software inventory report, based on dependencies in Snyk. Run it only on demand.

## Snyk token
Create Snyk service account and generate [token](https://support.snyk.io/hc/en-us/articles/360004037597-Service-accounts-overview) for API calls.

## Script
The script is written on Python and uses [Snyk API](https://snyk.docs.apiary.io/#) via module [pysnyk](https://github.com/snyk-labs/pysnyk).
For authorization we should set token as the environment variable `SNYK_TOKEN`.

Requirements file:
```sh
$ cat requirements.txt
pysnyk==0.6.1
```

Install requirements via `pip`:
```sh
$ pip install -r requirements.txt
```

Python script:
```python
"""
This script gets Snyk projects' dependencies and creates CSV report with
name pattern %d_%m_%Y.csv. For instance: 01_01_1970.csv

CSV file has 3 columns. For instance:
ID,NAME,VERSION
addressable@2.7.0,addressable,2.7.0
atomos@0.1.3,atomos,0.1.3
aws-eventstream@1.1.0,aws-eventstream,1.1.0
aws-partitions@1.363.0,aws-partitions,1.363.0
...

You should provide authorizatino token with envrionment variable SNYK_TOKEN.
"""
import csv
import logging
import snyk
import sys
import os

from datetime import date

LOG_FORMAT = "%(asctime)s - %(levelname)s  %(message)s"
CSV_COLUMNS = ["ID", "NAME", "VERSION"]
SNYK_TOKEN = os.getenv("SNYK_TOKEN")

logging.basicConfig(format=LOG_FORMAT,
                    level=logging.INFO,
                    stream=sys.stdout
                    )


def main(argv=None):
    client = snyk.SnykClient(SNYK_TOKEN)
    all_orgs = client.organizations.all()
    logging.info("Found %s snyk organizations" % len(all_orgs))
    csv_file = date.today().strftime("%d_%m_%Y") + ".csv"
    with open(csv_file, "w", newline="") as file:
        logging.info("Creating report file %s", csv_file)
        writer = csv.writer(file)
        writer.writerow(CSV_COLUMNS)
        for org in all_orgs:
            logging.info("Processing organization %s", org.name)
            all_projects = org.projects.all()
            for project in all_projects:
                all_dependencies = project.dependencies.all()
                for dependency in all_dependencies:
                    writer.writerow([dependency.id,
                                     dependency.name,
                                     dependency.version])

    file.close()
    logging.info("File %s has been completed", csv_file)

if __name__ == "__main__":
    main()

```

Run:
```sh
$ python get_libs.py

2021-04-16 11:57:38,861 - INFO  Found 2 snyk organizations
2021-04-16 11:57:38,861 - INFO  Creating report file 16_04_2021.csv
2021-04-16 11:57:38,861 - INFO  Processing organization Backend
2021-04-16 11:57:41,694 - INFO  Processing organization Frontend
2021-04-16 12:01:04,585 - INFO  File 16_04_2021.csv has been completed
```
