# Document 24 — Per-Package Deep Dives

> **Prerequisites:** [02 — Packages](./02-repository-and-package-architecture.md) and the relevant subsystem documents.
> **Source version:** commit `e1ebcba3c32c7cb68fdf00bb48bac49a5c10f07d`, branch `main`.

## Purpose

Recursive deep-dives into the most important packages, using a consistent template. Trivial packages (`icons`, `styles`, `releases`, `toast`) are summarized only. Sections irrelevant to a package are omitted rather than padded.

**Template:** Why it exists · Owns · Does NOT own · Public API · Internal architecture · State · Concurrency · Persistence · Error handling · Performance · Security · Invariants · Extension points · Known limitations · Source map.

---

## 24.1 `@standardnotes/models` — the domain core

- **Why:** a shared, immutable object model (payloads/items) that every layer can depend on without pulling in services or crypto. The bottom of the domain stack.
- **Owns:** `Payload` variants (Decrypted/Encrypted/Deleted), `Item` classes (`SNNote`, `SNTag`, `SNItemsKey`, `FileItem`, `SmartView`, `Component`), mutators, `ItemCollection`/`ImmutablePayloadCollection`, `ItemDisplayController`, `ContentType` usage, and the **sync deltas** (`ConflictDelta`, `DeltaRemoteRetrieved/Saved/…`).
- **Does NOT own:** encryption (only consumes key *shapes*), storage, sync orchestration, networking.
- **Public API:** `DecryptedPayload`, `CreateDecryptedItemFromPayload`, `ImmutablePayloadCollection`, `ConflictDelta`, `ContentType` value objects, `getIncrementedDirtyIndex`.
- **Internal architecture:** immutable value objects with `copy`/`copyAsSyncResolved`; collections indexed by uuid/content-type; deltas are **pure functions** returning `DeltaEmit`.
- **State:** none persistent — pure data. **Concurrency:** none (synchronous value ops).
- **Error handling:** type-checks (`isDecryptedPayload`, `isErrorDecryptingPayload`) drive conflict/errored routing.
- **Performance:** immutability means allocations on every mutation; collection ops are O(1) by uuid, O(N) for scans.
- **Security:** holds decrypted content in memory; no crypto itself.
- **Invariants:** S1 (immutability), Y4 (conflict duplication) originate here.
- **Extension points:** new `ContentType` + item subclass + optional `strategyWhenConflictingWithItem` ([Document 21 §3](./21-extension-points.md)).
- **Known limitations:** immutability allocation churn on hot edit paths.
- **Source map:** `src/Domain/Abstract/{Payload,Item}`, `src/Domain/Runtime/{Collection,Deltas}`, `src/Domain/Syncable/*`.

---

## 24.2 `@standardnotes/sncrypto-web` — crypto primitives

- **Why:** isolate all primitive cryptography (libsodium/WebCrypto) behind `PureCryptoInterface` so the rest of the app is platform- and library-agnostic.
- **Owns:** `SNWebCrypto` implementing argon2id, XChaCha20-Poly1305 (AEAD + secretstream), `crypto_box`/`crypto_sign`, `crypto_kdf`, hashing; libsodium `ready` lifecycle.
- **Does NOT own:** key hierarchy, operators, item semantics (those are in `encryption`).
- **Public API:** `SNWebCrypto` (injected as `options.crypto`), `PureCryptoInterface` (from `sncrypto-common`).
- **Internal architecture:** thin wrappers over `libsodium-wrappers`; `initialize()` awaits `sodium.ready`; hex/base64/Uint8Array marshaling.
- **State:** the libsodium module (WASM) + a `ready` promise. **Concurrency:** synchronous main-thread calls; WASM on main thread ([Document 11](./11-workers-concurrency-and-wasm.md)).
- **Persistence/Security:** copies keys into WASM memory per call; **no explicit zeroization** of JS-side buffers ([Document 17 §6](./17-memory-architecture.md)).
- **Performance:** measured — argon2 ~0.67 s; AEAD ~87 MB/s ([Document 16 §2](./16-performance-engineering.md)).
- **Invariants:** C5 (WASM ready before use).
- **Extension points:** add primitives to `PureCryptoInterface` for a new operator ([Document 21 §8](./21-extension-points.md)).
- **Known limitations:** main-thread execution; no worker offload.
- **Source map:** `src/crypto.ts`, `src/libsodium.ts`, `src/utils.ts`.

