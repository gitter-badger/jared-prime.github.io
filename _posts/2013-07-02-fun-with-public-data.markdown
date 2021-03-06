---
layout: post
title: "Fun with Public Data"
date: 2013-07-02
preview: play with some public datasets on Chicago Transit Authority ridership
---

Similar datasets can be found at [data.cityofchicago.org](https://data.cityofchicago.org/). In this post, I'll try to keep the prose at a minimum and let the code do the talking.

First, let's set some objectives: (a) acquire the data, (b) reformat the data for passing around our code, (c) store the data for querying, (d) run some queries and present the results. To prepare, let's grab some tools (in Ruby)

    # objective (a)
    require 'httparty'
    # objective (b)
    require 'json'
    # objective (c) and (d)
    require 'sqlite3'
    require 'sequel'
    # objective (d)
    require 'gnuplot'

## Objective (a)

HTTParty will do the acquisition for us, which is pretty easy given Chicago's data API:

    full_data = HTTParty.get('https://data.cityofchicago.org/api/views/mq3i-nnqe/rows.json')

## Objective (b)

I chose to grab the json document; the format of the document makes my job harder, but I'm more familiar with parsing json objects than other (potentially wackily formatted) filetypes.

## Objective (c)

Now, we set up the database. The following will create an in-memory database and some tables.

    CTA_BUSES = Sequel.sqlite

    CTA_BUSES.create_table :passenger_data do
      # auto increment id
      foriegn_key       :stop_id
      Decimal           :alightings
      Decimal           :boardings
    end

    CTA_BUSES.create_table :stop_locations do
      # auto increment id
      foriegn_key       :stop_id
      Decimal           :loc_lat
      Decimal           :loc_long
      String            :cross_street
      String            :on_street
    end

    CTA_BUSES.create_table :stops do
      primary_key       :id
    end

    CTA_BUSES.create_table :route_stops do
      # auto increment id
      foriegn_key       :stop_id
      String            :route_name
    end

I don't promise that these schemata are good architecture; it's just what I settled on to do the best analysis I could. I'd love to see better :)

After cleaning the data for insertion, we simply use Sequel's very straightforward syntax.

    stops.each          { |data| CTA_BUSES[:stops].insert(data) }
    stop_locations.each { |data| CTA_BUSES[:stop_locations].insert(data) }
    route_stops.each    { |data| CTA_BUSES[:route_stops].insert(data) }
    passenger_data.each { |data| CTA_BUSES[:passenger_data].insert(data) }

The code inside the loop - <code>CTA_BUSES[:stops].insert(data)</code> - is sugar for <code>INSERT INTO `stops` (keys) VALUES (values)</code> where <code>key</code> and <code>value</code> are the keys and values of the hash <code>data</code>. There's probably a better way to do a bulk insert that doesn't rely on a loop... just don't know Sequel's syntax for this case and didn't bother looking.

## Objective (d)

Now we can query! I just wrote raw SQL since I'm interested in learning better query techniques without the help of an ORM.

    full_query =<<SQL
      select rs.route_name, sum(pd.boardings) as total_boardings, sum(pd.alightings) as total_alightings
      from route_stops as rs
      left join passenger_data pd on pd.stop_id = rs.stop_id
      group by rs.route_name
      order by rs.route_name
    SQL

For reasons that will become clearer in a moment, I wrote a number of other queries, including these two:

    focused_query =<<SQL
      select rs.route_name, sum(pd.boardings) as total_boardings, sum(pd.alightings) as total_alightings
      from route_stops as rs
      left join passenger_data pd on pd.stop_id = rs.stop_id
      where rs.route_name in (146, 29, 2, 'X28', 4, 26)
      group by rs.route_name
      order by rs.route_name
    SQL

    map_query =<<SQL
      select rs.route_name, rs.stop_id,
      pd.boardings as total_boardings, pd.alightings as total_alightings,
      sl.loc_lat, sl.loc_long
      from route_stops as rs
      left join passenger_data pd on pd.stop_id = rs.stop_id
      left join stop_locations sl on sl.stop_id = rs.stop_id
      where rs.route_name in (146, 29, 2, 'X28', 4, 26)
    SQL

Let's grab the results...

    full_results   = CTA_BUSES.fetch(full_query).all
    focused_routes = CTA_BUSES.fetch(focused_query).all
    overlay_data   = CTA_BUSES.fetch(map_query).all

and throw some results into an histogram using gnuplot!

    Gnuplot.open do |gp|
      Gnuplot::Plot.new (gp) do |plot|
        plot.title "Total Boardings and Alightings by Route, 'Entry Routes'"
        plot.style "data histogram"
        plot.xtics  "nomirror rotate by -90"
        #plot.xrange "[0:160]"

        routes     = entry_routes.map {|data| data[:route_name]}
        boardings  = entry_routes.map {|data| data[:total_boardings]}
        alightings = entry_routes.map {|data| data[:total_alightings]}

        plot.data << Gnuplot::DataSet.new( [routes, boardings] )  {|ds| ds.title = "Boardings";  ds.using = '2:xtic(1)'}
        plot.data << Gnuplot::DataSet.new( [routes, alightings] ) {|ds| ds.title = "Alightings"; ds.using = '2:xtic(1)'}
      end
    end

gives us...

<img src="/all_routes.png" alt="All Routes" title="All Routes" style="max-width:100%;" />

...and...

    Gnuplot.open do |gp|
      Gnuplot::Plot.new (gp) do |plot|
        plot.title "Total Boardings and Alightings by Route, Select Routes"
        plot.style "data histogram"
        plot.xtics  "nomirror rotate by -90"

        routes     = focused_routes.map {|data| data[:route_name]}
        boardings  = focused_routes.map {|data| data[:total_boardings]}
        alightings = focused_routes.map {|data| data[:total_alightings]}

        plot.data << Gnuplot::DataSet.new( [routes, boardings] )  {|ds| ds.title = "Boardings";  ds.using = '2:xtic(1)'}
        plot.data << Gnuplot::DataSet.new( [routes, alightings] ) {|ds| ds.title = "Alightings"; ds.using = '2:xtic(1)'}
      end
    end

gives us...

<img src="/select_routes.png" alt="Select Routes" title="Select Routes" style="max-width:100%;" />

Lastly, we can also plot our data on a map!

    map_data = ['146', '29', '2', 'X28', '4', '26'].map { |selected_route|
      routes[selected_route] = overlay_data.
        select{|route| route[:route_name] == selected_route}.map { |route|
        {
         center: [route[:loc_lat], route[:loc_long]],
         boardings: route[:total_boardings].to_f,
         alightings: route[:total_alightings].to_f
        }
      }
    }.to_json

    File.open('./map.json','w'){ |f| f.write map_data }

    var busStop;

    function initialize() {
      var mapOptions = {
        zoom: 12,
        center: new google.maps.LatLng(41.8500, -87.6500),
        mapTypeId: google.maps.MapTypeId.ROADMAP
      };
      var map = new google.maps.Map(document.getElementById('map-canvas'),
          mapOptions);

      var colors = ['#8dd3c7','#ffffb3','#bebada','#fb8072','#80b1d3','#fd8462'];

      for (var i=0; i < bus_routes.length; i++){
        for (var j=0; j < bus_routes[i].length; j++){
          var stop = bus_routes[i][j];
          var passengerData = {
            strokeColor:colors[i],
            strokeOpacity: 0.8,
            strokeWeight: 2,
            fillColor: colors[i],
            fillOpacity: 0.35,
            map: map,
            center: new google.maps.LatLng(stop['center'][0], stop['center'][1]),
            radius: stop['boardings']
          };
          busStop = new google.maps.Circle(passengerData);
        }
      }
    }

    google.maps.event.addDomListener(window, 'load', initialize);

Giving us the lovely map shown below.

<script src="https://maps.googleapis.com/maps/api/js?v=3.exp&sensor=false"></script>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
<script src='/map.json'></script>
<script>
  var busStop;

  function initialize() {
    var mapOptions = {
      zoom: 12,
      center: new google.maps.LatLng(41.8500, -87.6500),
      mapTypeId: google.maps.MapTypeId.ROADMAP
    };
    var map = new google.maps.Map(document.getElementById('map-canvas'),
        mapOptions);

    var colors = ['#8dd3c7','#ffffb3','#bebada','#fb8072','#80b1d3','#fd8462'];

    for (var i=0; i < bus_routes.length; i++){
      for (var j=0; j < bus_routes[i].length; j++){
        var stop = bus_routes[i][j];
        var passengerData = {
          strokeColor:colors[i],
          strokeOpacity: 0.8,
          strokeWeight: 2,
          fillColor: colors[i],
          fillOpacity: 0.35,
          map: map,
          center: new google.maps.LatLng(stop['center'][0], stop['center'][1]),
          radius: stop['boardings']
        };
        busStop = new google.maps.Circle(passengerData);
      }
    }
  }

  google.maps.event.addDomListener(window, 'load', initialize);

</script>
<div id="map-canvas" style="height:400px"></div>

Not the best visual analytics work, but I hope to have given a broad overview of what one needs to do in order to do something cool with data. Just spend 10x the amount of time/effort I put into this code to create something truly amazing!
