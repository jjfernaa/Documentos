# Análisis Completo de Minishell - Flujo y Arquitectura

## 1. VISIÓN GENERAL DEL PROGRAMA

Tu minishell es un intérprete de comandos simplificado que imita el comportamiento de bash. El programa sigue un ciclo principal de **lectura-análisis-ejecución** (Read-Eval-Print Loop) donde:

1. **Lee** la entrada del usuario
2. **Analiza** léxicamente y sintácticamente
3. **Expande** variables de entorno
4. **Ejecuta** comandos (builtin o externos)
5. **Limpia** memoria y repite

## 2. ESTRUCTURA DE DATOS FUNDAMENTALES

### 2.1 Estructura Principal - t_shell
```c
typedef struct s_shell
{
    char    *input;        // Entrada del usuario
    t_env   *env;         // Lista enlazada de variables de entorno
    t_token *tokens;      // Lista de tokens del lexer
    t_cmd   *cmd;         // Lista de comandos parseados
    int     infile_fd;    // Descriptor de archivo de entrada
    int     outfile_fd;   // Descriptor de archivo de salida
    int     exit_status;  // Estado de salida del último comando
} t_shell;
```

Esta estructura actúa como el "cerebro" del shell, conteniendo toda la información necesaria para procesar comandos.

### 2.2 Sistema de Tokens
```c
typedef struct s_token
{
    t_token_type    type;        // WORD, PIPE, REDIR_IN, etc.
    char           *value;       // Valor del token
    t_quote_type   quote_type;   // Tipo de comillas
    t_token_segment *segments;   // Segmentos para manejo de comillas
    struct s_token *next;        // Siguiente token
} t_token;
```

Los tokens son la representación intermedia entre el texto crudo y los comandos ejecutables.

### 2.3 Sistema de Comandos
```c
typedef struct s_cmd
{
    char           **argv;          // Argumentos del comando
    t_redir        *input_redirs;   // Redirecciones de entrada
    t_redir        *output_redirs;  // Redirecciones de salida
    int            heredoc;         // Flag para heredoc
    struct s_cmd   *next;           // Siguiente comando en pipeline
} t_cmd;
```

## 3. FLUJO PRINCIPAL DEL PROGRAMA

### 3.1 Inicialización (main.c)
```c
int main(int argc, char **argv, char **envp)
{
    t_shell shell;
    
    init_shell(&shell, envp);  // Inicializar estructura
    setup_signals();           // Configurar manejo de señales
    
    while (1) {
        shell.input = readline("minishell$ ");
        if (!shell.input) break;  // Ctrl+D
        
        if (*shell.input)
            add_history(shell.input);
            
        process_command(&shell);   // NÚCLEO del procesamiento
        cleanup_loop(&shell);      // Limpiar memoria
    }
    
    cleanup_shell(&shell);
    return shell.exit_status;
}
```

**Puntos clave:**
- **readline()**: Función que maneja entrada interactiva con historial
- **add_history()**: Guarda comandos para navegación con ↑↓
- El bucle infinito solo se rompe con Ctrl+D o comando `exit`

### 3.2 Inicialización del Shell
```c
void init_shell(t_shell *shell, char **envp)
{
    shell->env = init_env(envp);  // Copia el entorno del sistema
    shell->input = NULL;
    shell->tokens = NULL;
    shell->cmd = NULL;
    shell->exit_status = 0;
}
```

## 4. PROCESAMIENTO DE COMANDOS - EL CORAZÓN DEL SISTEMA

### 4.1 Función Central: process_command()
```c
void process_command(t_shell *shell)
{
    // 1. ANÁLISIS LÉXICO
    shell->tokens = lexer(shell->input);
    if (!shell->tokens) {
        shell->exit_status = 2;
        return;
    }
    
    // 2. EXPANSIÓN DE VARIABLES
    expand_var(shell);
    
    // 3. VALIDACIÓN SINTÁCTICA
    if (!validate_tokens(shell->tokens)) {
        shell->exit_status = 2;
        return;
    }
    
    // 4. EJECUCIÓN
    execute_parser_command(shell);
}
```

Este es el **núcleo del procesamiento**. Cada paso es crucial:

## 5. ANÁLISIS LÉXICO (LEXER)

### 5.1 Propósito del Lexer
El lexer convierte texto crudo en tokens estructurados. Por ejemplo:
- Input: `"echo 'hello world' | grep hello"`
- Output: `[WORD:"echo"] [WORD:"hello world"] [PIPE] [WORD:"grep"] [WORD:"hello"]`

