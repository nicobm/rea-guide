# OpenCode + Ghidra + REA + romdev en Fedora Silverblue

> Guía para darle a OpenCode (Flatpak) acceso real a ingeniería inversa: desensamblar,
> descompilar, leer strings, seguir referencias y renombrar funciones, usando **REA** como
> servidor MCP y **Ghidra** instalado desde Flatpak. La Parte 2 añade **romdevtools**, un
> servidor MCP para compilar, ejecutar, analizar y recompilar ROMs de consolas retro.
>
> La guía tiene tres partes:
>
> - **Parte 1:** REA + Ghidra, explicación detallada, paso a paso, con verificación y diagnóstico.
> - **Parte 2:** romdevtools, instalación y uso junto a REA.
> - **Parte 3:** todos los comandos directos, al final, para copiar y pegar.

**Entorno de referencia:** Fedora Silverblue con GNOME, Ghidra `12.1.3` (Flatpak
`org.ghidra_sre.Ghidra`), OpenCode `2.1.3` (Flatpak `ai.opencode.opencode`), `rea-agents 6.7.0`,
`mcp-proxy 0.15.0`, `romdevtools 0.156.3`, `uv 0.13.7`, Java 25 y Node.js 24 en el toolbox.

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
- [Parte 2 — romdevtools: ROMs y recompilación](#parte-2--romdevtools-roms-y-recompilación)
  - [1. Qué añade romdevtools](#1-qué-añade-romdevtools)
  - [2. Qué no hace](#2-qué-no-hace)
  - [3. Cómo repartir el trabajo entre REA y romdev](#3-cómo-repartir-el-trabajo-entre-rea-y-romdev)
  - [4. Requisitos: Node.js 24](#4-requisitos-nodejs-24)
  - [5. Instalar romdevtools en el toolbox](#5-instalar-romdevtools-en-el-toolbox)
  - [6. Arrancar el servidor](#6-arrancar-el-servidor)
  - [7. Configurar OpenCode](#7-configurar-opencode)
  - [8. Verificación](#8-verificación)
  - [9. Recetas de uso](#9-recetas-de-uso)
  - [10. Problemas frecuentes](#10-problemas-frecuentes)
  - [11. Seguridad, licencias y mantenimiento](#11-seguridad-licencias-y-mantenimiento)
  - [12. Correcciones a la información de la charla previa](#12-correcciones-a-la-información-de-la-charla-previa)
- [Parte 3 — Comandos directos (TL;DR)](#parte-3--comandos-directos-tldr)

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
| **romdevtools** | Servidor MCP + HTTP para consolas retro | Toolbox, vía `npm -g` | Compila, ejecuta, analiza, parchea y recompila ROMs |
| **Rizin / Ghidra WASM** | Motores de análisis que trae romdev | Dentro de `romdevtools` | Desensamblado, CFG, xrefs y pseudocódigo C de ROMs |
| **mcp-proxy** | Puente stdio ⇄ HTTP para MCP | Toolbox, vía `uv tool` | Expone `rea mcp` en `127.0.0.1:8082` |
| **uv** | Gestor de paquetes Python | Toolbox | Instala `mcp-proxy` en un entorno aislado |
| **Java 25** | JDK | Toolbox | Lo usa Ghidra headless al ser lanzado por REA |

En la **Parte 1**, REA es el único servidor MCP. La **Parte 2** agrega `romdevtools`, que es un
proceso independiente y escucha en otro puerto (`7331`). No hace falta la extensión GhidraMCP ni
un bridge aparte para ninguno de los dos.

---

### 3. Las tres capas: host, toolbox y sandbox

Este es el punto donde más gente se atasca. Hay tres “mundos” y los comandos solo existen en
algunos:

| Comando | Host | Toolbox | Sandbox de OpenCode |
| --- | --- | --- | --- |
| `flatpak`, `flatpak kill` | ✅ | ❌ | ❌ |
| `toolbox enter/run`, `ss`, `curl` | ✅ | ❌ | ❌ |
| `dnf`, `java`, `node`, `npm`, `uv`, `rea`, `mcp-proxy` | ❌ | ✅ | ❌ |
| `npm-24`, `romdev-server` (Parte 2) | ❌ | ✅ | ❌ |
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
  el host, en el toolbox y en el sandbox de OpenCode (que tiene `--share=network`). Lo mismo vale
  para `127.0.0.1:7331`, el puerto de romdev en la Parte 2.
- **Dos servidores, dos puertos:** REA escucha en `8082` (a través de `mcp-proxy`) y romdev en
  `7331`. Son procesos independientes; podés tener los dos encendidos a la vez.

---

### 4. Requisitos previos

- Fedora Silverblue (o cualquier sistema inmutable con `rpm-ostree`, Flatpak y toolbox).
- Al menos un toolbox creado. Si no tienes ninguno:

  ```bash
  toolbox create
  ```

- OpenCode instalado como Flatpak (`ai.opencode.opencode`).
- Ghidra instalado como Flatpak (`org.ghidra_sre.Ghidra`); lo hacemos en el Paso 1.
- Puertos `8082` y `7331` libres en el host.
- ~3–4 GB libres para Ghidra, el JDK y las dependencias. romdevtools suma varios cientos de MB
  más y carga los binarios de cada plataforma la primera vez que la usas.

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

> **Prueba de humo sin binario propio:** pedile a OpenCode que abra `/usr/bin/ls` con REA y
> muestre el `binary_overview`. Es un ELF x86_64 del toolbox; si devuelve segmentos y funciones,
> Ghidra headless está funcionando. También sirve para comprobar que el arreglo de
> `launch.properties` del Paso 2.6 está bien aplicado.

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
| `rea doctor` dice `environment_healthy: false` pero Ghidra está `ok` | Faltan Hopper, IDA, JADX o Kuna, o el skill de REA; son checks opcionales | Ignóralo para el flujo con Ghidra; revisá que `providerInspections` muestre `ghidra: available: true` |
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

## Parte 2 — romdevtools: ROMs y recompilación

Esta parte es **independiente y complementaria** de la Parte 1. REA + Ghidra trabajan binarios
nativos; romdevtools trabaja ROMs, emuladores y recompilación. Forma parte del flujo normal: se
instala y configura igual que REA, y la guía asume que vas a tener los dos servidores MCP
disponibles.
romdevtools no modifica REA ni Ghidra: es un segundo servidor MCP para compilar homebrew, ejecutar
el juego en un core de emulador, inspeccionar memoria y generar proyectos de descompilación o de
recompilación estática.

### 1. Qué añade romdevtools

`romdevtools` es un *tool server* (MCP + HTTP) que corre en Node y trae, empaquetados como
WebAssembly o JavaScript, todo lo necesario para trabajar con 19 consolas y computadoras: desde
Atari 2600 hasta Dreamcast, más PICO-8, wasmcart y jsgame.

No es un emulador con interfaz: es una API para que el agente haga el trabajo. Expone 38
herramientas con forma de verbo y un eje de operación, por ejemplo `memory({op:'search'})`,
`disasm({target:'cfg'})` o `decomp({op:'...'})`.

| Área | Qué puedes hacer |
| --- | --- |
| **Compilar** | Toolchains por plataforma (cc65, SDCC, RGBDS, asar, GCC de m68k/ARM/MIPS/SH, SGDK, PVSnesLib, libtonc…) compiladas a WASM. `examples({op:'fork'})` crea un juego de ejemplo y `build({output:'run'})` lo compila, lo carga y lo ejecuta |
| **Ejecutar** | 15 cores de libretro como WASM: avanzar frames, capturar pantalla, simular input, leer/escribir memoria, guardar estados |
| **Analizar** | Rizin y el descompilador de Ghidra compilados a WASM: funciones, CFG, xrefs y pseudocódigo C en 18 de las 19 plataformas |
| **Parchear** | Búsqueda de valores, breakpoints de lectura/escritura/ejecución, cobertura, parcheo de ROM, reinyección de assets comprimidos y una base de datos de trucos como mapa de RAM |
| **Recompilar** | Recompilación estática de ROMs de NES, SNES, Game Boy, Game Boy Color, GBA, Master System, Game Gear y Genesis a un *wasmcart*, con verificación frame a frame |
| **Descompilar por coincidencia** | Proyectos splat de N64 y PlayStation: generar C candidato con `m2c`, compilar con el compilador original, comparar instrucción por instrucción, buscar con el *permuter* y verificar el ROM completo |
| **Assets** | Convertir PNG/Aseprite/Tiled/GIF a tiles, tilemaps y paletas; generar audio para los chips de cada consola |

El paquete npm ya incluye los binarios de todas las plataformas; se cargan e inicializan la
primera vez que usas cada consola. El servidor también publica documentación navegable en
`http://127.0.0.1:7331/documentation` y un *skill* para agentes en
`http://127.0.0.1:7331/skills/romdev/SKILL.md`.

### 2. Qué no hace

Conviene ser explícito para no repetir promesas infladas:

- **No recompila N64, PS2, Xbox, Xbox 360 ni GameCube.** La recompilación estática de romdev
  cubre consolas de 8 y 16 bits más GBA. Para N64 y PlayStation hay *descompilación por
  coincidencia* (proyectos tipo splat), que es otro flujo y no produce un port nativo de PC.
- **No incluye N64: Recompiled, PS2Recomp, XenonRecomp ni xboxrecomp.** Son proyectos aparte,
  con sus propios repositorios; no forman parte de romdev.
- **No trae el compilador IDO de SGI.** Es propietario y no se distribuye. Para descompilar N64
  con IDO hay que compilarlo con `packages/romdevtools/scripts/build-ido.sh` del repositorio, o
  apuntar `ROMDEV_IDO_DIR` a una copia propia. Sin eso, ese flujo específico no funciona.
- **No reemplaza a Ghidra sobre binarios nativos.** Para ELF/PE de PC, REA + Ghidra sigue siendo
  la herramienta de la Parte 1.
- **No convierte un ROM en un juego de PC jugable con un clic.** Genera código y proyectos; el
  trabajo de depuración, integración y verificación sigue existiendo.

> La palabra “recompilar” se usa para dos cosas distintas: **recompilación estática** (traducir el
> código de máquina de la ROM a código que se compila para otra plataforma) y **descompilación
> por coincidencia** (generar C que, compilado con el compilador original, reproduce el ROM byte
> por byte). romdev hace ambas, pero solo en las plataformas de la tabla anterior.

### 3. Cómo repartir el trabajo entre REA y romdev

| Tarea | Herramienta |
| --- | --- |
| Analizar un `.exe`, `.dll`, `.so` o ELF nativo | REA + Ghidra (Parte 1) |
| Renombrar funciones, tipos y comentarios en un proyecto Ghidra | REA + Ghidra |
| Abrir una ROM `.nes`, `.gb`, `.sfc`, `.md`, `.gba`… | romdev |
| Compilar y ejecutar un juego homebrew | romdev |
| Capturar pantalla, input, memoria y estados de un juego | romdev |
| Descompilar código de una ROM a pseudocódigo C | romdev (Rizin/Ghidra WASM) |
| Recompilar una ROM de 8/16 bits a wasmcart | romdev |
| Proyecto de descompilación por coincidencia de N64/PSX | romdev (`decomp`) |
| Depurar un binario nativo con GUI | Ghidra Flatpak |

Son procesos distintos y **pueden convivir**: REA en `127.0.0.1:8082`, romdev en
`127.0.0.1:7331`. OpenCode ve dos servidores MCP separados y tú eliges cuál usar según el
artefacto. El Ghidra del Flatpak no se toca: romdev lleva su propio descompilador de Ghidra
compilado a WASM, no llama al Ghidra instalado.

### 4. Requisitos: Node.js 24

romdevtools exige **Node.js 24 o superior** (`engines: >=24`). No alcanza con el Node 22 que
Fedora instala por defecto. Verifica dentro del toolbox:

```bash
toolbox enter
node --version
```

Si dice `v22.x` o menos, instala los paquetes versionados de Fedora:

```bash
sudo dnf install -y nodejs24 nodejs24-npm
/usr/bin/node-24 --version
/usr/bin/npm-24 --version
```

Fedora 43 y posteriores tienen `nodejs24`. Los binarios se llaman `node-24` y `npm-24` **a
propósito**, para no pisar el `node` del sistema que usan otras herramientas. Por eso más abajo
creamos un lanzador que invoca `node-24` explícitamente.

> Si tu release de Fedora no tiene `nodejs24`, instala Node 24 con `fnm` o `nvm` y usa la ruta
> absoluta de ese `node` en el lanzador del Paso 5. No cambies el `node` global del toolbox: REA
> también lo usa y conviene no romperlo.

### 5. Instalar romdevtools en el toolbox

**Dónde:** dentro del toolbox. Reutiliza el prefijo `~/.npm-global` de la Parte 1, así que no
escribe en `/usr`.

```bash
npm-24 config set prefix "$HOME/.npm-global"
npm-24 install --global romdevtools
```

Comprueba la versión instalada:

```bash
/usr/bin/node-24 -p "require('$HOME/.npm-global/lib/node_modules/romdevtools/package.json').version"
# 0.156.3 (o superior)
```

El binario que instala npm se llama `romdevtools`, pero su *shebang* es `#!/usr/bin/env node` y
podría resolverse al Node 22. Para evitar ambigüedades, crea un lanzador que use `node-24`:

```bash
cat > "$HOME/.local/bin/romdev-server" <<'EOF'
#!/bin/sh
exec /usr/bin/node-24 "$HOME/.npm-global/lib/node_modules/romdevtools/src/mcp/server.js" "$@"
EOF
chmod +x "$HOME/.local/bin/romdev-server"
```

Verifica que existe:

```bash
test -x "$HOME/.local/bin/romdev-server" && echo "lanzador listo"
```

> El lanzador también sirve para pasar opciones: `romdev-server --port 7332`, o variables como
> `PORT=7332`, `HOST=127.0.0.1`, `ROMDEV_HOST_WORKERS=0` y `ROMDEV_GPU=<PCI>`.

### 6. Arrancar el servidor

**Dónde:** en el host, con `toolbox run`. Déjalo vivo en su propia terminal, igual que el proxy
de REA.

```bash
toolbox run "$HOME/.local/bin/romdev-server"
```

La primera vez descarga los paquetes que falten y tarda. Cuando termine, debe imprimir algo
parecido a:

```text
romdev: server up pid=... v0.156.3
romdev (v0.156.3) listening on http://localhost:7331/mcp
```

En otra terminal del host, verifica que escucha y que responde:

```bash
ss -ltnp | grep 7331
curl -s http://127.0.0.1:7331/healthz
# {"ok":true,"version":"0.156.3","sessions":0}
```

`/healthz` es la comprobación más limpia: no requiere un *handshake* MCP. `GET /mcp` puede
devolver un `4xx` porque MCP usa `POST`; eso es normal.

> No hace falta `mcp-proxy` para romdev. El servidor ya habla HTTP/streamable MCP nativamente.
> Tampoco hace falta `--stateless`: romdev soporta tanto el protocolo con sesión como la revisión
> sin estado.

### 7. Configurar OpenCode

Agrega `romdev` **junto a** `rea` en el mismo archivo
`~/.var/app/ai.opencode.opencode/config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "rea": {
        "type": "remote",
        "url": "http://127.0.0.1:8082/mcp",
        "oauth": false
      },
      "romdev": {
        "type": "remote",
        "url": "http://127.0.0.1:7331/mcp",
        "oauth": false
      }
    }
  }
}
```

Los dos servidores quedan configurados y se conectan automáticamente. Si en una sesión no quieres
usar uno, desconéctalo desde `/mcps` en OpenCode: eso no borra la configuración. Para dejarlo
desconectado de forma permanente, agrega `"disabled": true` a su bloque.

Notas:

- OpenCode 2.x usa **`mcp.servers`**. El README de romdev muestra la forma antigua `mcp.romdev`;
  no la uses en esta versión.
- No hace falta `"enabled": true`; los servidores están habilitados por defecto.
- `"oauth": false` es correcto: el servidor no pide autenticación.
- Si el proyecto tiene su propio `opencode.json` con `mcp`, ese archivo tiene prioridad. Revisa
  que no esté ocultando la configuración global.
- El Flatpak de OpenCode ya comparte la red del host, así que `127.0.0.1:7331` funciona sin
  permisos extra. Si en tu instalación la red estuviera aislada, la corrección es
  `flatpak override --user --share=network ai.opencode.opencode` y reiniciar; **no** existe un
  `flatpak run --share=network`.

Reinicia OpenCode por completo:

```bash
flatpak kill ai.opencode.opencode
flatpak run ai.opencode.opencode
```

En una sesión nueva, `/mcp` debe mostrar **`rea` y `romdev` conectados**. romdev aporta del orden
de 38 herramientas.

### 8. Verificación

| # | Comando | Dónde | Resultado esperado |
| --- | --- | --- | --- |
| 1 | `toolbox run /usr/bin/node-24 --version` | host | `v24.x` |
| 2 | `toolbox run test -x "$HOME/.local/bin/romdev-server"` | host | código 0 |
| 3 | `ss -ltnp \| grep 7331` | host | `LISTEN 127.0.0.1:7331` |
| 4 | `curl -s http://127.0.0.1:7331/healthz` | host | `{"ok":true,...}` |
| 5 | `curl -s http://127.0.0.1:7331/openapi.json \| head -c 80` | host | JSON con `openapi` |
| 6 | `/mcp` en OpenCode | OpenCode | `rea` y `romdev` conectados |

Prueba de fuego sin ROM propio, usando un ejemplo incluido:

```text
Usa romdev para crear un proyecto de ejemplo de Game Boy llamado demo en ~/roms/demo.
Después compílalo con build({output:'run'}) y muéstrame el screenshot del primer frame.
```

Si eso funciona, la cadena compilador WASM + emulador + captura está operativa.

### 9. Recetas de uso

Pide acciones concretas y exige evidencia (dirección, frame, ruta, diferencia). Ejemplos:

```text
# Explorar una ROM propia
"Abre ~/roms/game.gb con romdev, lista las funciones que detecta Rizin y descompilá la
rutina que lee el registro de input. Dame direcciones y pseudocódigo."

# Recompilación estática de Genesis a wasmcart
"Usa disasm con target recompile y emit wasmcart sobre ~/roms/juego.md. Después corre
frame con op verifyRecompile y dime en qué frame y en qué registro diverge, si es que diverge."

# Compilar y probar homebrew
"Forkea el ejemplo genesis/platformer como ~/roms/mi-juego, compílalo con build output run
y muéstrame el screenshot. Si no compila, lee el error y corrígelo."

# Descompilación por coincidencia (PSX/N64)
"En este proyecto splat de PlayStation, genera C candidato con m2c para la función 0x80012340,
compílalo con el compilador del proyecto, compara instrucción por instrucción y ejecuta el
permuter solo si no coincide. No declares éxito sin el chequeo de ROM completo."

# Documentación antes de tocar nada
"Lee la guía de plataforma de SNES y el playbook de romhacking de romdev antes de proponer
un parche para este ROM."
```

Buenas prácticas:

- Trabaja sobre **copias** de tus ROMs y guarda los proyectos fuera de la carpeta de la ROM
  original.
- Una plataforma a la vez: cada una carga sus binarios y tiene sus trampas.
- Para emuladores 3D (N64, PSX, Dreamcast) el primer arranque es más lento y consume más RAM.
- Si el agente dice “recompilé el juego”, pídele la verificación (`verifyRecompile`,
  `findDiverge`, comparación de hash). Sin eso, es una afirmación sin evidencia.

### 10. Problemas frecuentes

| Síntoma | Causa | Solución |
| --- | --- | --- |
| `npm-24: orden no encontrada` | No instalaste `nodejs24-npm` | `sudo dnf install nodejs24 nodejs24-npm` |
| `Unsupported engine` o errores de sintaxis al arrancar | Se está usando Node 22 | Usa el lanzador con `/usr/bin/node-24`; verifica `romdev-server` |
| `Cannot find module '.../romdevtools/src/mcp/server.js'` | El prefijo de npm no es `~/.npm-global` o la instalación falló | Revisa `npm-24 config get prefix` y reinstala |
| `EADDRINUSE` en `7331` | Ya hay otro romdev (u otro proceso) en ese puerto | `ss -ltnp \| grep 7331`, ciérralo o arranca con `--port 7332` y cambia la URL del JSON |
| `healthz` no responde | El servidor murió o no arrancó | Mira la terminal del Paso 6; prueba `toolbox run "$HOME/.local/bin/romdev-server"` a mano |
| `/mcp` no muestra `romdev` | JSON con la forma antigua `mcp.romdev`, URL mal escrita o OpenCode no se reinició | Usa `mcp.servers`, revisa el puerto y `flatpak kill ai.opencode.opencode` |
| `romdev` conectado pero con 0 herramientas | El proceso arrancó con un Node incompatible o crasheó | Revisa la salida del servidor y la versión de Node |
| La primera llamada a una consola tarda mucho | Está cargando e inicializando los binarios WASM de esa plataforma | Espera; las siguientes son rápidas |
| `decomp` pide IDO | El compilador de SGI no se distribuye | Compílalo con `build-ido.sh` o define `ROMDEV_IDO_DIR` |
| `playtest` no abre ventana | No hay `DISPLAY`/`WAYLAND_DISPLAY` o faltan librerías gráficas en el toolbox | Úsalo desde una sesión gráfica del host; el resto de herramientas funciona headless |
| Errores de EGL/GLES en consolas 3D | `native-gles` no encuentra el driver en el contenedor | Instala `mesa-libEGL`/`mesa-libGLES` en el toolbox o usa el render por software |
| Consumo de RAM alto | Cada sesión puede tener su propio core en un proceso | `ROMDEV_HOST_WORKERS=0` lo desactiva; cierra sesiones viejas |

### 11. Seguridad, licencias y mantenimiento

**Seguridad**

1. **Sin autenticación en `127.0.0.1:7331`.** Es solo loopback, pero cualquier proceso local
   puede hablar con el servidor, cargar ROMs, compilar y escribir archivos de proyecto. No lo
   expongas en `0.0.0.0`.
2. **Ejecuta binarios nativos.** romdev incluye módulos nativos (SDL, GL/EGL) y lanza procesos
   de host por sesión. Corre con tu usuario y tus permisos.
3. **Contenido de la ROM no confiable.** Nombres de símbolos, strings y comentarios que el
   agente lee vienen del ROM. Pueden contener instrucciones maliciosas. Son datos, no órdenes.
4. **Cadena de suministro.** `romdevtools` arrastra decenas de paquetes `romdev-*` y
   dependencias de npm; todos se ejecutan con tu usuario. Instala versiones publicadas y revisa
   `NOTICE`.
5. **Aislamiento.** Si trabajas en una VM, el daño queda en el invitado. Es la mitigación más
   útil, igual que en la Parte 1.

**Licencias**

- El código propio de romdev es MIT.
- Los componentes de terceros que empaqueta conservan sus licencias (GPL, MPL, no comercial,
  según el caso) y están listados en `NOTICE`.
- Los juegos que crees con romdev son tuyos, incluidas las librerías de runtime permisivas que
  enlacen.
- Un **cartucho recompilado** contiene código del emulador de referencia (GPL/MPL/no comercial)
  **y** código del juego original. No lo distribuyas. Cada cartucho recompilado incluye una
  entrada `licence` que nombra esos componentes.
- Usa solo ROMs que poseas legalmente. La herramienta no incluye juegos.

**Mantenimiento**

```bash
# Actualizar romdevtools
toolbox run /usr/bin/npm-24 install --global romdevtools@latest

# Ver la versión instalada y la que corre
toolbox run /usr/bin/node-24 -p "require('$HOME/.npm-global/lib/node_modules/romdevtools/package.json').version"
curl -s http://127.0.0.1:7331/healthz
```

Después de actualizar, reinicia el servidor y OpenCode.

```bash
# Desinstalar
toolbox run /usr/bin/npm-24 uninstall --global romdevtools
rm -f "$HOME/.local/bin/romdev-server"
```

La caché de npm queda en `~/.npm` y los paquetes instalados en `~/.npm-global`; puedes limpiar
la caché con `npm-24 cache clean --force` si necesitas recuperar espacio.

### 12. Correcciones a la información de la charla previa

Si llegaste acá desde esa conversación, estas son las diferencias importantes:

1. **romdev no recompila N64, PS2, Xbox, Xbox 360 ni GameCube.** La recompilación estática cubre
   NES, SNES, GB, GBC, GBA, Master System, Game Gear y Genesis. N64 y PlayStation solo tienen
   descompilación por coincidencia.
2. **N64: Recompiled, PS2Recomp, XenonRecomp y xboxrecomp no vienen dentro de romdev.** Son
   proyectos independientes.
3. **No hay que configurar Ghidra Flatpak para romdev.** romdev trae el descompilador de Ghidra
   como WebAssembly. Son instalaciones separadas y no se comunican.
4. **La configuración de OpenCode 2.x es `mcp.servers`, no `mcp.romdev`.** El README de romdev
   todavía muestra la forma antigua.
5. **No hace falta `--share=network` al lanzar el Flatpak** (y `flatpak run` no acepta esa
   opción). Flatpak comparte la red del host por defecto; si hiciera falta, se usa
   `flatpak override --user --share=network`.
6. **`10.0.2.2` no aplica acá.** Es una dirección de NAT de QEMU/VirtualBox, no de Flatpak.
   `127.0.0.1:7331` es la correcta.
7. **Node 24 es obligatorio.** El `node` por defecto de Fedora puede ser 22 y no alcanza.
8. **No es un servicio que se instala “dentro de REA”.** Es otro servidor MCP, con otro puerto y
   otro ciclo de vida.
9. **La descompilación por coincidencia no es automática ni mágica.** Requiere proyecto splat,
   compilador original, comparación por instrucción y, en N64 con IDO, un compilador propietario
   que no se distribuye.
10. **Los recompilados no son ports de PC listos para jugar.** Son código generado que hay que
    verificar y depurar; la herramienta ayuda, no reemplaza el trabajo.

---

## Parte 3 — Comandos directos (TL;DR)

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
toolbox run "$HOME/.local/bin/mcp-proxy" \
  --stateless --port 8082 --host 127.0.0.1 \
  --pass-environment \
  -e GHIDRA_INSTALL_DIR /run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra \
  -e REA_ANALYSIS_PROVIDER ghidra \
  -- "$HOME/.npm-global/bin/rea" mcp
```

Comprueba en otra terminal del host:

```bash
ss -ltnp | grep 8082
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/mcp   # 406
```

### 4. Host: configurar OpenCode

El archivo debe quedar con **los dos servidores**, `rea` y `romdev`:

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
      },
      "romdev": {
        "type": "remote",
        "url": "http://127.0.0.1:7331/mcp",
        "oauth": false
      }
    }
  }
}
EOF
```

> Si ya tienes otros servidores MCP, agrega `rea` y `romdev` dentro de `servers` en vez de
> reemplazar el archivo completo. Los dos quedan configurados; puedes conectarlos o
> desconectarlos desde `/mcps` sin editar el JSON.

### 5. Host: reiniciar OpenCode

```bash
flatpak kill ai.opencode.opencode
flatpak run ai.opencode.opencode
# /mcp  →  rea y romdev conectados
```

### 6. Prueba de fuego

Dentro de OpenCode:

```text
Usa rea para listar las funciones del binario y descompilar la función main.
```

Si no tienes un binario a mano:

```text
Usa rea para abrir /usr/bin/ls con Ghidra y mostrarme el binary_overview.
```

### 7. romdevtools

**Toolbox:**

```bash
toolbox enter

sudo dnf install -y nodejs24 nodejs24-npm   # Fedora 43+
npm-24 config set prefix "$HOME/.npm-global"
npm-24 install --global romdevtools

cat > "$HOME/.local/bin/romdev-server" <<'EOF'
#!/bin/sh
exec /usr/bin/node-24 "$HOME/.npm-global/lib/node_modules/romdevtools/src/mcp/server.js" "$@"
EOF
chmod +x "$HOME/.local/bin/romdev-server"

exit
```

**Host — terminal aparte, déjala viva:**

```bash
toolbox run "$HOME/.local/bin/romdev-server"
```

**Host — comprobar:**

```bash
ss -ltnp | grep 7331
curl -s http://127.0.0.1:7331/healthz
```

**OpenCode — el JSON completo con ambos servidores ya está en la sección 4.** Así tiene que
quedar:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "rea": {
        "type": "remote",
        "url": "http://127.0.0.1:8082/mcp",
        "oauth": false
      },
      "romdev": {
        "type": "remote",
        "url": "http://127.0.0.1:7331/mcp",
        "oauth": false
      }
    }
  }
}
```
