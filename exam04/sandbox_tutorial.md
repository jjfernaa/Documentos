# TUTORIAL PASO A PASO: SANDBOX

## 🎯 OBJETIVO
Implementar `int sandbox(void (*f)(void), unsigned int timeout, bool verbose)` que ejecute una función en un entorno controlado y determine si es "nice" (buena) o "bad" (mala).

---

## 📋 ANÁLISIS DEL PROBLEMA

### **¿Qué hace sandbox?**
```c
// Función que queremos testear
void mi_funcion(void) {
    printf("Hola mundo\n");
    // exit(0);  ← implícito si termina normalmente
}

int resultado = sandbox(mi_funcion, 5, true);
// resultado = 1 (nice) si todo va bien
// resultado = 0 (bad) si hay problemas  
// resultado = -1 si hay error en sandbox
```

### **¿Cuándo es una función "bad"?**
1. **Exit code ≠ 0**: `exit(1)`, `exit(42)`, etc.
2. **Terminada por señal**: segfault, abort, kill, etc.  
3. **Timeout**: Tarda más que `timeout` segundos
4. **Stopped**: Pausada por SIGSTOP (nunca continúa)

### **¿Cuándo es una función "nice"?**
✅ Termina normalmente con `exit(0)` (o return implícito)
✅ En menos de `timeout` segundos
✅ Sin crashes ni señales

---

## 🧠 COMPRENSIÓN CONCEPTUAL

### **¿Por qué necesitamos fork?**
- La función `f()` puede ser **peligrosa** (segfault, loop infinito)
- Si crashea, **no debe afectar** nuestro programa principal
- Ejecutarla en proceso hijo = **aislamiento**

### **¿Cómo funciona el timeout?**
- `alarm(timeout)` programa una "alarma" 
- Si el proceso no termina en `timeout` segundos → recibe SIGALRM
- SIGALRM puede **interrumpir** `waitpid()` en el padre

### **¿Qué es waitpid() vs wait()?**
- `wait()`: Espera a cualquier hijo
- `waitpid(pid, &status, 0)`: Espera a un hijo específico
- Puede ser **interrumpido** por señales (errno = EINTR)

### **Estados de terminación**:
```c
if (WIFEXITED(status)) {
    // Terminó normalmente con exit()
    int code = WEXITSTATUS(status);
}
if (WIFSIGNALED(status)) {
    // Terminado por una señal  
    int signal = WTERMSIG(status);
}
```

---

## 🔧 IMPLEMENTACIÓN PASO A PASO

### **PASO 1: Fork y Proceso Hijo**

```c
int sandbox(void (*f)(void), unsigned int timeout, bool verbose)
{
    pid_t pid = fork();
    if (pid < 0)
        return -1;  // Error en fork

    if (pid == 0) {
        // --- PROCESO HIJO ---
        alarm(timeout);  // Programar timeout
        f();             // Ejecutar función
        _exit(0);        // Si f() retorna, consideramos éxito
    }
    
    // --- PROCESO PADRE ---
    // ... continúa abajo
}
```

**¿Por qué `_exit(0)` y no `exit(0)`?**
- `_exit()`: Termina inmediatamente, no ejecuta cleanup
- `exit()`: Ejecuta atexit handlers, flush buffers, etc.
- En un hijo simple, `_exit()` es más seguro

**¿Por qué `alarm()` en el hijo?**
- Si `f()` nunca termina → hijo recibe SIGALRM → muerte
- El padre detecta que murió por SIGALRM = timeout

### **PASO 2: Configurar Handler de Señales en el Padre**

```c
// --- PROCESO PADRE ---

// Handler vacío para SIGALRM
struct sigaction sa = {0};
sa.sa_handler = do_nothing;  // Función definida arriba
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;
if (sigaction(SIGALRM, &sa, NULL) < 0)
    return -1;

alarm(timeout);  // También programamos alarma en el padre
```

**Función handler**:
```c
static void do_nothing(int sig)
{
    (void)sig;  // No hacer nada, solo interrumpir waitpid()
}
```

**¿Por qué configurar SIGALRM en el padre?**
- `waitpid()` puede esperar indefinidamente
- Si llega SIGALRM → `waitpid()` retorna con errno=EINTR
- Así podemos detectar timeout desde el padre