### 5.2 Manejo de Comillas y Segmentos
**¿Por qué segmentos?**
```bash
echo "Hello $USER world"
```
Esto se divide en segmentos:
1. `"Hello "` (DOUBLE_QUOTE) 
2. `"$USER"` (DOUBLE_QUOTE - se expandirá)
3. `" world"` (DOUBLE_QUOTE)

### 5.3 Función Principal: lexer()
```c
t_token *lexer(const char *input)
{
    t_token *tokens = NULL;
    int i = 0;
    
    while (input[i]) {
        if (ft_isspace(input[i]))
            i++;  // Saltar espacios
        else if (input[i] == '|')
            add_token(&tokens, T_PIPE);
        else if (input[i] == '<' || input[i] == '>')
            handle_redirection(&tokens, input, &i);
        else
            handle_word(&tokens, input, &i);  // Palabras complejas
    }
    return tokens;
}
```

### 5.4 Manejo de Palabras Complejas
```c
static int handle_word(t_token **list, const char *input, int *i)
{
    t_token *token = add_token(list, T_WORD);
    
    // Continúa mientras no encuentre espacios o símbolos
    while (input[*i] && !ft_isspace(input[*i]) && !is_symbol(input[*i])) {
        char *result = NULL;
        t_quote_type quote_type;
        
        // Lee un segmento (con o sin comillas)
        if (!read_segment(input, i, &result, &quote_type))
            return 0;
            
        // Añade el segmento al token
        if (result) {
            add_segment_to_token(token, result, quote_type);
            free(result);
        }
    }
    return 1;
}
```

**Ejemplo práctico:**
- Input: `echo "hello $USER" world`
- Token 1: `echo` (1 segmento NO_QUOTE)
- Token 2: `hello $USER` (1 segmento DOUBLE_QUOTE) + `world` (1 segmento NO_QUOTE)

## 6. EXPANSIÓN DE VARIABLES

### 6.1 ¿Por qué es necesaria?
```bash
echo "Hello $USER, today is $(date)"
```
El shell debe:
1. Encontrar `$USER` y reemplazarlo con el valor real
2. Manejar `$?` (exit status)
3. Respetar las reglas de comillas (simples vs dobles)

### 6.2 Reglas de Expansión
- **Comillas simples**: NO se expande nada
- **Comillas dobles**: Se expanden variables pero no wildcards
- **Sin comillas**: Se expande todo

### 6.3 Función de Expansión
```c
void expand_var(t_shell *shell)
{
    t_token *current = shell->tokens;
    
    while (current) {
        if (current->type == T_WORD) {
            // Expandir usando segmentos
            char *new_value = expand_token_with_segments(current, 
                                shell->env, shell->exit_status);
            
            free(current->value);
            current->value = new_value ? new_value : ft_strdup("");
        }
        current = current->next;
    }
}
```

### 6.4 Expansión por Segmentos
```c
char *expand_token_with_segments(t_token *token, t_env *env, int exit_status)
{
    char *result = ft_strdup("");
    t_token_segment *current = token->segments;
    
    while (current) {
        char *expanded_segment;
        
        if (current->quote_type == SINGLE_QUOTE)
            expanded_segment = ft_strdup(current->text);  // Sin expandir
        else
            expanded_segment = expand_string(current->text, env, exit_status);
            
        // Concatenar resultado
        char *temp = ft_strjoin(result, expanded_segment);
        free(result);
        free(expanded_segment);
        result = temp;
        
        current = current->next;
    }
    return result;
}
```

## 7. VALIDACIÓN SINTÁCTICA

### 7.1 Errores que Detecta
```c
int validate_tokens(t_token *token)
{
    // Error: pipe al inicio
    if (token->type == T_PIPE) {
        print_syntax_error(token);
        return 0;
    }
    
    while (token) {
        if (token->type == T_PIPE) {
            // Error: pipe sin comando después
            if (!token->next || token->next->type == T_PIPE)
                return 0;
        }
        else if (is_redir(token)) {
            // Error: redirección sin archivo
            if (!token->next || token->next->type != T_WORD)
                return 0;
        }
        token = token->next;
    }
    return 1;
}
```

**Ejemplos de errores:**
- `| echo hello` → Error: pipe al inicio
- `echo hello |` → Error: pipe sin comando
- `echo > ` → Error: redirección sin archivo

