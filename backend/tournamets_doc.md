# 📋 **DISEÑO DEL SISTEMA DE TORNEOS - DOCUMENTACIÓN COMPLETA**

## 1. **VISIÓN GENERAL**

El sistema de torneos se integrará como una capa adicional sobre el sistema actual de partidas, reutilizando toda la lógica de juego existente en `pong-game-manager`. La gestión se dividirá en tres partes:
- **HTTP**: Operaciones CRUD (crear, unirse, invitar, aceptar)
- **WebSocket**: Comunicación en tiempo real durante el torneo activo
- **Manager**: Lógica de negocio y gestión del estado del torneo

## 2. **ARQUITECTURA DE BASE DE DATOS**

### **Nuevas Tablas:**

```/dev/null/schema.sql#L1-88
-- Tabla principal de torneos
CREATE TABLE IF NOT EXISTS tournaments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    creator_user_id INTEGER NOT NULL,
    max_participants INTEGER NOT NULL DEFAULT 8,
    min_participants INTEGER NOT NULL DEFAULT 4,
    status TEXT NOT NULL DEFAULT 'open',
    format TEXT NOT NULL DEFAULT 'single_elimination',
    rounds_total INTEGER,
    current_round INTEGER DEFAULT 0,
    winner_user_id INTEGER,
    -- Configuración del juego
    match_win_score INTEGER NOT NULL DEFAULT 5,
    match_max_time INTEGER DEFAULT 300,
    -- Timestamps
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    started_at DATETIME,
    ended_at DATETIME,
    FOREIGN KEY (creator_user_id) REFERENCES users(id),
    FOREIGN KEY (winner_user_id) REFERENCES users(id),
    CHECK (status IN ('open', 'in_progress', 'completed', 'cancelled')),
    CHECK (format IN ('single_elimination', 'double_elimination', 'round_robin'))
);

-- Participantes del torneo
CREATE TABLE IF NOT EXISTS tournament_participants (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tournament_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    seed_number INTEGER,
    status TEXT NOT NULL DEFAULT 'active',
    eliminated_round INTEGER,
    final_position INTEGER,
    joined_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tournament_id, user_id),
    FOREIGN KEY (tournament_id) REFERENCES tournaments(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id),
    CHECK (status IN ('active', 'eliminated', 'bye', 'waiting', 'disqualified'))
);

-- Invitaciones a torneos
CREATE TABLE IF NOT EXISTS tournament_invitations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tournament_id INTEGER NOT NULL,
    inviter_user_id INTEGER NOT NULL,
    invited_user_id INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    responded_at DATETIME,
    UNIQUE(tournament_id, invited_user_id),
    FOREIGN KEY (tournament_id) REFERENCES tournaments(id) ON DELETE CASCADE,
    FOREIGN KEY (inviter_user_id) REFERENCES users(id),
    FOREIGN KEY (invited_user_id) REFERENCES users(id),
    CHECK (status IN ('pending', 'accepted', 'rejected', 'expired'))
);

-- Relación entre torneos y partidas
CREATE TABLE IF NOT EXISTS tournament_matches (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tournament_id INTEGER NOT NULL,
    match_id INTEGER NOT NULL,
    round_number INTEGER NOT NULL,
    bracket_position INTEGER,
    next_match_id INTEGER,
    -- Control de tiempo para descalificación
    match_ready_deadline DATETIME,
    player1_ready BOOLEAN DEFAULT 0,
    player2_ready BOOLEAN DEFAULT 0,
    UNIQUE(match_id),
    FOREIGN KEY (tournament_id) REFERENCES tournaments(id) ON DELETE CASCADE,
    FOREIGN KEY (match_id) REFERENCES matches(id),
    FOREIGN KEY (next_match_id) REFERENCES tournament_matches(id)
);

-- Índices para optimización
CREATE INDEX IF NOT EXISTS idx_tournaments_status ON tournaments(status);
CREATE INDEX IF NOT EXISTS idx_tournaments_creator ON tournaments(creator_user_id);
CREATE INDEX IF NOT EXISTS idx_tournament_participants_tournament ON tournament_participants(tournament_id);
CREATE INDEX IF NOT EXISTS idx_tournament_participants_user ON tournament_participants(user_id);
CREATE INDEX IF NOT EXISTS idx_tournament_participants_status ON tournament_participants(status);
CREATE INDEX IF NOT EXISTS idx_tournament_matches_tournament ON tournament_matches(tournament_id);
CREATE INDEX IF NOT EXISTS idx_tournament_matches_round ON tournament_matches(round_number);
CREATE INDEX IF NOT EXISTS idx_tournament_invitations_invited ON tournament_invitations(invited_user_id);

-- Añadir el nuevo tipo de juego
INSERT OR IGNORE INTO game_types (name, min_players, max_players, supports_invitations)
VALUES ('pong_tournament', 2, 2, 0);
```

