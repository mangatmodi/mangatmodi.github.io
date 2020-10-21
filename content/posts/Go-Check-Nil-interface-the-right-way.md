---
title: "Go:Check Nil Interface the Right Way"
date: 2020-10-21T09:43:25+02:00
draft: false
tags: 
  - go
---

I had a simple problem. I have a function which takes a parameter of type `interface{}`. I need to check it for `nil`. Any seasoned Go developer will know that a simple `i==nil` check will not work because interfaces in Go contains both type and value. So you can have cases when —

1. Type is null-able(like map, pointer etc) and value is nil
1. Type itself is nil (of course value will be nil)

A `nil` will only work with option 2 above, as for option 1 a variable still have some type. As any good engineer I googled and found the following solution prominently suggested everywhere.

**Popular Solution**
```go
func isNil(i interface{}) bool {
   return i == nil || reflect.ValueOf(i).IsNil()
}
```
This seems to be working for the following case -

```go

type Animal interface {
  MakeSound() string
}

type Dog struct{}

func (d *Dog) MakeSound() string {
return "Bark"
}

func main() {
   var d *Dog = nil
   var a Animal = d
   fmt.Println(isNil(a))
}
```

We have a pointer to Dog type, which implements Animalinterface with the pointer receiver.

> *However the code fails when we give non pointer argument*

```go
type Cat struct{}

func (c Cat) MakeSound() string {
   return "Meow"
}

func main() {
   var c Cat
   var a Animal = c
   fmt.Println(isNil(a))
}
```
The `isNil` function will fail with a panic - `panic: reflect: call of reflect.Value.IsNil on struct Value`

It happens because we are trying to check if a struct value is nil, which is logically wrong for Go. One could argue that why would you give a struct value for nil check, but hey bugs happen! At least:

1. I would like to fail gracefully.
1. Or It shall not fail. If a function accepts an argument at compile time, it should be ready to give correct result for that argument(and not fail). This makes things more predictable.


If isNil takes an interface parameter, it should work for all types of interfaces — pointers, struct, channel etc..

**Better Version**

So I ended up writing following version

```go
func isNilFixed(i interface{}) bool {
   if i == nil {
      return true
   }
   switch reflect.TypeOf(i).Kind() {
   case reflect.Ptr, reflect.Map, reflect.Array, reflect.Chan, reflect.Slice:
      return reflect.ValueOf(i).IsNil()
   }
   return false
}
```
Now, if the type is kind pointer, it checks for nil, else it is of course false(not nil)

**Best Solution**

However the best solution for our example would be to have following

```go
func isNilBetter(i Animal) bool {
   var ret bool
   switch i.(type) {
   case *Dog:
      v := i.(*Dog)
      ret = v == nil
   case Cat:
      ret = false
   }
   return ret
}
```

Here the compiler will not even allow us to compile if we gave any other case in above, not even Dog as struct. So it is type safe, more readable as we know which types we can get and no reflection! But it is not always possible to have this solution because

1. We might have dozens of implementations of an interface.
1. We might have accidentally implemented an interface (duck-typing), so we miss checks for that(I miss exhaustiveness of switch in Kotlin)
1. We simply don’t know which interface we are going to use.

The complete example can be found here — https://gist.github.com/mangatmodi/06946f937cbff24788fa1d9f94b6b138
