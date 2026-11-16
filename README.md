# Ghidra + MCP en Fedora Silverblue

> **Qué es esto:** guía paso a paso para darle a un agente (OpenCode) acceso a ingeniería
> inversa real sobre binarios: desensamblar, descompilar, leer strings, renombrar
> funciones, seguir referencias, etc.
>
> **Estado:** verificada en una Silverblue con `Ghidra 12.1.3` (Flatpak `org.ghidra_sre.Ghidra`),
> `OpenCode 2.1.3` (Flatpak `ai.opencode.opencode`), `GhidraMCP 7.0.0`, `rea-agents 6.7.0`,
> `Node 22.23.1`, `Java 25`, `uv 0.13.7`, `mcp-proxy 0.15.0`.

---

## Inicio rápido (Ruta A, sin permisos de host)

```bash
# 1) En el toolbox
toolbox enter
uv tool install mcp-proxy

# 2) Vuelve al host (`exit`) y arranca el puente; déjalo vivo en una terminal
toolbox run "$HOME/.local/bin/mcp-proxy" --stateless --port 8082 --host 127.0.0.1 \
  -e GHIDRA_INSTALL_DIR /run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra \
  -e REA_GHIDRA_STARTUP_TIMEOUT_MS 60000 \
  -- "$HOME/.npm-global/bin/rea" mcp

# 3) En ~/.var/app/ai.opencode.opencode/config/opencode/opencode.json
#    "rea": { "type": "remote", "url": "http://127.0.0.1:8082/mcp", "oauth": false, "enabled": true }

# 4) Reinicia OpenCode
flatpak kill ai.opencode.opencode && flatpak run ai.opencode.opencode
# /mcp  →  rea conectado, ~207 herramientas
```

Probado con `mcp-proxy 0.15.0`, `rea-agents 6.7.0`, `Ghidra 12.1.3` y `OpenCode 2.1.3`.
El detalle de cada paso, la Ruta B (GhidraMCP + bridge), el diagnóstico y la sección de
seguridad están más abajo.

---

## Índice

