---
id: user-components
title: User components
---

A user component is intended to be written just like any other React component. Let's start with a simple Hero component like so:

```jsx
const Hero = ({title}) => {
  return (
    <div>
      <h2>{title}</h2>
    </div>
  )
}
```
Now, let's actually get the component to work with the editor. The `useNode` hook provides us several information and methods related to the corresponding `Node` that manages the component.

```jsx
const { connectors: {connect, drag}, setProp, ...collected } = useNode((node) => {});
```

Additionally we can pass configuration values via the static `craft` property:
```jsx
const Hero = () => {...}
Hero.craft = {
  defaultProps: {},
  rules: {
    canDrag: () => true,
    outgoing: () => true,
    incoming: () => true
  },
  related: {}
}
```

We'll explain what each of these values are in the following sections.

## Connectors
The first thing we would want to do is to actually let Craft.js to manage the DOM for our component. 

- `connect`: specifies the DOM that represents the user component. The dimensions of the DOM specified will taken into account during drag/drop.
- `drag`: specifies the DOM element that should be made draggable. When the user drags this element will be considered as dragged the entire component, thus moving the entire component to the drop location.

```jsx
const Hero = ({title}) => {
  const { connectors: {connect, drag} } = useNode();

  return (
    <div ref={connect(drag)}>
      <h2>{title}</h2>
    </div>
  )
}
```

## Props manipulation
You've probably seen page editors where you could directly interact with the components and manipulate them. For instance, drag to resize an image or visually edit a text. This is easily achievable with Craft.js as well.

Since components are managed by their corresponding `Node` which contains information including the component's props, thus we can call the `setProp` method to update the prop values stored in the `Node`. In turn, this will re-render the component with its updated values.

```jsx
const Hero = ({title}) => {
  const { connectors: {connect, drag}, setProp } = useNode();

  return (
    <div ref={connect(drag)}>
      <h2 contentEditable={true} onKeyUp={(e) => {
        setProp(props => {
          props.text = e.target.innerText;
        })
      }}>{title}</h2>
    </div>
  )
}
```

In the above example, we have created updated our `h2` element to be content editable, and added an event handler to update the `text` prop as the user visually enters in a new value.

## Collecting Node's state
The information stored in a corresponding `Node` could be useful in helping you build more usable components. We can retrieve information from a `Node` by passing a collector function to the `useNode` hook, every time the values we retrieved via the collector function changes, our component will re-render. This is very much similar to Redux's `connect` pattern.


For instance, let's say we would like to enable the content editable text from the previous section only when the user has actually clicked on our component: 

```jsx
const Hero = ({title}) => {
  const { connectors: {connect, drag}, setProp, isClicked } = useNode((node) => ({
    isClicked: node.events.selecteded
  }));

  return (
    <div ref={connect(drag)}>
      <h2 contentEditable={isClicked} onKeyUp={(e) => {
        setProp(props => {
          props.text = e.target.innerText;
        })
      }}>{title}</h2>
    </div>
  )
}
```

