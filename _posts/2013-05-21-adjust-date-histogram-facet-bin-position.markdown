---
layout: post
title: "Adjust Date Histogram Facet's Bin Position"
date: 2013-05-21
preview: basic histogram adjustment for ElasticSearch
---

ElasticSearch's *Date Histogram* is a nice built-in facet for binning data across a specified time interval.Using Tire (Ruby's ElasticSearch wrapper), you can get a timeseries data thusly

    Tire.index('articles').search do
      facet 'words_per_month' do
        date :created_at,
             :interval => "30d",
             :value_field => :word_count
      end
    end

The setting `:interval => "30d"` bins each document within a thirty day interval. ElasticSearch allows for bin widths from milliseconds to weeks; see the syntax in the documentation. Hence, if a document's `created_at` field is within a 30 day interval, the document will be placed in that bin and the specified `value_field` will be passed on to statistical functions.

This feature is pretty neat, but there's one quirk in the way the kernel is placed for each bin. Let's say you want to run statistics against documents relative to today's date; you'll have a problem, as the default kernel is set on the mid-month, eg. 04-14.

    Tire.index('articles').search do
      facet 'words_per_month' do
        date :created_at,
             :interval => "30d",
             :post_offset => "-13d",
             :value_field => :word_count
      end
    end

The above will place your kernel at the start of the month by offsetting the default (eg, 04-14) to the default minus 13 days (04-01). But what we want is an offset of *n* days where *n* is the difference between the default and today's date. Viola, in Ruby:

    n = (Time.now.day - 14).to_s
    Tire.index('articles').search do
      facet 'words_per_month' do
        date :created_at,
             :interval => "30d",
             :post_offset => "#{n}d",
             :value_field => :word_count
      end
    end

Try it!