## 3. **ESTRUCTURA DE ARCHIVOS PROPUESTA**

```/dev/null/tree.txt#L1-70
backend/src/features/
├── tournaments/                     # Nueva feature principal
│   ├── manager/                    # Lógica de negocio del torneo
│   │   ├── domain/
│   │   │   ├── Tournament.entity.ts
│   │   │   ├── TournamentBracket.ts
│   │   │   ├── TournamentParticipant.ts
│   │   │   └── TournamentRound.ts
│   │   │
│   │   ├── services/
│   │   │   ├── TournamentService.ts
│   │   │   ├── BracketGenerator.ts
│   │   │   ├── MatchScheduler.ts
│   │   │   ├── DisqualificationService.ts
│   │   │   └── ITournamentService.interface.ts
│   │   │
│   │   ├── repositories/
│   │   │   ├── TournamentRepository.ts
│   │   │   ├── TournamentInvitationRepository.ts
│   │   │   └── ITournamentRepository.interface.ts
│   │   │
│   │   └── utils/
│   │       ├── TournamentValidator.ts
│   │       └── TournamentFormatter.ts
│   │
│   ├── http/                       # API REST para operaciones CRUD
│   │   ├── routes/
│   │   │   └── tournament.routes.ts
│   │   │
│   │   ├── controllers/
│   │   │   ├── TournamentController.ts
│   │   │   └── TournamentInvitationController.ts
│   │   │
│   │   ├── dto/
│   │   │   ├── CreateTournamentDto.ts
│   │   │   ├── JoinTournamentDto.ts
│   │   │   └── InvitationDto.ts
│   │   │
│   │   └── middleware/
│   │       └── TournamentValidation.middleware.ts
│   │
│   └── websocket/                  # Comunicación en tiempo real
│       ├── websocket/
│       │   └── tournament.websocket.ts
│       │
│       ├── services/
│       │   ├── TournamentWebSocketService.ts
│       │   ├── TournamentStateSync.ts
│       │   └── ITournamentWebSocketService.interface.ts
│       │
│       └── handlers/
│           ├── TournamentEventHandlers.ts
│           ├── MatchReadyHandler.ts
│           └── SpectatorHandler.ts
│
├── pong-game-manager/              # Feature existente
│   └── [sin cambios, se reutiliza completamente]
│
└── pong-websocket/                 # Feature existente
    └── [modificación mínima para notificar eventos de torneo]
```

## 4. **MODELO DE DOMINIO**

### **Entidad Principal y Tipos:**

