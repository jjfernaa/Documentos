# 📚 **RESUMEN TEÓRICO COMPLETO: MINISHELL**

## 🎯 **¿QUÉ ES UN SHELL?**

### **🔍 Definición básica:**
Un **shell** es un **intérprete de comandos** - un programa que:
1. **Recibe** comandos del usuario
2. **Interpreta** lo que quiere hacer
3. **Ejecuta** el comando solicitado
4. **Muestra** el resultado

### **📊 Analogía simple:**
Imagina un **traductor** entre tú y tu computadora:
- **Tú dices**: "ls" 
- **El shell traduce**: "Muéstrame los archivos del directorio actual"
- **La computadora ejecuta**: Lista los archivos
- **Tú ves**: Los archivos en pantalla

---

## 🏗️ **ARQUITECTURA DE NUESTRO MINISHELL**

### **📋 Componentes principales:**

```
INPUT → LEXER → PARSER → EXECUTION → OUTPUT
  ↑        ↑        ↑         ↑         ↑
Usuario  Tokens  Comandos  Procesos  Resultado
```

---

## 🧠 **1. LEXER (Analizador Léxico)**

### **🎯 ¿Qué es un LEXER?**
Un **lexer** es como un **"lector inteligente"** que convierte texto en "piezas comprensibles" llamadas **tokens**.

### **📝 Ejemplo práctico:**
```bash
Input: echo "hello world" | cat > file.txt
                ↓ LEXER ↓
Tokens:
- T_WORD: "echo"
- T_WORD: "hello world" (con flag de double quotes)
- T_PIPE: "|"
- T_WORD: "cat"
- T_REDIR_OUT: ">"
- T_WORD: "file.txt"
```

### **🔧 Tipos de tokens en nuestro proyecto:**
```c
typedef enum e_token_type
{
    T_WORD,        // Palabras normales: echo, ls, archivo.txt
    T_PIPE,        // Pipe: |
    T_REDIR_IN,    // Redirección entrada: <
    T_REDIR_OUT,   // Redirección salida: >
    T_APPEND,      // Append: >>
    T_HEREDOC      // Here document: <<
} t_token_type;
```

### **💡 ¿Por qué necesitamos un lexer?**
Sin lexer: `echo "hello world" | cat` es solo una cadena confusa
Con lexer: Son 4 tokens claros que podemos procesar individualmente

---

## 🌍 **2. VARIABLES DE ENTORNO (Environment Variables)**

### **🎯 ¿Qué son las variables de entorno?**
Son **"configuraciones globales"** del sistema que todos los programas pueden usar.

### **📝 Ejemplos cotidianos:**
```bash
PATH=/bin:/usr/bin:/usr/local/bin    # Dónde buscar programas
HOME=/Users/juan                     # Directorio del usuario
USER=juan                           # Nombre del usuario actual
SHELL=/bin/bash                     # Shell por defecto
```

### **🔍 Analogía:**
Imagina las variables de entorno como **"configuraciones de tu teléfono"**:
- **Idioma**: Español (todos los apps lo usan)
- **Zona horaria**: Madrid (todos los apps la respetan)
- **Tema**: Oscuro (todos los apps lo aplican)

### **🏗️ Nuestra implementación:**
```c
typedef struct s_env
{
    char        *key;      // "PATH"
    char        *value;    // "/bin:/usr/bin"
    struct s_env *next;    // Siguiente variable
} t_env;
```

### **🎯 ¿Para qué las usamos?**
1. **export VAR=valor**: Crear/modificar variables
2. **unset VAR**: Eliminar variables
3. **$VAR**: Expandir variables en comandos
4. **env**: Mostrar todas las variables

---

## 🛣️ **3. PATH (Rutas de Ejecutables)**

### **🎯 ¿Qué es el PATH?**
El **PATH** es una **variable de entorno especial** que contiene una **lista de directorios** donde el sistema busca programas ejecutables.

### **📝 Ejemplo real:**
```bash
PATH="/bin:/usr/bin:/usr/local/bin"
```

### **🔍 ¿Cómo funciona?**
Cuando escribes `ls`, el shell busca en orden:
1. **`/bin/ls`** ← ¡Encontrado! Ejecuta este
2. `/usr/bin/ls` ← No lo revisa (ya encontró)
3. `/usr/local/bin/ls` ← No lo revisa

### **📊 Tipos de comandos:**
```bash
# Comando con PATH completo (absoluto):
/bin/ls                    # Ejecuta directamente /bin/ls

# Comando simple (busca en PATH):
ls                         # Busca "ls" en todos los directorios del PATH

# Comando relativo:
./mi_programa              # Ejecuta programa en directorio actual
```

### **💡 Implementación en nuestro proyecto:**
```c
char *find_executable(char *command)
{
    if (ft_strchr(command, '/'))        // Si tiene '/', es path directo
        return check_direct_path(command);
    else
        return search_in_path(command);  // Buscar en PATH
}
```

---

## ⚙️ **4. EXECUTION ENGINE (Motor de Ejecución)**

