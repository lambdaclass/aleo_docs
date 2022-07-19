# Crossbeam Speech

## Disclaimer

A diferencia de los demás frameworks crossbeam no cuenta con un runtime asincrónico, por lo que esta implementación no es una buena opción para el caso de uso de la organización de tareas asincrónicas independientes. 

## Solución?

<!-- ### Usando Threadpool

#### Funcionamiento general

```rust
fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    let pool = ThreadPool::new(4);
    for stream in listener.incoming() {
            let stream = stream.unwrap();
            pool.execute(|| {
                handle_connection(stream);
            });
    }
}
```

- Tenemos un listener listo para escuchar conexiones
    ```rust
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    ```

- Después esta la creación de la thread pool, que la vemos en detalle más tarde
    ```rust
    let pool = ThreadPool::new(4);
    ```

- Finalmente, se maneja cada conexión entrante.
    ```rust
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        pool.execute(|| {
            handle_connection(stream);
        });
    }
    ```

    - El quién (qué thread) ejecuta cada tarea es manejado por el thread pool.

#### Manejo de cada conexión

Consiste principalmente leer e interpretar una solicitud que hace un cliente y reponder en base a eso.

1. Se lee del stream;
```rust
fn handle_read(stream: &mut TcpStream, buffer: &mut [u8]) {
    stream.read(buffer).unwrap();
}
```
2. Se parsea la lectura en una abstracción que creamos;
```rust
let request = http::parse_http_request(&buffer).unwrap();
```
3. Se crea una respuesta a partir de la solicitud;
```rust
fn create_response(request: HttpRequest) -> String {
    match request.method {
        HttpMethod::GET => {
            format!(
                "{}\r\nContent-Length: {}\r\n\r\n{}",
                "HTTP/1.1 200 OK",
                request.content.len(),
                request.content
            )
        }
        HttpMethod::POST => {
            // ...
        }
        _ => "HTTP/1.1 404 NOT FOUND".to_string(),
    }
}
```
4. Se envía al stream la respuesta.
```rust
fn handle_write(mut stream: TcpStream, request: HttpRequest) {
    stream
        .write_all(create_response(request).as_bytes())
        .unwrap();
    stream.flush().unwrap();
}
```

## Thread Pool

En breves palabras lo que hace el thread pool delegar la ejecución de las tareas asignando las mismas a los distintos threads disponibles a traves de un canal de comunicación.

La administración de tareas funciona de la siguiente manera: 

- Un `ThreadPool` crea un canal de comunicación y se mantendrá en el lado de envío del canal. 

    ```rust
    // unbounded es un canal MPMC
    let (sender, receiver) = crossbeam::unbounded(); 
    ```

- A cada `Worker` se le asociará un receptor del canal. 

- Cada `Job`, como vimos antes, contendrá un closure que queremos enviar por el canal.

- El `ThreadPool` enviará el trabajo que desea ejecutar por el lado de envío del canal. 

    ```rust
    pub fn execute<F>(&self, closure_to_execute: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(closure_to_execute);
        self.sender.send(Message::NewJob(job)).unwrap();
    }
    ```

- El thread contenido en el `Worker` loopea su lado receptor del canal ejecutando los trabajos que reciba. 

    ```rust
    impl Worker {
        fn new(receiver: Receiver<Message>) -> Worker {
            let thread = thread::spawn(move || loop {
                let message = receiver.recv().unwrap();

                match message {
                    Message::NewJob(job) => {
                        job();
                    }
                    Message::Terminate => {
                        break;
                    }
                }
            });

            Worker {
                thread: Some(thread),
            }
        }
    }
    ```

