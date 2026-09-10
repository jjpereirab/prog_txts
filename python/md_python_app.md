### 1. La estructura de carpetas (El patrón `src` o `app`)
Todo tu código debe vivir dentro de una carpeta principal (por ejemplo, `app` o `src`), y el `.env` queda afuera, en la raíz del proyecto.

```text
mi_proyecto/
├── .env                  <-- Variables de entorno
├── requirements.txt
└── app/                  <-- Tu paquete principal
    ├── __init__.py       <-- Hace que Python trate a 'app' como un módulo
    ├── config.py         <-- Tu configuración (con pydantic-settings)
    └── modulo_hijo/
        ├── __init__.py
        └── script_hijo.py <-- ¿Cómo importamos config.py aquí?
```

### 2. La importación absoluta en el script hijo
Dentro de `script_hijo.py`, sin importar en qué nivel de profundidad esté, siempre vas a importar desde la raíz del paquete principal (`app`):

```python
# app/modulo_hijo/script_hijo.py

# Importación absoluta (La forma correcta)
from app.config import settings

def hacer_algo():
    print(f"Conectando a {settings.DATABASE_URL} en el puerto {settings.PORT}")
```

### 3. El truco final: ¿Cómo sabe Python dónde está `app`?
Si intentas ejecutar el script entrando a la carpeta (`cd app/modulo_hijo` y luego `python script_hijo.py`), Python fallará porque no sabe qué es `app`. 

Para que Python reconozca tu paquete principal desde cualquier lugar, se usan estas tres estrategias profesionales:

#### Opción A: Ejecutar como módulo (La forma estándar)
Nunca ejecutes los archivos internos directamente. Ubícate en la raíz del proyecto (`mi_proyecto/`) y ejecuta tu script usando el flag `-m` (módulo):

```bash
# Desde la carpeta raíz 'mi_proyecto/'
python -m app.modulo_hijo.script_hijo
```
Al usar `-m`, Python agrega automáticamente la carpeta raíz (`mi_proyecto/`) a su ruta de búsqueda (`sys.path`), por lo que encuentra `app` sin problemas.

#### Opción B: Instalación en modo "Editable" (La forma más pro)
Para proyectos grandes, conviertes tu propio código en una dependencia de tu entorno virtual.
Creas un archivo mínimo llamado `pyproject.toml` o `setup.py` en la raíz y ejecutas:

```bash
pip install -e .
```
Esto le dice al `venv`: *"Instala este proyecto localmente, pero en modo de edición"*. A partir de ese momento, Python conoce la existencia de `app` **estés en la carpeta que estés**. Puedes ejecutar cualquier script de tu proyecto sin preocuparte por las rutas.

#### Opción C: Usar PYTHONPATH (Común en Docker)
Si usas contenedores, simplemente le dices al sistema operativo dónde empieza tu código inyectando la variable `PYTHONPATH`.

```bash
# En la terminal o en tu Dockerfile
export PYTHONPATH="."
python app/modulo_hijo/script_hijo.py
```

### Resumen
1. **Nunca** uses `sys.path.append(os.path.dirname(...))` en tus scripts.
2. Agrupa tu código en una carpeta raíz (ej. `app/`).
3. Usa siempre **importaciones absolutas** (`from app.config import settings`).
4. Ejecuta tus scripts desde la raíz con `python -m` o instala tu app en el `venv` con `pip install -e .`.


---
# "MVP" de estructura

Aquí tienes el MVP (Producto Mínimo Viable) exacto para implementar esta arquitectura.

Primero, veamos cómo quedará la **estructura exacta de carpetas** para que sepas dónde va cada archivo. El archivo `pyproject.toml` (el estándar moderno que reemplaza a `setup.py`) va **en la raíz del proyecto**, al mismo nivel que tu `.env`.

