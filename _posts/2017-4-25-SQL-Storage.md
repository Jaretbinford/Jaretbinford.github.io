---
layout: post
comments: true
title: MySQL or Postgres Storage for Datomic
categories: datomic
---

In responding to a recent [Stack Overflow question](http://stackoverflow.com/questions/43422828/datomic-pro-bin-run-no-suitable-driver-found/43615059#43615059), 
 I found that the basic steps have not been spelled out sufficiently.  While we have [this section of the documentation](http://docs.datomic.com/storage.html#sec-5), the steps are not clear for setting up SQL based storage and deving on your local machine.

---
Setup the SQL database (create the table and users)
---

>Postgres:

{% highlight bash %}

psql -f bin/sql/postgres-db.sql -U postgres
psql -f bin/sql/postgres-table.sql -U postgres -d datomic
psql -f bin/sql/postgres-user.sql -U postgres -d datomic

{% endhighlight %}

>MySQL:

{% highlight bash %}

mysql -u root < bin/sql/mysql-db.sql 
mysql -u root datomic < bin/sql/mysql-table.sql
mysql -u root < bin/sql/mysql-user.sql 

{% endhighlight %}

---
Start the transactor
---

>Command:

{% highlight bash %}
bin/transactor config/samples/sql-transactor-template.properties 
{% endhighlight %}

>Postgres properties file:

{% highlight vim %}
protocol=sql
host=localhost
port=4334
sql-url=jdbc:postgresql://localhost:3306/datomic
sql-user=datomic
sql-password=datomic
sql-driver-class=org.postgresql.Driver
{% endhighlight %}


>MySQL properties file:

{% highlight vim %}
protocol=sql
sql-url=jdbc:mysql://localhost:3306/datomic
sql-user=datomic
sql-password=datomic
sql-driver-class=com.mysql.jdbc.Driver
{% endhighlight %}
