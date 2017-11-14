---
layout: post
comments: true
title: Merge two attributes
categories: datomic
---


Let's pretend our database has recorded two attributes for a user.  The user's e-mail and the users username.
The username is required to be the e-mail for the user.  Due to a design decision, e-mails will no longer
be recorded.  As such we need to depricate the e-mail attribute and merge it to our username attribute to ensure
all e-mails represent usernames.

The steps to this process are as follows:

1. Rename the old attribute (i.e. :user/email to :user/email-deprecated)
2. Make the new attribute (:user/username)
3. Migrate values from the old attribute to the new attribute

   It should be noted that if you have a lot of data to merge you will want to appropriately batch the
   merge transactions.  This is also not a solution for bad schema design and it should not be relied upon
   to correct what are in reality schema design problems.  d/history will still point to the previous entry
    and :db/ident is not t-aware.

Please see Stu Halloways blog post http://blog.datomic.com/2017/01/the-ten-rules-of-schema-growth.html

## Show me how to merge babay

<center>
<img src="/images/merge.png" alt="mergebaby" style="width: 200px;" align="bottom"/>
</center>

{% highlight clojure %}
      (ns merge.core
       (:import datomic.Util)
         (require [datomic.api :as d]
                    [clojure.java.io :as io]))
     ;;Create the DB
     (def db-uri "datomic:dev://localhost:4334/merge")
     (d/create-database db-uri)
     (def conn (d/connect db-uri))
{% endhighlight %}

slurp in the schema.edn file which contains:

{% highlight clojure %}
     ; [{:db/ident :user/email,
     ; :db/valueType :db.type/string,
     ; :db/cardinality :db.cardinality/one,
     ; :db/doc "E-mail of user",
     ; :db.install/_attribute :db.part/db,
     ; :db/id #db/id[:db.part/db 250]}
     ;{:db/ident :user/username,
     ; :db/valueType :db.type/string,
     ; :db/cardinality :db.cardinality/one,
     ; :db/doc "Username of user",
     ; :db.install/_attribute :db.part/db,
     ; :db/id #db/id[:db.part/db 251]}]

     (def schema-tx (read-string (slurp "/Users/jbin/Desktop/Jaret/Projects/workproof/merge/resources/schema.edn")))
     @(d/transact conn schema-tx)
{% endhighlight %}

Pull the attributes I have inserted.

{% highlight clojure %}
     (d/pull (d/db conn) '[*] 250)
     (d/pull (d/db conn) '[*] 251)
{% endhighlight %}

Transact some data against e-mails and usernames.  We will then merge this data. Here are the contents of Data.edn which will be slurped

{% highlight clojure %}
     ;;[{:db/id #db/id[:db.part/user -1000001], :user/email "user1@gmail.com"}
     ;; {:db/id #db/id[:db.part/user -1000002], :user/username "JBIN0204"}
     ;; {:db/id #db/id[:db.part/user -1000003], :user/email "user2@gmail.com"}
     ;; {:db/id #db/id[:db.part/user -1000004], :user/username "Datomicfan"}
     ;; {:db/id #db/id[:db.part/user -1000005], :user/email "user3@gmail.com"}
     ;; {:db/id #db/id[:db.part/user -1000006], :user/username "Parenpower"}
     ;; {:db/id #db/id[:db.part/user -1000007], :user/email "user4@gmail.com"}
     ;; {:db/id #db/id[:db.part/user -1000008], :user/email "user5@gmail.com"}]


     (def test-data-tx (read-string (slurp "/Users/jbin/Desktop/Jaret/Projects/workproof/merge/resources/Data.edn")))
     @(d/transact conn test-data-tx)
{% endhighlight %}

{% highlight clojure %}
     ;;create alter email-depricated
     (def db (d/db conn))
     (def schema [{:db/id #db/id [:db.part/db 250],
                   :db/ident :user/email-deprecated,
                                 :db.alter/_attribute :db.part/db}])
     @(d/transact conn schema)
{% endhighlight %}

{% highlight clojure %}
     ;;query all the emails and IDs
     (d/q '[:find ?e ?email
               :where [?e :user/email-deprecated ?email]] (d/db conn))


     ;; Move all e-mail-depricated to the username attribute
     (def db (d/db conn))
     (defn merge-all
       "Merge all mapped results"
         [db]
             (->> (d/q '[:find ?e ?email
                             :where
                                             [?e :user/email-deprecated ?email]]
                                                           db)
                     (mapcat (fn[[e email]]
                                    [[:db/add e :user/username email]
                                                   [:db/retract e :user/email-deprecated email]]))
                              ))

     ;;Call the function to merge-all
     (merge-all db)


     @(d/transact conn (merge-all db))

     ;; Review all usernames and note they are now merged under one attribute!
     (def db (d/db conn) )
     (d/q '[:find ?e ?email
            :where [?e :user/username ?email]] (d/db conn))
{% endhighlight %}

And there you have it, you've merged all the usernames under one attribute!!!!

## Fulltext happens

This process can be applied when needing to implement :db/fulltext. But instead of altering existing
schema we need to add a new attribute and transact with fulltext enabled.  We will then want to pour in/import our data and use the new attribute going forward

{% highlight clojure %}
     ;;Create the new username with fulltext attribute
     (def db (d/db conn))
     (def schema [{:db/ident :user/usernamefulltext,
                   :db/valueType :db.type/string,
                                 :db/fulltext true,
                                               :db/cardinality :db.cardinality/one,
                                                             :db/doc "username with fulltext search",
                                                                           :db.install/_attribute :db.part/db,
                                                                                         :db/id #db/id[:db.part/db 252]}])

     @(d/transact conn schema)

     (d/pull (d/db conn) '[*] 252)

     ;;Next transact all usernames to the new usernamefulltext and retract the previous usernames:
     (defn merge-all
       "Merge all mapped results"
         [db]
           (->> (d/q '[:find ?e ?email
                         :where
                                       [?e :user/username ?email]]
                                                   db)
                  (mapcat (fn[[e email]]
                                   [[:db/add e :user/usernamefulltext email]
                                                     [:db/retract e :user/username email]]))
                         ))
     ;;Call the function to merge-all and review
     (merge-all db)

     ;;transact it

     @(d/transact conn (merge-all db))

     (def db (d/db conn) )
     (d/q '[:find ?e ?email
            :where [?e :user/usernamefulltext ?email]] (d/db conn))


{% endhighlight %}


Cheers,

Jaret


