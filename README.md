# MapR DB 6.0 Beta : Getting Started

## Introduction

This getting started will show you how to work with MapR-DB JSON using:

* MapR-DB Shell to interact with JSON tables from the command line
* Apache Drill to so some analytics with SQL
* Java and the OJAI library to build operational application.

You will learn how to optimize query using MapR-DB Secondary indexes.

**Prerequisites**

* MapR Converged Data Platform 6.0 Beta with Apache Drill
* JDK 8
* Maven 3.x



## Importing Data into MapR JSON Tables

The application use the Yelp Academy Dataset that contains a list of Business and User Reviews.

#### 1- Download the Yelp Dataset from [here](https://www.yelp.com/dataset_challenge)

#### 2- Copy the dataset to MapR Cluster

```
scp yelp_dataset_challenge_round9.tar mapr@mapr60:/mapr/my.cluster.com/tmp/
```

* `mapr60` is one of the node of your cluster
* `my.cluster.com` is the name of your cluster

#### 3- Unarchive the dataset

Connect to one of the node of your cluster using `mapr` user

```
$ cd /mapr/my.cluster.com/tmp/

$ tar xvf yelp_dataset_challenge_round9.tar
```

#### 4- Import the data into MapR DB Tables

Create new tables using maprcli tool

```
$ maprcli table create -path /apps/business -tabletype json

$ maprcli table create -path /apps/review -tabletype json

$ maprcli table create -path /apps/user -tabletype json  
```



#### 5- Import the JSON documents into MapR tables

```

$ mapr importJSON -idField business_id -src /tmp/yelp_academic_dataset_business.json -dst /apps/business -mapreduce false

$ mapr importJSON -idField review_id -src /tmp/yelp_academic_dataset_review.json -dst /apps/review -mapreduce false

$ mapr importJSON -idField user_id -src /tmp/yelp_academic_dataset_user.json -dst /apps/user -mapreduce false

```

You have now 3 JSON tables, let's query these tables using SQL with Apache Drill.


## MapR DB Shell

In this section you will learn how to use DB shell to query JSON table but also how to update a document with a new field.

As mapr user in a terminal enter the following commands to get familiar with the shell
```
$ mapr dbshell

maprdb mapr:> jsonoptions --pretty true --withtags false

maprdb mapr:> find /apps/user --limit 2

```

To learn more about the various commands, run `help`' or  help command , for example `help insert`.


**1- Retrieve one document using its id**

```
maprdb mapr:> find /apps/user --id l52TR2e4p9K4Z9zezyGKfg
```

**2- Add a projection to limit the number of fields**
```
maprdb mapr:> find /apps/user --id l52TR2e4p9K4Z9zezyGKfg --f name,average_stars,useful
```

**3- Insert a new Document**
```
maprdb mapr:> insert /apps/user --value '{"_id":"pdavis000222333", "name":"PDavis", "type" : "user", "yelping_since" : "2016-09-22", "fans" : 0, "support":"gold"}'
```

**4- Query document with Condition**
```
maprdb mapr:> find /apps/user --where '{ "$eq" : {"support":"gold"} }' --f _id,name,support
```

You will see later how to improve query performance with indices.


**5- Update a document**

Let's add the support status "gold" to an existing user, and find all users with support equals to gold

```
maprdb mapr:> update /apps/user --id l52TR2e4p9K4Z9zezyGKfg --m '{ "$set" : [ {"support" : "gold" } ] }'


maprdb mapr:> find /apps/user --where '{ "$eq" : {"support":"gold"} }' --f _id,name,support 

```

**6- Adding index to improve queries**

Let's now add indices to the user table.

In a terminal window:

```
$ maprcli table index add -path /apps/user -index idx_support -indexedfields '"support":1'
```


In MapR-DB Shell, find all users with support equals to gold, and compare with previous query performance:

```
maprdb mapr:> find /apps/user --where '{ "$eq" : {"support":"gold"} }' --f _id,name,support

```

**7- Using MapR DB Shell Query Syntax**

MapR-DB Shell has a complete JSON syntax to express the query including projection, condition, orderby, limit, skip. For example 

```
maprdb mapr:> find /apps/user --q ' { "$select" : ["_id","name","support"]  ,"$where" : {"$eq" : {"support":"gold"}}, "$limit" : 2 }'
```

Now that you are familiar with MapR-DB Shell, and you have seen the impact on index on query performance, let's now use Apache Drill to run some queries.

## Querying the data using Drill

Open a terminal and run `sqlline`:

```
$ sqlline

sqlline> !connect jdbc:drill:zk=mapr60:5181
```

Where `mapr60` is a node where Zookeeper is running, then enter the mapr username and password when prompted.


