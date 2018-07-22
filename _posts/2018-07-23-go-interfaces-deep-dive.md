---
layout: default
comments: true
title: Go interfaces deep dive
---

> Empty boxes. He really should flatten these... so they can be recycled.

# Diving into some Golang interface internals

I was recently doing some memory optimisations on a Go program which contained some large slices of interface references and I thought it would be good to understand exactly what was going on with them. In my case, there were 10,000 slices, each holding 5,120 interface references.

How much memory is that, and where is it being spent?

## The interface struct
In Go, an interface reference is a pair of pointers. The first pointer points to the type definition for whatever type of value has been stored, and the second pointer points to the value itself.

Here's the definition of an `iface` from the Go runtime [Go](https://github.com/golang/go/blob/0a7ac93c27c9ade79fe0f66ae0bb81484c241ae5/src/runtime/runtime2.go#L143)

```go
type iface struct {
        tab  *itab
        data unsafe.Pointer
}
```

On a 64 bit architecture, a pointer is obviously 8 bytes, meaning an interface reference is 16 bytes in size.

So, an individual slice of 5,120 interface references will cost 80KiB. But that's not the whole story, what if we actually store some values in it?

## Storing simple values and dumping them out

When you put a simple value, like a number into an interface, it has to be pointed to by the `data` member of the `iface` struct. That means that the value needs to be alloced onto the heap.

Using the `unsafe` package, it is possible to print out the contents of the `iface` table, as differently sized values are assigned, which is a good way of confirming understanding of what's going on.

First, let's assume we have a simple slice of `interface{}` that we can use to store references to any value we like.

```go
a := make([]interface{},5)
```

