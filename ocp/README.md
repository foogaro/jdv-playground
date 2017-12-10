# JBoss Data Virtualization on OpenShift Container Platform 3.6

The goal of this example is how to use a custom database driver to connect to an external database, thorugh a Virtual Database (aka VDB).
For this example, we will use a SQL Server database (believe it or not, running on a Linux container), and the latest SQL Server JDBC driver.

## Prerequisites
**Before you being make sure you have the firewall and other security tools in a _permissive_ mode.**

Follows the list of the tools needed to follow the example:
* Internet connection
* Git client
* OpenJDK 1.8
* Docker
* CLI OC client

You can download and unpack the CLI from the Red Hat Customer Portal for use on Linux, MacOSX, and Windows clients. After logging in with your Red Hat account, you must have an active OpenShift Enterprise subscription to access the downloads page.
Link to Red Hat Customer Portal:

https://access.redhat.com/downloads/content/290

Link to the community version:

https://www.openshift.org/download.html

## Git clone the project
To clone the projectm thus download the repository, do assa foolows:
```bash
$ git clone https://github.com/foogaro/jdv-playground.git
```

Then navigate to the `ocp` folder, as it will be our working directory.

## Setting up the SQL Server database
Docker should be up and running, if it's not, start it as follows: 
```bash
$ service docker start
Redirecting to /bin/systemctl start  docker.service
```

Run SQL Server on a container as follows:
```bash
$ docker run --name="sqlserver-loves-linux" -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=datavirt.2017' -p 1433:1433 -d microsoft/mssql-server-linux:latest
Unable to find image 'microsoft/mssql-server-linux:latest' locally
Trying to pull repository registry.access.redhat.com/microsoft/mssql-server-linux ...
Trying to pull repository docker.io/microsoft/mssql-server-linux ...
latest: Pulling from docker.io/microsoft/mssql-server-linux
f6fa9a861b90: Downloading [======================>                            ] 20.91 MB/46.41 MB
da7318603015: Downloading [==================================================>]    851 B/851 B
6a8bd10c9278: Download complete
d5a40291440f: Download complete
bbdd8a83c0f1: Download complete
3a52205d40a6: Downloading [========================>                          ] 14.45 MB/28.98 MB
6192691706e8: Downloading [=====================>                             ] 16.51 MB/38.7 MB
1a658a9035fb: Waiting
97fa7291bda1: Waiting
b27ed30c4cf6: Waiting
```

Once done, you should have you database up and running, as follows:
```bash
$ docker ps -aq
CONTAINER ID        IMAGE                                                                                                                               COMMAND                  CREATED             STATUS                         PORTS                    NAMES
d0bc27cdbf85        microsoft/mssql-server-linux:latest                                                                                                 "/bin/sh -c /opt/mssq"   2 hours ago         Up 2 hours                     0.0.0.0:1433->1433/tcp   sqlserver-loves-linux
```

Once you have the database up and running you can login into it and create the database, the schema and the table, as follows:
### Create the database
```sql
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P datavirt.2017 -Q "CREATE DATABASE DATAVIRT2;"
```

### Create the table
```sql
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P datavirt.2017 -Q "USE DATAVIRT2;
CREATE TABLE dbo.ITEMS ( 
  ITEM_ID INT NOT NULL, 
  ITEM_CODE VARCHAR(20) NOT NULL,  
  ITEM_DESCRITION VARCHAR(255) ,
  DT_INSERT DATETIME NOT NULL,
  DT_UPDATE DATETIME NOT NULL,
  CONSTRAINT PK_ITEM_ID PRIMARY KEY (ITEM_ID)
);"
```

### Populate the table
```sql
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P datavirt.2017 -Q "USE DATAVIRT2;
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (1, '0000-0000-0000-0001', 'One', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (2, '0000-0000-0000-0002', 'Two', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (3, '0000-0000-0000-0003', 'Three', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (4, '0000-0000-0000-0004', 'Four', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (5, '0000-0000-0000-0005', 'Five', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (6, '0000-0000-0000-0006', 'Six', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (7, '0000-0000-0000-0007', 'Seven', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (8, '0000-0000-0000-0008', 'Eight', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (9, '0000-0000-0000-0009', 'Nine', GETDATE(), GETDATE());
INSERT INTO dbo.ITEMS (ITEM_ID, ITEM_CODE, ITEM_DESCRITION, DT_INSERT, DT_UPDATE) VALUES (10, '0000-0000-0000-0010', 'Ten', GETDATE(), GETDATE());"
```

