# database-jmeter-load-testing

Premise:
Database throughput is an aspect of database selection where customer’s determine what amount of requests/second that their use case requires. From there they can estimate the price and size of the underlying resources of their available selection and plan accordingly. It is most easily determined if there is a benchmark of resources already available and also if there are published throughput information available. This workshop is meant to provide a sampling of throughput information and techniques as well as compare the different database throughputs. 

General set up:
A GCE VM runs JMeter, which is a popular load testing program. JMeter requires Java and Maven to run, the download and install steps are included below. Downloading the JDBC files for Cloud Spanner and PostgreSQL for Cloud SQL and AlloyDB is required. After the JMeter setup is completed within a GCE VM then the database should be created with the tables already created. Files named [RDMS name]-Initial-Load.jmx should be run to populate the database. Once the database is populated then the database can be load tested.  Files named [RDMS name]-PT.jmx should be run to load testing on the database. 

Export settings and config permissions: 
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
export REGION=us-central1
gcloud config set compute/region $REGION

git config --global user.email "you@example.com"  

git config --global user.name "Your Name"

Create a VM:

gcloud compute instances create test-jmeter \ --zone=us-central1-a \ --machine-type=n1-standard-1 \ --subnet=default \ --no-address

SSH into the VM: 

#download Java 

wget https://download.oracle.com/java/22/latest/jdk-22_linux-x64_bin.deb

sudo dpkg -i jdk-22_linux-x64_bin.deb

#double check it works 

java -version
#download Maven 

wget https://downloads.apache.org/maven/maven-3/3.9.7/binaries/apache-maven-3.9.7-bin.tar.gz

tar -xvzf apache-maven-3.9.7-bin.tar.gz

#get the jar files 

mkdir Development 

cd Development

mkdir -p maven-jarfiles

cd 

find . -type f -name "*.jar" -exec mv {} ~/Development/maven-jarfiles/ \;


# move maven files to be generally accessible

sudo mv apache-maven-3.9.7 /opt/

#add the following lines to the end of the file & apply changes 

nano ~/.bashrc

export MAVEN_HOME=/opt/apache-maven-3.9.7
export PATH=$PATH:$MAVEN_HOME/bin

#verify installation 

mvn -version

#download JMeter
wget https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-5.6.3.tgz

tar -xvzf apache-jmeter-5.6.3.tgz

mv apache-jmeter-5.6.3 ~/Development/

#download the .jmx files and move them into jmeter file 

mv [.jmx files] ~/Development/apache-jmeter-5.6.3.tgz/bin










Spanner Instructions: 

# create Cloud Spanner 
gcloud spanner instances create test-instance \ --config=us-central1 \ --description="Testing Cloud Spanner" \ --processing-units=100 \ --enable-autoscaler=false

gcloud spanner databases create singers --instance=test-instance

# use gcloud or Spanner Studio to execute these queries (these are in 

CREATE TABLE Singers (
  SingerId   STRING(36) NOT NULL,
  FirstName  STRING(1024),
  LastName   STRING(1024),
  SingerInfo BYTES(MAX),
) PRIMARY KEY (SingerId);


CREATE TABLE Albums (
  SingerId     STRING(36) NOT NULL,
  AlbumId      STRING(36) NOT NULL,
  AlbumTitle   STRING(MAX),
) PRIMARY KEY (SingerId, AlbumId),
  INTERLEAVE IN PARENT Singers ON DELETE CASCADE;


CREATE TABLE Songs (
  SingerId     STRING(36) NOT NULL,
  AlbumId      STRING(36) NOT NULL,
  TrackId      STRING(36) NOT NULL,
  SongName     STRING(MAX),
) PRIMARY KEY (SingerId, AlbumId, TrackId),
  INTERLEAVE IN PARENT Albums ON DELETE CASCADE;

Going back to the vm:

cd Development 

mkdir Spanner

cd Spanner

mvn dependency:get -Dartifact=com.google.cloud:google-cloud-spanner-jdbc:RELEASE -Dmaven.repo.local=.

cd Development 

mkdir spanner-jarfiles 


# Find all .jar files in the Spanner folder and move them to spanner-jarfiles 

find Spanner -type f -name "*.jar" -exec mv {} spanner-jarfiles/ \;

Populate Spanner tables: 

cd apache-jmeter-5.6.3/bin

./jmeter -n -t Spanner-Initial-Load.jmx -l spanner-il.csv -Jusers=1000 -Jiterations=1000

