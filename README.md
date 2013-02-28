Analyzing Twitter Data Using CDH
================================

Based on http://blog.cloudera.com/blog/2012/09/analyzing-twitter-data-with-hadoop/.

This repository contains an example application for analyzing Twitter data using a variety of CDH components, including [Flume](http://flume.apache.org), [Oozie](http://incubator.apache.org/oozie), and [Hive](http://hive.apache.org).

Getting Started
---------------

1. **Install Cloudera Manager 4.0 and CDH4**

   Before you get started with the actual application, you'll first need CDH4 installed. Specifically, you'll need Hadoop, Flume, Oozie, and Hive. The easiest way to get the core components is to use Cloudera Manager to set up your initial environment. You can download Cloudera Manager from the [Cloudera website](https://ccp.cloudera.com/display/SUPPORT/Cloudera+Manager+Downloads#ClouderaManagerDownloads-ClouderaManager4.0), or install [CDH](https://ccp.cloudera.com/display/SUPPORT/CDH+Downloads#CDHDownloads-CDH4PackagesandDownloads) manually.

   If you go the Cloudera Manager route, you'll still need to [install Flume manually](https://ccp.cloudera.com/display/CDH4DOC/Flume+Installation).

2. **Install MySQL**

    MySQL is the recommended database for the Oozie database and the Hive metastore. Click [here](http://dev.mysql.com/doc/refman/5.1/en/linux-installation-native.html) for installation documentation.

Configuring Flume
------------------

1. **Build or Download the custom Flume Source**

   A pre-built version of the custom Flume Source is available [here](http://files.cloudera.com/samples/flume-sources-1.0-SNAPSHOT.jar).

   The `flume-sources` directory contains a Maven project with a custom Flume source designed to connect to the Twitter Streaming API and ingest tweets in a raw JSON format into HDFS.

   To build the flume-sources JAR, from the root of the git repository:

   <pre>
   $ cd flume-sources  
   $ mvn package
   $ cd ..  
   </pre>

   This will generate a file called `flume-sources-1.0-SNAPSHOT.jar` in the `target` directory.

2. **Add the JAR to the Flume classpath**

   <pre>$ sudo cp /etc/flume-ng/conf/flume-env.sh.template /etc/flume-ng/conf/flume-env.sh</pre>
   
    Edit the `flume-env.sh` file and uncomment the `FLUME_CLASSPATH` line, and enter the path to the JAR. If adding multiple paths, separate them with a colon.

3. **Set the Flume agent name to TwitterAgent in /etc/default/flume-ng-agent**

    If you don't see the `/etc/default/flume-ng-agent` file, it likely means that you didn't install the `flume-ng-agent` package. In the file, you should have the following:

    <pre>FLUME_AGENT_NAME=TwitterAgent</pre>

4. **Modify the provided Flume configuration and copy it to /etc/flume-ng/conf**

   There is a file called `flume.conf` in the `flume-sources` directory, which needs some minor editing. There are four fields which need to be filled in with values from Twitter. The relevant information is available on the Details page for [your Twitter app](https://dev.twitter.com/apps). Fill in the consumer key, consumer secret, access token, and access token secret. The `keywords` parameter accepts a comma-separated list of keywords to use to filter tweets and collect a relevant set of data. If the parameter is not defined, the Twitter Sample API will be used to collect a sample of the entire Twitter Firehose.

   <pre>$ sudo cp flume.conf /etc/flume-ng/conf</pre>

Setting up Hive
----------------

1. **Build or Download the JSON SerDe**

   A pre-built version of the JSON SerDe is available [here](http://files.cloudera.com/samples/hive-serdes-1.0-SNAPSHOT.jar).

   The `hive-serdes` directory contains a Maven project with a JSON SerDe which enables Hive to query raw JSON data.

   To build the hive-serdes JAR, from the root of the git repository:

   <pre>
   $ cd hive-serdes    
   $ mvn package  
   $ cd ..  
   </pre>

   This will generate a file called `hive-serdes-1.0-SNAPSHOT.jar` in the `target` directory.

2. **Create the Hive directory hierarchy**

    <pre>
    $ sudo -u hdfs hadoop fs -mkdir /user/hive/warehouse   
    $ sudo -u hdfs hadoop fs -chown -R hive:hive /user/hive  
    $ sudo -u hdfs hadoop fs -chmod 750 /user/hive  
    $ sudo -u hdfs hadoop fs -chmod 770 /user/hive/warehouse  
    </pre>

    You'll also want to add whatever user you plan on executing Hive scripts with to the hive Unix group:

    <pre>$ sudo usermod -a -G hive &lt;username&gt;</pre>

3. **Configure the Hive metastore**

    The Hive metastore should be configured to use MySQL. Follow these [instructions](https://ccp.cloudera.com/display/CDH4DOC/Hive+Installation#HiveInstallation-ConfiguringtheHiveMetastore) to configure the metastore. Make sure to install the MySQL JDBC driver in `/usr/lib/hive/lib`.

4. **Create the tweets table**

    Run `hive`, and execute the following commands:

    <pre>
    ADD JAR &lt;path-to-hive-serdes-jar&gt;;
    
    CREATE EXTERNAL TABLE tweets (
      id BIGINT,
      created_at STRING,
      source STRING,
      favorited BOOLEAN,
      retweet_count INT,
      retweeted_status STRUCT&lt;
        text:STRING,
        user:STRUCT&lt;screen_name:STRING,name:STRING&gt;&gt;,
      entities STRUCT&lt;
        urls:ARRAY&lt;STRUCT&lt;expanded_url:STRING&gt;&gt;,
        user_mentions:ARRAY&lt;STRUCT&lt;screen_name:STRING,name:STRING&gt;&gt;,
        hashtags:ARRAY&lt;STRUCT&lt;text:STRING&gt;&gt;&gt;,
      text STRING,
      user STRUCT&lt;
        screen_name:STRING,
        name:STRING,
        friends_count:INT,
        followers_count:INT,
        statuses_count:INT,
        verified:BOOLEAN,
        utc_offset:INT,
        time_zone:STRING&gt;,
      in_reply_to_screen_name STRING
    ) 
    PARTITIONED BY (datehour INT)
    ROW FORMAT SERDE 'com.cloudera.hive.serde.JSONSerDe'
    LOCATION '/user/flume/tweets';</pre>

    The table can be modified to include other columns from the Twitter data, but they must have the same name, and structure as the JSON fields referenced in the [Twitter documentation](https://dev.twitter.com/docs/tweet-entities).

Prepare the Oozie workflow
--------------------------

1. **Configure Oozie to use MySQL**

    If using Cloudera Manager, Oozie can be reconfigured to use MySQL via the service configuration page on the Databases tab. Make sure to restart the Oozie service after reconfiguring. You will need to install the MySQL JDBC driver in `/usr/lib/oozie/libext`.

    If Oozie was installed manually, Cloudera provides [instructions](https://ccp.cloudera.com/display/CDH4DOC/Oozie+Installation#OozieInstallation-ConfiguringOozietoUseMySQL) for configuring Oozie to use MySQL.

2. **Create a lib directory and copy any necessary external JARs into it**

    External JARs are provided to Oozie through a `lib` directory in the workflow directory. The workflow will need a copy of the MySQL JDBC driver and the hive-serdes JAR.

    <pre>
    $ mkdir oozie-workflows/lib
    $ cp hive-serdes/target/hive-serdes-1.0-SNAPSHOT.jar oozie-workflows/lib
    $ cp /var/lib/oozie/mysql-connector-java.jar oozie-workflows/lib
    </pre>

3. **Copy hive-site.xml to the oozie-workflows directory**

    To execute the Hive action, Oozie needs a copy of `hive-site.xml`.

    <pre>
    $ sudo cp /etc/hive/conf/hive-site.xml oozie-workflows
    $ sudo chown &lt;username&gt;:&lt;username&gt; oozie-workflows/hive-site.xml
    </pre>

4. **Copy the oozie-workflows directory to HDFS**

    <pre>$ hadoop fs -put oozie-workflows /user/&lt;username&gt;/oozie-workflows</pre>

5. **Install the Oozie ShareLib in HDFS**

    <pre>
    $ sudo -u hdfs hadoop fs -mkdir /user/oozie
    $ sudo -u hdfs hadoop fs -chown oozie:oozie /user/oozie
    </pre>

    In order to use the Hive action, the Oozie ShareLib must be installed. Installation instructions can be found [here](https://ccp.cloudera.com/display/CDH4DOC/Oozie+Installation#OozieInstallation-InstallingtheOozieShareLibinHadoopHDFS).

Starting the data pipeline
------------------------

1. **Start the Flume agent**

    Create the HDFS directory hierarchy for the Flume sink. Make sure that it will be accessible by the user running the Oozie workflow.  
    
    Create 'flume' user in Hue. As a Hue superuser, go in the User Admin application and create a new user 'flume'.
    
    This can also be done on the command line:
    <pre>
    $ hadoop fs -mkdir /user/flume/tweets
    $ hadoop fs -chown -R flume:flume /user/flume
    $ hadoop fs -chmod -R 770 /user/flume
    </pre>

   Then start Flume:
   <pre>
    $ sudo /etc/init.d/flume-ng-agent start
  </pre>

2. **Adjust the start time of the Oozie coordinator workflow in job.properties**

    You will need to modify the `job.properties` file, and change the `jobStart`, `jobEnd`, and `initialDataset` parameters. The start and end times are in UTC, because the version of Oozie packaged in CDH4 does not yet support custom timezones for workflows. The initial dataset should be set to something before the actual start time of your job in your local time zone. Additionally, the `tzOffset` parameter should be set to the difference between the server's timezone and UTC. By default, it is set to -8, which is correct for US Pacific Time.

3. **Start the Oozie coordinator workflow**
    
    <pre>$ oozie job -oozie http://&lt;oozie-host&gt;:11000/oozie -config oozie-workflows/job.properties -run</pre>