## 8. PARSING - CONVERTIR TOKENS A COMANDOS

### 8.1 Estructura de Comandos
El parser convierte la lista de tokens en comandos ejecutables:
```bash
echo hello | grep h > output.txt
```
Se convierte en:
- **Comando 1**: `argv=["echo", "hello"]`, `output_redirs=NULL`
- **Comando 2**: `argv=["grep", "h"]`, `output_redirs=[{filename="output.txt", type=T_REDIR_OUT}]`

### 8.2 Función Principal del Parser
```c
t_cmd *parse_tokens(t_token *tokens)
{
    t_cmd *cmds = NULL;
    t_cmd *current_cmd = new_cmd();
    
    while (tokens) {
        if (tokens->type == T_WORD)
            current_cmd->argv = add_to_argv(current_cmd->argv, tokens->value);
        else if (tokens->type == T_PIPE) {
            cmd_add_back(&cmds, current_cmd);  // Guardar comando actual
            current_cmd = new_cmd();           // Crear nuevo comando
        }
        else if (is_redir(tokens)) {
            handle_redirection(tokens, current_cmd);
            tokens = tokens->next;  // Saltar el archivo
        }
        tokens = tokens->next;
    }
    
    cmd_add_back(&cmds, current_cmd);  // Último comando
    return cmds;
}
```

## 9. EJECUCIÓN DE COMANDOS

### 9.1 Decisión: Builtin vs Externo vs Pipeline
```c
static void execute_parser_command(t_shell *shell)
{
    if (has_pipes_or_redirects(shell->tokens)) {
        // Comando complejo con pipes/redirections
        t_cmd *cmds = parse_tokens(shell->tokens);
        execute_pipeline(cmds, shell);
        free_cmds(cmds);
    }
    else {
        // Comando simple
        char **args = tokens_to_args(shell->tokens);
        
        int builtin_result = execute_builtin(args, shell);
        if (builtin_result == -1)  // No es builtin
            shell->exit_status = execute_external(args, shell);
        else
            shell->exit_status = builtin_result;
            
        free_array(args);
    }
}
```

### 9.2 Comandos Builtin
Los builtins son comandos implementados directamente en el shell:
- `pwd`, `echo`, `env`, `cd`, `export`, `unset`, `exit`

```c
int execute_builtin(char **args, t_shell *shell)
{
    if (ft_strcmp(args[0], "pwd") == 0)
        return builtin_pwd();
    else if (ft_strcmp(args[0], "echo") == 0)
        return builtin_echo(args, shell);
    // ... más builtins
    else
        return -1;  // No es builtin
}
```

**¿Por qué algunos comandos son builtin?**
- `cd`: Debe cambiar el directorio del proceso padre (shell)
- `export`: Debe modificar el entorno del shell
- `exit`: Debe terminar el shell, no crear un proceso hijo

### 9.3 Comandos Externos
```c
int execute_external(char **args, t_shell *shell)
{
    char **envp = env_to_array(shell->env);  // Convertir entorno
    char *path = find_executable(args[0], envp);  // Buscar en PATH
    
    if (!path) {
        print_cmd_not_found(args[0]);
        free_array(envp);
        return 127;  // Comando no encontrado
    }
    
    pid_t pid = fork();  // Crear proceso hijo
    
    if (pid == 0) {
        // PROCESO HIJO
        setup_signals_child();
        execve(path, args, envp);  // Reemplazar proceso
        perror("execve");
        exit(126);
    }
    else if (pid > 0) {
        // PROCESO PADRE
        int status;
        waitpid(pid, &status, 0);  // Esperar al hijo
        
        if (WIFEXITED(status))
            return WEXITSTATUS(status);
        if (WIFSIGNALED(status))
            return 128 + WTERMSIG(status);
    }
    
    return 1;  // Error en fork
}
```

**Conceptos clave:**
- **fork()**: Crea una copia exacta del proceso actual
- **execve()**: Reemplaza el proceso actual con el programa especificado
- **waitpid()**: El padre espera a que termine el hijo

## 10. SISTEMA DE PIPES - EL MÁS COMPLEJO

### 10.1 ¿Qué son los Pipes?
Un pipe conecta la salida estándar de un comando con la entrada estándar del siguiente:
```bash
echo "hello world" | grep hello | wc -l
```

