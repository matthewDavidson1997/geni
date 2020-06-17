<img src="logo/geni.png" width="250px">

[![Continuous Integration](https://github.com/zero-one-group/geni/workflows/Continuous%20Integration/badge.svg?branch=develop)](https://github.com/zero-one-group/geni/commits/develop)
[![Code Coverage](https://codecov.io/gh/zero-one-group/geni/branch/develop/graph/badge.svg)](https://codecov.io/gh/zero-one-group/geni)
[![Clojars Project](https://img.shields.io/clojars/v/zero.one/geni.svg)](http://clojars.org/zero.one/geni)

WARNING! This library is still unstable. Some information here may be outdated. Do not use it in production just yet!

See [Flambo](https://github.com/sorenmacbeth/flambo) and [Sparkling](https://github.com/gorillalabs/sparkling) for more mature alternatives.

# Introduction

`geni` (*/gɜni/* or "gurney" without the r) is a Clojure library that wraps Apache Spark. The name comes from the Javanese word for fire.

# Why?

Clojure is particularly well-suited for data wrangling due to its particular focus on fast feedback - most notably through its REPL. Being hosted on the JVM, Clojure interops well with Java (and, by extension, Scala) libaries. However, Spark's pleasant API in Scala becomes quite unidiomatic in Clojure. Geni aims to provide an ergonomic Spark interface for the Clojure REPL.

Geni allows us to write queries without making sure that the types line up:

```clojure
(-> dataframe                    ;; Idiomatic threading macro as the main interface
    (group-by (lower "SellerG")  ;; Dynamism with mixed Column, string and keyword types
              "Suburb"
              :Regionname)
    (agg {:mean (mean "Price")   ;; No need for into-array to handle interop
          :std  (stddev "Price")
          :min  (min "Price")
          :max  (max "Price")})
    show)
```

In contrast, you would have to write the following instead with pure interop:

```clojure
(-> dataframe
    (.groupBy (into-array Column [(functions/lower (functions/col "SellerG"))
                                  (functions/col "Suburb")
                                  (functions/col "Regionname")]))
    (.agg
      (.as (functions/mean "Price") "mean")
      (into-array Column [(.as (functions/stddev "Price") "std")
                          (.as (functions/min "Price") "min")
                          (.as (functions/max "Price") "max")]))
    .show)
```

Instead of having to deal with interop:

```clojure
(->> (.collect dataframe) ;; .collect returns an array of Spark rows
     (map
       #(JavaConversions/seqAsJavaList
       (.. % toSeq))))    ;; returns a seq of seqs
                          ;; must zip into map to recover row-like maps
```

Geni's `(collect dataframe)` returns a seq of maps.

Finally, Geni supports various Clojure (or Lisp) idioms by making some functions variadic (`+`, `<=`, `&&`, etc.) and providing functions with Clojure analogues that are not available in Spark such as `remove`. For example:

```clojure
(-> melbourne-df
    (remove (like :Regionname "%Metropolitan%"))
    (filter (&& (< 2 :Rooms 5)
                (< 5e5 :Price 6e5)
                (< :YearBuilt 2010)))
    (select :Regionname :Rooms :Price :YearBuilt)
    show)
```

# Examples

Spark SQL API for grouping and aggregating:

```clojure
(require '[zero-one.geni.core :as g])

(-> melbourne-df
    (g/group-by :Suburb)
    g/count
    (g/order-by (g/desc :count))
    (g/limit 5)
    g/show)
; +--------------+---+
; |Suburb        |n  |
; +--------------+---+
; |Reservoir     |359|
; |Richmond      |260|
; |Bentleigh East|249|
; |Preston       |239|
; |Brunswick     |222|
; +--------------+---+
```

MLlib's pipeline:

```clojure
(require '[zero-one.geni.core :as g])
(require '[zero-one.geni.ml :as ml])

(def training-set
  (g/table->dataset
    spark
    [[0 "a b c d e spark"  1.0]
     [1 "b d"              0.0]
     [2 "spark f g h"      1.0]
     [3 "hadoop mapreduce" 0.0]]
    [:id :text :label]))

(def pipeline
  (ml/pipeline
    (ml/tokenizer {:input-col "text"
                   :output-col "words"})
    (ml/hashing-tf {:num-features 1000
                    :input-col "words"
                    :output-col "features"})
    (ml/logistic-regression {:max-iter 10
                             :reg-param 0.001})))

(def model (ml/fit training-set pipeline))

(def test-set
  (g/table->dataset
    spark
    [[4 "spark i j k"]
     [5 "l m n"]
     [6 "spark hadoop spark"]
     [7 "apache hadoop"]]
    [:id :text]))

(-> test-set
    (ml/transform model)
    (g/select :id :text :probability :prediction)
    g/show)
;; +---+------------------+----------------------------------------+----------+
;; |id |text              |probability                             |prediction|
;; +---+------------------+----------------------------------------+----------+
;; |4  |spark i j k       |[0.1596407738787411,0.8403592261212589] |1.0       |
;; |5  |l m n             |[0.8378325685476612,0.16216743145233883]|0.0       |
;; |6  |spark hadoop spark|[0.0692663313297627,0.9307336686702373] |1.0       |
;; |7  |apache hadoop     |[0.9821575333444208,0.01784246665557917]|0.0       |
;; +---+------------------+----------------------------------------+----------+
```

More detailed examples can be found [here](examples/README.md).

There is also a one-to-one walkthrough of Chapter 5 of NVIDIA's [Accelerating Apache Spark 3.x](https://www.nvidia.com/en-us/deep-learning-ai/solutions/data-science/apache-spark-3/ebook-sign-up/), which can be found [here](examples/nvidia_pipeline.clj).

# Geni Semantics

## Column Coercion

Many SQL functions and Column methods are overloaded to take either a keyword, a string or a Column instance as argument. For such cases, Geni implements Column coercion where

1. Column instances are left as they are,
2. strings and keywords are interpreted as column names and;
3. other values are interpreted as a literal Column.

Because of this, basic arithmetic operations do not require `lit` wrapping:

```clojure
; The following two expressions are equivalent
(g/- (g// (g/sin Math/PI) (g/cos Math/PI)) (g/tan Math/PI))
(g/- (g// (g/sin (g/lit Math/PI)) (g/cos (g/lit Math/PI))) (g/tan (g/lit Math/PI)))
```

However, string literals do require `lit` wrapping:

```clojure
; The following fails, because "Nelson" is interpreted as a Column
(-> dataframe (g/filter (g/=== "SellerG" "Nelson")))

; The following works, as it checks the column "SellerG" against "Nelson" as a literal
(-> dataframe (g/filter (g/=== "SellerG" (g/lit "Nelson"))))
```

## Dataset Creation: ArrayType vs. VectorType

Inspired by Pandas' flexible DataFrame creation, Geni provides three main ways to create Spark Datasets:

```clojure
; The following three expressions are equivalent
(g/table->dataset spark
                  [[1 "x"]
                   [2 "y"]
                   [3 "z"]]
                  [:a :b])
(g/map->dataset spark {:a [1 2 3] :b ["x" "y" "z"]})
(g/records->dataset spark [{:a 1 :b "x"}
                           {:a 2 :b "y"}
                           {:a 3 :b "z"}])
```

It it sometimes convenient to be able to create a Spark vector column, which is different to SQL array columns. For that reason, Geni provides an easy way to create vector columns, but it comes with a potential gotcha. A vector of numbers is interpreted as a Spark vector, but any list is always interpreted as a SQL array:

```clojure
(g/print-schema
  (g/table->dataset spark
                    [[0.0 [0.5 10.0]]
                     [1.0 [1.5 30.0]]]
                    [:label :features]))
; root
;  |-- label: double (nullable = true)
;  |-- features: vector (nullable = true)

(g/print-schema
  (g/table->dataset spark
                    [[0.0 '(0.5 10.0)]
                     [1.0 '(1.5 30.0)]]
                    [:label :features]))
; root
;  |-- label: double (nullable = true)
;  |-- features: array (nullable = true)
;  |    |-- element: double (containsNull = true)
```

# Quick Start

Use [Leiningen](http://leiningen.org/) to create a template of a Geni project:

```bash
lein new geni <project-name>
```

Step into the directory, and run the command `lein run`!

# Installation

Note that `geni` wraps Apache Spark 2.4.5, which uses Scala 2.12, which has [incomplete support for JDK 11](https://docs.scala-lang.org/overviews/jdk-compatibility/overview.html). JDK 8 is recommended.

Add the following to your `project.clj` dependency:

[![Clojars Project](https://clojars.org/zero.one/geni/latest-version.svg)](http://clojars.org/zero.one/geni)

You would also need to add Spark as provided dependencies. For instance, have the following key-value pair for the `:profiles` map:

```clojure
:provided
{:dependencies [;; Spark
                [org.apache.spark/spark-core_2.12 "2.4.6"]
                [org.apache.spark/spark-hive_2.12 "2.4.6"]
                [org.apache.spark/spark-mllib_2.12 "2.4.6"]
                [org.apache.spark/spark-sql_2.12 "2.4.6"]
                [org.apache.spark/spark-streaming_2.12 "2.4.6"]
                ;; Optional: Spark XGBoost
                [ml.dmlc/xgboost4j-spark_2.12 "1.0.0"]
                [ml.dmlc/xgboost4j_2.12 "1.0.0"]
                ;; Optional: Google Sheets Integration
                [com.google.api-client/google-api-client "1.30.9"]
                [com.google.apis/google-api-services-drive "v3-rev197-1.25.0"]
                [com.google.apis/google-api-services-sheets "v4-rev612-1.25.0"]
                [com.google.oauth-client/google-oauth-client-jetty "1.30.6"]
                [org.apache.hadoop/hadoop-client "2.7.3"]]}
```

# Future Work

Features:
- Data-oriented queries and pipeline stages.
- Setup on GCP's Dataproc + guide.
- Clojure docs.

# License

Copyright 2020 Zero One Group.

geni is licensed under Apache License v2.0.

# Mentions

Some code was taken from:

* [finagle-clojure](https://github.com/finagle/finagle-clojure) - especially in terms of Scala interop.
* [LispCast](https://lispcast.com/) for [exponential backoff](https://lispcast.com/exponential-backoff/).
