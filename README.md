---
layout: post
title: "d3js and google maps"
date: 2013-06-16 12:28
comments: true
categories: 
---

I have recently been working on a simple data visualization, using Google Maps and d3.js. At the end my though was: d3 is so cool and it makes writing data visualizations so easy! Really? This wasn't always my idea.

### Struggling with d3

d3 needs no presentations. Anyone that has an interest in client-side JavaScript data visualizations must have played, or at least heard of it at one point. Along with its popularity though, an assumption that the library is difficult to understand and to use has also grown.  Nevertheless, recognizing that data visualizations and informatics is a vast, complex and mathematics dense field does not necessarily mean that all d3 visualizations must be hard and complex to achieve.

I am no expert, but I do think that writing simple data driven documents with d3 can become surprisingly easy, once one manages to grab a few fundamental concepts such as the enter-update-exit pattern.

Thinking about `selectAll` as a placeholder for my data is something it took me time for me to feel comfortable with. And rightly, why should I be selecting non existing stuff in the first place?

At one point in time I was writing code like [this](https://github.com/deciob/data-story/blob/master/app/controllers/viz/bar_base.coffee):

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

Pretty nasty stuff. If I had an empty selection, I appended stuff to the DOM, else I made some transitions on the existing elements. It did work, there is nothing here that harnesses the power and simplicity of the enter-update-exit pattern.

So yes, I did struggle to learn the basics and as I have said, I am no expert. Still, I must have learned something useful because today I feel this way of handling data and dom elements within d3 super natural and I think it greatly simplifies things. Let see why.


### Basic architecture of the application

The backbone.js page is part of a larger rails website, but it is self contained enough to be treated as a simple one page javaScript application. 

It has 2 views, one for the map and one for a list of the data related to a single location (circle) on the map.

We have some data (for the scope of this talk it is irrelevant what the data is about), that gets drawn with d3 on a Google Map, grouped on 3 levels, depending on the zoom level: country, city, location. The user can click on the circles (the data), the circle is re-drawn as active and a list of information about the data appears on the bottom view.

Nothing more. A super simple app. As a starting point for it I copied code from [this block](http://bl.ocks.org/mbostock/899711).

Nevertheless, even the most innocent and naive javaScript web application can go wrong very easily and turn into a inextricable mess. It just tend to happen, and it is not as easy as seems to avoid it, especially when one starts adding previously unforseen features.

And it is within this scenario that I HAVE REALIZED THAT that using d3 and enter-update-exit pattern, helped me to keep things surprisingly simple. 

### How the data is handled

Key point here. If we are talking about data driven documents we need to know how the data is handled. 

As stated, the data is organized on zoom levels by country, city and location. And for these we have 3 backbone model-collections with 3 separate requests. They all share a number of methods though (see `models/base.js.coffee`) and the most important of the is `parseDataForMap`. This method returns the data from the current collection ready to be fed into d3's `selection.data` method.


### The application in detail

Practically all the code related to d3 and Google maps is located in the `views/map_view.js.coffee`. Here the 2 key methods are `initOverlays` and `drawSvg`.

`initOverlays` sets up the initial Google map, with its initial overlay (country related). The methods in here refer to the Gmaps API. 2 important things are happening here:

1. In `overlay.onAdd` we are creating a d3 selection within GMaps for our SVGs and we are then partially applying (using a method in underscore.js) the first 2 arguments of the `drawSvg` method. This because these first 2 arguments never change across the entire life of the application.
2. Once the map is set (`overlay.setMap @map`) the `overlay.draw` is called, calling the partially applied `drawSvg` with the initial data. At this point svg circle will appear on the map.


```coffee
  # Derived from: https://gist.github.com/mbostock/899711
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









