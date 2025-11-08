# 📚 Guía Completa de MLX42

## Índice
1. Introducción
2. Conceptos Básicos
3. Inicialización y Ventanas
4. Sistema de Imágenes
5. Colores
6. Texturas
7. Hooks y Callbacks
8. Input del Usuario
9. Shaders (Avanzado)
10. XPM42 - Formato de Imágenes
11. Funciones Completas

---

## Introducción

**MLX42** es una librería gráfica simple y eficiente basada en OpenGL, diseñada específicamente para proyectos de 42 School. Es una versión moderna y mejorada de la antigua MiniLibX.

### Características principales:
- ✅ Multiplataforma (Linux, macOS, Windows)
- ✅ Basada en OpenGL moderno
- ✅ Soporte para transparencia (canal Alpha)
- ✅ Sistema de hooks flexible
- ✅ Gestión eficiente de imágenes y texturas
- ✅ Input avanzado de teclado y ratón

---

## Conceptos Básicos

### Estructura del Programa

Todo programa con MLX42 sigue este flujo:

```c
#include <MLX42/MLX42.h>

int main(void)
{
    // 1. Inicializar MLX
    mlx_t* mlx = mlx_init(WIDTH, HEIGHT, "Título", true);
    
    // 2. Crear imágenes/texturas
    mlx_image_t* img = mlx_new_image(mlx, WIDTH, HEIGHT);
    
    // 3. Dibujar en la imagen
    mlx_put_pixel(img, x, y, color);
    
    // 4. Mostrar imagen en ventana
    mlx_image_to_window(mlx, img, 0, 0);
    
    // 5. Configurar hooks (callbacks)
    mlx_loop_hook(mlx, &mi_funcion, mlx);
    
    // 6. Iniciar loop principal
    mlx_loop(mlx);
    
    // 7. Limpiar recursos
    mlx_terminate(mlx);
    return (0);
}
```

### Tipos de Datos Principales

```c
// Estructura principal de MLX
typedef struct mlx_t mlx_t;

// Imagen (buffer de píxeles)
typedef struct mlx_image_t
{
    uint8_t*    pixels;     // Array de píxeles (RGBA)
    uint32_t    width;      // Ancho de la imagen
    uint32_t    height;     // Alto de la imagen
    int32_t     instances;  // Número de instancias dibujadas
    bool        enabled;    // Si está visible o no
} mlx_image_t;

// Textura (imagen cargada desde archivo)
typedef struct mlx_texture_t
{
    uint8_t*    pixels;     // Array de píxeles (RGBA)
    uint32_t    width;      // Ancho
    uint32_t    height;     // Alto
    uint32_t    bytes_per_pixel; // Bytes por píxel (normalmente 4)
} mlx_texture_t;
```

---

## Inicialización y Ventanas

### Funciones de Inicialización

#### `mlx_init()`
Crea una ventana y inicializa el contexto de MLX.

```c
mlx_t* mlx_init(int32_t width, int32_t height, const char* title, bool resize);
```

**Parámetros:**
- `width`: Ancho de la ventana en píxeles
- `height`: Alto de la ventana en píxeles
- `title`: Título de la ventana
- `resize`: Si la ventana puede cambiar de tamaño

**Retorno:** Puntero a `mlx_t` o `NULL` si falla

**Ejemplo:**
```c
mlx_t* mlx = mlx_init(1920, 1080, "Mi Juego", true);
if (!mlx)
{
    fprintf(stderr, "Error al inicializar MLX\n");
    exit(1);
}
```

#### `mlx_terminate()`
Libera todos los recursos y cierra la ventana.

```c
void mlx_terminate(mlx_t* mlx);
```

**Ejemplo:**
```c
mlx_terminate(mlx);
```

#### `mlx_close_window()`
Cierra la ventana pero no libera recursos.

```c
void mlx_close_window(mlx_t* mlx);
```

### Información de la Ventana

#### `mlx_get_monitor_size()`
Obtiene el tamaño del monitor.

```c
void mlx_get_monitor_size(int32_t index, int32_t* width, int32_t* height);
```

**Ejemplo:**
```c
int32_t width, height;
mlx_get_monitor_size(0, &width, &height);
printf("Resolución: %dx%d\n", width, height);
```

#### `mlx_set_window_size()`
Cambia el tamaño de la ventana.

