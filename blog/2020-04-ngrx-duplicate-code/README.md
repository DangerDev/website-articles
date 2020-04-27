---
title: "NgRx: How to avoid duplicated code"
author: Johannes Hoppe
mail: johannes.hoppe@haushoppe-its.de
published: 2020-04-26
keywords:
  - NgRx
  - Redux
language: en
thumbnail: xxx.jpg
---

I have noticed more than once that it can easily happen that duplicated code occurs in an architecture with NgRx. 
In this post I want to highlight a few ideas that helped me working with NgRx/Redux. 

## A regular app – Our starting point

I work under the assumption that redundant code is acceptable up to a certain limit. If we are over-engineer into the wrong direction and immediately choose a generic solution, that generic solution may be worse than the original problem. So if you have two similar actions, then this is not a problem for me. Therefore I use the following example: Our application has to load data, and we need a loading indication and we want to display the last error in case of an error. Unfortunately we already have altered the existing code **three times**...

### Our actions

```ts
export const loadBooks = createAction(
  '[Book] Load Books'
);

export const loadBooksSuccess = createAction(
  '[Book] Load Books Success',
  props<{ data: Book[] }>()
);

export const loadBooksFailure = createAction(
  '[Book] Load Books Failure',
  props<{ error: HttpErrorResponse }>()
);

// second duplication 🤨

export const loadAuthors = createAction(
  '[Book] Load Authors'
);

export const loadAuthorsSuccess = createAction(
  '[Book] Load Authors Success',
  props<{ data: string[] }>()
);

export const loadAuthorsFailure = createAction(
  '[Book] Load Authors Failure',
  props<{ error: HttpErrorResponse }>()
);

// third duplication 😞

export const loadThumbnails = createAction(
  '[Book] Load Thumbnails'
);

export const loadThumbnailsSuccess = createAction(
  '[Book] Load Thumbnails Success',
  props<{ data: string[] }>()
);

export const loadThumbnailsFailure = createAction(
  '[Book] Load Thumbnails Failure',
  props<{ error: HttpErrorResponse }>()
);
```

### Our reducers

I know it's horrible to read, but we have to go through it now.

```ts
export enum Status {
  NotSubmitted,
  Submitting,
  Successful,
  Failure
}

export interface State {
  books: Book[];
  booksStatus: Status;
  booksError: HttpErrorResponse;

  authors: string[];
  authorsStatus: Status;
  authorsError: HttpErrorResponse;

  thumbnails: string[];
  thumbnailsStatus: Status;
  thumbnailsError: HttpErrorResponse;
}

export const initialState: State = {
  books: [],
  booksStatus: Status.NotSubmitted,
  booksError: undefined,

  authors: [],
  authorsStatus: Status.NotSubmitted,
  authorsError: undefined,

  thumbnails: [],
  thumbnailsStatus: Status.NotSubmitted,
  thumbnailsError: undefined,
};


export const reducer = createReducer(
  initialState,

  on(BookActions.loadBooks, state => ({
    ...state,
    booksStatus: Status.Submitting,
  })),

  on(BookActions.loadBooksSuccess, (state, { data: books }) => ({
    ...state,
    books,
    booksStatus: Status.Successful,
    booksError: undefined
  })),

  on(BookActions.loadBooksFailure, (state, { error: booksError }) => ({
    ...state,
    books: [],
    booksStatus: Status.Failure,
    booksError
  })),

  // second duplication 🤨

  on(BookActions.loadAuthors, state => ({
    ...state,
    authorsStatus: Status.Submitting,
  })),

  on(BookActions.loadAuthorsSuccess, (state, { data: authors }) => ({
    ...state,
    authors,
    authorsStatus: Status.Successful,
    authorsError: undefined
  })),

  on(BookActions.loadAuthorsFailure, (state, { error: authorsError }) => ({
    ...state,
    authors: [],
    authorsStatus: Status.Failure,
    authorsError
  })),

  // third duplication 😞

  // ... okay, let's stop this!
);
```

### Our effects


