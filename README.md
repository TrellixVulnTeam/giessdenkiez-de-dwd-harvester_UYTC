![](https://img.shields.io/badge/Built%20with%20%E2%9D%A4%EF%B8%8F-at%20Technologiestiftung%20Berlin-blue)

# giessdenkiez-de-dwd-harvester

- Gather precipitation data from DWD's radolan data set, for the region of Berlin and connect to the giessdenkiez.de postgres DB (AWS RDS)
- Uploads trees combined with weather data to Mapbox and uses its API to create vector tiles for use on mobile devices
- Generates CSV and GeoJSON files that contain trees locations and weather data (grid) and uploads them to a AWS S3

## Pre-Install

I am using venv to setup a virtual python environment for separating dependencies:

```
python -m venv REPO_DIRECTORY
```

## Install

```
pip install -r requirements.txt
```

I had some trouble installing psycopg2 on MacOS, there is a problem with the ssl-lib linking. Following install resolved the issue:

```
env LDFLAGS='-L/usr/local/lib -L/usr/local/opt/openssl/lib -L/usr/local/opt/readline/lib' pip install psycopg2
```

### GDAL

As some of python's gdal bindings are not as good as the command line tool, i had to use the original. Therefore, `gdal` needs to be installed. GDAL is a dependency in requirements.txt, but sometimes this does not work. Then GDAL needs to be installed manually. Afterwards, make sure the command line calls for `gdalwarp` and `gdal_polygonize.py` are working.

#### Linux

Here is a good explanation on how to install gdal on linux: https://mothergeo-py.readthedocs.io/en/latest/development/how-to/gdal-ubuntu-pkg.html

#### Mac

For mac we can use `brew install gdal`.

The current python binding of gdal is fixed to GDAL==2.4.2. If you get another gdal (`ogrinfo --version`), make sure to upgrade to your version: `pip install GDAL==VERSION_FROM_PREVIOUS_COMMAND`

### Configuration

Copy the `sample.env` file and rename to `.env` then update the parameters, most importantly the database connection parameters.

## Running

`harvester/prepare.py` shows how the assets/buffer.shp was created. If a bigger buffer is needed change `line 10` accordingly and re-run.

`harvester/harvester.py` is the actual file for harvesting the data. Simply run, no command line parameters, all settings are in `.env`.

The code in `harvester/harvester.py` tries to clean up after running the code. But, when running this in a container, as the script is completely stand alone, its probably best to just destroy the whole thing and start from scratch next time.

## Docker

To have a local database for testing you need Docker and docker-compose installed. You will also have to create a public S3 Bucket. You also need to update the `.env` file with the values from `sample.env` below the line `# for your docker environment`.

to start only the database run

```bash
docker-compose -f  docker-compose.postgres.yml up
```

This will setup a postgres/postgis DB and provision the needed tables and insert some test data.

To run the harvester and the postgres db run

```bash
docker-compose up
```

### Known Problems

#### harvester.py throws Error on first run

When running the setup for the first time `docker-compose up` the provisioning of the database is slower then the execution of the harvester container. You will have to stop the setup and run it again to get the desired results.

#### Postgres Provisioning

The provisioning `sql` script is only run once when the container is created. When you create changes you will have to run:

```bash
docker-compose down
docker-compose up --build

```

## Terraform

Terrafrom is used to create the needed S3 Bucket, the Postres RDS and the Fargate container service. [Install and configure Terraform](https://learn.hashicorp.com/terraform?track=getting-started#getting-started). Update `terraform.tfvars` with your profile and region.

Run:

```bash
# once
# cd into the directories
# create them in this order
# 1. s3-bucket
# 2. rds
# 3. ecs-harvester
# the last setup needs some variables from you
# - vpc
# - public subnet ids
# - profile
# - and all the env variables for the container
terraform init
# and after changes
terraform apply
```