- Por último, el `ThreadPool` implementa el Drop trait para que cuando se va del scope, envíe un mensaje a cada `Worker` y espera a que cada uno termine de ejecutar el `Job` correspondiente.

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        self.tell_workers_to_terminate();
        self.hold_on_until_all_workers_are_done();
    }
}
``` -->

### Usando Work-strealing Scheduler

#### Funcionamiento general

```rust
fn main() {
    crossbeam::scope(|scope| {
        let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
        let thread_count = 4;
        let work_stealing_scheduler = WorkStealingScheduler::new(scope, thread_count);

        for stream in listener.incoming() {
            let stream = stream.unwrap();
            work_stealing_scheduler.push_job(Box::new(|| {
                handle_connection(stream);
            }));
        }
    })
    .unwrap();
}
```

- Tenemos un listener listo para escuchar conexiones
    ```rust
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    ```

- Después la instanciación del work-stealing scheduler, que la vemos en detalle más tarde
    ```rust
    let work_stealing_scheduler = WorkStealingScheduler::new(scope, thread_count);
    ```

- Finalmente, se maneja cada conexión entrante.
    ```rust
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        work_stealing_scheduler.push_job(Box::new(|| {
            handle_connection(stream);
        }));
    }
    ```

#### Manejo de cada conexión

Consiste principalmente leer e interpretar una solicitud que hace un cliente y reponder en base a eso.

1. Se lee del stream;
```rust
fn handle_read(stream: &mut TcpStream, buffer: &mut [u8]) {
    stream.read(buffer).unwrap();
}
```
2. Se parsea la lectura en una abstracción que creamos;
```rust
let request = http::parse_http_request(&buffer).unwrap();
```
3. Se crea una respuesta a partir de la solicitud;
```rust
fn create_response(request: HttpRequest) -> String {
    match request.method {
        HttpMethod::GET => {
            format!(
                "{}\r\nContent-Length: {}\r\n\r\n{}",
                "HTTP/1.1 200 OK",
                request.content.len(),
                request.content
            )
        }
        HttpMethod::POST => {
            // ...
        }
        _ => "HTTP/1.1 404 NOT FOUND".to_string(),
    }
}
```
4. Se envía al stream la respuesta.
```rust
fn handle_write(mut stream: TcpStream, request: HttpRequest) {
    stream
        .write_all(create_response(request).as_bytes())
        .unwrap();
    stream.flush().unwrap();
}
```


#### Work-stealing Scheduler

```rust
pub struct WorkStealingScheduler {
    main: WorkPool,
}

impl WorkStealingScheduler {
    pub fn new(scope: &crossbeam::thread::Scope, thread_count: usize) -> Self {
        let work_pool = WorkPool::new();
        for _ in 0..thread_count {
            let work_pool = work_pool.clone();
            scope.spawn(move |_| {
                let work_pool = work_pool;
                loop {
                    if let Some(job) = work_pool.find_job().take() {
                        job();
                    }
                }
            });
        }
        Self { main: work_pool }
    }

    pub fn push_job(&self, job: Job) {
        self.main.push_job(job);
    }
}

pub struct WorkPool {
    global: Arc<Injector<Job>>,
    local: Worker<Job>,
    stealers: Arc<Mutex<Vec<Stealer<Job>>>>,
}

impl WorkPool {
    pub fn new() -> Self {
        let local = Worker::new_fifo();
        let stealers = Arc::new(Mutex::new(vec![local.stealer()]));
        let global = Arc::new(Injector::<Job>::new());

        Self {
            global,
            local,
            stealers,
        }
    }

    pub fn find_job(&self) -> Option<Job> {
        self.local.pop().or_else(|| {
            iter::repeat_with(|| {
                self.global.steal_batch_and_pop(&self.local).or_else(|| {
                    self.stealers
                        .lock()
                        .unwrap()
                        .iter()
                        .map(|stealer| stealer.steal())
                        .collect()
                })
            })
            .find(|steal| !steal.is_retry())
            .and_then(|steal| steal.success())
        })
    }

    pub fn push_job(&self, job: Job) {
        self.global.push(job);
    }
}

impl Clone for WorkPool {
    fn clone(&self) -> Self {
        let local = Worker::new_fifo();
        self.stealers.lock().unwrap().push(local.stealer());

        Self {
            global: self.global.clone(),
            local,
            stealers: self.stealers.clone(),
        }
    }
}
```



## Problemas que traen estas implementaciones

- No son asincrónica.

- Son complejas.

Además de ser compleja, es single-producer multi-consumer. Las tareas se encolan a un extremo y desencolan del otro. La mayoría de las veces, el thread que encola la tarea la desencolará, sin embargo, otros threads "robarán" ocasionalmente al desencolar la tarea ellos mismos. El deque usa un array y un conjunto de índices que siguen la cabeza y la cola. Cuando el deque esté lleno, encolar dará como resultado un aumento del almacenamiento. Se asigna un nuevo array más grande y los valores se mueven a la nueva tienda. 

El hecho de la cola pueda modificar su tamaño trae consigo cierta complejidad, ya que las operaciones de `push` y `pop` deben manejar esto, volviéndolas costosas. Además, la tarea de liberar la memoria del array viejo puede traer consecuencias negativas, dado que Rust no dispone de un Garbage Collector debemos encargarnos de liberar esa memoria nosotros. El "problema" con esto es que para hacer esto el thread se debe bloquear. Lo que vuelve a la implementación sincrónica. Crossbeam nos trae la solución a esto usando una estrategia de reclamación de memoria.

`
