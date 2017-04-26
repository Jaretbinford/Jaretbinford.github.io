---
layout: post
comments: true
title: MySQL or Postgres Storage for Datomic
categories: datomic
---

In responding to a recent [Stack Overflow question](http://stackoverflow.com/questions/43422828/datomic-pro-bin-run-no-suitable-driver-found/43615059#43615059), 
 I ran into a few hitches getting MySQL and PostgreSQL working as backend storages.  As a reference, I followed [this section of the documentation](http://docs.datomic.com/storage.html#sec-5), to setup MySQL and PostgreSQL storage.

<center>
<img src="/images/mysql.jpg" alt="MySQL logo" style="width: 200px;" align="bottom"/>
<img src="/images/postgresql.png" alt="Postgresql logo" style="width: 200px;" align="top"/>
</center>
----

These snippits assume that you have Postgres and MySQL installed and that you are operating from the _Datomic root directory_ on your local machine.  

## Setup the SQL database (create the table and users)


#### Postgres:

{% highlight bash %}

psql -f bin/sql/postgres-db.sql -U postgres
psql -f bin/sql/postgres-table.sql -U postgres -d datomic
psql -f bin/sql/postgres-user.sql -U postgres -d datomic

{% endhighlight %}

#### MySQL:

{% highlight bash %}

mysql -u root < bin/sql/mysql-db.sql 
mysql -u root datomic < bin/sql/mysql-table.sql
mysql -u root < bin/sql/mysql-user.sql 

{% endhighlight %}

----
## Start the transactor


#### Command:

{% highlight bash %}
bin/transactor config/samples/sql-transactor-template.properties 
{% endhighlight %}

#### Postgres properties file:

{% highlight vim %}
protocol=sql
host=localhost
port=4334
sql-url=jdbc:postgresql://localhost:3306/datomic
sql-user=datomic
sql-password=datomic
sql-driver-class=org.postgresql.Driver
{% endhighlight %}


#### MySQL properties file:

{% highlight vim %}
protocol=sql
sql-url=jdbc:mysql://localhost:3306/datomic
sql-user=datomic
sql-password=datomic
sql-driver-class=com.mysql.jdbc.Driver
{% endhighlight %}

----

With the addition of the Client library in _0.9.5530_ you can start a peer server to serve access to databases for the client library.  If you're going to do so on SQL storage you'll need to ensure the database is created before standing up the peer-server.  To do so launch a peer against the transactor (in this case I ran _bin/repl_ from the _Datomic root directory_ and run the following:

#### MySQL:

{% highlight clojure %}
(require '[datomic.api :as d])
(def uri "datomic:sql://test?jdbc:mysql://localhost:3306/datomic?user=datomic&password=datomic")
(d/create-database uri)
{% endhighlight %}

#### Postgres

{% highlight clojure %}
(require '[datomic.api :as d])
(def uri "datomic:sql://test?jdbc:postgresql://localhost:3306/datomic?user=datomic&password=datomic")
(d/create-database uri)
{% endhighlight %} 

Now that you have your database created you can pass in the "test" db name to the peer server connection string:

{% highlight bash %}
./bin/run -m datomic.peer-server -h localhost -p 8998 -a myaccesskey,mysecret -d demo,"datomic:sql://test?jdbc:mysql://localhost:3306/datomic?user=datomic&password=datomic"
{% endhighlight %}

>Please note that when launching your peer-server, you'll need to ensure you have the proper credentials (username and password) as configured in MySQL or PostgreSQL.

Let me know if you have any questions on setting up MySQL or PostgreSQL as your underlying storage for Datomic.

Cheers,
Jaret