```c
void mlx_set_window_size(mlx_t* mlx, int32_t width, int32_t height);
```

#### `mlx_set_window_pos()`
Cambia la posición de la ventana.

```c
void mlx_set_window_pos(mlx_t* mlx, int32_t x, int32_t y);
```

#### `mlx_set_window_limit()`
Establece límites de tamaño para la ventana.

```c
void mlx_set_window_limit(mlx_t* mlx, int32_t min_w, int32_t min_h, 
                          int32_t max_w, int32_t max_h);
```

---

## Sistema de Imágenes

### Crear y Gestionar Imágenes

#### `mlx_new_image()`
Crea una nueva imagen en memoria.

```c
mlx_image_t* mlx_new_image(mlx_t* mlx, uint32_t width, uint32_t height);
```

**Retorno:** Nueva imagen o `NULL` si falla

**Ejemplo:**
```c
mlx_image_t* img = mlx_new_image(mlx, 800, 600);
if (!img)
{
    fprintf(stderr, "Error al crear imagen\n");
    exit(1);
}
```

#### `mlx_delete_image()`
Elimina una imagen de la memoria.

```c
void mlx_delete_image(mlx_t* mlx, mlx_image_t* image);
```

**Ejemplo:**
```c
mlx_delete_image(mlx, img);
```

#### `mlx_image_to_window()`
Dibuja una imagen en la ventana.

```c
int32_t mlx_image_to_window(mlx_t* mlx, mlx_image_t* img, int32_t x, int32_t y);
```

**Parámetros:**
- `x, y`: Posición donde dibujar la imagen

**Retorno:** ID de la instancia o `-1` si falla

**Ejemplo:**
```c
// Dibujar en el centro de la ventana
int32_t instance = mlx_image_to_window(mlx, img, 400, 300);
```

### Manipulación de Píxeles

#### `mlx_put_pixel()`
Dibuja un píxel en una imagen.

```c
void mlx_put_pixel(mlx_image_t* image, uint32_t x, uint32_t y, uint32_t color);
```

**Ejemplo:**
```c
// Dibujar un píxel rojo en (10, 10)
mlx_put_pixel(img, 10, 10, 0xFF0000FF);
```

#### `mlx_get_pixel()`
Obtiene el color de un píxel.

```c
uint32_t mlx_get_pixel(mlx_image_t* image, uint32_t x, uint32_t y);
```

**Ejemplo:**
```c
uint32_t color = mlx_get_pixel(img, 10, 10);
```

### Manipulación Directa del Buffer

Para operaciones más eficientes, puedes acceder directamente al array de píxeles:

```c
// Acceso directo al píxel en (x, y)
uint32_t index = (y * img->width + x) * 4;
img->pixels[index + 0] = R; // Rojo
img->pixels[index + 1] = G; // Verde
img->pixels[index + 2] = B; // Azul
img->pixels[index + 3] = A; // Alpha (transparencia)
```

**Ejemplo - Limpiar pantalla:**
```c
void clear_image(mlx_image_t* img, uint32_t color)
{
    uint32_t* pixels = (uint32_t*)img->pixels;
    uint32_t total = img->width * img->height;
    
    for (uint32_t i = 0; i < total; i++)
        pixels[i] = color;
}
```

### Propiedades de Instancias

Cada vez que llamas a `mlx_image_to_window()`, se crea una "instancia" de la imagen.

#### `mlx_set_instance_depth()`
Cambia el orden Z (profundidad) de una instancia.

```c
void mlx_set_instance_depth(mlx_image_t* image, int32_t instance, int32_t depth);
```

**Ejemplo:**
```c
// Mover una instancia al frente
mlx_set_instance_depth(img, 0, 100);
```

#### `mlx_resize_image()`
Redimensiona una imagen (escala).

```c
void mlx_resize_image(mlx_image_t* image, uint32_t new_width, uint32_t new_height);
```

---

## Colores

### Formato de Color

MLX42 usa el formato **RGBA** (Red, Green, Blue, Alpha) en formato hexadecimal:

```c
0xRRGGBBAA
```

- **RR**: Componente rojo (0x00 - 0xFF)
- **GG**: Componente verde (0x00 - 0xFF)
- **BB**: Componente azul (0x00 - 0xFF)
- **AA**: Canal alpha/transparencia (0x00 - 0xFF)

