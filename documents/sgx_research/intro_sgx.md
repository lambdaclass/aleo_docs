# Intro a SGX

Basado fuertemente en https://youtu.be/bAYcZ2eAHkA, y en intentar ejecutar algunos samples del [repo](https://github.com/apache/incubator-teaclave-sgx-sdk) de la sdk de sgx de rust 

## Motivación
1. Lo pidió la gente de Aleo para el record-scanner, aunque a futuro podría tener sentido tenerlo si hacemos una wallet. 
2. Manejamos data sensible, por ejemplo viewkeys y records entonces queremos entornos "lo más seguro posibles"

## Trusted Execution Environments
Los Trusted Execution Environments son "lugares de cómputo seguro", en los que se intenta asegurar:

- Integridad de los datos y el código (Salvo para uso autorizado el código y los datos no deberían sufrir modificaciones estando adentro del environment)
- Confidencialidad de los datos (Salvo para uso autorizado no se debería leakear data de los environments)

Las distintas arquitecturas de procesadores implementan distintos mecanismos de TEEs, entre ellos:

- **Intel SGX**
- ARM trustzone
- AMD PSP
- RISC-V TrustedRV

## Intel SGX
Se basa en el seteo de **Enclaves**, que son estos espacios donde se realiza el cómputo de manera segura