### **🎯 ¿Qué es un execution engine?**
Es el **"cerebro ejecutor"** que decide **cómo ejecutar** cada comando y lo hace realidad.

### **🔧 Dos tipos de comandos:**

#### **🏠 Builtins (Comandos Internos):**
```c
// Comandos que ejecuta el propio shell:
pwd, echo, cd, env, export, unset, exit
```
**¿Por qué internos?** Porque necesitan **modificar el estado del shell**:
- `cd`: Cambia directorio **del shell**
- `export`: Modifica variables **del shell**
- `exit`: Termina **el shell**

#### **🌐 External Commands (Comandos Externos):**
```c
// Programas separados del sistema:
ls, cat, grep, python3, vim, etc.
```

### **🔄 Proceso de ejecución externa:**
```c
1. fork()     // Crear proceso hijo (copia del shell)
2. execve()   // En el hijo: reemplazar por el programa deseado
3. waitpid()  // En el padre: esperar que termine el hijo
4. WEXITSTATUS() // Obtener código de salida
```

### **📊 Flujo completo:**
```
Comando "ls" →
¿Es builtin? NO →
Buscar en PATH → /bin/ls →
fork() → execve("/bin/ls") → wait() → Exit code
```

---

## 🚦 **5. SIGNALS (Señales del Sistema)**

### **🎯 ¿Qué son las signals?**
Las **signals** son **"mensajes especiales"** que el sistema operativo envía a los programas para comunicar eventos importantes.

### **📝 Signals importantes en minishell:**
```c
SIGINT  (Ctrl+C)  → "Interrumpir proceso"
SIGQUIT (Ctrl+\)  → "Salir con core dump" 
SIGTERM           → "Terminar ordenadamente"
```

### **🎯 Comportamiento requerido:**
```bash
# En el prompt:
minishell$ ^C           # Nueva línea, nuevo prompt (NO salir)
minishell$ 

# Durante comando:
minishell$ sleep 10
^C                      # Termina sleep, NO el shell
minishell$ 
```

### **🔧 Implementación:**
```c
void setup_signals(void)
{
    signal(SIGINT, handle_sigint);    // Manejar Ctrl+C
    signal(SIGQUIT, SIG_IGN);         // Ignorar Ctrl+\
}

void handle_sigint(int sig)
{
    write(STDOUT_FILENO, "\n", 1);    // Nueva línea
    rl_on_new_line();                 // Avisar a readline
    rl_redisplay();                   // Redibujar prompt
}
```

### **💡 ¿Por qué `write()` y no `printf()`?**
Las funciones de signal handlers deben ser **"async-signal-safe"**. `write()` es segura, `printf()` no.

---

## 🎭 **6. PROCESS MANAGEMENT (Gestión de Procesos)**

### **🎯 Conceptos fundamentales:**

#### **🔄 fork():**
```c
pid_t pid = fork();
// Ahora existen 2 procesos idénticos:
// - Padre: pid > 0 (recibe PID del hijo)
// - Hijo:  pid = 0
```

#### **🚀 execve():**
```c
execve("/bin/ls", args, envp);
// Reemplaza el proceso actual por /bin/ls
// Si execve() retorna = ERROR (solo retorna en error)
```

#### **⏳ wait() / waitpid():**
```c
wait(&status);                    // Esperar a cualquier hijo
waitpid(pid, &status, 0);        // Esperar a hijo específico
```

#### **📊 Exit codes:**
```c
if (WIFEXITED(status))           // ¿Terminó normalmente?
    exit_code = WEXITSTATUS(status);  // Obtener código de salida
```

### **📝 Flujo completo con ejemplo:**
```bash
Usuario: "ls -la"
                ↓
1. fork() → Crear hijo
2. En hijo: execve("/bin/ls", ["ls", "-la", NULL], envp)
3. En padre: waitpid(hijo_pid, &status, 0)
4. Obtener exit code: WEXITSTATUS(status)
5. Continuar shell con nuevo prompt
```

---

## 🏗️ **7. ESTRUCTURAS DE DATOS**

### **🎯 Estructura principal del shell:**
```c
typedef struct s_shell
{
    char    **envp;         // Array de variables de entorno
    int     exit_status;    // Último código de salida
    t_token *tokens;        // Lista de tokens del lexer
    t_env   *env;          // Lista enlazada de variables
} t_shell;
```

### **🔗 Lista de tokens (del lexer):**
```c
typedef struct s_token
{
    t_token_type    type;           // T_WORD, T_PIPE, etc.
    char           *value;          // "echo", "|", "file.txt"
    int            single_quotes;   // 1 si está en 'quotes'
    int            double_quotes;   // 1 si está en "quotes"
    struct s_token *next;          // Siguiente token
} t_token;
```

### **🌍 Lista de variables de entorno:**
```c
typedef struct s_env
{
    char           *key;     // "PATH"
    char           *value;   // "/bin:/usr/bin"
    struct s_env   *next;    // Siguiente variable
} t_env;
```

---

## 🧪 **8. MANEJO DE ERRORES Y EXIT CODES**

