---
layout: post
title:  "Functiong programming style in C and JAVA"
date:   2014-12-10 14:05:37
categories: programming
tags: tags
author: "ScorpiusZ"
---

###SML language code

``` sml
(* Note this file is sort of misnamed.  It /does/ use closures.  It is code
   that we will compare to code in Java or C that does not use closures. *)

(* re-implementation of a list library with map, filter, length *)

datatype 'a mylist = Cons of 'a * ('a mylist) | Empty

fun map f xs =
    case xs of
	Empty => Empty
      | Cons(x,xs) => Cons(f x, map f xs)

fun filter f xs =
    case xs of
	Empty => Empty
      | Cons(x,xs) => if f x then Cons(x,filter f xs) else filter f xs 

fun length xs =
    case xs of
	Empty => 0
      | Cons(_,xs) => 1 + length xs

(* One fine way to double all numbers in a list *)
val doubleAll = map (fn x => x * 2)
(* One way to count Ns in a list *)
fun countNs (xs, n : int) = length (filter (fn x => x=n) xs)
```


### JAVA code with same fuction

```java
/ Note: This code compiles but has not been carefully tested. 
//       Bug reports welcome.


interface Func<B,A> {
    B m(A x);
}
interface Pred<A> {
    boolean m(A x);
}

class List<T> {
    T       head;
    List<T> tail;
    List(T x, List<T> xs) {
	head = x;
	tail = xs;
    }

    // * the advantage of a static method is it allows xs to be null
    //    -- a more OO way would be a subclass for empty lists
    // * a more efficient way in Java would be a messy while loop
    //   where you keep a pointer to the previous element and mutate it
    //   -- (try it if you do not believe it is messy)
    static <A,B> List<B> map(Func<B,A> f, List<A> xs) {
	if(xs==null)
	    return null;
	return new List<B>(f.m(xs.head), map(f,xs.tail));
    }

    static <A> List<A> filter(Pred<A> f, List<A> xs) {
	if(xs==null)
	    return null;
	if(f.m(xs.head))
	    return new List<A>(xs.head, filter(f,xs.tail));
	return filter(f,xs.tail);
    }

    // * again recursion would be more elegant but less efficient
    // * again an instance method would be more common, but then
    //   all clients have to special-case null 
    static <A> int length(List<A> xs) {
	int ans = 0;
	while(xs != null) {
	    ++ans;
	    xs = xs.tail;
	}
	return ans;
    }
}

class ExampleClients {
    static List<Integer> doubleAll(List<Integer> xs) {
	return List.map((new Func<Integer,Integer>() { 
		             public Integer m(Integer x) { return x * 2; } 
                         }),
			xs);
    }
    static int countNs(List<Integer> xs, final int n) {
	return List.length(List.filter(
		   (new Pred<Integer>() { 
		       public boolean m(Integer x) { return x==n; } 
		   }),
		   xs));
    }
}
```


### C code with same function 
```c
// Note: This code compiles but has not been carefully tested. 
//       Bug reports welcome.

#include <stdlib.h>
#include <stdint.h>
#include <stdbool.h>

// The key point is that function pointers are only code pointers

// Rather than create structs for closures, which would work fine,
// we follow common C practice of having higher-order functions 
// take explicit environment fields as another argument
//  -- if they don't, then they are much less useful 

// void* requires lots of unchecked conversions between types,
// but C has no notion of type variables

typedef struct List list_t;
struct List {
  void * head;
  list_t * tail;
};

list_t * makelist (void * x, list_t * xs) {
  list_t * ans = (list_t *)malloc(sizeof(list_t));
  ans->head = x;
  ans->tail = xs;
  return ans;
}

// as in the Java version, we show simple recursive solutions because
// the loop-based ones require mutation and previous pointers.
// But the more important point is the explicit env field passed to the
// function pointer
list_t * map(void* (*f)(void*,void*), void* env, list_t * xs) {
  if(xs==NULL)
    return NULL;
  return makelist(f(env,xs->head), map(f,env,xs->tail));
}

list_t * filter(bool (*f)(void*,void*), void* env, list_t * xs) {
  if(xs==NULL)
    return NULL;
  if(f(env,xs->head))
    return makelist(xs->head, filter(f,env,xs->tail));
  return filter(f,env,xs->tail);
}

int length(list_t* xs) {
  int ans = 0;
  while(xs != NULL) {
    ++ans;
    xs = xs->tail;
  }
  return ans;
}

// clients of our list implementation:
// [the clients that cast from void* to intptr_t are technically not legal C, 
//  as explained in detail below if curious]

// awful type casts to match what map expects
void* doubleInt(void* ignore, void* i) {
  return (void*)(((intptr_t)i)*2);
}

// assumes list holds intptr_t fields
list_t * doubleAll(list_t * xs) {
  return map(doubleInt, NULL, xs);
}

// awful type casts to match what filter expects
bool isN(void* n, void* i) {
  return ((intptr_t)n)==((intptr_t)i);
}

// assumes list hold intptr_t fields
int countNs(list_t * xs, intptr_t n) {
  return length(filter(isN, (void*)n, xs));
}

/*
  The promised explanation: Some of the client code above tries to use
  a number for the environment by using a number (intptr_t) where a
  pointer (void *) is expected.  This is technically not allowed: any
  pointer (including void*) can be cast to intptr_t (always) and the
  result can be cast back to the pointer type, but that is different
  than starting with an intptr_t and casting it to void* and then back
  to intptr_t.

  It appears there is no legal, portable way to create a number that
  can be cast to void* and back.  People do this sort of thing often,
  but the resulting code is not strictly portable.  So what should we
  do for our closures example besides ignore this and write
  non-portable code using int or intptr_t?
  
  Option 1 is to use an int* for the environment, passing a pointer to the
  value we need.  That is the most common approach and what we need to do for
  environments with more than one value in them anyway.  For the examples above,
  it would work to pass the address of a stack-allocated int, but that works
  only because the higher-order functions we are calling will not store those
  pointers in places where they might be used later after the stack variable
  is deallocated.  So it works fine for examples like map and filter, but would
  not work for callback idioms.  For those, the pointer should refer to memory
  returned by malloc or a similar library function.

  Option 2 is to change our approach to have the higher-order functions use
  intptr_t for the type of the environment instead of void*.  This works in
  general since other code can portably cast from any pointer type to
  intptr_t.  It is a less standard approach -- one commonly sees void* used
  as we have in the code above.
*/
```