```/dev/null/Tournament.types.ts#L1-75
// Entidad principal del torneo
interface Tournament {
    id: number;
    name: string;
    description?: string;
    creatorId: number;
    status: TournamentStatus;
    format: TournamentFormat;
    maxParticipants: number;
    minParticipants: number;
    currentRound: number;
    totalRounds?: number;
    participants: Map<number, TournamentParticipant>;
    bracket: TournamentBracket;
    activeMatches: Map<number, TournamentMatch>;
    winnerId?: number;
    // Configuración del juego
    matchConfig: {
        winScore: number;
        maxTime: number;
    };
}

enum TournamentStatus {
    OPEN = 'open',                 // Aceptando participantes
    IN_PROGRESS = 'in_progress',   // Torneo en curso
    COMPLETED = 'completed',       // Torneo finalizado
    CANCELLED = 'cancelled'        // Torneo cancelado
}

enum TournamentFormat {
    SINGLE_ELIMINATION = 'single_elimination'
    // Future: DOUBLE_ELIMINATION, ROUND_ROBIN
}

interface TournamentParticipant {
    userId: number;
    username: string;
    seed?: number;
    status: ParticipantStatus;
    eliminatedRound?: number;
    finalPosition?: number;
}

enum ParticipantStatus {
    ACTIVE = 'active',
    ELIMINATED = 'eliminated',
    BYE = 'bye',
    WAITING = 'waiting',
    DISQUALIFIED = 'disqualified'
}

interface TournamentMatch {
    matchId: number;
    roundNumber: number;
    player1Id: number;
    player2Id: number;
    bracketPosition: number;
    nextMatchId?: number;
    readyDeadline: Date;
    player1Ready: boolean;
    player2Ready: boolean;
    status: 'pending' | 'ready_phase' | 'in_progress' | 'completed';
}

// Restricciones del sistema
interface TournamentRestrictions {
    MAX_TOURNAMENTS_PER_USER: 1;
    READY_TIMEOUT_SECONDS: 120;  // 2 minutos para dar "listo"
    MIN_PARTICIPANTS: 4;
    MAX_PARTICIPANTS: 32;
}
```

## 5. **API HTTP - ENDPOINTS**

### **Rutas REST para operaciones CRUD:**

```/dev/null/tournament.routes.ts#L1-60
// POST /api/tournaments
// Crear un nuevo torneo
{
    name: string;
    description?: string;
    maxParticipants: number;
    minParticipants?: number;
    format: 'single_elimination';
    matchWinScore: number;    // Puntos para ganar
    matchMaxTime?: number;    // Tiempo máximo por partida
}

// GET /api/tournaments
// Listar todos los torneos (abiertos, en progreso, completados)
// Query params: ?status=open&page=1&limit=10

// GET /api/tournaments/:id
// Obtener detalles de un torneo específico

// POST /api/tournaments/:id/join
// Unirse a un torneo abierto
// Body: {} (vacío, usa el userId del token)

// DELETE /api/tournaments/:id/leave
// Abandonar un torneo (solo si está en estado OPEN)

// POST /api/tournaments/:id/start
// Iniciar el torneo (solo el creador, cuando min_participants alcanzado)

// POST /api/tournaments/:id/invite
// Enviar invitación a un torneo
{
    invitedUserId: number;
}

// GET /api/tournaments/invitations
// Listar invitaciones pendientes del usuario autenticado

// POST /api/tournaments/invitations/:id/accept
// Aceptar una invitación

// POST /api/tournaments/invitations/:id/reject
// Rechazar una invitación

// GET /api/tournaments/active
// Obtener el torneo activo del usuario (si existe)

// POST /api/tournaments/:id/cancel
// Cancelar un torneo (solo el creador, si está OPEN)

// Validaciones importantes:
// - Un usuario no puede crear un torneo si ya tiene uno activo
// - Un usuario no puede unirse si ya está en otro torneo activo
// - Un usuario no puede unirse si ya tiene una partida en curso
// - El creador no puede abandonar su propio torneo
```

## 6. **COMUNICACIÓN WEBSOCKET**

### **Mensajes WebSocket (Solo para torneos activos):**

