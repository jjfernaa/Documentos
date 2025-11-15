# 📚 Guía Completa de MLX42
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

# 📘 MLX42 - Documentación Completa

## Índice
1. [Tipos de datos](#tipos-de-datos)
2. [Funciones de inicialización](#funciones-de-inicialización)
3. [Funciones de ventana](#funciones-de-ventana)
4. [Funciones de input](#funciones-de-input)
5. [Funciones de imagen](#funciones-de-imagen)
6. [Funciones de textura](#funciones-de-textura)
7. [Hooks](#hooks)

---

## Tipos de datos

### `mlx_t`
Estructura principal de MLX42.

**Campos:**
- `void *window` - Ventana GLFW
- `void *context` - Contexto OpenGL
- `int32_t width` - Ancho de la ventana
- `int32_t height` - Alto de la ventana
- `double delta_time` - Tiempo entre frames

**Ejemplo:**
```c
mlx_t *mlx = mlx_init(1920, 1080, "Mi juego", true);
printf("Delta: %f\n", mlx->delta_time);
```

---

### `mlx_image_t`
Buffer de píxeles renderizable.

**Campos:**
- `const uint32_t width` - Ancho (READ-ONLY)
- `const uint32_t height` - Alto (READ-ONLY)
- `uint8_t *pixels` - Array de píxeles RGBA
- `mlx_instance_t *instances` - Array de instancias
- `size_t count` - Número de instancias
- `bool enabled` - Si se renderiza o no

**Formato de píxeles:**
```c
// Cada píxel son 4 bytes: RGBA
uint8_t r = img->pixels[y * img->width * 4 + x * 4 + 0];
uint8_t g = img->pixels[y * img->width * 4 + x * 4 + 1];
uint8_t b = img->pixels[y * img->width * 4 + x * 4 + 2];
uint8_t a = img->pixels[y * img->width * 4 + x * 4 + 3];
```

**Ejemplo:**
```c
mlx_image_t *img = mlx_new_image(mlx, 800, 600);
img->enabled = true;  // Habilitar renderizado
```

---

### `mlx_texture_t`
Textura cargada desde archivo.

**Campos:**
- `uint32_t width` - Ancho
- `uint32_t height` - Alto
- `uint8_t bytes_per_pixel` - Siempre 4 (RGBA)
- `uint8_t *pixels` - Datos de píxeles

**Ejemplo:**
```c
mlx_texture_t *tex = mlx_load_png("wall.png");
printf("Textura: %dx%d\n", tex->width, tex->height);
```

---

### `mlx_key_data_t`
Información sobre una tecla presionada.

**Campos:**
- `keys_t key` - Código de la tecla (MLX_KEY_W, etc.)
- `action_t action` - Acción (PRESS, RELEASE, REPEAT)
- `int32_t os_key` - Código específico del OS
- `modifier_key_t modifier` - Modificadores (SHIFT, CTRL, etc.)

**Ejemplo:**
```c
void key_callback(mlx_key_data_t keydata, void *param)
{
    if (keydata.key == MLX_KEY_SPACE && keydata.action == MLX_PRESS)
        printf("¡Espacio presionado!\n");
}
```

---

## Funciones de inicialización

### `mlx_init()`
Inicializa MLX42.

**Prototipo:**
```c
mlx_t* mlx_init(int32_t width, int32_t height, const char* title, bool resize);
```

**Parámetros:**
- `width` - Ancho de la ventana
- `height` - Alto de la ventana
- `title` - Título de la ventana
- `resize` - Si la ventana es redimensionable

**Retorno:**
- `mlx_t*` - Puntero a la instancia, NULL si falla

**Ejemplo:**
```c
mlx_t *mlx = mlx_init(1920, 1080, "cub3D", false);
if (!mlx)
{
    fprintf(stderr, "Error al inicializar MLX\n");
    exit(1);
}
```

---

### `mlx_terminate()`
Libera todos los recursos de MLX.

**Prototipo:**
```c
void mlx_terminate(mlx_t* mlx);
```

**⚠️ Importante:**
- Llamar al final del programa
- Después de esto, NO usar ninguna función MLX

**Ejemplo:**
```c
mlx_terminate(mlx);  // Libera ventana, imágenes, texturas, etc.
```

---

### `mlx_close_window()`
Solicita cerrar la ventana.

**Prototipo:**
```c
void mlx_close_window(mlx_t* mlx);
```

**Uso:**
```c
if (mlx_is_key_down(mlx, MLX_KEY_ESCAPE))
    mlx_close_window(mlx);
```

---

### `mlx_loop()`
Inicia el bucle principal de renderizado.

**Prototipo:**
```c
void mlx_loop(mlx_t* mlx);
```

**⚠️ Importante:**
- Esta función NO retorna hasta que se cierre la ventana
- Debe ser la ÚLTIMA llamada antes de cleanup

**Ejemplo:**
```c
mlx_loop_hook(mlx, &update_game, &game);
mlx_loop(mlx);  // ← Bloquea hasta cerrar ventana
cleanup_game(&game);
```

---

## Funciones de ventana

### `mlx_set_window_size()`
Cambia el tamaño de la ventana.

**Prototipo:**
```c
void mlx_set_window_size(mlx_t* mlx, int32_t new_width, int32_t new_height);
```

**Ejemplo:**
```c
mlx_set_window_size(mlx, 1920, 1080);
```

---

### `mlx_get_window_pos()`
Obtiene la posición de la ventana.

**Prototipo:**
```c
void mlx_get_window_pos(mlx_t* mlx, int32_t* xpos, int32_t* ypos);
```

**Ejemplo:**
```c
int32_t x, y;
mlx_get_window_pos(mlx, &x, &y);
printf("Ventana en: (%d, %d)\n", x, y);
```

---

### `mlx_set_window_title()`
Cambia el título de la ventana.

**Prototipo:**
```c
void mlx_set_window_title(mlx_t* mlx, const char* title);
```

**Ejemplo:**
```c
char title[100];
sprintf(title, "cub3D - FPS: %.1f", 1.0 / mlx->delta_time);
mlx_set_window_title(mlx, title);
```

---

## Funciones de input

### `mlx_is_key_down()`
Verifica si una tecla está presionada.

**Prototipo:**
```c
bool mlx_is_key_down(mlx_t* mlx, keys_t key);
```

**Retorno:**
- `true` - La tecla ESTÁ presionada
- `false` - La tecla NO está presionada

**Ejemplo:**
```c
if (mlx_is_key_down(mlx, MLX_KEY_W))
    move_forward(game);
if (mlx_is_key_down(mlx, MLX_KEY_S))
    move_backward(game);
```

---

### `mlx_is_mouse_down()`
Verifica si un botón del ratón está presionado.

**Prototipo:**
```c
bool mlx_is_mouse_down(mlx_t* mlx, mouse_key_t key);
```

**Ejemplo:**
```c
if (mlx_is_mouse_down(mlx, MLX_MOUSE_BUTTON_LEFT))
    shoot(game);
```

---

### `mlx_get_mouse_pos()`
Obtiene la posición del ratón.

**Prototipo:**
```c
void mlx_get_mouse_pos(mlx_t* mlx, int32_t* x, int32_t* y);
```

**Ejemplo:**
```c
int32_t mouse_x, mouse_y;
mlx_get_mouse_pos(mlx, &mouse_x, &mouse_y);
printf("Ratón en: (%d, %d)\n", mouse_x, mouse_y);
```

---

### `mlx_set_cursor_mode()`
Cambia el modo del cursor.

**Prototipo:**
```c
void mlx_set_cursor_mode(mlx_t* mlx, mouse_mode_t mode);
```

**Modos:**
- `MLX_MOUSE_NORMAL` - Cursor visible y funcional
- `MLX_MOUSE_HIDDEN` - Cursor oculto pero funcional
- `MLX_MOUSE_DISABLED` - Cursor oculto y deshabilitado

**Ejemplo:**
```c
mlx_set_cursor_mode(mlx, MLX_MOUSE_HIDDEN);  // Para FPS
```

---

## Funciones de imagen

### `mlx_new_image()`
Crea una nueva imagen vacía.

**Prototipo:**
```c
mlx_image_t* mlx_new_image(mlx_t* mlx, uint32_t width, uint32_t height);
```

**Retorno:**
- `mlx_image_t*` - Puntero a la imagen, NULL si falla

**Ejemplo:**
```c
mlx_image_t *img = mlx_new_image(mlx, 1920, 1080);
if (!img)
{
    fprintf(stderr, "Error al crear imagen\n");
    exit(1);
}
```

---

### `mlx_put_pixel()`
Dibuja un píxel en una imagen.

**Prototipo:**
```c
void mlx_put_pixel(mlx_image_t* image, uint32_t x, uint32_t y, uint32_t color);
```

**Formato de color:**
```c
// Color en formato RGBA (32 bits)
uint32_t color = (R << 24) | (G << 16) | (B << 8) | A;

// Ejemplo: Rojo opaco
uint32_t red = (255 << 24) | (0 << 16) | (0 << 8) | 255;
```

**⚠️ Importante:**
- Coordenadas fuera de límites = Undefined Behavior
- Verificar límites ANTES de llamar

**Ejemplo:**
```c
// Dibujar píxel rojo en (100, 100)
uint32_t red = 0xFF0000FF;
mlx_put_pixel(img, 100, 100, red);
```

---

### `mlx_image_to_window()`
Dibuja una imagen en la ventana.

**Prototipo:**
```c
int32_t mlx_image_to_window(mlx_t* mlx, mlx_image_t* img, int32_t x, int32_t y);
```

**Parámetros:**
- `x, y` - Posición en la ventana (0,0 = esquina superior izquierda)

**Retorno:**
- Índice de la instancia, o -1 si falla

**Ejemplo:**
```c
int instance = mlx_image_to_window(mlx, img, 0, 0);
if (instance < 0)
{
    fprintf(stderr, "Error al mostrar imagen\n");
    exit(1);
}
```

---

### `mlx_delete_image()`
Elimina una imagen y todas sus instancias.

**Prototipo:**
```c
void mlx_delete_image(mlx_t* mlx, mlx_image_t* image);
```

**⚠️ Importante:**
- Libera memoria y elimina de la cola de renderizado
- Después de esto, el puntero es inválido

**Ejemplo:**
```c
mlx_delete_image(mlx, img);
img = NULL;  // Buena práctica
```

---

## Funciones de textura

### `mlx_load_png()`
Carga una textura PNG desde archivo.

**Prototipo:**
```c
mlx_texture_t* mlx_load_png(const char* path);
```

**Retorno:**
- `mlx_texture_t*` - Textura cargada, NULL si falla

**Ejemplo:**
```c
mlx_texture_t *wall = mlx_load_png("./textures/wall.png");
if (!wall)
{
    fprintf(stderr, "Error: Textura no encontrada\n");
    exit(1);
}
```

---

### `mlx_texture_to_image()`
Convierte una textura en imagen renderizable.

**Prototipo:**
```c
mlx_image_t* mlx_texture_to_image(mlx_t* mlx, mlx_texture_t* texture);
```

**Ejemplo:**
```c
mlx_texture_t *tex = mlx_load_png("wall.png");
mlx_image_t *img = mlx_texture_to_image(mlx, tex);
mlx_image_to_window(mlx, img, 0, 0);
mlx_delete_texture(tex);  // Ya no se necesita
```

---

### `mlx_delete_texture()`
Libera una textura de memoria.

**Prototipo:**
```c
void mlx_delete_texture(mlx_texture_t* texture);
```

**Ejemplo:**
```c
mlx_texture_t *tex = mlx_load_png("wall.png");
// ... usar textura ...
mlx_delete_texture(tex);
tex = NULL;
```

---

## Hooks

### `mlx_loop_hook()`
Registra una función para ejecutar cada frame.

**Prototipo:**
```c
bool mlx_loop_hook(mlx_t* mlx, void (*f)(void*), void* param);
```

**Parámetros:**
- `f` - Función callback
- `param` - Parámetro a pasar (típicamente `t_game*`)

**Retorno:**
- `true` - Hook registrado
- `false` - Error

**Ejemplo:**
```c
void update_game(void *param)
{
    t_game *game = (t_game *)param;
    handle_input(game);
    render_frame(game);
}

mlx_loop_hook(mlx, &update_game, &game);
```

---

### `mlx_key_hook()`
Registra callback para eventos de teclado.

**Prototipo:**
```c
void mlx_key_hook(mlx_t* mlx, mlx_keyfunc func, void* param);
```

**Diferencia con `mlx_is_key_down()`:**
- `mlx_key_hook()` - Se llama UNA VEZ al presionar/soltar
- `mlx_is_key_down()` - Verifica estado CONSTANTEMENTE

**Ejemplo:**
```c
void key_callback(mlx_key_data_t keydata, void *param)
{
    if (keydata.key == MLX_KEY_SPACE && keydata.action == MLX_PRESS)
        printf("¡Espacio presionado!\n");
}

mlx_key_hook(mlx, &key_callback, &game);
```

---

### `mlx_mouse_hook()`
Registra callback para clicks del ratón.

**Prototipo:**
```c
void mlx_mouse_hook(mlx_t* mlx, mlx_mousefunc func, void* param);
```

**Ejemplo:**
```c
void mouse_callback(mouse_key_t button, action_t action, modifier_key_t mods, void *param)
{
    if (button == MLX_MOUSE_BUTTON_LEFT && action == MLX_PRESS)
        printf("Click izquierdo\n");
}

mlx_mouse_hook(mlx, &mouse_callback, &game);
```

---

### `mlx_cursor_hook()`
Registra callback para movimiento del ratón.

**Prototipo:**
```c
void mlx_cursor_hook(mlx_t* mlx, mlx_cursorfunc func, void* param);
```

**Ejemplo:**
```c
void cursor_callback(double xpos, double ypos, void *param)
{
    printf("Ratón en: (%.1f, %.1f)\n", xpos, ypos);
}

mlx_cursor_hook(mlx, &cursor_callback, &game);
```

---

### `mlx_close_hook()`
Registra callback para cierre de ventana.

**Prototipo:**
```c
void mlx_close_hook(mlx_t* mlx, mlx_closefunc func, void* param);
```

**Ejemplo:**
```c
void close_callback(void *param)
{
    printf("Cerrando ventana...\n");
    mlx_close_window((mlx_t *)param);
}

mlx_close_hook(mlx, &close_callback, mlx);
```

---

## Códigos de teclas

### Letras
```c
MLX_KEY_A ... MLX_KEY_Z  // A-Z
```

### Números
```c
MLX_KEY_0 ... MLX_KEY_9  // 0-9
```

### Flechas
```c
MLX_KEY_UP
MLX_KEY_DOWN
MLX_KEY_LEFT
MLX_KEY_RIGHT
```

### Especiales
```c
MLX_KEY_SPACE
MLX_KEY_ESCAPE
MLX_KEY_ENTER
MLX_KEY_TAB
MLX_KEY_BACKSPACE
```

### Modificadores
```c
MLX_KEY_LEFT_SHIFT
MLX_KEY_LEFT_CONTROL
MLX_KEY_LEFT_ALT
```

---

## Códigos de error

```c
MLX_SUCCESS      // Sin errores
MLX_INVFILE      // Archivo no existe
MLX_INVPNG       // PNG inválido
MLX_MEMFAIL      // Fallo de memoria
MLX_GLFWFAIL     // GLFW falló
MLX_WINFAIL      // No se pudo crear ventana
```

**Uso:**
```c
mlx_texture_t *tex = mlx_load_png("wall.png");
if (!tex)
{
    fprintf(stderr, "Error: %s\n", mlx_strerror(mlx_errno));
    exit(1);
}
```

---

## Tips y Buenas Prácticas

### Performance

1. **Usa acceso directo al buffer** para operaciones intensivas:
```c
uint32_t	*pixels;
uint32_t	i;
uint32_t	total_pixels;

pixels = (uint32_t *)img->pixels;
total_pixels = img->width * img->height;
i = 0;
while (i < total_pixels)
{
    pixels[i] = color;
    i++;
}
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
---

*Guía creada para proyectos de 42 School*  
*Última actualización: Noviembre 2025*