### 1. Estructura del proyecto
```text
mi_proyecto/
├── .env
├── pyproject.toml         <-- Aquí va (en la raíz)
└── app/
    ├── __init__.py
    ├── config.py
    └── modulo_hijo/
        ├── __init__.py
        └── script_hijo.py
```

---

### 2. Contenido de los archivos (MVP)

#### `mi_proyecto/.env`
Crea este archivo en la raíz con un par de variables de prueba.
```env
APP_NAME="Hola Mundo App"
SECRET_KEY="super-secreto-123"
```

#### `mi_proyecto/pyproject.toml`
Este archivo convierte tu carpeta `app` en un paquete instalable. Va en la raíz del proyecto.
```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "app"
version = "0.1.0"
description = "MVP de Hola Mundo"
dependencies = [
    "pydantic-settings>=2.0.0",
]
```

#### `mi_proyecto/app/__init__.py` y `mi_proyecto/app/modulo_hijo/__init__.py`
**Déjalos completamente vacíos.**
*(Nota: En Python, un archivo `__init__.py` vacío es simplemente una "bandera" que le dice al sistema: "trata esta carpeta como un paquete del cual puedo importar cosas").*

#### `mi_proyecto/app/config.py`
Aquí cargamos el `.env` dinámicamente buscando la raíz del proyecto.
```python
from pathlib import Path
from pydantic_settings import BaseSettings, SettingsConfigDict

# Esto sube un nivel: desde /app/config.py -> hasta la carpeta /mi_proyecto/
BASE_DIR = Path(__file__).resolve().parent.parent

class Settings(BaseSettings):
    # Definimos qué variables esperamos y su tipo de dato
    APP_NAME: str = "App por Defecto"  # Tiene valor por defecto
    SECRET_KEY: str                    # Es obligatoria (no tiene valor por defecto)

    # Configuración de Pydantic para leer el archivo
    model_config = SettingsConfigDict(
        env_file=BASE_DIR / ".env",
        env_file_encoding="utf-8",
        extra="ignore"
    )

# Instanciamos las configuraciones para usarlas en el resto de la app
settings = Settings()
```

#### `mi_proyecto/app/modulo_hijo/script_hijo.py`
El script que consume las variables desde cualquier profundidad usando importación absoluta.
```python
# Importación absoluta desde la raíz de nuestro paquete 'app'
from app.config import settings

def main():
    print("--- INICIANDO SCRIPT ---")
    print(f"Nombre de la App: {settings.APP_NAME}")
    print(f"Clave secreta cargada: {settings.SECRET_KEY}")
    print("------------------------")

if __name__ == "__main__":
    main()
```

---

### 3. Cómo proceder con la Opción B (`pip install -e .`)

Ahora que tienes los archivos en su lugar, abre tu terminal y sigue estos pasos:

1. **Ubícate en la raíz del proyecto** (donde está el `pyproject.toml`):
   ```bash
   cd ruta/a/tu/mi_proyecto
   ```

2. **Crea y activa tu entorno virtual** (si no lo has hecho ya):
   ```bash
   # En Windows:
   python -m venv venv
   venv\Scripts\activate
   
   # En Mac/Linux:
   python3 -m venv venv
   source venv/bin/activate
   ```

3. **Ejecuta la instalación en modo editable (La Opción B):**
   *(El punto `.` al final es súper importante, significa "instala el directorio actual")*
   ```bash
   pip install -e .
   ```
   *¿Qué hace esto? Instala las dependencias que pusimos en el `.toml` (como `pydantic-settings`) y "registra" tu carpeta `app` en el entorno virtual de Python.*

4. **Prueba tu script desde CUALQUIER lugar:**
   Ahora tu script puede encontrar `app.config` sin importar dónde estés parado en la terminal.
   ```bash
   python app/modulo_hijo/script_hijo.py
   ```

**¡Y listo!** Verás en pantalla los valores sacados directamente de tu `.env`. Este es el estándar de la industria para estructurar aplicaciones modernas en Python (FastAPI, por ejemplo, usa exactamente este mismo patrón por defecto).