As for effects, the amount of lines is tolerable, but duplicated code is still duplicated code.

```ts
@Injectable()
export class BookEffects {

  loadBooks$ = createEffect(() => {
    return this.actions$.pipe(
      ofType(BookActions.loadBooks),
      concatMap(() =>
        this.service.getBooks().pipe(
          map(data => BookActions.loadBooksSuccess({ data })),
          catchError(error => of(BookActions.loadBooksFailure({ error }))))
      )
    );
  });

  // second duplication 🤨

  loadAuthors$ = createEffect(() => {
    return this.actions$.pipe(
      ofType(BookActions.loadAuthors),
      concatMap(() =>
        this.service.getAuthors().pipe(
          map(data => BookActions.loadAuthorsSuccess({ data })),
          catchError(error => of(BookActions.loadAuthorsFailure({ error }))))
      )
    );
  });

  // TODO: add third duplication 😞
  });

  constructor(private actions$: Actions, private service: DataService) {}
}
```

### Our selects

Let's finish the example, of course our selectors are also redundant:

```ts
export const selectBookState = createFeatureSelector<fromBook.State>(
  fromBook.bookFeatureKey
);

export const selectBooks = createSelector(
  selectBookState,
  (booksState: fromBook.State) => booksState.books
);

export const selectBooksStatus = createSelector(
  selectBookState,
  (booksState: fromBook.State) => booksState.booksStatus
);

export const selectBooksError = createSelector(
  selectBookState,
  (booksState: fromBook.State) => booksState.booksError
);

// second duplication 🤨

export const selectAuthors = createSelector(
  selectBookState,
  authorsState => authorsState.authors
);

export const selectAuthorsStatus = createSelector(
  selectBookState,
  authorsState => authorsState.authorsStatus
);

export const selectAuthorsError = createSelector(
  selectBookState,
  authorsState => authorsState.authorsError
);

// third duplication 😞

// [...]
```

### Conclusion

Okay, it should all be clear. This cannot go on. We should find solutions to reduce the amount of lines of code!