### 10.2 Creación de Pipes
```c
int **create_pipes(int pipe_count)
{
    int **pipes = malloc(sizeof(int *) * pipe_count);
    
    for (int i = 0; i < pipe_count; i++) {
        pipes[i] = malloc(sizeof(int) * 2);
        if (pipe(pipes[i]) == -1) {
            // Error: limpiar pipes creados
            handle_pipe_error(pipes, i);
            return NULL;
        }
    }
    return pipes;
}
```

**Para N comandos necesitas N-1 pipes:**
- `cmd1 | cmd2 | cmd3` → 2 pipes
- `pipe[0]`: conecta cmd1 con cmd2
- `pipe[1]`: conecta cmd2 con cmd3

### 10.3 Configuración de Pipes en Procesos Hijos
```c
void setup_child_pipes(int **pipes, int cmd_index, int cmd_count)
{
    if (cmd_index == 0) {
        // PRIMER COMANDO: solo redirigir stdout al pipe
        dup2(pipes[0][1], STDOUT_FILENO);
    }
    else if (cmd_index == cmd_count - 1) {
        // ÚLTIMO COMANDO: solo redirigir stdin desde pipe
        dup2(pipes[cmd_index - 1][0], STDIN_FILENO);
    }
    else {
        // COMANDOS INTERMEDIOS: redirigir stdin y stdout
        dup2(pipes[cmd_index - 1][0], STDIN_FILENO);   // Entrada del pipe anterior
        dup2(pipes[cmd_index][1], STDOUT_FILENO);      // Salida al pipe actual
    }
    
    // CRUCIAL: Cerrar todos los descriptores en el hijo
    close_all_pipes_in_child(pipes, cmd_count - 1);
}
```

### 10.4 Ejecución Completa de Pipeline
```c
void execute_pipeline_real(t_cmd *cmds, t_shell *shell)
{
    int cmd_count = count_commands(cmds);
    int **pipes = create_pipes(cmd_count - 1);  // N-1 pipes para N comandos
    pid_t *pids = malloc(sizeof(pid_t) * cmd_count);
    
    t_cmd *current = cmds;
    
    // Crear un proceso hijo para cada comando
    for (int i = 0; i < cmd_count; i++) {
        pids[i] = fork();
        
        if (pids[i] == 0) {
            // PROCESO HIJO
            setup_child_pipes(pipes, i, cmd_count);
            execute_single_cmd(current, shell);  // No retorna
        }
        
        current = current->next;
    }
    
    // PROCESO PADRE
    cleanup_pipeline(pipes, pids, cmd_count, shell);
    free(pids);
}
```

**Flujo visual para `cmd1 | cmd2 | cmd3`:**
```
Proceso Padre:
├── fork() → Hijo1 (cmd1)
│   ├── stdout → pipe[0][1]
│   └── ejecuta cmd1
├── fork() → Hijo2 (cmd2)  
│   ├── stdin ← pipe[0][0]
│   ├── stdout → pipe[1][1]
│   └── ejecuta cmd2
├── fork() → Hijo3 (cmd3)
│   ├── stdin ← pipe[1][0]
│   └── ejecuta cmd3
└── espera a todos los hijos
```

## 11. REDIRECCIONES

### 11.1 Tipos de Redirección
```c
typedef enum e_token_type
{
    T_REDIR_IN,    // <  : redirigir entrada desde archivo
    T_REDIR_OUT,   // >  : redirigir salida a archivo (sobrescribir)
    T_APPEND,      // >> : redirigir salida a archivo (anexar)
    T_HEREDOC      // << : entrada multilínea hasta delimitador
} t_token_type;
```

### 11.2 Aplicación de Redirecciones
```c
void apply_redirections(t_cmd *cmd)
{
    // Procesar redirecciones de entrada
    if (cmd->input_redirs) {
        if (process_input_redirections(cmd->input_redirs) != 0)
            exit(1);
    }
    
    // Procesar redirecciones de salida
    if (cmd->output_redirs) {
        if (process_output_redirection(cmd->output_redirs) != 0)
            exit(1);
    }
}
```

### 11.3 Heredoc - El Más Complejo
```bash
cat << EOF
Esta es una entrada
multilínea que termina
cuando encuentro EOF
EOF
```

