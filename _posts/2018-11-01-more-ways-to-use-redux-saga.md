---
layout: single
author_profile: false
title: "More ways to use Redux Saga"
date:   2018-11-01 21:17:08 -0500
categories: [tech]
---

A while ago I wrote [an intro piece on Redux Saga](https://medium.com/20spokes-whiteboard/getting-familiar-with-redux-sagas-5b5856c97a75){:target="_blank"} where I described how to get your feet wet using sagas for some basic async requests. Here’s an update with a few more ways you can use sagas for common patterns we come across in our apps.

### Cancelling a saga

Say we’re using a saga to upload media. Perhaps a media upload is taking a long time, and we’d like the user to have the option of cancelling the upload.

Redux saga has an effect combinator called `race` where we can dispatch an action to cancel a saga.

[Race](https://redux-saga.js.org/docs/api/){:target="_blank"} can be used like any other effect creator and receives an object with specific attributes, in this case `task` and `cancel`. Here’s a saga function using `race` that will start an `uploadMedia` saga but cancel it if it receives an action called `CANCEL_UPLOAD`:

```javascript
function* raceUpload(action) {
  yield race({
    task: call(uploadMedia, action),
    cancel: take('CANCEL_UPLOAD')
  })
}
```

Wherever you’re listening for your `uploadMedia` action, replace the called saga function with the new race function.

```javascript
function* mediaSaga() {
  yield takeEvery('UPLOAD_MEDIA_SAGA', raceUpload) /* used to be uploadMedia */
}
```

Now you can dispatch an action called `CANCEL_UPLOAD` from any component.

```javascript
// In a React component

cancelUpload = () => this.props.dispatch({ type: 'CANCEL_UPLOAD' })

render() {
  return <button onClick={this.cancelUpload}> Cancel Upload </button>
}
```

### Yield Blocks + Iterators

Say we’re passed a list of media files to a saga that each need to be uploaded with separate requests. Redux Saga’s `all` effect creator can be passed an iterator that will return an array of the responses for each request.

```javascript
// saga that uploads media file
function* uploadMedia(action) {
  try {
    const file = action.payload
    const response = yield call(api.post, '/upload', file)
    if (response.success) return true;
    if (response.failure) return false;

  catch (e) {
    return false
  }
}

// saga that calls uploadMedia for every file given
function* uploadMultiple(action) {

  const files = action.payload.files // [ file1, file2, file3 ]
  const responses = yield all(files.map((file) => uploadMedia({ payload: file })))
  const allSuccess = responses.filter(res) => res == false).length == 0

  return allSuccess
}
```

Just some food for thought. Every time I use Redux Saga I feel like I’m only scratching the surface of all it can do.

I’m interested in how other people use Redux Saga for common patterns in their applications. If there’s a pattern you use a lot, let me know!