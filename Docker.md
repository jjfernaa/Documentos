# Docker Commands Cheat Sheet - 42 Projects

## 🐳 **Comandos Esenciales**

### **Ver Estado**
```bash
docker ps           # Containers corriendo
docker ps -a        # Todos los containers (incluso parados)
docker images       # Imágenes disponibles
docker system df    # Uso de espacio en disco
docker stats        # Uso de recursos en tiempo real
```

### **Crear y Gestionar Imágenes**
```bash
docker build -t nombre-imagen .              # Crear imagen desde Dockerfile
docker build -t nombre-imagen --no-cache .   # Build sin usar cache
docker rmi nombre-imagen                     # Eliminar imagen
docker image prune                           # Eliminar imágenes no usadas
```

### **Crear y Usar Containers**
```bash
# Container temporal (se elimina al salir):
docker run -it --rm nombre-imagen

# Container con volume mounting (código actualizado):
docker run -it --rm -v $(pwd):/workspace nombre-imagen

# Container persistente:
docker run -it --name mi-container nombre-imagen

# Container con límites de recursos:
docker run -it --memory="512m" --cpus="1.0" nombre-imagen
```

### **Gestionar Containers**
```bash
docker start mi-container           # Iniciar container parado
docker start -ai mi-container       # Iniciar y entrar directamente
docker stop mi-container            # Parar container
docker restart mi-container         # Reiniciar container
docker exec -it mi-container bash   # Entrar a container corriendo
docker rm mi-container              # Eliminar container
docker rm -f mi-container           # Forzar eliminación
```

### **Limpieza y Mantenimiento**
```bash
docker container prune              # Eliminar containers parados
docker image prune                  # Eliminar imágenes no usadas
docker volume prune                 # Eliminar volumes no usados
docker system prune                 # Limpieza general
docker system prune -a              # Limpieza completa (¡CUIDADO!)
```

## 🚀 **Workflows para 42**

### **Setup Inicial (UNA VEZ por proyecto)**
```bash
# En directorio del proyecto:
docker build -t proyecto-env .
```

### **Desarrollo Diario**
```bash
# 1. Editar código en Mac (VS Code, vim, etc.)
vim srcs/main.c

# 2. Testing en Linux:
docker run -it --rm -v $(pwd):/workspace proyecto-env
cd /workspace && make clean && make

# 3. Testing con valgrind:
valgrind --leak-check=full ./programa

# 4. Salir:
exit  # Container se elimina automáticamente
```

### **Container Persistente (para desarrollo largo)**
```bash
# Crear container de trabajo:
docker run -it --name dev-container -v $(pwd):/workspace proyecto-env

# Instalar herramientas extra (se guardan):
apt install gdb

# Salir y volver después:
exit
docker start -ai dev-container
```

## 🔧 **Troubleshooting**

### **Problemas Comunes**
```bash
# Error: "Cannot connect to Docker daemon":
# → Abrir Docker Desktop

# Error: "Container name already exists":
docker rm nombre-container

# Error: "No space left on device":
docker system prune -a

# Ver logs de container:
docker logs nombre-container

# Inspeccionar container:
docker inspect nombre-container
```

### **Volume Mounting**
```bash
# Sintaxis: -v [ruta-local]:[ruta-container]
docker run -it -v $(pwd):/workspace imagen    # Directorio actual
docker run -it -v /Users/user/proyecto:/app imagen  # Ruta específica

# Solo lectura:
docker run -it -v $(pwd):/workspace:ro imagen
```

## 📁 **Estructura Recomendada**

```
proyecto/
├── Dockerfile          # Configuración del entorno
├── docker-commands.md  # Este archivo
├── src/                # Código fuente
└── Makefile           # Build instructions
```

## 🎯 **Tips y Mejores Prácticas**

- ✅ **Usar `--rm`** para containers temporales
- ✅ **Volume mounting** para desarrollo activo
- ✅ **Nombres descriptivos** para containers persistentes
- ✅ **Limpiar regularmente** con `docker container prune`
- ❌ **NO usar** `docker system prune -a` sin pensar
- ❌ **NO crear containers** sin `--rm` a menos que los necesites después

## 🚀 **Comandos por Frecuencia de Uso**

### **Diarios**
```bash
docker run -it --rm -v $(pwd):/workspace proyecto-env
docker ps
exit
```

### **Semanales**
```bash
docker container prune
docker image prune
```

### **Ocasionales**
```bash
docker build -t proyecto-env .
docker system df
```