## Running OCP
The OC client tool will use Docker to create all the nodes it needs, such as the docker registry, the HAProxy node, and so on.
It's a good practice to have the OpenShift configuration persisted, so we don't need to eventually re-import tablates, images and so on.
Here is the script that we will use to launch OCP:
```bash
oc cluster up \
--host-data-dir="/opt/rh/oc-cluster-up/data"  \
--host-pv-dir="/opt/rh/oc-cluster-up/pv"  \
--host-volumes-dir="/opt/rh/oc-cluster-up/vol" \
--logging=false \
--metrics=true \
--public-hostname="ocp.foogaro.com" \
--routing-suffix="apps.foogaro.com" \
--use-existing-config ocp-datavirt
```

Feel free to change the hostname and the routing suffix to whatever you want. Also, make sure to add the hostname in your `/etc/hosts` file.
If everything started properly ì, you should have an output similar to the following:
```bash
Starting OpenShift using registry.access.redhat.com/openshift3/ose:v3.6.173.0.5 ...
OpenShift server started.

The server is accessible via web console at:
    https://ocp.foogaro.com:8443

```

##Login into OCP
We can now start using the OpenShift Container Platform.
First of all we need to login as _system administrator_ and import the Docker _images_ we need, as follows: 
```bash
oc login -u system:admin https://127.0.0.1:8443
Logged into "https://127.0.0.1:8443" as "system:admin" using existing credentials.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    kube-public
    kube-system
    myproject
    openshift
    openshift-infra

Using project "default".
```
To access the paltform as _system administrator_ you don't need credentials, it use a certificate that has created during the installation.
Once, we are in we can start importing the templates as follows:
```bash
oc create -n openshift -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/jboss-image-streams.json
oc create -n openshift -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/datavirt/datavirt63-basic-s2i.json
oc create -n openshift -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/datavirt/datavirt63-extensions-support-s2i.json
oc create -n openshift -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/datavirt/datavirt63-secure-s2i.json
```

Now taht we have the templates loaded into the platform, we can use them to create our project.
First login as _admin_ as follows:
```bash
oc login -u admin -p admin
Login successful.

You don't have any projects. You can try to create a new project, by running

    oc new-project <projectname>

```

Now create a project
```bash
oc new-project jdv-playground --description="JDV Playground on OCP" --display-name="JDV Playground"
```

Now create the app, as follows:
```bash
oc new-app --template=datavirt63-extensions-support-s2i \
-p APPLICATION_NAME=datavirt-app \
-p CONFIGURATION_NAME=datavirt-app-config \
-p SOURCE_REPOSITORY_URL=https://github.com/foogaro/jdv-playground \
-p SOURCE_REPOSITORY_REF=master \
-p CONTEXT_DIR=ocp/vdbs \
-p EXTENSIONS_REPOSITORY_URL=https://github.com/foogaro/jdv-playground \
-p EXTENSIONS_REPOSITORY_REF=master \
-p EXTENSIONS_DIR=ocp/drivers \
-p EXTENSIONS_DOCKERFILE=Dockerfile \
-p SERVICE_ACCOUNT_NAME=datavirt-service-account \
-p HTTPS_SECRET=datavirt-app-secret \
-p HTTPS_KEYSTORE=keystore.jks \
-p HTTPS_KEYSTORE_TYPE=JKS \
-p HTTPS_NAME=datavirt \
-p HTTPS_PASSWORD=datavirt.2017 \
-p TEIID_USERNAME=teiidUser \
-p TEIID_PASSWORD=datavirt.2017 \
-p MODESHAPE_USERNAME=modeshapeUser \
-p MODESHAPE_PASSWORD=datavirt.2017 \
-p IMAGE_STREAM_NAMESPACE=jdv-playground \
-p JGROUPS_ENCRYPT_SECRET=datavirt-app-secret \
-p JGROUPS_ENCRYPT_KEYSTORE=jgroups.jceks \
-p JGROUPS_ENCRYPT_NAME=datavirt \
-p JGROUPS_ENCRYPT_PASSWORD=datavirt.2017 \
-p JGROUPS_CLUSTER_PASSWORD=datavirt.2017 \
-p VDB_DIRS=
```

This will automatically create the app and start building the image based on the code and resource files.
While is building the first images, we need to create a couple of secrets and link them to our application, as follows: 
```bash
oc create serviceaccount datavirt-service-account
oc policy add-role-to-user view system:serviceaccount:jdv-play:datavirt-service-account -n jdv-playground
oc secrets new datavirt-app-secret keystore.jks jgroups.jceks -n jdv-playground
oc secrets new datavirt-app-config datasources.env -n jdv-playground
oc secrets link datavirt-service-account datavirt-app-secret datavirt-app-config -n jdv-playground
```

