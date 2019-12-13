# iRODS for power users

** Authors**
- Arthur Newton (SURFsara)

**License**
Copyright 2019 SURFsara BV

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.


## Goal
Provide a few entry points for power users to handle data within iRODS. We will cover the installation, configuration and basic functionality of the native CLI client of iRODS, the icommands. The installation, configuration and initial steps in Python API will also be covered. 


## icommands
The icommands are a powerful high performant set of tools that can be installed in any unix environment. It can be used both for administrative tasks as well as all the data handling. it is especially powerful, because it makes full use of the ability of iRODS to transfer a file in parallel. It can securely send a large file completely filling up the available network. Note that the icommands can be solely operated from within the terminal. Following is all based on assuming basic knowledge of the terminal.

### installation
Installation on different linux distros need to be done in a terminal. Basic knowledge of terminal commands is assumed.

For Ubuntu 18.04:

```sh
wget -qO - https://packages.irods.org/irods-signing-key.asc | sudo apt-key add -
echo "deb [arch=amd64] https://packages.irods.org/apt/ xenial main" | sudo tee /etc/apt/sources.list.d/renci-irods.list
sudo apt-get update
sudo apt -y install irods-icommands
```

For CentOS:

```sh
sudo yum -y install wget epel-release
sudo rpm --import https://packages.irods.org/irods-signing-key.asc
wget -qO - https://packages.irods.org/renci-irods.yum.repo | sudo tee /etc/yum.repos.d/renci-irods.yum.repo
sudo yum -y install irods-icommands
```

For MacOS:

