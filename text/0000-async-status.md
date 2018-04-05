# RFC 0: Refactor AsyncStatus from Type Code to Class

## Summary
We currently use an eunum type property to track the state of an async operation in our client-side applications. It is necessary for the UI to be able to present realizations of this state (loading, error, etc.). To do so the current value of the async property must be directly compared to a possible value in question, for instance `userLoaded = fetchUserStatus === AsyncStatus.Done`. By performing Fowler's _Replace Type Code with Class_ refactoring an instance of `AsyncStatus` can be queried directly for it's state: `userLoaded = AsyncStatus.isDone()`. This abstraction could be added to the gca-react-components libary making it available for use in all our client-side JavaScript projects.

## Motivation
Tracking the async status of CRUD operations in our front-end apps is a common need. It is necessary for our microapps to reliably know async status in order to hide or reveal new parts of the UI, display loading spinners, error dialogs, or kick-off other async operations. Our code idioms encapsulating this state have evolved over time, becoming more concise.

### Legacy 1
Legacy implementations used three boolean flag properties for every operation we wanted to track:

```
isMessagespecSaving: false,
isMessagespecSaveError: false,
isMessagespecSaveSuccess: false,
isMessagePreviewFetching: false,
isMessagePreviewFetchingError: false,
isMessagePreviewFetchingSuccess: false,
```
While this worked, it was verbose and potentially error prone, as there was no intrinsic guarantee the flags could not be in an inconsistent state (both error and success `true` for instance).

### Legacy 2
An improvement to the first design was to introduce a single state type property for each async status we wanted to track:

```
const ASYNC_INITIAL = 'initial';
const ASYNC_STARTED = 'started';
const ASYNC_DONE = 'done';
const ASYNC_FAILED = 'failed';

...

messagespecSaveState: ASYNC_INITIAL,
messagespecPreviewFetchState: ASYNC_INITIAL,
```

However it was still possible to mis-spell a constant, which is potentially a tricky error to catch as JavaScript would allow assigning an `undefined` value.

### Legacy 3 (Current state of the art)
TypeScript improves on this by being able to use typed enums instead of string constants:

```
const enum AsyncStatusType {
  Initial = 'ASYNC_INITIAL',
  Started = 'ASYNC_STARTED',
  Done = 'ASYNC_DONE',
  Failed = 'ASYNC_FAILED',
};

...

messagespecSaveState: AsyncStatusType.Initial,
messagespecPreviewFetchState: AsyncStatusType.Initial,
```

By enforcing the types of state properties it's no longer possible to assign an undefined value inadvertently — as long as you're using TypeScript at least.

### Issues with current solution
The current solution works very well for TypeScript projects. My personal quibble with it is that determining the async state at the point you need to know it requires a slightly verbose comparision test:

```
gsSelectors.fetchState(state) === AsyncStatus.Failure
```

Sometimes I have written selectors to abstract this a bit, but it is overhead to write the implementations and tests for all async states I want to abstract:

```
function isSavingGSGroup(state: State): boolean {
  return state.guestShareApproval.saveState === AsyncStatus.Open;
}
```

I've begun feeling the selectors aren't worth writing. Others may not share or be bothered by my quibble (the explicit comparison). But the TypeScript solution is not applicable to non-TypeScript, pure JS projects. A solution which provides ease of use and some degree of type safety even in non-TypeScript projects could be beneficial.

## Detailed Design
Implement async state properties as specific instances of an `AsyncStatus` class.  These are assigned to static properties of the class.

I'm representing this as JavaScript below for greatest applicability.

```
const ASYNC_INITIAL = 'initial';
const ASYNC_STARTED = 'started';
const ASYNC_DONE = 'done';
const ASYNC_FAILED = 'failed';

export default class AsyncStatus {
	constructor(status) {
		this.status = status;
	}
	
	toString() {
		return `[AsyncStatus: ${this.status}]`;
	}
	
	isInitial() {
		return this.status === ASYNC_INITIAL;
	}
	
	isStarted() {
		return this.status === ASYNC_STARTED;
	}
	
	isDone() {
		return this.status === ASYNC_DONE;
	}
	
	isFailed() {
		return this.status === ASYNC_FAILED;
	}
}

AsyncStatus.Initial = new AsyncStatus(ASYNC_INITIAL);
AsyncStatus.Started = new AsyncStatus(ASYNC_STARTED);
AsyncStatus.Done = new AsyncStatus(ASYNC_DONE);
AsyncStatus.Failed = new AsyncStatus(ASYNC_FAILED);
```

Specific `AsyncStatus` instances are assigned as the value of async state properties (such as in a redux reducer function):

```
  .case(updateUser.started, state => ({ ...state, updateUserStatus: AsyncStatus.Started }))
  .case(updateUser.done, (state, { params, result }) => (
    { ...state, updateUserStatus: AsyncStatus.Done, user: { ...state.user, ...params } }
  ))
  .case(updateUser.failed, state => ({ ...state, updateUserStatus: AsyncStatus.Failed }))
  .case(resetUpdateUser, state => ({ ...state, updateUserStatus: AsyncStatus.Initial }))
```

Therefore an ansync property can be easily and cleanly interrogated for which state it is currently in. This provides a measure of type-like safety for even pure JS projects, since a run-time exception would be thrown if trying to call a method on an accidentally undefined property:

```
function getFetchUserState(state: State): AsyncStatus {
  return state.myProfile.fetchUserStatus;
}
```
```
const mapState = (state: State) => ({
  isUpdatingUser: selectors.getUpdateUserStatus(state).isStarted(),
  isUpdateUserDone: selectors.getUpdateUserStatus(state).isDone(),
  isUpateUserFailed: selectors.getUpdateUserStatus(state).isFailed(),
  ...
});
```

## Drawbacks/Consequences
* The current approach of using a string value for async states is probably a little easier to understand, due to a lack of general familiarity with this refactoring
* This refactoring provides less value with TypeScript, since TypeScript can enforce the type safety of using a enum at author time
* When used with redux it's a little odd to assign a class instance as a state property, although in my experience it has worked fine — the value of the `status` property is what is seen in the dev tools
* The `merge()` function in gca-react-components does not currently handle instance properties properly

## Alternatives
* A set of pure typescript types could be defined which would provide the same conditional ergonomics in TypeScript only projects
* A similar solution could probably be arrived at without relying on a `class`, but it's not clear to me there would be any practical benefits to that
* The obvious alternative would be to stick with our current state of the art, which is clear and works well for TypeScript projects

## Unresolved Questions
* Unknown where this would live in gca-react-components — possibly under _utils/_?

## Status
Proposed