Next, create the following environment variables for the build config and the deployment config, as follows:
```bash
oc env bc/datavirt-app VDB_DIRS=
oc env dc/datavirt-app SQLSERVER_DS_DATABASE=DATAVIRT
oc env dc/datavirt-app SQLSERVER_DS_JNDI=java:/SQLSERVER_DS
oc env dc/datavirt-app SQLSERVER_DS_USERNAME=sa
oc env dc/datavirt-app SQLSERVER_DS_PASSWORD=jdv-play.2017
oc env dc/datavirt-app SQLSERVER_DS_URL="jdbc:sqlserver://192.168.59.105:1433;DatabaseName=DATAVIRT;"
oc env dc/datavirt-app SQLSERVER_DS_SERVICE_HOST=192.168.59.105
oc env dc/datavirt-app SQLSERVER_DS_SERVICE_PORT=1433
```

If you need to restart the build, do as follows:
```bash
oc start-build datavirt-app-ext
```

If you need to delete the _serviceaccount_, and the _secrets_, do as follows:
```bash
oc delete secrets datavirt-app-secret
oc delete secrets datavirt-app-config
oc delete serviceaccount datavirt-service-account
```

If you need to provide your own certificates, here is how I created mine:
```bash
keytool -genkeypair -alias datavirt -storetype JKS   -keystore keystore.jks  -storepass "datavirt.2017" -keypass "datavirt.2017" --dname "CN=lfugaro,OU=Consulting,O=redhat.com,L=Raleigh,S=NC,C=US"
keytool -genseckey  -alias datavirt -storetype JCEKS -keystore jgroups.jceks -storepass "datavirt.2017" -keypass "datavirt.2017" --dname "CN=lfugaro,OU=Consulting,O=redhat.com,L=Raleigh,S=NC,C=US"
```

### The web console
If everything worked fine, you should be able to connect to the web console on port 8443, login as admin (password "admin" as well), and see something similar to the following image:

![alt text][ocp-console]

### Getting the data out of the VDB
JDV out-of-the-box exposes its data through the OData protocol version 2 and 4. The schema of the database can be obtained with the following URL:

http://datavirt-app-jdv-playground.apps.foogaro.com/odata4/ITEMS/ITEMS/$metadata

Where `odata4` specifies the protocol version to use; the first _ITEMS_ refers to the VDB name, and the second _ITEMS_ refers to the schema.
Here is how the output should look like:
```xml
<?xml version='1.0' encoding='UTF-8'?><edmx:Edmx Version="4.0" xmlns:edmx="http://docs.oasis-open.org/odata/ns/edmx"><edmx:Reference Uri="http://datavirt-app-jdv-playground.apps.foogaro.com/odata4/static/org.apache.olingo.v1.xml"><edmx:Include Namespace="org.apache.olingo.v1" Alias="olingo-extensions"/></edmx:Reference><edmx:DataServices><Schema xmlns="http://docs.oasis-open.org/odata/ns/edm" Namespace="ITEMS.1.ITEMS" Alias="ITEMS"><EntityType Name="ITEMS"><Key><PropertyRef Name="ITEM_ID"/></Key><Property Name="ITEM_ID" Type="Edm.Int32" Nullable="false"/><Property Name="ITEM_CODE" Type="Edm.String" Nullable="false" MaxLength="20"/><Property Name="ITEM_DESCRITION" Type="Edm.String" MaxLength="255"/><Property Name="DT_INSERT" Type="Edm.DateTimeOffset" Nullable="false" Precision="4"/><Property Name="DT_UPDATE" Type="Edm.DateTimeOffset" Nullable="false" Precision="4"/></EntityType><EntityContainer Name="ITEMS"><EntitySet Name="ITEMS" EntityType="ITEMS.ITEMS"/></EntityContainer></Schema></edmx:DataServices></edmx:Edmx>
```

The XML code above shows the structure of the datatabse, along with the tables, that are the entities, it has.
We have one table named _ITEMS_. To access the table, point to the following URL:

http://datavirt-app-jdv-playground.apps.foogaro.com/odata4/ITEMS/ITEMS/ITEMS

