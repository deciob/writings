## Title (TODO)


**Note**: the following article assumes the reader has at least some basic knowledge of d3. If one has never used d3 before I would suggest reading the [introduction](http://d3js.org/) on the library homepage and [Thinking with Joins](http://bost.ocks.org/mike/join/) before continuing the read.


I have recently been working at a [simple visualization](http://wcmc.io/map), where some circles, generated in **d3** and representing ip addresses, are layered out on **Google Maps**. It turned out that using **d3** made it surprisingly clean and easy to handle the user-driven visualization changes. Surprisingly because, as many others, I somehow struggled to fully understand and correctly use **d3**.

To give an idea about how **d3** can be misused, here is an excerpt from some [old code](https://github.com/deciob/data-story/blob/master/app/controllers/viz/bar_base.coffee) of mine:

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

Although this code works, inserting a `selectAll` into an if statement, to check if the selection is empty, is nasty and bad stuff! But thinking about `selectAll` as a placeholder for my data is something that took some time for me to feel comfortable with. 

These days, after some struggles and experiences, I think that writing simple **Data Driven Documents** with **d3**, once one manages to grab a few fundamental concepts such as the **enter-update-exit** pattern, can become surprisingly easy. And hopefully, the following walk-trough will help others to understand.


### The architecture basics.

The [page](http://wcmc.io/map) in question is a **backbone.js** application that is part of a larger **rails** website, but can easily be analyzed in isolation. The starting point for this code was the following [block](http://bl.ocks.org/mbostock/899711).

It has two views, one for the map and one for listing the ip locations data details that relate to the circles on the map.

The data rendered on **Google Maps** is clustered on 3 levels, depending on the zoom: country, city and individual locations. The user can click on the circles (the data) and these are re-drawn as active whilst a list of information appears on the bottom view. 

Frankly, a quite simple application, with not much interaction. Nevertheless, even the most innocent and naive **JavaScript** Web application, we all know, can easily grow into an inextricable mess. **Backbone** here helps, but the **update-data-update-view** pattern doesn't always fit well with a **Google Maps** data representation. It is here that **d3** enters the stage and helps, with its **data enter-update-exit** pattern, to keep things surprisingly simple.


### How the data is structured and handled.

For every cluster level we have a **backbone** model-collection and all of them inherit a number of methods from the base-class [base.js.coffee](https://github.com/unepwcmc/wukumurl/blob/master/app/assets/javascripts/map/models/base.js.coffee). The most important of these is `parseDataForMap`, that returns the current collection data in the form expected by **d3**'s `selection.data` method in [map_view.js.coffee](https://github.com/unepwcmc/wukumurl/blob/master/app/assets/javascripts/map/views/map_view.js.coffee).

For clarity, this is an example of an object in the array returned from `parseDataForMap` on the countries collection:

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

Almost all the code related to **d3** and **Google Maps** lives in [map_view.js.coffee](https://github.com/unepwcmc/wukumurl/blob/master/app/assets/javascripts/map/views/map_view.js.coffee). Here, two key functions occupy the scene: `initOverlays` and `drawSvg`.

**initOverlays**

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

`initOverlays` sets up the initial **Google Maps** configuration and initial overlay (the **SVG** circles related to the country clusters). The nested methods here refer to the [Google Maps JavaScript API](https://developers.google.com/maps/documentation/javascript/). It is worth noticing:

1. In `overlay.onAdd` we are setting a **d3** selection in the map, for our **SVG** elements, and we are then partially applying the `drawSvg` method with it, because this initial selection never changes across the entire life of the application.
2. Once the map is set (after calling `overlay.setMap`) `drawSvg` is called with the initial data. At this point the **SVG** circles, representing the ip locations clustered on the country centroids, will appear on the map.


**drawSvg**
```coffee
  drawSvg: (layer, self, data) ->
    #
    # ... more code...
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
      .each(transformTxt)
      .attr("dy", ".31em")
      .text((d) -> d.size)
      .style("font-size", (d) -> 
        s = self.getFontSize d.size
        "#{s}px"
      )

    circle = enter.append("svg:circle")
      .each(addEventListeners)
      .each(transformCircle)

    exit = marker.exit()
      #.each(removeEventListener) # TODO?
      .remove()
```

And finally we get to analyze the `drawSvg` method, that hopefully will shed some light on how **d3**'s **enter-update-exit** pattern works. Let's walk through it across three different states of our application.


#### 1. We start from the initial data load state.

* `marker = layer.selectAll("svg")` selects all the **SVG** elements within our layer element (the div we have created earlier, in the `initOverlays` method, remember?). The first time this is called, the selection will be empty, just a placeholder for future elements.
* `.data(data, (d) -> d.uique_id)` is the data join. Note how I am passing a key function as the second argument. Not passing this key value, that uniquely identifies each object, would insert a bug in the code when changing levels (country-city-locations). The data objects entering or exiting the selection would be compared by array index, rather than object id and would no longer be distinguishable.
* `.each(transform)` is used to correctly set the elements on Google Maps. It is called before and after the `enter` point because of how Google Maps works: on every zoom change `overlay.draw` will be called and everything will need updating again.
* `enter = marker.enter().append("svg:svg")` is the entry point where our new **SVG**s are appended to the DOM. On the first call all the data is new and entering. The function then continues appending text (numbers indicating the size of the data) and circle elements to our recently created **SVG** elements.
* `exit = marker.exit().remove()` finally cleans up the old data. On the first load though, all data is new and nothing is in exit.

#### So what happens when we pass from zoom level 5 to 6 and our data is now clustered around cities and no longer around country centroids?

* `marker = layer.selectAll("svg")` is now no longer empty.
* `.data(data, (d) -> d.uique_id)`, here new data is coming into the selection, with new unique_ids, all different from the existing ids (the country cluster objects).
* `enter = marker.enter().append("svg:svg")`, the incoming data is all new, so we have an entry point for every object in the data array.
* `exit = marker.exit().remove()`, the exit is full, all the old, existing data is now old, because the object ids no longer match, so they are all removed.


#### But what about the `update` portion of the pattern? Are we using it here at all? Yes we are. The data gets updated when the user selects, with a click, an **SVG** circle on the map. 

* `marker = layer.selectAll("svg")`, again, this selection is not empty.
* `.data(data, (d) -> d.uique_id)`, new data is coming in, but in this case, none of the object ids have changed! 
* `enter = marker.enter().append("svg:svg")`, so no new data is appended.
* `exit = marker.exit().remove()`: and no old data is removed.
But
* `.attr("class", (d) -> d.state)`, but the state attribute will be updated, only for the the selected **SVG**.


### Conclusion

..... TODO ... 










