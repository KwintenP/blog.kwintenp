---
layout: post
cover: false
title: Using decorators and observables to implement retry
date:   2017-02-15
subclass: 'post'
categories: 'casper'
published: true
disqus: true
---

Last week, I was at a meetup in Ghent where I was talking with <a href="https://twitter.com/stefanlapers" target="_blank">Stefan Lapers</a> about programming languages in general. We started talking about writing your backend using either Java or node.js. We agreed that node.js has a lot of potential but in a lot of situations companies choose Java because of it's maturity. If you're using a microservices based architecture for example, you can rely heavily on spring cloud which does a whole bunch of stuff for you.
One element in spring cloud is hystrix. When you're doing a network call which is protected by hystrix, you can, by just adding an annotation, tell how many times you want to retry this if it fails and even provide a fallback if it fails entirely. 

When I was driving home later that night, I was thinking to myself that using observables and typescript decorators, it should be possible to implement something similar myself. The next morning I tried it out and about 30 minutes later I had a working version. 

## The example

In the example below, we have a service which fetches a number of Star Wars characters from the backend, at least it tries. It seems I kind of screwed up the implementation a little :).

```typescript
public getCharactersAndFail(): Observable<StarWarsCharacter[]> {
    return Observable.throw('Failing on purpose');
}
```
Instead of actually doing a backend call, I'm just returning an observable which will throw as soon as its subscribed to. This is of course not to handy but for demonstration purposes, it's quite ideal.
Using the decorator I created myself, we can make this code a little more resillient (we'll dive into how this decorator is constructed later on). 

```typescript
@retry(3, [{name: 'Obi Wan', birth_year: '1234', gender: 'Male'}])
public getCharactersAndFail(): Observable<StarWarsCharacter[]> {
    return Observable.throw('Failing on purpose');
}
```

I've added the `retry` decorator. The first argument it takes tells the decorator the times it should retry the backend call. The second parameter is the fallback that will be used if the backend keeps on failing.

The following is a gif of what happens when you try to run this code:

![example-gif](https://www.dropbox.com/s/bwoxrvgixrc40gv/Mar-04-2017%2017-22-23.gif?raw=1)

You can see that there are 3 tries before the method returns the fallback we defined. Pretty cool right. 

You can find the live working example <a href="http://blog-kwintenp-examples.surge.sh/retry" target="_blank">here</a>.

## The implementation

To implement this, there are two important things we need to do. First is implementing the logic to retry the call if it failed. We can leverage the compositiblity of streams for this. Next we need to extract that logic into a typescript decorator. 

### Observable composition

Retrying a subscription after it has failed is fairly easy using RxJS. We can use the `retryWhen` operator for this. Let's look at some code.

```typescript
private starWarsService: StarWarsService;

characters$: Observable<StarWarsCharacter> = 
	// we call the starWarsService to get the characters
	this.starWarsService
		.getCharacters()
		// we use the retryWhen operator to retry a number of times
		.retryWhen((errors: Observable<any>) => {
		  // we use the scan operator to count the number of tries
          return errors.scan((errorCount, err) => {
            console.log('Try ' + (errorCount + 1));
            if (errorCount >= 3) {
              throw err;
            }
            return errorCount + 1;
          }, 0).delay(1000);
        })
        // we catch the error if it keeps on failing and return
        // the fallback
        .catch(() => Observable.of(fallback));
```
I'm not going to go into the details of the RxJS implementation, that whould require a totally separate post.
What this code does however, is create an observable that, once subscribed to, will try to execute the backend call. If it fails, which it will in our case, it will re-execute it 2 more times with a single second delay in between. If it still fails, it will return the fallback we can define.

That's the exact logic we want our decorator to do. So let's see how we can extract this logic in the decorator.

### Creating a decorator

There are different types of decorators. We can put a decorator on a class, method, property or accessor method. In our case, we are going to use the method decorator.
To create a method decorator, we c