```/dev/null/websocket-messages.ts#L1-90
// Cliente → Servidor
interface TournamentWebSocketMessage {
    type: TournamentMessageType;
    payload: any;
    token: string;  // JWT para autenticación
}

enum TournamentMessageType {
    // Conexión inicial
    AUTHENTICATE = 'authenticate',

    // Estado del torneo
    GET_STATE = 'get_state',
    GET_BRACKET = 'get_bracket',

    // Durante partidas
    PLAYER_READY = 'player_ready',        // Confirmar listo para jugar
    REQUEST_MATCH_INFO = 'request_match_info',

    // Modo espectador
    SPECTATE_TOURNAMENT = 'spectate_tournament',
    STOP_SPECTATING = 'stop_spectating'
}

// Servidor → Cliente
interface TournamentWebSocketResponse {
    type: TournamentEventType;
    status: 'success' | 'error' | 'info';
    data?: any;
    error?: string;
    timestamp: number;
}

enum TournamentEventType {
    // Eventos de conexión
    AUTH_SUCCESS = 'auth_success',
    AUTH_ERROR = 'auth_error',

    // Estados del torneo
    TOURNAMENT_STATE = 'tournament_state',
    BRACKET_UPDATE = 'bracket_update',

    // Eventos de ronda
    ROUND_STARTED = 'round_started',
    ROUND_COMPLETED = 'round_completed',

    // Eventos de partida
    MATCH_READY_PHASE = 'match_ready_phase',    // Tienes 2 min para dar listo
    MATCH_STARTING = 'match_starting',
    MATCH_COMPLETED = 'match_completed',
    OPPONENT_READY = 'opponent_ready',

    // Descalificaciones
    PLAYER_DISQUALIFIED = 'player_disqualified',
    YOU_WERE_DISQUALIFIED = 'you_were_disqualified',

    // BYE y espera
    YOU_HAVE_BYE = 'you_have_bye',
    WAITING_FOR_NEXT_ROUND = 'waiting_for_next_round',

    // Final del torneo
    TOURNAMENT_COMPLETED = 'tournament_completed',
    TOURNAMENT_CANCELLED = 'tournament_cancelled',

    // Actualizaciones para espectadores
    SPECTATOR_UPDATE = 'spectator_update'
}

// Ejemplos de payloads específicos
interface MatchReadyPhasePayload {
    matchId: number;
    opponentName: string;
    deadline: string;  // ISO timestamp
    yourStatus: 'not_ready' | 'ready';
    opponentStatus: 'not_ready' | 'ready';
}

interface DisqualificationPayload {
    reason: 'no_ready' | 'abandoned' | 'timeout';
    round: number;
    disqualifiedUserId: number;
}

interface BracketUpdatePayload {
    currentRound: number;
    totalRounds: number;
    matches: Array<{
        roundNumber: number;
        player1: { id: number; name: string; status: string };
        player2: { id: number; name: string; status: string };
        winnerId?: number;
    }>;
}
```

## 7. **FLUJO COMPLETO DEL SISTEMA**

### **Diagrama de Estados y Transiciones:**

```/dev/null/tournament-flow.md#L1-80
## FLUJO COMPLETO DEL TORNEO

### 1. FASE DE CREACIÓN Y REGISTRO
```
[Usuario] --HTTP POST--> [/api/tournaments]
    → Valida: no tiene torneo/partida activa
    → Crea torneo en BD (status: OPEN)
    → Retorna tournamentId

[Otros usuarios] --HTTP POST--> [/api/tournaments/:id/join]
    → Valida: no tiene torneo/partida activa
    → Añade a tournament_participants
    → Si alcanza min_participants → Notifica al creador
```

### 2. INICIO DEL TORNEO
```
[Creador] --HTTP POST--> [/api/tournaments/:id/start]
    → Valida: min_participants alcanzado
    → Genera brackets (BracketGenerator)
    → Crea partidas de Ronda 1 en BD
    → Status: IN_PROGRESS
    → Notifica a todos los participantes vía WebSocket
```

### 3. FASE DE PARTIDAS (WebSocket)
```
[Participantes] --WS Connect--> [tournament.websocket]
    → Autenticación con JWT
    → Recibe TOURNAMENT_STATE actual