### **PASO 3: Esperar al Proceso Hijo**

```c
int status;
pid_t result = waitpid(pid, &status, 0);

if (result < 0) {
    if (errno == EINTR) {
        // ¡TIMEOUT! waitpid() interrumpido por SIGALRM
        kill(pid, SIGKILL);     // Matar al hijo
        waitpid(pid, NULL, 0);  // Recolectar el zombie
        if (verbose)
            printf("Bad function: timed out after %u seconds\n", timeout);
        return 0;  // Bad function
    }
    return -1;  // Error inesperado
}

// Cancelar alarma restante
alarm(0);
```

**Flujo del timeout**:
1. Padre espera con `waitpid()`
2. Pasan `timeout` segundos
3. SIGALRM llega al padre
4. `waitpid()` se interrumpe (errno = EINTR)
5. Matamos al hijo con SIGKILL
6. Recolectamos el zombie

### **PASO 4: Analizar el Status de Salida**

```c
if (WIFEXITED(status)) {
    // El proceso terminó con exit()
    int code = WEXITSTATUS(status);
    if (code == 0) {
        // ¡Nice function!
        if (verbose)
            printf("Nice function!\n");
        return 1;
    } else {
        // Bad: exit code != 0
        if (verbose)
            printf("Bad function: exited with code %d\n", code);
        return 0;
    }
}

if (WIFSIGNALED(status)) {
    // El proceso murió por una señal
    int sig = WTERMSIG(status);
    if (sig == SIGALRM) {
        // Timeout (murió por nuestra alarma)
        if (verbose)
            printf("Bad function: timed out after %u seconds\n", timeout);
    } else {
        // Otra señal (crash, kill, etc.)
        if (verbose)
            printf("Bad function: %s\n", strsignal(sig));
    }
    return 0;  // Bad function
}

return -1;  // Caso inesperado
```

**Tipos de señales comunes**:
- `SIGALRM`: Timeout (por alarm)
- `SIGSEGV`: Segmentation fault
- `SIGABRT`: abort() llamado
- `SIGKILL`: kill -9 externo
- `SIGTERM`: Terminación normal

---

## 🔍 CÓDIGO COMPLETO EXPLICADO

```c
#include <unistd.h>     // fork(), alarm(), _exit()
#include <sys/wait.h>   // waitpid(), WIFEXITED, etc.
#include <signal.h>     // sigaction, SIGALRM, etc.
#include <stdbool.h>    // bool
#include <stdio.h>      // printf()
#include <errno.h>      // errno, EINTR
#include <string.h>     // strsignal()

// Handler vacío para interrumpir waitpid()
static void do_nothing(int sig)
{
    (void)sig;
}

int sandbox(void (*f)(void), unsigned int timeout, bool verbose)
{
    pid_t pid = fork();
    if (pid < 0)
        return -1;

    if (pid == 0) {
        // --- PROCESO HIJO ---
        alarm(timeout);
        f();
        _exit(0);
    }

    // --- PROCESO PADRE ---
    struct sigaction sa = {0};
    sa.sa_handler = do_nothing;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    if (sigaction(SIGALRM, &sa, NULL) < 0)
        return -1;

    alarm(timeout);

    int status;
    pid_t r = waitpid(pid, &status, 0);
    if (r < 0) {
        if (errno == EINTR) {
            // Timeout
            kill(pid, SIGKILL);
            waitpid(pid, NULL, 0);
            if (verbose)
                printf("Bad function: timed out after %u seconds\n", timeout);
            return 0;
        }
        return -1;
    }

    // Cancelar alarma
    alarm(0);

    if (WIFEXITED(status)) {
        int code = WEXITSTATUS(status);
        if (code == 0) {
            if (verbose)
                printf("Nice function!\n");
            return 1;
        }
        if (verbose)
            printf("Bad function: exited with code %d\n", code);
        return 0;
    }

    if (WIFSIGNALED(status)) {
        int sig = WTERMSIG(status);
        if (sig == SIGALRM) {
            if (verbose)
                printf("Bad function: timed out after %u seconds\n", timeout);
        } else {
            if (verbose)
                printf("Bad function: %s\n", strsignal(sig));
        }
        return 0;
    }

    return -1;  // Caso imprevisto
}
```

---

## 🎨 EJEMPLOS PRÁCTICOS