Like interfaces, slices also have a definition which we can access through cunning use of the `unsafe` module. In this case, the slice header is defined in the reflect package as `reflect.SliceHeader`, you can see it [here](https://github.com/golang/go/blob/47be3d49c7d7ff77e675b0d0fb78c05fdb43dee2/src/reflect/value.go#L1800). Its definition is:

```go
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

Armed with this knowledge, we can get to the data pointer within the slice and dump the values out as hex to see what is going on, a bit like a poor-mans memory dumping tool.

The data points to `uintptr`, which is a type which is large enough to hold a pointer for the target architecture, i.e., it is either 4 bytes or 8 bytes for 32 or 64 bit architectures respectively.

As our slice is a slice of 5 interface references, this array of `uintptr`s will actually be an array of 5 `iface` structures, each of which has a size equivalent to two `uintptr`s, meaning we have a total of 10 `uintptr`s in this `Data` array.

The follow snippet will extract the 4th `uintptr` from the `Data` member of the slice, which will therefore be the `data` field of the 2nd `iface` struct which has been stored in the slice.

```go
var _uv uintptr
i := 3
sliceHeader := (*reflect.SliceHeader)(unsafe.Pointer(&a))
dumpedVal := *(*uintptr)(unsafe.Pointer(sliceHeader.Data + unsafe.Sizeof(_uv) * uintptr(i)))
```

Ok, armed with this trick, we can assign some test values into the slice of `interface{}` then dump the contents of the slice to see what is going on.

Here's the complete loop, which is also on the [Go playground](https://play.golang.org/p/xG7jqldK2jZ)


```go
package main

import (
	"fmt"
	"unsafe"
	"reflect"
)

var _uv uintptr

func main() {
	fmt.Println("Hello, playground")
	a := make([]interface{},5)
	a[0] = bool(true)
	a[1] = bool(false)
	a[2] = uint16(0x1234)
	a[3] = float64(1.23)
	a[4] = string("test")
	sliceHeader := (*reflect.SliceHeader)(unsafe.Pointer(&a))
	
	fmt.Printf("size of uintptr = %d\n",unsafe.Sizeof(_uv))
	for i := 0; i<10; i++ {
		dumpedVal := *(*uintptr)(unsafe.Pointer(sliceHeader.Data + unsafe.Sizeof(_uv) * uintptr(i)))
		fmt.Printf("offset 0x%x : 0x%016x",i,dumpedVal)
		if i&1 != 0 && dumpedVal != 0 {
			deref := *(*uint64)(unsafe.Pointer(dumpedVal))
			fmt.Printf(" (*%p = 0x%016x)",unsafe.Pointer(dumpedVal),deref)
			if i == 9 {
				dumpedVal = (uintptr)(deref&0xffffffff)
				deref = *(*uint64)(unsafe.Pointer(dumpedVal))
				fmt.Printf(" (*%p = 0x%016x)",unsafe.Pointer(dumpedVal),deref)
			}
		}
		fmt.Printf("\n")
	}
}
```

This outputs the following:

```
Hello, playground
size of uintptr = 4
offset 0x0 : 0x00000000000edde0
offset 0x1 : 0x00000000001738e1 (*0x1738e1 = 0x0807060504030201)
offset 0x2 : 0x00000000000edde0
offset 0x3 : 0x00000000001738e0 (*0x1738e0 = 0x0706050403020100)
offset 0x4 : 0x00000000000ee720
offset 0x5 : 0x0000000000127ee8 (*0x127ee8 = 0xfffffffe00001234)
offset 0x6 : 0x00000000000edee0
offset 0x7 : 0x00000000001280a8 (*0x1280a8 = 0x3ff3ae147ae147ae)
offset 0x8 : 0x00000000000ee620
offset 0x9 : 0x00000000001280b0 (*0x1280b0 = 0x00000004000fe380) (*0xfe380 = 0x6575727474736574)
```

Let's break that down. First, I output the size of the `uintptr`, this is just so I know what architecture Go playground is running on, which in this case is 32-bit.

I then looped through the 10 `uintptr`s in the slice (remember, 5 `iface` each of which has 2 pointers, so 10 pointers in total) and printed each value. In the output, each even numbered "offset" output will correspond to the `tab` member of an `iface`, and each odd numbered "offset" output which will correspond to the `data` member of the same `iface`.

So the first value in the slice was assigned with `a[0] = bool(true)`, so we expect an `iface` with a pointer to a `bool` type, and a data value which is pointing to a piece of memory containing a boolean true value (ie `0x1`, as `bool` is 1 byte in Go).

Looking at the output, "offset 0x0" has the value `0x00000000000edde0`, which therefore must be a type pointer to the `bool` type. This is followed by a pointer `0x00000000001738e1` which is our pointer to a boolean value, which we expect to be pointing to a value of `true` (`0x1`). For these `tab` values, I also dereference the memory address and print the 8 bytes I find there, so we can see what the `data` field is actually pointing to. In this case it prints: `(*0x1738e1 = 0x0807060504030201)`. The last byte of the dereferenced 8 byte memory block is `0x01`, so `0x1738e1` points to a byte value of `0x01`, i.e. a value of `true`. No surprises so far.

Following the rest of the output, we see that `a[1] = bool(false)` results in an `iface.tab` value of `0x00000000000edde0` (which is the same type pointer, as element 0, as again it's a `bool`). The value is `(*0x1738e0 = 0x0706050403020100)`, so this time it's pointing to a byte value of `0x00`, i.e. `false`.


`a[2] = uint16(0x1234)` results in `(*0x127ee8 = 0xfffffffe00001234)`, i.e. it points to `0x00001234` as we expect.

`a[3] = float64(1.23)` results in  `(*0x1280a8 = 0x3ff3ae147ae147ae)`. If you take the value `0x3ff3ae147ae147ae` and decode it as a IEEE754 double precision floating point value, perhaps by using an [online converter](http://www.binaryconvert.com/result_double.html?hexadecimal=3FF3AE147AE147AE) you will see that it holds our float value of `1.23` as expected.

Finally, we come to the string `a[4] = string("test")`. In this case, its value points to `(*0x1280b0 = 0x00000004000fe380)`. That is a Go string struct, which is a 32-bit length value of `0x00000004`, next to a 32-bit pointer to a char buffer `0x000fe380`. I added a bit of code to dereference this char buffer pointer and that printed out `(*0xfe380 = 0x6575727474736574)`, which if you decode it using an ASCII table is the character sequence "eurttset", or, more correctly if we account for the little endian byte ordering "testtrue". Our string is only 4 bytes, so it has the value "true" as expected. The "true" part is actually a neighbouring string which just happened to be next to our string in memory.

## A cunning optimisation

You may have noticed that the two `data` pointers for the boolean true and boolean false value were sequential in memory. That is no coincidence, it is actually an optimisation that Go uses to avoid having to allocate memory on the heap for single byte values which are stored in interfaces. Within the runtime, it has a block of 256 bytes with all the values 0-255 listed in it. When the runtime stores any byte value, such as a `bool`, into an `interface{}` then instead of allocating 1 byte on the heap, and storing the value in there and pointing the `iface.data` at that, it instead skips the alloc and just points the `iface.data` into this 256 byte table. So that's why the two addresses of the boolean `true` and boolean `false` value are sequential in memory.

Further, if you look at the dump of a boolean value from the output above, `(*0x1738e0 = 0x0706050403020100)`, you will clearly see that not only do `0x00` and `0x01` lie next to each other in memory, but all the numbers up to `0x07` are there too. If we'd kept dumping, we would have uncovered the whole 256 byte static byte table.

## What does this mean for the 5,120 element slice?

If you were to store a `uint16` at every element in this 5,120 element slice, at the very least, it would take `5,120 * sizeof(iface) + 5,120 * sizeof(uint16)` bytes, i.e. 90KiB on a 64-bit architecture.

I say, "at the very least", as the fact that there is a separate alloc for each `uint16` will almost certainly not simply cost the 2 bytes you get out of the alloc, as memory allocators have **overhead**. They have to track meta data on the alloc, such as how big is is, and often have to "pad" the allocation up to some multiple of 32/64 bits depending on the architecture. I haven't looked into the Go runtime memory allocator overhead yet (maybe I would be pleasantly surprised), but I would expect this a 2 byte alloc would cost at least 16 bytes on a 64-bit architecture. If this were the case, we could expect to be spending 160KiB on this array, which is a far leap from the 10KiB you may have naively assumed for a 5,120 element uint16 array.

Also, each of these 5,120 tiny allocs is 1 more bit of garbage that the Go garbage collection has to think about when it's time to garbage collect.

And don't forget, I had 10,000 of these 5,120 slices in the program I was analysing, so this was starting to add up.

# Deep diving into the runtime

So, to recap - when a simple value is assigned to an `interface{}` variable, a heap alloc is done and the value is copied into the allocated memory. Then the `iface` struct is filled out with a `type` pointer and a pointer to the allocated memory holding the value. The only exception is for single byte values, where the memory alloc is skipped, and the `data` pointer is set to point to the correct value in a shared static byte array instead.

So that's the theory, and it's backed up our memory dumping, so now let's look at the code.

If you create a new Go file called "interface.go" using the following code:

```go
package main

import (
	"fmt"
)


func main() {
	t := 5
	var myval interface{} = uint16(t)
	fmt.Println(myval)
}
```

Then compile and disassemble it using:

```
go build -gcflags="-S" interface.go
```

You'll get a bunch of Go intermediate assembly, which if you pick through it you will find the following section (generated from `go version go1.10.3 darwin/amd64` in my case):

```
	0x0000 00000 (interface.go:8)	TEXT	"".main(SB), $80-0
	0x0000 00000 (interface.go:8)	MOVQ	(TLS), CX
	0x0009 00009 (interface.go:8)	CMPQ	SP, 16(CX)
	0x000d 00013 (interface.go:8)	JLS	132
	0x000f 00015 (interface.go:8)	SUBQ	$80, SP
	0x0013 00019 (interface.go:8)	MOVQ	BP, 72(SP)
	0x0018 00024 (interface.go:8)	LEAQ	72(SP), BP
	0x001d 00029 (interface.go:8)	FUNCDATA	$0, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x001d 00029 (interface.go:8)	FUNCDATA	$1, gclocals·e226d4ae4a7cad8835311c6a4683c14f(SB)
	0x001d 00029 (interface.go:10)	MOVW	$5, ""..autotmp_2+54(SP)
	0x0024 00036 (interface.go:10)	LEAQ	type.uint16(SB), AX
	0x002b 00043 (interface.go:10)	MOVQ	AX, (SP)
	0x002f 00047 (interface.go:10)	LEAQ	""..autotmp_2+54(SP), AX
	0x0034 00052 (interface.go:10)	MOVQ	AX, 8(SP)
	0x0039 00057 (interface.go:10)	PCDATA	$0, $0
	0x0039 00057 (interface.go:10)	CALL	runtime.convT2E16(SB)
	0x003e 00062 (interface.go:10)	MOVQ	24(SP), AX
	0x0043 00067 (interface.go:10)	MOVQ	16(SP), CX
	0x0048 00072 (interface.go:11)	XORPS	X0, X0
	0x004b 00075 (interface.go:11)	MOVUPS	X0, ""..autotmp_3+56(SP)
	0x0050 00080 (interface.go:11)	MOVQ	CX, ""..autotmp_3+56(SP)
	0x0055 00085 (interface.go:11)	MOVQ	AX, ""..autotmp_3+64(SP)
	0x005a 00090 (interface.go:11)	LEAQ	""..autotmp_3+56(SP), AX
	0x005f 00095 (interface.go:11)	MOVQ	AX, (SP)
	0x0063 00099 (interface.go:11)	MOVQ	$1, 8(SP)
	0x006c 00108 (interface.go:11)	MOVQ	$1, 16(SP)
	0x0075 00117 (interface.go:11)	PCDATA	$0, $1
	0x0075 00117 (interface.go:11)	CALL	fmt.Println(SB)
	0x007a 00122 (interface.go:12)	MOVQ	72(SP), BP
	0x007f 00127 (interface.go:12)	ADDQ	$80, SP
	0x0083 00131 (interface.go:12)	RET
	0x0084 00132 (interface.go:12)	NOP
	0x0084 00132 (interface.go:8)	PCDATA	$0, $-1
	0x0084 00132 (interface.go:8)	CALL	runtime.morestack_noctxt(SB)
	0x0089 00137 (interface.go:8)	JMP	0
```

Now we don't need to go through all this disassembly, but the thing to note is the call to `runtime.convT2E16`. You can see the source code for this in the [Go source code](https://github.com/golang/go/blob/0a7ac93c27c9ade79fe0f66ae0bb81484c241ae5/src/runtime/iface.go#L297), it looks like this:

```go
func convT2E16(t *_type, val uint16) (e eface) {
	var x unsafe.Pointer
	if val == 0 {
		x = unsafe.Pointer(&zeroVal[0])
	} else {
		x = mallocgc(2, t, false)
		*(*uint16)(x) = val
	}
	e._type = t
	e.data = x
	return
}
```

So this is the method in the Go runtime which converts a `uint16` value into an interface structure. Actually, as you might have noticed, it converts it into an `eface` struct, not the `iface` struct we were looking at earlier.

```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

The `eface` struct is the "empty interface", and is used for converting values into `interface{}` references. It's pretty much the same as `iface` except that it just has a type pointer instead of a function table; because the empty interface `interface{}` has no function table and so doesn't need the full `iface` struct to back it. One can convert `eface` pointers into `iface` pointers for a given interface by using type assertions, but we won't go into that in this article.

So, we can see here that by storing this `uint16` into an `interface{}` we have ended up allocating the value on the heap in a 2 pointer wide empty interface struct. You can also see, that there is a little optimisation in the `convT2E16` function, which will return a pointer to a common `zeroVal` if the value being stored is a 0. As 0 is a default value in Go, this little optimisation can potentially save on a bunch of memory allocs for this very common case.

## Back to the byte case

So, if we disassemble the `uint8` case, do we see the use of the shared static table?

Disassembling:

```go
package main

import (
	"fmt"
)


func main() {
	t := 5
	var myval interface{} = uint8(t)
	fmt.Println(myval)
}
```

We get:

```
	0x0000 00000 (interface.go:8)	TEXT	"".main(SB), $72-0
	0x0000 00000 (interface.go:8)	MOVQ	(TLS), CX
	0x0009 00009 (interface.go:8)	CMPQ	SP, 16(CX)
	0x000d 00013 (interface.go:8)	JLS	103
	0x000f 00015 (interface.go:8)	SUBQ	$72, SP
	0x0013 00019 (interface.go:8)	MOVQ	BP, 64(SP)
	0x0018 00024 (interface.go:8)	LEAQ	64(SP), BP
	0x001d 00029 (interface.go:8)	FUNCDATA	$0, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x001d 00029 (interface.go:8)	FUNCDATA	$1, gclocals·e226d4ae4a7cad8835311c6a4683c14f(SB)
	0x001d 00029 (interface.go:11)	XORPS	X0, X0
	0x0020 00032 (interface.go:11)	MOVUPS	X0, ""..autotmp_3+48(SP)
	0x0025 00037 (interface.go:11)	LEAQ	type.uint8(SB), AX
	0x002c 00044 (interface.go:11)	MOVQ	AX, ""..autotmp_3+48(SP)
	0x0031 00049 (interface.go:11)	LEAQ	runtime.staticbytes+5(SB), AX
	0x0038 00056 (interface.go:11)	MOVQ	AX, ""..autotmp_3+56(SP)
	0x003d 00061 (interface.go:11)	LEAQ	""..autotmp_3+48(SP), AX
	0x0042 00066 (interface.go:11)	MOVQ	AX, (SP)
	0x0046 00070 (interface.go:11)	MOVQ	$1, 8(SP)
	0x004f 00079 (interface.go:11)	MOVQ	$1, 16(SP)
	0x0058 00088 (interface.go:11)	PCDATA	$0, $1
	0x0058 00088 (interface.go:11)	CALL	fmt.Println(SB)
	0x005d 00093 (interface.go:12)	MOVQ	64(SP), BP
	0x0062 00098 (interface.go:12)	ADDQ	$72, SP
	0x0066 00102 (interface.go:12)	RET
	0x0067 00103 (interface.go:12)	NOP
	0x0067 00103 (interface.go:8)	PCDATA	$0, $-1
	0x0067 00103 (interface.go:8)	CALL	runtime.morestack_noctxt(SB)
	0x006c 00108 (interface.go:8)	JMP	0
```

If you compare it to the disassembly for earlier, you will see that instead of calling some sort of `convT2E8` as you might expect, it instead is loading the effective address (_LEA_) of `runtime.staticbytes+5`. Checking the [runtime source code](https://github.com/golang/go/blob/0a7ac93c27c9ade79fe0f66ae0bb81484c241ae5/src/runtime/iface.go#L592) again, we see:

```go
// staticbytes is used to avoid convT2E for byte-sized values.
var staticbytes = [...]byte{
	0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
	0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f,
	0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17,
	0x18, 0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x1f,
	0x20, 0x21, 0x22, 0x23, 0x24, 0x25, 0x26, 0x27,
	0x28, 0x29, 0x2a, 0x2b, 0x2c, 0x2d, 0x2e, 0x2f,
	0x30, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37,
	0x38, 0x39, 0x3a, 0x3b, 0x3c, 0x3d, 0x3e, 0x3f,
	0x40, 0x41, 0x42, 0x43, 0x44, 0x45, 0x46, 0x47,
	0x48, 0x49, 0x4a, 0x4b, 0x4c, 0x4d, 0x4e, 0x4f,
	0x50, 0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57,
	0x58, 0x59, 0x5a, 0x5b, 0x5c, 0x5d, 0x5e, 0x5f,
	0x60, 0x61, 0x62, 0x63, 0x64, 0x65, 0x66, 0x67,
	0x68, 0x69, 0x6a, 0x6b, 0x6c, 0x6d, 0x6e, 0x6f,
	0x70, 0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77,
	0x78, 0x79, 0x7a, 0x7b, 0x7c, 0x7d, 0x7e, 0x7f,
	0x80, 0x81, 0x82, 0x83, 0x84, 0x85, 0x86, 0x87,
	0x88, 0x89, 0x8a, 0x8b, 0x8c, 0x8d, 0x8e, 0x8f,
	0x90, 0x91, 0x92, 0x93, 0x94, 0x95, 0x96, 0x97,
	0x98, 0x99, 0x9a, 0x9b, 0x9c, 0x9d, 0x9e, 0x9f,
	0xa0, 0xa1, 0xa2, 0xa3, 0xa4, 0xa5, 0xa6, 0xa7,
	0xa8, 0xa9, 0xaa, 0xab, 0xac, 0xad, 0xae, 0xaf,
	0xb0, 0xb1, 0xb2, 0xb3, 0xb4, 0xb5, 0xb6, 0xb7,
	0xb8, 0xb9, 0xba, 0xbb, 0xbc, 0xbd, 0xbe, 0xbf,
	0xc0, 0xc1, 0xc2, 0xc3, 0xc4, 0xc5, 0xc6, 0xc7,
	0xc8, 0xc9, 0xca, 0xcb, 0xcc, 0xcd, 0xce, 0xcf,
	0xd0, 0xd1, 0xd2, 0xd3, 0xd4, 0xd5, 0xd6, 0xd7,
	0xd8, 0xd9, 0xda, 0xdb, 0xdc, 0xdd, 0xde, 0xdf,
	0xe0, 0xe1, 0xe2, 0xe3, 0xe4, 0xe5, 0xe6, 0xe7,
	0xe8, 0xe9, 0xea, 0xeb, 0xec, 0xed, 0xee, 0xef,
	0xf0, 0xf1, 0xf2, 0xf3, 0xf4, 0xf5, 0xf6, 0xf7,
	0xf8, 0xf9, 0xfa, 0xfb, 0xfc, 0xfd, 0xfe, 0xff,
}
```

So indeed, loading from `runtime.staticbytes+5` will result in a pointer to the 5th element of this, which holds the value `0x05`.

# Saving memory

What these little tests have shown is that storing naked types to `interface{}` references has a hidden memory cost. Indeed, storing any value into an interface reference of any type has a memory cost, as at the very least you will be paying an extra pointer for every reference, compared to what you would have if you just held a pointer to the concrete type instead.

If you commonly work with a lot of `interface{}` references, perhaps because you have built some sort of general purpose data structure and you have used `interface{}` to hold the data in it, you can definitely make a memory saving by using the concrete type in place of the `interface{}`. As Go lacks generics, the only way of doing this is literally to duplicate the code and change the `interface{}` references to your concrete type, or to use one of the Go code generators which essentially do the same thing for you. This also gives you back compile time type safety, something that is lost when you start throwing around `interface{}` instead of the true underlying types.

But - once you start copying/pasting code, you introduce code maintenance problems. If you use Go code generators, things are a bit better, but not everyone is familiar with these tools and so you potentially make life harder for future code maintainers / debuggers. So, be sure that it is worth while before making changes.

Another option is to venture into unsafe territory, by using the `unsafe` module to construct `iface` structures on the fly from your generic data type structures, thus effectively factoring out the common `itab` pointer from being stored repeatedly for every value. Needless to say, any future maintainer is going to curse you for doing this (even if it's you in 6 months time) so be very sure that the memory savings are necessary before even considering this. For "educational entertainment" purposes, I might cover this in a future article.

# Conclusion

Interfaces in Go are a great feature, but references to them have a hidden memory cost. If you are wondering where all your memory is going, and you have large slices/arrays/data structures of interface references, hopefully this article will help you understand a little better why the memory is disappearing, and potentially what you can do about it.