Para cada ronda:
    [Sistema] → Envía ROUND_STARTED a todos

    Para cada partida de la ronda:
        [Sistema] → Envía MATCH_READY_PHASE a los 2 jugadores
        [Timer] → 2 minutos para confirmar

        [Jugador] --WS PLAYER_READY--> [tournament.websocket]
            → Marca player_ready = true en BD
            (Esto se hace con el is_ready en pong websocket)

        Si ambos listos:
            → Crea partida en pong-game-manager
            → Jugadores juegan en pong-websocket

        Si timeout (2 min):
            → Jugador no listo = DISQUALIFIED
            → Oponente gana por W.O.
            → Actualiza bracket
```

### 4. PROGRESIÓN ENTRE RONDAS
```
[pong-websocket] --Event--> [tournament-websocket]
    → match:completed con winnerId
    → Actualiza tournament_matches
    → Elimina perdedor (single elimination)
    → Verifica si ronda completa

Si ronda completa:
    → Genera siguiente ronda
    → Asigna BYEs si jugadores impares
    → Notifica ROUND_COMPLETED + ROUND_STARTED
```

### 5. FINALIZACIÓN
```
Cuando queda 1 jugador:
    → Status: COMPLETED
    → Asigna winner_user_id
    → Calcula posiciones finales
    → Envía TOURNAMENT_COMPLETED a todos
```

### 6. CASOS ESPECIALES

**Abandono durante partida:**
- Se maneja en pong-websocket normalmente
- tournament-websocket recibe el evento y actualiza

**Desconexión temporal:**
- WebSocket se desconecta pero mantiene estado
- Puede reconectar sin penalización
- La partida sigue en pong-game-manager

**Sistema de BYE:**
- Jugador con mejor seed recibe BYE
- Notificación: YOU_HAVE_BYE
- Pasa automático a siguiente ronda
```

## 8. **INTEGRACIÓN CON SISTEMA EXISTENTE**

### **Patrón de Comunicación entre Módulos:**

```/dev/null/integration.ts#L1-55
// === En tournament-websocket ===
class TournamentWebSocketService {
    constructor(fastify: FastifyInstance) {
        // Escucha eventos del pong-websocket
        this.setupPongEventListeners();
    }

    private setupPongEventListeners() {
        // Registra listeners para eventos de pong
        fastify.pongEvents.on('match:completed', this.handleMatchCompleted);
        fastify.pongEvents.on('match:abandoned', this.handleMatchAbandoned);
        fastify.pongEvents.on('player:disconnected', this.handlePlayerDisconnected);
    }

    private handleMatchCompleted = async (data: {
        matchId: number;
        winnerId: number;
        scores: { player1: number; player2: number }
    }) => {
        // Verificar si es partida de torneo
        const tournamentMatch = await this.getTournamentMatch(data.matchId);
        if (!tournamentMatch) return;

        // Actualizar bracket
        await this.updateBracket(tournamentMatch, data.winnerId);

        // Verificar progresión del torneo
        await this.checkRoundCompletion(tournamentMatch.tournamentId);
    };
}

// === En pong-websocket (modificación mínima) ===
// Añadir en handleMatchEnd o similar:
if (match.game_type === 'pong_tournament') {
    fastify.pongEvents?.emit('match:completed', {
        matchId: match.id,
        winnerId: winner.id,
        scores: finalScores
    });
}

// === Sistema de Coordinación ===
class TournamentMatchCoordinator {
    async createTournamentMatch(
        tournamentId: number,
        player1Id: number,
        player2Id: number,
        roundNumber: number
    ) {
        // 1. Crear entrada en matches con tipo 'pong_tournament'
        const match = await this.matchRepository.create({
            game_type_id: GAME_TYPES.PONG_TOURNAMENT,
            status: 'pending'
        });

        // 2. Crear relación en tournament_matches
        // 3. Establecer deadline para ready (2 minutos)
        // 4. Notificar a jugadores vía WebSocket
    }
}
```

