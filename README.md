##  Overview
1. Install latest version of CKAN from source. Directions below adapted from the [CKAN installation docs](http://docs.ckan.org/en/ckan-2.0/install-from-source.html).
2. Install Solr v4.6.0 
3. Install Tomcat v7.0.42
4. Install GeoServer v2.4.0.
5. Build a PostGIS template database.
6. Install CKAN extensions:
7. Install PyCSW.

## Detailed Instructions

### Linux Environment
The CKAN set up was done on Ubuntu 14.04 T2.Medium instance on AWS cloud.

### Base Updates

    $ sudo apt-get update
    $ sudo apt-get upgrade

### CKAN Prerequisites

    $ sudo apt-get install python-dev postgresql-9.3-postgis-2.1 libpq-dev python-pip python-virtualenv git-core openjdk-7-jdk

### Install Solr

    $ wget http://archive.apache.org/dist/lucene/solr/4.6.0/solr-4.6.0.tgz
    $ tar zxvf solr-4.6.0.tgz
    $ sudo mv solr-4.6.0 /var/lib/solr-4.6.0
    $ rm solr-4.6.0.tgz
    
Start Solr

    $ cd /var/lib/solr-4.6.0/example
    $ java -jar start.jar
    
Verify that __Solr__ is running at __http://localhost:8983/solr__.

Stop __Solr__ for now.

    Ctrl-C
    $ cd ~

### Install Tomcat

Tomcat should be installed with apt-get so it automatically runs as a service.

    $ sudo apt-get install tomcat7 unzip

Verify that __Tomcat__ is running at __http://localhost:8080/__.
    
### Install GeoServer
    $ wget http://sourceforge.net/projects/geoserver/files/GeoServer/2.4.0/geoserver-2.4.0-war.zip
    $ unzip geoserver-2.4.0-war.zip
    $ sudo cp geoserver.war /var/lib/tomcat7/webapps/
    $ rm geoserver-2.4.0-war.zip
    $ rm geoserver.war

Verify that __GeoServer__ is running at __http://localhost:8080/geoserver/__. 

 The __GeoServer__ default configuration uses __admin:geoserver__ as username:password.

### Build a PostGIS Template DB

    $ sudo su postgres
    postgres $ createdb -E utf8 template_postgis
    postgres $ psql -d template_postgis -f /usr/share/postgresql/9.3/contrib/postgis-2.1/postgis.sql
    postgres $ psql -d template_postgis -f /usr/share/postgresql/9.3/contrib/postgis-2.1/spatial_ref_sys.sql
    postgres $ psql -d template_postgis -c "GRANT ALL on geometry_columns to PUBLIC;"
    postgres $ psql -d template_postgis -c "GRANT ALL on geography_columns to PUBLIC;"
    postgres $ psql -d template_postgis -c "GRANT ALL on spatial_ref_sys to PUBLIC;"
    postgres $ psql -d postgres -c "UPDATE pg_database SET datistemplate='true' WHERE datname='template_postgis';"
    postgres $ exit

This template database should be used to create any new __PostGIS__-enabled database.

### Install Some Geo-Spatial Dependencies

    $ sudo apt-get install rabbitmq-server gdal-bin build-essential libxml2-dev libxslt-dev

- __RabbitMQ__ is used to manage the queue associated with the `ckanext-harvest` extension.
- __GDAL__ is a geospatial data manipulation library we need for geoserver interactions. 
- __build-essential__ installs a number of packages that are used to compile source code.

Set up an updated version of __libxml2__. We need this for schema-validation for some of the standard ISO metadata functions that are implemented both in `ckanext-spatial` and in the csw module of our `ckanext-ngds`.

    $ cd ~
    $ wget ftp://xmlsoft.org/libxml2/libxml2-2.9.0.tar.gz
    $ tar zxvf libxml2-2.9.0.tar.gz
    $ cd libxml2-2.9.0/
    $ ./configure --libdir=/usr/lib/x86_64-linux-gnu
    $ make
    $ sudo make install
    $ cd ..
    $ rm libxml2-2.9.0.tar.gz

### Setup CKAN in a Virtual Environment

Create a virutal environment and activate it. _Now Python packages installed will be part of this virtual environment, and will not modify the system's Python installation._

    $ virtualenv --no-site-packages ckanenv
    $ cd ~/ckanenv
    $ source bin/activate

Install __CKAN__ and its Python dependencies.

    (ckanenv) $ pip install -e git+https://github.com/okfn/ckan.git@ckan#egg=ckan
    (ckanenv) $ pip install -r src/ckan/pip-requirements.txt
    (ckanenv) $ pip install -r src/ckan/pip-requirements-test.txt

### Create Databases

Create users for production and testing setups for __CKAN__ with the __Datastore__ enabled. _The little `d` in the last line below is not a typo._

    $ sudo su postgres
    postgres $ createuser -S -D -R -P ckan_user
    postgres $ createuser -s -P ckan_tester
    postgres $ createuser -S -D -R -P datastore_reader
    postgres $ createuser -S -D -R -P datastore_writer
    postgres $ createuser -S -D -R -P datastore_test_reader
    postgres $ createuser -S -d -R -P datastore_test_writer

Create the databases for our production and test environments.

    postgres $ createdb -E utf8 -O ckan_user -T template_postgis ckan_main
    postgres $ createdb -E utf8 -O ckan_tester -T template_postgis ckan_test
    postgres $ createdb -E utf8 -O datastore_writer -T template_postgis datastore
    postgres $ createdb -E utf8 -O datastore_test_writer -T template_postgis datastore_test
    postgres $ exit

### Install CKAN Extensions

    (ckanenv) $ pip install -e git+https://github.com/okfn/ckanext-harvest.git@stable#egg=ckanext-harvest
    (ckanenv) $ pip install -e git+https://github.com/okfn/ckanext-spatial.git@stable#egg=ckanext-spatial
    (ckanenv) $ pip install -e git+https://github.com/okfn/ckanext-datastorer.git#egg=ckanext-datastorer
    (ckanenv) $ pip install -e git+https://github.com/okfn/ckanext-importlib#egg=ckanext-importlib
    (ckanenv) $ pip install -r src/ckanext-harvest/pip-requirements.txt
    (ckanenv) $ pip install -r src/ckanext-spatial/pip-requirements.txt
    (ckanenv) $ pip install -r src/ckanext-datastorer/pip-requirements.txt
    (ckanenv) $ pip install -r src/ckanext-importlib/pip-requirements.txt

### Install PyCSW
Instructions adapted from [CSW Support](http://ckanext-spatial.readthedocs.org/en/latest/csw.html#setup).

    (ckanenv) $ pip install -e git+https://github.com/geopython/pycsw.git@1.8.0#egg=pycsw
    
Build a database for the PyCSW server:

    $ sudo su postgres
    postgres $ createdb -E utf8 -O ckan_user ckan_pycsw
    
Install PostGIS into PyCSW database

    postgres $ psql -d ckan_pycsw -f /usr/share/postgresql/9.3/contrib/postgis-2.1/postgis.sql
    postgres $ psql -d ckan_pycsw -f /usr/share/postgresql/9.3/contrib/postgis-2.1/spatial_ref_sys.sql
    postgres $ psql -d ckan_user -c "GRANT ALL on geometry_columns to PUBLIC;"
    postgres $ psql -d ckan_user -c "GRANT ALL on geography_columns to PUBLIC;"
    postgres $ psql -d ckan_user -c "GRANT ALL on spatial_ref_sys to PUBLIC;"
    postgres $ exit

Copy the configuration file and create a link to it from the CKAN directory:

    (ckanenv) $ cd ~/ckanenv/src/pycsw
    (ckanenv) $ cp default-sample.cfg default.cfg
    (ckanenv) $ ln -s ~/ckanenv/src/pycsw/default.cfg ~/ckanenv/src/ckan/pycsw.cfg

Change these variables in default.cfg:

    home=/ckanenv/src/pycsw
    url = http://localhost:8000/csw
    database = postgresql://ckan_user:ckan_user@localhost/ckan_pycsw 

Setup the pycsw table:
    
    (ckanenv) $ paster --plugin=ckanext-spatial ckan-pycsw setup -p ~/ckanenv/src/ckan/pycsw.cfg

### Setup CKAN's development.ini

    (ckanenv) $ cd ~/ckanenv/src/ckan
    (ckanenv) $ paster make-config ckan development.ini

Next, edit this new development.ini file. The entries to be edited already exist, you just need to comment/uncomment them or adjust their values. Note that the passwords established when you created the database users need to be substituted for _password_ below. Also, the last line for the storage_dir assumes the user is ngds.

    sqlalchemy.url = postgresql://ckan_user:password@localhost/ckan_main
    #sqlalchemy.url = sqlite://
    ckan.datastore.write_url = postgresql://datastore_writer:password@localhost/datastore
    ckan.datastore.read_url = postgresql://datastore_reader:password@localhost/datastore
    ckan.site_url = http://localhost:5000
    solr_url = http://localhost:8983/solr
    ckan.plugins = stats json_preview recline_preview datastore spatial_metadata spatial_query harvest spatial_harvest_metadata_api csw_harvester
    ofs.impl = pairtree
    ofs.storage_dir = /home/ngds/ckanenv/uploaded-files

_About that _site_url_:
If you don't have this configured to the name of the domain from which CKAN will be accessed, all kinds of things won't work. CKAN uses this setting any time it needs to construct an absolute URL. A good example is when you want to download a file -- if site_url is set or set incorrectly, you will not be able to download the file.


Make central.ini and node.ini.

    (ckanenv) $ cp development.ini central.ini
    (ckanenv) $ cp development.ini node.ini

_Do not delete development.ini_

Edit the plugins in central.ini:

    ckan.plugins = stats json_preview recline_preview datastore spatial_metadata spatial_query harvest spatial_harvest_metadata_api csw_harvester
    
Edit the plugins in node.ini:

    ckan.plugins = stats json_preview recline_preview datastore spatial_metadata spatial_query datastorer

_Note about plugins_: `datastorer` and `harvest` don't mix. You can't run __CKAN__ with both of them enabled. Our "central" node needs the harvesting things, while our "node" does not, but does need the datastorer.

Now you've pointed __CKAN__ at your databases, pointed it at __Solr__, and configured the plugins that you want to run.

Make sure the place exists where uploaded files will go to:

    $ mkdir ~/ckanenv/uploaded-files

### Setup a test.ini

    $ cd ~/ckanenv/src/ckan
    $ cp development.ini our-tests.ini

Then edit `our-tests.ini`...

    sqlalchemy.url = postgresql://ckan_tester:password@localhost/ckan_test
    ckan.datastore.write_url = postgresql://datastore_test_writer:password@localhost/datastore_test
    ckan.datastore.read_url = postgresql://datastore_test_reader:password@localhost/datastore_test

Don't overwrite `test.ini`. If you do, you won't be able to successfully run CKAN's core tests.

### Configure & Start Solr
Create a link to the CKAN schema.xml:

    $ sudo mv /var/lib/solr-4.6.0/example/solr/collection1/conf/schema.xml /var/lib/solr-4.6.0/example/solr/collection1/conf/schema.xml.bak
    $ cp ~/ckanenv/src/ckan/ckan/config/solr/schema-2.0.xml ~/ckanenv/src/ckan/schema.xml
    $ sudo ln -s ~/ckanenv/src/ckan/schema.xml /var/lib/solr-4.6.0/example/solr/collection1/conf/schema.xml

Start Solr:

    $ cd /var/lib/solr-4.6.0/example
    $ java -jar start.jar

### Configure Databases

    (ckanenv) $ paster --plugin=ckan db init -c ~/ckanenv/src/ckan/central.ini
    (ckanenv) $ paster --plugin=ckan datastore set-permissions postgres -c ~/ckanenv/src/ckan/central.ini
    (ckanenv) $ paster --plugin=ckan db init -c ~/ckanenv/src/ckan/node.ini
    (ckanenv) $ paster --plugin=ckan datastore set-permissions postgres -c ~/ckanenv/src/ckan/node.ini
    (ckanenv) $ paster --plugin=ckan db init -c ~/ckanenv/src/ckan/test.ini
    (ckanenv) $ paster --plugin=ckan datastore set-permissions postgres -c ~/ckanenv/src/ckan/test.ini

### Run CKAN

In a new terminal start Celery:  

    $ source ~/ckanenv/bin/activate
    (ckanenv) $ cd ~/ckanenv/src/ckan
    (ckanenv) $ paster celeryd
    
In a new terminal start development.ini:

    $ source ~/ckanenv/bin/activate
    (ckanenv) $ cd ~/ckanenv/src/ckan
    (ckanenv) $ paster serve development.ini

Verify that the site is being served at __http://localhost:5000/__. 