### **🎯 Exit codes estándar:**
```c
0    →  Éxito
1    →  Error general
2    →  Uso incorrecto (argumentos incorrectos)
126  →  Comando no ejecutable
127  →  Comando no encontrado
130  →  Terminado por Ctrl+C (128 + 2)
```

### **📝 Ejemplos prácticos:**
```bash
minishell$ pwd
/Users/juan
minishell$ echo $?        # ← El exit code de pwd
0

minishell$ ls /no/existe
ls: /no/existe: No such file or directory
minishell$ echo $?
1

minishell$ /usr/bin/no_existe
command not found
minishell$ echo $?
127
```

---

## 🔧 **9. FUNCIONES PERMITIDAS (Subject)**

### **✅ Process Management:**
```c
fork()      → Crear proceso hijo
execve()    → Ejecutar programa
wait()      → Esperar proceso hijo
waitpid()   → Esperar proceso específico
```

### **✅ File Operations:**
```c
access()    → Verificar permisos de archivo
open()      → Abrir archivo
close()     → Cerrar archivo
read()      → Leer archivo
write()     → Escribir archivo
```

### **✅ Directory Operations:**
```c
getcwd()    → Obtener directorio actual
chdir()     → Cambiar directorio
```

### **✅ Signal Handling:**
```c
signal()    → Configurar manejador de señal
kill()      → Enviar señal a proceso
```

### **✅ Memory Management:**
```c
malloc()    → Asignar memoria
free()      → Liberar memoria
```

---

## 🎯 **10. FLUJO COMPLETO DE EJECUCIÓN**

### **📋 Ejemplo paso a paso: `echo "hello" | cat`**

#### **1. Input del usuario:**
```
"echo \"hello\" | cat"
```

#### **2. Lexer (Tokenización):**
```c
Tokens:
[T_WORD: "echo"]
[T_WORD: "hello" (double_quotes=1)]
[T_PIPE: "|"]
[T_WORD: "cat"]
```

#### **3. Parser (Análisis):**
```c
Comando 1: ["echo", "hello"]
Pipe: |
Comando 2: ["cat"]
```

#### **4. Execution:**
```c
// Para cada comando:
if (is_builtin("echo"))
    result = execute_builtin(["echo", "hello"], shell);
else
    result = execute_external(["echo", "hello"], shell);
```

#### **5. Result:**
```bash
hello
```

---

## 📚 **11. CONCEPTOS CLAVE PARA EXPLICAR**

### **🎯 ¿Qué le dirías a alguien que no sabe nada?**

#### **🔍 Lexer:**
*"Es como leer una oración y separar cada palabra para entender qué significa cada parte"*

#### **🌍 Variables de entorno:**
*"Son como configuraciones globales de tu computadora que todos los programas pueden usar"*

#### **🛣️ PATH:**
*"Es como una lista de lugares donde tu computadora busca programas cuando le dices 'ejecuta esto'"*

#### **⚙️ Execution:**
*"Es el proceso de realmente hacer que tu computadora ejecute el programa que pediste"*

#### **🚦 Signals:**
*"Son como mensajes urgentes que el sistema envía a los programas - como cuando presionas Ctrl+C"*

#### **🔄 Fork/Exec:**
*"Fork es como hacer una copia de un proceso, y exec es como reemplazar esa copia con otro programa"*

---

## 🏆 **12. LO QUE HEMOS LOGRADO**

### **✅ Funcionalidades completadas:**
1. **✅ Builtins completos**: pwd, echo, env, cd, exit, export, unset
2. **✅ Lexer funcional**: Tokenización completa con quotes
3. **✅ Signals básicos**: Ctrl+C maneja correctamente
4. **✅ Environment**: Gestión de variables de entorno
5. **✅ External commands**: Ejecución de programas del sistema
6. **✅ Error handling**: Códigos de salida correctos
7. **✅ Memory management**: Sin leaks
8. **✅ Process management**: fork/execve/wait correcto

### **🔵 Por implementar:**
1. **🔵 PATH resolution**: Búsqueda automática en PATH
2. **🔵 Pipes**: cmd1 | cmd2
3. **🔵 Redirections**: >, <, >>
4. **🔵 Variable expansion**: $VAR, $?
5. **🔵 Quote processing**: Diferencias entre "..." y '...'

---

## 💡 **13. PREGUNTAS PARA AUTOEXAMINARTE**

### **🤔 Preguntas técnicas:**
1. ¿Qué pasa cuando haces `fork()`?
2. ¿Por qué `cd` tiene que ser un builtin?
3. ¿Cómo encuentra el shell el programa `ls`?
4. ¿Qué significa el exit code 127?
5. ¿Por qué usamos `write()` en signal handlers?

### **🎯 Preguntas de comprensión:**
1. ¿Cómo explicarías un lexer a un niño de 10 años?
2. ¿Para qué sirven las variables de entorno en la vida real?
3. ¿Qué pasaría si no existiera el PATH?
4. ¿Por qué necesitamos signals en un shell?

---