## 9. **SISTEMA DE DESCALIFICACIÓN Y TIMEOUTS**

```/dev/null/disqualification.ts#L1-40
class DisqualificationService {
    private readonly READY_TIMEOUT = 120; // 2 minutos en segundos

    async startReadyTimer(tournamentMatch: TournamentMatch) {
        const deadline = new Date(Date.now() + this.READY_TIMEOUT * 1000);

        // Guardar deadline en BD
        await this.updateMatchDeadline(tournamentMatch.id, deadline);

        // Programar verificación
        setTimeout(async () => {
            await this.checkReadyStatus(tournamentMatch);
        }, this.READY_TIMEOUT * 1000);
    }

    async checkReadyStatus(tournamentMatch: TournamentMatch) {
        const match = await this.getMatchStatus(tournamentMatch.id);

        // Descalificar a quien no esté listo
        if (!match.player1Ready && !match.player2Ready) {
            // Ambos descalificados, match cancelado
            await this.disqualifyBoth(match);
        } else if (!match.player1Ready) {
            await this.disqualifyPlayer(match.player1Id, 'no_ready');
            await this.declareWinner(match.player2Id);
        } else if (!match.player2Ready) {
            await this.disqualifyPlayer(match.player2Id, 'no_ready');
            await this.declareWinner(match.player1Id);
        }
    }

    async disqualifyPlayer(userId: number, reason: string) {
        // 1. Actualizar estado en tournament_participants
        // 2. Notificar vía WebSocket
        // 3. Registrar en logs
    }
}
```

## 10. **VALIDACIONES Y RESTRICCIONES**

```/dev/null/validations.ts#L1-65
class TournamentValidator {
    // Verificar si el usuario tiene un torneo activo
    async getUserActiveTournament(userId: number): Promise<Tournament | null> {
        // Opción 1: Es creador de un torneo activo
        const createdTournament = await this.db.query(`
            SELECT * FROM tournaments
            WHERE creator_user_id = ?
            AND status IN ('open', 'in_progress')
            LIMIT 1
        `, [userId]);

        if (createdTournament) return createdTournament;

        // Opción 2: Es participante en un torneo activo
        const participatingTournament = await this.db.query(`
            SELECT t.* FROM tournaments t
            JOIN tournament_participants tp ON t.id = tp.tournament_id
            WHERE tp.user_id = ?
            AND t.status IN ('open', 'in_progress')
            AND tp.status NOT IN ('eliminated', 'disqualified')
            LIMIT 1
        `, [userId]);

        return participatingTournament || null;
    }

    // Antes de crear torneo
    async canCreateTournament(userId: number): Promise<boolean> {
        // No tiene otro torneo activo
        const activeTournament = await this.getUserActiveTournament(userId);
        if (activeTournament) return false;

        // No tiene partida en curso
        const activeMatch = await this.getActiveMatch(userId);
        if (activeMatch) return false;

        return true;
    }

    // Antes de unirse
    async canJoinTournament(userId: number, tournamentId: number): Promise<boolean> {
        // Torneo está OPEN
        const tournament = await this.getTournament(tournamentId);
        if (tournament.status !== 'open') return false;

        // No excede max_participants
        const participantCount = await this.getParticipantCount(tournamentId);
        if (participantCount >= tournament.maxParticipants) return false;

        // No está ya en el torneo
        const isParticipant = await this.isUserInTournament(userId, tournamentId);
        if (isParticipant) return false;

        // No tiene otro torneo activo
        const activeTournament = await this.getUserActiveTournament(userId);
        if (activeTournament) return false;

        // No tiene partida en curso
        const activeMatch = await this.getActiveMatch(userId);
        if (activeMatch) return false;

        return true;
    }

    // Durante el torneo, un jugador no puede:
    // - Crear otro torneo
    // - Unirse a otro torneo
    // - Iniciar partida normal de pong
    // - Aceptar invitaciones a partidas normales
}
```