Here is how the output should look like:
```xml
<?xml version='1.0' encoding='UTF-8'?><edmx:Edmx Version="4.0" xmlns:edmx="http://docs.oasis-open.org/odata/ns/edmx">
<edmx:Reference Uri="http://datavirt-app-jdv-playground.apps.foogaro.com/odata4/static/org.apache.olingo.v1.xml">
<edmx:Include Namespace="org.apache.olingo.v1" Alias="olingo-extensions"/>
</edmx:Reference>
<edmx:DataServices><Schema xmlns="http://docs.oasis-open.org/odata/ns/edm" Namespace="ITEMS.1.ITEMS" Alias="ITEMS"><EntityType Name="ITEMS"><Key><PropertyRef Name="ITEM_ID"/></Key><Property Name="ITEM_ID" Type="Edm.Int32" Nullable="false"/><Property Name="ITEM_CODE" Type="Edm.String" Nullable="false" MaxLength="20"/><Property Name="ITEM_DESCRITION" Type="Edm.String" MaxLength="255"/><Property Name="DT_INSERT" Type="Edm.DateTimeOffset" Nullable="false" Precision="4"/><Property Name="DT_UPDATE" Type="Edm.DateTimeOffset" Nullable="false" Precision="4"/></EntityType><EntityContainer Name="ITEMS"><EntitySet Name="ITEMS" EntityType="ITEMS.ITEMS"/></EntityContainer></Schema></edmx:DataServices></edmx:Edmx>
```

You can eventually have the result in different format, for example in __JSON__, by specifing the _$format_, as follows:

http://datavirt-app-jdv-playground.apps.foogaro.com/odata4/ITEMS/ITEMS/ITEMS?$format=json

Here is how the output should look like:
```json
{"@odata.context":"$metadata#ITEMS","value":[{"ITEM_ID":1,"ITEM_CODE":"0000-0000-0000-0001","ITEM_DESCRITION":"One","DT_INSERT":"2017-12-09T19:52:50.797Z","DT_UPDATE":"2017-12-09T19:52:50.797Z"},{"ITEM_ID":2,"ITEM_CODE":"0000-0000-0000-0002","ITEM_DESCRITION":"Two","DT_INSERT":"2017-12-09T19:52:50.797Z","DT_UPDATE":"2017-12-09T19:52:50.797Z"},{"ITEM_ID":3,"ITEM_CODE":"0000-0000-0000-0003","ITEM_DESCRITION":"Three","DT_INSERT":"2017-12-09T19:52:50.8Z","DT_UPDATE":"2017-12-09T19:52:50.8Z"},{"ITEM_ID":4,"ITEM_CODE":"0000-0000-0000-0004","ITEM_DESCRITION":"Four","DT_INSERT":"2017-12-09T19:52:50.807Z","DT_UPDATE":"2017-12-09T19:52:50.807Z"},{"ITEM_ID":5,"ITEM_CODE":"0000-0000-0000-0005","ITEM_DESCRITION":"Five","DT_INSERT":"2017-12-09T19:52:50.807Z","DT_UPDATE":"2017-12-09T19:52:50.807Z"},{"ITEM_ID":6,"ITEM_CODE":"0000-0000-0000-0006","ITEM_DESCRITION":"Six","DT_INSERT":"2017-12-09T19:52:50.81Z","DT_UPDATE":"2017-12-09T19:52:50.81Z"},{"ITEM_ID":7,"ITEM_CODE":"0000-0000-0000-0007","ITEM_DESCRITION":"Seven","DT_INSERT":"2017-12-09T19:52:50.81Z","DT_UPDATE":"2017-12-09T19:52:50.81Z"},{"ITEM_ID":8,"ITEM_CODE":"0000-0000-0000-0008","ITEM_DESCRITION":"Eight","DT_INSERT":"2017-12-09T19:52:50.817Z","DT_UPDATE":"2017-12-09T19:52:50.817Z"},{"ITEM_ID":9,"ITEM_CODE":"0000-0000-0000-0009","ITEM_DESCRITION":"Nine","DT_INSERT":"2017-12-09T19:52:50.817Z","DT_UPDATE":"2017-12-09T19:52:50.817Z"},{"ITEM_ID":10,"ITEM_CODE":"0000-0000-0000-0010","ITEM_DESCRITION":"Ten","DT_INSERT":"2017-12-09T19:52:50.82Z","DT_UPDATE":"2017-12-09T19:52:50.82Z"}]}
```


That's it, I hope it helped!


Ciao,
Luigi

[ocp-console]: https://github.com/foogaro/jdv-playground/blob/master/ocp/ocp-console.png "OpenShift Container Platform"
