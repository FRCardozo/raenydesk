# RaenyDesk — Arquitectura (Fase 2)

Documento de diseño **propio**. No describe la implementación interna de ninguna app de terceros:
define cómo se construiría un prototipo educativo de acceso remoto desde cero.

## 1. Stack y justificación de cada elección

| Componente | Elección | Por qué |
|---|---|---|
| Lenguaje | **Kotlin** | Idioma oficial de Android, conciso, null-safety, coroutines/flows nativos para eventos de red y UI reactiva. |
| UI | **Jetpack Compose + Material 3** | UI declarativa = menos código de vistas, preview rápido del prototipo. M3 da theming moderno sin assets propietarios. |
| Arquitectura | **MVVM + Clean Architecture** (presentation → domain → data) | Separación de responsabilidades: el dominio (casos de uso, entidades, repositorios como interfaces) es puro Kotlin y testeable; los detalles (WebRTC, DataStore, señalización) quedan en `data`/`core` y son intercambiables. |
| Navegación | **Navigation Compose** | Gráfica de navegación tipada por rutas (`home`, `connect`, `session/{remoteId}`, `settings`), deep links y back-stack manejados por el sistema. |
| Red/video | **WebRTC** (`org.webrtc:google-webrtc` o fork `libwebrtc`) | Estándar abierto, P2P real con ICE/STUN/TURN, codecs H264/VP8 ya integrados, licencia BSD — ideal para prototipar escritorio remoto sin servidores propios. Requiere un servidor de **señalización** (WebSocket simple) para intercambio de SDP/candidatos. |
| Persistencia | **DataStore (Preferences)** | Config ligera (ID local, nombre de dispositivo, contraseña simulada, calidad por defecto) con API basada en Flow, sin ORM ni migraciones. |
| Inyección | **Hilt** (opcional en MVP: manual DI con un `AppContainer`) | Menos magia para un prototipo; se puede migrar a Hilt después sin tocar el dominio. |

## 2. Estructura de carpetas

```
app/src/main/java/com/raenydesk/
├── MainActivity.kt                  #宿主 Compose + NavHost
├── ui/
│   ├── theme/                       # Material 3: colores, tipos, forma (branding RaenyDesk)
│   ├── home/                        # HomeScreen + HomeViewModel (ID 9 dígitos, estado En espera/Conectado)
│   ├── connect/                     # ConnectScreen + ConnectViewModel (ID remoto, contraseña)
│   ├── session/                     # SessionScreen + SessionViewModel (superficie remota, mouse, teclado, chat, calidad, desconectar)
│   └── settings/                    # SettingsScreen + SettingsViewModel (contraseña, nombre dispositivo, calidad default)
├── domain/
│   ├── model/                       # DeviceId, ConnectionState, QualityLevel, ChatMessage, RemoteContact
│   ├── repository/                  # Interfaces: SettingsRepository, SessionRepository, ChatRepository, ContactsRepository
│   └── usecase/                     # GenerateLocalDeviceId, ConnectToPeer, DisconnectSession, SetQuality, SendMessage, UpdateSettings…
├── data/
│   ├── repository/                  # Implementaciones (DataStore, sesión simulada/WebRTC, contactos locales)
│   └── datastore/                   # DataStoreSchema / extensiones
└── core/
    ├── network/                     # Cliente de señalización (WebSocket) + DTOs SDP/ICE/chat
    └── webrtc/                      # PeerConnectionManager: wrappers de VideoSource, DataChannel, SurfaceTexture
```

**Regla de dependencias:** `ui → domain ← data/core`. El dominio no conoce Compose ni WebRTC.

## 3. Flujo de datos (texto)

```
UI (Composable) --event--> ViewModel --invoca--> UseCase (domain)
                                |                      |
                          StateFlow<UiState> <---- Repository (interface)
                                                         |
                                            impl: DataStore / WebSocket / WebRTC
```

- Estado de sesión: máquina de estados `Idle → Connecting → Auth → Connected → Closed` expuesta como `StateFlow`.
- Chat: sobre **DataChannel** de WebRTC (mensajería bidireccional sin servidor); en el MVP local se simula con un canal in-memory.
- Calidad (Baja/Media/Alta): mapea a parámetros de encoding (resolución/bitrate/FPS) vía `RtpParameters` o `setParameters` del sender.

## 4. Decisiones del MVP (Fase 3)

1. Todo funciona **en local/simulado**: ID generado con `SecureRandom`, "conexión" simulada con delays, chat en memoria — sin servidor real, cumpliendo el objetivo educativo.
2. La capa `core/webrtc` y `core/network` se deja preparada (interfaces + stubs) para enchufar WebRTC real en la siguiente iteración.
3. Branding propio: paleta violeta/teal, nombre RaenyDesk, cero assets externos.
