# ARCHITECTURE.md — llm-status

Menú de barra nativo para macOS (AppKit, un solo fichero `main.swift`) que muestra las peticiones LLM en curso y los tok/s del clúster DGX. Repo público, licencia MIT. Tronco: **`main`**.

## Clientes y versiones
- Un único cliente: la app de barra de menús, macOS 13+ (`LSMinimumSystemVersion` 13.0, `LSUIElement`, sin icono en el Dock), versión 1.0, bundle id `com.e-dani.llm-status`.
- Es **cliente del panel** de `pocharlies-org/dgx-infra` (Control Nexus). Mismo producto y mismo servidor que `pocharlies/llm-status-ios` (app DGX para iPhone, iPad, Watch y Mac): un cambio del dato que ambos leen se planifica en dgx-infra y en los dos clientes.
- **Solape**: `llm-status-ios` incluye un target Mac nativo (`LLMStatusMac`) con el MISMO bundle id `com.e-dani.llm-status`. Ver Decisiones.

## Dependencias (ambos sentidos)
- **Del servidor** (`pocharlies-org/dgx-infra`, base por defecto `https://dgx.lan.e-dani.com`, sin SSO, solo LAN/tailnet, certificado de confianza del sistema): `/api/activity/stream?v=2` (SSE: vLLM cada ~1 s, actividad cada 15 s, colas de estudio cada 2 s), `/api/llm/live` (contrato `dgx.llm.live.v1`, respaldo del chip), `/api/llm/company` y `/api/llm/sessions` (proyecciones propias del dashboard), `/api/activity?sections=…`, `/api/compute/mode`, `/api/service-health`, `/api/image/queue`. El registro de contratos es `dgx-infra/CONTRACTS.yaml`; la entrada de contadores de compañía cita a este repo como consumidor. No se pueden renombrar esas rutas sin avisar aquí.
- **Quién depende de este repo**: nadie en código; `dgx-infra` lo cita en contratos y comentarios (`services/dashboard/api/routes_litellm.py`).
- Sin dependencias de terceros (solo `swiftc` de las Command Line Tools).
- Variables: `LLM_LIVE_URL`, `LLM_DASHBOARD_URL` (ver `README.md`); la base de las demás rutas es `https://dgx.lan.e-dani.com` por defecto.

## Stack
Swift con AppKit, un solo fichero (`main.swift`), compilado con `swiftc -O` desde `build.sh`. No hay SwiftUI, ni Xcode project, ni paquetes. Sondeo cada 30 s (una petición) más el stream SSE.

## Componentes compartidos (canónicos)
No hay componentes compartidos con otros repos: toda la lógica vive en `main.swift`. El modelo de datos del servidor se interpreta a mano (JSON suelto), sin cliente generado. Lo canónico para esto, si se necesita, es `Packages/DGXKit` de `llm-status-ios` (cliente, tema, componentes): no se copia código de aquí a allí ni al revés.

## Cómo se construye aquí
- `./build.sh` produce `build/LLM Status.app` (firma ad hoc `codesign -s -`) y con `--install` lo copia a `~/Applications` e instala el LaunchAgent `com.e-dani.llm-status` (`RunAtLoad`, `KeepAlive`).
- Colores iguales a los chips del dashboard (verde sirviendo, naranja con cola, gris ocioso o sin lectura); nunca rojo.
- Cambios de endpoint: solo aditivos; si falta una clave, se pinta el estado «sin lectura», no se falla.

## Tests y validaciones
No hay tests ni CI en el repo (hueco conocido). Validación: `./build.sh`, arrancar la app y comparar el chip con `https://dgx.lan.e-dani.com/inferencia`.

## CI/CD y despliegue
No hay CI ni despliegue automático. Se compila e instala a mano en el Mac con `./build.sh --install`. No pasa por ArgoCD ni por el runner `nexus-mac`.

## Decisiones y trampas
- **Duplicado propuesto (no se archiva en SC-1429):** `LLMStatusMac` de `llm-status-ios` cubre lo mismo (y más) con el mismo bundle id; instalar las dos en el mismo Mac comparte identificador y LaunchAgent/entradas de login. Decisión pendiente del architect: retirar este repo (archivarlo) o declararlo legado.
- Es público: no meter URLs internas nuevas, claves ni nombres de hosts privados más allá de los ya presentes (`dgx.lan.e-dani.com` por defecto).
- Los defaults apuntan a la LAN de Dani; fuera de la tailnet muestra «sin lectura».