![](https://i.imgur.com/Bd8I1r5.png)

El objetivo original de SGX era la de tener un TEE inmune a memory bugs.
Es una extensión de la ISA para ejecutar sobre ambientes seguros.
No es tan relevante entender completamente cómo funciona por abajo pero lo más importante es que las aplicaciones ahora se ajustan al esquema de separar el codigo "seguro" del "no seguro". El código seguro corre por afuera del sistema operativo y tiene mecanismos de acceso particulares y restringidos.

 Nuestra aplicación empieza corriendo en el código no asegurado y hace a llamados al enclave a demanda por medio de [E]calls (mete al enclave) y [O]calls (saca del enclave)
 También se encarga de inicializar el enclave. Esto es importante porque hasta que no se inicializa, el enclave **no es seguro**. De la misma manera, cuando terminamos de usarlo hay que hacerle un free como los punteros de C.
 
 
## Attestation
Un mecanismo importante en SGX es el de **attestation** que te permite verificar que algo corrió en el enclave y no hubo spoofing (tiene un aire a zkp)

## SDK para rust
Investigamos un [SDK](https://github.com/apache/incubator-teaclave-sgx-sdk) en particular, que es el soportado por apache. Pero hay otra alternativa que se llama [Fontanix](https://edp.fortanix.com/). Me pareció que teaclave tenía más ejemplos y el video en el que nos basamos también lo usa de ejemplo así que vamos a hablar de teaclave.
De acá en más asumimos que estamos usando rust.

### No todo es color de rosas
Uno de los mayores problemas que tiene es la compatibilidad, ya que para correr cosas en el enclave estás obligado a que tu código sea `#[no_std]`, con lo cual no necesariamente vas a poder usar tu crate favorito. Podés revisar en el repo los crates que están porteados a teaclave. En particular portearon una libc, la std (o una parte al menos), primitivas de sincronización (Por ej. `SgxMutex`, `SgxRWLock`). Sin embargo, no hay soporte para async todavía.

## Cómo hay que estructurar los crates en Teaclave
En el video se recomienda basarse en la estructura de los samples del repo, e incluso comenta que en casos reales se basaron en los ejemplos para armar el código.
Toda aplicación la tenemos que separar en 2: la *App* y el *Enclave*. En el enclave tenemos que poner el código que queremos que corra usando sgx. Eso hay que tenerlo en cuenta porque hay un overhead en los llamados al enclave.

En nuestro proyecto esto se ve reflejado como un crate para la app y otro para el enclave. Hay que hacer un poco de magia de compilación y linkeo que por suerte ya viene resuelto en los makefiles de ejemplo.


### Interacción entre los crates

Para integrar la app con el enclave hace falta declarar las funciones del enclave que se llaman desde la app. En ambos crates tratamos a esas funciones como funciones de c.

En la app:
```rust
extern {
    fn say_something(eid: sgx_enclave_id_t, retval: *mut sgx_status_t,
                     some_string: *const u8, len: usize) -> sgx_status_t;
}
```

En el enclave:
```rust
#[no_mangle]
pub extern "C" fn say_something(some_string: *const u8, some_len: usize) -> sgx_status_t {
 ...   
}

```
El `#[no_mangle]` es necesario, no se lo olviden. Y como van a poder ver en los ejemplos, el estilo de escritura es muy "C-like".
Para pasar la data de la app al enclave también hay que notar que no tenemos los mismos tipos de datos. En general lo que se hace es pasar el array de c junto con la longitud y se reconstruye un slice en el enclave usando [`slice.from_raw_parts`](https://doc.rust-lang.org/std/slice/fn.from_raw_parts.html). Esto también implica que los llamados al enclave van a ser código unsafe, por eso se puede ver que los llamados a la función tienen la pinta:

```rust
let result = unsafe {
        say_something(enclave.geteid(),
                      &mut retval,
                      input_string.as_ptr() as * const u8,
                      input_string.len())
    };
```

### Inicialización
Esto está en los samples del repo, pero vale la pena destacar, que las apps tienen que tener el bloque de inicialización y de desmontado del enclave:

```rust
fn init_enclave() -> SgxResult<SgxEnclave> {
    let mut launch_token: sgx_launch_token_t = [0; 1024];
    let mut launch_token_updated: i32 = 0;
    // call sgx_create_enclave to initialize an enclave instance
    // Debug Support: set 2nd parameter to 1
    let debug = 1;
    let mut misc_attr = sgx_misc_attribute_t {secs_attr: sgx_attributes_t { flags:0, xfrm:0}, misc_select:0};
    SgxEnclave::create(ENCLAVE_FILE,
                       debug,
                       &mut launch_token,
                       &mut launch_token_updated,
                       &mut misc_attr)
}

fn main() {
    // Initialize the enclave
    let enclave = match init_enclave() {
        Ok(r) => {
            println!("[+] Init Enclave Successful {}!", r.geteid());
            r
        },
        Err(x) => {
            println!("[-] Init Enclave Failed {}!", x.as_str());
            return;
        },
    };
    // App go brr brr...

    // Cleanup
    enclave.destroy();
}

```

### Configuración del Enclave
En el crate del enclave tenemos el archivo `Enclave.config.xml`. Ej:

```xml
<EnclaveConfiguration>
  <ProdID>0</ProdID> 
  <ISVSVN>0</ISVSVN>
  <StackMaxSize>0x40000</StackMaxSize>
  <HeapMaxSize>0x100000</HeapMaxSize>
  <TCSNum>1</TCSNum> -> Para threading
  <TCSPolicy>1</TCSPolicy> -> Para threading (+ info en la c++ developer's guide)
  <DisableDebug>0</DisableDebug>
  <MiscSelect>0</MiscSelect>
  <MiscMask>0xFFFFFFFF</MiscMask>
</EnclaveConfiguration>
```

### Declaración del Enclave
En el crate del enclave hay que definir un archivo llamado `Enclave.edl` donde se declaran las `[E]calls` y `[O]calls` disponibles:

```rust
enclave {
    // Declaro algunos módulos que importo y voy a usar en el enclave
    from "sgx_tstd.edl" import *;
    from "sgx_stdio.edl" import *;
    from "sgx_backtrace.edl" import *;
    from "sgx_tstdc.edl" import *;
    trusted {
        /* defino [E]Calls bajo trusted. */

        public sgx_status_t say_something([in, size=len] const uint8_t* some_string, size_t len);
    };
    untrusted {
        /* defino [O]Calls bajo untrusted*/
    }
};
```

### Samples
El repo del sdk tiene varios ejemplos
- [hello-rust](https://github.com/apache/incubator-teaclave-sgx-sdk/tree/master/samplecode/hello-rust)
- [tls](https://github.com/apache/incubator-teaclave-sgx-sdk/tree/master/samplecode/tls)
- [thread](https://github.com/apache/incubator-teaclave-sgx-sdk/tree/master/samplecode/thread)
- [unit-test](https://github.com/apache/incubator-teaclave-sgx-sdk/tree/master/samplecode/unit-test). Este está interesante porque tiene casos de uso de cosas que no están tan bien documentadas en la documentación.
- [1Passworkd](https://blog.1password.com/using-intels-sgx-to-keep-secrets-even-safer/)

## Problemas para correrlo en M1, intentos, soluciones parciales y dónde estamos parados
Vamos a tomar de ejemplo el sample de hello-rust, pero el proceso debería ser lo mismo para el desarrollo en general.
Si tenés un procesador Intel que haya salido después del 2015, lo más probable es que no tengas muchos problemas a la hora de correr los ejemplos. Incluso vas a poder correrlo [de forma nativa](https://download.01.org/intel-sgx/sgx-linux/2.9/docs/) si instalás las bibliotecas necesarias de sgx.
Si no estás en esa situación, todavía tenés esperanzas y mejor si lo intentás correr desde linux. El sdk permite correr en un modo simulado que debería funcionar "independientemente" de la arquitectura, más info sobre cómo correrlo en el [repo](https://github.com/apache/incubator-teaclave-sgx-sdk/).

En particular en las mac M1 que cuentan con procesadores ARM esto genera algunos problemas que todavía no pudimos resolver. Por si no viste el link de correrlo en modo simulado vamos a repasar los pasos para lograr eso. Notar que al docker run le tuvimos que agregar el parámetro `--platform linux/amd64` que es la plataforma para la que se buildeó la imagen. 

```bash
git clone https://github.com/apache/incubator-teaclave-sgx-sdk.git
docker run --platform linux/amd64 -v /ruta-al-repo-de-sgx:/root/sgx -ti baiduxlab/sgx-rust

# de vuelta en la consola attacheada al container
cd /root/sgx/samplecode/helloworld
make SGX_MODE=SW # acá falla
cd bin
./app
```

El make falla cuando intenta compilar el crate de la app, en particular cuando está compilando el crate sgx_types. Intentando lo sugerido en [este](https://github.com/rust-lang/rust/issues/60925) issue del compilador de rust pudo compilar este y un par de crates más pero sigue sin terminar la compilación, en particular se cuelga al intentar buildear el crate sgx_urts (llega a buildear las dependencias definidas en el Cargo.toml)

### Posibles explicaciones
- Problema de qemu/docker al correr una imagen buildeada para amd64 sobre una arquitectura arm (https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=221185#c11, https://github.com/rust-lang/rust/issues/94967)
    - Habría que probar buildear la imagen de docker directamente para arm, pero hay que tener instaladas las bibliotecas de intel para hacerlo.

### Primera Solución
Probamos cambiar a una computadora distinta a la M1. En particular esta tenía un i5 que debería de tener soporte para SGX. Sin embargo lo pudimos hacer andar sólamente usando el `Simulation Mode`. Además, no funcionaba out-of-the-box y hubo que hacer unos arreglos.

- La parte de levantar la imagen de docker funcionó normalmente, sin embargo el comando `make` fallaba en la ejecución del script en `sgx_unwind/libunwind/autogen.sh`. La solución fue agregar un `chmod +x` al archivo `configure` que intentaba ejecutar. No sé cómo nadie más tuvo ese problema.
- La primera vez quise compilar el sample project `hello-rust` usando directamente `make`, o sea en modo hardware. Como no funcionó probé con el modo simulación corriendo `make SGX_MODE=SW` y seguía sin funcionar. Esto es porque make no se da cuenta que tiene que recompilar todo, por lo que hay que correr el comando `make clean` antes de compilar nuevamente.


### Samples que se pudieron correr so far
- hello-rust
- http_req
- crypto
- kvdb-memdb

## Más info
- [Charla más sobre Hardware enclaves en general](https://www.infoq.com/presentations/iot-cloud-options-production/)
- [Paper del SDK](https://dl.acm.org/doi/10.1145/3319535.3354241)
- [Video sobre el que nos basamos en gran parte para el documento](https://youtu.be/bAYcZ2eAHkA)
- [Video hablando de la historia y futuro del sdk](https://www.youtube.com/watch?v=w-ktYzeSuj8)
- [Un poquito de documentación de la sdk](https://teaclave.apache.org/sgx-sdk-docs/environment-setup/)
- [Documentación Oficial de Intel SGX](https://download.01.org/intel-sgx/sgx-linux/2.9/docs/)