#the output should return 200
vi spanner-il.csv

#in Spanner Studio or gcloud to check the the population was successful

select * from singers;

Execute Performance Test: 
cd Development/apache-jmeter-5.6.3/bin

./jmeter -n -t SpannerPerformanceTest.jmx -l spanner-pt.csv -Jusers=100 -Jduration=120

 Cloud SQL Instructions: 

#create the Cloud SQL instance and populate the database "singer" this a sample gcloud command

gcloud sql instances create [INSTANCE_NAME] \ --network=default \ 
--region=us-central1 \
--no-assign-ip \ 	--private-network=projects/[PROJECT_ID]/global/networks/default

#to connect to Cloud SQL from the VM
https://cloud.google.com/sql/docs/postgres/connect-instance-private-ip

CREATE DATABASE singers;

\connect singers;

CREATE TABLE Singers ( 
SingerId varchar(1024) NOT NULL, 
FirstName varchar(1024), 
LastName varchar(1024), 
SingerInfo bytea, 
PRIMARY KEY (SingerId) );


CREATE TABLE Albums ( 
SingerId VARCHAR(1024) NOT NULL, 
AlbumId VARCHAR(1024) NOT NULL, 
AlbumTitle VARCHAR(1024), 
PRIMARY KEY (SingerId, AlbumId), 
CONSTRAINT fk_singer FOREIGN KEY (SingerId) REFERENCES Singers (SingerId) ON DELETE CASCADE );


CREATE TABLE Songs (
    SingerId VARCHAR(1024) NOT NULL,
    AlbumId VARCHAR(1024) NOT NULL,
    TrackId VARCHAR(1024) NOT NULL,
    SongName VARCHAR(1024),
    PRIMARY KEY (SingerId, AlbumId, TrackId),
    CONSTRAINT fk_album
        FOREIGN KEY (SingerId, AlbumId)
        REFERENCES Albums (SingerId, AlbumId)
        ON DELETE CASCADE
);



#in another window

cd Development/apache-jmeter-5.6.3/bin

#If the database isnt populated then populate with dummy data 

./jmeter -n -t CloudSql-Initial-Load.jmx -l load-out.csv -Jusers=1000 -Jiterations=1000

# Run preformance test 
./jmeter -n -t CloudSQLPT.jmx -l test-out.csv -Jusers=100 -Jduration=900












AlloyDB Instruction: 

https://cloud.google.com/alloydb/docs/connect-psql
sudo apt-get update
sudo apt-get install postgresql-client

#connect to AlloyDB 
psql -h [alloydb private ip] -p 5432 -U postgres

create database singers;

\connect singers;

#create 

CREATE TABLE Singers ( 
SingerId varchar(1024) NOT NULL, 
FirstName varchar(1024), 
LastName varchar(1024), 
SingerInfo bytea, 
PRIMARY KEY (SingerId) );


CREATE TABLE Albums ( 
SingerId VARCHAR(1024) NOT NULL, 
AlbumId VARCHAR(1024) NOT NULL, 
AlbumTitle VARCHAR(1024), 
PRIMARY KEY (SingerId, AlbumId), 
CONSTRAINT fk_singer FOREIGN KEY (SingerId) REFERENCES Singers (SingerId) ON DELETE CASCADE );


CREATE TABLE Songs (
    SingerId VARCHAR(1024) NOT NULL,
    AlbumId VARCHAR(1024) NOT NULL,
    TrackId VARCHAR(1024) NOT NULL,
    SongName VARCHAR(1024),
    PRIMARY KEY (SingerId, AlbumId, TrackId),
    CONSTRAINT fk_album
        FOREIGN KEY (SingerId, AlbumId)
        REFERENCES Albums (SingerId, AlbumId)
        ON DELETE CASCADE
);


#In the execution VM

cd Development/apache-jmeter-5.6.3/bin

./jmeter -n -t AlloyDB-Initial-Load.jmx -l alloy-test.csv -Jusers=100 -Jduration=90

To view AlloyDB population: 

# https://cloud.google.com/alloydb/docs/connect-psql 
# from the vm connect via psql client and then enter password

psql -h [IP_ADDRESS} -U [USERNAME]

SELECT * FROM singers;

SELECT * FROM albums;

SELECT * FROM songs;

#these tables should be populated 



Trigger load test: 

cd Development/apache-jmeter-5.6.3/bin

./jmeter -n -t AlloyDB-PT.jmx -l alloy-pt.csv -Jusers=1000 -Jduration=120