### **Ejemplo 1: Nice Function**
```c
void good_function(void) {
    printf("Everything is fine!\n");
    // return implícito → exit(0)
}

int main() {
    int result = sandbox(good_function, 2, true);
    printf("Result: %d\n", result);
    // Output: "Nice function!" 
    // Result: 1
}
```

### **Ejemplo 2: Bad Exit Code**
```c
void bad_exit_function(void) {
    printf("Something went wrong\n");
    exit(42);
}

int main() {
    int result = sandbox(bad_exit_function, 2, true);  
    printf("Result: %d\n", result);
    // Output: "Bad function: exited with code 42"
    // Result: 0
}
```

### **Ejemplo 3: Timeout (Loop Infinito)**
```c
void infinite_loop(void) {
    while (1) {
        // Loop infinito - nunca termina
    }
}

int main() {
    int result = sandbox(infinite_loop, 2, true);
    printf("Result: %d\n", result);
    // Output: "Bad function: timed out after 2 seconds"  
    // Result: 0
}
```

### **Ejemplo 4: Segfault**
```c
void crash_function(void) {
    int *p = NULL;
    *p = 42;  // Segmentation fault
}

int main() {
    int result = sandbox(crash_function, 2, true);
    printf("Result: %d\n", result);
    // Output: "Bad function: Segmentation fault"
    // Result: 0  
}
```

### **Ejemplo 5: Abort**
```c
void abort_function(void) {
    abort();  // Genera SIGABRT
}

int main() {
    int result = sandbox(abort_function, 2, true);
    printf("Result: %d\n", result);
    // Output: "Bad function: Aborted"
    // Result: 0
}
```

---

## 🚀 CASOS EXTREMOS Y EDGE CASES

### **Función que ignora SIGALRM**
```c
void ignore_alarm(void) {
    signal(SIGALRM, SIG_IGN);  // Ignorar SIGALRM
    sleep(10);  // Dormir más que el timeout
    printf("I ignored the alarm!\n");
}
```
**¿Qué pasa?**: El timeout desde el **padre** sigue funcionando porque `waitpid()` se interrumpe por SIGALRM del padre, no del hijo.

### **Función muy rápida**
```c
void fast_function(void) {
    printf("Quick!\n");
}
```
**¿Qué pasa?**: Funciona perfectamente, termina antes del timeout.

### **Función que se detiene (SIGSTOP)**  
```c
void stop_function(void) {
    raise(SIGSTOP);  // Pausar el proceso
    printf("This never executes\n");
}
```
**¿Qué pasa?**: El proceso se pausa indefinidamente → timeout del padre → kill SIGKILL → "timed out".

---

## 🐛 ERRORES COMUNES Y CÓMO EVITARLOS

### **❌ Error 1: No manejar EINTR**
```c
// MAL:
pid_t r = waitpid(pid, &status, 0);
if (r < 0)
    return -1;  // ¡No distingues timeout de error real!

// BIEN:
pid_t r = waitpid(pid, &status, 0);  
if (r < 0) {
    if (errno == EINTR) {
        // Timeout
        kill(pid, SIGKILL);
        waitpid(pid, NULL, 0);
        return 0;
    }
    return -1;  // Error real
}
```

### **❌ Error 2: No recolectar zombie después de SIGKILL**
```c
// MAL:
kill(pid, SIGKILL);
// ¡Proceso queda zombie!

// BIEN:
kill(pid, SIGKILL);
waitpid(pid, NULL, 0);  // Recolectar zombie
```

### **❌ Error 3: No cancelar alarm**
```c
// MAL: Si el hijo termina rápido
// La alarma queda programada y puede interferir después

// BIEN:
if (r >= 0)  // waitpid() exitoso
    alarm(0);  // Cancelar alarma restante
```

### **❌ Error 4: Confundir timeout del hijo vs padre**
```c
// Ambos procesos tienen alarm(timeout):
// - HIJO: Si f() tarda mucho → SIGALRM mata al hijo
// - PADRE: Si waitpid() espera mucho → SIGALRM interrumpe waitpid()
// ¡Los dos mecanismos cooperan para el timeout!
```

### **❌ Error 5: Usar exit() en vez de _exit() en el hijo**
```c
// MAL en el hijo:
f();
exit(0);  // Puede causar problemas con buffers compartidos

// BIEN en el hijo:
f(); 
_exit(0);  // Terminación limpia y simple
```