### Crear Colores

#### Método 1: Hexadecimal directo
```c
uint32_t red    = 0xFF0000FF; // Rojo opaco
uint32_t green  = 0x00FF00FF; // Verde opaco
uint32_t blue   = 0x0000FFFF; // Azul opaco
uint32_t black  = 0x000000FF; // Negro opaco
uint32_t white  = 0xFFFFFFFF; // Blanco opaco
uint32_t trans  = 0xFF000080; // Rojo semi-transparente
```

#### Método 2: Función helper
```c
uint32_t create_rgba(uint8_t r, uint8_t g, uint8_t b, uint8_t a)
{
    return (r << 24 | g << 16 | b << 8 | a);
}

// Uso
uint32_t purple = create_rgba(128, 0, 128, 255);
```

#### Método 3: Con la función de MLX
```c
int get_rgba(int r, int g, int b, int a)
{
    return (r << 24 | g << 16 | b << 8 | a);
}
```

### Extraer Componentes

```c
void extract_color(uint32_t color, uint8_t* r, uint8_t* g, uint8_t* b, uint8_t* a)
{
    *r = (color >> 24) & 0xFF;
    *g = (color >> 16) & 0xFF;
    *b = (color >> 8) & 0xFF;
    *a = color & 0xFF;
}
```

### Transparencia (Alpha)

- `0x00` = Totalmente transparente (invisible)
- `0x80` = 50% transparente
- `0xFF` = Totalmente opaco (sin transparencia)

**Ejemplo:**
```c
// Dibujar un cuadrado semi-transparente
uint32_t semi_trans = 0xFF000080; // Rojo 50% transparente

for (int y = 0; y < 100; y++)
{
    for (int x = 0; x < 100; x++)
        mlx_put_pixel(img, x, y, semi_trans);
}
```

---

## Texturas

Las texturas son imágenes cargadas desde archivos (PNG, JPG, etc.).

### Cargar Texturas

#### `mlx_load_png()`
Carga una textura desde un archivo PNG.

```c
mlx_texture_t* mlx_load_png(const char* path);
```

**Retorno:** Textura o `NULL` si falla

**Ejemplo:**
```c
mlx_texture_t* texture = mlx_load_png("./textures/wall.png");
if (!texture)
{
    fprintf(stderr, "Error al cargar textura\n");
    exit(1);
}
```

#### `mlx_delete_texture()`
Libera la memoria de una textura.

```c
void mlx_delete_texture(mlx_texture_t* texture);
```

**Ejemplo:**
```c
mlx_delete_texture(texture);
```

### Convertir Textura a Imagen

Para dibujar una textura, primero debes convertirla a imagen:

```c
mlx_image_t* mlx_texture_to_image(mlx_t* mlx, mlx_texture_t* texture);
```

**Ejemplo completo:**
```c
// 1. Cargar textura
mlx_texture_t* tex = mlx_load_png("sprite.png");

// 2. Convertir a imagen
mlx_image_t* img = mlx_texture_to_image(mlx, tex);

// 3. Dibujar en ventana
mlx_image_to_window(mlx, img, 100, 100);

// 4. Liberar textura (la imagen sigue existiendo)
mlx_delete_texture(tex);
```

### Acceder a Píxeles de Textura

```c
// Obtener color de píxel en textura
uint32_t get_texture_pixel(mlx_texture_t* tex, uint32_t x, uint32_t y)
{
    if (x >= tex->width || y >= tex->height)
        return (0);
    
    uint32_t index = (y * tex->width + x) * tex->bytes_per_pixel;
    uint8_t r = tex->pixels[index + 0];
    uint8_t g = tex->pixels[index + 1];
    uint8_t b = tex->pixels[index + 2];
    uint8_t a = tex->pixels[index + 3];
    
    return (r << 24 | g << 16 | b << 8 | a);
}
```

---

## Hooks y Callbacks

Los hooks permiten ejecutar funciones en respuesta a eventos.

### Loop Hook

#### `mlx_loop_hook()`
Ejecuta una función en cada frame del loop principal.

```c
void mlx_loop_hook(mlx_t* mlx, void (*f)(void*), void* param);
```

