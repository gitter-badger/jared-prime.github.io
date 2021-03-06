---
layout: post
title: "Concatenating Statistical Factets"
date: 2013-05-07
categories: ElasticSearch
preview: a quick tip about collecting summary data with ElasticSearch
---

*TL;DR* - When running multiple statistical facets with the same name, ElasticSearch will catenate the data from all fields in running the results. Using Ruby's Tire wrapper on ElasticSearch, only the last named facet will be sent to ElasticSearch.

*The data in these examples are trivial, but do a clear job illustrating this unintended 'feature'.*

##Example 1

    Sales.search do 
      query { all }
      facet 'stats' do
        statistical :amount
      end

      puts to_curl
    end

    curl -X GET 'http://localhost:9200/sales/sale/_search?size=10&pretty' -d '{"query":{"match_all":{}},"facets":{"stats":{"statistical":{"field":"amount"}}},"size":10}'

    @facets={"stats"=>{"_type"=>"statistical",
      "count"=>1476,
      "total"=>865389.1399999999, 
      "min"=>1.0, "max"=>9289.0, 
      "mean"=>586.3070054200541, 
      "sum_of_squares"=>1252317429.8326, 
      "variance"=>504697.63864238764, 
      "std_deviation"=>710.4207476153745}}

##Example 2

    Sales.search do 
      query { all }
      facet 'stats' do
        statistical :id
      end

      puts to_curl
    end

    curl -X GET 'http://localhost:9200/sales/sale/_search?size=10&pretty' -d '{"query":{"match_all":{}},"facets":{"stats":{"statistical":{"field":"id"}}},"size":10}'
 
    @facets={"stats"=>{"_type"=>"statistical", 
      "count"=>20, 
      "total"=>34.0, 
      "min"=>1.0, 
      "max"=>4.0, 
      "mean"=>1.7, 
      "sum_of_squares"=>78.0, 
      "variance"=>1.0100000000000002, 
      "std_deviation"=>1.0049875621120892}}

##Example 3

    Sales.search do 
      query { all }
      facet 'stats' do
        statistical :id
      end
      facet 'stats' do
        statistical :amount
      end

      puts to_curl
    end

    curl -X GET 'http://localhost:9200/sales/sale/_search?size=10&pretty' -d '{"query":{"match_all":{}},"facets":{"stats":{"statistical":{"field":"amount"}}},"size":10}'

    @facets={"stats"=>{"_type"=>"statistical",
      "count"=>1476, 
      "total"=>865389.1399999999, 
      "min"=>1.0,
      "max"=>9289.0,
      "mean"=>586.3070054200541,
      "sum_of_squares"=>1252317429.8326,
      "variance"=>504697.63864238764,
      "std_deviation"=>710.4207476153745}}

*Notice above that Tire ignored the first facet and only sent the second.*

##Example 4

*Handwritten curl request, no Tire wrapper*

    curl -X GET 'http://localhost:9200/sales/sale/_search?size=10&pretty' -d '{"query":{"match_all":{}},"facets":{"stats":{"statistical":{"field":"amount"}},"stats":{"statistical":{"field":"id"}}},"size":10}'

    "facets" : {
      "stats" : {
        "_type" : "statistical",
        "count" : 1496,
        "total" : 865423.1399999999,
        "min" : 1.0,
        "max" : 9289.0,
        "mean" : 578.4914037433155,
        "sum_of_squares" : 1.2523175078326E9,
        "variance" : 502458.32937302964,
        "std_deviation" : 708.8429511344735
      },
      "stats" : {
        "_type" : "statistical",
        "count" : 1496,
        "total" : 865423.1399999999,
        "min" : 1.0,
        "max" : 9289.0,
        "mean" : 578.4914037433155,
        "sum_of_squares" : 1.2523175078326E9,
        "variance" : 502458.32937302964,
        "std_deviation" : 708.8429511344735
      }
    }

*Here, both facets are registered, and both contain information from both fields.*

see: http://elasticsearch-users.115913.n3.nabble.com/Statistical-facet-on-multiple-fields-td1676993.html for a discussion on the issue.
also see: http://www.elasticsearch.org/guide/reference/api/search/facets/statistical-facet/ for the proper way to *intentionally* run a multi-field facet.

