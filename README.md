---
layout: post
title: "d3js and google maps"
date: 2013-06-16 12:28
comments: true
categories: 
---

I have recently been working at a [simple visualization](http://wcmc.io/map), where some circles, generated in d3 and representing ip addresses, are layered out on a Google Map. To some surprise it turned out that using d3 made it very easy to handle the user-driven visualization changes. To some surprise because I have never considered myself a d3 expert and as many others, I often struggled to understand it and use it the right way.

To give an idea about how bad I was, not long ago I was still writing code like [this](https://github.com/deciob/data-story/blob/master/app/controllers/viz/bar_base.coffee):

```coffee
  # first check if there is no histogram around...
  if @g.selectAll('path')[0].length == 0
    @bar = @g.selectAll("g.bar")
      .data(data)
      .enter().append("g")
    @bar.append("rect")
  #
  # ...more code...
  #
  else
    @g.selectAll("rect")
      .data(data)
      .transition()
```

I know, inserting a `selectAll` into an if statement, to check if the selection is empty, is pretty nasty stuff. But thinking about `selectAll` as a placeholder for my data is something that took some time for me to feel comfortable with. My dominant thought was: why should I be selecting non existing stuff in the first place?

These days, after some struggles and hard lessons, I think that writing simple data driven documents with d3 can become surprisingly easy, once one manages to grab a few fundamental concepts such as the enter-update-exit pattern. And hopefully this application is an excellent real world example for explaining this in a practical way.


### The architecture basics.

The page is backbone.js application that is part of a larger rails website, but it can easily be analyzed in isolation. 

It has 2 views, one for the map and one for listing the data related to ip locations (circle) on the map.

The data that gets drawn on Google Map is clustered on 3 levels, depending on the zoom: country, city and individual location. The user can click on the circles (the data), the circle is re-drawn as active and a list of information about that data appears on the bottom view. The starting point for the code was the following [block](http://bl.ocks.org/mbostock/899711).

A quite simple application, with not much interaction. Nevertheless, even the most innocent and naive JavaScript Web application can easily grow into an inextricable mess. Backbone helps, but the update-data-update-view pattern does not easily fit on a slippy Google Map. And it is here that enters d3 and its enter-update-exit pattern to keep things surprisingly simple.


### How the data is structured and handled.

For every cluster level we have a backbone model and collection, but all 3 share a number of methods (see: [base.js.coffee](https://github.com/unepwcmc/wukumurl/blob/master/app/assets/javascripts/map/models/base.js.coffee)). The most important of these methods is `parseDataForMap`. It returns the current collection data with the correct structure to be passed d3's `selection.data` method.

This is an example of an object in the array returned from `parseDataForMap` on the countries collection:

```js
[
  {
    "lat":35.8706911472,
    "lng":104.1904317352,
    "state":"inactive",
    "size":2,
    "id":122,
    "uique_id":262.0611228824
  },
]
```

### The application flow.

Practically all the code related to d3 and Google Maps lives in [map_view.js.coffee](https://github.com/unepwcmc/wukumurl/blob/master/app/assets/javascripts/map/views/map_view.js.coffee). Here, the 2 key functions to understand what is happening are `initOverlays` and `drawSvg`.

`initOverlays` sets up the initial Google Maps configuration and its initial overlay (the svg circles related to the country clusters). The methods in here refer to the [Google Maps JavaScript API v3](https://developers.google.com/maps/documentation/javascript/) and two important things are happening:

1. In `overlay.onAdd` we are creating a d3 selection within the map for our SVG elements and we are then partially applying (using a method from underscore.js) the first 2 arguments of the `drawSvg` method. This because these first 2 arguments will never change across the entire life of the application.
2. Once the map is set (`overlay.setMap @map`) `overlay.draw` is called, consequently calling the partially applied `drawSvg` with the initial data. At this point the SVG circles representing the ip locations clustered on the country centroids will appear on the map.


```coffee
  initOverlays: ->
    self = this
    # Remove loading gif
    $(".ajax-loader").hide()
    @data = @collection.parseDataForMap()
    @max = @collection.getMaxVal()
    # Create an overlay.
    overlay = new google.maps.OverlayView()
  
    # Add the container when the overlay is added to the map.
    overlay.onAdd = ->
      layer = d3.select(@getPanes().overlayMouseTarget)
        .append("div").attr("class", "locations")
      self.drawSvg = _.bind self.drawSvg, @, layer, self
  
    overlay.draw = ->
      self.drawSvg self.data
  
    # Bind our overlay to the map…
    overlay.setMap @map
```

In the `drawSvg` methods a lot is happening, so lets go through its main point and lets try to understand them in the context of the enter-update-exit pattern.

```coffee
  # The function is partially applied with `layer` and `self` arguments.
  # `layer` is the div element that wraps all the svg elements that 
  #  appear on the map.
  # `self` is a reference to the instance (this) of WukumUrl.Map.Views.Map.
  drawSvg: (layer, self, data) ->
    #
    # More code
    #
    marker = layer.selectAll("svg")
      .data(data, (d) -> d.uique_id)
      .each(transform) # update existing markers
      .attr("class", (d) -> d.state)

    enter = marker.enter().append("svg:svg")
      .each(transform)
      .attr("class", (d) -> d.state)

    # Lets position the text first (under the circle), so it does not interfere
    # with the click events. Only works because our circles are semi-transparent.
    txt = enter.append("svg:text")
      .attr("x", (d) ->
        cR(d.size * rFactor) - self.centreText(d.size))
      .attr("y", (d) -> 
        cR(d.size * rFactor) )
      .attr("dy", ".31em")
      .text((d) -> d.size)
      .style("font-size", (d) -> 
        s = self.getFontSize d.size
        "#{s}px"
      )

    circle = enter.append("svg:circle")
      .each(addEventListeners)
      .attr("r", (d) -> cR(d.size * rFactor) )
      .attr("cx", (d) -> cR(d.size * rFactor) )
      .attr("cy", (d) -> cR(d.size * rFactor) )

    exit = marker.exit()
      #.each(removeEventListener) # TODO?
      .remove()
```

* `marker = layer.selectAll("svg")`: I am selecting all the svg elements within our layer element (the div we have created earlier in the `initOverlays` method, remember?). The first time this is called the selection will be empty.
* `.data(data, (d) -> d.uique_id)`: The data join point, notice here that I am passing in a key value as the second argument. This is necessary for the following reason. The data is an array of objects and these objects have different attributes, among witch lat and lng values. Now say that on my first level, the country one, i pass in an array of length 10 and later, on my city level, i pass in an array of length 15. Without passing the key function that uniquely identifies each object, changing levels would trigger a wrong update of the svg elements. The first 10 objects in the city data would be treated not as enter points, but as updates of the existing data. the last 5, would be correctly entered as new. The 10 country objects would not exit.
* `.each(transform)`: This is used to correctly set the elements on to google maps. It is called before and after the `enter` because of how google maps works. On every zoom change Google maps will call `overlay.draw` again and everithing needs to be updated.
* `enter = marker.enter().append("svg:svg")`: the entry point where our new svgs are appended to the DOM, depending not on the data array index but on the data uique_id attribute. On its first call all the data is new and entering. We then continue appending text (numbers indicating the size of the data) and circle elements to our svgs.
* `exit = marker.exit().remove()`: Finally we are getting rid of the old data. On our first call no data is in exit.

So what happens when we pass from zoom level 5 to 6 and our data is now clustered around cities and no longer around country centroids?

* `marker = layer.selectAll("svg")`: This selection is now no longer empty.
* `.data(data, (d) -> d.uique_id)`: New data is coming in, with new uique_ids, all different from the ids in the existing selection.
* `enter = marker.enter().append("svg:svg")`: the data is all new, so we have an entry point for every object in the data array.
* `exit = marker.exit().remove()`: All the data in the previous selection (the country centered data) is now all old, because entering uique_ids do not match the existing ones, so it is all removed. 


But what about the `update` portion of the pattern. Are we using it here, or is everything entering and exiting in block? Yes we are updating the data, when the user selects, with a click, an svg circle on the map. let see what happenes again in this case:

* `marker = layer.selectAll("svg")`: Again, this selection is not empty.
* `.data(data, (d) -> d.uique_id)`: New data is coming in, but in this case none of the unique_ids in the data objects have changed! 
* `enter = marker.enter().append("svg:svg")`: No new data is appended!
* `exit = marker.exit().remove()`: No old data is removed!
But
* `.attr("class", (d) -> d.state)`: this will be updated for the svg element that has been selected, because this attribute, for that single element, changes.


### Conclusion

The `drawSvg` method, we have seen, is called across the whole existence the application. No ifs, no elses. i am not interested in anything... everything is automatically updated!