There are no up-to-date packages for MacOS writing this document. Either the icommands can be built from source (https://github.com/irods/irods_client_icommands), however it is not trivial.

The easiest installation on Mac is via the installer provided by Cyverse:
https://wiki.cyverse.org/wiki/display/DS/Setting+Up+iCommands#SettingUpiCommands-mac
where you can follow the steps. 

For Windows:

The icommands work in Linux. They should be installable via the Ubuntu method on the linux subsystem of Windows 10. 


### Connecting to iRODS
To connect to iRODS you need to know the hostname of iRODS, the port number used for connection, and obviously your username and password. 
If you have a simple iRODS installation without SSL (not recommended for production), you can simply connect via:

```sh
iinit
```

The system will ask you for information where to connect to:

```
Enter the host name (DNS) of the server to connect to:  your_hostname
Enter the port number: 1247
Enter your irods user name: your_username
Enter your irods zone: your_zonename
```

Provide your hostname, zonename, username and password, and there you are. The environment variables will be stored in `~/.irods/irods_environment.json` and a scrambled password file, `~/.irods/irodsA`, will be created which can be used for subsequent connections.


If you want to connecto to an YODA enabled iRODS zone, the `iinit` command does not cover the full connection details. You will have to manually create the `~/.irods/irods_environment.json` file and edit as follows:

```sh
{
    "irods_host": "your_hostname",
    "irods_port": 1247,
    "irods_home": "/your_zonename/home",
    "irods_user_name": "your_username",
    "irods_zone_name": "your_zonename",
    "irods_authentication_scheme": "pam"
}
```

Note to change the specific details.

You can find help for the icommands with:

```sh
ihelp
```

which will show you all the possible commands. 

Additionally, the following commands gives you more information about the iRODS environment (typically not needed):

```sh
ienv
```


### Basic data/collection handling
Note that in iRODS files are called data objects, and folders are called collections. 

The icommands have basic file handling functionality that have their (almost) equivalent in the Linux bash commands but with an `i` in front of the name, *e.g.* `ls` and `ils`, `cp` and `icp`, `mv` and `imv`, `pwd` and `ipwd`, `mkdir` and `imkdir`, `cd` and `icd`.  `rm` and `irm`. 
Try out `icommand -h` if you want to know more about the available options.


#### Uploading a file with `iput`
In order to get a file in iRODS:

```sh
iput <source_file> <dest_do>
```

when no destination data object is given, the local filename will be used. iRODS will issue a warning if the file already exists (which you can force to do with the `-f` option).

In order to get a folder in iRODS:

```sh
iput -r <source_folder> <dest_coll>
```

you need to add the `-r` option for recursively putting data in iRODS.

You can also immediately add metadata to the data object or collection by doing:

```sh
iput <source_file> --metadata "key1;val1;unit1;key2;val2;unit2"
```

Per metadata item you can store three strings: key, value, unit. You can dismiss the unit string. In the above example simply leave out the string:

```sh
iput <source_file> --metadata "key1;val1;;key2;val2;unit2"
```

Depending on the policy of your iRODS zone, you can also immediately calculate the checksum while uploading a file:

```sh
iput -K <source_file>
```

Note that his can also be done afterwards with the `ichksum` command or by using an asynchronous rule. 

There are many more options for `iput` which could (or could not) optimize data transfer, *e.g.* `-b` for bulk upload to overcome network overhead, `-N` for number of threads, `-Q` for Reliable Blast UDP protocol.


#### Downloading a file with `iget`
To download a file, you can use the `iget` command:

```sh
iget <source_do> <dest_file>
```

Again, if the local destination is not specified, the name of the data object will be used. 

To download a collection you have to specify the `-r` option:

```sh
iget -r <source_coll> <dest_folder>
```


### Adding metadata and querying for data
In iRODS a data object is not only the bitstream and the filename, but user defined metadata is part of the data object. This makes it very powerfull, as metadata and data can not be out of sync which can happen in other custom solutions where it is not so integrated. You can manually add metadata or let a rule add metadata. Data objects can also be found by querying for metadata.

#### Metadata handling
Manual metadata handling can be done via the `imeta` command. 
For each command, -d, -C, -R, or -u is used to specify which type of
object to work with: dataobjs (iRODS files), collections, resources,
or users.
To add metadata to a data object or a collection:

```sh
imeta add -d <do_name> Key Val Unit

imeta add -C <coll_name> Key Val Unit
```

Again, it is possible to leave out the unit, and you can set multiple key value pairs with the same key. There is not logical limit to the number of metadata items per data object or collection. 

You can remove metadata as such:

```sh
imeta rm -d <do_name> Key Val Unit

imeta rm -C <coll_name> Key Val Unit
```

where you do have to explicitly specify the whole set of key, value and unit to remove it. You can use wildcards (`%`) with `imeta rmw`.
You can also modify, copy, show metadata by the `imeta mod`, `imeta cp`, `imeta ls` commands respectively:

```sh
imeta mod -d <do_name> key1 val1 unit1 n:key1 v:diffval1 u:unit1

imeta cp -C -C <coll_name1> <coll_name2>

imeta ls -C <coll_name>
```


#### Querying based on metadata

Metadata attached to data objects and collections becomes very usefull when you want to query for data. Queries can mainly be performed by the `iquest` command, but simple queries can be done by the `imeta` command:

```sh
#find data objects where distance is larger than 12
imeta qu -d distance '<=' 12

#find collections where key a is value b and key c smaller than value 10
imeta qu -C a = b and c '<' 10

#find data objects where r is not in between 5 and 7
imeta qu -d r '<' 5 or '>' 7

#find data-objects with attribute 'a' with a value that starts with 'b'.
imeta qu -d a like b%

#find data-objects with attribute 'a' defined (with any value).
imeta qu -d a like %
```

You can enter the `imeta` prompt by just typing `imeta` and Enter. You can get more helpfull tips by then doing `help qu`.

More complicated queries can be performed via the `iquest` command which uses SQL like syntax. Do note that not all SQL options are available by default, but rodsadmin users can add custom queries via the `iadmin asq`.

You can look up the available keys you can query for via:

```sh
iquest attrs
```
Note that these ar a lot.

General `iquest` queries look like:

```sh
iquest "select COLL_NAME, DATA_NAME, META_DATA_ATTR_VALUE where \
META_DATA_ATTR_NAME like 'author'" 
```

In above command we wanted to look for everything which has the metadata attribute name (metadata key) that resembles 'author' and retrieve the collection name, data name and metadata attribute value (metadata value). 
iRODS will respond something like:

```sh
COLL_NAME = /tempZone/home/username
DATA_NAME = testfile
META_DATA_ATTR_VALUE = val1
------------------------------------------------------------
```

You can change the format of the iRODS response by using a C like format string after `iquest` like so:

```sh
iquest "User %-6.6s has %-5.5s access to file %s" "SELECT USER_NAME,  DATA_ACCESS_NAME, DATA_NAME WHERE COLL_NAME = '/tempZone/home/rods'"
```
where after `iquest` there is the format string and the second string is again the SQL like query.  This can be convenient for making reports but also for retrieving absolute filepaths:

```sh
iquest "%s/%s" "select COLL_NAME, DATA_NAME where \
META_DATA_ATTR_NAME like 'author' and META_DATA_ATTR_VALUE = 'Lewis Carroll'" 
```

This could be useful in concatenating commands in HPC data staging:

```sh
iquest "%s/%s" "select COLL_NAME, DATA_NAME where \
META_DATA_ATTR_NAME like 'key1' and META_DATA_ATTR_VALUE = 'val1'" | xargs iget
```



## Python iRODS API

There is also a Python iRODS API which is quite complete and can be used for iRODS applications or more compute jobs. iPython can be used to interactively test out the following python commands.

### Installation
Installation of the python iRODS API is easily done via pip:

```sh
pip install git+https://github.com/irods/python-irodsclient.git
```

which should work for most distributions (Linux, Mac, Windows).

### Establish a connection
It is most convenient if you already have an `~/.irods/irods_environment` file. 


Import the iRODS module:

```py
from irods.session import iRODSSession
```

First create a session variable via the environment file:

```py
try:
     env_file = os.environ['IRODS_ENVIRONMENT_FILE'
except KeyError:
     env_file = os.path.expanduser('~/.irods/irods_environment.json')
     
session = iRODSSession(irods_env_file=env_file)
```

or via explicitly specifying the irods environment variables:

```py
session = iRODSSession(host='<HOSTNAME>', port=1247, user='<USERNAME>', password='<PASSWORD>', zone='<ZONENAME>')
```

We can test whether we have done everything correctly and have access:

```py
coll = session.collections.get('/aliceZone/home/irods-user1')
print(coll.path)
print(coll.data_objects)
print(coll.subcollections)
```

### Upload a data object
The preferred way to upload data to iRODS is a data object *put*. 


```py

session.data_objects.put('<LOCALFILE>', '<IRODSPATH>')
```
The object carries some vital system information, otherwise it is empty. 

```
obj = session.data_objects.get('<IRODSPATH>')
print("Name: ", obj.name)
print("Owner: ", obj.owner_name)
print("Size: ", obj.size)
print("Checksum:", obj.checksum)
print("Create: ", obj.create_time)
print("Modify: ", obj.modify_time)
print("Metadata: ", obj.metadata.items())
```

### Download a data object
We can download a data object as follows:

```py
obj = session.data_objects.get('<IRODSPATH>','<LOCALFILE>')
```

### Creating metadata
Working with metadata is not completely intuitive, you need a good understanding of python dictionaries and the iRODS python API classes *dataobject*, *collection*, *iRODSMetaData* and *iRODSMetaCollection*.

We start slowly with first creating some metadata for our data. 
Currently, our data object does not carry any user-defined metadata:

```py
obj = session.data_objects.get('<IRODSPATH>')
print(obj.metadata.items())
```

Create a key, value, unit entry for our data object:

```py
obj.metadata.add('SOURCE', 'python API training', 'version 1')
obj.metadata.add('TYPE', 'test file')
```
If you now print the metadata again, you will see a cryptic list:

```py
print(obj.metadata.items())
```
The list contains two metadata python objects.
To work with the metadata you need to iterate over them and extract the AVU triples:

```py
[(item.name, item.value, item.units) for item in obj.metadata.items()]
```
Metadata can be used to search for your own data but also for data that someone shared with you. You do not need to know the exact iRODS logical path to retrieve the file, you can search for data wich is annotated accordingly. We will see that in the next section.


### Searching for data in iRODS
We will now try to find all data in this iRODS instance we have access to and which carries the key *author* with value *Lewis Carroll*. And we need to assemble the iRODS logical path.

```py
from irods.models import Collection, DataObject, CollectionMeta, DataObjectMeta
```

We need the collection name and data object name of the data objects. This command will give us all data objects we have access to:

```py
query = session.query(Collection.name, DataObject.name)
```

Now we can filter the results for data objects which carry a user-defined metadata item with name 'author' and value 'Lewis Carroll'. To this end we have to chain two filters:

```py
filteredQuery = query.filter( \
        Criterion('=', DataObjectMeta.name, 'author')).filter( \
        Criterion('like', DataObjectMeta.value, 'Lewis Carroll'))
print(filteredQuery.all())
```

Python prints the results neatly on the prompt, however to extract the information and parsing it to other functions is pretty complicated. Every entry you see in the output is not a string, but actually a python object with many functions. That gives you the advantage to link the output to the rows and comlumns in the sql database running in the background of iRODS. For normal user interaction, however, it needs some explanation and help.

#### Parsing the iquest output
To work with the results of the query, we need to get them in an iterable format:

```py
results = filteredQuery.get_results()
```
**Watch out**: *results* is a generator which you can only use once to iterate over.

We can now iterate over the results and build our iRODS paths (*COLL_NAME/DATA_NAME*) of the data files:

```py
iPaths = []

for item in results:
    for k in item.keys():
        if k.icat_key == 'DATA_NAME':
            name = item[k]
        elif k.icat_key == 'COLL_NAME':
            coll = item[k]
        else:
            continue
    iPaths.append(coll+'/'+name)
print('\n'.join(iPaths))
```

How did we know which keys to use? 
We asked in the query for *Collection.name* and *DataObject.name*.
Have look at these two objects:

```py
print(Collection.name.icat_key)
print(DataObject.name.icat_key)
```
The *icat_key* is the keyword used in the database behind iRODS to store the information.

```py
DataObject.checksum
```
which we can use in the query:

```py
query = session.query(Collection.name, 
    DataObject.name, 
    DataObject.checksum, 
    DataObject.size, 
    DataObjectMeta.value)
					  
filteredQuery = query.filter(DataObjectMeta.name == 'author').\
    filter(DataObjectMeta.value == 'Lewis Carroll')
print(filteredQuery.all())
```
Metadata that the user creates with *obj.metadata.add* or *coll.metadata.add* are accessible via *DataObjectMeta* or *CollectionMeta* respectively. Other metadata is directly stored as attributes in *Collection* or *DataObject*.