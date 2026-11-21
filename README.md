# OpenCode + Ghidra + REA en Fedora Silverblue

> Guía para darle a OpenCode (Flatpak) acceso real a ingeniería inversa: desensamblar,
> descompilar, leer strings, seguir referencias y renombrar funciones, usando **REA** como
> servidor MCP y **Ghidra** instalado desde Flatpak.
>
> La guía tiene dos partes:
>
> - **Parte 1:** explicación detallada, paso a paso, con verificación y diagnóstico.
> - **Parte 2:** los mismos pasos en comandos directos, al final, para copiar y pegar.

**Entorno de referencia:** Fedora Silverblue con GNOME, Ghidra `12.1.3` (Flatpak
`org.ghidra_sre.Ghidra`), OpenCode `2.1.3` (Flatpak `ai.opencode.opencode`), `rea-agents 6.7.0`,
`mcp-proxy 0.15.0`, `uv 0.13.7` y Java 25 en el toolbox.

---

## Índice

- [Parte 1 — Guía detallada](#parte-1--guía-detallada)
  - [1. Qué vas a montar](#1-qué-vas-a-montar)
  - [2. Quién es quién](#2-quién-es-quién)
  - [3. Las tres capas: host, toolbox y sandbox](#3-las-tres-capas-host-toolbox-y-sandbox)
  - [4. Requisitos previos](#4-requisitos-previos)
  - [5. Paso 1 — Instalar Ghidra (Flatpak)](#5-paso-1--instalar-ghidra-flatpak)
  - [6. Paso 2 — Dependencias en el toolbox](#6-paso-2--dependencias-en-el-toolbox)
  - [7. Paso 3 — Arrancar mcp-proxy](#7-paso-3--arrancar-mcp-proxy)
  - [8. Paso 4 — Configurar OpenCode](#8-paso-4--configurar-opencode)
  - [9. Verificación](#9-verificación)
  - [10. Uso diario](#10-uso-diario)
  - [11. Problemas frecuentes](#11-problemas-frecuentes)
  - [12. Seguridad](#12-seguridad)
  - [13. Mantenimiento](#13-mantenimiento)
  - [14. Notas de precisión](#14-notas-de-precisión)
- [Parte 2 — Comandos directos (TL;DR)](#parte-2--comandos-directos-tldr)

---

## Parte 1 — Guía detallada

### 1. Qué vas a montar

OpenCode corre en un sandbox Flatpak y **no** necesita permisos especiales sobre el host. Solo
abre una conexión HTTP al loopback. Del otro lado, `mcp-proxy` traduce ese HTTP a un servidor MCP
por stdio (`rea mcp`), y REA lanza Ghidra en modo headless para analizar el binario.

```text
OpenCode (Flatpak, sandbox)
   │  HTTP POST → http://127.0.0.1:8082/mcp
   ▼
mcp-proxy (toolbox; comparte la red del host)
   │  stdio
   ▼
rea mcp (toolbox; Node.js)
   │  GHIDRA_INSTALL_DIR=/run/host/var/lib/flatpak/.../lib/ghidra
   ▼
Ghidra 12.1.3 headless (archivos del Flatpak; Java del toolbox)
```

**La idea central:** el proxy y REA viven en el toolbox, donde están Node, Java y uv. OpenCode
solo habla HTTP contra `127.0.0.1:8082`. Ghidra no necesita estar abierto: REA lo ejecuta en
modo headless y le pasa la ruta de instalación del Flatpak.

> **Corrección importante:** `mcp-proxy` **no** debe ejecutarse en el host. Los contenedores de
> toolbox comparten la pila de red del host, así que un proceso que escucha en
> `127.0.0.1:8082` dentro del toolbox es alcanzable desde el host y desde el Flatpak de
> OpenCode. Además, el entorno virtual que crea `uv tool install` pertenece al Python del
> toolbox; ejecutar ese mismo binario desde Silverblue suele fallar con
> `ModuleNotFoundError: No module named 'mcp_proxy'` o con un intérprete inexistente. La forma
> correcta es `toolbox run mcp-proxy ... -- rea mcp`, con **todo** dentro del toolbox.

---

### 2. Quién es quién

| Pieza | Qué es | Dónde vive | Para qué sirve |
| --- | --- | --- | --- |
| **Ghidra** | Suite de ingeniería inversa de la NSA | Flatpak, en el host | Desensamblar, descompilar, analizar |
| **Ghidra headless** | Ghidra sin interfaz, lanzado por REA | Archivos del Flatpak, ejecutado desde el toolbox | Análisis automático de un binario |
| **OpenCode** | El agente | Flatpak, en el host | Consume el servidor MCP |
| **toolbox** | Contenedor Fedora con tu `$HOME` y la red del host compartidos | Host | Aquí van Node, Java, uv, REA y el proxy |
| **REA** (`rea-agents`) | CLI + servidor MCP en Node | Toolbox, vía `npm -g` | Habla con Ghidra y expone las herramientas MCP |
| **mcp-proxy** | Puente stdio ⇄ HTTP para MCP | Toolbox, vía `uv tool` | Expone `rea mcp` en `127.0.0.1:8082` |
| **uv** | Gestor de paquetes Python | Toolbox | Instala `mcp-proxy` en un entorno aislado |
| **Java 25** | JDK | Toolbox | Lo usa Ghidra headless al ser lanzado por REA |

REA es el único servidor MCP de esta guía. No hace falta la extensión GhidraMCP ni un bridge
aparte.

---

### 3. Las tres capas: host, toolbox y sandbox

Este es el punto donde más gente se atasca. Hay tres “mundos” y los comandos solo existen en
algunos:

| Comando | Host | Toolbox | Sandbox de OpenCode |
| --- | --- | --- | --- |
| `flatpak`, `flatpak kill` | ✅ | ❌ | ❌ |
| `toolbox enter/run`, `ss`, `curl` | ✅ | ❌ | ❌ |
| `dnf`, `java`, `node`, `npm`, `uv`, `rea`, `mcp-proxy` | ❌ | ✅ | ❌ |
| Editar `opencode.json` | ✅ | ✅ (es tu `$HOME`) | ✅ (es tu `$HOME`) |

Reglas prácticas:

- `sudo dnf …` va **dentro del toolbox**.
- `flatpak install`, `flatpak kill` y `toolbox run` van **en el host**.
- Si dentro de OpenCode ves `flatpak: orden no encontrada` o `toolbox: orden no encontrada`, no
  estás en el host: estás en el sandbox del Flatpak. Esta guía no necesita ejecutar comandos del
  host desde OpenCode.
- **`$HOME` se comparte:** un binario instalado en `~/.npm-global/bin` o `~/.local/bin` dentro
  del toolbox también existe en el host, y viceversa.
- **La red se comparte:** toolbox usa la red del host. `127.0.0.1:8082` significa lo mismo en
  el host, en el toolbox y en el sandbox de OpenCode (que tiene `--share=network`).

---

### 4. Requisitos previos

- Fedora Silverblue (o cualquier sistema inmutable con `rpm-ostree`, Flatpak y toolbox).
- Al menos un toolbox creado. Si no tienes ninguno:

  ```bash
  toolbox create
  ```

- OpenCode instalado como Flatpak (`ai.opencode.opencode`).
- Ghidra instalado como Flatpak (`org.ghidra_sre.Ghidra`); lo hacemos en el Paso 1.
- Puerto `8082` libre en el host.
- ~3–4 GB libres para Ghidra, el JDK y las dependencias.

Comprobación básica en el host:

```bash
flatpak --version
toolbox list
flatpak list --app --columns=application,version | grep -E 'ai\.opencode|ghidra'
```

---

### 5. Paso 1 — Instalar Ghidra (Flatpak)

**Dónde:** en el host, fuera del toolbox.

```bash
# Si todavía no tienes el remoto Flathub:
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

flatpak install -y flathub org.ghidra_sre.Ghidra
```

Verifica la versión y la ruta de instalación:

```bash
flatpak list --app --columns=application,version | grep -i ghidra
# org.ghidra_sre.Ghidra   12.1.3

ls /var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra
# Debe listar: Ghidra  support  ghidraRun  Extensions  ...
```

Abre Ghidra una vez y ciérralo. Así se crean sus preferencias y su archivo de configuración:

```bash
flatpak run org.ghidra_sre.Ghidra
```

**Rutas que hay que tener claras:**

| Qué | Ruta en el host | Ruta dentro del toolbox |
| --- | --- | --- |
| Instalación de Ghidra | `/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra` | `/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra` |
| Configuración de la GUI | `~/.var/app/org.ghidra_sre.Ghidra/config/ghidra.properties` | No es la misma ruta; ver Paso 2.6 |

> Si instalaste Ghidra con `flatpak install --user`, los archivos están en
> `~/.local/share/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra` y, desde
> el toolbox, en `/run/host/var/home/$USER/.local/share/flatpak/...`. Ajusta la ruta en todos
> los comandos.

> El Flatpak de Ghidra **sí** trae un JDK 21 para su interfaz gráfica, pero ese entorno no se
> hereda cuando REA lanza Ghidra headless desde el toolbox. Por eso instalamos Java en el
> toolbox en el paso siguiente.

---

### 6. Paso 2 — Dependencias en el toolbox

**Dónde:** en el host, entrando al contenedor.

```bash
toolbox enter
```

#### 6.1 Paquetes base

```bash
sudo dnf install -y \
  java-25-openjdk java-25-openjdk-devel \
  uv nodejs npm procps-ng
```

- **Java 25:** Ghidra 12.1.3 espera un JDK moderno; el `-devel` incluye el compilador que
  algunas verificaciones de REA necesitan.
- **Node y npm:** para REA.
- **uv:** para `mcp-proxy`.
- **procps-ng:** REA necesita un `ps` compatible en Linux para comprobar la propiedad de
  procesos.

> Si tu release de Fedora no tiene `java-25-openjdk`, instala `java-21-openjdk` y
> `java-21-openjdk-devel`. Ghidra 12.1.3 acepta Java 21 o superior (el propio Flatpak se
> compila con OpenJDK 21).

Verifica:

```bash
java -version
node --version
npm --version
uv --version
```

> **Sobre Node:** `rea-agents 6.7.0` declara `^22.19.0 || ^24.11.0 || >=26.0.0`. El Node de
> Fedora suele cumplirlo. Si `npm` imprime `EBADENGINE`, actualiza Node con `fnm` o `nvm`; en
> ese caso necesitarás que `mcp-proxy` herede el `PATH` correcto (el comando del Paso 3 ya usa
> `--pass-environment`).

#### 6.2 Instalar mcp-proxy

```bash
uv tool install mcp-proxy
~/.local/bin/mcp-proxy --version
```

Esperado: `mcp-proxy 0.15.0` o superior.

#### 6.3 Instalar REA

```bash
mkdir -p ~/.npm-global
npm config set prefix "$HOME/.npm-global"
npm install --global rea-agents

~/.npm-global/bin/rea --version
```

Esperado: `6.7.0` o superior.

Agrega los binarios al `PATH` para usarlos cómodamente dentro del toolbox:

```bash
echo 'export PATH="$HOME/.npm-global/bin:$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
which rea
```

> **Por qué un prefijo propio:** en un sistema inmutable no conviene escribir en `/usr`. Con
> `npm config set prefix`, todo queda en tu `$HOME`, que se comparte con el host.

#### 6.4 Comprobar que el toolbox ve el Flatpak de Ghidra

```bash
export GHIDRA_FLATPAK=/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra
ls "$GHIDRA_FLATPAK"
```

Debe listar `Ghidra`, `support`, `ghidraRun`, `Extensions`…

#### 6.5 Verificar que REA encuentra Ghidra

```bash
export GHIDRA_INSTALL_DIR="$GHIDRA_FLATPAK"
rea doctor --provider ghidra
```

Debe reportar Ghidra como disponible. `rea --version` solo confirma que el CLI arranca; **no**
prueba la integración con Ghidra.

#### 6.6 Preparar Ghidra para el modo headless

El Flatpak reemplaza `support/launch.properties` por un enlace simbólico a
`/var/config/ghidra.properties`. Ese archivo existe dentro del sandbox del Flatpak, pero no
necesariamente dentro del toolbox. Si el enlace queda roto, `launch.sh` cree que es una compilación
de desarrollo y falla con:

```text
ERROR: Cannot launch from repo because Ghidra has not been compiled with Eclipse or Gradle.
```

La solución es crear el archivo en la ruta que espera el enlace, dentro del toolbox:

```bash
sudo mkdir -p /var/config
sudo cp "$GHIDRA_FLATPAK/support/launch.properties.orig" /var/config/ghidra.properties
```

Verifica:

```bash
test -f /var/config/ghidra.properties && echo "launch.properties listo"
```

> `/var/config` pertenece al contenedor del toolbox. Si recreas el toolbox con
> `toolbox rm` y `toolbox create`, repite este paso.

---

### 7. Paso 3 — Arrancar mcp-proxy

**Dónde:** en el host. El comando se ejecuta con `toolbox run`, así que el proceso corre dentro
del toolbox.

Abre una terminal y déjala viva:

```bash
export GHIDRA_FLATPAK=/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra

toolbox run "$HOME/.local/bin/mcp-proxy" \
  --stateless --port 8082 --host 127.0.0.1 \
  --pass-environment \
  -e GHIDRA_INSTALL_DIR "$GHIDRA_FLATPAK" \
  -e REA_ANALYSIS_PROVIDER ghidra \
  -- "$HOME/.npm-global/bin/rea" mcp
```

Qué hace cada parte:

- **`toolbox run …`** ejecuta `mcp-proxy` dentro del contenedor, donde está su Python, Node y
  Java. El puerto queda en el loopback del host porque toolbox comparte la red.
- **`--stateless`** evita depender del header `Mcp-Session-Id`; es la opción más compatible con
  clientes MCP.
- **`--host 127.0.0.1`** lo deja solo en loopback: no lo expongas a la red.
- **`--pass-environment`** hace que REA herede `PATH`, `HOME` y el resto del entorno del
  toolbox. Es importante si instalaste Node con `fnm` o `nvm`.
- **`-e GHIDRA_INSTALL_DIR …`** le dice a REA dónde están los archivos de Ghidra. La ruta es la
  que se ve **desde el toolbox** (`/run/host/...`).
- **`-e REA_ANALYSIS_PROVIDER ghidra`** fuerza el proveedor correcto si algún día instalas
  Hopper o IDA.
- **`-- "$HOME/.npm-global/bin/rea" mcp`** es el servidor stdio que el proxy lanza. Usa la ruta
  absoluta para no depender del `PATH`. **No** vuelvas a poner `toolbox run` después del `--`:
  el proxy y REA ya están dentro del mismo contenedor.

Verifica que escucha en el host:

```bash
ss -ltnp | grep 8082
# LISTEN 0  ...  127.0.0.1:8082  ...
```

Y que responde como servidor MCP (un `GET` sin streaming devuelve `406`, es normal):

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/mcp
# 406
```

Deja esa terminal abierta. Si el puerto no aparece, revisa el error de `mcp-proxy`: lo más
probable es que estés intentando ejecutarlo en el host y no dentro del toolbox.

> Si tienes más de un toolbox, agrega `--container <nombre>` a `toolbox run` para usar el mismo
> contenedor donde instalaste REA y `mcp-proxy`.

---

### 8. Paso 4 — Configurar OpenCode

**Dónde:** en el host. El archivo está en tu `$HOME`.

```bash
mkdir -p ~/.var/app/ai.opencode.opencode/config/opencode
```

Crea o edita `~/.var/app/ai.opencode.opencode/config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "rea": {
        "type": "remote",
        "url": "http://127.0.0.1:8082/mcp",
        "oauth": false
      }
    }
  }
}
```

Notas:

- OpenCode 2.x usa **`mcp.servers`**. La forma antigua `mcp.rea` todavía se lee en algunas
  versiones 2.1.x, pero no es la que escribe `opencode mcp add` y puede dejar de funcionar.
- No hace falta `"enabled": true`: los servidores están habilitados por defecto. Para dejarlo
  configurado pero desconectado, usa `"disabled": true`.
- `"oauth": false` es correcto porque el proxy no usa autenticación.
- Si OpenCode **no** fuera Flatpak, la ruta sería `~/.config/opencode/opencode.json`.
- Si el proyecto tiene su propio `opencode.json` con un servidor `rea`, ese archivo tiene
  prioridad. Revisa que no esté reemplazando la configuración global.

Reinicia OpenCode por completo. Cerrar la ventana no basta: la app mantiene un servicio de fondo.

```bash
flatpak kill ai.opencode.opencode
flatpak run ai.opencode.opencode
```

En una sesión nueva, ejecuta `/mcp` (o `/mcps` en versiones más nuevas). Debe aparecer `rea`
como conectado y sus herramientas, del orden de 200, con prefijo `rea_`.

> **No configures `rea` como `"type": "local"` con `command: ["toolbox", "run", …]`.** Ese
> comando se ejecutaría dentro del sandbox de OpenCode, donde `toolbox` no existe. La conexión
> remota por HTTP es justamente la que evita ese problema.

---

### 9. Verificación

Hazlo en este orden; el primer fallo te dice qué capa está rota.

| # | Comando | Dónde | Resultado esperado |
| --- | --- | --- | --- |
| 1 | `flatpak list --app --columns=application,version \| grep -i ghidra` | host | `org.ghidra_sre.Ghidra  12.1.3` |
| 2 | `toolbox run test -x "$HOME/.npm-global/bin/rea"` | host | sin salida, código 0 |
| 3 | `toolbox run test -f /run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra/support/analyzeHeadless` | host | código 0 |
| 4 | `toolbox run test -f /var/config/ghidra.properties` | host | código 0 |
| 5 | `ss -ltnp \| grep 8082` | host | `LISTEN 127.0.0.1:8082` |
| 6 | `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8082/mcp` | host | `406` |
| 7 | `toolbox run env GHIDRA_INSTALL_DIR=… "$HOME/.npm-global/bin/rea" doctor --provider ghidra` | host | Ghidra disponible |
| 8 | `/mcp` en OpenCode | OpenCode | `rea` conectado |

Prueba de fuego, dentro de OpenCode y con un binario que tengas derecho a analizar:

```text
Usa rea para listar las funciones del binario y descompilar la función main.
```

Si devuelve pseudocódigo real, la cadena completa funciona.

---

### 10. Uso diario

1. **Terminal 1 (host):** el `mcp-proxy` del Paso 3, vivo.
2. **OpenCode abierto:** `/mcp` → `rea` conectado.
3. Pide cosas concretas y verificables:

```text
# Buenos prompts
"Lista las funciones de ejemplo.exe y dime cuáles no tienen nombre."
"Descompila la función que referencia el string 'license'."
"Renombra esa función a CheckLicense y documenta sus parámetros."
"¿Qué imports de kernel32 usa? Marca los sospechosos."

# Malos prompts
"Analiza el binario."   ← demasiado abierto: gasta contexto y no verifica nada
```

Consejo: pide **evidencia** (dirección, nombre, pseudocódigo) y ve de una función a la vez. Así
detectas alucinaciones y aprendes.

---

### 11. Problemas frecuentes

| Síntoma | Causa | Solución |
| --- | --- | --- |
| `ModuleNotFoundError: No module named 'mcp_proxy'` o `bad interpreter` | Ejecutaste el `mcp-proxy` del toolbox directamente en el host | Arráncalo con `toolbox run "$HOME/.local/bin/mcp-proxy" …` |
| `connection refused` en `127.0.0.1:8082` | El proxy no está corriendo o murió | Arráncalo de nuevo (Paso 3) y comprueba `ss -ltnp \| grep 8082` |
| `env: «rea»: No existe el fichero o el directorio` | `rea` no está en el `PATH` del toolbox | Usa la ruta absoluta `$HOME/.npm-global/bin/rea` |
| `spawn rea ENOENT` | Ruta mal escrita después del `--` | `ls ~/.npm-global/bin/rea` en host y toolbox |
| `Cannot launch from repo because Ghidra has not been compiled…` | Enlace roto de `support/launch.properties` | Repite el Paso 2.6 |
| `ERROR: The 'java' command could not be found…` | Falta JDK en el toolbox o no se pasó el entorno | `sudo dnf install java-25-openjdk java-25-openjdk-devel`; usa `--pass-environment` |
| `POSIX Ghidra requires its inspected JDK home` | REA no pudo resolver Java | Instala el JDK, verifica `java -version`, o pasa `-e JAVA_HOME /usr/lib/jvm/java-25-openjdk` |
| `Ghidra did not finish importing and analyzing… within its … ms startup deadline` | Binario grande y timeout corto | Sube `REA_GHIDRA_STARTUP_TIMEOUT_MS` (por ejemplo `600000`); el valor por defecto ya es `330000` |
| `GET http://127.0.0.1:8082/mcp` = `406` | Correcto: MCP usa POST/streaming | Ignóralo |
| `address already in use` al arrancar el proxy | Otro proceso ocupa `8082` | `ss -ltnp \| grep 8082`, ciérralo o cambia de puerto (y la URL del JSON) |
| `EBADENGINE` al instalar `rea-agents` | Node por debajo de `^22.19 \| ^24.11 \| >=26` | Actualiza Node con `fnm`/`nvm`; usa `--pass-environment` en el proxy |
| `/mcp` no muestra `rea` | Ruta o esquema del JSON incorrectos, o OpenCode no se reinició | Revisa `mcp.servers`, la ruta del Flatpak y ejecuta `flatpak kill ai.opencode.opencode` |
| `rea` conectado pero con 0 herramientas | El proxy lanzó un comando equivocado | Revisa lo que va después del `--`; debe ser `$HOME/.npm-global/bin/rea mcp` |
| `flatpak: orden no encontrada` o `toolbox: orden no encontrada` | Estás dentro del sandbox de OpenCode | Usa una terminal del host; no configures comandos locales del host dentro del JSON |
| Todo funciona tras reiniciar, pero no antes | OpenCode no se reinició del todo | `flatpak kill ai.opencode.opencode` y vuelve a abrir |

Diagnóstico rápido de puertos en el host:

```bash
ss -ltnp | grep 8082
```

---

### 12. Seguridad

1. **`mcp-proxy` escucha sin autenticación en `127.0.0.1:8082`.** Es solo loopback, no internet,
   pero cualquier proceso local puede hablar con él y, a través de REA, abrir archivos y lanzar
   Ghidra headless. No es una frontera de seguridad. No lo expongas en `0.0.0.0`.
2. **El Flatpak de OpenCode no es un sandbox fuerte.** Su manifiesto incluye
   `--share=network`, `--filesystem=home`, `/media`, `/mnt` y `/run/media`. Puede leer y escribir
   todo tu `$HOME`, pero **no** tiene `org.freedesktop.Flatpak=talk`, así que no puede ejecutar
   comandos del host. Esta guía no necesita ese permiso; si lo agregas, le das a OpenCode (y a
   cualquier plugin o dependencia comprometida) un shell como tu usuario.
3. **Ghidra y REA procesan binarios hostiles.** Un parser con un bug o un binario preparado son
   superficie de ataque real.
4. **Inyección de prompts.** Los strings, nombres de símbolos y comentarios que REA devuelve al
   modelo provienen del binario analizado. Un binario puede contener texto que intente convencer
   al agente de ejecutar algo. Trátalo como datos, nunca como instrucciones. REA marca esas
   respuestas con `untrusted_content`.
5. **Cadena de suministro.** `rea-agents`, `mcp-proxy` y sus dependencias se instalan desde la
   red y corren con tu usuario.
6. **Si trabajas en una VM**, el radio de daño se queda en el invitado: es la mitigación más
   importante. Mantén apagadas las carpetas compartidas, el portapapeles compartido y el
   drag-and-drop de SPICE. Analiza solo binarios que tengas derecho a analizar.

---

### 13. Mantenimiento

#### Actualizar

```bash
# Host: Ghidra y OpenCode
flatpak update org.ghidra_sre.Ghidra ai.opencode.opencode

# Toolbox: REA
toolbox run "$HOME/.npm-global/bin/rea" update

# Toolbox: mcp-proxy
toolbox run uv tool upgrade mcp-proxy
```

Después de actualizar Ghidra, vuelve a copiar el archivo de launch properties por si cambió:

```bash
toolbox enter
sudo cp \
  /run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra/support/launch.properties.orig \
  /var/config/ghidra.properties
exit
```

Reinicia el proxy y OpenCode después de actualizar.

#### Desinstalar

```bash
# REA
toolbox run npm uninstall --global rea-agents
toolbox run npm config delete prefix
rm -rf ~/.npm-global

# mcp-proxy
toolbox run uv tool uninstall mcp-proxy

# Ghidra
flatpak uninstall org.ghidra_sre.Ghidra

# Si alguna vez agregaste el permiso de host a OpenCode (esta guía no lo usa)
flatpak override --user --reset ai.opencode.opencode
```

---

### 14. Notas de precisión

Estas son las diferencias importantes respecto de la versión que circuló con el workflow
“mcp-proxy en el host”:

1. **`mcp-proxy` va dentro del toolbox, no en el host.** Toolbox comparte la red del host, así
   que el puerto queda igualmente accesible. El binario instalado por `uv tool` puede quedar
   atado al Python del toolbox; ejecutarlo desde Silverblue no es portable.
2. **La configuración de OpenCode 2.x usa `mcp.servers`.** `mcp.rea` es la forma antigua.
   Tampoco se usa `enabled`; para desconectar, `disabled`.
3. **`REA_GHIDRA_STARTUP_TIMEOUT_MS` por defecto es `330000` ms (5,5 minutos), no un valor
   corto.** Poner `60000` lo reduce a un minuto y puede romper binarios grandes. Omítelo o sube
   el valor.
4. **El requisito de Node de `rea-agents 6.7.0` es `^22.19.0 || ^24.11.0 || >=26.0.0`.**
   No es `22.23.4`/`24.22.1`; el Node de Fedora suele cumplir.
5. **El Flatpak de Ghidra sí incluye un JDK 21 para la GUI**, pero ese entorno no se hereda al
   lanzar Ghidra headless desde el toolbox. Instala Java en el toolbox igualmente.
6. **El enlace `support/launch.properties` del Flatpak puede quedar roto fuera del sandbox.**
   Por eso el Paso 2.6 crea `/var/config/ghidra.properties` dentro del toolbox.
7. **`rea --version` no prueba la integración con Ghidra.** Usa `rea doctor --provider ghidra`.

---

## Parte 2 — Comandos directos (TL;DR)

Copia y pega en orden. Los comentarios indican en qué capa va cada bloque.

### 1. Host: instalar Ghidra y abrirlo una vez

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install -y flathub org.ghidra_sre.Ghidra
flatpak list --app --columns=application,version | grep -i ghidra
flatpak run org.ghidra_sre.Ghidra   # espera a que abra y ciérralo
```

### 2. Toolbox: dependencias, REA, mcp-proxy y arreglo de headless

```bash
toolbox list     # si no aparece ninguno: toolbox create
toolbox enter

sudo dnf install -y \
  java-25-openjdk java-25-openjdk-devel \
  uv nodejs npm procps-ng

uv tool install mcp-proxy

mkdir -p ~/.npm-global
npm config set prefix "$HOME/.npm-global"
npm install --global rea-agents
echo 'export PATH="$HOME/.npm-global/bin:$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

GHIDRA_FLATPAK=/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra
sudo mkdir -p /var/config
sudo cp "$GHIDRA_FLATPAK/support/launch.properties.orig" /var/config/ghidra.properties

export GHIDRA_INSTALL_DIR="$GHIDRA_FLATPAK"
rea doctor --provider ghidra

exit
```

### 3. Host: arrancar el proxy (déjalo vivo en una terminal)

```bash
GHIDRA_FLATPAK=/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra

toolbox run "$HOME/.local/bin/mcp-proxy" \
  --stateless --port 8082 --host 127.0.0.1 \
  --pass-environment \
  -e GHIDRA_INSTALL_DIR "$GHIDRA_FLATPAK" \
  -e REA_ANALYSIS_PROVIDER ghidra \
  -- "$HOME/.npm-global/bin/rea" mcp
```

Comprueba en otra terminal del host:

```bash
ss -ltnp | grep 8082
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/mcp   # 406
```

### 4. Host: configurar OpenCode

```bash
mkdir -p ~/.var/app/ai.opencode.opencode/config/opencode

cat > ~/.var/app/ai.opencode.opencode/config/opencode/opencode.json <<'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "rea": {
        "type": "remote",
        "url": "http://127.0.0.1:8082/mcp",
        "oauth": false
      }
    }
  }
}
EOF
```

> Si ya tienes otros servidores MCP, agrega `rea` dentro de `servers` en vez de reemplazar el
> archivo completo.

### 5. Host: reiniciar OpenCode

```bash
flatpak kill ai.opencode.opencode
flatpak run ai.opencode.opencode
# /mcp  →  rea conectado
```

### 6. Prueba de fuego

Dentro de OpenCode:

```text
Usa rea para listar las funciones del binario y descompilar la función main.
```
