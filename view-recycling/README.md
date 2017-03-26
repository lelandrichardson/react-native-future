# Collection View Recycling in React Native

## Motivation

View Recycling is a really important optimization strategy for both the iOS and Android platforms.
Highly related to the "ListView Problem" in React Native, view recycling on both platforms is the
most common way to get high-performing screens displaying lots of data.

The platform solutions for this come in the form of:

### iOS

- [UITableView](https://developer.apple.com/reference/uikit/uitableview)
- [UICollectionView](https://developer.apple.com/reference/uikit/uicollectionview)

### Android

- [RecyclerView](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.html)


Most of the solutions for lists in react native employ performance optimizations on the JS / React
side of the bridge, and have been somewhat successful with this. The latest `FlatList` implementation
has shown to be performant enough for a lot of applications.

There is still a limit though. This is still not recycling any views, and by allocating / deallocating
views rapidly on the native side, we can run into scroll performance and memory pressure issues.

I believe if we are to finally "put this issue to bed", the solution will need to involve some
ingenuity on both the Native and JS side of the bridge.


### Current solutions and reading

A lot of work has been put into the "ListView Problem" on React Native already. For context, check
out:

- [Good discussion on the design decisions behind ListView](https://github.com/facebook/react-native/issues/499)
- [RFC for new FlatList](https://gist.github.com/sahrens/902d49c6c154cd09fafc52a79503728f)
- [Brent Vatne building li.st at ReactEurope 2016](https://www.youtube.com/watch?v=cI9bDvDEsYE)
- [react-native-tableview by Pavel Aksonov](https://github.com/aksonov/react-native-tableview)
- [FlatList commit in RN Core](https://github.com/facebook/react-native/commit/a3457486e39dc752799b1103ebe606224a8e8d32)
- [BindingListView from Wix](https://github.com/wix/BindingListView)
- [Recycling Rows For High Performance React Native List Views](https://medium.com/@talkol/recycling-rows-for-high-performance-react-native-list-views-628fd0363861#.23ra3rkf2)
- [React Native ListView Performance Revisited — Recycling Without the Bridge](https://hackernoon.com/react-native-listview-performance-revisited-recycling-without-the-bridge-c4f62d18c7dd#.jiwqk2o6l)
- [react-native-sglistview by Shaheen Ghiassy](https://github.com/sghiassy/react-native-sglistview)


## Goals

- minimize allocation / deallocation of native views during scrolling
- don't allocate views that aren't visible
- minimize react component allocation / deallocation
- minimize data serialized across the bridge
- don't rely on JS thread for any scroll interaction
- possible to scale out to extremely large datasets
- support heterogenous lists


## "Layout" Components

"Layout" native components is a term I'm creating for purposes of this RFC. A
Layout component is a native component whose direct children are expected to be
`ViewPool` components. The Layout component will draw its actual children (the child
views in the actual view hierarchy) from their children pools. Layouts themselves
determine how the views are used and laid out. For simplicity, I'm showing a
`VerticalLayout` component here, which would be your common "vertical list" layout.
Other layouts could be horizontally laid out components, grids, masonry style grids,
etc. The Layout components are in charge of emitting an `onUpdate` event when the
need for new views not available in its child view pools. When this event is fired,
it is expected that the owning component will then rerender (in the React sense) the
child ViewPools so that they satisfy the new needs.

```js
const VerticalLayout = requireNativeComponent('VerticalLayout', {
  onUpdate: Function,
  totalItems: number,
  offset: number,
  limit: number,
  types: [ComponentKey],
  children: [ReactElement<ViewPool>],
});
```

## ViewPool Components

This is a general purpose native component called a `ViewPool`. A `ViewPool` is a native
component that doesn't actually attach any views to the hierarchy. It merely intercepts
`insertReactSubView` and holds those views in memory for someone (a Layout) to consume.
Each ViewPool has roughly "homogeneous" views that it holds on to... at least as
can be guaranteed by the guarantee that all of the views were rendered at the top level
by the same composite react component. This component is for the most part just a way to
have a pool of react-rendered views without modifying the react native architecture too
much.

```js
const ViewPool = requireNativeComponent('ViewPool', {
  type: string,
  children: any, // React elements of the type that `type` signifies here
});
```


### Basic Types

```js
// a string identifier for the react component type of the pool
type ComponentKey = string;

// Essentially an array of indexes annotated with a ComponentKey type
type Pool = {
  key: ComponentKey,
  indexes: Array<number>, // a list of indexes to use to retreive the items by index
}
```


## Public Component Implementation

With the `ViewPool` and `VerticalLayout` building blocks, we are able to assemble a react component
with the public API that we want. Below is a rough implementation for one, though not complete.

```js
class RecyclingVerticalListView<T> extends React.Component {
  props = {
    // the actual data source. We could provide an alternative version of this component
    // that didn't need this prop, but instead just had an `itemForIndex` method or
    // something, but I'm not going to worry about that for now as this is simpler to
    // visualize.
    items: [T],

    // the initial number of items to render
    initialItemCount: number,

    // provided a ComponentKey, return the correct react component.
    componentForKey: (key: ComponentKey) => Function,

    // provided an item, return the ComponentKey we want to use
    keyForItem: (item: T) => ComponentKey,

    // provided an item, return the props to be passed into that given item's component
    itemToProps: (item: T) => any,
  };
  state = {
    pooledChildren: [Pool],
  };
  onUpdate(
    pooledChildren: [Pool],
    limit: number,
    offset: number,
  ) {
    this.setState({ pooledChildren, limit, offset });
  }
  constructor(props) {
    super(props);
    this.state = {
      pooledChildren: null,
      offset: 0,
      limit: props.initialItemCount,
    };
  }
  render() {
    const {
      items,
      initialItemCount,
      componentForKey,
      keyForItem,
      itemToProps,
    } = this.props;
    let {
      pooledChildren,
      offset,
      limit,
    } = this.state;

    const types = items.slice(offset, limit).map(keyForItem);

    if (pooledChildren === null) {
      // first render, so we have to create the initial state ourselves.
      pooledChildren = poolItems(keyForItem, items.slice(0, initialItemCount));
    }

    return (
      <VerticalLayout
        totalItems={items.length}
        types={types}
        offset={offset}
        limit={limit}
        onUpdate={this.onPooledChildrenUpdated}
      >
        {pooledChildren.map(({ key, indexes }) => {
          const ItemComponent = componentForKey(key);
          return (
            <ViewPool
              key={key}
              type={key}
            >
              {indexes.map((index, i) => (
                <ItemComponent
                  key={i}
                  {...itemToProps(items[index])}
                />
              ))}
            </ViewPool>
          );
        })}
      </VerticalLayout>
    );
  }
}
```


In this case, if we have `O(n)` items, but `O(k)` of them are visible on the screen at any given
time, our `render` method is always `O(k)`, and the data that we serialize across the bridge at
any given time is always `O(k)`. Moreover, we are not serializing entire data models over the
bridge, but rather just really light-weight arrays of indexes and keys.

Additionally, consider the scenario where all of the `ItemComponent`s implement a
`shouldComponentUpdate` function. In the case where an item of a given type goes offscreen, and
a new item of the same type comes onscreen, there will be NO react components mounted or unmounted,
and instead just a *single* `componentWillReceiveProps` that gets fired. This is ensured by the
fact that the `key`s we use for the `ItemComponents` are not actually the "stable ids" of the data
source, but rather





```
      React View Hierarchy        ───────────▶     Native View Hierarchy

┌─────────────────────────────────┐           ┌─────────────────────────────┐
│<VerticalLayout />               │           │0                            │
│┌───────────────────────────────┐│           │                             │
││<ViewPool type="A" />          ││           │            <A />            │
││┌─────────────────────────────┐││           │                             │
│││0                            │││          ┌│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│┐ ◁
│││                             │││           ├─────────────────────────────┤  │
│││            <A />            │││          ││1                            ││ │
│││                             │││           │                             │  │
│││                             │││          ││            <A />            ││ │
││└─────────────────────────────┘││           │                             │  │
││┌─────────────────────────────┐││          ││                             ││ │
│││1                            │││           ├─────────────────────────────┤  │
│││                             │││          ││2                            ││ │
│││            <A />            │││           │            <B />            │  │
│││                             │││          ││                             ││ │
│││                             │││           ├─────────────────────────────┤  │
││└─────────────────────────────┘││          ││3                            ││ │ Viewable
││┌─────────────────────────────┐││           │                             │  │  Screen
│││3                            │││          ││            <A />            ││ │
│││                             │││           │                             │  │
│││            <A />            │││          ││                             ││ │
│││                             │││           ├─────────────────────────────┤  │
│││                             │││          ││4                            ││ │
││└─────────────────────────────┘││           │            <B />            │  │
││┌─────────────────────────────┐││          ││                             ││ │
│││7                            │││           ├─────────────────────────────┤  │
│││                             │││          ││5                            ││ │
│││            <A />            │││           │            <B />            │  │
│││                             │││          └│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│┘ ◁
│││                             │││           ├─────────────────────────────┤
││└─────────────────────────────┘││           │6                            │  │
│└───────────────────────────────┘│           │            <B />            │  │
│┌───────────────────────────────┐│           │                             │  │
││<ViewPool type="B" />          ││           ├─────────────────────────────┤  │
││┌─────────────────────────────┐││           │7                            │  │
│││2                            │││           │                             │  │ Render Ahead
│││            <B />            │││           │            <A />            │  │   Distance
│││                             │││           │                             │  │
││└─────────────────────────────┘││           │                             │  │
││┌─────────────────────────────┐││           ├─────────────────────────────┤  │
│││4                            │││           │8                            │  │
│││            <B />            │││           │            <B />            │  │
│││                             │││           │                             │  ▼
││└─────────────────────────────┘││           └─────────────────────────────┘
││┌─────────────────────────────┐││                          │
│││5                            │││                          │
│││            <B />            │││                          │
│││                             │││                          ▼
││└─────────────────────────────┘││
││┌─────────────────────────────┐││
│││6                            │││
│││            <B />            │││
│││                             │││
││└─────────────────────────────┘││
││┌─────────────────────────────┐││
│││8                            │││
│││            <B />            │││
│││                             │││
││└─────────────────────────────┘││
│└───────────────────────────────┘│
└─────────────────────────────────┘
```


## Native Implementation

I'm still trying to figure this part of things out. I'll add some notes on this shortly.

The gist of it is:

### ViewPool Component

- Use `insertReactSubview` and `removeReactSubview` to intercept view insertions / deletions and hold
onto them in memory.

### Layout Component

**On iOS**:

- will probably implement `UICollectionViewLayout`
- will probably implement `UICollectionViewDelegate`
- will probably implement `UICollectionViewDataSource`
- will probably implement `UICollection​View​Data​Source​Prefetching`
- will dequeue cells from the child viewpools
- will use the prefetching behavior to message back to JS thread which new views it needs

**On Android**:

- will probably implement `RecyclerView.LayoutManager`
- will probably implement `RecyclerView.Adapter`

## Known problems with this approach

This implementation is not without issue, though I think the technique can be tweaked to handle
a lot of these use cases, and configuration options provided that minimize the problems.


### Item components do not maintain state

Since the React components themselves are reused here, if they are stateful, that state will
persist across different "items" in terms of the data source since they are being reused for
new items. It's worth noting that the recycling on the React side and the Native side are somewhat
decoupled, and one could easily allow for this as a configurable prop of the `RecyclingVerticalListView`
component. The only thing you would need to change would be to make the `key`s of the `ItemComponent`s
use a "stable id" from the data source. This could just be a `getItemKey` prop or something.  If
you don't actually need the components to be stateful (which might often be the case), being able to
reuse the component instances is a valuable performance optimization.


### Need to keep up with scrolling

The native side can't ask for new views in its pool synchronously, so it will have to be smart
about how many views it allocates in the pool, and will probably need a healthy
"render ahead distance". This is really going to be a problem with any RN listview implementation
that does not want to block scrolling, and doesn't want to render the whole list at once. The
options here are to 1) block the main thread, 2) show empty cells while we wait for the JS, 3)
block the scrolling.

There is an additional nuanced problem here in that we are reusing views, and so there might be
some cases where we are scrolling too fast for the JS to apply the new props for the views that
are being recycled, and so you see a different cell's rendered view in a new cell's position. This
can be handled (we can know at runtime if the view has been repopulated with the right data yet
or not), but all this does is put us back in the situation above where we have to choose what to
do in this situation.


## More things to think about

### Decoration Views

Both `UICollectionView` and `RecyclerView` have concepts of "decoration views". Perhaps we should
try to figure out a clean way to abstract that out to react so we can further leverage these
concepts.


### Drag and drop / reordering

Both platforms provide some support for dragging / dropping reordering and it seems like this would
be pretty easily integrated.


### Unique Layout components

I haven't fully thought about this, but I think some pretty unique Layout components could be made.
The "Masonry" grid is one such idea, another is an "Excel" like spreadsheet grid. There must be many
more. Leveraging this type of generic view recycling could be pretty interesting.



## Minor Details

Some methods above are moved down here for brevity, but rough implementations are here if
more understanding is desired:

```js
function poolItems<T>(getKey: ((item: T) => ComponentKey), items: Array<T>): Array<Pool> {
  const result = [];
  const pools = {};
  for (var i = 0; i < items.length; i++) {
    const item = items[i];
    const key = getKey(item);
    if (pools[key] !== undefined) {
      const pool = [];
      pools[key] = pool;
      result.push(pool);
    }
    pools[key].push(i);
  }
  return result;
}
```
