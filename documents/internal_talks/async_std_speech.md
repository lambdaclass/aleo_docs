# Async-std Speech
## Dependencies
Only async-std is used to set the runtime for async/await. There is another `use` but to an internal library named `http`.

## Implementation
The explanation is bottom-up

## General Observations
### About async/await
- One possible scenario for using async/await is when performing many I/O operations, things that leave you waiting for a result and are not cpu intensive. 
- The return type of an async function is a future and thus needs to be await'ed for to be used.
- async functions can only be called from other async functions (DOUBT)
- currently you can't implement dynamic dispatch for traits with async functions (might change based on https://blog.rust-lang.org/inside-rust/2022/02/03/async-in-2022.html)

### async-std related stuff
- async functions can also be called from sync functions  using `task::block_on`
- `task:spawn` creates a new task but not necessarily create a new thread
- Both `task::spawn` and `task::block_on` return what the future parameter passed to it returns.
- To read from a `async-std::net::TcpStream` you can use the `read(mut buf: &[u8])` method or use a `async-std::io::BufReader`
- Notice that async-std's TcpStreams are not flushed after writes.


### handle_connection
The core of the logic for deciding what to do consists of the following steps:

1. Reading from the TcpStream
    ```rust
    let n = stream.read(&mut buf).await?; // 1
    ```
2. Parse the http request
    ```rust
    let parsed_http = http::parse_http_request(&buf)?;
    ```
3. Build a Response
    ```rust
    let response = match http_method {
        HttpMethod::GET => { // 3
            let fs_path = format!(".{}", resource_path);
            let contents = async_std::fs::read_to_string(fs_path).await?;

            format!(
                "HTTP/1.1 200 OK\r\nContent-Length:{}\r\n\r\n{}", 
                contents.len(), 
                contents
            );
        }
        HttpMethod::POST => {
            //...
        }
        _ => {
            println!("Request could not be parsed");
        }
    }
    ```
    (http_method and resource_path are taken from the parsed http struct).
4. Write the response back
    ```rust
        stream.write(&mut response.as_bytes()).await?;
    ```
    
The function to handle a single connection is then:
``` rust
async fn handle_connection(stream: TcpStream) -> std::io::Result<()> {
    // Read from the TcpStream
    // ...

    // parse the http (more than 0 bytes were read)
    // ...
    
    // build the response
    // ...
    
    // write the response back to the TcpStream
    // ...
    
    Ok(())
}
```

Extra details:
- the std::io::Result type is also used by async-std and having that as a return type allows us to use the `?` syntax
        - `TcpStream` is implemented by async-std which implements a non-blocking version of the `Read` trait (`AsyncRead`). The same goes for the corresponding `Write` and `AsyncWrite` 
- The http module takes care of http related tasks, such as parsing the bytes that came from the TcpStream
- On GET requests, a file is searched using the resource path as a relative path. Even though this might be a security concern, i opted for simplicity in this case.
- On POST requests, the contents of the body of the request are returned, not the whole request. 

### connections_loop
``` rust
async fn connections_loop() -> std::io::Result<()> {
    // bind to a socket address
    let listener = TcpListener::bind("127.0.0.1:8080").await?; // 1

    let mut incoming = listener.incoming(); // 2
    while let Some(stream) = incoming.next().await {
        let stream = stream?; // 2.1
        task::spawn(handle_connection(stream)); // 3
    }

    Ok(())
}
```

1. TcpListener is also part of async-std. 
2. In async-std there are two ways for the `TcpListener` to accept new packets. The first one is the `.accept()` method in which a single connnection is accepted (and returns a result wrapped in a future). The second one is the `.incoming()` method which accepts new connections in a loop infinitely. Both are done **asynchronously**.
3. spawn a async-std task. Remember that **it is not equivalent to a single thread**. The task may share a thread of execution with other tasks.  

Extra details:
- (2.1) This line is needed because the next() returns an Option<Result<...>> so we destructured the Option and now we're destructuring the Result
- The TcpStream can also be read using `async_std::io::BufReader`, and it has some utility methods such as read_line, lines, etc.

### main
``` rust
fn main() {
    let fut = connections_loop();
    if let Err(e) = task::block_on(fut) {
        eprintln!("{}", e);
    }
}
```
`block_on` as the name says blocks the current thread of execution until the future is resolved.
An alternative of this is the following:
``` rust
#[async_std::main]
async fn main() {
    if let Err(e) = connections_loop().await {
        eprintln!("{}", e);
    }
}
```

While this being a bit more consice, i like that the former is more explicit about the latter.