---

## 24.3 `@standardnotes/encryption` — the stateless crypto library

- **Why:** encode the SN protocol (versions, key hierarchy, envelope) as **stateless** operators, separate from the stateful `EncryptionService`.
- **Owns:** `Operator001–004`, `SNRootKey`/`RootKeyParams`, `ItemsKey`, key-system keys, KDF/derivation use-cases, backup encode/decode, `V00xAlgorithm` constants, encryption **splits**.
- **Does NOT own:** in-memory key state, storage, sync (that’s `EncryptionService`/`RootKeyManager` in `services`).
- **Public API:** `EncryptionOperators`, `Operator004`, `SNRootKey`, `CreateEncryptionSplitWithKeyLookup`, `SplitPayloadsByEncryptionType`.
- **Internal architecture:** an operator is a facade over per-operation use-cases (encrypt/decrypt/derive/asymmetric); `004` uses envelope encryption ([Document 07 §4](./07-cryptographic-architecture.md)).
- **State:** none (operators are stateless; keys are passed in).
- **Security:** the crypto boundary; produces/consumes `004` strings; AEAD + AAD binding.
- **Invariants:** C1–C4 (crypto boundary, envelope, AEAD/AAD, protocol versioning).
- **Extension points:** new `OperatorInterface` version; **never remove old ones** ([Document 21 §8](./21-extension-points.md), [Document 23 §5](./23-legacy-architecture-and-technical-debt.md)).
- **Known limitations:** legacy operators add surface (intentional).
- **Source map:** `src/Domain/Operator/00x`, `src/Domain/Keys`, `src/Domain/Algorithm.ts`, `src/Domain/Split`, `src/Domain/Backups`.

---

## 24.4 `@standardnotes/api` — HTTP endpoints

- **Why:** isolate all networking + endpoint shapes behind services so the rest of the app is transport-agnostic.
- **Owns:** `HttpService`, endpoint “servers” (User, Subscription, WebSocket, Revision, Authenticator, Auth, SharedVault, AsymmetricMessage), request/response typing.
- **Does NOT own:** session/token *lifecycle* (that’s `SessionManager` in snjs), sync orchestration.
- **Public API:** `HttpService`/`HttpServiceInterface`, `*ApiService`, `*Server`.
- **State:** in-memory auth token/host; set via callbacks from `LegacyApiService` (`Application.ts:248-251`).
- **Concurrency:** async HTTP; retries/backoff at the sync layer.
- **Error handling:** returns `HttpResponse` with `ClientDisplayableError`; 401 → session refresh/revoke ([Document 18 §4](./18-error-handling-and-resilience.md)).
- **Security:** sends only ciphertext + bearer tokens + `serverPassword`.
- **Invariants:** C1 (nothing but ciphertext/auth leaves).
- **Known limitations:** coexists with `DeprecatedHttpService`/`LegacyApiService` ([Document 23 §5](./23-legacy-architecture-and-technical-debt.md)).
- **Source map:** `src/Domain/Http`, `src/Domain/*Api`, `src/Domain/*Server`.

---

## 24.5 `@standardnotes/services` — the orchestration hub