**Ejemplo:**
```c
void game_update(void* param)
{
    mlx_t* mlx = (mlx_t*)param;
    
    // Tu código de actualización aquí
    // Se ejecuta cada frame (~60 FPS)
}

int main(void)
{
    mlx_t* mlx = mlx_init(800, 600, "Game", false);
    mlx_loop_hook(mlx, &game_update, mlx);
    mlx_loop(mlx);
    mlx_terminate(mlx);
}
```

### Hooks de Teclado

#### `mlx_key_hook()`
Detecta cuando se presiona o suelta una tecla.

```c
void mlx_key_hook(mlx_t* mlx, mlx_keyfunc func, void* param);

// Prototipo de la función callback
typedef void (*mlx_keyfunc)(mlx_key_data_t keydata, void* param);
```

**Estructura `mlx_key_data_t`:**
```c
typedef struct mlx_key_data
{
    keys_t      key;        // Código de la tecla
    action_t    action;     // Acción (press, release, repeat)
    modifier_t  modifier;   // Modificadores (shift, ctrl, etc.)
} mlx_key_data_t;
```

**Acciones posibles:**
- `MLX_PRESS`: Tecla presionada
- `MLX_RELEASE`: Tecla soltada
- `MLX_REPEAT`: Tecla mantenida

**Ejemplo:**
```c
void key_handler(mlx_key_data_t keydata, void* param)
{
    t_game* game = (t_game*)param;
    
    if (keydata.key == MLX_KEY_ESCAPE && keydata.action == MLX_PRESS)
        mlx_close_window(game->mlx);
    
    if (keydata.key == MLX_KEY_W && keydata.action == MLX_PRESS)
        move_forward(game);
    
    if (keydata.key == MLX_KEY_SPACE && keydata.action == MLX_PRESS)
        jump(game);
}

// Registrar el hook
mlx_key_hook(mlx, &key_handler, &game);
```

### Hooks de Ratón

#### `mlx_mouse_hook()`
Detecta clics del ratón.

```c
void mlx_mouse_hook(mlx_t* mlx, mlx_mousefunc func, void* param);

typedef void (*mlx_mousefunc)(mouse_button_t button, action_t action,
                              modifier_t mods, void* param);
```

**Ejemplo:**
```c
void mouse_handler(mouse_button_t button, action_t action, 
                   modifier_t mods, void* param)
{
    if (button == MLX_MOUSE_BUTTON_LEFT && action == MLX_PRESS)
        printf("Click izquierdo!\n");
    
    if (button == MLX_MOUSE_BUTTON_RIGHT && action == MLX_PRESS)
        printf("Click derecho!\n");
}

mlx_mouse_hook(mlx, &mouse_handler, NULL);
```

#### `mlx_cursor_hook()`
Detecta movimiento del cursor.

```c
void mlx_cursor_hook(mlx_t* mlx, mlx_cursorfunc func, void* param);

typedef void (*mlx_cursorfunc)(double xpos, double ypos, void* param);
```

**Ejemplo:**
```c
void cursor_handler(double xpos, double ypos, void* param)
{
    printf("Cursor en: %.0f, %.0f\n", xpos, ypos);
}

mlx_cursor_hook(mlx, &cursor_handler, NULL);
```

#### `mlx_scroll_hook()`
Detecta scroll del ratón.

```c
void mlx_scroll_hook(mlx_t* mlx, mlx_scrollfunc func, void* param);

typedef void (*mlx_scrollfunc)(double xdelta, double ydelta, void* param);
```

### Otros Hooks

#### `mlx_resize_hook()`
Detecta cuando se redimensiona la ventana.

```c
void mlx_resize_hook(mlx_t* mlx, mlx_resizefunc func, void* param);

typedef void (*mlx_resizefunc)(int32_t width, int32_t height, void* param);
```

#### `mlx_close_hook()`
Detecta cuando se intenta cerrar la ventana.

```c
void mlx_close_hook(mlx_t* mlx, mlx_closefunc func, void* param);

typedef void (*mlx_closefunc)(void* param);
```

---

## Input del Usuario

### Detectar Estado de Teclas

#### `mlx_is_key_down()`
Verifica si una tecla está presionada.

```c
bool mlx_is_key_down(mlx_t* mlx, keys_t key);
```