**Implementación:**
```c
int create_heredoc_pipe(char *delimiter)
{
    int p[2];
    if (pipe(p) == -1) return -1;
    
    pid_t pid = fork();
    
    if (pid == 0) {
        // PROCESO HIJO: lee entrada hasta delimiter
        close(p[0]);  // No necesita leer
        
        char *line;
        while ((line = readline("> "))) {
            if (ft_strcmp(line, delimiter) == 0) {
                free(line);
                break;
            }
            write(p[1], line, ft_strlen(line));
            write(p[1], "\n", 1);
            free(line);
        }
        close(p[1]);
        exit(0);
    }
    else if (pid > 0) {
        // PROCESO PADRE
        close(p[1]);  // No necesita escribir
        
        int status;
        waitpid(pid, &status, 0);
        
        if (WIFSIGNALED(status) && WTERMSIG(status) == SIGINT)
            return -1;  // Usuario canceló con Ctrl+C
            
        return p[0];  // Devolver descriptor de lectura
    }
    
    return -1;
}
```

## 12. MANEJO DE SEÑALES

### 12.1 ¿Por qué son Importantes?
Las señales son interrupciones asíncronas que permiten:
- **Ctrl+C (SIGINT)**: Interrumpir comando actual
- **Ctrl+\ (SIGQUIT)**: Terminar con core dump
- **Ctrl+D (EOF)**: Cerrar entrada estándar

### 12.2 Configuración Inicial
```c
void setup_signals(void)
{
    signal(SIGINT, handle_sigint);   // Manejar Ctrl+C
    signal(SIGQUIT, SIG_IGN);        // Ignorar Ctrl+\
}
```

### 12.3 Manejador de SIGINT
```c
void handle_sigint(int sig)
{
    g_signal_received = SIGINT;
    write(STDERR_FILENO, "\n", 1);   // Nueva línea
    
    if (rl_done == 0) {  // Si readline está activo
        rl_on_new_line();
        rl_replace_line("", 0);  // Limpiar línea
        rl_redisplay();          // Mostrar nuevo prompt
    }
}
```

### 12.4 Señales en Procesos Hijos
```c
void setup_signals_child(void)
{
    signal(SIGINT, SIG_DFL);   // Comportamiento por defecto
    signal(SIGQUIT, SIG_DFL);  // Comportamiento por defecto
}
```

**¿Por qué cambiar en hijos?**
- El shell debe continuar ejecutándose tras Ctrl+C
- Los comandos externos deben poder ser interrumpidos

## 13. GESTIÓN DE MEMORIA

### 13.1 Principio Fundamental
**"Quien aloca, libera"** - Cada malloc() debe tener su free() correspondiente.

### 13.2 Limpieza por Ciclo
```c
void cleanup_loop(t_shell *shell)
{
    if (shell->input) free(shell->input);
    if (shell->tokens) free_tokens(shell->tokens);
    if (shell->cmd) free_cmds(shell->cmd);
    
    // Resetear punteros
    shell->input = NULL;
    shell->tokens = NULL; 
    shell->cmd = NULL;
}
```

### 13.3 Limpieza Final
```c
void cleanup_shell(t_shell *shell)
{
    cleanup_loop(shell);
    if (shell->env) free_env(shell->env);
    rl_clear_history();  // Liberar historial de readline
}
```

### 13.4 Funciones de Liberación Específicas
```c
void free_tokens(t_token *tokens)
{
    while (tokens) {
        t_token *tmp = tokens;
        tokens = tokens->next;
        
        if (tmp->segments) free_segments(tmp->segments);
        if (tmp->value) free(tmp->value);
        free(tmp);
    }
}
```

## 14. CASOS EDGE Y MANEJO DE ERRORES

### 14.1 Comandos Vacíos
```bash
minishell$     # Solo espacios → no hacer nada
minishell$ |  # Pipe inválido → error sintáctico  
```

### 14.2 Archivos Inexistentes
```c
static int open_redirection_file(t_redir *redir)
{
    int fd;
    
    if (redir->type == T_REDIR_IN) {
        fd = open(redir->filename, O_RDONLY);
        if (fd < 0) {
            perror(redir->filename);  // "archivo: No such file or directory"
            return -1;
        }
    }
    // ... otros tipos
    
    return fd;
}
```

### 14.3 Comandos No Encontrados
```c
char *find_executable(char *command, char **envp)
{
    // Si tiene '/', es ruta absoluta/relativa
    if (ft_strchr(command, '/'))
        return check_direct_path(command);
    
    // Buscar en PATH
    char *path_env = get_env_value("PATH", envp);
    if (!path_env) return NULL;
    
    // Dividir PATH y buscar en cada directorio
    char **dirs = ft_split(path_env, ':');
    return search_in_dirs(dirs, command);
}
```