**☞ You can find [the full source code of the regular app](https://github.com/angular-schule/ngrx-duplicated-code/tree/master/projects/regular-app) at GitHub.**


## 1. First Idea: Combining reducers and effects

Mike Ryan, a well-known core developer of NgRx, recommends having explicit actions in the code. He even goes further by not reusing actions at all. The best way to find out more about this is to listen to the great talk ["Good Action Hygiene with NgRx"](https://youtu.be/JmnsEvoy-gY). He teaches that its principially a good practice to have unique actions for every component. Please have a look at this stream of actions:

```ts
// not that great 🤨
[Book] Load Books
[Book] Load Books Success
[Book] Load Books
[Book] Load Books Failure
```

And instead of this, compare the following stream:

```ts
// much more explicit 🤩
[Book List]    Load Books
[Book API]     Load Books Success
[Book Details] Load Books
[Book API]     Load Books Failure
```

In the second example it is much easier to understand from which source the action was dispatched. NgRx is designed to support this style very well. So it accepts multiple actions as arguments for both reducers and effects:

```ts
// reducer
on(BookActions.loadBooksList
   BookActions.loadBooksDetails, state => ({
  ...state,
  booksStatus: Status.Submitting,
})),

// effects
loadBooks$ = createEffect(() => {
  return this.actions$.pipe(
    ofType(BookActions.loadBooksList,
           BookActions.loadBooksDetails),
    concatMap(() =>
      this.service.getBooks().pipe(
        map(data => BookActions.loadBooksSuccess({ data })),
        catchError(error => of(BookActions.loadBooksFailure({ error }))))
    )
  );
});
```



This idea can be adopted. Let’s see how it looks like to use the same technique for independet actions. Instead of nine reducer creators we could use three creators to handle all cases:

```ts
export const reducer = createReducer(
  initialState,

  on(BookActions.loadBooks,
     BookActions.loadAuthors,
     BookActions.loadThumbnails, (state, { type }) => ({
    ...state,
    ...(type === BookActions.loadBooks.type      ? { booksStatus:      Status.Submitting } : {}),
    ...(type === BookActions.loadAuthors.type    ? { authorsStatus:    Status.Submitting } : {}),
    ...(type === BookActions.loadThumbnails.type ? { thumbnailsStatus: Status.Submitting } : {})
  })),

  on(BookActions.loadBooksSuccess,
     BookActions.loadAuthorsSuccess,
     BookActions.loadThumbnailsSuccess, (state, { type, data }) => ({
    ...state,

    ...(type === BookActions.loadBooksSuccess.type ? {
      books: data,
      booksStatus: Status.Successful,
      booksError: undefined
    } : {}),

    // still some kind of second duplication 🤨

    ...(type === BookActions.loadAuthorsSuccess.type ? {
      authors: data,
      authorsStatus: Status.Successful,
      authorsError: undefined
    } : {}),

      // still some kind of third duplication 😞

    ...(type === BookActions.loadThumbnailsSuccess.type ? {
      thumbnails: data,
      thumbnailsStatus: Status.Successful,
      thumbnailsError: undefined
    } : {})
  })),

  on(BookActions.loadBooksFailure,
     BookActions.loadAuthorsFailure,
     BookActions.loadThumbnailsFailure, (state, { type, error }) => ({
    ...state,

    ...(type === BookActions.loadBooksFailure.type ? {
      books: [],
      booksStatus: Status.Failure,
      booksError: error
    } : {}),

    ...(type === BookActions.loadAuthorsFailure.type ? {
      authors: [],
      authorsStatus: Status.Failure,
      authorsError: error
    } : {}),

    ...(type === BookActions.loadThumbnailsFailure.type ? {
      thumbnails: [],
      thumbnailsStatus: Status.Failure,
      thumbnailsError: error
    } : {})
  }))
);
```

We can do the same for the effects. Instead of three effects, we can also use only one, but this one will still need some branches to handle to different cases:

```ts
@Injectable()
export class BookEffects {

  loadItems$ = createEffect(() => {
    return this.actions$.pipe(
      ofType(BookActions.loadBooks,
             BookActions.loadAuthors,
             BookActions.loadThumbnails),
      concatMap(({ type }) => defer(() => {
        switch (type) {
          case BookActions.loadBooks.type: return this.service.getBooks();
          case BookActions.loadAuthors.type: return this.service.getAuthors();
          case BookActions.loadThumbnails.type: return this.service.getThumbnails();
        }}).pipe(
          map((data: any) => {
            switch (type) {
              case BookActions.loadBooks.type: return BookActions.loadBooksSuccess({ data });
              case BookActions.loadAuthors.type: return BookActions.loadAuthorsSuccess({ data });
              case BookActions.loadThumbnails.type: return BookActions.loadThumbnailsSuccess({ data });
            }
          }),
          catchError(error => {
            switch (type) {
              case BookActions.loadBooks.type: return of(BookActions.loadBooksFailure({ error }));
              case BookActions.loadAuthors.type: return of(BookActions.loadAuthorsFailure({ error }));
              case BookActions.loadThumbnails.type: return of(BookActions.loadThumbnailsFailure({ error }));
            }
          })
      ))
    );
  });

  constructor(private actions$: Actions, private service: DataService) { }
}
```

### Conclusion

We see that this works in principle. But we can't ignore it, by defintion, we really have way too much actions. Also for the reducer and the effects the number of lines of code is about the same. We just moved the complexity into the inner functions. In the end, we still have nine cases for the reducers and nine cases for the effects. If you have looked carefully, you will have noticed that I used `any` at one point. In fact, it is not so easy to remain type-safe over the whole time. Another important point is that with this technique we can not cross the boundaries of a NgRx feature. 

**☞ You can find [the full source code of this app](https://github.com/angular-schule/ngrx-duplicated-code/tree/master/projects/example-combining) at GitHub.**

Let's try it another varriation!


## 2. Second Idea: Action Subtyping

Explicit actions are a fine thing, but we are getting a lot of lines of code just because we defined so many actions. Now we want to do the opposite and fight the duplicated code by using generic actions, or to be specific: actions with a subtype. And we are aware that the folowing example can be seen as anti-pattern.

Such actions could look like this:

```ts
export type ActionKinds = 'books' | 'authors' | 'thumbnails';

export const loadItems = createAction(
  '[Book] Load Items',
  props<{ kind: ActionKinds }>()
);
export const loadItemsSuccess = createAction(
  '[Book] Load Items Success',
  props<{ kind: ActionKinds, data: Book[] | string[] }>()
);
export const loadItemsFailure = createAction(
  '[Book] Load Items Failure',
  props<{ kind: ActionKinds, error: HttpErrorResponse }>()
);
```

One thing is quite obvious. We have fewer actions and therefore no more duplicated code at this place.
We can adapt the previously used reducer accordingly:

```ts
export const reducer = createReducer(
  initialState,

  on(BookActions.loadItems, (state, { kind }) => ({
    ...state,
    ...(kind === 'books' ? { booksStatus: Status.Submitting } : {}),
    ...(kind === 'authors' ? { authorsStatus: Status.Submitting } : {}),
    ...(kind === 'thumbnails' ? { thumbnailsStatus: Status.Submitting } : {})
  })),

  on(BookActions.loadItemsSuccess, (state, { kind, data }) => ({
    ...state,

    ...(kind === 'books' ? {
      books: data,
      booksStatus: Status.Successful,
      booksError: undefined
    } : {}),

    ...(kind === 'authors' ? {
      authors: data,
      authorsStatus: Status.Successful,
      authorsError: undefined
    } : {}),

    ...(kind === 'thumbnails' ? {
      thumbnails: data,
      thumbnailsStatus: Status.Successful,
      thumbnailsError: undefined
    } : {})
  })),

  on(BookActions.loadItemsFailure, (state, { kind, error }) => ({
    ...state,

    ...(kind === 'books' ? {
      books: [],
      booksStatus: Status.Failure,
      booksError: error
    } : {}),

    ...(kind === 'authors' ? {
      authors: [],
      authorsStatus: Status.Failure,
      authorsError: error
    } : {}),

    ...(kind === 'thumbnails' ? {
      thumbnails: [],
      thumbnailsStatus: Status.Failure,
      thumbnailsError: error
    } : {})
  }))
);
```

Okay, that's at least a little bit shorter now, but the result is not that impressive.s
The biggest gain we have with the effect, here we can now remove some lines:

```ts
@Injectable()
export class BookEffects {

  loadItems$ = createEffect(() => {
    return this.actions$.pipe(
      ofType(BookActions.loadItems),
      concatMap(({ kind }) => defer(() => {
        switch (kind) {
          case 'books': return this.service.getBooks();
          case 'authors': return this.service.getAuthors();
          case 'thumbnails': return this.service.getThumbnails();
        }}).pipe(
          map((data) => BookActions.loadItemsSuccess({ kind, data })),
          catchError(error => of(BookActions.loadItemsFailure({ kind, error })))
      ))
    );
  });

  constructor(private actions$: Actions, private service: DataService) {}
}
```

### Conclusion

At least for the effects, the changes are worth it. The complexity is much smaller than before, and we can still handle all casess. But we have to be aware that the stream of actions ([Redux DevTools](https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd)) is hard to read because all action have the same type. We also have to keep in mind that we still can't reuse the code beetween different NgRx features. 

**☞ You can find [the full source code of this app](https://github.com/angular-schule/ngrx-duplicated-code/tree/master/projects/example-action-subtyping) at GitHub.**

## 3. Third Idea: Action Subtyping with nested state

The effects (or more precisely, it's actually only one effect) are very pleasant and short. But how can we make the reducer smaller?

Of course, we can! 😇 An elegant solution is to generalize the state a bit. We could combine the three recurring objects (`data`, `status` and `error`) to a single object and access it via the ES6 bracket notation `[]`.

```ts
export interface SubmittableItem<TData> {
  data: TData;
  status: Status;
  error: HttpErrorResponse;
}

const initialSubmittableItem = {
  data: [],
  status: Status.NotSubmitted,
  error: undefined,
};

export interface State {
  books: SubmittableItem<Book[]>;
  authors: SubmittableItem<string[]>;
  thumbnails: SubmittableItem<string[]>;
}

export const initialState: State = {
  books: { ... initialSubmittableItem },
  authors: { ... initialSubmittableItem },
  thumbnails: { ... initialSubmittableItem },
};


export const reducer = createReducer(
  initialState,

  on(BookActions.loadItems, (state, { kind }) => ({
    ...state,
    [kind]: {
      ...state[kind],
      status: Status.Submitting
    }
  })),

  on(BookActions.loadItemsSuccess, (state, { kind, data }) => ({
    ...state,
    [kind]: {
      ...state[kind],
      data,
      status: Status.Successful,
      error: undefined
    }
  })),

  on(BookActions.loadItemsFailure, (state, { kind, error }) => ({
    ...state,
    [kind]: {
      ...state[kind],
      data: [],
      status: Status.Failure,
      error
    }
  }))
);
```

There we go. That looks a lot shorter. We can now generilise the selectors, too.

```ts
export const selectBookState = createFeatureSelector<fromBook.State>(
  fromBook.bookFeatureKey
);

export const selectItems = createSelector(
  selectBookState,
  (booksState: fromBook.State, props: { kind: BookActions.ActionKinds }) => booksState[props.kind].data
);

export const selectItemsStatus = createSelector(
  selectBookState,
  (booksState: fromBook.State, props: { kind: BookActions.ActionKinds }) => booksState[props.kind].status
);

export const selectItemsError = createSelector(
  selectBookState,
  (booksState: fromBook.State, props: { kind: BookActions.ActionKinds }) => booksState[props.kind].error
);
```

For the selectors we use the additional object with the name `props`. It's usage looks like this:

```ts
books$ = this.store.select(BookSelectors.selectItems, { kind: 'books'});
booksStatus$ = this.store.select(BookSelectors.selectItemsStatus, { kind: 'books'});
booksError$ = this.store.select(BookSelectors.selectItemsError, { kind: 'books'});
```

### Conclusion

By using actions with subtypes, we will have less redundant actions by definition.Furthermore, we use a nested state, so that we no longer need case distinctions. Gradually we are getting closer to the desired result.

Of course we can come to a similar results by using other approaches, e.g. by composing the keys (`books`, `booksStatus`, `booksError` and so on) at runtime. 

**☞ You can find [the full source code of this app](https://github.com/angular-schule/ngrx-duplicated-code/tree/master/projects/example-action-subtyping-nested) at GitHub.**

## 4. Fourth Idea: Higher-order functions

So far we have only shared the code within one single NgRx feature. As soon as we want to use the same logic again in another feature, we can't get very far with the shown approaches. Since everything is driven by functions, we would need some helper that creates those functions for us. To archive this, we should take a look at higher-order functions. This approach is well-known and established, even in the original Redux documentation it is described as [higher-order reducers](https://redux.js.org/recipes/structuring-reducers/reusing-reducer-logic#customizing-behavior-with-higher-order-reducers). 


We will adapt our code to the API from the **[Entity Adapter (@ngrx/data)](https://ngrx.io/guide/entity/adapter)**.
Here you call the method `createEntityAdapter<T>` once and then you comfortably get a number of factories, which do the work for us. In contrast to @ngrx/data we will also create actions and reducers directly. 




Unfortunately there is currently not way to concatenate two or more string literal types to a single string literal type in TypeScript! ([see this answer on stackoverflow](https://stackoverflow.com/a/57300713)) So we can't have a type `Load Books` and extend it to `Load Books Success` without falling back to the generic `string` type.



**☞ You can find [the full source code of this app](https://github.com/angular-schule/ngrx-duplicated-code/tree/master/projects/example-higher-order) at GitHub.**


## X. @ngrx/data

https://ngrx.io/guide/data