You can execute the same query that you have used in the previous example using a simple SQL statement:

```sql
sqlline> select _id, name, support from dfs.`/apps/user` where support = 'gold';
```

Let's now use another table and query to learn more about MapR-DB JSON Secondary Index capabilities.

```sql
sqlline> select name, stars from dfs.`/apps/business` where stars = 5 order by name limit 10;
```

Execute the query multiple times and looks at the execution time.

**Simple Index**

You can now improve the performance of this query using an index. In a new terminal window run the following command to create an index on the `stars` field sorted in descending order (`-1`).

```
$ maprcli table index add -path /apps/business -index idx_stars -indexedfields '"stars":-1'
```

If you execute the query, multiple times, now it should be faster, since the index is used.

In this case Apache Drill is using the index to find all the stars equal to 5, then retrieve the name of the business from the document itseld (in the `/apps/user` table).

You can use the `EXPLAIN PLAN` command to see how drill is executing the query, is sqlline:

```
sqlline> explain plan for select name, stars from dfs.`/apps/business` where stars = 5;
```

The result of the explain plan will be a JSON document explaining the query plan, you can look in the `scanSpec` attribute that use the index if present. You can look at the following attributes in the `scanSpec` attribute:

* `secondaryIndex` equals true when a secondary index is created
* `startRow` and `stopRow` that show the index key used by the query
* `indexName` set to `idx_stars` that is the index used by the query


If you want to delete the index to compare the execution plan run the followind command as `mapr` in a terminal:

```
$ maprcli table index remove -path /apps/business -index idx_stars  
```

**Covering Queries**

In the index you have created in the previous step, we have specific the `-indexedfields` parameter to set the index key with the `stars` values. It is also possible to add some non indexed field to the index using the `-includedfields`. 

In this case the index includes some other fields that could be used by applications to allow the query to only access the index without having to retrieve any value from the JSON table itself.

Let's recreate the index on `stars` field and add the `name` to the list of included fields.

```
# if you have not deleted the index yet
$ maprcli table index remove -path /apps/business -index idx_stars  

$ maprcli table index add -path /apps/business -index idx_stars -indexedfields '"stars":-1' -includedfields '"name"'
```

If you execute the query, multiple times, now it should be even faster than previous queries since Drill is doing a covered queries, only accessing the index.

In sqlline, run the explain plan with the new index:

```
sqlline> explain plan for select name, stars from dfs.`/apps/business` where stars = 5;
```


The main change in the explain plan compare to previous queries is:

* `coveredFields` attribute that show the fields used by the queries.



## Working with OJAI

OJAI the Java API used to access MapR-DB JSON, leverages the same query engine than MapR-DB Shell and Apache Drill. The various examples use the `/apps/user` user table.

* `OJAI_001_YelpSimpleQuery` : return all the users with the `support` field equals to "gold". Note that the `support` field is a new field you are adding in the demonstration
* `OJAI_002_YelpInsertAndQuery` : insert a new user in the table, with support set to gold and do the query. The query will not necessary returns the value from the index since it is indexed ascynchronously (see next example)
* `OJAI_003_YelpInsertAndQueryRYOW` : insert a new user in the table, and do the query. This example use the "Tracking Writes" to be sure the query waits for the index to be updated
* `OJAI_004_YelpQueryAndDelete` : delete the documents created in example 002 and 003.

#### Run the sample application

**Build and the project**

```
$ mvn clean package
```

If you are not on the cluster, copy the generated jar on your cluster for example:

```
$ scp target/mapr-db-getting-started-1.0-SNAPSHOT.jar mapr@mapr60:/home/mapr/
```

**Run the application**

Connected to your cluster where you have deployed the `mapr-db-getting-started-1.0-SNAPSHOT.jar` run the following command:

```
$ java -cp /home/mapr/mapr-db-getting-started-1.0-SNAPSHOT.jar:com.mapr.db.samples.OJAI_001_YelpSimpleQuery
```

Change the name of the application to test the various examples.


## Conclusion

In this example you have learned how to:

* use MapR-DB Shell to insert, update and query MapR-DB JSON Tables
* add an index to improve queries performances
* use Drill to query JSON Tables and see the impact of indexes
* develop Java application with OJAI to query documents, and the impact of indexes on performance
* track you writes to be sure the index is updated when you query the table and a document has been inserted/updated


You can also looked at the following examples:

* [Ojai 2.0 Examples](https://github.com/mapr-demos/ojai-2-examples) to learn more about OJAI 2.0 features
* [MapR-DB Change Data Capture](https://github.com/mapr-demos/mapr-db-cdc-sample) to capture database events such as insert, update, delete and react to this events.





