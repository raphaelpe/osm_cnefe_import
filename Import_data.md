###Preparing the machine and importing data

Scripts to prepare CNEFE data, on street names and addresses in Brasil, to be imported/integrated into Opeen Street Map

Commands bellow are to be run from a Linux Terminal

###Preparing the machine 
Install Postgresql and PostGIS flowing this totorial: http://switch2osm.org/loading-osm-data/

```
mkdir ~/osm_cnefe_import
```
###Creating Database
```
sudo -u postgres createuser -s $USER
createdb osm_cnefe_import
psql -d osm_cnefe_import -c 'CREATE EXTENSION hstore; CREATE EXTENSION postgis;'
```



###Importing CNEFE data:
Still incomplete. Just some references bellow

Downloading the data:
```
mkdir ~/osm_cnefe_import/CNEFE
cd ~/osm_cnefe_import/CNEFE
wget -r -nd  --read-timeout=5 ftp://ftp.ibge.gov.br/Censos/Censo_Demografico_2010/Cadastro_Nacional_de_Enderecos_Fins_Estatisticos/
unzip  '*.zip'
```
obs: downloading will take some time, there are 10904 files  totaling 926Mb 

Now we can follow two strategies:

#### Importing with pgloader 

haven't tryed yet, could not find a tutorial including pgloader in a multiple files to one table setting. 

#### Importing with psql \copy command

This is based on [this sugestion](http://www.postgresonline.com/journal/archives/157-Import-fixed-width-data-into-PostgreSQL-with-just-psql.html) to use only psql \copy command, import as a text blob to a staging table in Postgres and parse the content of each colum in Postgres.  To deal with the several files to be imported I'm using the suggestion in [this StackOverflow quetion](http://stackoverflow.com/questions/12646305/efficient-way-to-import-a-lot-of-csv-files-into-postgresql-db), of creating a creating an import file with one copy command perCNEFE file. 

This may also be usefull (depois tirar): http://postgresql.nabble.com/Multiple-COPY-statements-td5701101.html


Creating an import file with one copy command perCNEFE file. 

```
(for FILE in ./*.TXT; do echo "\COPY cnefe_staging FROM '$FILE'"; done) > temp_CNEFE_import-commands.sql
```

Correcting Encoding:
when importing the oringinal unzipped files into Postgres I encountered encoding problems ("psql:temp_CNEFE_import-commands.sql:1: ERROR:  invalid byte sequence for encoding "UTF8": 0xe9 0x63 0x69
CONTEXT:  COPY cnefe_staging, line 452"  similar for command 6, line 6538). I solved this with the 'recode' package:
```
sudo apt-get install recode
recode iso-8859-1..utf8 *.TXT
```

Importing to a table in Postgres
```
psql -d osm_cnefe_import  -c 'DROP TABLE cnefe_staging';
psql -d osm_cnefe_import  -c 'CREATE TABLE cnefe_staging (data text)'
psql -d osm_cnefe_import  -f temp_CNEFE_import-commands.sql
```
about 70 milion rows (it should have 84 milion)


Even with the above "recode" I'm still getting the the following encode errors:
"
psql:temp_CNEFE_import-commands.sql:8346: ERROR:  invalid byte sequence for encoding "UTF8": 0xc2 0x20
CONTEXT:  COPY cnefe_staging, line 49070: "42 4202 5 0 1781TRAVESSA                                          TAILANDIA                         ..."
psql:temp_CNEFE_import-commands.sql:9189: ERROR:  invalid byte sequence for encoding "UTF8": 0x94
CONTEXT:  COPY cnefe_staging, line 1023: "43 9100 5 0   21AVENIDA                                           BORGES DE MEDEIROS                ..."
psql:temp_CNEFE_import-commands.sql:10794: ERROR:  invalid byte sequence for encoding "UTF8": 0x00
CONTEXT:  COPY cnefe_staging, line 10418: "5215702 5 0  391RUA                                               TREZE                             ..."
"


Separating fields
```
psql -d osm_cnefe_import  -f create_cnefe_unnormalized.sql
```
osb: see the create_cnefe_unnormalized.sql in this gihub repo. 
Table is created but I get an error:
"CREATE TABLE
psql:create_cnefe_unnormalized.sql:84: ERROR:  invalid input syntax for integer: "     

nada é adicionado à tabela

next tings to do: 

create the CNEFE table, separating the fields from the stagging table

add indexes

normalize the CNEFE table into the tables: roads, enumeration districts, addresses, localidades

add the geoms of the enumeration districts



###Importing polygons of Setores Censitarios (enumeration districts):
Downloading the data:
```
mkdir ~/osm_cnefe_import/set_censitario
cd ~/osm_cnefe_import/set_censitario
wget -r -nd ftp://geoftp.ibge.gov.br/malhas_digitais/censo_2010/setores_censitarios/
unzip  '*.zip'
cp -p $(find . -name *SEE250GC_SIR.*) . 
rm -r */
```

Importing into Postgresql:
```
shp2pgsql -c -s 4674:4326 -I -W LATIN1 2010  public.shape_setor_censitario | psql -d osm_cnefe_import
```
obs: above, the switch -s changes spatial reference from SIRGAS2000(SRID=4674)  to WGS84 (SRID=4326)


###Importing OSM Brasil data:
Commands to be run from linux terminal. Make sure you have osm2pgsql installed
Inspiration: http://switch2osm.org/loading-osm-data/



Downloading the data:
I use the [Brasil OSM file produced by Geofabrik](http://download.geofabrik.de/south-america/brazil.html)
```
wget http://download.geofabrik.de/south-america/brazil-latest.osm.pbf.md5
wget http://download.geofabrik.de/south-america/brazil-latest.osm.pbf
md5sum -c brazil-latest.osm.pbf.md5
```
Importing into PostgreSQL
```
osm2pgsql --create --slim -l \
    --cache 4000 --hstore \
    --style openstreetmap-carto.style --multi-geometry \
    --host MYMACHINE--port 1234 --database MYDATABASE \
    brazil-latest.osm.pbf
```
above replace the host, port and database info with the values for your setup. 





### (optional) Importing OSM Brasil History File:


Following this tutorial:  https://github.com/MaZderMind/osm-history-renderer/blob/master/TUTORIAL.md

Importing the data:
```
mkdir ~/osm_cnefe_import/OSM_Brasil/Brasil_history_file
cd ~/osm_cnefe_import/OSM_Brasil/Brasil_history_file
wget http://osm.personalwerk.de/full-history-extracts/latest/south-america/brazil.osh.pbf
```

Importing into postgres
```
...
```

##Data sources summary

* CNEFE data: ftp://ftp.ibge.gov.br/Censos/Censo_Demografico_2010/Cadastro_Nacional_de_Enderecos_Fins_Estatisticos

* Setor sensitario: ftp://geoftp.ibge.gov.br/malhas_digitais/censo_2010/setores_censitarios/

* [Brasil OSM file produced by Geofabrik](http://download.geofabrik.de/south-america/brazil.html):

  * http://download.geofabrik.de/south-america/brazil-latest.osm.pbf.md5
  * http://download.geofabrik.de/south-america/brazil-latest.osm.pbf
  * http://osm.personalwerk.de/full-history-extracts/latest/south-america/brazil.osh.pbf