## 15. FLUJO COMPLETO - EJEMPLO PRÁCTICO

Analicemos: `echo "Hello $USER" | grep $USER > output.txt`

### Paso 1: Lexer
```
Input: echo "Hello $USER" | grep $USER > output.txt
Tokens: [WORD:"echo"] [WORD:"Hello $USER"] [PIPE] [WORD:"grep"] [WORD:"$USER"] [REDIR_OUT] [WORD:"output.txt"]
```

### Paso 2: Expansión (asumiendo USER=juan)
```
Tokens: [WORD:"echo"] [WORD:"Hello juan"] [PIPE] [WORD:"grep"] [WORD:"juan"] [REDIR_OUT] [WORD:"output.txt"]
```

### Paso 3: Parser
```
Comando 1: argv=["echo", "Hello juan"]
Comando 2: argv=["grep", "juan"], output_redirs=[{filename="output.txt", type=T_REDIR_OUT}]
```

### Paso 4: Ejecución
```
1. Crear pipe[0]
2. fork() → Proceso hijo 1:
   - dup2(pipe[0][1], STDOUT_FILENO)  // Salida al pipe
   - execve("/bin/echo", ["echo", "Hello juan"], envp)
3. fork() → Proceso hijo 2:
   - dup2(pipe[0][0], STDIN_FILENO)   // Entrada del pipe
   - open("output.txt", O_CREAT|O_WRONLY|O_TRUNC)
   - dup2(output_fd, STDOUT_FILENO)   // Salida al archivo
   - execve("/bin/grep", ["grep", "juan"], envp)
4. Proceso padre:
   - close_all_pipes()
   - waitpid() para ambos hijos
```

## 16. CONCEPTOS CLAVE PARA TU DESARROLLO

### 16.1 Descriptores de Archivo
- **0 (STDIN_FILENO)**: Entrada estándar
- **1 (STDOUT_FILENO)**: Salida estándar  
- **2 (STDERR_FILENO)**: Error estándar
- **dup2(old_fd, new_fd)**: Duplica descriptor, redirigiendo flujos

### 16.2 Procesos
- **fork()**: Crea proceso hijo idéntico
- **execve()**: Reemplaza proceso actual con nuevo programa
- **wait/waitpid()**: Sincronización padre-hijo
- **PID**: Identificador único de proceso

### 16.3 Pipes
- **pipe()**: Crea par de descriptores conectados
- **[0]**: Extremo de lectura
- **[1]**: Extremo de escritura
- Siempre cerrar descriptores no utilizados

### 16.4 Señales
- Mecanismo de comunicación asíncrona
- **signal()**: Registra manejador
- **SIG_DFL**: Comportamiento por defecto
- **SIG_IGN**: Ignorar señal

### 16.5 Gestión de Memoria
- **malloc/free**: Gestión explícita
- **Listas enlazadas**: Estructuras dinámicas
- **Cleanup functions**: Liberación sistemática
- **Valgrind**: Herramienta para detectar leaks

## 17. PATRONES DE DISEÑO UTILIZADOS

### 17.1 State Machine
El shell mantiene estado (variables, directorio, exit status) entre comandos.

### 17.2 Chain of Responsibility  
Procesamiento secuencial: lexer → expander → parser → executor

### 17.3 Strategy Pattern
Diferentes estrategias de ejecución según tipo de comando.

### 17.4 Command Pattern
Encapsulación de comandos en estructuras t_cmd.

## CONCLUSIÓN

Tu minishell es un proyecto complejo que integra múltiples conceptos de sistemas operativos:

1. **Gestión de procesos** (fork, exec, wait)
2. **Comunicación entre procesos** (pipes, signals)
3. **Sistema de archivos** (redirecciones, descriptores)
4. **Análisis sintáctico** (lexer, parser)
5. **Gestión de memoria** (malloc/free, listas enlazadas)

Cada componente tiene su responsabilidad específica, pero todos trabajan juntos para crear una experiencia de shell funcional. La clave está en entender cómo fluye la información y cómo cada paso transforma los datos hasta llegar a la ejecución final.

El diseño modular permite mantener, debuggear y extender el código fácilmente, siguiendo principios de ingeniería de software sólidos.