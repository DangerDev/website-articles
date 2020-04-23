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

## The starting point

I work under the assumption that redundant code is acceptable up to a certain limit. If we are over-engineer into the wrong direction and immediately choose a generic solution, that generic solution may be worse than the original problem. So if you have two similar actions, then this is not a problem for me. Therefore I use the following example: Our application has to load data, and we need a loading indication and we want to display the last error in case of an error. Unfortunately we already have altered the existing code **three times**...

**Our actions:**

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

**Our reducers:**

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
  booksError: HttpErrorResponse | undefined;

  authors: string[];
  authorsStatus: Status;
  authorsError: HttpErrorResponse | undefined;

  thumbnails: string[];
  thumbnailsStatus: Status;
  thumbnailsError: HttpErrorResponse | undefined;
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

**Our effects:**


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

**Our selects:**

Let's finish the example, of course our selectors are also redundant:

```ts
export const selectBookState = createFeatureSelector<fromBook.State>(
  fromBook.bookFeatureKey
);

export const selectBooks = createSelector(
  selectBookState,
  booksState => booksState.books
);

export const selectBooksStatus = createSelector(
  selectBookState,
  booksState => booksState.booksStatus
);

export const selectBooksError = createSelector(
  selectBookState,
  booksState => booksState.booksError
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

## Conclusion

Okay, it should all be clear. This cannot go on. We should find solutions to reduce the amount of lines of code!