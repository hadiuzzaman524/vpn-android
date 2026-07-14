# From Tap-to-Connect to WireGuard Handshake: Building a Production VPN Flow on Android

*By Md Hadiuzzaman · Software Engineer · Android | iOS | TV App Development*

Most VPN tutorials stop at "call `VpnService` and you're done." Real VPN apps are nothing like that. Between the moment a user taps the connect button and the moment the first encrypted packet flows, there is a **permission dance**, a **foreground service**, a **config fetch from your backend**, a **userspace tunnel**, a **handshake watchdog**, and a **state machine** that has to survive process death, system revokes, and Android's increasingly hostile background-execution rules.

This post walks through the complete connect/disconnect sequence of **AnyVPN**, a production WireGuard-based Android VPN built with Jetpack Compose and clean architecture. Every snippet is real code from the app, and the complete interactive connection-flow diagram is available in the docs: **[VPN Connection Flow — Sequence Diagram](https://hadiuzzaman524.github.io/vpn-android/index.html)**. The flow follows this sequence:

```
┌──────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐   ┌──────────────────────┐   ┌────────────────────┐
│    UI    │   │     VM      │   │     VC      │   │         FS          │   │          CM          │   │         WG         │
│  Home ·  │──▶│VPNViewModel │──▶│VpnController│──▶│VpnForegroundService │──▶│ VpnConnectionManager │──▶│ WireGuard GoBackend│
│ Compose  │   └─────────────┘   └─────────────┘   └─────────────────────┘   └──────────────────────┘   └────────────────────┘
└──────────┘         ▲                                                                  │
     ▲               └────────────── State / Traffic emitted via StateFlow ◀────────────┘
     └────────────────────── UI recomposes · speed, map, timer
```

Six actors, four phases:

1. **User taps to connect** — UI → ViewModel, plus the VPN consent dialog if needed
2. **Start the foreground service** — ViewModel → Controller → Service
3. **Fetch config & open tunnel** — Service fetches a WireGuard config from the backend
4. **Build the network tunnel** — Manager brings the tunnel up and confirms a real handshake

Then everything flows *back up* reactively: `CONNECTED` state, live throughput, and the session timer stream to the notification and the Compose UI through a single `StateFlow`.

---

## The architecture in one paragraph

The design rule that makes everything else fall into place: **exactly one component owns the tunnel**. `VpnConnectionManager` is an app-singleton that owns the WireGuard backend and publishes two `StateFlow`s — `state` (connection status) and `traffic` (live throughput). Everyone else either *drives* it (the foreground service) or *observes* it (the Compose UI, the notification). The ViewModel never touches the tunnel; the service never touches the UI; the UI never touches Android services. Messages travel down the chain as commands and travel back up as state emissions.

```kotlin
@Single
class VpnConnectionManager(
    private val context: Context,
    private val stateStore: VpnStateStore,
) {
    private val backend: Backend by lazy { GoBackend(context) }
    private val tunnel = AppTunnel(TUNNEL_NAME)

    private val _state = MutableStateFlow(VpnConnectionState())
    val state: StateFlow<VpnConnectionState> = _state.asStateFlow()

    private val _traffic = MutableStateFlow(VpnTraffic())
    val traffic: StateFlow<VpnTraffic> = _traffic.asStateFlow()
    ...
}
```

The state model is deliberately small. WireGuard's `GoBackend` only knows `UP`/`DOWN`, so the app layers its own semantics on top:

```kotlin
data class VpnConnectionState(
    val status: ConnectionStatus = ConnectionStatus.DISCONNECTED, // CONNECTING / CONNECTED / DISCONNECTED
    val error: VpnError? = null,          // one-shot, consumed by the ViewModel
    val connectedSinceEpochMs: Long? = null, // set on real handshake; drives the session timer
    val serverName: String? = null,
    val serverIp: String? = null,
    val serverCountryName: String? = null,
    val serverCountryCode: String? = null,
)
```

Note what's *not* there: no `FAILED` status. Failures are carried in `error` while `status` falls back to `DISCONNECTED` — the UI reverts the connect button and shows the error as a one-shot snackbar rather than getting stuck in a distinct error state.

---

## Step 1 — User taps to connect (and the permission alt-path)

The tap calls `VPNViewModel.connect()`. Before anything can happen, Android requires **user consent** for any app that wants to route the device's traffic. `VpnService.prepare()` is the check: it returns `null` if consent was already granted, or an `Intent` that must be launched *from an Activity* to show the system consent dialog.

The ViewModel can't launch activities, so this is modeled as a one-shot event:

```kotlin
/** One-shot effects the Home screen must run against an Activity/UI. */
sealed interface HomeEvent {
    /** The VPN consent dialog must be launched from the Activity via this intent. */
    data class RequestVpnPermission(val intent: Intent) : HomeEvent
    data class ShowError(@StringRes val messageRes: Int) : HomeEvent
}

/** Triggered by tap-to-connect. Requires VPN consent (once), then starts the service. */
fun connect() {
    val consentIntent = vpnController.prepare()
    if (consentIntent != null) {
        _events.tryEmit(HomeEvent.RequestVpnPermission(consentIntent))
    } else {
        vpnManager.markConnecting()
        vpnController.connect()
    }
}

/** Called by the screen after the system VPN-consent dialog resolves. */
fun onPermissionResult(granted: Boolean) {
    if (granted) {
        vpnManager.markConnecting()
        vpnController.connect()
    } else {
        _events.tryEmit(HomeEvent.ShowError(VpnError.NOT_AUTHORIZED.messageRes))
    }
}
```

On the Compose side, the consent dialog is wired with an `ActivityResult` launcher — the sequence diagram's `[ VPN permission not yet granted ]` alt-block:

```kotlin
val consentLauncher = rememberLauncherForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    viewModel.onPermissionResult(result.resultCode == Activity.RESULT_OK)
}

LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        when (event) {
            is HomeEvent.RequestVpnPermission -> consentLauncher.launch(event.intent)
            is HomeEvent.ShowError ->
                snackbarHostState.showSnackbar(context.getString(event.messageRes))
        }
    }
}
```

**Why `markConnecting()` before anything else?** The config fetch takes a network round-trip. Without an optimistic `CONNECTING` emission, the button would sit idle for a second or two after the user's tap — the classic "did my tap register?" moment. Setting state first makes the UI respond instantly while the real work happens behind it:

```kotlin
/** Optimistic CONNECTING state used before the config fetch resolves. */
fun markConnecting() {
    _traffic.value = VpnTraffic()
    _state.update {
        it.copy(status = ConnectionStatus.CONNECTING, error = null, connectedSinceEpochMs = null)
    }
}
```

---

## Step 2 — Start the foreground service

The ViewModel never touches `Context` or service intents. A thin bridge does:

```kotlin
/**
 * Thin Android-glue bridge between the ViewModel and VpnForegroundService, so
 * the ViewModel never touches Context or service intents directly.
 */
@Single
class VpnController(private val context: Context) {

    fun prepare(): Intent? = VpnService.prepare(context)

    fun connect() {
        ContextCompat.startForegroundService(context, VpnForegroundService.connectIntent(context))
    }

    fun disconnect() {
        // startForegroundService (not startService): a plain startService throws
        // IllegalStateException on API 26+ if the app has slipped into the background.
        ContextCompat.startForegroundService(context, VpnForegroundService.disconnectIntent(context))
    }
}
```

That comment on `disconnect()` is a scar from real debugging: on API 26+ you cannot `startService()` from the background, and a disconnect tap from the notification arrives while the app *is* in the background.

**Why a foreground service at all?** Two reasons:

1. WireGuard's `GoBackend` runs the tunnel **in your process** (its own internal `GoBackend$VpnService`). If Android kills your process, the tunnel dies with it. A foreground service keeps the process alive.
2. The user needs a persistent, glanceable status — "connected via Netherlands, 42 Mbps down" — and a disconnect button that works without opening the app.

Crucially, this service is a plain `Service`, **not** a `VpnService`:

```kotlin
/**
 * Foreground orchestrator that keeps the process (and therefore WireGuard's own
 * in-process GoBackend$VpnService) alive in the background, owns the persistent
 * status notification, and drives connect/disconnect. It is a plain Service,
 * NOT a android.net.VpnService — the tunnel interface is owned by GoBackend.
 */
class VpnForegroundService : Service() {

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        ensureChannels()

        // Seed CONNECTING before observing so the observer never stops us on the
        // initial DISCONNECTED snapshot (which would abort the connect).
        if (intent?.action == ACTION_CONNECT) manager.markConnecting()

        // Must call startForeground synchronously and early to avoid an ANR/crash.
        startInForeground(buildStatusNotification(manager.state.value, manager.traffic.value))

        when (intent?.action) {
            ACTION_CONNECT -> {
                startObserving()
                scope.launch { connectAndScheduleLimit(freshSession = true) }
            }
            ACTION_DISCONNECT -> {
                startObserving()
                handleDisconnect()
            }
            // Null intent: the system restarted us (START_STICKY) after killing the
            // process. Reconnect if that's what the user last asked for.
            else -> handleRestart()
        }
        return START_STICKY
    }
}
```

Three production details worth stealing:

- **`startForeground()` synchronously in `onStartCommand`.** Android gives you ~5 seconds after `startForegroundService()` to call `startForeground()`, or it crashes the app. Don't put a coroutine between them.
- **Seed `CONNECTING` before `startObserving()`.** The observer stops the service whenever it sees `DISCONNECTED` — which is exactly what the initial state snapshot is. Without the seed, the service would kill itself before the connect even started. Classic reactive-startup race.
- **`FOREGROUND_SERVICE_TYPE_SPECIAL_USE` on Android 14+.** Since `UPSIDE_DOWN_CAKE`, foreground services must declare a type, and VPN orchestration falls under `specialUse`.

---

## Step 3 — Fetch config & open tunnel

With the service alive and the notification showing "Connecting…", the actual work starts. The service records the user's *intent* (more on that later), fetches a fresh WireGuard config from the backend, and hands it to the manager:

```kotlin
private suspend fun connectAndScheduleLimit(freshSession: Boolean) {
    if (freshSession) stateStore.clearSession()
    stateStore.setDesiredOn(true)
    // Retried: right after a tunnel teardown (e.g. a server switch) the device
    // network needs a moment to settle and the first request often times out.
    val entity = retry { getVpnConnection.execute() }
    if (entity == null) {
        stateStore.setDesiredOn(false)
        manager.reportError(VpnError.CONFIG_FETCH)
        return
    }
    manager.connect(entity)
    scheduleFreeSessionLimit()
}

private suspend fun <T> retry(attempts: Int = CONNECT_RETRIES, block: suspend () -> T): T? {
    repeat(attempts) { attempt ->
        runCatching { return block() }
        if (attempt < attempts - 1) delay(CONNECT_RETRY_DELAY_MS.milliseconds)
    }
    return null
}
```

The config fetch goes through a clean-architecture chain — use case → repository → Retrofit data source:

```kotlin
@Single
class GetVpnConnectionUseCase(private val repository: VpnConnectionRepository) {
    suspend fun execute(): VpnConnectionEntity = repository.get()
}

@Single(binds = [VpnConnectionRepository::class])
class VpnConnectionRepositoryImpl(
    private val api: VpnApiService,
    private val serverSelection: ServerSelectionRepository,
) : VpnConnectionRepository {
    override suspend fun get(): VpnConnectionEntity {
        val request = StartSessionRequest(serverId = serverSelection.get()?.id)
        return api.startSession(request).data.toEntity()
    }
}

interface VpnApiService {
    /** serverId is optional; null lets the backend pick the best server. */
    @POST("api/v1/vpn/session/start")
    suspend fun startSession(@Body request: StartSessionRequest): ApiResponse<VpnSessionDto>

    @DELETE("api/v1/vpn/session/end")
    suspend fun endSession(): ApiResponse<Unit>
}

data class StartSessionRequest(val serverId: String? = null)
```

*(API names, endpoints, and request bodies here are illustrative placeholders — swap in your own backend contract.)*

What comes back is everything the tunnel and the UI need:

```kotlin
data class VpnConnectionEntity(
    val vpnConfig: String,        // full WireGuard config text ([Interface] + [Peer])
    val serverName: String,
    val serverIp: String,
    val serverCountry: ServerCountryEntity,
)
```

**Why fetch the config per-connect instead of shipping it in the app?** Because the backend allocates a session: it picks a server (or honors the user's `serverId`), assigns the client an address, and registers the peer. That's also why disconnect calls the session-end endpoint — the server needs to release that session. Config material is short-lived and per-session, which is exactly what you want for a WireGuard fleet.

---

## Step 4 — Build the network tunnel

Now the manager takes over. This is the densest part of the flow, and the sequence diagram expands it into its own view: parse the config, handle the server-swap case, bring the interface up, then start **two background jobs** — a 1-second polling loop and a 15-second watchdog.

```kotlin
suspend fun connect(entity: VpnConnectionEntity) = mutex.withLock {
    _state.update {
        it.copy(
            status = ConnectionStatus.CONNECTING,
            serverName = entity.serverName,
            serverIp = entity.serverIp,
            serverCountryName = entity.serverCountry.name,
            serverCountryCode = entity.serverCountry.code,
        )
    }

    val config = try {
        Config.parse(entity.vpnConfig.byteInputStream())
    } catch (e: Exception) {
        fail(VpnError.CONFIG_PARSE)
        return@withLock
    }

    if (isTunnelRunning()) {
        if (entity.vpnConfig == activeConfigText) {
            // Re-issued connect for the same server: just resume.
            markConnected()
            startPolling()
            return@withLock
        }
        // Server switch: bounce the tunnel, suppressing the intermediate DOWN so
        // observers (and the foreground service) don't treat it as a disconnect.
        swappingConfig = true
        try {
            withContext(Dispatchers.IO) { backend.setState(tunnel, Tunnel.State.DOWN, null) }
        } finally {
            swappingConfig = false
        }
    }

    try {
        withContext(Dispatchers.IO) { backend.setState(tunnel, Tunnel.State.UP, config) }
    } catch (e: Exception) {
        fail(mapBackendError(e))
        return@withLock
    }
    activeConfigText = entity.vpnConfig

    // Tunnel interface is up; poll flips CONNECTING -> CONNECTED on first handshake.
    startPolling()
    startHandshakeWatchdog()
}
```

Things to notice:

- **A `Mutex` guards connect/disconnect.** A user can tap connect, immediately tap disconnect, and pick a new server within a second. Serializing tunnel mutations turns that from a race into a queue.
- **The server-swap alt-path.** Switching servers means DOWN then UP. But the manager's own `DOWN` callback treats an unexpected drop as a disconnect. The `swappingConfig` flag marks the intermediate DOWN as *ours*, so the notification never flashes "Disconnected" mid-switch.
- **`setState(UP, config)` is where Android actually builds the tunnel.** Under the hood, GoBackend's `VpnService` calls `Builder.establish()`, creating the TUN interface — this is the moment the key icon appears in the status bar.

### CONNECTED means "handshake seen", not "interface up"

Here's the subtle bug most VPN apps ship with: **`Tunnel.State.UP` does not mean you're connected.** The interface can be up while the server is unreachable — wrong port, dead server, blocked UDP. The user sees "Connected" and has no internet.

WireGuard is silent (no persistent control channel), but it does expose *statistics*: byte counters and the timestamp of the latest peer handshake. So the manager treats **the first handshake** as the true CONNECTED signal, checked by the polling job:

```kotlin
/** Sample rx/tx once per second → Mbps, and confirm real connectivity via handshake. */
private fun startPolling() {
    pollJob?.cancel()
    pollJob = scope.launch {
        var lastRx = 0L; var lastTx = 0L
        var lastNanos = System.nanoTime()
        var primed = false
        while (isActive) {
            val stats = runCatching { backend.getStatistics(tunnel) }.getOrNull()
            if (stats != null) {
                val now = System.nanoTime()
                val seconds = ((now - lastNanos).coerceAtLeast(1)) / 1_000_000_000.0
                val rx = stats.totalRx(); val tx = stats.totalTx()
                if (primed) {
                    val down = (rx - lastRx).coerceAtLeast(0) * 8.0 / seconds / 1_000_000.0
                    val up = (tx - lastTx).coerceAtLeast(0) * 8.0 / seconds / 1_000_000.0
                    _traffic.value = VpnTraffic(down, up, rx, tx)
                }
                lastRx = rx; lastTx = tx; lastNanos = now; primed = true

                val latestHandshake = stats.peers()
                    .maxOfOrNull { stats.peer(it)?.latestHandshakeEpochMillis() ?: 0L } ?: 0L
                if (latestHandshake > 0L && _state.value.status != ConnectionStatus.CONNECTED) {
                    markConnected()  // CONNECTING -> CONNECTED on first real handshake
                }
            }
            delay(POLL_INTERVAL_MS) // 1 second
        }
    }
}
```

One loop, two jobs done: **throughput** (delta of byte counters → Mbps, `primed` skips the garbage first sample) and **liveness** (handshake timestamp → `CONNECTED`).

### The watchdog: fail loudly, not silently

If the handshake never arrives, the user must not stare at "Connecting…" forever. A watchdog gives the handshake 15 seconds, then tears everything down:

```kotlin
/** If no handshake lands within the window, tear the tunnel down and report a timeout. */
private fun startHandshakeWatchdog() {
    watchdogJob?.cancel()
    watchdogJob = scope.launch {
        val connected = withTimeoutOrNull(HANDSHAKE_TIMEOUT_MS) {   // 15s
            state.first { it.status == ConnectionStatus.CONNECTED }
            true
        }
        if (connected == null && _state.value.status != ConnectionStatus.CONNECTED) {
            mutex.withLock {
                if (_state.value.status != ConnectionStatus.CONNECTED) {
                    runCatching {
                        withContext(Dispatchers.IO) { backend.setState(tunnel, Tunnel.State.DOWN, null) }
                    }
                    fail(VpnError.HANDSHAKE_TIMEOUT)
                }
            }
        }
    }
}
```

Note the double-check of `status` around the lock — the handshake may land while the watchdog is waiting for the mutex. Success cancels the watchdog implicitly (the `state.first { CONNECTED }` completes); timeout cancels the connection explicitly.

---

## ✓ Connected — state flows back up

Nothing in the upward direction is a callback. Both consumers just collect the same two flows.

**The service** mirrors state into the notification and stops itself when the tunnel is gone:

```kotlin
private fun startObserving() {
    if (observing) return
    observing = true
    scope.launch {
        combine(manager.state, manager.traffic) { s, t -> s to t }.collect { (state, traffic) ->
            if (state.status == ConnectionStatus.DISCONNECTED) {
                stopSelfSafely()
            } else {
                postStatusNotification(state, traffic)
            }
        }
    }
}
```

(With one nice touch: it fingerprints the rendered content and skips the `notify()` binder call when nothing visible changed — traffic updates arrive every second, and there's no reason to spam the notification manager.)

**The ViewModel** mirrors the same flows into Compose state — speed sparklines, the session timer, and the map marker:

```kotlin
private fun observeVpn() {
    screenModelScope.launch {
        combine(vpnManager.state, vpnManager.traffic) { s, t -> s to t }.collect { (s, t) ->
            _state.update {
                it.copy(
                    connectionStatus = s.status,
                    downloadMbps = t.downloadMbps,
                    uploadMbps = t.uploadMbps,
                    downloadHistory = (it.downloadHistory + t.downloadMbps.toFloat())
                        .takeLast(SPARKLINE_SAMPLES),
                    connectedSinceEpochMs = s.connectedSinceEpochMs,
                    countryName = s.serverCountryName ?: it.countryName,
                    ...
                )
            }
            s.error?.let { err ->
                _events.tryEmit(HomeEvent.ShowError(err.messageRes))
                vpnManager.clearError()   // one-shot: consume so it never re-fires
            }
        }
    }
}
```

A small trick hides in the map logic: the geo lookup for the server's coordinates is kicked off **already at `CONNECTING`** — the server IP is known before the tunnel takes over routing, so the lookup usually completes on the untunneled network and the marker is ready the very moment the state flips to `CONNECTED`.

---

## The free-session timer — business logic that must survive process death

Free users get a limited session (the limit comes from app settings, in minutes; `0` means unlimited). The countdown lives **in the service, not a ViewModel** — deliberately, because it must keep running with the app swiped away. The math is a pure object so it stays unit-testable:

```kotlin
object FreeSessionLimit {
    fun deadlineEpochMs(connectedSinceEpochMs: Long, maxSessionMinutes: Int): Long? =
        if (maxSessionMinutes <= 0) null
        else connectedSinceEpochMs + maxSessionMinutes * 60_000L

    fun remainingMs(deadlineEpochMs: Long, nowEpochMs: Long): Long =
        (deadlineEpochMs - nowEpochMs).coerceAtLeast(0L)
}
```

The service arms it only after the tunnel *actually* connects, and — key detail — a **persisted deadline wins over computing a fresh one**, so force-killing the app never extends a free session:

```kotlin
private fun scheduleFreeSessionLimit() {
    sessionLimitJob?.cancel()
    sessionLimitJob = scope.launch {
        val settled = manager.state.first { it.status != ConnectionStatus.CONNECTING }
        if (settled.status != ConnectionStatus.CONNECTED) return@launch

        if (userViewModel.state.value.isPremium()) return@launch   // no limit for premium

        val deadline = stateStore.sessionDeadlineEpochMs() ?: run {
            val settings = runCatching { getAppSettings.execute() }.getOrNull() ?: return@launch
            val since = settled.connectedSinceEpochMs ?: System.currentTimeMillis()
            FreeSessionLimit.deadlineEpochMs(since, settings.freeUserMaxSessionTime)
                ?: return@launch   // 0 = unlimited
        }
        stateStore.setSessionDeadline(deadline)

        delay(FreeSessionLimit.remainingMs(deadline, System.currentTimeMillis()).milliseconds)

        // Session expired: notify, drop the tunnel, and don't auto-reconnect.
        if (canPostNotifications()) showSessionEndedNotification()
        stateStore.setDesiredOn(false)
        teardown()
    }
}
```

---

## The disconnect flow

Disconnect looks symmetric on the diagram but has two traps of its own.

The path: tap disconnect → `viewModel.disconnect()` → `vpnController.disconnect()` → `startForegroundService(ACTION_DISCONNECT)` → `handleDisconnect()`:

```kotlin
private fun handleDisconnect() {
    scope.launch {
        sessionLimitJob?.cancel()
        stateStore.setDesiredOn(false)   // record intent: the user wants the VPN OFF
        teardown()
    }
}

/**
 * Tears down the tunnel, then closes the server-side session. NonCancellable
 * because the DISCONNECTED emission makes the observer stop this service,
 * which cancels [scope] — the close call must survive that.
 */
private suspend fun teardown() = withContext(NonCancellable) {
    sessionDeadlineEpochMs = null
    manager.disconnect()
    withContext(Dispatchers.IO) { retry { closeVpnConnection.execute() } }
}
```

**Trap #1: teardown cancels itself.** `manager.disconnect()` emits `DISCONNECTED` → the observer calls `stopSelf()` → `onDestroy()` cancels the service's coroutine scope → which would cancel the very coroutine still running `teardown()`, killing the session-end API call and leaking the session server-side. `withContext(NonCancellable)` is the fix: the teardown finishes even as the service dies around it.

**Trap #2: the DOWN callback fires for every teardown — yours or the system's.** The manager funnels all of them through one handler:

```kotlin
private fun onTunnelStateChange(newState: Tunnel.State) {
    if (newState == Tunnel.State.DOWN && !swappingConfig) {
        // Tunnel dropped (revoked, killed, or torn down) — preserve any error already set.
        pollJob?.cancel()
        watchdogJob?.cancel()
        _traffic.value = VpnTraffic()
        _state.update { it.copy(status = ConnectionStatus.DISCONNECTED, connectedSinceEpochMs = null) }
        scope.launch { stateStore.clearSession() }
    }
}
```

This single path covers the clean disconnect, the handshake-timeout teardown, **and the case where another VPN app steals the tunnel** (Android revokes yours). Whatever the cause, state converges to `DISCONNECTED`, the service observer sees it, stops the service, and the UI resets — the connect button returns to its idle state, the timer stops, and the map recenters to the user's real location.

---

## Surviving process death: intent vs. state

Android *will* kill your process. The design that makes this survivable is separating **what the user wants** from **what currently is**. The current state lives in memory (`StateFlow`); the user's *intent* and the session's timestamps are persisted in a dedicated DataStore:

```kotlin
// Deliberately a separate DataStore file: TokenLocalDataSource.clear() wipes the whole
// "token_prefs" store on logout, and VPN state must survive that.
private val Context.vpnDataStore by preferencesDataStore(name = "vpn_prefs")

@Single
class VpnStateStore(context: Context) {
    suspend fun isDesiredOn(): Boolean = ...          // "I want the VPN on"
    suspend fun connectedSinceEpochMs(): Long? = ...  // session timer survives restart
    suspend fun sessionDeadlineEpochMs(): Long? = ... // free-session deadline survives restart
    suspend fun selectedServer(): ServerEntity? = ... // the server the user picked
}
```

When the system restarts the service (`START_STICKY`, null intent), it reconciles intent against reality:

```kotlin
private fun handleRestart() {
    scope.launch {
        // If the free session expired while we were dead, don't reconnect at all.
        val expiredDeadline = stateStore.sessionDeadlineEpochMs()
            ?.let { FreeSessionLimit.remainingMs(it, System.currentTimeMillis()) == 0L } == true
        val desired = stateStore.isDesiredOn() && !expiredDeadline
        when {
            manager.state.value.status != ConnectionStatus.DISCONNECTED -> {
                // Tunnel/manager survived (service-only restart); just re-arm.
                startObserving(); scheduleFreeSessionLimit()
            }
            // Consent can be revoked while we're dead; VpnService.prepare() is the check.
            desired && VpnService.prepare(this@VpnForegroundService) == null -> {
                manager.markConnecting(); startObserving()
                connectAndScheduleLimit(freshSession = false)  // resume the old countdown
            }
            else -> {
                stateStore.setDesiredOn(false); stateStore.clearSession()
                startObserving(); stopSelfSafely()
            }
        }
    }
}
```

And if only the *UI* died while the tunnel lived on, the manager re-adopts it at startup — restoring the persisted session start so the Home timer doesn't restart from zero:

```kotlin
/** Re-adopt a tunnel that survived the UI/process, restoring the persisted session start. */
private fun restoreState() {
    scope.launch {
        if (isTunnelRunning()) {
            val persistedSince = stateStore.connectedSinceEpochMs()
            _state.update {
                it.copy(
                    status = ConnectionStatus.CONNECTED,
                    connectedSinceEpochMs = it.connectedSinceEpochMs
                        ?: persistedSince ?: System.currentTimeMillis(),
                )
            }
            startPolling()
        }
    }
}
```

---

## Error handling: one enum, one channel

Every failure in the whole flow funnels into a small user-facing enum, resolved to localized text only at the UI layer:

```kotlin
enum class VpnError(@StringRes val messageRes: Int) {
    CONFIG_FETCH(R.string.vpn_error_config_fetch),        // backend unreachable / session denied
    CONFIG_PARSE(R.string.vpn_error_config_parse),        // malformed WireGuard config
    NOT_AUTHORIZED(R.string.vpn_error_not_authorized),    // user declined the consent dialog
    HANDSHAKE_TIMEOUT(R.string.vpn_error_handshake_timeout), // interface up, server silent (15s)
    DNS_FAILURE(R.string.vpn_error_dns),                  // endpoint hostname didn't resolve
    UNKNOWN(R.string.vpn_error_unknown),
}
```

Backend exceptions map into it (`BackendException.Reason.VPN_NOT_AUTHORIZED → NOT_AUTHORIZED`, `DNS_RESOLUTION_FAILURE → DNS_FAILURE`), and the ViewModel consumes each error exactly once: show snackbar → `clearError()`. No sticky error states, no error dialogs surviving a screen rotation.

---

## Takeaways

If you're building a VPN app on Android, these are the decisions from this codebase I'd carry into any new one:

1. **One owner for the tunnel.** A singleton manager publishing `StateFlow`s; everyone else drives or observes. The moment two components can call `setState()`, you have races.
2. **`UP` ≠ connected.** Gate `CONNECTED` on the first WireGuard handshake, and back it with a watchdog so failure is loud and bounded (15s), not an infinite spinner.
3. **Foreground service = process keep-alive + notification, nothing more.** It fetches configs and delegates; it never touches the tunnel. And it must `startForeground()` synchronously.
4. **Persist intent, not just state.** `isDesiredOn` + persisted deadlines let `START_STICKY` restarts, reboots, and force-kills reconcile correctly — and prevent free users from resetting their session limit by killing the app.
5. **Make teardown `NonCancellable`.** Any cleanup that includes a network call (closing the server-side session) must survive the service cancelling its own scope.
6. **Optimistic `CONNECTING`, suppressed intermediate `DOWN`s.** Users judge the app by the connect button and the notification; never let internal transitions (server swap, initial snapshot) leak as visible flicker.

The full sequence — every arrow in this post — lives in the interactive diagram **[VPN Connection Flow](https://hadiuzzaman524.github.io/vpn-android/index.html)**, covering the connect flow, disconnect flow, the expanded service layer (`VpnController → VpnForegroundService → VpnConnectionManager`), and the tunnel/monitoring detail (1-second polling, 15-second watchdog).

*Happy tunneling.* 🔐

---

## Connect with me

If you enjoyed this deep dive, I write more about Android, iOS, and TV app development:

- 📄 **Interactive VPN flow docs:** [hadiuzzaman524.github.io/vpn](https://hadiuzzaman524.github.io/vpn-android/index.html)
- 💼 **LinkedIn:** [Md Hadiuzzaman](https://www.linkedin.com/pub/dir/Md/Hadiuzzaman) — let's connect!
- ✍️ **Medium:** [md-hadi.medium.com](https://md-hadi.medium.com/) — follow for more articles like this one
