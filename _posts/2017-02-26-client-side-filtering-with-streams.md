---
layout: post
cover: false
title: Client side filtering using streams
date:   2017-02-15
subclass: 'post'
categories: 'casper'
published: true
disqus: true
---

I have been coaching people in using RxJS for a while now. During this time, I've noticed that the hardest part for people to learn is not the API, concept or operators but the paradigm switch. Thinking reactively is not something that comes easily and you really have to get your hands dirty to get there. 

Below I will show a piece of code that is used to do some basic client side filtering. It's a snippet of code from somebody that I was coaching which perfecly shows how someone just starting with RxJS often handles this situation. Later on, we will update this to show how you would implement this with the reactive paradigm.

### Example case

The example application we will be using is really easy. We have service which fetches a number of Star Wars characters from the swapi.com API. We will show these in a list and provide a select element to filter the fetched characters based on the gender.
The screen looks like this:

![example-app](https://www.dropbox.com/s/2s9e877rpdaa5w0/Screenshot%202017-02-25%2011.16.57.png?raw=1)

### Client side filtering without streams

First we are going to look at an example where we implement the client side filtering without streams. Of course, we are going to use a stream to fetch the data from the backend, but afterwards, the implementation will be imperative.

```typescript
// keep a local list of all the characters
characters: Array<StarWarsCharacter>;
// keep a list of all the filtered characters
filteredCharacters: Array<StarWarsCharacter>;

constructor(private starWarsService: StarWarsService) {}

// at startup time, we fetch the characters and save them
ngOnInit() {
  this.starWarsService.getCharacters()
    .subscribe((fetchedCharacters) => {
        this.characters = fetchedCharacters;
        this.filteredCharacters = fetchedCharacters;
    })
}

// when the filter value changes, we filter the local list of
// characters and save the result to the
// filteredCharacters array
filterChanged(value: string)
  if (value === "All") {
    this.filteredCharacters = this.characters;
  } else {
    this.filteredCharacters =
       this.characters.filter(
            (character: StarWarsCharacter) => {
               character.gender.toLowerCase() === value.toLowerCase()
            }
       );
  }
}
```
At startup time, we fetch the characters and save them in two local arrays, `characters` and `filteredCharacters`. When the filter actually changes, we use the local copy of the characters to filter out all the correct ones and create a new array. We then assign this new array to the `filteredCharacters` reference.
The component's html looks like this:

```html
<h1>Client side filtering without streams example</h1>
<div class="row">
  <div class="col-sm-3">
    <!-- Component that holds the select and throws an event -->
    <!-- when it changes -->
    <app-gender-filter (filterChange)="filterChanged($event)">
    </app-gender-filter>
  </div>
  <div class="col-sm-9"></div>
  <div class="col-sm-6">
    <!-- component that holds the list and displays the -->
    <!-- characters -->
    <app-character-list [characters]="filteredCharacters">
    </app-character-list>
  </div>
</div>

```

While this all works perfectly, it's not the best solution possible with the reactive paradigm in mind. We have to hold a local copy of the characters array, which kind of bugs me.
It's also not really flexible. Here, we are fetching the characters via a backend call. This will thus only hold one result. But what if it's an observable we get from Firebase? In that case the characters array can change as well. To be able to update the view properly when the characters change, we would also have to keep a local copy of the filter at any given time to update the `filteredCharacters` array reference accordingly.

Using streams up until the template of our component, this can all be fixed an be extremely flexible.

### Client side filtering with streams

Let's first of all try to think what should happen by thinking in streams of data. We will then try to reason about how we can use these streams, combine them and create a result.

If you think about it, we have two inputs that might change our view. On the one hand, we have our list of characters, which should be displayed. And on the other hand we have the dropdown which might filter this list. 

Let's try to create an ASCII marble diagram of what these streams might look like:

characters$:    -----R|					R = list of characters
filter$:        A--M---F---M--A---		A = All, M = Male, F = Female

The resulting stream we want is one that holds an array with the filtered characters. If you think about the two input streams we just created, this resulting stream is actually a combination of both of the input streams.
If we have characters, we want to combine these with a filter and display them onto the screen. If the filter changes, we want to re-execute this logic. If we would have new characters (think about a stream from Firebase), we would also want to re-execute this logic.

It turns out that RxJS provides us with a perfect operator to do something like this: `combineLatest`. This operator merges streams by executing a projector function on the latest values of these streams. Let's take a look at the code!

```typescript
export class ClientSideFilterComponent implements OnInit {
  filter$: Observable<string>;
  characters$: Observable<StarWarsCharacter[]>;
  filteredCharacters$: Observable<StarWarsCharacter[]>;

  constructor(private starWarsService: StarWarsService) {
  }

  ngOnInit() {
    // We create a stream ourselves to map an event form the child
    // component to a stream of 'filter values'
    this.filter$ = new BehaviorSubject("All");

    // we keep the stream containing our characters
    this.characters$  = this.starWarsService.getCharacters();

    // we create a new stream based on the two input
    // streams we defined
    this.filteredCharacters$ = this.createFilterCharacters(
            this.filter$,
            this.characters$
    );
  }

  public createFilterCharacters(
            filter$: Observable<string>,
            characters$: Observable<StarWarsCharacter[]>) {
    // We combine both of the input streams using the combineLatest
    // operator. Every time one of the two streams we are combining
    // here changes value, the project function is re-executed and
    // the result stream will get a new value.
    return characters$.combineLatest(
      filter$, (characters: StarWarsCharacter[], filter: string) => {
        // this is the project function where we imperatively
        // implement the filtering logic
        if (filter === "All") return characters;
        return characters.filter(
                (character: StarWarsCharacter) =>
                    character.gender.toLowerCase()
                        === filter.toLowerCase()
        );
      });
  }

  filterChanged(value: string) {
    // Everytime we have new value, we pass it to the filter$
    this.filter$.next(value);
  })
}
```
