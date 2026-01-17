# Manual de Codificación del Kernel de Linux en Rust

Este documento es el complemento moderno al manual de codificación en C. Describe cómo Rust se integra en el kernel ("Rust-for-Linux"), sus patrones arquitectónicos únicos, y cómo mantiene la filosofía de rendimiento del kernel mientras garantiza la seguridad de memoria.

## 1. Filosofía: "Safety First, No Panics"

La integración de Rust no es solo cambiar de sintaxis; es cambiar de paradigma de seguridad.

*   **No Panic:** El kernel no puede "paniquear". Funciones como `unwrap()` o `expect()` están **prohibidas** en código de producción. Si una asignación falla, se debe retornar `Result<T, Error>`.
*   **Abstracciones de Cero Costo:** Se busca que la capa de Rust no añada overhead sobre C.
*   **Interoperabilidad:** Rust debe hablar con C fluidamente, pero encapsulando la inseguridad de C en bloques seguros de Rust.

## 2. Arquitectura del Subsistema Rust

El código Rust en el kernel está organizado en crates clave:

*   **`rust/kernel/` (Crate `kernel`):** La API principal. Contiene abstracciones seguras para subsistemas del kernel (`fs`, `sync`, `driver`, `net`).
*   **`rust/bindings/` (Crate `bindings`):** Generado automáticamente por `bindgen`. Contiene las declaraciones crudas (unsafe) de las funciones y estructuras de C.
*   **`rust/uapi/` (Crate `uapi`):** Bindings para la API de usuario (User API).
*   **`rust/macros/`:** Macros procedurales para simplificar tareas repetitivas (`module!`, `vtable`).

## 3. Mejores Prácticas y Estándares

### 3.1 Manejo de Errores con `Result`
En lugar de retornar enteros negativos (`-ENOMEM`) como en C, Rust usa `Result<T, Error>`.

**Patrón Estándar:**
```rust
// C: int my_func(void) { return -EINVAL; }

// Rust
fn my_func() -> Result {
    if condition {
        return Err(EINVAL);
    }
    Ok(())
}
```
El crate `kernel::error` convierte automáticamente códigos de error de C (negativos) a `Error` de Rust y viceversa al cruzar la frontera FFI.

### 3.2 Comentarios de Seguridad (`// SAFETY:`)
Toda bloque `unsafe` **DEBE** ir acompañado de un comentario `// SAFETY:` que explique por qué es seguro realizar esa operación.

```rust
// SAFETY: `ptr` es garantizado válido por el llamante y no es NULL.
unsafe { bindings::some_c_function(ptr) };
```

### 3.3 Inicialización "Pinned" (`PinInit`)
El kernel tiene muchas estructuras autoreferenciales (mutexes, spinlocks) que no pueden moverse en memoria una vez inicializadas. Rust estándar asume que todo es movible (`Move`).
Para solucionar esto, se usa el sistema `PinInit`:

```rust
impl MyDriver {
    fn new() -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            lock: new_spinlock((), "MyDriver::lock"),
            buffer: KBox::new(..., GFP_KERNEL)?,
        })
    }
}
```

## 4. Patrones Brillantes y "Magia" de Rust

### 4.1 La Macro `container_of!` en Rust
Al igual que en C, necesitamos recuperar la estructura padre desde un puntero a un campo. Rust lo implementa de forma segura usando `offset_of!` (cuando está disponible) o aritmética de punteros verificada.

```rust
// rust/kernel/lib.rs
macro_rules! container_of {
    ($field_ptr:expr, $Container:ty, $($fields:tt)*) => {{
        let offset = ::core::mem::offset_of!($Container, $($fields)*);
        ...
        container_ptr
    }}
}
```

### 4.2 Tipos Opacos (`Opaque<T>`)
Para interactuar con estructuras de C que Rust no debe tocar directamente (porque tienen layouts complejos o bits internos), se usa `Opaque<T>`. Garantiza que Rust solo maneje el puntero y no intente leer/escribir campos inválidos.

### 4.3 `Workqueue` y Concurrencia Segura
Rust impide condiciones de carrera en tiempo de compilación.
*   **`SpinLock<T>`:** Protege los datos `T`. No puedes acceder a `T` sin adquirir el lock.
    ```rust
    let guard = lock.lock(); // Adquiere el lock
    *guard = 5;              // Acceso seguro
    // El lock se libera automáticamente al salir del scope
    ```
*   **`Arc<T>`:** Reference counting atómico para compartir datos entre hilos (equivalente a `kref` pero automatizado).

### 4.4 Macros de Módulo (`module!`)
Simplifica la declaración de metadatos de un módulo, eliminando el boilerplate de C (`MODULE_LICENSE`, `module_init`, etc.).

```rust
module! {
    type: MyModule,
    name: "my_module",
    author: "Jules",
    description: "A Rust kernel module",
    license: "GPL",
}
```

## 5. Gestión de Memoria

*   **`KBox<T>`:** Versión del kernel de `Box<T>`. Requiere flags de asignación explícitos (`GFP_KERNEL`, `GFP_ATOMIC`).
    ```rust
    let b = KBox::new(MyStruct { ... }, GFP_KERNEL)?;
    ```
*   **No Alloc Panics:** Si la memoria se agota, `KBox::new` retorna `Err(ENOMEM)`, no mata el hilo.

## 6. Consejos para el Desarrollador de Rust-for-Linux

1.  **Respeta las Invariantes:** Si escribes `unsafe`, eres responsable de mantener las garantías de Rust (aliasing, data races).
2.  **Usa las Abstracciones:** No llames a funciones de C (`bindings::`) directamente si existe un wrapper seguro en `kernel::`.
3.  **Documenta:** `rustdoc` es un ciudadano de primera clase. Usa `///` para documentar funciones y tipos.
4.  **No reinventes la rueda:** El crate `kernel` ya tiene listas, rbtrees, y workqueues. Úsalos.
