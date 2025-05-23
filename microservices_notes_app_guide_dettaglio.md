# Applicazione Note Collaborative con Architettura Microservizi

## Indice
1. [Definizione del Progetto](#definizione-del-progetto)
2. [Architettura del Sistema](#architettura-del-sistema)
3. [Stack Tecnologico Backend](#stack-tecnologico-backend)
4. [Analisi Frontend Technologies](#analisi-frontend-technologies)
5. [SvelteKit - Soluzione Raccomandata](#sveltekit-soluzione-raccomandata)
6. [Implementazione Backend](#implementazione-backend)
7. [Implementazione Frontend](#implementazione-frontend)
8. [Integrazione e Comunicazione](#integrazione-e-comunicazione)
9. [Deployment e Scalabilità](#deployment-e-scalabilità)
10. [Roadmap di Sviluppo](#roadmap-di-sviluppo)

---

## Definizione del Progetto

### Obiettivo
Sviluppare un prototipo di applicazione collaborativa per la gestione di note in Markdown che serva come scheletro per future applicazioni commerciali in produzione.

### Funzionalità Core
- **Autenticazione utenti**: Sistema sicuro di login/logout con token JWT
- **Gestione note**: Creazione, modifica, eliminazione di note in formato Markdown
- **Collaborazione**: Modifica simultanea delle note da parte di più utenti
- **Tagging**: Sistema di etichettatura per organizzare le note
- **Evidenziazione**: Possibilità di mettere in risalto note importanti
- **Multi-piattaforma**: Supporto per web, mobile e desktop

### Requisiti Non Funzionali
- **Scalabilità**: Architettura microservizi per gestire crescita
- **Performance**: Risposta rapida e sincronizzazione real-time
- **Sicurezza**: Autenticazione robusta e protezione dati
- **Manutenibilità**: Codice pulito e ben documentato
- **Portabilità**: Deployment su diverse piattaforme

---

## Architettura del Sistema

### Overview Architetturale
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Client    │    │  Mobile Client  │    │ Desktop Client  │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
          ┌────────────────────────────────────────────┐
          │              API Gateway                   │
          └────────────────┬───────────────────────────┘
                           │
          ┌────────────────┼───────────────────────────┐
          │                │                           │
    ┌─────▼─────┐    ┌─────▼─────┐              ┌─────▼─────┐
    │   Auth    │    │   Notes   │              │   NATS    │
    │  Service  │    │  Service  │              │  Broker   │
    └─────┬─────┘    └─────┬─────┘              └─────┬─────┘
          │                │                          │
          └────────────────┼──────────────────────────┘
                           │
                  ┌────────▼────────┐
                  │    Database     │
                  │   PostgreSQL    │
                  └─────────────────┘
```

### Principi Architetturali
- **Separation of Concerns**: Ogni microservizio ha una responsabilità specifica
- **Loose Coupling**: Servizi indipendenti comunicanti via message broker
- **High Cohesion**: Funzionalità correlate raggruppate nello stesso servizio
- **Stateless Services**: Servizi senza stato per facilitare scaling
- **Event-Driven**: Comunicazione asincrona tramite eventi

---

## Stack Tecnologico Backend

### Linguaggio e Framework
**Rust** è stato scelto per i seguenti vantaggi:
- **Performance**: Velocità comparabile a C/C++ senza garbage collection
- **Memory Safety**: Prevenzione di memory leaks e buffer overflow
- **Concurrency**: Gestione eccellente di operazioni parallele
- **Ecosystem**: Crate di alta qualità per web development

### Framework e Librerie Backend

#### Axum (Web Framework)
```rust
// Esempio di router Axum
use axum::{
    routing::{get, post},
    Router,
    Json,
    response::Json as ResponseJson,
};

async fn create_note(Json(payload): Json<CreateNote>) -> ResponseJson<Note> {
    // Logic per creare nota
}

fn app() -> Router {
    Router::new()
        .route("/api/notes", post(create_note))
        .route("/api/notes", get(list_notes))
        .route("/api/notes/:id", get(get_note))
}
```

**Vantaggi di Axum:**
- Ergonomia eccellente per API REST
- Type-safe routing e extractors
- Middleware composabili
- Integrazione nativa con Tokio
- Performance elevate

#### Diesel (ORM)
```rust
// Schema definition
table! {
    notes (id) {
        id -> Uuid,
        title -> Varchar,
        content -> Text,
        author_id -> Uuid,
        tags -> Array<Text>,
        highlighted -> Bool,
        created_at -> Timestamptz,
        updated_at -> Timestamptz,
    }
}

// Model definition
#[derive(Queryable, Serialize, Deserialize)]
pub struct Note {
    pub id: Uuid,
    pub title: String,
    pub content: String,
    pub author_id: Uuid,
    pub tags: Vec<String>,
    pub highlighted: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}
```

**Vantaggi di Diesel:**
- Compile-time query validation
- Type-safe database operations
- Migration system robusto
- Performance ottimizzate

#### Tokio (Async Runtime)
- Runtime asincrono per I/O non bloccante
- Gestione efficiente di migliaia di connessioni simultanee
- Task scheduling avanzato
- Integrazione con tutto l'ecosistema async di Rust

#### NATS (Message Broker)
```rust
// Esempio di publisher
async fn publish_note_event(client: &nats::Client, event: NoteEvent) -> Result<()> {
    let payload = serde_json::to_vec(&event)?;
    client.publish("notes.events", payload).await?;
    Ok(())
}

// Esempio di subscriber
async fn handle_note_events(mut subscriber: Subscriber) {
    while let Some(msg) = subscriber.next().await {
        let event: NoteEvent = serde_json::from_slice(&msg.data)?;
        process_note_event(event).await;
    }
}
```

**Vantaggi di NATS:**
- Latenza ultra-bassa (microsecond)
- Clustering e high availability nativo
- Pattern pub/sub e request/reply
- Lightweight e semplice da operare

---

## Analisi Frontend Technologies

### Confronto Dettagliato delle Opzioni

#### React.js
**Vantaggi:**
- Ecosistema maturo e vastissimo
- Community support eccellente
- Librerie specifiche per markdown editing (react-markdown, @uiw/react-md-editor)
- Time-to-market veloce
- Facilità nel trovare sviluppatori
- Ottima gestione delle REST API

**Svantaggi:**
- Bundle size considerevole (~40KB+ per app complesse)
- Virtual DOM overhead
- Boilerplate verboso
- Performance non ottimali per applicazioni real-time intensive

**Caso d'uso ideale:** Quando si prioritizza velocità di sviluppo e disponibilità di talenti.

#### Rust WASM
**Vantaggi:**
- Performance native-like
- Condivisione di tipi e logica con backend
- Memory safety anche nel frontend
- Nessun runtime JavaScript

**Svantaggi:**
- Ecosistema immaturo per UI complesse
- Curva di apprendimento molto ripida
- Poche librerie per markdown editing
- Debugging più complesso
- Bundle size iniziale elevato

**Caso d'uso ideale:** Applicazioni con requisiti di performance critici e team con expertise Rust avanzata.

#### Flutter
**Vantaggi:**
- UI consistente cross-platform
- Performance buone su mobile
- Single codebase per mobile e desktop
- Dart language relativamente semplice

**Svantaggi:**
- Meno naturale per applicazioni web
- Ecosistema limitato per editing markdown collaborativo
- Bundle size elevato per web
- SEO challenges per web

**Caso d'uso ideale:** Applicazioni mobile-first con necessità di consistenza UI.

#### Alternative Moderne

##### Solid.js
**Vantaggi:**
- Performance superiori (no Virtual DOM)
- Sintassi simile a React
- Bundle size ridotto
- Reattività fine-grained

**Svantaggi:**
- Ecosistema più piccolo
- Meno librerie per markdown
- Community limitata

##### Leptos (Full Rust)
**Vantaggi:**
- Stack unificato Rust
- Performance eccellenti
- Type safety completa
- SSR nativo

**Svantaggi:**
- Molto giovane e sperimentale
- Ecosistema quasi inesistente
- Documentazione limitata

---

## SvelteKit - Soluzione Raccomandata

### Perché SvelteKit è la Scelta Ottimale

#### Vantaggi Tecnici
1. **Performance Superiori**
   - Nessun Virtual DOM overhead
   - Bundle size ridotto (5-10KB vs 40KB+ di React)
   - Compilazione ottimizzata a build-time
   - Memoria utilizzata minimale

2. **Developer Experience Eccellente**
   - Sintassi intuitiva e meno verbosa
   - Reattività automatica con `$:`
   - State management built-in
   - Hot module replacement veloce

3. **Full-Stack Integration**
   - API routes integrate
   - SSR/SSG nativo
   - File-based routing
   - Deployment versatile

4. **Ecosistema Maturo**
   - Librerie per markdown editing
   - WebSocket support nativo
   - UI component libraries
   - Testing tools

### Architettura SvelteKit per il Progetto

#### Struttura del Progetto
```
src/
├── routes/                    # File-based routing
│   ├── +layout.svelte        # Layout principale
│   ├── +page.svelte          # Homepage
│   ├── login/
│   │   └── +page.svelte      # Pagina login
│   ├── notes/
│   │   ├── +page.svelte      # Lista note
│   │   ├── [id]/
│   │   │   ├── +page.svelte  # Visualizza nota
│   │   │   └── edit/
│   │   │       └── +page.svelte # Modifica nota
│   │   └── new/
│   │       └── +page.svelte  # Nuova nota
│   └── api/                  # API routes (proxy to Rust backend)
│       ├── auth/
│       │   └── +server.js
│       └── notes/
│           └── +server.js
├── lib/                      # Librerie condivise
│   ├── components/           # Componenti riutilizzabili
│   │   ├── NoteCard.svelte
│   │   ├── MarkdownEditor.svelte
│   │   └── TagInput.svelte
│   ├── stores/              # Gestione stato
│   │   ├── auth.js
│   │   ├── notes.js
│   │   └── websocket.js
│   └── utils/               # Utility functions
│       ├── api.js
│       └── markdown.js
└── app.html                 # Template HTML
```

#### Gestione Stato con Stores
```javascript
// src/lib/stores/notes.js
import { writable, derived } from 'svelte/store';
import { websocketStore } from './websocket.js';

function createNotesStore() {
    const { subscribe, set, update } = writable([]);
    
    return {
        subscribe,
        
        // Carica tutte le note
        load: async () => {
            const response = await fetch('/api/notes');
            const notes = await response.json();
            set(notes);
        },
        
        // Crea nuova nota
        create: async (noteData) => {
            const response = await fetch('/api/notes', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(noteData)
            });
            
            if (response.ok) {
                const newNote = await response.json();
                update(notes => [...notes, newNote]);
                return newNote;
            }
        },
        
        // Aggiorna nota esistente
        update: async (id, updates) => {
            const response = await fetch(`/api/notes/${id}`, {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(updates)
            });
            
            if (response.ok) {
                const updatedNote = await response.json();
                update(notes => 
                    notes.map(note => note.id === id ? updatedNote : note)
                );
            }
        },
        
        // Elimina nota
        delete: async (id) => {
            const response = await fetch(`/api/notes/${id}`, {
                method: 'DELETE'
            });
            
            if (response.ok) {
                update(notes => notes.filter(note => note.id !== id));
            }
        }
    };
}

export const notesStore = createNotesStore();

// Store derivato per note filtrate
export const filteredNotes = derived(
    [notesStore, searchStore, tagFilterStore],
    ([$notes, $search, $tagFilter]) => {
        return $notes.filter(note => {
            const matchesSearch = note.title.toLowerCase().includes($search.toLowerCase()) ||
                                 note.content.toLowerCase().includes($search.toLowerCase());
            const matchesTags = $tagFilter.length === 0 || 
                               $tagFilter.some(tag => note.tags.includes(tag));
            
            return matchesSearch && matchesTags;
        });
    }
);
```

#### Real-time Collaboration
```javascript
// src/lib/stores/websocket.js
import { writable } from 'svelte/store';

function createWebSocketStore() {
    const { subscribe, set, update } = writable({
        connected: false,
        socket: null
    });
    
    return {
        subscribe,
        
        connect: (url) => {
            const socket = new WebSocket(url);
            
            socket.onopen = () => {
                update(state => ({ ...state, connected: true, socket }));
            };
            
            socket.onmessage = (event) => {
                const data = JSON.parse(event.data);
                handleWebSocketMessage(data);
            };
            
            socket.onclose = () => {
                update(state => ({ ...state, connected: false, socket: null }));
                // Reconnection logic
                setTimeout(() => connect(url), 5000);
            };
        },
        
        send: (message) => {
            update(state => {
                if (state.socket && state.connected) {
                    state.socket.send(JSON.stringify(message));
                }
                return state;
            });
        }
    };
}

function handleWebSocketMessage(data) {
    switch (data.type) {
        case 'note_updated':
            notesStore.handleRealTimeUpdate(data.payload);
            break;
        case 'user_typing':
            typingIndicatorStore.update(data.payload);
            break;
        case 'note_created':
            notesStore.handleRealTimeCreate(data.payload);
            break;
    }
}

export const websocketStore = createWebSocketStore();
```

---

## Implementazione Backend

### Servizio di Autenticazione

#### Struttura del Servizio
```rust
// src/auth/main.rs
use axum::{
    routing::{post, get},
    Router,
    Json,
    http::StatusCode,
    response::Json as ResponseJson,
};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use jsonwebtoken::{encode, decode, Header, EncodingKey, DecodingKey, Validation};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: Uuid,
    exp: usize,
    iat: usize,
}

#[derive(Deserialize)]
struct LoginRequest {
    email: String,
    password: String,
}

#[derive(Serialize)]
struct LoginResponse {
    token: String,
    user: User,
}

#[derive(Serialize, Deserialize)]
struct User {
    id: Uuid,
    email: String,
    name: String,
}

async fn login(Json(payload): Json<LoginRequest>) -> Result<ResponseJson<LoginResponse>, StatusCode> {
    // Validate credentials against database
    let user = authenticate_user(&payload.email, &payload.password).await
        .map_err(|_| StatusCode::UNAUTHORIZED)?;
    
    // Generate JWT token
    let claims = Claims {
        sub: user.id,
        exp: (chrono::Utc::now() + chrono::Duration::hours(24)).timestamp() as usize,
        iat: chrono::Utc::now().timestamp() as usize,
    };
    
    let token = encode(&Header::default(), &claims, &EncodingKey::from_secret("secret".as_ref()))
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    
    // Publish login event to NATS
    publish_auth_event(AuthEvent::UserLoggedIn {
        user_id: user.id,
        timestamp: chrono::Utc::now(),
    }).await;
    
    Ok(ResponseJson(LoginResponse { token, user }))
}

async fn verify_token(Json(token): Json<String>) -> Result<ResponseJson<User>, StatusCode> {
    let claims = decode::<Claims>(
        &token,
        &DecodingKey::from_secret("secret".as_ref()),
        &Validation::default()
    ).map_err(|_| StatusCode::UNAUTHORIZED)?;
    
    let user = get_user_by_id(claims.claims.sub).await
        .map_err(|_| StatusCode::NOT_FOUND)?;
    
    Ok(ResponseJson(user))
}

fn app() -> Router {
    Router::new()
        .route("/auth/login", post(login))
        .route("/auth/verify", post(verify_token))
        .route("/auth/refresh", post(refresh_token))
}
```

### Servizio Note

#### Modelli e Schema Database
```rust
// src/notes/models.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Queryable, Serialize, Deserialize, Clone)]
pub struct Note {
    pub id: Uuid,
    pub title: String,
    pub content: String,
    pub author_id: Uuid,
    pub tags: Vec<String>,
    pub highlighted: bool,
    pub collaborators: Vec<Uuid>,
    pub version: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Insertable, Deserialize)]
#[table_name = "notes"]
pub struct NewNote {
    pub title: String,
    pub content: String,
    pub author_id: Uuid,
    pub tags: Vec<String>,
    pub highlighted: bool,
}

#[derive(Deserialize)]
pub struct UpdateNote {
    pub title: Option<String>,
    pub content: Option<String>,
    pub tags: Option<Vec<String>>,
    pub highlighted: Option<bool>,
}

// Database schema
table! {
    notes (id) {
        id -> Uuid,
        title -> Varchar,
        content -> Text,
        author_id -> Uuid,
        tags -> Array<Text>,
        highlighted -> Bool,
        collaborators -> Array<Uuid>,
        version -> Int4,
        created_at -> Timestamptz,
        updated_at -> Timestamptz,
    }
}

table! {
    note_revisions (id) {
        id -> Uuid,
        note_id -> Uuid,
        content -> Text,
        author_id -> Uuid,
        version -> Int4,
        created_at -> Timestamptz,
    }
}
```

#### Service Implementation
```rust
// src/notes/service.rs
use axum::{
    routing::{get, post, patch, delete},
    Router,
    Json,
    Path,
    Query,
    http::StatusCode,
    response::Json as ResponseJson,
    extract::Extension,
};

#[derive(Deserialize)]
struct ListNotesQuery {
    page: Option<i32>,
    limit: Option<i32>,
    tags: Option<String>,
    search: Option<String>,
}

async fn list_notes(
    Query(params): Query<ListNotesQuery>,
    Extension(user): Extension<User>,
) -> Result<ResponseJson<Vec<Note>>, StatusCode> {
    let notes = notes_repository::list_notes(
        user.id,
        params.page.unwrap_or(0),
        params.limit.unwrap_or(20),
        params.tags,
        params.search,
    ).await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    
    Ok(ResponseJson(notes))
}

async fn create_note(
    Json(payload): Json<NewNote>,
    Extension(user): Extension<User>,
) -> Result<ResponseJson<Note>, StatusCode> {
    let mut new_note = payload;
    new_note.author_id = user.id;
    
    let note = notes_repository::create_note(new_note).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    
    // Publish event to NATS
    publish_note_event(NoteEvent::Created {
        note: note.clone(),
        author_id: user.id,
    }).await;
    
    Ok(ResponseJson(note))
}

async fn update_note(
    Path(note_id): Path<Uuid>,
    Json(payload): Json<UpdateNote>,
    Extension(user): Extension<User>,
) -> Result<ResponseJson<Note>, StatusCode> {
    // Check permissions
    let existing_note = notes_repository::get_note(note_id).await
        .map_err(|_| StatusCode::NOT_FOUND)?;
    
    if !can_edit_note(&existing_note, &user) {
        return Err(StatusCode::FORBIDDEN);
    }
    
    // Create revision before updating
    notes_repository::create_revision(&existing_note).await;
    
    let updated_note = notes_repository::update_note(note_id, payload).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    
    // Publish event to NATS
    publish_note_event(NoteEvent::Updated {
        note: updated_note.clone(),
        editor_id: user.id,
        previous_version: existing_note.version,
    }).await;
    
    Ok(ResponseJson(updated_note))
}

fn can_edit_note(note: &Note, user: &User) -> bool {
    note.author_id == user.id || note.collaborators.contains(&user.id)
}

pub fn app() -> Router {
    Router::new()
        .route("/notes", get(list_notes).post(create_note))
        .route("/notes/:id", get(get_note).patch(update_note).delete(delete_note))
        .route("/notes/:id/collaborators", post(add_collaborator).delete(remove_collaborator))
        .route("/notes/:id/revisions", get(list_revisions))
}
```

### Integrazione NATS

#### Event System
```rust
// src/events/mod.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum NoteEvent {
    Created {
        note: Note,
        author_id: Uuid,
    },
    Updated {
        note: Note,
        editor_id: Uuid,
        previous_version: i32,
    },
    Deleted {
        note_id: Uuid,
        deleted_by: Uuid,
    },
    CollaboratorAdded {
        note_id: Uuid,
        collaborator_id: Uuid,
        added_by: Uuid,
    },
}

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum AuthEvent {
    UserLoggedIn {
        user_id: Uuid,
        timestamp: DateTime<Utc>,
    },
    UserLoggedOut {
        user_id: Uuid,
        timestamp: DateTime<Utc>,
    },
}

pub async fn publish_note_event(event: NoteEvent) -> Result<(), Box<dyn std::error::Error>> {
    let client = get_nats_client().await?;
    let payload = serde_json::to_vec(&event)?;
    client.publish("notes.events", payload).await?;
    Ok(())
}

pub async fn publish_auth_event(event: AuthEvent) -> Result<(), Box<dyn std::error::Error>> {
    let client = get_nats_client().await?;
    let payload = serde_json::to_vec(&event)?;
    client.publish("auth.events", payload).await?;
    Ok(())
}

// Event handlers
pub async fn handle_note_events() {
    let client = get_nats_client().await.expect("Failed to connect to NATS");
    let mut subscriber = client.subscribe("notes.events").await.expect("Failed to subscribe");
    
    while let Some(message) = subscriber.next().await {
        if let Ok(event) = serde_json::from_slice::<NoteEvent>(&message.data) {
            match event {
                NoteEvent::Updated { note, .. } => {
                    // Broadcast to WebSocket clients
                    websocket_broadcast(WebSocketMessage::NoteUpdated { note }).await;
                    
                    // Update search index
                    update_search_index(&note).await;
                    
                    // Send notifications
                    send_collaboration_notifications(&note).await;
                },
                NoteEvent::Created { note, .. } => {
                    websocket_broadcast(WebSocketMessage::NoteCreated { note }).await;
                },
                _ => {}
            }
        }
    }
}
```

---

## Implementazione Frontend

### Componenti Principali

#### Editor Markdown Collaborativo
```svelte
<!-- src/lib/components/MarkdownEditor.svelte -->
<script>
    import { onMount, onDestroy } from 'svelte';
    import { websocketStore } from '$lib/stores/websocket.js';
    import { debounce } from '$lib/utils/debounce.js';
    
    export let noteId;
    export let content = '';
    export let readonly = false;
    
    let editor;
    let isTyping = false;
    let typingTimeout;
    let cursors = new Map(); // Cursori di altri utenti
    
    // Debounced save function
    const debouncedSave = debounce(async (newContent) => {
        if (newContent !== content) {
            await saveNote(newContent);
        }
        stopTyping();
    }, 1000);
    
    // Handle typing events
    function onInput(event) {
        const newContent = event.target.value;
        content = newContent;
        
        if (!isTyping) {
            startTyping();
        }
        
        // Send typing indicator
        $websocketStore.send({
            type: 'user_typing',
            noteId,
            userId: $authStore.user.id,
            position: event.target.selectionStart
        });
        
        debouncedSave(newContent);
    }
    
    function startTyping() {
        isTyping = true;
        clearTimeout(typingTimeout);
        typingTimeout = setTimeout(stopTyping, 3000);
    }
    
    function stopTyping() {
        isTyping = false;
        $websocketStore.send({
            type: 'user_stopped_typing',
            noteId,
            userId: $authStore.user.id
        });
    }
    
    async function saveNote(newContent) {
        try {
            await fetch(`/api/notes/${noteId}`, {
                method: 'PATCH',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${$authStore.token}`
                },
                body: JSON.stringify({ content: newContent })
            });
        } catch (error) {
            console.error('Failed to save note:', error);
            // Handle error (show notification, retry, etc.)
        }
    }
    
    // Handle real-time updates from other users
    function handleRemoteUpdate(data) {
        if (data.noteId === noteId && data.userId !== $authStore.user.id) {
            // Apply operational transformation here
            content = data.content;
        }
    }
    
    // Handle cursor positions
    function handleCursorUpdate(data) {
        if (data.noteId === noteId && data.userId !== $authStore.user.id) {
            cursors.set(data.userId, {
                position: data.position,
                user: data.user
            });
            cursors = cursors; // Trigger reactivity
        }
    }
    
    onMount(() => {
        // Subscribe to WebSocket events
        const unsubscribe = websocketStore.subscribe(ws => {
            if (ws.connected) {
                // Handle incoming messages
                ws.socket.addEventListener('message', (event) => {
                    const data = JSON.parse(event.data);
                    switch (data.type) {
                        case 'note_updated':
                            handleRemoteUpdate(data);
                            break;
                        case 'cursor_position':
                            handleCursorUpdate(data);
                            break;
                    }
                });
            }
        });
        
        return unsubscribe;
    });
    
    onDestroy(() => {
        clearTimeout(typingTimeout);
        stopTyping();
    });
</script>

<div class="editor-container">
    <div class="editor-toolbar">
        <button on:click={() => insertMarkdown('**', '**')} title="Bold">
            <strong>B</strong>
        </button>
        <button on:click={() => insertMarkdown('*', '*')} title="Italic