## Automatización de Despliegue Web con Contenedores

> **DOCUMENTACIÓN TÉCNICA**
> 
> **Autor:** Aitor Arcas Romero
> **Repositorio:** https://github.com/AitorArcas890/WebFusionProyect

---

## 1. Introducción

Este documento describe la arquitectura, implementación y uso de un sistema de despliegue automatizado desarrollado para la empresa *WebFusion Digital S.L.* La solución permitirá a cualquier desarrollador, independientemente de su nivel, levantar un entorno WordPress completamente funcional ejecutando un único comando. 

El sistema combina tres tecnologías:

- **GitHub** — control de versiones y almacenamiento del código fuente
- **Vagrant** — creación y gestión de la máquina virtual reproducible
- **Docker Compose** — orquestación de contenedores con WordPress y sincronización Git
---

## 2. Arquitectura del Sistema

El sistema sigue una arquitectura en capas como esta:

```HTML, CSS, JavaSCript 
┌─────────────────────────────────────────┐
│         Ordenador Local (Windows)        │
│                                          │
│   ┌──────────────────────────────────┐   │
│   │     Vagrant + VirtualBox          │   │
│   │                                  │   │
│   │   ┌────────────────────────────┐ │   │
│   │   │    Ubuntu 22.04 LTS (VM)   │ │   │
│   │   │                            │ │   │
│   │   │  ┌──────────┐ ┌─────────┐ │ │   │
│   │   │  │WordPress │ │git-sync │ │ │   │
│   │   │  │:8080     │ │         │ │ │   │
│   │   │  └────┬─────┘ └────┬────┘ │ │   │
│   │   │       │   wp_data  │      │ │   │
│   │   └───────┴────────────┴──────┘ │   │
│   └──────────────────────────────────┘   │
└─────────────────────────────────────┬───┘
                                      │
                           ┌──────────▼──────────┐
                           │  GitHub Repository   │
                           │  /wordpress-theme/   │
                           └─────────────────────┘
```

### 2.2 Componentes Necesarios

#### Vagrant + VirtualBox
Vagrant gestiona el ciclo de vida completo de la maquina virtual. El fichero `Vagrantfile` define todos los parámetros: imagen base (`ubuntu/jammy64`), recursos asignados (2 GB RAM, 2 CPUs), reenvío de puertos y el script de aprovisionamiento.

#### Contenedor WordPress
Utiliza la imagen oficial `wordpress:latest`. Expone el puerto 80 internamente, que Vagrant reenvía al puerto 8080 del host. Almacena todos sus datos en el volumen Docker `wp_data`.

#### Contenedor git-sync
Basado en `alpine/git`, una imagen ultra-ligera de solo 30 MB. Su única función es clonar el repositorio de GitHub y copiar los archivos PHP al directorio de temas de WordPress dentro del volumen compartido. Se ejecuta una vez y se elimina automáticamente (`--rm`).

#### Volumen compartido wp_data
Es el nexo entre los dos contenedores. WordPress lee los archivos de temas desde este volumen, y git-sync escribe en él. Esto permite que los cambios del repositorio se reflejen en WordPress sin reiniciar ningún servicio.

---

## 3. Requisitos Previos

Tras instalar las herramientas es obligatorio **reiniciar el PC**. Para verificar la instalación, abrir Git Bash y ejecutar:

| Herramienta      | Versión mínima |
| ---------------- | -------------- |
| VirtualBox       | 6.1 o superior |
| Vagrant          | 2.3 o superior |
| Git para Windows | 2.x o superior |

```Bash
vagrant --version     # debe mostrar: Vagrant 2.x.x
git --version         # debe mostrar: git version 2.x.x
```

---

## 4. Estructura del repo de Github

### 4.1 Repositorio de código (WebFusionProyect)

```HTML, CSS, JavaSCript 
WebFusionProyect/
├── README.md
├── webfusion-structure
└── wordpress-theme/
    ├── index.php        # Página principal en php
    └── style.css        # Cabecera del tema de WordPress
```

### 4.2 Repositorio de infraestructura (webfusion-structure)

```HTML
webfusion-structure/
├── Vagrantfile          # Configuración de la maquina y su aprovisionamiento
└── docker-compose.yml   # Definición de los servicios Docker
```

---

## 5. Instrucciones de Despliegue

### Paso 1 — Clonar el repositorio de infraestructura
Desde la terminal llegaremos a la carpeta donde queramos tener la estructura guardada y ejecutaremos los siguientes comandos.

```bash
git clone https://github.com/AitorArcas890/webfusion-structure.git # Clonamos el Repo
cd webfusion-structure # Entramos dentro del Repo
```

### Paso 2 — Levantar la VM

```bash
vagrant up
```

Este comando realiza automáticamente y en el siguiente orden:

