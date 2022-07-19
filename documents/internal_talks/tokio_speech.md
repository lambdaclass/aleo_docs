# Tokio Speech
- Async main y async/await en funciones
    - Las funciones async no corren hasta que son ejecutadas por un executor ya sea con await o block_on, llamar a una funcion async solo devuelve un tipo anonimo que implementa el trait Future
    - El async main se transforma por la macro de tokio por una funcion main sincrona que empieza a correr el runtime de tokio y ejecuta en un bloque asincrono el cuerpo de la funcion main
- Tokio spawn
    - Spawn de una nueva task, las tasks en tokio representan a threads que estan sincronizados por el runtime de tokio

## Tokio implementation
### Main
La funcion main es async y se transforma por la macro de tokio por una funcion main sincrona que empieza a correr el runtime de tokio y ejecuta en un bloque asincrono el cuerpo de la funcion.

Este Runtime de Tokio crea un nuevo thread corriendo un Reactor en background que se encarga de manejar los recursos del I/O, crea un Threadpool para ejecutar los diferentes Futures y crea una instancia de un Timer por thread pool que maneja los tiempos de ejecucion de cada tarea.

```rust
#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
    loop {
        let (stream, _) = listener.accept().await.unwrap();
        tokio::spawn(async {
            handle_new_connection(stream).await;
        });
    }
}
```
Se transforma por la macro en lo siguiente
```rust
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
        loop {
            let (stream, _) = listener.accept().await.unwrap();
            tokio::spawn(async {
                handle_new_connection(stream).await;
            });
        }
    })
}
```

Utilizamos el TCPListener de Tokio que escucha por nuevas conexiones con el metodo `.accept()` y lo hace de manera asincrona.
Luego utilizamos `tokio::spawn(async ||)` para que cada conexion se maneje en una nueva task y el servidor se comporte de manera concurrente.

### Handling new connections
```rust
async fn handle_new_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    let mut reader = BufReader::new(&mut stream);
    reader.read(&mut buffer).await.unwrap();
    let request = parse_http_request(&buffer).unwrap();
    let response = create_response(request).await;
    stream.write_all(response.as_bytes()).await.unwrap();
}
```
Utilizamos BufReader de tokio que mantiene un buffer en memoria de lo leido y trabaja sobre AsyncRead para leer el request completo que recibimos.
El request es parseado por un modulo http, que se encarga de formar correctamente todo el request recibido y separarlo en sus distintas partes.
Creamos la respuesta segun el metodo de request que se recibe.
Escribimos la respuesta creada en el mismo stream de conexion.

### Creating a response
```rust
async fn create_response(request: HttpRequest) -> String {
    match request.method {
        HttpMethod::GET => "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK".to_string(),
        HttpMethod::POST => {
            format!(
                "HTTP/1.1 200 OK\r\nContent-Length: {}\r\n\r\n{}",
                request.content.len(),
                request.content,
            )
        }
        _ => "HTTP/1.1 400".to_string(),
    }
}
```
La creacion de la respuesta unicamente esta hecha para metodos GET y POST, en caso de un método GET simplemente se devuelve una respuesta indicando que el request fue exitoso y se devuelve un body con contenido OK.En caso de un POST simplemente se devuelve la respuesta con el mismo body recibido.


