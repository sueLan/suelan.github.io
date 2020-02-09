---
title: Understand onViewableItemsChanged in FlatList

date: 2020-01-21 23:03:53
tags: Dive into React Native
categories: React Native
---

## What is onViewableItemsChanged

`onViewableItemsChanged` is a props in [VirtualizedList](https://facebook.github.io/react-native/docs/flatlist#onviewableitemschanged) and it's subclass [FlatList]((https://facebook.github.io/react-native/docs/flatlist#onviewableitemschanged)). When we scroll a FlatList, the items showed on the screen change. This function is called telling what current `viewableItems` are and what `changed` items are. 

This function should be used together with [viewabilityConfig](https://facebook.github.io/react-native/docs/virtualizedlist#viewabilityconfig). A specific `onViewableItemsChanged` will be called when its corresponding `ViewabilityConfig`'s conditions are met.

Here is [ViewabilityConfig](https://facebook.github.io/react-native/docs/flatlist#viewabilityconfig) 
```js 
export type ViewabilityConfig = {
  /**
   * Minimum amount of time (in milliseconds) that an item must be physically viewable before the
   * viewability callback will be fired. A high number means that scrolling through content without
   * stopping will not mark the content as viewable.
   */
  minimumViewTime?: number,

  /**
   * Percent of viewport that must be covered for a partially occluded item to count as
   * "viewable", 0-100. Fully visible items are always considered viewable. A value of 0 means
   * that a single pixel in the viewport makes the item viewable, and a value of 100 means that
   * an item must be either entirely visible or cover the entire viewport to count as viewable.
   */
  viewAreaCoveragePercentThreshold?: number,

  /**
   * Similar to `viewAreaPercentThreshold`, but considers the percent of the item that is visible,
   * rather than the fraction of the viewable area it covers.
   */
  itemVisiblePercentThreshold?: number,

  /**
   * Nothing is considered viewable until the user scrolls or `recordInteraction` is called after
   * render.
   */
  waitForInteraction?: boolean,
|};
```
Here is the type of `onViewableItemsChanged`: 
```js
 /**
   * Called when the viewability of rows changes, as defined by the
   * `viewabilityConfig` prop.
   */
  onViewableItemsChanged?: ?(info: {
    viewableItems: Array<ViewToken>,
    changed: Array<ViewToken>,
    ...
  }) => void,
  
export type ViewToken = {
  item: any,
  // The key of this item
  key: string,
  index: ?number,
  // indicated whether this item is viewable or not
  isViewable: boolean,
  section?: any,
  ...
};
```
## How to use it 

```js
  viewabilityConfig = {
    waitForInteraction: true,
    viewAreaCoveragePercentThreshold: 95,
    itemVisiblePercentThreshold: 75
  }

  onViewableItemsChanged = ({viewableItems, changed}) => {
    console.log("Visible items are", viewableItems);
    console.log("Changed in this iteration", changed);
  };

  render() {
    return (
      <FlatList
        style={{
          paddingTop: 60
        }}
        viewabilityConfig={this.viewabilityConfig}
        onViewableItemsChanged={this.onViewableItemsChanged}
        data={this._items}
        renderItem={this._renderItem}
        keyExtractor={item => item.id}
      />
    )
  }
```

## The Implementation 

### 1. Overview 


![a0336b02.png](/img/9af4f3b0-fe64-4a98-b94e-d456f65ae7cc/a0336b02.png)

### 2. The Viewable Region 

1. Get VirtualizedList layout info from the `nativeEvent` using the `onScroll` func, and store these info in `scrollMetrics` object. 


```js
const timestamp = e.timeStamp;
let visibleLength = this._selectLength(e.nativeEvent.layoutMeasurement);
let contentLength = this._selectLength(e.nativeEvent.contentSize);
let offset = this._selectOffset(e.nativeEvent.contentOffset);
let dOffset = offset - this._scrollMetrics.offset;

// ...
 this._scrollMetrics = {
      contentLength,
      dt,
      dOffset,
      offset,
      timestamp,
      velocity,
      visibleLength,
    };

```

If this is a vertical VirtualizedList, we get these layout info from `nativeEvent` event. The  `layout.layoutMeasurement.height` is the `viewportHeight`, `visibleLength`, the height of viewable region.  

![5d152b82.png](/img/9af4f3b0-fe64-4a98-b94e-d456f65ae7cc/5d152b82.png)

## ViewabilityHelperCallbackTuple

```js
_viewabilityTuples: Array<ViewabilityHelperCallbackTuple> = [];
```

`ViewabilityHelperCallbackTuple` is an array storing `ViewabilityHelper/onViewableItemsChanged` pairs

```js
type ViewabilityHelperCallbackTuple = {
  viewabilityHelper: ViewabilityHelper,
  onViewableItemsChanged: (info: {
    viewableItems: Array<ViewToken>,
    changed: Array<ViewToken>,
    ...
  }) => void,
  ...
};
```

If you define [viewabilityConfigCallbackPairs](https://facebook.github.io/react-native/docs/flatlist#viewabilityconfigcallbackpairs),  each `viewabilityConfig` will be used to initialize a different `ViewabilityHelper` object. 

[ref code](https://github.com/facebook/react-native/blob/84adc85523770ebfee749a020920e0b216cf69f8/Libraries/Lists/VirtualizedList.js#L743). 

```js
 if (this.props.viewabilityConfigCallbackPairs) {
      this._viewabilityTuples = this.props.viewabilityConfigCallbackPairs.map(
        pair => ({
          viewabilityHelper: new ViewabilityHelper(pair.viewabilityConfig),
          onViewableItemsChanged: pair.onViewableItemsChanged,
        }),
      );
    } else if (this.props.onViewableItemsChanged) {
      this._viewabilityTuples.push({
        viewabilityHelper: new ViewabilityHelper(this.props.viewabilityConfig),
        onViewableItemsChanged: this.props.onViewableItemsChanged,
      });
    }

```

Here, we should pay attention to `ViewabilityHelper`, which is `
a utility class for calculating viewable items based on the viewabilityConfig and metrics, like scroll position and layout.
`
For a `VirtualizedList`, there would be multiple `ViewabilityHelper` objects in `_viewabilityTuples`,  containing different `viewabilityConfig` to handle different viewability conditions.  

There are some important props in `ViewabilityHelper`. 

```js
class ViewabilityHelper {
  _config: ViewabilityConfig;
  _hasInteracted: boolean = false;
  /* A set of `timeoutID`, used for memory management */
  _timers: Set<number> = new Set();
  // Indexs of the viewable items
  _viewableIndices: Array<number> = [];
  // A map for viewable items
  _viewableItems: Map<string, ViewToken> = new Map();
}

```

There is a func `_updateViewableItems`,  which is called in many scenarios, like `onScroll` method. It will call `viewabilityHelper.onUpdate` to calculate the viewable items. 

```js
_updateViewableItems(data: any) {
    const {getItemCount} = this.props;

    this._viewabilityTuples.forEach(tuple => {
      tuple.viewabilityHelper.onUpdate(
        getItemCount(data),
        // contentOffset of the list 
        this._scrollMetrics.offset,
        // 🌟 viewportHeight 
        this._scrollMetrics.visibleLength, 
        this._getFrameMetrics,
        this._createViewToken,
        tuple.onViewableItemsChanged,
        this.state,
      );
    });
  }

```
- `this._scrollMetrics.visibleLength` is used as `viewportHeight`
- `this._createViewToken` is used to construct a `ViewToken` object, which contains `item` data, `index`, `key` and viewability of the `item`. 

- [this._getFrameMetrics](1864) is a function to get frame of the cell by index from  `this._frames` map. 

```js
// The map storing the cell layout info
{ [cellKey]: {
      // offset of the item cell
      offset: number, 
      // length of the item cell. width or height determined by the direction of the VirtualizedList
      length: number, 
      index: number,
      inLayout: boolean,
    }
}
```

- Inside `this.state`, we know the range of the rendered items by  `first` and `last` value. 

```js
type State = {
  // The range of the rendered items, 
  // used for the optimization to reduce the scan size
  first: number,
  last: number,
  ...
};
```

### 3. How to get viewable items
 
In `onUpdate` method, it calls `computeViewableItems` to get `viewableIndices` 

#### computeViewableItems

In `computeViewableItems`, it only go through the specific range of items, from `${first}` to `${last}`. If a item is viewable, it will be stored in an array named `viewableIndices`. 

```js
  for (let idx = first; idx <= last; idx++) {
      const metrics = getFrameMetrics(idx);
      if (!metrics) {
        continue;
      }
      // The top of current item cell, relative to the screen 
      const top = metrics.offset - scrollOffset;
      // The bottom of current item cell 
      const bottom = top + metrics.length;
      if (top < viewportHeight && bottom > 0) {
        firstVisible = idx;
        if (
          _isViewable(
            viewAreaMode,
            viewablePercentThreshold,
            top,
            bottom,
            viewportHeight,
            metrics.length,
          )
        ) {
          viewableIndices.push(idx);
        }
      } else if (firstVisible >= 0) {
        break;
      }
    }
    return viewableIndices;
  }
```

![layout.png](/img/9af4f3b0-fe64-4a98-b94e-d456f65ae7cc/3252d60e.png)

### 4. What kind of item is viewable? 

 An item is said to be viewable when any of the following
  is true for longer than `minimumViewTime` milliseconds (after an interaction if `waitForInteraction`
 is true):
 
1.  Occupying >= `viewAreaCoveragePercentThreshold` of the view area XOR fraction of the item
   visible in the view area >= `itemVisiblePercentThreshold`.
 When it comes to the fraction of the item visible in the view area, 
 there are 5 cases we need to care about when the height of a item is small than the viewportHeight. RN use `Math.min(bottom, viewportHeight) - Math.max(top, 0)` to calculate the viewable length. 
    
  ![partial](viewable-partial.png)
2. Entirely visible on screen when the height of a item is bigger than the viewportHeight. 

![7c4f2df0.png](entire-viewable.png)

[ref](https://github.com/facebook/react-native/blob/84adc85523770ebfee749a020920e0b216cf69f8/Libraries/Lists/ViewabilityHelper.js#L64)

```js
function _isViewable(
  viewAreaMode: boolean,
  viewablePercentThreshold: number,
  top: number,
  bottom: number,
  viewportHeight: number,
  itemLength: number,
): boolean {
  if (_isEntirelyVisible(top, bottom, viewportHeight)) {
    // Entirely visible 
    return true;
  } else {
    // Get viewable height of this item cell 
    const pixels = _getPixelsVisible(top, bottom, viewportHeight);
    // Get the viewable percentage of this item cell 
    const percent =
      100 * (viewAreaMode ? pixels / viewportHeight : pixels / itemLength);
    return percent >= viewablePercentThreshold;
  }
}

function _getPixelsVisible(
  top: number,
  bottom: number,
  viewportHeight: number,
): number {
  const visibleHeight = Math.min(bottom, viewportHeight) - Math.max(top, 0);
  return Math.max(0, visibleHeight);
}

function _isEntirelyVisible(
  top: number,
  bottom: number,
  viewportHeight: number,
): boolean {
  return top >= 0 && bottom <= viewportHeight && bottom > top;
}
```
[code here](https://github.com/facebook/react-native/blob/84adc85523770ebfee749a020920e0b216cf69f8/Libraries/Lists/ViewabilityHelper.js#L300)

### 5.Timer and Schedule 
Inside [onUpdate func in ViewabilityHelper](https://github.com/facebook/react-native/blob/84adc85523770ebfee749a020920e0b216cf69f8/Libraries/Lists/ViewabilityHelper.js#L228), if we define `minimumViewTime` value, the `_onUpdateSync` should be scheduled to be called, using `setTimeout`.

```js
 this._viewableIndices = viewableIndices;
 if (this._config.minimumViewTime) {
      const handle = setTimeout(() => {
        this._timers.delete(handle);
        this._onUpdateSync(
          viewableIndices,
          onViewableItemsChanged,
          createViewToken,
        );
      }, this._config.minimumViewTime);
      this._timers.add(handle);
    } else {
      this._onUpdateSync(
        viewableIndices,
        onViewableItemsChanged,
        createViewToken,
      );
    }
```

And, in the [_onUpdateSync](https://github.com/facebook/react-native/blob/84adc85523770ebfee749a020920e0b216cf69f8/Libraries/Lists/ViewabilityHelper.js#L267) func, it will filter out indices that have gone out of view.


```js
    // Filter out indices that have gone out of view since this call was scheduled.
    viewableIndicesToCheck = viewableIndicesToCheck.filter(ii =>
      this._viewableIndices.includes(ii),
    );

```


### 6. How to get changed items 

There is a `preItems`  map, it stores the previous visible items. Now we have a `nextItems` map, we got to know how to the `changed` array by extract the changed items. 

For example, if scrolling down, some viewable items will become out of the screen, some hidden items will become viewable. 
![99830c30.png](/img/9af4f3b0-fe64-4a98-b94e-d456f65ae7cc/99830c30.png)

```js
_onUpdateSync(
    viewableIndicesToCheck,
    onViewableItemsChanged,
    createViewToken,
  ) {
    // Filter out indices that have gone out of view since this call was scheduled.
    viewableIndicesToCheck = viewableIndicesToCheck.filter(ii =>
      this._viewableIndices.includes(ii),
    );
    const prevItems = this._viewableItems;
    // Using map, so the time complexity would be o(n) 
    const nextItems = new Map(
      viewableIndicesToCheck.map(ii => {
        const viewable = createViewToken(ii, true);
        return [viewable.key, viewable];
      }),
    );

    const changed = [];
    for (const [key, viewable] of nextItems) {
      if (!prevItems.has(key)) {
        changed.push(viewable);
      }
    }
    for (const [key, viewable] of prevItems) {
      if (!nextItems.has(key)) {
        changed.push({...viewable, isViewable: false});
      }
    }
    if (changed.length > 0) {
      this._viewableItems = nextItems;
      onViewableItemsChanged({
        viewableItems: Array.from(nextItems.values()),
        changed,
        viewabilityConfig: this._config,
      });
    }
  }
}
```