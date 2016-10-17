# carmine-sentinel

A Clojure library designed to connect redis by [sentinel](redis.io/topics/sentinel), make [carmine](https://github.com/ptaoussanis/carmine) to support [sentinel](redis.io/topics/sentinel)。

## Usage

```clojure
[net.fnil/carmine-sentinel "0.1.0-RC1"]
```

It's a beta release, you can try it.Feedback is welcome.

First, require carmine and carmine-sentinel:

```clojure
(ns my-app
  (:require [taoensso.carmine :as car]
            [carmine-sentinel.core :as cs :refer [set-sentinel-groups!]]))
```

The only difference compares with carmine is that we will use `carmine-sentinel.core/wcar` to replace `taoensso.carmine/wcar` and add a new function `set-sentinel-groups!`.

Second, configure sentinel groups:

```clojure
(set-sentinel-groups! 
  {:group1 
   {:specs [{:host "127.0.0.1" :port 5000} {:host "127.0.0.1" :port 5001} {:host "127.0.0.1" :port 5002}] 
    :pool  {<opts>} }})
```

There is only one group named `:group1` above, and it has three sentinel instances (port from 5000 to 5002 at 127.0.0.1). Optional, you can set the pool option values and add more sentinel groups.

Next, we can define the `wcar*`:

```clojure
(def server1-conn {:pool {<opts>} :spec {} :sentinel-group :group1 :master-name "mymaster"})
(defmacro wcar* [& body] `(cs/wcar server1-conn ~@body))
```

The spec in `server1-conn` is empty, and there are two new options in server1-conn:

* `:sentinel-group` Which sentinel instances group to resolve master addr.Here is `:group1`.
* `:master-name` Master name configured in that sentinel group.Here is `mymaster`.

The `spec` in server1-conn will be merged to resolved master spec at runtime.So you can set `:password`,`:timeout-ms` etc. other options in it.

Also, you can define many `wcar*`-like macros to use other sentinel group and master name.

At last, you can use `wcar*` as the same in carmine.

```clojure
(wcar* (car/set "key" 1))
(wcar* (car/get "key"))
```

## Pub/Sub, MessageQueue and Lock

You have to invoke `update-conn-spec` before using other APIs in carmine:

```clojure
(def server1-conn {:pool {<opts>} :spec {} :sentinel-group :group1 :master-name "mymaster"})

;;Pub/Sub
(def listener
  (car/with-new-pubsub-listener (:spec (cs/update-conn-spec server1-conn))
    {"foobar" (fn f1 [msg] (println "Channel match: " msg))
     "foo*"   (fn f2 [msg] (println "Pattern match: " msg))}
   (car/subscribe  "foobar" "foobaz")
   (car/psubscribe "foo*")))

;;Message queue
(def my-worker
  (car-mq/worker (cs/update-conn-spec server1-conn) "my-queue"
   {:handler (fn [{:keys [message attempt]}]
               (println "Received" message)
               {:status :success})}))
   

;;;Lock
(locks/with-lock (cs/update-conn-spec server1-conn) "my-lock"
  1000 ; Time to hold lock
  500  ; Time to wait (block) for lock acquisition
  (println "This was printed under lock!"))  
```

## Reading From Slaves

If you want to read data from slave, you can set `prefer-slave?` to be true:

```clojure
(def slave-conn {:pool {<opts>} :spec {} 
                 :sentinel-group :group1 :master-name "mymaster" 
                 :prefer-slave? true})

(defmacro wcars* [& body] `(cs/wcar slave-conn ~@body))

(wcars* (car/set "key" 1)) ;; ExceptionInfo READONLY You can't write against a read only slave
```

If you have many slaves for one master, the default balancer is `first` function, but you can custom it by `slaves-balancer`,
for example, using random strategy:

```clojure
(def slave-conn {:pool {<opts>} :spec {}
                 :sentinel-group :group1
                 :master-name "mymaster"
                 :prefer-slave? true
                 :slaves-balancer rand-nth)
```

## Listen on switching master

You can register a listener to listen on switching master event:

```clojure
(cs/register-listener! (fn [e] (println "Event " e " happens")))
```

## Failover

At startup, carmine-sentinel will connect the first sentinel instance to resolve the master address, if if fails, carmine-sentinel will try the next sentinel until find a resolved master address or throw an exception.The resolved addr will be cached in memory.

And Carmine-sentinel subcribes `+switch-master` channel in sentinel.When the master redis instance is down, sentinel will publish a `+switch-master` message, while carmine-sentinel receives this message, it will clean the last cached result and try to connect the new redis master at once.

At last, carmine-sentinel will refresh the sentinel instance list by the response from command `SENTINEL sentinels [name]`.

## API docs

* [Carmine-sentinel APIs](http://fnil.net/docs/carmine_sentinel/)

## License

Copyright © 2016 killme2008

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