**Ejemplo - Movimiento continuo:**
```c
void update(void* param)
{
    t_game* game = (t_game*)param;
    
    // Movimiento continuo mientras se mantienen las teclas
    if (mlx_is_key_down(game->mlx, MLX_KEY_W))
        game->player.y -= 5;
    
    if (mlx_is_key_down(game->mlx, MLX_KEY_S))
        game->player.y += 5;
    
    if (mlx_is_key_down(game->mlx, MLX_KEY_A))
        game->player.x -= 5;
    
    if (mlx_is_key_down(game->mlx, MLX_KEY_D))
        game->player.x += 5;
}
```

### Detectar Estado del Ratón

#### `mlx_is_mouse_down()`
Verifica si un botón del ratón está presionado.

```c
bool mlx_is_mouse_down(mlx_t* mlx, mouse_button_t button);
```

**Ejemplo:**
```c
if (mlx_is_mouse_down(mlx, MLX_MOUSE_BUTTON_LEFT))
    printf("Click izquierdo mantenido\n");
```

### Obtener Posición del Cursor

#### `mlx_get_mouse_pos()`
Obtiene la posición actual del cursor.

```c
void mlx_get_mouse_pos(mlx_t* mlx, int32_t* x, int32_t* y);
```

**Ejemplo:**
```c
int32_t mouse_x, mouse_y;
mlx_get_mouse_pos(mlx, &mouse_x, &mouse_y);
printf("Ratón en: %d, %d\n", mouse_x, mouse_y);
```

### Controlar el Cursor

#### `mlx_set_cursor_mode()`
Cambia el modo del cursor.

```c
void mlx_set_cursor_mode(mlx_t* mlx, mouse_mode_t mode);
```

**Modos disponibles:**
- `MLX_MOUSE_NORMAL`: Cursor visible y normal
- `MLX_MOUSE_HIDDEN`: Cursor invisible
- `MLX_MOUSE_DISABLED`: Cursor oculto y bloqueado (FPS)

**Ejemplo - Modo FPS:**
```c
// Ocultar y bloquear cursor para juego FPS
mlx_set_cursor_mode(mlx, MLX_MOUSE_DISABLED);
```

### Códigos de Teclas Comunes

```c
// Letras
MLX_KEY_A, MLX_KEY_B, ..., MLX_KEY_Z

// Números
MLX_KEY_0, MLX_KEY_1, ..., MLX_KEY_9

// Flechas
MLX_KEY_UP
MLX_KEY_DOWN
MLX_KEY_LEFT
MLX_KEY_RIGHT

// Especiales
MLX_KEY_SPACE
MLX_KEY_ESCAPE
MLX_KEY_ENTER
MLX_KEY_TAB
MLX_KEY_BACKSPACE

// Modificadores
MLX_KEY_LEFT_SHIFT
MLX_KEY_LEFT_CONTROL
MLX_KEY_LEFT_ALT
```

---

## Shaders (Avanzado)

MLX42 permite usar shaders personalizados de OpenGL.

### Cargar Shader Personalizado

```c
void mlx_set_shader(mlx_image_t* image, const char* vertex_shader, 
                    const char* fragment_shader);
```

**Ejemplo básico de fragment shader:**
```glsl
#version 330 core

in vec2 TexCoords;
out vec4 FragColor;

uniform sampler2D Texture;
uniform float Time;

void main()
{
    vec4 color = texture(Texture, TexCoords);
    
    // Efecto de onda
    color.rgb *= sin(Time * 2.0) * 0.5 + 0.5;
    
    FragColor = color;
}
```

**Nota:** Los shaders son un tema avanzado. Para proyectos básicos no son necesarios.

---

## XPM42 - Formato de Imágenes

XPM42 es un formato de imagen simple basado en texto, útil para sprites pequeños.

### Estructura de un archivo XPM42

```c
/* XPM */
static char *sprite[] = {
/* width height colors chars_per_pixel */
"8 8 2 1",
/* colors */
". c #000000",  // Negro
"X c #FFFFFF",  // Blanco
/* pixels */
"........",
".XXXXXX.",
".X....X.",
".X.XX.X.",
".X.XX.X.",
".X....X.",
".XXXXXX.",
"........"
};
```

### Cargar XPM42

```c
mlx_texture_t* mlx_load_xpm42(const char* path);
```

**Ejemplo:**
```c
mlx_texture_t* sprite = mlx_load_xpm42("./sprites/player.xpm42");
mlx_image_t* img = mlx_texture_to_image(mlx, sprite);
mlx_image_to_window(mlx, img, 100, 100);
mlx_delete_texture(sprite);
```

