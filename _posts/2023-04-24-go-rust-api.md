---
title: "Comparing Go and Rust performance for a high-throughput HTTP server"
date: 2023-04-24
permalink: /posts/2023/04/go-rust/
tags:
  - go
  - rust
---

Let's begin with the results, because that's why you're here. Go was faster than Rust in this test. I do not know why. When Go outperforms Rust, it means you've done something wrong in Rust. I can't tell what. All the code is in [this repo](https://github.com/jmkopper/Go-Throughput-Tester).

The tests work as follows.

- The client sends a JSON request to the server. The requests are formatted like this:

```json
{
    "secret": "JUST A DUMb ApI kEY",
    "budget": 500,
    "tests": [
        {"value": 664},
        {"value": 868},
        {"value": 133}
    ]
}
```

- The server then takes the "tests", sorts them by value, then returns the sorted list of test values that are less than the budget.

Why this? I wanted the server to have to run an algorithm that is *slower* than linear time. The sorting algorithms Go and Rust use are presumably $O(n\log n)$. I wanted something slow so that the algorithm itself took up most of the processing time, rather than potentially weird things like copying data to and from memory, all of which should be done in $O(n)$ and is very hard to control for unless you're an expert in these languages.

I also wanted to test Go and Rust performance on a *single* connection (for personal reasons. Please respect my privacy and don't ask) that may handle large amounts of data in a single request.

Using different length `test`s allowed me to separate out how much time was spent during transmission, during server data handling, and during the execution of the algorithm itself.

I did 3 tests:

1. 10,000 requests with an array of length 1,000
2. 1,000 requests with an array of length 10,000
3. 100 requests with an array of length 100,000

I measured the execution time of the sorting function and the total "roundtrip" time of each request. The average and max values are in the tables below.

| request size | go server avg | go rountrip avg | rust server avg | rust roundtrip avg
|--------------|---------------|-----------------| ----------------|------------------
| 1,000 | 0.12 ms |	1.04 ms |	0.43 ms | 1.99 ms
| 10,000 | 0.96 ms | 8.37 ms |	4.11 ms | 15.44 ms
| 100,000 | 6.85 ms | 59.84 ms | 34.32 ms | 139.93 ms

| request size | go server max | go rountrip max | rust server max | rust roundtrip max
|--------------|---------------|-----------------| ----------------|------------------
| 1,000 | 0.72 ms |	2.77 ms | 1.47 ms | 5.35 ms
| 10,000 | 2.71 ms | 15.84 ms | 11.27 ms | 34.67 ms
| 100,000 | 9.38 ms | 66.54 ms | 46.69 ms | 176.14 ms

![](/images/go_rust_chart.png "Go vs Rust")

## Ecosystems

Both Go and Rust have robust ecosystems that provided everything I needed. I will, however, highlight two irritations, one for each language.

The first irritation is that Rust has no convenient way to build an http API using the standard library. This feels absurd. Go's `net/http` package is more than sufficient for servers like the ones I have built. With Rust, you have to choose between one of many possibilities, each with its own syntax and paradigms, its own quirks, and its own hidden opportunities for optimization. By standardizing the http package, Go enforces structure on the ecosystem and it is much more pleasant to use.

The second irritation is that Go has no convenient way to read environment variables from a `.env` file. This also feels absurd. For a language that is commonly used to build web services, a language whose designers had the foresight to build a `net/http` package for, it is insane that you have to go to github to install a package to do this simple task (or, somehow worse, you manually open and read the file yourself: yuck!)

## Implementing the algorithm

The code underlying the actual computation is concise and readable in both languages, but Rust's is clearly the more elegant.

```go
func processTestData(tests []Test, budget int) []Test {
	sort.Slice(tests, func(i, j int) bool { return tests[i].Value < tests[j].Value })
	var results []Test
	for i := 0; i < len(tests) && tests[i].Value < budget; i++ {
		results = append(results, tests[i])
	}
	return results
}
```

```rs
async fn process_test_data(test_request: &mut TestRequest) -> Vec<Test> {
    test_request.tests.sort_unstable_by_key(|test| test.value);
    test_request
        .tests
        .iter()
        .take_while(|test| test.value < test_request.budget)
        .cloned()
        .collect()
}
```

As a devout parishioner of the Church of Functional Programming, I love that Rust incorporates functional-style tools. I also appreciate that, unlike most functional programming languages, Rust acknowledges that it was stupid for mathematicians to make functions contravariant, and instead uses the "dot" syntax for function composition.

Nevertheless, the algorithm is simple and readable in Go, so we shouldn 't criticize too harshly.

A final point is that it's very obvious in Rust when something is a reference and when it is not. My original implementation of the Rust algorithm passed a `TestRequest` instead of a mutable reference to a test request, because I thought that was what the Go code was doing. Not so! While any Go fanatic will tell you that Go is 100% pass-by-value, they will also smugly tell you that *technically*, a slice has a *pointer to an array*, not an array itself. Therefore if you pass a slice by value, it is like passing an array by reference. These murky standards are exactly the reason Rust is designed the way it is... and also exactly the reason that true programming languages, like Haskell, have all data types immutable.

## Building the webserver

Despite my complaints about having to use a third-party package to build the server, the Rust code is remarkably short. The Go server code is 94 lines (78 SLOC on github) and the Rust server code is 65 lines (56 SLOC). How do we account for this difference? Well, there is one feature in the Go server that I didn't bother figuring out how to do in Rust: a 5 second timeout. That accounts for 5 of the lines. The rest, I think, is divided between two things

First, error handling in Rust is extremely simple. Here's the `main` function from the Rust server:

```rs
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok(); // error handling here!
    let address = format!("0.0.0.0:{}", PORT);
    HttpServer::new(|| {
        App::new().service(web::resource("/runtest").route(web::post().to(run_test_handler)))
    })
    .bind(&address)? // error handling here!
    .run()
    .await
}
```

Using algebraic types allows Rust to make error handling simple and elegent. 10 points to Rust. Second, `actix` automatically handles a lot of the boilerplate around creation and use of the API endpoint. In particular, the JSON parsing is built into the *types*. Here's the type annotation for the `run_test_handler_function`:

```rs
async fn run_test_handler(mut test_request: web::Json<TestRequest>) -> impl Responder {
    //...
}
```

Here's the analogous Go code:

```go
func (th *testHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    //...
    if err := json.NewDecoder(r.Body).Decode(&testRequest); err != nil {
		http.Error(w, http.StatusText(http.StatusBadRequest), http.StatusBadRequest)
		return
	}
    //...
}
```

Four lines to handle a JSON parsing error!

## Conclusion
Haskell is the superior choice by all metrics. Obviously it was absent in this test, but that is of no concern. I'm stunned by how well Go performed. Both Go and Rust were pleasant to code in. I don't really know why people say the dev experience in Go is superior.