- **Why:** house the stateful application services and the base primitives (`AbstractService`, `InternalEventBus`, interfaces). The “brain,” minus the classic services still in `snjs` ([Document 23 §1](./23-legacy-architecture-and-technical-debt.md)).
- **Owns:** `EncryptionService`, `ItemsEncryptionService`, `RootKeyManager`, `KeySystemKeyManager`, Vaults/Contacts/AsymmetricMessage services, `StatusService`, `IntegrityService`, `SubscriptionManager`, `UserService`, `NotificationService`, `AbstractService`, `InternalEventBus`, and the huge interface/use-case surface.
- **Does NOT own:** Sync/Items/Payloads/Storage/Session/Components (those live in `snjs/lib/Services`, mid-extraction).
- **Public API:** consumed via the snjs barrel; `AbstractService`, `InternalEventBus`, `EncryptionService`, service interfaces, use-cases, `DeviceInterface`.
- **Internal architecture:** each service extends `AbstractService` (dual-dispatch events, `deinit`, `blockDeinit`); heavy use of **use-case** classes for testability ([Document 22 §4](./22-tests-as-architecture.md)).
- **State:** most durable/session app state (keys, features, vaults, status).
- **Concurrency:** async; the event bus + stage ladder sequence init ([Document 15](./15-events-and-internal-communication.md)).
- **Security:** `EncryptionService` is the crypto orchestrator; `RootKeyManager` holds the in-memory root key.
- **Invariants:** C6 (root key dropped on deinit), E1/E2 (event planes), B5/B6 (deinit + critical writes).
- **Extension points:** new service = `AbstractService` + DI factory (prefer this package) ([Document 21 §9](./21-extension-points.md)).
- **Known limitations:** consumed as **raw TS source** (`main: src/index.ts`), contributing to the bundle-duplication concern ([Document 14 §5](./14-build-system-and-delivery.md)); ~60+ services in one package.
- **Source map:** `src/Domain/{Service,Internal,Encryption,RootKeyManager,ItemsEncryption,Vault,Contacts,Status,Integrity,User,...}`.

---

## 24.6 `@standardnotes/snjs` — aggregator + composition root + classic services

- **Why:** provide a single SDK surface (`SNApplication`, barrel) and host the classic core services + migrations + composition.
- **Owns:** `SNApplication`, `SNApplicationGroup`, the DI `Dependencies` container, **classic services** (Sync/Items/Payloads/Mutator/Storage/Session/Components/History/Features/Preferences/Settings/Protection/Challenge/Migration/Actions/Listed/Singleton/KeyRecovery/Mfa/Api), `Migrations`, `Version`, the e2e server.
- **Does NOT own:** the UI, platform shells.
- **Public API:** `SNApplication`, `SNApplicationGroup`, and re-exports of `models`/`services`/`encryption`/`files`/`api`/`responses`.
- **Internal architecture:** lazy-singleton DI ([Document 03 §3](./03-bootstrap-and-dependency-construction.md)); staged launch; observer wiring.
- **State:** owns the DI container (all singletons).
- **Concurrency:** the whole runtime; single-flight sync; serialized emit.
- **Persistence:** delegates to `DiskStorageService`/`DeviceInterface`.
- **Invariants:** B1–B6 (bootstrap), Y1–Y6 (sync), S2–S5 (state).
- **Known limitations:** overloaded god package; prebuilt bundle → duplication ([Document 23 §2–3](./23-legacy-architecture-and-technical-debt.md)).
- **Source map:** `lib/Application`, `lib/ApplicationGroup`, `lib/Services`, `lib/Migrations`.

---

## 24.7 `@standardnotes/ui-services` — UI-facing services

- **Why:** bridge domain services to UI concerns (theme, keyboard, routing, archive, autolock) and define the `WebApplicationInterface` **port** the shell implements (dependency inversion, [Document 02 §6](./02-repository-and-package-architecture.md)).
- **Owns:** `ThemeManager`, `KeyboardService`, `RouteService`, `ArchiveManager`, `AutolockService`, `Importer`, `PluginsService`, `WebApplicationInterface`, `VaultDisplayService`, various UI use-cases.
- **Does NOT own:** React components (those are in `web`), domain state.
- **Public API:** the above services + `WebApplicationInterface`.
- **State:** theme selection, autolock timers, route state.
- **Concurrency:** DOM timers/listeners; `deinit` cleanup.
- **Security:** none directly; mediates file import/export.
- **Invariants:** the UI↔service dependency inversion (X3-adjacent).
- **Known limitations:** consumed as source (bundle duplication factor).
- **Source map:** `src/Domain/{Theme,Keyboard,Route,Archive,Autolock,Import,...}`.

