---
title: "✍️What I learned while writing Go"
date: 2019-04-07T20:48:30+01:00
tags: ["golang", "opinion"]
slug: "what-i-learned-writing-go"
---

I have been writing Go for about 6 months now at my new job. Coming from a Java
world, it is a joy to write Go. Some may call the language boring, because it is
easy to learn and easy to write. This is precisely what I love the most about
Go. There usually isn't more than one way of achieving the same result.

If you are able to work through the [gotour](https://tour.golang.org/welcome/1),
I believe you should be able to understand 99% of the open source Go code. There
are no threads, no pointer arithmetic operations, no semi-colons, no
inheritance, and well, no generics either (cough).

The things I have learned are mostly things _I have found_ interesting or
surprising. These are biased opinions, and more like notes to self.

### 1. Encapsulation in Go

[Names](https://golang.org/doc/effective_go.html#names) are important in Go.
Naming of an identifier decides its visibility (eg. like `private` or `public`
in Java). An identifier is exported (or `public`) if the first character of a
name is in upper case.

Confusingly, if you use [`golint`](https://github.com/golang/lint), you might
encounter this warning-
```go
exported func NewMagicPuppy returns unexported type *magicPuppy, which can be
annoying to use (golint)
```

I disagree with this linter warning. It's **totally fine** to return an
unexported type from an exported type. This is a common pattern of enforcing
encapsulation. This makes it impossible for someone to create a type like
`magicPuppy` manually and miss some of the set-up steps, and it becomes the
responsibility of the function `NewMagicPuppy` to create a correctly formed
magic puppy.

For eg. consider the hypothetical puppy package

```Go
package puppy
    
import "fmt"
    
// an unexported type can contain exported fields
type magicPuppy struct {
    // Name is exported, can be changed 
    Name string
    // breed is unexported, it cannot be changed
    breed string
}
    
func (p *magicPuppy) secretMagicBark() {
    fmt.Println("meow")
}
    
func (p *magicPuppy) Bark() {
    p.secretMagicBark()
    fmt.Println("woof boink boink 🐶")
}
    
func NewMagicPuppy() *magicPuppy {
     return &magicPuppy{"sparkles ✨", "Westie"}
}
    
func (p *magicPuppy) Description() string {
     return fmt.Sprintf("I am a %s and my magic puppy name is %s",
         p.breed, p.Name)
}
```
And it's corresponding main function-

```Go
package main
    
import (
    "fmt"
    "puppy"
)
    
func main() {
    var p *puppy.magicpuppy  // compilation error
    p := puppy.NewMagicPuppy()
    p.Bark()
    p.secretMagicBark()  // compilation error
    fmt.Println(p.breed) // compilation error
    fmt.Println(p.Name) // sparkles ✨
    p.Name = "🦄💖"
    fmt.Println(p.Description()) // I am a Westie and my magic puppy name is 🦄💖
}
```

### 2. context.Background()

Do not create a new `context.Background()` unless you want it to live for the
lifetime of your programme. If you don't watch out for its usage, you might
encounter memory leaks.

From the [documentation](https://blog.golang.org/context)-

```
// Background returns an empty Context. It is never canceled, has no deadline,
// and has no values. Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
```
You would be better off using `WithCancel` or `WithTimeout`.

Eg.
```Go
// this is very bad!! Don't ignore the context cancellation
ctx, _ := context.WithCancel(parentCtx)
    
ctx, cancel := context.WithCancel(parentCtx)
defer cancel()
// do some work
    
// you shouldn't wait for timeout, cancel immediately after work is complete
ctx, cancel := context.WithTimeout(parentCtx, time.Minute)
defer cancel()
// do some work
```

### 3. Errors as values

I find Rob Pike's blog on [error
handling](https://blog.golang.org/errors-are-values) very interesting. At the
same time, it frustrates me because now there are multiple accepted ways of
error handling.

For eg, this is the common pattern you would see in a Go codebase-

```Go
// daily tasks for the magic puppy
if err := pup.Train(); err != nil {
    return err
}
if err := pup.Walk(); err != nil {
     return err
}
if err := pup.Feed(); err != nil {
     return err
}

// Train is an example magic puppy training implmentation
func (p *magicPuppy) Train() error {
    if pup.isHungry() {
        return errors.New("Magic puppy 😿cannot train on a hungry stomach!"
    }
    // train
}
```

Most functions return an error type as the last returned value. But sometimes,
you can be caught off-guard if an operation does not return an error. With the
"error as value" pattern, the error is abstracted away while you finish doing a
series of operations, and then make the error check later. 

For eg.

```Go
// finish doing the series of tasks first, the next sequence of tasks
// become a no-op when the first failure is encountered
pup.Train()
pup.Walk()
pup.Feed()

if pup.err != nil {
    return pup.err
}
    
func (p *magicPuppy) Train() {
    // no-op if there was an error while executing daily tasks
    if pup.err != nil {
        return
    }
    if pup.isHungry() {
        // the first instance of the error is recorded
        pup.err = errors.New("Magic puppy 😿cannot train on a hungry stomach!"
    }
    // train
}

// ... similar implementations for functions Walk and Feed
```

I have been caught by this once when I forgot to check the error value 🙈

Honestly, I prefer the former error check pattern to the latter, and there's a
couple of reasons for it:

* The second approach causes a side-effect by writing to the error value. The
  implementation of `Train`, `Walk`, and `Feed` is now less obvious. You have to
  read their implementation to make the correct error check. (Or atleast I had
  to, when I missed it.)
* How does one decide what is the best time to decide that the sequence of
  operations is done, so that now you should error check?

Well, these three have been my small annoyances and gotchas. The focus of [Go
2](https://github.com/golang/go/wiki/Go2) seems to be on error handling, error
values and generics. I can't wait to find out what I'll learn in the next 6
months 😃