1. [Qué vas a montar](#1-qué-vas-a-montar)
2. [Quién es quién](#2-quién-es-quién)
3. [Las tres capas: host, toolbox y sandbox](#3-las-tres-capas-host-toolbox-y-sandbox)
4. [Requisitos previos](#4-requisitos-previos)
5. [Paso 0 — Permisos del Flatpak de OpenCode (solo variante stdio)](#paso-0--permisos-del-flatpak-de-opencode-solo-variante-stdio)
6. [Paso 1 — Preparar el toolbox](#paso-1--preparar-el-toolbox)
7. [Paso 2 — Instalar Ghidra (Flatpak)](#paso-2--instalar-ghidra-flatpak)
8. [Paso 3 — Instalar REA (ruta recomendada)](#paso-3--instalar-rea-ruta-recomendada)
9. [Paso 4 — Extensión GhidraMCP dentro de Ghidra (opcional)](#paso-4--extensión-ghidramcp-dentro-de-ghidra-opcional)
10. [Paso 5 — Bridge MCP en Python (opcional)](#paso-5--bridge-mcp-en-python-opcional)
11. [Paso 6 — Configurar OpenCode](#paso-6--configurar-opencode)
12. [Paso 7 — Verificación completa](#paso-7--verificación-completa)
13. [Uso diario](#uso-diario)
14. [Problemas frecuentes](#problemas-frecuentes)
15. [Seguridad: qué estás confiando y por qué](#seguridad-qué-estás-confiando-y-por-qué)
16. [Mantenimiento: actualizar y desinstalar](#mantenimiento-actualizar-y-desinstalar)
17. [Glosario](#glosario)

---

## 1. Qué vas a montar

Hay **dos rutas independientes** para que el agente hable con Ghidra. No son complementarias:
son alternativas. Elige una.

### Ruta A — REA por HTTP (recomendada)

```
OpenCode (Flatpak)
   │  HTTP  →  http://127.0.0.1:8082/mcp
   ▼
mcp-proxy  ──stdio──►  rea mcp        (los dos dentro del toolbox)
   │
   ▼
Ghidra 12.1.3 headless (Flatpak, vía GHIDRA_INSTALL_DIR)
   │
   ▼
Resultados: pseudocódigo, assembly, strings, xrefs, símbolos…
```

Ventajas: un solo servidor MCP, ~200 herramientas, no necesitas tener Ghidra abierto, REA también
sirve para JavaScript, .NET, APK, firmware, etc., y **OpenCode nunca sale de su sandbox**: habla
HTTP contra `127.0.0.1`, así que no hace falta `org.freedesktop.Flatpak=talk`.

El precio: igual que la Ruta B, hay que dejar un proceso vivo (`mcp-proxy`) en una terminal. Si
prefieres que OpenCode arranque el servidor solo, existe la variante stdio (§6.2), pero esa sí
requiere el permiso de host.

### Ruta B — GhidraMCP + bridge (la de bethington)

```
OpenCode (Flatpak)
   │  HTTP  →  http://127.0.0.1:8081/mcp
   ▼
bridge-mcp-ghidra (Python / uv)
   │  HTTP  →  http://127.0.0.1:8089
   ▼
Ghidra GUI abierto + extensión GhidraMCP (plugin activo)
```

Ventajas: trabaja sobre **lo que tú tienes abierto** en el CodeBrowser, así que puedes ir
mirando la pantalla mientras el agente opera. Requiere Ghidra abierto y un programa cargado.

> **Regla importante:** en `opencode.json` deja **activado un solo servidor**. Si activas los
> dos, el modelo ve dos catálogos casi idénticos y se confunde (herramientas duplicadas).

---

## 2. Quién es quién

| Pieza | Qué es | Dónde vive | Para qué sirve |
| --- | --- | --- | --- |
| **Ghidra** | Suite de ingeniería inversa de la NSA | Flatpak, en el host | Desensamblar, descompilar, analizar |
| **GhidraMCP** (`.zip`) | Extensión de Ghidra (Java) | Dentro de Ghidra | Levanta un servidor HTTP en `127.0.0.1:8089` |
| **GhidraMCPPlugin** | El plugin concreto dentro de la extensión | Dentro de Ghidra | Hay que **activarlo** a mano; no basta con instalar el zip |
| **bridge-mcp-ghidra** (`.whl`) | Puente en Python | toolbox (vía `uv tool`) | Traduce MCP ⇄ HTTP de Ghidra, escucha en `127.0.0.1:8081` |
| **REA** (`rea-agents`) | CLI + servidor MCP en Node | toolbox (vía `npm -g`) | Habla con Ghidra (y otros motores) y expone MCP |
| **mcp-proxy** | Puente stdio ⇄ HTTP para MCP | toolbox (vía `uv tool`) | Expone `rea mcp` en `127.0.0.1:8082` sin sacar a OpenCode del sandbox |
| **uv** | Gestor de paquetes Python (Rust) | toolbox | Instala el bridge y `mcp-proxy` en entornos aislados |
| **toolbox** | Contenedor Fedora con tu `$HOME` compartido | host | Aquí van Node, Java, uv, REA y el bridge |
| **OpenCode** | El agente | Flatpak, en el host | Consume los servidores MCP |

**Dato clave:** REA **no necesita** ni la extensión GhidraMCP ni el bridge. Son dos caminos
distintos para el mismo objetivo.

---

## 3. Las tres capas: host, toolbox y sandbox

Este es el punto donde más gente se atasca. Hay tres "mundos" y los comandos solo existen
en algunos:

| Comando | Host | toolbox | Sandbox de OpenCode |
| --- | --- | --- | --- |
| `flatpak`, `flatpak override` | ✅ | ❌ | ❌ |
| `toolbox enter/run` | ✅ | ❌ | ❌ |
| `dnf`, `java`, `node`, `uv`, `rea` | ❌ | ✅ | ❌ |
| `mcp-proxy` (Ruta A) | ❌ | ✅ | ❌ |
| `flatpak-spawn --host …` (solo variante stdio) | ❌ | ❌ | ✅ |

Reglas prácticas:

- Los comandos `sudo dnf …` van **dentro de toolbox**.
- `flatpak install` y `flatpak override` van **en el host**.
- Si estás escribiendo en OpenCode y ves `flatpak: orden no encontrada`, no estás en el host:
  estás dentro del sandbox del Flatpak. En la Ruta A no deberías necesitar salir nunca: todo va por
  `127.0.0.1`. Si eliges la variante stdio, usa `flatpak-spawn --host <comando>`.

**El truco de `$HOME`:** toolbox comparte tu carpeta personal con el host. Por eso un binario
instalado en `~/.npm-global/bin` o `~/.local/bin` dentro del contenedor **también existe en el
host**, y viceversa. Así el host puede arrancar `mcp-proxy` y `rea` sin reinstalar nada.

**La red también se comparte:** los Flatpak y los contenedores de toolbox usan la misma pila de
red del host, así que `127.0.0.1:8081`, `127.0.0.1:8082` y `127.0.0.1:8089` significan lo mismo en
las tres capas. **De ahí sale la Ruta A:** el sandbox solo abre un socket a loopback; no ejecuta
nada del host.

---

## 4. Requisitos previos

- Fedora Silverblue (o cualquier inmutable con `rpm-ostree` + Flatpak + toolbox).
- Al menos un toolbox creado (`toolbox list`; si no hay, `toolbox create`).
- OpenCode instalado como Flatpak (`ai.opencode.opencode`).
- ~3 GB libres: Ghidra + JDK + extensiones pesan.
- Puerto `8082` libre en el host (lo usa `mcp-proxy`; 8081 y 8089 son de la Ruta B).
- Un binario de práctica (un `.exe` o un ELF pequeño).

Comprueba lo básico:

```bash
# En el host
flatpak --version
toolbox list

# Dentro del toolbox
toolbox enter
cat /etc/fedora-release
```

> En Silverblue el toolbox suele ser la misma versión que el host. Si tu toolbox es más antiguo,
> `toolbox create --distro fedora --release 43` (o la que uses) te da uno nuevo.

---

## Paso 0 — Permisos del Flatpak de OpenCode (solo variante stdio)

**Dónde:** en el host.

> **Si vas por la Ruta A (REA por HTTP), sáltate este paso entero.** OpenCode solo abre un socket a
> `127.0.0.1:8082` y el Flatpak ya comparte la red del host: no necesita ejecutar nada fuera del
> sandbox ni permiso alguno. Compruébalo con el proxy arrancado:
> `curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/mcp` → `406` (normal en GET: el endpoint espera POST).

Este permiso solo hace falta en la **variante stdio** (§6.2), donde OpenCode lanza
`flatpak-spawn --host toolbox run … rea mcp` por sí mismo. Por defecto, un Flatpak está encerrado:
no puede ejecutar programas del host, así que hay que darle permiso en el bus de sesión:

```bash
flatpak override --user --talk-name=org.freedesktop.Flatpak ai.opencode.opencode

# Verificar
flatpak override --user --show ai.opencode.opencode
```

Debe aparecer:

```ini
[Session Bus Policy]
org.freedesktop.Flatpak=talk
```

**Reinicia OpenCode** después de aplicar el override: los permisos se leen al arrancar la app.
Cerrar la ventana **no** basta: la app de escritorio mantiene un servicio de fondo (`opencode-cli
serve --service`), y `flatpak run` solo abre otra ventana en el mismo sandbox. Mata la instancia
completa:

```bash
flatpak kill ai.opencode.opencode
flatpak run ai.opencode.opencode
```

### 0.1 Si `/mcp` sigue mostrando `ServiceUnknown` (solo variante stdio)

Este es el error que aparece cuando el permiso nunca llegó al sandbox en ejecución:

```
rea: Connection closed: MCP server process exited with code 1:
Portal call failed: org.freedesktop.DBus.Error.ServiceUnknown
Hint: --host only works when the Flatpak is allowed to talk to org.freedesktop.Flatpak
```

`ServiceUnknown` (y no `AccessDenied`) significa que el bus de sesión filtrado no expone
`org.freedesktop.Flatpak` **en absoluto**, así que `flatpak-spawn --host` no puede llegar al host.
Diagnostica en este orden:

```bash
# 1) Dentro de la terminal de OpenCode: la prueba definitiva. Debe imprimir "ok".
flatpak-spawn --host echo ok

# 2) En el host: ¿la política está guardada para el app ID y el ámbito correctos?
flatpak list --app --columns=application | grep -i opencode
flatpak override --user --show ai.opencode.opencode
cat ~/.local/share/flatpak/overrides/ai.opencode.opencode

# 3) Si falta el bloque [Session Bus Policy], aplícalo otra vez…
flatpak override --user --talk-name=org.freedesktop.Flatpak ai.opencode.opencode

# 4) …y fuerza un sandbox nuevo (obligatorio; ver arriba).
flatpak kill ai.opencode.opencode && flatpak run ai.opencode.opencode
```

Solo cuando el paso 1 imprima `ok` debes activar `rea`. Causas habituales, por frecuencia:

- el override se aplicó **después** de arrancar OpenCode y la app nunca se mató del todo;
- el override se aplicó a otro app ID (compruébalo con `flatpak list --app`);
- el override se aplicó como root sin `--user` (queda en `/var/lib/flatpak/overrides`), o hay una
  entrada vieja/errónea en un ámbito mientras revisas el otro.

Para empezar limpio, resetea y vuelve a aplicar:

```bash
flatpak override --user --reset ai.opencode.opencode
flatpak override --user --talk-name=org.freedesktop.Flatpak ai.opencode.opencode
flatpak kill ai.opencode.opencode
```

> ⚠️ Este permiso es potente: permite a OpenCode ejecutar **cualquier** comando en el host, como tu
> usuario. Por eso la ruta recomendada no lo usa. Lo detallamos en
> [Seguridad](#seguridad-qué-estás-confiando-y-por-qué).

---

## Paso 1 — Preparar el toolbox

**Dónde:** en el host, entrando al contenedor.

```bash
toolbox enter
```

### 1.1 Java

Ghidra 12.x exige Java 21 o superior. El Flatpak de Ghidra **no** trae JDK, así que Java en el
toolbox es lo que usa REA cuando lanza Ghidra en modo headless.

```bash
sudo dnf install -y java-25-openjdk java-25-openjdk-devel
java -version
```

Esperado: `openjdk version "25.x"`.

> Si algún día ves `JAVA_HOME` mal, exporta `JAVA_HOME=/usr/lib/jvm/java-25-openjdk` (o el que
> devuelva `alternatives --list | grep java`).

### 1.2 uv (para el bridge y mcp-proxy)

```bash
sudo dnf install -y uv
uv --version
```

### 1.3 Node y npm (para REA)

```bash
sudo dnf install -y nodejs npm
node --version
npm --version
```

**Ojo con la versión:** `rea-agents` declara como requisito `Node 22.x ≥ 22.23.4`, `24.x ≥ 24.22.1`
o `26.12.1+`. Fedora 43 trae `Node 24.14` / `22.23.1`, es decir, *por debajo* del mínimo.

En la práctica npm solo imprime un aviso `EBADENGINE` y REA funciona igual (así está probado).
Si prefieres cumplir el requisito al pie de la letra, instala Node con `fnm` o `nvm` en lugar del
paquete de la distro. Si `npm install` falla directamente con `EBADENGINE`, es esto.

### 1.4 mcp-proxy (para la Ruta A)

**Dónde:** dentro del toolbox.

Es el puente que convierte `rea mcp` (stdio) en un servidor MCP por HTTP, para que OpenCode se
conecte sin salir del sandbox.

```bash
toolbox enter
uv tool install mcp-proxy
~/.local/bin/mcp-proxy --version
```

Esperado: `mcp-proxy 0.15.0` (o superior). El binario queda en `~/.local/bin/mcp-proxy`, que
también existe en el host porque `$HOME` se comparte.

> `uvx mcp-proxy …` funciona sin instalar nada, pero resuelve y descarga el paquete en cada
> arranque. Para uso diario, `uv tool install` es más rápido y predecible.

### 1.5 Comprobar que el toolbox ve el Flatpak de Ghidra

```bash
ls /run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra
```

Debe listar `Ghidra`, `support`, `ghidraRun`, `Extensions`… Si falla, es que Ghidra aún no está
instalado (Paso 2) o que la ruta cambió de nombre.

---

## Paso 2 — Instalar Ghidra (Flatpak)

**Dónde:** en el host (fuera del toolbox).

```bash
flatpak install -y flathub org.ghidra_sre.Ghidra
```

Verificar:

```bash
flatpak list --app --columns=application,version | grep -i ghidra
# org.ghidra_sre.Ghidra   12.1.3

flatpak run org.ghidra_sre.Ghidra
```

La primera vez tarda: Ghidra indexa extensiones al arrancar.

**Rutas que hay que memorizar:**

| Qué | Ruta en el host | Ruta dentro del toolbox |
| --- | --- | --- |
| Instalación de Ghidra | `/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra` | `/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra` |
| Preferencias y extensiones de usuario | `~/.var/app/org.ghidra_sre.Ghidra/config/ghidra/` | igual (es tu `$HOME`) |

Con Flatpak, los directorios XDG del usuario están sandboxeados: por eso las extensiones caen en
`ghidra_12.1.3_FLATPAK/` y no en `.config/ghidra/`. Si no lo encuentras:

```bash
find ~ -maxdepth 8 -type d -path '*ghidra*/Extensions' 2>/dev/null
```

---

## Paso 3 — Instalar REA (ruta recomendada)

**Dónde:** dentro del toolbox.

```bash
toolbox enter

mkdir -p ~/.npm-global
npm config set prefix "$HOME/.npm-global"
npm install --global rea-agents

# Verificar (aquí NO hace falta Ghidra para el --version)
~/.npm-global/bin/rea --version
```

Esperado: `6.7.0` (o la versión actual).

`npm` deja los binarios en `~/.npm-global/bin`. Añádelo al PATH para no escribir la ruta completa:

```bash
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
which rea
```

> **Por qué un prefix propio:** en un sistema inmutable no quieres escribir en `/usr/bin`. Con
> `npm config set prefix` todo queda en tu `$HOME`, que se comparte con el host, y así OpenCode
> puede ejecutarlo desde fuera del contenedor.

Prueba de que REA encuentra Ghidra:

```bash
toolbox run env \
  GHIDRA_INSTALL_DIR=/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra \
  rea --version
```

Si eso funciona, el 90 % de la integración está hecho.

---

## Paso 4 — Extensión GhidraMCP dentro de Ghidra (opcional)

**Solo si vas a usar la Ruta B.** Si usas REA, puedes saltarte este paso y el Paso 5.

### 4.1 Descargar el zip

No lo descomprimas: Ghidra acepta el `.zip` tal cual.

```bash
wget https://github.com/bethington/ghidra-mcp/releases/download/v7.0.0/GhidraMCP-7.0.0.zip
```

### 4.2 Instalarlo

En la ventana de proyecto de Ghidra (la inicial, no el CodeBrowser):

1. **File → Install Extensions…**
2. Pulsa **+** y elige `GhidraMCP-7.0.0.zip`
3. Acepta y **reinicia Ghidra**

### 4.3 Activar el plugin (esto es lo que casi nadie hace)

Instalar la extensión **no** arranca nada. Hay que habilitar el plugin:

1. **File → Configure → Utility → Configure** (o **Configure All Plugins**)
2. Marca **GhidraMCPPlugin**
3. Reinicia Ghidra

El servidor HTTP arranca junto con el plugin, en **`127.0.0.1:8089`** (no 8080).

### 4.4 Verificar

Con Ghidra abierto:

```bash
curl -s http://127.0.0.1:8089/check_connection
```

Esperado:

```json
{"status":"ok","server_kind":"gui","version":"7.0.0","program":"tu_binario.exe"}
```

Si `program` no aparece, es que no hay ningún binario abierto en el CodeBrowser.

Otros dos endpoints útiles:

```bash
curl -s http://127.0.0.1:8089/mcp/health
curl -s http://127.0.0.1:8089/mcp/schema | jq '.tools | length'
```

### 4.5 Sobre versiones

La release 7.0.0 se compila contra **Ghidra 12.1.4** y el Flatpak de Flathub está en **12.1.3**.
En la práctica la extensión carga en 12.1.3, pero es lo primero que hay que mirar si un día
Ghidra deja de mostrarla tras una actualización: usa la release de GhidraMCP correspondiente a
tu versión de Ghidra, o instala Ghidra oficial (no Flatpak) de la versión exacta.

---

## Paso 5 — Bridge MCP en Python (opcional)

**Solo para la Ruta B.** Si usas REA, sáltalo.

**Dónde:** dentro del toolbox (la instalación). El proceso puede correr en el toolbox o en el
host, porque `$HOME` es el mismo.

```bash
toolbox enter
uv tool install https://github.com/bethington/ghidra-mcp/releases/download/v7.0.0/ghidra_mcp_bridge-7.0.0-py3-none-any.whl
```

`uv tool install` crea un entorno aislado en `~/.local/share/uv/tools/` y deja el ejecutable en
`~/.local/bin/bridge-mcp-ghidra`. Verifícalo:

```bash
ls -l ~/.local/bin/bridge-mcp-ghidra
```

Arrancarlo (en el toolbox o en el host, da igual):

```bash
bridge-mcp-ghidra --transport streamable-http --mcp-host 127.0.0.1 --mcp-port 8081
```

Deja ese proceso vivo en una terminal aparte. Comprobación:

```bash
# 8081: puerto MCP (POST). Un GET devuelve 400: eso es normal, significa que el server está.
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8081/mcp
# 8089: Ghidra (GET con JSON)
curl -s http://127.0.0.1:8089/check_connection
```

Notas útiles del bridge:

- `GHIDRA_MCP_URL` (por defecto `http://127.0.0.1:8089`) apunta a Ghidra.
- Lazy loading: por defecto solo expone los grupos `listing,function,program`. Si tu cliente no
  procesa notificaciones `tools/list_changed`, arranca con `--no-lazy` para ver las ~209
  herramientas desde el principio (más contexto, más capacidades).
- `GHIDRA_MCP_REQUIRE_PROGRAM_SELECTORS=1` evita que una llamada sin `program=` acabe escribiendo
  en el binario equivocado cuando tienes varios abiertos.

---

## Paso 6 — Configurar OpenCode

**Dónde:** en el host (puedes escribir el archivo desde cualquier sitio; es una ruta del `$HOME`).

```bash
mkdir -p ~/.var/app/ai.opencode.opencode/config/opencode
touch ~/.var/app/ai.opencode.opencode/config/opencode/opencode.json
```

Si OpenCode **no** fuera Flatpak, la ruta sería `~/.config/opencode/opencode.json`.

### 6.1 Configuración verificada (Ruta A: REA por HTTP)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "rea": {
      "type": "remote",
      "url": "http://127.0.0.1:8082/mcp",
      "oauth": false,
      "enabled": true
    }
  }
}
```

Eso es todo: ni `flatpak-spawn`, ni rutas absolutas, ni permisos de Flatpak. El puerto `8082` lo
sirve `mcp-proxy` (Paso 1.4), que hay que tener arrancado:

```bash
# Desde el host. Si ya estás dentro del toolbox, quita el prefijo `toolbox run`.
toolbox run ~/.local/bin/mcp-proxy --stateless --port 8082 --host 127.0.0.1 \
  -e GHIDRA_INSTALL_DIR /run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra \
  -e REA_GHIDRA_STARTUP_TIMEOUT_MS 60000 \
  -- "$HOME/.npm-global/bin/rea" mcp
```

Al arrancar imprime las dos rutas que expone (`/sse` y `/mcp`). Detalles que importan:

- `--stateless` evita depender del header `Mcp-Session-Id`; es la opción más compatible con
  clientes MCP. Quítalo solo si necesitas sesiones de transporte.
- `--host 127.0.0.1` lo deja en loopback: no lo expongas a la red.
- Lo que va **después de `--`** es el comando que el proxy lanza por stdio.
- Cambia la ruta de `rea` si `which rea` no da `~/.npm-global/bin/rea`.

### 6.2 Variante stdio (Ruta A sin proxy; requiere el Paso 0)

Si prefieres que OpenCode arranque el servidor él mismo, aplica el permiso de host del Paso 0 y usa:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "rea": {
      "type": "local",
      "command": [
        "flatpak-spawn",
        "--host",
        "toolbox",
        "run",
        "env",
        "REA_GHIDRA_STARTUP_TIMEOUT_MS=60000",
        "GHIDRA_INSTALL_DIR=/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra",
        "/var/home/vm/.npm-global/bin/rea",
        "mcp"
      ],
      "enabled": true
    }
  }
}
```

Cambia **solo** dos cosas si tu usuario no es `vm`:

```bash
echo $HOME          # /var/home/vm
which rea           # dentro del toolbox
```

### 6.3 Variante con la Ruta B (bridge)

Añade este bloque dentro de `"mcp"`, y **pon `enabled: false` en `rea`** para no duplicar
herramientas:

```json
"ghidra-mcp": {
  "type": "remote",
  "url": "http://127.0.0.1:8081/mcp",
  "oauth": false,
  "enabled": true
}
```

### 6.4 Por qué esa línea de comando (variante stdio)

Se lee de dentro hacia fuera:

```
flatpak-spawn --host        → salta del sandbox del Flatpak al host
toolbox run                 → entra en el contenedor
env VAR=...                 → define el entorno del proceso
/…/rea mcp                  → arranca el servidor MCP por stdio
```

- `GHIDRA_INSTALL_DIR` usa la ruta **vista desde el toolbox** (`/run/host/...`), no la del host.
- `REA_GHIDRA_STARTUP_TIMEOUT_MS=60000` da un minuto a Ghidra headless para arrancar; con
  binarios grandes el valor por defecto se queda corto.

> Alternativa más limpia (equivale a lo mismo, sin `env` en el array):
> ```json
> "command": ["flatpak-spawn", "--host", "toolbox", "run", "{env:HOME}/.npm-global/bin/rea", "mcp"],
> "environment": {
>   "GHIDRA_INSTALL_DIR": "/run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra",
>   "REA_GHIDRA_STARTUP_TIMEOUT_MS": "60000"
> }
> ```
> Si `{env:HOME}` no se expande en tu versión, vuelve a la ruta absoluta.

### 6.5 Reiniciar

Cierra y abre OpenCode. En una sesión nueva:

```
/mcp
```

Debe listar `rea` como conectado y sus herramientas (del orden de 200, con nombres `rea_*`).

---

## Paso 7 — Verificación completa

Hazlo en este orden; el primer fallo te dice exactamente qué capa está rota.

| # | Comando | Dónde | Resultado esperado |
| --- | --- | --- | --- |
| 1 | `ss -ltnp \| grep 8082` | host | `LISTEN 127.0.0.1:8082` |
| 2 | `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8082/mcp` | host o sandbox | `406` (GET sin `Accept: text/event-stream`; MCP usa POST) |
| 3 | `toolbox run java -version` | host | `openjdk 25…` |
| 4 | `toolbox run ls /run/host/var/lib/flatpak/app/org.ghidra_sre.Ghidra/current/active/files/lib/ghidra` | host | `Ghidra support ghidraRun …` |
| 5 | `toolbox run ~/.npm-global/bin/rea --version` | host | `6.7.0` |
| 6 | `curl -s http://127.0.0.1:8089/check_connection` | host | `{"status":"ok",…}` *(solo Ruta B)* |
| 7 | `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8081/mcp` | host | `400` *(solo Ruta B)* |
| 8 | Habilidad completa de OpenCode | sandbox | `ok` |
| 9 | `/mcp` en OpenCode | OpenCode | `rea` conectado |

> Si elegiste la **variante stdio** (§6.2), añade los dos chequeos de host: `flatpak override
> --user --show ai.opencode.opencode` debe mostrar `org.freedesktop.Flatpak=talk`, y
> `flatpak-spawn --host echo ok` debe imprimir `ok` dentro de OpenCode.

Prueba de fuego (dentro de OpenCode, con un binario ya analizado):

```
Usa rea para listar las funciones del binario y descompilar la función main.
```

Si devuelve pseudocódigo real, está todo funcionando.

---

## Paso 8 — Abrir el binario a analizar

### Con REA (Ruta A)

No hace falta que abras nada: REA lanza Ghidra headless y analiza el archivo. Solo dile la ruta.

### Con GhidraMCP (Ruta B)

1. Abre Ghidra → **File → New Project** (no versionado).
2. **File → Import File…** y elige el `.exe` / ELF. Acepta los defaults del importador.
3. En el proyecto, doble clic (o clic derecho → **Open in CodeBrowser**).
4. Cuando pregunte, acepta el análisis. Si no, **Analysis → Auto Analyze** y espera.
5. `curl -s http://127.0.0.1:8089/check_connection` debe mostrar `"program":"tu_binario"`.

> **Auto Analyze no es opcional.** Sin análisis, Ghidra no tiene funciones, ni xrefs, ni
> decompilación: el agente recibirá errores 500 o listas vacías.

---

## Uso diario

1. Terminal 1 (Ruta A): el `mcp-proxy` del §6.1. Si usas Ruta B: `bridge-mcp-ghidra --transport streamable-http --mcp-host 127.0.0.1 --mcp-port 8081`
2. Ghidra abierto con el binario analizado (solo Ruta B).
3. OpenCode abierto `/mcp` → `rea` (o `ghidra-mcp`) conectado.
4. Pídele cosas concretas y verificables:

```
# Buenos prompts
"Lista las funciones de otomedius.exe y dime cuáles no tienen nombre."
"Descompila la función que referencia el string 'license'."
"Renombra esa función a CheckLicense y documenta sus parámetros."
"¿Qué imports de kernel32 usa? Marca los sospechosos."

# Malos prompts
"Analiza el binario."   ← demasiado abierto, gasta contexto y no verifica nada
```

Consejo: pide **evidencia** (dirección, nombre, pseudocódigo) y ve de una función a la vez.
Así aprendes y detectas alucinaciones.

---

## Problemas frecuentes

| Síntoma | Causa | Solución |
| --- | --- | --- |
| `flatpak: orden no encontrada` | Estás en el sandbox de OpenCode | `flatpak-spawn --host flatpak …` |
| `/mcp` muestra `rea` pero las llamadas fallan por Ghidra | `GHIDRA_INSTALL_DIR` incorrecto o Ghidra roto | Prueba el paso 5 de la tabla de verificación |
| `spawn rea ENOENT` | Ruta mal escrita o REA no instalado en ese prefix | `ls ~/.npm-global/bin/rea` |
| REA falla con errores de Java | Falta JDK en el toolbox | `sudo dnf install java-25-openjdk` |
| `8089 connection refused` | Ghidra cerrado, plugin desactivado o servidor parado | File → Configure → Utility → Configure → GhidraMCPPlugin; Tools → GhidraMCP → Server Status |
| Menú **Tools → GhidraMCP** no aparece | Extensión instalada pero plugin sin activar, o versión incompatible | Reinicia Ghidra; revisa compatibilidad 12.1.3 vs 12.1.4 |
| `curl 8089` responde sin `program` | No hay binario abierto en el CodeBrowser | Abre el programa |
| Bridge arranca pero OpenCode no lo ve | `enabled: false` en el JSON, o bridge caído | `ss -ltnp \| grep 8081` |
| `8082 connection refused` | `mcp-proxy` no está arrancado o murió | Arráncalo otra vez (§6.1) y comprueba `ss -ltnp \| grep 8082` |
| `GET http://127.0.0.1:8082/mcp` = `406` | Es correcto: el endpoint espera POST/streaming | Ignóralo |
| `address already in use` al arrancar el proxy | Otro proceso ocupa 8082 | `ss -ltnp \| grep 8082`, ciérralo o cambia de puerto (y el JSON) |
| `rea` conectado pero con 0 herramientas | El proxy lanzó otro comando o `rea` no está en esa ruta | Prueba a mano `toolbox run ~/.npm-global/bin/rea mcp` y revisa lo que va tras `--` |
| `GET http://127.0.0.1:8081/mcp` = `400` | Es correcto: MCP usa POST | Ignóralo |
| El agente mezcla respuestas de dos servidores | `rea` y `ghidra-mcp` activos a la vez | Deja solo uno `enabled: true` |
| `EBADENGINE` al instalar rea-agents | Node de Fedora por debajo del mínimo declarado | Suele funcionar igual; si no, instala Node con `fnm`/`nvm` |
| Ghidra tarda muchísimo en el primer análisis | Normal (análisis completo de PE grande) | Sube `REA_GHIDRA_STARTUP_TIMEOUT_MS` |
| `rea: … Portal call failed: …ServiceUnknown` / pista `--host only works when the Flatpak is allowed to talk to org.freedesktop.Flatpak` *(solo variante stdio)* | El permiso `org.freedesktop.Flatpak` no está en efecto en el sandbox en ejecución: override ausente, app ID/ámbito equivocado, o la app nunca se reinició del todo | Aplica el override (Paso 0), luego `flatpak kill ai.opencode.opencode` y vuelve a abrir; verifica dentro de OpenCode con `flatpak-spawn --host echo ok` |
| `flatpak-spawn --host` funciona pero `toolbox run` falla | El permiso está bien; el problema está en la capa host/toolbox | Ejecuta los pasos 3 y 5 de la tabla de verificación en el host |
| Todo funciona tras reiniciar, pero no antes | Override de Flatpak aplicado sin reiniciar OpenCode | `flatpak kill ai.opencode.opencode` (una ventana nueva reutiliza el sandbox viejo) |

Diagnóstico rápido de puertos:

```bash
ss -ltnp | grep -E '8081|8082|8089'
```

---

## Seguridad: qué estás confiando y por qué

La **Ruta A de esta guía (REA por HTTP) no le da a OpenCode ningún permiso extra sobre el host**: el
sandbox solo abre un socket a `127.0.0.1:8082`, y el Flatpak ya comparte la red. Si quieres
mantener el aislamiento, no apliques el `override` del Paso 0 y quédate en la Ruta A o en la B.

Aun así, hay cosas que sí estás confiando:

1. **Los servidores MCP escuchan sin autenticación en loopback** (`mcp-proxy` en 8082, el bridge en
   8081, GhidraMCP en 8089). Cualquier proceso local —tuyo o no— puede hablar con ellos y, a
   través de REA, abrir archivos y lanzar Ghidra headless. Es loopback, no internet, pero no es una
   frontera de seguridad. Si alguna vez expones algo fuera de loopback, define
   `GHIDRA_MCP_AUTH_TOKEN` (GhidraMCP se niega a arrancar en una IP no-loopback sin token) o pon
   `--auth-bearer-token` en `mcp-proxy`.
2. **El Flatpak de OpenCode no es un sandbox fuerte**: trae `filesystems=home` (lee y escribe todo
   tu `$HOME`) y `shared=network`. Aunque no tenga `talk`, cualquier código que corra dentro puede
   tocar tus archivos y, por ejemplo, dejar algo en `~/.bashrc`. El aislamiento que ganas al evitar
   `talk` es real, pero parcial.
3. **Ghidra y REA procesan binarios hostiles.** Un parser con un bug, una extensión de Ghidra
   maliciosa o un binario preparado son superficie de ataque real. Y ojo con la **inyección de
   prompts**: los strings, nombres de símbolos y comentarios que REA devuelve al modelo provienen
   del binario analizado. Un binario puede contener texto que intente convencer al agente de
   ejecutar algo. Trátalo como datos, nunca como instrucciones; REA ya marca esas respuestas con
   `untrusted_content`.
4. **Cadena de suministro**: `rea-agents`, `mcp-proxy`, el wheel del bridge y las extensiones se
   instalan desde la red y corren con tu usuario.

Además, **no actives** `GHIDRA_MCP_ALLOW_SCRIPTS=1` salvo que lo necesites: habilita ejecución de
código Java arbitrario dentro de Ghidra.

### Si aun así eliges la variante stdio

**`org.freedesktop.Flatpak=talk` permite a OpenCode ejecutar cualquier comando en el host**, como
tu usuario. Es una fuga del sandbox por diseño: `flatpak-spawn --host` llama a `HostCommand` del
`flatpak-session-helper`, que no filtra por comando ni pide confirmación. Deja de ser "una app
encerrada" y pasa a ser "una app con shell en tu máquina": tus claves SSH, tu sesión de escritorio,
`systemctl --user`, todo lo que tu usuario puede leer y ejecutar. No da root por sí solo (`sudo`
pide contraseña), pero no lo necesitas para hacer daño. Y lo hereda cualquier cosa que corra en el
sandbox: el agente, un plugin, un paquete npm comprometido.

Si trabajas en una **VM** (GNOME Boxes, VirtualBox…), el radio de daño se queda en el invitado: es
la mitigación más importante. Aun así, mantén apagadas las carpetas compartidas, el portapapeles
compartido y el drag-and-drop de SPICE, haz snapshots antes de analizar algo sospechoso y no metas
credenciales reales en esa VM.

Y recuerda lo obvio: analiza solo binarios que tengas derecho a analizar.

---

## Mantenimiento: actualizar y desinstalar

### Actualizar

```bash
# Host: Ghidra y OpenCode
flatpak update org.ghidra_sre.Ghidra ai.opencode.opencode

# Toolbox: REA
toolbox run ~/.npm-global/bin/rea update

# Toolbox: bridge
toolbox run uv tool upgrade ghidra-mcp-bridge

# Toolbox: mcp-proxy
toolbox run uv tool upgrade mcp-proxy
```

Al actualizar Ghidra, **reinstala la extensión GhidraMCP** de la release correspondiente: las
extensiones compiladas para otra versión no cargan.

### Desinstalar

```bash
# REA
toolbox run npm uninstall --global rea-agents
rm -rf ~/.npm-global

# Bridge
toolbox run uv tool uninstall ghidra-mcp-bridge

# mcp-proxy
toolbox run uv tool uninstall mcp-proxy

# Extension: File → Install Extensions en Ghidra (o borra la carpeta en Extensions/)

# Permiso del Flatpak (solo si usaste la variante stdio)
flatpak override --user --reset ai.opencode.opencode

# Ghidra
flatpak uninstall org.ghidra_sre.Ghidra
```

---

## Glosario

- **MCP (Model Context Protocol):** protocolo con el que un agente descubre y llama herramientas
  de un servidor local. "Servidor MCP" = proceso que ofrece herramientas; "cliente" = el agente.
- **stdio vs HTTP:** el servidor puede hablar por entrada/salida estándar (lo lanza el cliente) o
  por HTTP (el cliente se conecta a una URL, caso del bridge).
- **Proxy MCP:** proceso que traduce entre transportes. Aquí `mcp-proxy` convierte el `rea mcp` de
  stdio en un servidor HTTP, para que el sandbox lo consuma sin permisos de host.
- **Headless:** Ghidra sin interfaz gráfica, controlado por línea de comandos. Ideal para agentes.
- **Auto Analyze:** el pase de análisis que descubre funciones, crea referencias y habilita el
  descompilador.
- **Pseudocódigo / decompilación:** C aproximado que Ghidra reconstruye a partir del assembly.
- **Xref:** referencia cruzada; quién llama a quién o quién lee una dirección.
- **Extensión vs plugin (Ghidra):** la extensión es el paquete (`.zip`, `.jar`); el plugin es el
  módulo que se habilita dentro de Ghidra. Instalar ≠ activar.
- **Flatpak / toolbox / sandbox:** ver [sección 3](#3-las-tres-capas-host-toolbox-y-sandbox).

---

**Siguiente paso sugerido para aprender:** abre un binario pequeño y conocido, pide a REA que
descompile `main`, y compara el pseudocódigo con el assembly. El hábito de contrastar ambos es lo
que convierte esto en una herramienta de estudio y no en una caja de respuestas.