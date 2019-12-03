# Redux level reusable components
## Introduction
There are a lot of ways to reuse code in `React`. In this guide we consider reusing code on `Redux` level.
There is a simple application with two counters. The counters have two props: `count` and `color`. They could be _fetched_ and _saved_ (`session storage` is used as `API` mock).

![alt app_demo](https://github.com/nrg-soft/eas-frontend/blob/master/doc/redux-level-reusing-counters-app.gif?raw=true)

_The appliction in codesandbox: [Reusable redux counters application](https://codesandbox.io/s/reusable-redux-phi5g)_

## Component logic
The main parts of a _redux_ component are: 
- `component` - to display
- `actions` - to dispatch
- `reducer` - to describe how the state reacts to dispatching actions
- `selectors` - to get data from state

### Actions (`src/shared/actions.js`)
To control the component we have to create several actions:
- `setData` - to initialize component
- `increment`
- `decrement`
- `setColor`

Every action should have data to identify which counter component exactly should be updated. We use `action.meta.counterId` for such purpose. An action could be written this way:
```javascript
setColor(counterId, color) {
    return {
      type: counterTypes.SET_COLOR,
      payload: { color },
      meta: { counterId }
    };
  }
```

### Reducer (`src/shared/reducer.js`)
Each reducer describes single counter logic. In the following code reducer describes what state should be returned after `counterTypes.SET_COLOR` action.
```javascript
export const conterInitialState = {
  count: 0,
  color: "green"
};

export const counterReducer = (state = conterInitialState, action) => {
  switch (action.type) {
    ...
    case counterTypes.SET_COLOR: {
      const { color } = action.payload;
      return R.set(R.lensProp("color"), color, state);
    }
    ...
  }
};
```

### Selectors (`src/shared/selectors.js`)
In this example we have only one selector which returns counter state as it is:
```javascript
const getCounterState = state => state;
```
### Component (`src/shared/components/index.js`)
There is a simple pure component, which only maps props into react elements:
```jsx
const Counter = ({
  count,
  color,
  disabled = false,
  increment,
  decrement,
  setColor
}) => {
  return (
    <div>
      <div style={{ fontSize: 22, color }}>{count}</div>
      <div>
        <button disabled={disabled} onClick={decrement}>
          -
        </button>
        <button disabled={disabled} onClick={increment}>
          +
        </button>
        <select
          disabled={disabled}
          value={color}
          onChange={e => setColor(e.target.value)}
        >
          <option value="green">green</option>
          <option value="red">red</option>
          <option value="blue">blue</option>
        </select>
      </div>
    </div>
  );
};
```

## Preparing component to be reusable
In order to reuse the counter component we should do some stuff before.
For each part of the `redux` logic we should create kind of _factory_.
### Actions
We already created actions with `counterId` argument, and put it into `meta`. Let's create a helper function to map each action to action with defined `counterId`. It is done to skip `counterId` each time we dispatch actions (especially when we dispatch actions right from the counter  component) .
```javascript
export const getCounterActions = counterId => {
  return R.map(action => (...args) => action(counterId, ...args))(
    counterActions
  );
};
```
### Reducer
For the reducer we use similar approach and just put `counterId` into _closure_:
```javascript
export const makeCounterReducer = counterId => (
  state = conterInitialState,
  action
) => {
  if (action.meta && counterId === action.meta.counterId) {
    return counterReducer(state, action);
  }

  return state;
};
```
As you can see this _new_ reducer will be called only if `action.meta.conterId` matches `counterId` from _closure_.
### Selectors
We use `globalizeSelectors` function and put `lens` into _closure_ to get certain part of state.
```javascript
const getCounterState = state => state;

export const getCounterSelectors = lens =>
  globalizeSelectors(lens, { getCounterState });
```
### Component
And same _factory_ approach for component:
```javascript
export const makeCounterComponent = (componentId, selectors) =>
  connect(
    state => selectors.getCounterState(state),
    getCounterActions(componentId)
  )(Counter);
```
Now the counter _redux_ component can be reused.
## Reusing
Let's render two independent counters.
Counter ids:
```javascript
export const counter1Id = "coutner1Id";
export const counter2Id = "coutner2Id";
```
Reducer:
```javascript
combineReducers({
  counter1: makeCounterReducer(counter1Id),
  counter2: makeCounterReducer(counter2Id)
})
```
Selectors:
```javascript
export const counter1Selectors = getCounterSelectors(
  R.lensPath(["feature_state", "counter1"])
);
export const counter2Selectors = getCounterSelectors(
  R.lensPath(["feature_state", "counter2"])
);
```
Components:
```javascript
const Counter1 = makeCounterComponent(counter1Id, counter1Selectors);
const Counter2 = makeCounterComponent(counter2Id, counter2Selectors);
```
Rendering:
```jsx
<Counter1 />
<Counter2 />
```
Component management from feature's `saga` after fetching data from the server:
```javascript
yield put(counterActions.setData(counter1Id, response[0]));
yield put(counterActions.setData(counter2Id, response[1]));
```
Getting data from state before sending to the server:
```javascript
const counter1 = yield select(counter1Selectors.getCounterState);
const counter2 = yield select(counter2Selectors.getCounterState);
```
## Conclusion
This approach could be used for simple widgets like filters, searches etc...

It is simple and important to make such components covered by unit tests.
```javascript
test.each([["green", "red", "red"], ["green", "green", "green"]])(
    "on setColor set new color",
    (color, newColor, expectedColor) => {
      expect(
        counterReducer({ color }, counterActions.setColor(newColor))
      ).toEqual({
        color: expectedColor
      });
    }
  );
```
It is simple to understand which parts _feature reducer_ includes.
```javascript
const featureReducer: combineReducers({
  searchField: makeSearchFieldReducer(searchFieldId),
  objectivesFilter: makeSearchFilterReducer(objectivesFilterId),
  taxonomiesFilter: makeSearchFilterReducer(taxonomiesFilterId),
  labelsFilter: makeSearchFilterReducer(labelsFilterId),
  selectedFilter: makeCheckboxFilterReducer(selectedFilterId),
  sidebar: sidebarReducer,
  sharingComponent: makeSharingComponentReducer(sharingComponentId),
  ...
})
```
Which is suitable for _overview pages_.