---

## Funciones Completas

### Referencia Rápida de Funciones

#### Inicialización
| Función | Descripción |
|---------|-------------|
| `mlx_init()` | Inicializa MLX y crea ventana |
| `mlx_terminate()` | Libera recursos y cierra MLX |
| `mlx_close_window()` | Cierra la ventana |

#### Imágenes
| Función | Descripción |
|---------|-------------|
| `mlx_new_image()` | Crea una nueva imagen |
| `mlx_delete_image()` | Elimina una imagen |
| `mlx_image_to_window()` | Dibuja imagen en ventana |
| `mlx_put_pixel()` | Dibuja un píxel |
| `mlx_get_pixel()` | Obtiene color de un píxel |
| `mlx_resize_image()` | Redimensiona una imagen |

#### Texturas
| Función | Descripción |
|---------|-------------|
| `mlx_load_png()` | Carga textura PNG |
| `mlx_load_xpm42()` | Carga textura XPM42 |
| `mlx_texture_to_image()` | Convierte textura a imagen |
| `mlx_delete_texture()` | Libera textura |

#### Hooks
| Función | Descripción |
|---------|-------------|
| `mlx_loop_hook()` | Hook principal del loop |
| `mlx_key_hook()` | Hook de teclado |
| `mlx_mouse_hook()` | Hook de clicks de ratón |
| `mlx_cursor_hook()` | Hook de movimiento de cursor |
| `mlx_scroll_hook()` | Hook de scroll |
| `mlx_resize_hook()` | Hook de redimensionamiento |
| `mlx_close_hook()` | Hook de cierre de ventana |

#### Input
| Función | Descripción |
|---------|-------------|
| `mlx_is_key_down()` | Verifica si tecla está presionada |
| `mlx_is_mouse_down()` | Verifica si botón está presionado |
| `mlx_get_mouse_pos()` | Obtiene posición del cursor |
| `mlx_set_cursor_mode()` | Cambia modo del cursor |

#### Ventana
| Función | Descripción |
|---------|-------------|
| `mlx_set_window_size()` | Cambia tamaño de ventana |
| `mlx_set_window_pos()` | Cambia posición de ventana |
| `mlx_set_window_limit()` | Establece límites de tamaño |
| `mlx_get_monitor_size()` | Obtiene tamaño del monitor |

#### Loop
| Función | Descripción |
|---------|-------------|
| `mlx_loop()` | Inicia el loop principal |
| `mlx_get_time()` | Obtiene tiempo actual |
| `mlx_get_delta_time()` | Obtiene delta time |

---

## Ejemplos Prácticos

### Ejemplo 1: Ventana Básica con Color

```c
#include <MLX42/MLX42.h>
#include <stdlib.h>

#define WIDTH 800
#define HEIGHT 600

int main(void)
{
    mlx_t* mlx;
    mlx_image_t* img;
    
    // Inicializar
    mlx = mlx_init(WIDTH, HEIGHT, "Mi Primera Ventana", false);
    if (!mlx)
        exit(EXIT_FAILURE);
    
    // Crear imagen
    img = mlx_new_image(mlx, WIDTH, HEIGHT);
    if (!img)
    {
        mlx_terminate(mlx);
        exit(EXIT_FAILURE);
    }
    
    // Llenar de color azul
    for (uint32_t y = 0; y < HEIGHT; y++)
    {
        for (uint32_t x = 0; x < WIDTH; x++)
            mlx_put_pixel(img, x, y, 0x0000FFFF); // Azul
    }
    
    // Mostrar
    mlx_image_to_window(mlx, img, 0, 0);
    mlx_loop(mlx);
    
    // Limpiar
    mlx_terminate(mlx);
    return (EXIT_SUCCESS);
}
```

### Ejemplo 2: Cuadrado que se Mueve