---

## 🧪 TESTING Y VALIDACIÓN

### **Funciones de prueba útiles**:
```c
// 1. Nice function
void ok_function(void) {
    printf("OK\n");
}

// 2. Bad exit
void bad_exit(void) {
    exit(1);
}

// 3. Timeout
void timeout_function(void) {
    sleep(10);
}

// 4. Segfault  
void segfault_function(void) {
    *(int*)0 = 42;
}

// 5. Abort
void abort_function(void) {
    abort();
}

// 6. Ignora alarm (tricky)
void ignore_alarm(void) {
    signal(SIGALRM, SIG_IGN);
    sleep(5);
}
```

### **Casos extremos de timeout**:
```c
sandbox(fast_function, 0, true);    // timeout = 0 
sandbox(slow_function, 1, true);    // timeout muy corto
sandbox(ok_function, 3600, true);   // timeout muy largo
```

---

## 🎯 COMPARACIÓN CON OTROS EJERCICIOS

### **vs picoshell**:
| Aspecto | sandbox | picoshell |
|---------|---------|-----------|
| Propósito | Ejecutar función con control | Pipeline de comandos |
| Procesos | 1 hijo (la función) | N hijos (los comandos) |
| Comunicación | No pipes (solo status) | Pipes entre comandos |
| Señales | SIGALRM para timeout | Solo manejo de errores |
| Sincronización | waitpid() específico | wait() múltiple |

### **vs ft_popen**:
| Aspecto | sandbox | ft_popen |
|---------|---------|----------|
| Función | Ejecutar y evaluar | Ejecutar y comunicar |
| Retorno | Resultado (1/0/-1) | Descriptor de archivo |
| Timeout | Sí (con SIGALRM) | No |
| Wait | Sí (espera terminación) | No (proceso sigue) |

---

## 🧠 CONCEPTOS CLAVE PARA RECORDAR

### **1. Double timeout mechanism**:
```c
// HIJO: alarm() mata al hijo si f() tarda mucho
// PADRE: alarm() interrumpe waitpid() si hijo no termina
```

### **2. Signal handling pattern**:
```c
// Configurar handler → waitpid() → analizar errno EINTR
```

### **3. Status analysis**:
```c
WIFEXITED → exit() normal → revisar WEXITSTATUS
WIFSIGNALED → muerte por señal → revisar WTERMSIG  
```

### **4. Zombie prevention**:
```c
// Siempre hacer waitpid() después de kill()
```

---

## 🎯 EJERCICIO DE AUTOEVALUACIÓN

**Sin mirar el código, responde**:

1. **¿Qué pasa si waitpid() retorna -1 con errno=EINTR?**
   - [ ] Error fatal, retornar -1
   - [ ] Timeout, matar hijo y retornar 0
   - [ ] Continuar esperando

2. **¿Por qué hacemos alarm() tanto en hijo como en padre?**
   - [ ] Es redundante, solo necesitamos uno
   - [ ] Hijo: mata f() lenta; Padre: interrumpe waitpid()
   - [ ] Es un error, solo debería ser en el padre

3. **¿Cuándo retornamos 1 (nice function)?**
   - [ ] Cuando f() no crashea
   - [ ] Cuando WIFEXITED && WEXITSTATUS == 0
   - [ ] Cuando no hay timeout

4. **Después de kill(pid, SIGKILL), ¿qué debemos hacer?**
   - [ ] Nada más
   - [ ] waitpid(pid, NULL, 0)  
   - [ ] alarm(0)

**Respuestas**: 1-b, 2-b, 3-b, 4-b

---

## 🚀 ¿ESTÁS LISTO PARA EL NIVEL 2?

**Puedes pasar al nivel 2 si**:
- ✅ Entiendes el mecanismo de double timeout
- ✅ Sabes manejar EINTR en waitpid()  
- ✅ Comprendes la diferencia entre WIFEXITED y WIFSIGNALED
- ✅ Puedes implementar sandbox desde cero en 20 minutos
- ✅ Has probado con funciones que crashean, timeout, etc.

**¡Ya tienes las bases para el NIVEL 2!** 🎉

**Próximo**: Tutorial de **vbc** (parser de expresiones matemáticas) 🧮