## 11. **MODO ESPECTADOR**

```/dev/null/spectator.ts#L1-35
class SpectatorHandler {
    // Cualquier usuario puede espectar un torneo
    async handleSpectateRequest(
        userId: number,
        tournamentId: number,
        socket: WebSocket
    ) {
        const tournament = await this.getTournament(tournamentId);

        // Verificar que el torneo existe y está activo
        if (!tournament || tournament.status === 'cancelled') {
            return this.sendError(socket, 'Tournament not available');
        }

        // Determinar rol del usuario
        const isParticipant = await this.isUserParticipant(userId, tournamentId);

        if (isParticipant) {
            // Enviar estado completo con información privada
            await this.sendParticipantView(socket, tournament, userId);
        } else {
            // Enviar vista de espectador (solo bracket público)
            await this.sendSpectatorView(socket, tournament);

            // Suscribir a actualizaciones públicas
            this.subscribeToPublicUpdates(socket, tournamentId);
        }
    }

    // Los espectadores reciben:
    // - Actualizaciones del bracket
    // - Resultados de partidas
    // - Progresión de rondas
    // NO reciben: mensajes de ready, timeouts individuales
}
```

## 12. **CONSULTAS SQL ÚTILES**

```/dev/null/queries.sql#L1-45
-- Verificar si usuario tiene torneo activo (como creador o participante)
SELECT t.* FROM tournaments t
LEFT JOIN tournament_participants tp ON t.id = tp.tournament_id
WHERE (t.creator_user_id = ? OR tp.user_id = ?)
AND t.status IN ('open', 'in_progress')
LIMIT 1;

-- Obtener todos los participantes activos de un torneo
SELECT u.id, u.username, tp.*
FROM tournament_participants tp
JOIN users u ON tp.user_id = u.id
WHERE tp.tournament_id = ?
AND tp.status IN ('active', 'bye', 'waiting')
ORDER BY tp.seed_number;

-- Verificar partidas activas del usuario
SELECT m.* FROM matches m
JOIN match_players mp ON m.id = mp.match_id
WHERE mp.user_id = ?
AND m.status IN ('pending', 'in_progress');

-- Obtener bracket completo del torneo
SELECT tm.*, m.status as match_status,
       mp1.user_id as player1_id, mp2.user_id as player2_id,
       u1.username as player1_name, u2.username as player2_name
FROM tournament_matches tm
JOIN matches m ON tm.match_id = m.id
LEFT JOIN match_players mp1 ON m.id = mp1.match_id AND mp1.id = (
    SELECT MIN(id) FROM match_players WHERE match_id = m.id
)
LEFT JOIN match_players mp2 ON m.id = mp2.match_id AND mp2.id = (
    SELECT MAX(id) FROM match_players WHERE match_id = m.id
)
LEFT JOIN users u1 ON mp1.user_id = u1.id
LEFT JOIN users u2 ON mp2.user_id = u2.id
WHERE tm.tournament_id = ?
ORDER BY tm.round_number, tm.bracket_position;

-- Limpiar torneos abandonados (mantenimiento)
UPDATE tournaments
SET status = 'cancelled', ended_at = CURRENT_TIMESTAMP
WHERE status = 'open'
AND created_at < datetime('now', '-24 hours');
```

## 13. **VENTAJAS DEL DISEÑO ACTUALIZADO**

1. **Sin redundancia en BD**: Usamos el estado del torneo existente en lugar de tablas adicionales
2. **Consultas eficientes**: Los índices permiten verificación rápida de torneos activos
3. **Separación clara HTTP/WebSocket**: Operaciones CRUD por HTTP, tiempo real por WebSocket
4. **Reutilización máxima**: El sistema de juego existente no necesita cambios
5. **Control de concurrencia**: El estado del torneo controla las restricciones
6. **Sistema robusto de timeouts**: Descalificación automática por no presentarse