```c
#include <MLX42/MLX42.h>
#include <stdlib.h>

typedef struct s_game
{
    mlx_t*          mlx;
    mlx_image_t*    player;
    int32_t         x;
    int32_t         y;
} t_game;

void update(void* param)
{
    t_game* game = (t_game*)param;
    
    // Movimiento con WASD
    if (mlx_is_key_down(game->mlx, MLX_KEY_W))
        game->y -= 5;
    if (mlx_is_key_down(game->mlx, MLX_KEY_S))
        game->y += 5;
    if (mlx_is_key_down(game->mlx, MLX_KEY_A))
        game->x -= 5;
    if (mlx_is_key_down(game->mlx, MLX_KEY_D))
        game->x += 5;
    
    // ESC para salir
    if (mlx_is_key_down(game->mlx, MLX_KEY_ESCAPE))
        mlx_close_window(game->mlx);
    
    // Actualizar posición
    game->player->instances[0].x = game->x;
    game->player->instances[0].y = game->y;
}

int main(void)
{
    t_game game = {0};
    
    // Inicializar
    game.mlx = mlx_init(800, 600, "Mover Cuadrado", false);
    game.x = 400;
    game.y = 300;
    
    // Crear cuadrado rojo 50x50
    game.player = mlx_new_image(game.mlx, 50, 50);
    for (uint32_t y = 0; y < 50; y++)
        for (uint32_t x = 0; x < 50; x++)
            mlx_put_pixel(game.player, x, y, 0xFF0000FF);
    
    mlx_image_to_window(game.mlx, game.player, game.x, game.y);
    
    // Loop
    mlx_loop_hook(game.mlx, &update, &game);
    mlx_loop(game.mlx);
    
    mlx_terminate(game.mlx);
    return (0);
}
```

### Ejemplo 3: Cargar y Mostrar Textura

```c
#include <MLX42/MLX42.h>
#include <stdlib.h>

int main(void)
{
    mlx_t* mlx;
    mlx_texture_t* texture;
    mlx_image_t* img;
    
    // Inicializar
    mlx = mlx_init(800, 600, "Mostrar Textura", false);
    
    // Cargar textura
    texture = mlx_load_png("./textures/wall.png");
    if (!texture)
    {
        fprintf(stderr, "Error al cargar textura\n");
        mlx_terminate(mlx);
        exit(EXIT_FAILURE);
    }
    
    // Convertir a imagen
    img = mlx_texture_to_image(mlx, texture);
    
    // Mostrar en ventana
    mlx_image_to_window(mlx, img, 100, 100);
    
    // Ya no necesitamos la textura
    mlx_delete_texture(texture);
    
    mlx_loop(mlx);
    mlx_terminate(mlx);
    return (0);
}
```

---

## Tips y Buenas Prácticas

### Performance

1. **Usa acceso directo al buffer** para operaciones intensivas:
```c
uint32_t* pixels = (uint32_t*)img->pixels;
for (uint32_t i = 0; i < img->width * img->height; i++)
    pixels[i] = color;
```

2. **Minimiza llamadas a `mlx_put_pixel()`** en loops grandes.

3. **Reutiliza imágenes** en lugar de crear y destruir constantemente.

### Gestión de Memoria

1. Siempre libera texturas después de convertirlas a imágenes.
2. Llama a `mlx_terminate()` antes de salir del programa.
3. Verifica que las funciones que retornan punteros no devuelvan `NULL`.

### Organización del Código

```c
// Estructura típica para un juego
typedef struct s_game
{
    mlx_t*          mlx;
    mlx_image_t*    img;
    // ... tus datos del juego
} t_game;

void init_game(t_game* game);
void update_game(void* param);
void render_game(t_game* game);
void cleanup_game(t_game* game);
```

### Delta Time

Para movimiento independiente de FPS:

```c
void update(void* param)
{
    t_game* game = (t_game*)param;
    double delta = mlx_get_delta_time(game->mlx);
    
    // Mover a velocidad constante
    game->x += game->velocity * delta;
}
```

---

## Recursos Adicionales

- **Repositorio oficial:** [https://github.com/codam-coding-college/MLX42](https://github.com/codam-coding-college/MLX42)
- **Documentación original:** En la carpeta `docs/` del repositorio
- **Ejemplos:** En la carpeta `examples/` del repositorio

---

## Conclusión

Esta guía cubre todos los aspectos esenciales de MLX42. Con esta información deberías poder:

✅ Crear ventanas y gestionar el ciclo de renderizado  
✅ Dibujar y manipular imágenes  
✅ Cargar y usar texturas  
✅ Gestionar input de teclado y ratón  
✅ Implementar hooks para eventos  
✅ Optimizar tu código gráfico  

---

*Guía creada para proyectos de 42 School*  
*Última actualización: Noviembre 2025*
