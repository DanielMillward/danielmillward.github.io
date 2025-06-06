+++
date = '2025-05-05T09:07:33-06:00'
draft = false
title = 'Go Crash Course'
categories = ["Golang"]
+++

Want to write [Go](https://go.dev/) quickly and already have some experience in other languages? This post is for you!

This post is essentially a cleaned-up version of my notes of [The Little Go Book](https://www.openmymind.net/The-Little-Go-Book/) with some additional clarifications I found helpful. If you have the time, I would highly recommend reading The Little Go Book instead of this 😅
## Getting Started

To get Go (a.k.a. GoLang for googlability) code running on your computer, follow the [official getting started guide](https://go.dev/doc/tutorial/getting-started). It boils down to:

1. Download and install the Go CLI tool
2. In a new folder, run `go mod init github.com/yourUsername/yourModuleName`
3. Make a file `hello.go` and write some code, e.g.
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```
4. Run `go run .` to build and run the program.

## How Go organizes dependencies

Go calls each dependency/application a "module". Each module can contain one or more "packages", which are just logical groups of source code files (essentially namespaces). For example, the module `golang.org/x/net` has a bunch of useful networking functions, grouped into packages like "net/html", "net/http2", etc. You add new dependencies to a module to your application by running `go get your/dependency/moduleName`.

Running `go mod init your/module/name` generates a new module with that name. If you plan to publish this module online for others to download, the name *must* be the path to where they can download it. For example, running `go get github.com/fatih/color` downloads the module at that URL. If you don't intend to use your module as a dependency, you can name it whatever you want. 

> To be overly specific, running `go get github.com/fatih/color` makes a GET request to `https://proxy.golang.org/github.com/fatih/color` or - if it's the first time it's been queried - `github.com/fatih/color?go-get=1` for information on where the actual repo is. It then downloads it from that url to the location set by your computer's `GOMODCACHE` environment variable. On windows, this is `C:\Users\YourUsername\go\pkg\mod`. It can then be used from there inside any module on your computer.

In the example code above the package specified was named `main`, which tells Go to make it into a binary when you run `go run .`. Libraries usually don't have a main package.

## Rundown of Go

I'll go through what I think are the more important parts of Go from the perspective of someone coming from, say, JS or Typescript.

- Unlike Javascript, Go is an explicitly typed and compiled language. 
- `:=` lets you declare variables without writing out the type. Instead of typing `var age int = 29`, you can instead do `age := 29`.
- Parameters are passed by value, i.e. by a copy. Go does have pointers, but it's a garbage-collected language so it's not too scary.
	- Rule of Thumb: Pass by value for fundamental types, pass by (const) reference for structs
- Go uses structs to organize data as opposed to classes:
```go
type Vertex struct {
	X int
	Y int
}

// Add a function to the struct:
func (v *Vertex) MyVertexFunc() {...}
// Use a pointer in the first parenthesis if you need to modify the struct,
//use a regular reference otherwise. See: https://stackoverflow.com/questions/25382073/defining-golang-struct-function-using-pointer-or-not
...
myVertex := Vertex{100,200}
myDefaultVertex := Vertex{} // Any empty spots fill the parameters with the default, in this case, 0s.
...
myVertexXVal := myVertex.X // 100
myVertexFuncVal := myVertex.MyVertexFunc()

```
- Go doesn't have constructors. Instead, make a function that returns your struct.
- `goku := new(Saiyan)` is the same as `goku := &Saiyan{}`
- If you add a parameter to a struct without giving it a name, you can access any functions that parameter's type has directly from the outside struct. This is called [embedded types](https://gobyexample.com/struct-embedding).
- You can make a constant size array with `var scores [10]int`. This isn't commonly used.
- Using arrays is more often done via *slices*. You use a slice as an abstraction over an array. It's helpful to think of it as a window into an underlying array. A slice is made of 3 parts: a pointer to the underlying array, the length of the slice, and the capacity/size of the underlying array starting from the pointer. 
- Make a slice with the initial values or via `make([]type, length, initialCapacity)`. Omitting the initial capacity defaults it to the length.
```go
letters := []string{"a", "b", "c", "d"}
s := make([]string, 5)
// s == []byte{0, 0, 0, 0, 0}
```
- Slices are useful since they have an `append()` function that automatically resizes the underlying array if it exceeds the initial capacity:
```go
s = append(s, "new", "elements", "to", "add")
```
- Under the hood, Go uses the `copy(dest, src)` function for append by just copying everything to a bigger array as needed.
- Write for/in loops as: ` for index, value := range arrayName {}`
- Make a dictionary (called a map in Go) with 
```go
myMap := make(map[string]int) // key is string, value is int
myMap["route"] = 66
j := myMap["nonExistent string"] // j == ""
i, ok := myMap["nonExistent string"] // ok == false, i == ""
delete(myMap, "route")
for key, value := range myMap { ... }
```
 - One package == one directory
 - Uppercase stuff are "exported" out from a package, i.e. can be accessed from importing the package. Lowercase is private to it.
 - `go install` is for binaries (e.g. CLI tools), `go get` is for dependencies.
 - Define an interface with:
```go
type Logger interface {
RequiredFunc(funcParam string)
} // Interfaces are only a collection of methods, not parameters.
```
 - No need to explicitly say something implements an interface: the compiler will figure that out.
 - Make regular, non-struct related functions with:
 ```go
 func process(varName varType) returnType { ... }
```
 - A big conceptual shift Go does is that you explicitly return errors as opposed to throwing them. This is done with `return errors.New("messages")` and importing the "errors" package.
 - If something truly unexpected happens and we don't know how to handle it, then we or the system can call `panic()` which acts similar to throwing an error in JS. Note that using panic frequently is *not* recommended! Panics are reserved for program-crashing situations.
 - Safely close files with `defer fileObj.Close()`. `defer` always runs after the enclosing function returns regardless of whether the parent function panics or not. A special function that can be used inside a `defer`'d function is `recover()`, which as the name implies recovers from a panic. This is useful for server middleware to send the client a 500 rather than keep them in the dark:
```go
defer func() {
	if r := recover(); r != nil {
		fmt.Println("Recovered. Error:\n", r)
	}
}()
```
 - Go has a default standard formatter, so no more arguing over what linter to use. Run it with `go fmt ./..`.
 - If you instantiate a variable only to be used within an if/else block, you can inline it with semicolons:
```go
if err := process(); err != nil {
return err
}
```
 - If you aren't sure of the type of a parameter, you can use a sort of "any" type via the `interface{}` type, since everything implements the 0 methods of a blank interface:
```go
func add(a interface{}) {...}
```
 - Strings are made of "runes", which can be one or more bytes. A rune is what you would intuitively call a single character, avoiding the messiness that C has of only using one byte per character when all of Unicode exists.
 - Go has first-class functions. You can make a type a function and set it to a variable:
```go
type add func(a int, b int) int

func main() {
	var a add = func(a int, b int) int {
		return a + b
	}
	s := a(5, 6)
	fmt.Println("Sum", s)
}
```
 - Go has direct support for multithreading via [goroutines](https://go.dev/tour/concurrency/1). To run a function in its own thread, simply call `go yourFunction()`. Go has a data structure built specifically for passing data to multiple goroutines: the channel. Think of it as a safe buffer with built-in lock functionality. Create a channel with `c := make(chan int)`, then put data into the channel with `c <- newData()`. Extract data from the channel with `v := <-c`. 
	 - You can also limit the size of the channel with `c := make(chan int, 10)`
- Another useful tool to use with goroutines is the `select` block. Think of it as telling the program "chill here until you can do one of these cases without panicking". You can use this to chill until there's space in a channel to put new data or to chill until there's data to read from a channel. If you don't want to block, you can put a default:
```go
// Waiting for space to put into the channel
for { // infinite loop if have a stream of data
	select {
	case c<- rand.Int():
		//optional code here
	default:
		// what to do when the channel is full and you're dropping the data. If no default is given, select blocks the main thread.
	}
}
...
// Waiting for data to come into the channel
for i := 0; i < 2; i++ {
        select {
        case msg1 := <-c1:
            fmt.Println("received", msg1)
        case msg2 := <-c2:
            fmt.Println("received", msg2)
        }
}
```

That's it for the introduction to Go. The next post should hopefully be a crash course in writing web servers in Go, following the very helpful [Let's Go](https://lets-go.alexedwards.net/) book.