## Default Props
While it's not necessary as we could simply define default parameters (eg: ES6 defaults) directly within our components, however these default values will not actually be recorded into the component's corresponding `Node`, which could leave us with a lots of empty prop values when we wish to retrieve the `Node` for a component when building other parts of our editor (eg: a Toolbar for editing a component's values).

To prevent that, we can explicitly specify default prop values via the `craft.defaultProps` like so:

```jsx
const Hero = ({text}) => { /** same as previous example **/ }
Hero.craft = {
  defaultProps: {
    text: "Hi there!"
  }
}
```

## Specify drag/drop rules
You may want to restrict when your components could be dragged, or what goes in and out of your component. These rules can be specified in the static `craft.rules`:

- `canDrag(currentNode)` : Specifies if a component can be dragged. Applicable only to components whose corresponding `Node` are direct children of a `Canvas`.
- `canMoveIn(incomingNode, currentNode)`: Decide if an incoming Node can be dropped into the current component. Applicable only to components whose corresponding `Node` is a `Canvas`.
- `canMoveOut(outgoingNode, currentNode)` : Decide if a child Node can be dragged out of the current component. Applicable only to components whose corresponding `Node` is a `Canvas`.


Let us write a (pretty strange) rule for our Hero component, whereby users can only drag it if they change the `text` prop to "Drag": 
```jsx
const Hero = ({text}) => { /** same as previous example **/ }
Hero.craft = {
  defaultProps: { /** same as previous example **/ },
  rules: {
    canDrag: (node) => !!node.data.props.text == "Drag"
  }
}
```


## Related components
What happens if you need to design some component to complement our  user component ? For instance, if we were planning on building a Toolbar somewhere in our page editor, we would like the Toolbar to display a bunch of text inputs to allow the user the edit the currently selected component. It would be great if we could retrieve a specific component that has all the relevant inputs for the user to edit the currently selected component.


This is where related components becomes useful. These components share the same corresponding `Node` as the actual user component, hence the `useNode` hook that we have using all these while will be made available to these components as well. 

```jsx
const Hero = ({text}) => { /** same as previous example **/ }
Hero.craft = {
  related: {
    toolbar: HeroToolbarSettings
  }
}

const HeroToolbarSettings = () => {
  const { setProp, text } = useNode((node) => ({
    text: node.data.props.text
  }));

  return (
    <div>
      <h2>Hero settings</h2>
      <input 
        type = "text" 
        value={text} 
        onChange={e => 
          setProp(prop => prop.text = e.target.value) 
        }
       />
    </div>
  )
}
```
Now, let's say we have a Toolbar component somewhere in our editor. We can easily retrieve the related component as such:

```jsx
const Toolbar = () => {
  const { selectededNodeId, toolbarSettings } = useEditor((state) => ({
    selectededNodeId : state.eventselectedcted,
    toolbarSettings:  state.nodes[state.events.selecteded].related.toolbar
  }));
  return (
    <div>
      <h2>My Awesome Toolbar</h2>
      {
        selectededNodeId && toolbarSettings ? 
          React.createElement(toolbarSettings)
        : null
      }
    </div>
  )
}

```

## Defining droppable regions
Let's say we are creating a Hero component that has 2 sections where the users could drop other user elements into. 

```jsx
const Hero = ({title}) => {
  return (
    <div>
      <h2>{title}</h2>
      <section>
        <h3>Yo</h3>
      </section>
      I need some coffee to continue writing this
      <section>
        <h2>Hi</h2>
      </section>
    </div>
  )
}
```

Previously, we discussed how `<Canvas />` creates a droppable Canvas node; that concept is applied universally in Craft.js. Hence, we just have to wrap our `section` elements in the example above with a `Canvas`. 

```jsx {5,9}
const Hero = ({title}) => {
  return (
    <div>
      <h2>{title}</h2>
      <Canvas id="Header" is="section">
        <h3>Yo</h3>
      </section>
      I need some coffee to continue writing this
      <Canvas id="Footer" is="section">
        <h2>Hi</h2>
      </section>
    </div>
  )
}
```

But wait, what if I want to define rules for these 2 new droppable regions ? 

The `is` is used to specify the `type` of the User Element for the Canvas Node. Hence, instead of specifying a simple DOM element which we don't have much control over - let's specify a brand new User Component.


```jsx {1-23,29,33}
const HeroHeader = ({children}) => {
  return (
    <section>
      {children}
    </section>
  )
}

HeroHeader.craft = {
  rules : {...}
}

const HeroFooter = ({children}) => {
  return (
    <section>
      {children}
    </section>
  )
}

HeroFooter.craft = {
  rules : {...}
}

const Hero = ({title}) => {
  return (
    <div>
      <h2>{title}</h2>
      <Canvas id="Hero" is={HeroHeader}>
        <h3>Yo</h3>
      </section>
      I need some coffee to continue writing this
      <Canvas id="Footer" is={HeroFooter}>
        <h2>Hi</h2>
      </section>
    </div>
  )
}
```

`HeroHeader` and `HeroFooter` are user components, so we can now design and configure them just like any other user components. On that note, don't forget that you will need to include these in the `resolver` as well.