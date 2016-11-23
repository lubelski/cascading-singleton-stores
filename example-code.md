CSS is a experiment in making flux more like React, until it's not really flux any more. 


### list-store.coffee

This store holds onto a list based on some id stored elsewhere.  it does not cache old list data and simply requests new data when the `list_id` changes.  

```coffee
CSS = require("cascading-singleton-stores")
Api = require("./api")
LocationStore = require("./location-store")

CSS.createStore "ListStore", 
   props:
      list_id: LocationStore.getListId
      
      
   storeWillRecieveProps: (prevProps) -> 
      if nextProps.list_id isnt @props.list_id
         @fetchList(nextProps.list_id)

   getInititalState: -> 
      list: null
      
   fetchList: (list_id) -> 
      @setState loading: true
      Api.fetchList(@props.list_id)
         .then (list) => 
            if list.id is @props.list_id
               @setState list: list

   actions: 
     refresh: -> 
        if list_id = @props.list_id
          @fetchList(list_id)
      
   views: 
      getIsLoading: -> 
         @props.list_id and @state.list?.id isnt @props.list_id

      getList: -> 
         @state.list      
```

### items-cache-store.coffee

This store holds onto a cache of items (which have to lazy loaded and are not returned in the list response).  it caches these items responses so that it does not rerequest items when the list changes.  

This store exposes a view which contains the list data merged with item responses.  

```coffee

CSS = require("cascading-singleton-stores")
Api = require("./api")
LocationStore = require("./location-store")
ListStore = require("./list-store")

CSS.createStore "ListWithItemsStore", 
   props:
      list: ListStore.getList
   
   getInititalState: -> 
      items: {}
      loading_ids: []

   addItem: (item) -> 
      @setState items: _.extend(@state.items, [item.id]:item)
     
   storeWillRecieveProps: (nextProps) -> 
      current_ids = _.keys(@state.items).concat(@state.loading_ids) 
      new_ids = _.pluck(nextProps.list, "id")
      to_be_loaded = _.difference(new_ids, current_ids) 
      if not _.isEmpty(to_be_loaded)

   refreshItems: (item_ids) ->
      @setState loading_ids: _.uniq @state.loading_ids.concat(item_ids)
      
      Api.fetchItems(item_ids)
         .then (items) => 
            @setState loading_items: _.without(@state.loading_items, item_ids)
            _.each items, (item) => @addItem(item)
   

   views: 
      getListWithItems: -> 
         _.map @props.list, (list_item) =>
             {id, name} = list_item
                name: name
                id: id
                details: @state.items[list_item.id]
                is_loading: _.contains(@state.loading_ids, id)
```

### list-view.coffee

This react component takes props and can display a list of items.  

```coffee
React = require("react")
{div, span} = React.DOM
{array, func} = React.PropTypes

React.createClass
   displayName: "ListView"
   propTypes: 
      list: array.isRequired
      onItemClick: func.isRequired # args: (item_id) 

   render: -> 
      div className: "list-view", 
         _.map @props.list, (item) -> 
            div 
               className: "list-item", 
               onClick: => @props.onItemClick(item.id)
               span className: "name", item.name
               span className: "info", item.info
```

### list-controller.coffee

This controller binds the list data and onClick callbacks to the display component

```coffee
CSS = require("cascading-singleton-stores")
ListWithItemsStore = require("./list-with-items-store")
ListView = require("./list-view")

ListController = CSS.connect ListView, -> 
   list: ListWithItemsStore.getListWithItems
   onItemClick: => alert(1)
      
```
