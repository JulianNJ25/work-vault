# Set up Go Development Environment

> Complete guide here https://medium.com/codex/how-to-set-up-a-go-development-environment-67b4b002182e

**Dir structure**
```
src/   Source code files
pkg/   Compiles package objects
bin/   Compiled executables
```

**Env variables**
```bash
export GOROOT=/usr/local/go  
export GOPATH=$HOME/go  
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

**Check workspace setup**
```bash
#display of all of go's env varibles
go env
```

**Dependency management and modules **
```go
//Initialize a module in the proyect
go mod init example.com/helloworld

//Add a depenency
go get github.com/sirupsen/logrus

//the updated information will be in
go.mod
```

**Format code**
```go
gofmt -w main.go
```

**Run tests**
```go
package main

import "testing"
func TestHelloWorld(t *testing.T) {
	result := "Hello, World!"
	expected := "Hello, World!"
	if result != expected {
		t.Errorf("got %q, want %q", result, expected)
	}
}
```

```go
//run tests
go test
```
# Variables and Constants

**Declare variables**
```go
var number = 12
var c, python, java bool

// multiple type declaration
var lua, rust, go = true, false, "no!"
var i, j int = 1, 3

//------------------------------------
// := is allowed outside of functions
number := 54
func foo() int {
	//not valid
	number := 43
	return number
}
```

Variables can exist at the package level
```go
package main

import "fmt"

var c, python, java bool

func main() {
	var i int
	fmt.Println(i, c, python, java)
}

```
## Packages 

**Import**
```go
package main

import (
	"fmt"
	"os"
	"math/rand"
)
```

Exported named begin with a capital letter
```go
package main

import (
	"fmt"
	"math"
)

func main() {
	//math.Pi -> capital letter
	fmt.Println(math.Pi)
}
```

# Functions

```go
package main

import (
	"fmt"
)

// return type after arguments clause
func add(x int, y int) int {
	return x + y
}

// arguments are of the same type, it can be ommited
func add(x, y int) int {
	return x + y
}

//----------------------------------------------

// multiple return types
func swap(x, y string) (string, string) {
	return y, x
}

func split(sum int) (x, y int) {

}

func main() {
	// both return values assigned 
	a, b := swap("hello", "world")
	fmt.Println(a, b)
}
```

# Types
```go
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // alias for uint8

rune // alias for int32
     // represents a Unicode code point

float32 float64

complex64 complex128
```

# Go routines - concurrency
**Goroutines** are lightweight threads

```go
package main

import (
	"fmt"
	"time"
)

func sayHelo() {
	fmt.Println("Hello World")
}

func main() {
	go sayHello()
	time.Sleep(time.Second)
}
```

**Channels** syncronize goroutines

``` go
package main

import "fmt"

func main() {
	ch := make(chan string)
	
	go func() {
		ch <- "Hello, Channel!"
	}()
	
	fmt.Println(<-ch)
}
```