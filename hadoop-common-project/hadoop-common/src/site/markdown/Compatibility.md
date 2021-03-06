<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

Apache Hadoop Compatibility
===========================

* [Apache Hadoop Compatibility](#Apache_Hadoop_Compatibility)
    * [Purpose](#Purpose)
    * [Compatibility types](#Compatibility_types)
        * [Java API](#Java_API)
            * [Use Cases](#Use_Cases)
            * [Policy](#Policy)
        * [Semantic compatibility](#Semantic_compatibility)
            * [Policy](#Policy)
        * [Wire compatibility](#Wire_compatibility)
            * [Use Cases](#Use_Cases)
            * [Policy](#Policy)
        * [Java Binary compatibility for end-user applications i.e. Apache Hadoop ABI](#Java_Binary_compatibility_for_end-user_applications_i.e._Apache_Hadoop_ABI)
            * [Use cases](#Use_cases)
            * [Policy](#Policy)
        * [REST APIs](#REST_APIs)
            * [Policy](#Policy)
        * [Metrics/JMX](#MetricsJMX)
            * [Policy](#Policy)
        * [File formats & Metadata](#File_formats__Metadata)
            * [User-level file formats](#User-level_file_formats)
                * [Policy](#Policy)
            * [System-internal file formats](#System-internal_file_formats)
                * [MapReduce](#MapReduce)
                * [Policy](#Policy)
                * [HDFS Metadata](#HDFS_Metadata)
                * [Policy](#Policy)
        * [Command Line Interface (CLI)](#Command_Line_Interface_CLI)
            * [Policy](#Policy)
        * [Web UI](#Web_UI)
            * [Policy](#Policy)
        * [Hadoop Configuration Files](#Hadoop_Configuration_Files)
            * [Policy](#Policy)
        * [Directory Structure](#Directory_Structure)
            * [Policy](#Policy)
        * [Java Classpath](#Java_Classpath)
            * [Policy](#Policy)
        * [Environment variables](#Environment_variables)
            * [Policy](#Policy)
        * [Build artifacts](#Build_artifacts)
            * [Policy](#Policy)
        * [Hardware/Software Requirements](#HardwareSoftware_Requirements)
            * [Policies](#Policies)
    * [References](#References)

Purpose
-------

This document captures the compatibility goals of the Apache Hadoop project. The different types of compatibility between Hadoop releases that affects Hadoop developers, downstream projects, and end-users are enumerated. For each type of compatibility we:

* describe the impact on downstream projects or end-users
* where applicable, call out the policy adopted by the Hadoop developers when incompatible changes are permitted.

Compatibility types
-------------------

### Java API

Hadoop interfaces and classes are annotated to describe the intended audience and stability in order to maintain compatibility with previous releases. See [Hadoop Interface Classification](./InterfaceClassification.html) for details.

* InterfaceAudience: captures the intended audience, possible values are Public (for end users and external projects), LimitedPrivate (for other Hadoop components, and closely related projects like YARN, MapReduce, HBase etc.), and Private (for intra component use).
* InterfaceStability: describes what types of interface changes are permitted. Possible values are Stable, Evolving, Unstable, and Deprecated.

#### Use Cases

* Public-Stable API compatibility is required to ensure end-user programs and downstream projects continue to work without modification.
* LimitedPrivate-Stable API compatibility is required to allow upgrade of individual components across minor releases.
* Private-Stable API compatibility is required for rolling upgrades.

#### Policy

* Public-Stable APIs must be deprecated for at least one major release prior to their removal in a major release.
* LimitedPrivate-Stable APIs can change across major releases, but not within a major release.
* Private-Stable APIs can change across major releases, but not within a major release.
* Classes not annotated are implicitly "Private". Class members not annotated inherit the annotations of the enclosing class.
* Note: APIs generated from the proto files need to be compatible for rolling-upgrades. See the section on wire-compatibility for more details. The compatibility policies for APIs and wire-communication need to go hand-in-hand to address this.

### Semantic compatibility

Apache Hadoop strives to ensure that the behavior of APIs remains consistent over versions, though changes for correctness may result in changes in behavior. Tests and javadocs specify the API's behavior. The community is in the process of specifying some APIs more rigorously, and enhancing test suites to verify compliance with the specification, effectively creating a formal specification for the subset of behaviors that can be easily tested.

#### Policy

The behavior of API may be changed to fix incorrect behavior, such a change to be accompanied by updating existing buggy tests or adding tests in cases there were none prior to the change.

### Wire compatibility

Wire compatibility concerns data being transmitted over the wire between Hadoop processes. Hadoop uses Protocol Buffers for most RPC communication. Preserving compatibility requires prohibiting modification as described below. Non-RPC communication should be considered as well, for example using HTTP to transfer an HDFS image as part of snapshotting or transferring MapTask output. The potential communications can be categorized as follows:

* Client-Server: communication between Hadoop clients and servers (e.g., the HDFS client to NameNode protocol, or the YARN client to ResourceManager protocol).
* Client-Server (Admin): It is worth distinguishing a subset of the Client-Server protocols used solely by administrative commands (e.g., the HAAdmin protocol) as these protocols only impact administrators who can tolerate changes that end users (which use general Client-Server protocols) can not.
* Server-Server: communication between servers (e.g., the protocol between the DataNode and NameNode, or NodeManager and ResourceManager)

#### Use Cases

* Client-Server compatibility is required to allow users to continue using the old clients even after upgrading the server (cluster) to a later version (or vice versa). For example, a Hadoop 2.1.0 client talking to a Hadoop 2.3.0 cluster.
* Client-Server compatibility is also required to allow users to upgrade the client before upgrading the server (cluster). For example, a Hadoop 2.4.0 client talking to a Hadoop 2.3.0 cluster. This allows deployment of client-side bug fixes ahead of full cluster upgrades. Note that new cluster features invoked by new client APIs or shell commands will not be usable. YARN applications that attempt to use new APIs (including new fields in data structures) that have not yet deployed to the cluster can expect link exceptions.
* Client-Server compatibility is also required to allow upgrading individual components without upgrading others. For example, upgrade HDFS from version 2.1.0 to 2.2.0 without upgrading MapReduce.
* Server-Server compatibility is required to allow mixed versions within an active cluster so the cluster may be upgraded without downtime in a rolling fashion.

#### Policy

* Both Client-Server and Server-Server compatibility is preserved within a major release. (Different policies for different categories are yet to be considered.)
* Compatibility can be broken only at a major release, though breaking compatibility even at major releases has grave consequences and should be discussed in the Hadoop community.
* Hadoop protocols are defined in .proto (ProtocolBuffers) files. Client-Server protocols and Server-protocol .proto files are marked as stable. When a .proto file is marked as stable it means that changes should be made in a compatible fashion as described below:
    * The following changes are compatible and are allowed at any time:
        * Add an optional field, with the expectation that the code deals with the field missing due to communication with an older version of the code.
        * Add a new rpc/method to the service
        * Add a new optional request to a Message
        * Rename a field
        * Rename a .proto file
        * Change .proto annotations that effect code generation (e.g. name of java package)
    * The following changes are incompatible but can be considered only at a major release
        * Change the rpc/method name
        * Change the rpc/method parameter type or return type
        * Remove an rpc/method
        * Change the service name
        * Change the name of a Message
        * Modify a field type in an incompatible way (as defined recursively)
        * Change an optional field to required
        * Add or delete a required field
        * Delete an optional field as long as the optional field has reasonable defaults to allow deletions
    * The following changes are incompatible and hence never allowed
        * Change a field id
        * Reuse an old field that was previously deleted.
        * Field numbers are cheap and changing and reusing is not a good idea.

### Java Binary compatibility for end-user applications i.e. Apache Hadoop ABI

As Apache Hadoop revisions are upgraded end-users reasonably expect that their applications should continue to work without any modifications. This is fulfilled as a result of support API compatibility, Semantic compatibility and Wire compatibility.

However, Apache Hadoop is a very complex, distributed system and services a very wide variety of use-cases. In particular, Apache Hadoop MapReduce is a very, very wide API; in the sense that end-users may make wide-ranging assumptions such as layout of the local disk when their map/reduce tasks are executing, environment variables for their tasks etc. In such cases, it becomes very hard to fully specify, and support, absolute compatibility.

#### Use cases

* Existing MapReduce applications, including jars of existing packaged end-user applications and projects such as Apache Pig, Apache Hive, Cascading etc. should work unmodified when pointed to an upgraded Apache Hadoop cluster within a major release.
* Existing YARN applications, including jars of existing packaged end-user applications and projects such as Apache Tez etc. should work unmodified when pointed to an upgraded Apache Hadoop cluster within a major release.
* Existing applications which transfer data in/out of HDFS, including jars of existing packaged end-user applications and frameworks such as Apache Flume, should work unmodified when pointed to an upgraded Apache Hadoop cluster within a major release.

#### Policy

* Existing MapReduce, YARN & HDFS applications and frameworks should work unmodified within a major release i.e. Apache Hadoop ABI is supported.
* A very minor fraction of applications maybe affected by changes to disk layouts etc., the developer community will strive to minimize these changes and will not make them within a minor version. In more egregious cases, we will consider strongly reverting these breaking changes and invalidating offending releases if necessary.
* In particular for MapReduce applications, the developer community will try our best to support provide binary compatibility across major releases e.g. applications using org.apache.hadoop.mapred.
* APIs are supported compatibly across hadoop-1.x and hadoop-2.x. See [Compatibility for MapReduce applications between hadoop-1.x and hadoop-2.x](../../hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduce_Compatibility_Hadoop1_Hadoop2.html) for more details.

### REST APIs

REST API compatibility corresponds to both the request (URLs) and responses to each request (content, which may contain other URLs). Hadoop REST APIs are specifically meant for stable use by clients across releases, even major releases. The following are the exposed REST APIs:

* [WebHDFS](../hadoop-hdfs/WebHDFS.html) - Stable
* [ResourceManager](../../hadoop-yarn/hadoop-yarn-site/ResourceManagerRest.html)
* [NodeManager](../../hadoop-yarn/hadoop-yarn-site/NodeManagerRest.html)
* [MR Application Master](../../hadoop-yarn/hadoop-yarn-site/MapredAppMasterRest.html)
* [History Server](../../hadoop-yarn/hadoop-yarn-site/HistoryServerRest.html)

#### Policy

The APIs annotated stable in the text above preserve compatibility across at least one major release, and maybe deprecated by a newer version of the REST API in a major release.

### Metrics/JMX

While the Metrics API compatibility is governed by Java API compatibility, the actual metrics exposed by Hadoop need to be compatible for users to be able to automate using them (scripts etc.). Adding additional metrics is compatible. Modifying (eg changing the unit or measurement) or removing existing metrics breaks compatibility. Similarly, changes to JMX MBean object names also break compatibility.

#### Policy

Metrics should preserve compatibility within the major release.

### File formats & Metadata

User and system level data (including metadata) is stored in files of different formats. Changes to the metadata or the file formats used to store data/metadata can lead to incompatibilities between versions.

#### User-level file formats

Changes to formats that end-users use to store their data can prevent them for accessing the data in later releases, and hence it is highly important to keep those file-formats compatible. One can always add a "new" format improving upon an existing format. Examples of these formats include har, war, SequenceFileFormat etc.

##### Policy

* Non-forward-compatible user-file format changes are restricted to major releases. When user-file formats change, new releases are expected to read existing formats, but may write data in formats incompatible with prior releases. Also, the community shall prefer to create a new format that programs must opt in to instead of making incompatible changes to existing formats.

#### System-internal file formats

Hadoop internal data is also stored in files and again changing these formats can lead to incompatibilities. While such changes are not as devastating as the user-level file formats, a policy on when the compatibility can be broken is important.

##### MapReduce

MapReduce uses formats like I-File to store MapReduce-specific data.

##### Policy

MapReduce-internal formats like IFile maintain compatibility within a major release. Changes to these formats can cause in-flight jobs to fail and hence we should ensure newer clients can fetch shuffle-data from old servers in a compatible manner.

##### HDFS Metadata

HDFS persists metadata (the image and edit logs) in a particular format. Incompatible changes to either the format or the metadata prevent subsequent releases from reading older metadata. Such incompatible changes might require an HDFS "upgrade" to convert the metadata to make it accessible. Some changes can require more than one such "upgrades".

Depending on the degree of incompatibility in the changes, the following potential scenarios can arise:

* Automatic: The image upgrades automatically, no need for an explicit "upgrade".
* Direct: The image is upgradable, but might require one explicit release "upgrade".
* Indirect: The image is upgradable, but might require upgrading to intermediate release(s) first.
* Not upgradeable: The image is not upgradeable.

##### Policy

* A release upgrade must allow a cluster to roll-back to the older version and its older disk format. The rollback needs to restore the original data, but not required to restore the updated data.
* HDFS metadata changes must be upgradeable via any of the upgrade paths - automatic, direct or indirect.
* More detailed policies based on the kind of upgrade are yet to be considered.

### Command Line Interface (CLI)

The Hadoop command line programs may be use either directly via the system shell or via shell scripts. Changing the path of a command, removing or renaming command line options, the order of arguments, or the command return code and output break compatibility and may adversely affect users.

#### Policy

CLI commands are to be deprecated (warning when used) for one major release before they are removed or incompatibly modified in a subsequent major release.

### Web UI

Web UI, particularly the content and layout of web pages, changes could potentially interfere with attempts to screen scrape the web pages for information.

#### Policy

Web pages are not meant to be scraped and hence incompatible changes to them are allowed at any time. Users are expected to use REST APIs to get any information.

### Hadoop Configuration Files

Users use (1) Hadoop-defined properties to configure and provide hints to Hadoop and (2) custom properties to pass information to jobs. Hence, compatibility of config properties is two-fold:

* Modifying key-names, units of values, and default values of Hadoop-defined properties.
* Custom configuration property keys should not conflict with the namespace of Hadoop-defined properties. Typically, users should avoid using prefixes used by Hadoop: hadoop, io, ipc, fs, net, file, ftp, s3, kfs, ha, file, dfs, mapred, mapreduce, yarn.

#### Policy

* Hadoop-defined properties are to be deprecated at least for one major release before being removed. Modifying units for existing properties is not allowed.
* The default values of Hadoop-defined properties can be changed across minor/major releases, but will remain the same across point releases within a minor release.
* Currently, there is NO explicit policy regarding when new prefixes can be added/removed, and the list of prefixes to be avoided for custom configuration properties. However, as noted above, users should avoid using prefixes used by Hadoop: hadoop, io, ipc, fs, net, file, ftp, s3, kfs, ha, file, dfs, mapred, mapreduce, yarn.

### Directory Structure

Source code, artifacts (source and tests), user logs, configuration files, output and job history are all stored on disk either local file system or HDFS. Changing the directory structure of these user-accessible files break compatibility, even in cases where the original path is preserved via symbolic links (if, for example, the path is accessed by a servlet that is configured to not follow symbolic links).

#### Policy

* The layout of source code and build artifacts can change anytime, particularly so across major versions. Within a major version, the developers will attempt (no guarantees) to preserve the directory structure; however, individual files can be added/moved/deleted. The best way to ensure patches stay in sync with the code is to get them committed to the Apache source tree.
* The directory structure of configuration files, user logs, and job history will be preserved across minor and point releases within a major release.

### Java Classpath

User applications built against Hadoop might add all Hadoop jars (including Hadoop's library dependencies) to the application's classpath. Adding new dependencies or updating the version of existing dependencies may interfere with those in applications' classpaths.

#### Policy

Currently, there is NO policy on when Hadoop's dependencies can change.

### Environment variables

Users and related projects often utilize the exported environment variables (eg HADOOP\_CONF\_DIR), therefore removing or renaming environment variables is an incompatible change.

#### Policy

Currently, there is NO policy on when the environment variables can change. Developers try to limit changes to major releases.

### Build artifacts

Hadoop uses maven for project management and changing the artifacts can affect existing user workflows.

#### Policy

* Test artifacts: The test jars generated are strictly for internal use and are not expected to be used outside of Hadoop, similar to APIs annotated @Private, @Unstable.
* Built artifacts: The hadoop-client artifact (maven groupId:artifactId) stays compatible within a major release, while the other artifacts can change in incompatible ways.

### Hardware/Software Requirements

To keep up with the latest advances in hardware, operating systems, JVMs, and other software, new Hadoop releases or some of their features might require higher versions of the same. For a specific environment, upgrading Hadoop might require upgrading other dependent software components.

#### Policies

* Hardware
    * Architecture: The community has no plans to restrict Hadoop to specific architectures, but can have family-specific optimizations.
    * Minimum resources: While there are no guarantees on the minimum resources required by Hadoop daemons, the community attempts to not increase requirements within a minor release.
* Operating Systems: The community will attempt to maintain the same OS requirements (OS kernel versions) within a minor release. Currently GNU/Linux and Microsoft Windows are the OSes officially supported by the community while Apache Hadoop is known to work reasonably well on other OSes such as Apple MacOSX, Solaris etc.
* The JVM requirements will not change across point releases within the same minor release except if the JVM version under question becomes unsupported. Minor/major releases might require later versions of JVM for some/all of the supported operating systems.
* Other software: The community tries to maintain the minimum versions of additional software required by Hadoop. For example, ssh, kerberos etc.

References
----------

Here are some relevant JIRAs and pages related to the topic:

* The evolution of this document - [HADOOP-9517](https://issues.apache.org/jira/browse/HADOOP-9517)
* Binary compatibility for MapReduce end-user applications between hadoop-1.x and hadoop-2.x - [MapReduce Compatibility between hadoop-1.x and hadoop-2.x](../../hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduce_Compatibility_Hadoop1_Hadoop2.html)
* Annotations for interfaces as per interface classification schedule - [HADOOP-7391](https://issues.apache.org/jira/browse/HADOOP-7391) [Hadoop Interface Classification](./InterfaceClassification.html)
* Compatibility for Hadoop 1.x releases - [HADOOP-5071](https://issues.apache.org/jira/browse/HADOOP-5071)
* The [Hadoop Roadmap](http://wiki.apache.org/hadoop/Roadmap) page that captures other release policies


