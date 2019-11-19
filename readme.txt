
# start OpsCenter
docker run -e DS_LICENSE=accept -d -p 8888:8888 --name my-opscenter datastax/dse-opscenter

# start DSE with Search
docker run -e DS_LICENSE=accept --link my-opscenter:opscenter --name my-dse -d datastax/dse-server -s
docker run -e DS_LICENSE=accept --name my-dse -d datastax/dse-server -s

# Open OPS Center at
http://localhost:8888

# get DSE server IP
docker inspect my-dse | grep IPAddress

# start Studio
docker run -e DS_LICENSE=accept --link my-dse --name my-studio -p 9091:9091 -d datastax/dse-studio

# Open Studio at
http://localhost.digicert.com:9091

# connect to CQL shell
winpty docker exec -it my-dse cqlsh

# check server status
winpty docker exec -it my-dse nodetool nodesyncservice status

# restart all dockers
docker-compose restart

# start DSE server only - without OPS Center & Studio
docker-compose up -d dse

#==============================================================================

# log into DSE server OS shell
winpty docker exec -it datastaxsearch_dse_1 bash

# check DSE status
cd bin
./dsetool status

# node tool
./nodetool help
./nodetool info
./nodetool status
./nodetool describecluster
./nodetool getlogginglevels
./nodetool setlogginglevel org.apache.cassandra TRACE
./nodetool settraceprobability 0.1
./nodetool flush
./nodetool drain
./nodetool stopdaemon
./nodetool gossipinfo
./nodetool getendpoints killrvideo videos_by_tag 'cassandra'

# stress test
./cassandra-stress write n=50000 no-warmup -rate threads=1

# start CQL shell
./cqlsh

#==============================================================================

# config ring
cd /opt/dse/resources/cassandra/conf/
cat cassandra.yaml | grep initial_token

#==============================================================================

winpty docker exec -it 4895f78475e7 bash

dsetool get_core_config mariadb.users
dsetool get_core_config mariadb.users current=true

dsetool core_indexing_status mariadb.users
dsetool core_indexing_status mariadb.users --all

winpty docker exec -it 4895f78475e7 dsetool core_indexing_status mariadb.users --all

#==============================================================================

docker inspect 06918e471a05 | grep IPAddress
http://localhost:8983/solr

q	# query in 'field:value' format
firstname:James OR lastname:Gruber
fl	# field names list in result list
lastname

#==============================================================================

docker ps

winpty docker exec -it 4895f78475e7 cqlsh

CREATE KEYSPACE mariadb WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor' : 1};
use mariadb;
create table users (
	id int,
	acct_id int,
	firstname varchar,
	lastname varchar,
	email varchar,
	username varchar,
	job_title varchar,
	PRIMARY KEY (id));
select * from users;

CREATE SEARCH INDEX ON mariadb.users;
SELECT count(*) FROM mariadb.users WHERE solr_query = '*:*';

winpty docker exec -it 4895f78475e7 dsetool core_indexing_status mariadb.users --all

desc active search index schema on mariadb.users;

ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD lowerCaseString text;
ALTER SEARCH INDEX SCHEMA ON mariadb.users SET fields.field[@name='text'] @stored='false';
ALTER SEARCH INDEX SCHEMA ON mariadb.users SET fields.field[@name='text'] @multiValued='true';
ALTER SEARCH INDEX SCHEMA ON mariadb.users SET fields.field[@name='text'] @type='SearchTextField';

ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD types.fieldType[@class='org.apache.solr.schema.TextField', @name='TextField'];
ALTER SEARCH INDEX SCHEMA ON mariadb.users SET fields.field[@name='text'] @type='TextField';

ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD copyField[@source='firstname', @dest='text'];
ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD copyField[@source='lastname', @dest='text'];
ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD copyField[@source='job_title', @dest='text'];
ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD copyField[@source='username', @dest='text'];
ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD copyField[@source='email', @dest='text'];
ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD copyField[@source='id', @dest='text'];
ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD copyField[@source='acct_id', @dest='text'];

RELOAD SEARCH INDEX ON mariadb.users;
REBUILD SEARCH INDEX ON mariadb.users;

#==============================================================================

ALTER SEARCH INDEX SCHEMA ON mariadb.users ADD types.fieldType [ 
	@name='SearchTextField',
	@class='org.apache.solr.schema.TextField'
] WITH  $${
	"analyzer": [{
		"type": "index",
		"tokenizer": {"class": "solr.StandardTokenizerFactory"},
		"filter": [{
			"class": "solr.LowerCaseFilterFactory"}, {
			"class": "solr.ASCIIFoldingFilterFactory"}
		]}, {
		"type": "search",
		"tokenizer": {"class": "solr.StandardTokenizerFactory"},
		"filter": [{
			"class": "solr.LowerCaseFilterFactory"}, {
			"class": "solr.ASCIIFoldingFilterFactory"}
		]}
	]
}$$;

//ALTER SEARCH INDEX SCHEMA ON mariadb.users SET field[@name='job_title'] @type = 'SearchTextField';
ALTER SEARCH INDEX SCHEMA ON mariadb.users SET fields.field[@name='text'] @type='SearchTextField';

// cross fields, case insensitive, tokenized keyword match