1. Descarga la imagen base de Ubuntu (solo si no esta instalada)
2. Crea y configura la máquina en VirtualBox
3. Ejecuta el script de aprovisionamiento => actualiza el sistema e instala Docker
4. Lanza los contenedores `db`, `wordpress` y `git-sync`
5. `git-sync` clona el repositorio de GitHub y copia los archivos PHP

### Paso 3 — Configurar WordPress

Abrir el navegador en http://localhost:8080.
> Es posible un error llamado "Error establishing a database connection", si es así, ir al aparatado [[#Error 'Error establishing a database connection']]

El asistente de configuración de wordpress solicitara:
- Idioma: **Español**
- Título del sitio:
- Usuario y contraseña de administrador
- Correo electrónico del administrador

### Paso 4 — Activar el tema

Desde el panel de administración (`http://localhost:8080/wp-admin`):

1. Ir a **Apariencia → Temas**
2. Localizar el tema **WebFusion** en la lista
3. Pulsar **Activar**
4. Acceder a `http://localhost:8080`. Debe mostrarse la página corporativa de WebFusion Digital.

> [!info] Tiempo estimado
> El proceso completo tarda aproximadamente **10-15 minutos** la primera vez. En ejecuciones el tiempo se reducirá significativamente.

---

## 7. Actualización Automática del Contenido

### 7.1 Flujo de actualización

```bash
# 1. Editar la pagina Web FusionProyect
cd WebFusionProyect
notepad wordpress-theme/index.php # Esto es para hacerlo en terminal, se podria abrir el archivo con cualquier IDE 

# 2. Subir los cambios a GitHub
git add .
git commit -m "descripcion del cambio"
git push origin main

# 3. Aplicar los cambios en el entorno
cd ../webfusion-structure
vagrant provision
```

### 7.2 ¿Qué Hace vagrant provision?

Al ejecutar el comando `vagrant provision`, el script de aprovisionamiento volverá a iniciarse siguiendo los siguientes pasos:

- Detiene el contenedor `git-sync` si estuviera corriendo
- Vuelve a ejecutar `git-sync`, que hace `git pull` del repositorio
- Copia los archivos PHP actualizados al volumen `wp_data`
- WordPress sirve inmediatamente el contenido nuevo sin reiniciarse

> [!example] Ejemplo
> Si un desarrollador cambia el texto de la cabecera en `index.php`, hace `git push` y ejecuta `vagrant provision`, el cambio se refleja en `http://localhost:8080` sin intervención manual.

---

## 8. Referencia de Comandos

| Comando             | Descripción                                           |
| ------------------- | ----------------------------------------------------- |
| `vagrant up`        | Crea la VM y ejecuta el aprovisionamiento completo    |
| `vagrant provision` | Re-ejecuta el aprovisionamiento y actualiza el sitio  |
| `vagrant ssh`       | Entras dentro de la Maquina Virutal                   |
| `vagrant halt`      | Apaga la VM de forma limpia                           |
| `vagrant destroy`   | Elimina completamente la Maquina Virtual              |
| `vagrant status`    | Muestra el estado actual de la Maquina Virtual        |

---

## 9. Problemas Comunes

### Error: 'Error establishing a database connection'

**Causa:** MySQL no terminó de iniciarse antes de que WordPress.
**Solución:**

```bash
vagrant ssh
cd /vagrant
sudo docker compose restart db
sudo docker compose restart wordpress
```

### El tema WebFusion no aparece en WordPress

**Causa:** WordPress requiere un archivo `style.css` con cabecera especial para reconocer un tema.
**Solución:** Verificar que el tema está en el directorio correcto:

```bash
vagrant ssh
sudo ls /var/lib/docker/volumes/vagrant_wp_data/_data/wp-content/themes/
```

### Vagrant up falla con error de red

**Causa:** El puerto 8080 puede estar en uso por otra aplicación.
**Solución:** Cambiar el puerto en el `Vagrantfile` (`host: 8081`) o liberar el puerto 8080.

### Vagrant se queda intentando crear la MV
**Causa:** Saturation del sistema y falla time limit.  
**Solución:** Apagar la maquina, eliminarla y volver a levantar el servicio.

---

## 10. Conclusiones

La solución cumple con los requisitos técnicos del proyecto y resuelve los problemas mencionados:

- **Errores de versiones:** eliminados gracias al control de versiones con Git
- **Desincronización entre desarrolladores:** resuelta con un repositorio centralizado en GitHub
- **Pérdida de cambios:** imposible gracias al historial de commits
- **Procesos manuales:** automatizados completamente con `vagrant up` y `vagrant provision`

La arquitectura es completamente reproducible: cualquier desarrollador con los requisitos instalados puede clonar el repositorio y tener un entorno WordPress idéntico al del resto del equipo en cuestión de minutos, sin conocimientos avanzados de sistemas.

---

*WebFusion Digital S.L. — Documentación Técnica — Aitor Arcas*
