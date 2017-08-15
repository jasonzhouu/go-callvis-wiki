The `-focus` flag is by default set to `main` since all the programs 
have it and keeping focus empty by default would generate gigantic output.

> Only one argument is supported currently (which has to be a *main package*). 
> Other arguments after the first one are ignored, because important 
> is the *main package* where the program starts, and the code analysis 
> can figure out all other calls from there.

## Single package

Using only single package example is pretty straightforward since there are only calls inside that package.

![callvis](https://cloud.githubusercontent.com/assets/1229233/24300848/147b3336-10ae-11e7-9702-bf376ce2870e.png)

```
go-callvis playground/callvis | dot -Tpng -o callvis.png 
```

**Code**:

_main.go_
```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	setupRoutes()
	http.ListenAndServe(":4321", nil)
}

func setupRoutes() {
	http.HandleFunc("/", index)
}

func index(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "index")
}
```

#### Output contains all the calls that:

* either have both **caller and callee function** inside the _focused package_,
  - __`main.main()`__ --> __`main.setupRoutes()`__

* or have only **caller function** inside the _focused package_,
  - __`main.main()`__ --> `http.ListenAndServe()`
  - __`main.setupRoutes()`__ --> `http.HandleFunc()`
  - __`main.index()`__ --> `fmt.Fprintf()`

* or have only **callee function** inside the _focused package_.
  - `http.(HandlerFunc).ServeHTTP()` - - > __`main.index()`__


## Multiple packages

If the program consists of multiple packages like in your case, you can choose another package for `-focus` or use empty focus 
with relevant `-limit` and `-ignore` flags to constraint the size of output.

### Code

_main.go_
```go
package main

import (
	"playground/callvis/api"
	"net/http"
)

func main() {
	api.SetupRoutes()
	http.ListenAndServe(":4321", nil)
}
```

_api/api.go_
```go
package api

import (
    "fmt"
    "net/http"
)

func Index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "index")
}

func SetupRoutes() {
	http.HandleFunc("/", Index)
}
```

### Output variants

- #### Focusing package `main`
  
  Output doesn't show call to `http.HandleFunc()` and `fmt.Fprintf()`, because neither caller nor callee are inside focused package.
  
  ![callvis_main](https://cloud.githubusercontent.com/assets/1229233/24291637/19f377b6-108a-11e7-99ab-a0bd1574479b.png)

  ```
  go-callvis -focus main -group pkg playground/callvis | ...
  ```

- #### Focusing package `api`
  
  Output doesn't show call to `http.ListenAndServe()`, because neither caller nor callee are inside focused package. 
  However it still shows `api.SetupRoutes()` call since it is inside `api` package.

  ![callvis_api](https://cloud.githubusercontent.com/assets/1229233/24300617/55173e40-10ad-11e7-8c51-3d4f6d000952.png)

  ```
  go-callvis -focus api -group pkg playground/callvis | ...
  ```

- #### No focusing

  By using `-limit` we can define import path prefix and packages without this prefix will be ignored. 
  Same way the `-ignore` flag defines import path for ignoring packages. 
  Both can contain multiple prefix paths.

  ![callvis](https://cloud.githubusercontent.com/assets/1229233/24302966/a80b4270-10b4-11e7-9737-c27e1ab3b0a0.png)

  ```
  go-callvis -focus="" -limit playground/callvis -group pkg playground/callvis | ...
  ```

- #### Including `net/http`

  We can add `net/http` package to the `-limit` to include it's callgraph. This generates huge output, because it 
  contains internal calls inside std packages.

  [![hugethumb](https://cloud.githubusercontent.com/assets/1229233/24303891/eb0fe32a-10b7-11e7-8989-0ac49f28c30f.png)](https://cloud.githubusercontent.com/assets/1229233/24303606/e5e8d4ac-10b6-11e7-8712-0a6aa2441cda.jpg)

  ```
  go-callvis -focus="" -limit "net/http,playground/callvis" -group pkg playground/callvis | ...
  ```

----
### Conclusion

To me it seems there is **missing a way** to show output similar to the last one **without the internal calls** of the std package.

I think the behaviour of `-nostd` should be changed so it only omits internal calls inside and between std packages. Or some new flag added with default `true` since I assume that not many users are interested in internal calls inside std package.

I will look into this further to find the most optimal solution.