---

## 24.8 `@standardnotes/web` — the React app + web device

- **Why:** the single UI for all platforms plus the browser `DeviceInterface`.
- **Owns:** `WebApplication` (extends `SNApplication`), `WebApplicationGroup`, `WebDevice`/`WebOrDesktopDevice` (localStorage + IndexedDB), all MobX **controllers**, React components, editors integration (Plain, Super/Lexical, iframe hosting), the PDF worker.
- **Does NOT own:** domain/services logic (delegated to snjs), native platform capabilities.
- **Public API:** the built `app.js` (consumed by desktop/mobile shells), `WebApplication` (for desktop/mobile), controllers.
- **Internal architecture:** two-container DI (snjs + `WebDependencies`); MobX controllers bridge domain streams to `observer()` components ([Document 10](./10-react-architecture.md)); `ApplicationGroupView`/`ApplicationView` mount + gate.
- **State:** controller (MobX) UI state; the web `DeviceInterface` (IndexedDB/localStorage).
- **Concurrency:** React render + the one PDF worker; main-thread crypto via injected `WebCrypto`.
- **Persistence:** implements the storage sinks (IndexedDB `items`, localStorage KV + keychain).
- **Security:** web keychain is `localStorage` (weakest, [Document 19 §4](./19-security-boundaries.md)).
- **Invariants:** X1–X3 (editor writes via pipeline; content-type safety; domain outside React).
- **Known limitations:** god package (app + device + controllers + editors); single `app.js`, no vendor split.
- **Source map:** `src/javascripts/{Application,Components,Controllers,Hooks,Event}`.

---

## 24.9 Platform shells (`desktop`, `mobile`, `clipper`) — summary

| Package | Owns | Key files |
| ------- | ---- | --------- |
| `desktop` | Electron main + renderer, `DesktopDevice`, IPC `RemoteBridge`, Keychain (keytar), UpdateManager, ExtensionsServer, Spellchecker, Menus, PackageManager | `app/index.ts`, `app/javascripts/Main/*` ([Document 13 §4](./13-multi-platform-architecture.md)) |
| `mobile` | RN WebView host, `MobileDevice` bridge, native services (state/back/color-scheme), notifications, share | `src/MobileWebAppContainer.tsx`, `src/Lib/MobileDevice.ts` ([Document 13 §5](./13-multi-platform-architecture.md)) |
| `clipper` | browser-extension build reusing snjs | `web` with `BUILD_TARGET=clipper` ([Document 14 §6](./14-build-system-and-delivery.md)) |

---

## 24.10 Leaf/support packages — summary

| Package | Role |
| ------- | ---- |
| `sncrypto-common` | `PureCryptoInterface` + algorithm enums (contract) |
| `utils` | pure helpers, `DependencyContainer`, `Result` re-exports |
| `features` | static capability descriptors (editors/themes/perms) |
| `responses` | server response DTOs + `ConflictType`/errors |
| `files` | encrypted file streaming (`crypto_secretstream`) + backup iface |
| `filepicker` | platform file open/save + stream targets |
| `toast`/`icons`/`styles` | presentational UI primitives |
| `releases` | generated release metadata JSON |

---

## What you should now understand

- The responsibility boundary, state, and invariants of each important package.
- Where the crypto stack splits (`sncrypto-web` primitives → `encryption` protocol → `services` orchestration).
- Why `services`/`snjs`/`web` are broad, and the mid-extraction/duplication caveats to keep in mind.

## Architectural invariants learned

- Consolidated from the per-package sections; the authoritative list is [Document 20](./20-architectural-invariants.md).

## Open questions

- Deferred to each subsystem document’s open questions.

## Source index

- Each package’s `src`/`lib` root as mapped above; cross-referenced to the subsystem documents.

## Next document

Continue to **[Document 25 — Maintainer Handbook](./25-maintainer-handbook.md)** — the final synthesis.
