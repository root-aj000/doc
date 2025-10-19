This TypeScript file acts as the primary entry point and configuration hub for a **real-time, collaborative server application** built on Node.js. It leverages `http` for basic web requests and `Socket.IO` for bidirectional, event-driven communication, making it ideal for applications like live chat, collaborative editing, or multiplayer games.

The core purpose is to:
1.  **Initialize and configure an HTTP server** that also hosts a **Socket.IO server**.
2.  **Manage real-time connections**, including authentication and event handling.
3.  **Provide a "room" system** for grouping connected clients.
4.  **Implement robust logging and error handling** for stability.
5.  **Enable graceful shutdown** to prevent data loss or abrupt disconnections.
6.  **Handle standard HTTP requests**, such as health checks.

---

## Simplified Logic Breakdown

Imagine this file as the central control room for a bustling online community.

*   **The Building (HTTP Server):** This is the basic web server that listens for incoming traffic. It's like the main entrance to our online community.
*   **The Communication System (Socket.IO Server):** This is a specialized, super-fast messaging system installed *within* our building. It allows people (clients) to send and receive real-time updates without constantly refreshing.
*   **The Bouncer (Authentication Middleware):** Before anyone can use our special communication system, they have to show their ID. This bouncer (`authenticateSocket`) verifies who they are. If valid, they get a special pass.
*   **The Room Organizer (RoomManager):** Our community has different "rooms" (e.g., "General Chat," "Project Alpha"). The `RoomManager` is responsible for keeping track of who's in which room and helping them communicate only with others in that room.
*   **The Event Handlers (setupAllHandlers):** Once inside and authenticated, people can do things: "send a message," "join a room," "draw something." These handlers are the specific instructions for what to do when someone performs one of these actions.
*   **The Watchdog (Logger & Error Handlers):** We have staff (`logger`) constantly monitoring what's happening, reporting any issues, and ensuring everything runs smoothly. If something goes wrong (`uncaughtException`, `unhandledRejection`), they know how to handle it so the whole system doesn't crash.
*   **The Welcome Desk (HTTP Handler):** Besides the real-time communication, there might be a few standard requests, like someone asking, "Is this server still alive?" (`/health`). The `httpHandler` takes care of these simple, non-real-time requests.
*   **The Port (PORT):** This is the specific "address" on the internet where our building (server) can be found.
*   **The Shutdown Procedure (SIGINT, SIGTERM):** When it's time to close the community for maintenance or upgrades, we have a polite way to tell everyone to leave and ensure all ongoing conversations are neatly wrapped up before turning off the lights.

---

## Line-by-Line Explanation

Let's break down the code step by step:

```typescript
import { createServer } from 'http'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
import { createSocketIOServer } from '@/socket-server/config/socket'
import { setupAllHandlers } from '@/socket-server/handlers'
import { type AuthenticatedSocket, authenticateSocket } from '@/socket-server/middleware/auth'
import { RoomManager } from '@/socket-server/rooms/manager'
import { createHttpHandler } from '@/socket-server/routes/http'
```
These lines import necessary modules and custom utilities from various parts of the project:
*   `createServer` from `http`: Node.js's built-in module for creating an HTTP server.
*   `env`: A utility to access environment variables (e.g., `PORT`, `DATABASE_URL`).
*   `createLogger`: A custom function to create a logger instance for structured logging.
*   `createSocketIOServer`: A function to configure and instantiate the Socket.IO server.
*   `setupAllHandlers`: A function that will attach all application-specific Socket.IO event listeners (e.g., `join-room`, `send-message`) to a connected socket.
*   `AuthenticatedSocket`, `authenticateSocket`: `AuthenticatedSocket` is a TypeScript type extending a basic Socket.IO socket with authentication-related properties (like user ID). `authenticateSocket` is a middleware function that will verify a client's identity.
*   `RoomManager`: A class responsible for managing "rooms" where clients can group together for targeted communication.
*   `createHttpHandler`: A function to create an Express-like handler for standard HTTP requests (e.g., health checks).

```typescript
const logger = createLogger('CollaborativeSocketServer')
```
Initializes a logger instance specifically named 'CollaborativeSocketServer'. This helps in tracing logs back to this core server file.

```typescript
// Enhanced server configuration - HTTP server will be configured with handler after all dependencies are set up
const httpServer = createServer()
```
Creates a barebones Node.js HTTP server instance. At this point, it's just an object; it's not yet listening for any connections. Socket.IO will attach itself to this server.

```typescript
const io = createSocketIOServer(httpServer)
```
Initializes the Socket.IO server, passing the `httpServer` instance to it. This means the Socket.IO server will "piggyback" on the HTTP server, listening on the same port and handling WebSocket upgrades.

```typescript
// Initialize room manager after io is created
const roomManager = new RoomManager(io)
```
Creates an instance of `RoomManager`, passing the Socket.IO server instance (`io`) to it. This allows the `RoomManager` to interact directly with connected sockets and emit events to specific rooms.

```typescript
io.use(authenticateSocket)
```
Applies the `authenticateSocket` middleware to the Socket.IO server. This middleware will run for *every new incoming Socket.IO connection* before any other event handlers are processed. Its job is to verify the client's identity and attach authentication information (like `userId`) to the `socket` object if successful. If authentication fails, it can prevent the connection.

```typescript
const httpHandler = createHttpHandler(roomManager, logger)
httpServer.on('request', httpHandler)
```
*   `const httpHandler = createHttpHandler(roomManager, logger)`: Creates a handler function for regular HTTP requests. This handler likely contains logic for routes that are not Socket.IO based, such as `/health` endpoints or simple API routes. It's passed the `roomManager` and `logger` so it can potentially interact with rooms or log specific HTTP requests.
*   `httpServer.on('request', httpHandler)`: Attaches the `httpHandler` to the `httpServer`. This means whenever a standard HTTP request comes in (e.g., a browser navigating to a URL on this server), this handler will be executed.

---

### Robust Error Handling

These blocks are crucial for making the server resilient to unexpected issues.

```typescript
process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception:', error)
  // Don't exit in production, just log
})
```
This listens for "uncaught exceptions" – errors that occur synchronously and aren't caught by `try...catch` blocks. It logs the error. The comment suggests that in a production environment, the server *might* continue running to prevent a full crash, but it's important to fix such exceptions quickly. In development, you might want `process.exit(1)` here to ensure immediate awareness of the bug.

```typescript
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection at:', promise, 'reason:', reason)
})
```
This listens for "unhandled promise rejections" – asynchronous errors from promises that don't have a `.catch()` block. It logs the rejection, which is vital for debugging asynchronous code.

```typescript
httpServer.on('error', (error) => {
  logger.error('HTTP server error:', error)
})
```
This specifically catches errors related to the `httpServer` itself, such as the port being in use when the server tries to start, or other network-related issues.

```typescript
io.engine.on('connection_error', (err) => {
  logger.error('Socket.IO connection error:', {
    req: err.req?.url,
    code: err.code,
    message: err.message,
    context: err.context,
  })
})
```
This captures errors at the **Engine.IO** layer, which is the low-level transport layer that Socket.IO uses (handling WebSockets, long polling, etc.). This often indicates issues with the initial connection handshake before a full Socket.IO connection is established (e.g., invalid requests, transport errors). It logs detailed information about the error.

---

### Socket.IO Connection & Request Logging

```typescript
io.on('connection', (socket: AuthenticatedSocket) => {
  logger.info(`New socket connection: ${socket.id}`)

  setupAllHandlers(socket, roomManager)
})
```
This is the **heart of the Socket.IO application logic**.
*   `io.on('connection', ...)`: This event fires *every time* a new client successfully connects and authenticates with the Socket.IO server.
*   `(socket: AuthenticatedSocket)`: The callback receives the newly connected `socket` object, which is typed as `AuthenticatedSocket` (meaning it has passed through the authentication middleware and contains user data).
*   `logger.info(...)`: Logs that a new connection has been established, including the unique ID of the socket.
*   `setupAllHandlers(socket, roomManager)`: This crucial function call attaches *all the custom event listeners* (e.g., 'join-room', 'send-message', 'draw-update') to this specific `socket`. This makes the socket ready to receive and respond to application-specific events.

```typescript
httpServer.on('request', (req, res) => {
  logger.info(`🌐 HTTP Request: ${req.method} ${req.url}`, {
    method: req.method,
    url: req.url,
    userAgent: req.headers['user-agent'],
    origin: req.headers.origin,
    host: req.headers.host,
    timestamp: new Date().toISOString(),
  })
})
```
This is a *separate* `request` handler for the `httpServer`. While `httpHandler` (defined earlier) handles the *logic* for HTTP routes, this one is specifically for **logging every incoming HTTP request**. It logs useful details like method, URL, user agent, and origin, which is invaluable for debugging and monitoring.

```typescript
io.engine.on('connection_error', (err) => {
  logger.error('❌ Engine.IO Connection error:', {
    code: err.code,
    message: err.message,
    context: err.context,
    req: err.req
      ? {
          url: err.req.url,
          method: err.req.method,
          headers: err.req.headers,
        }
      : 'No request object',
  })
})
```
This is another `io.engine.on('connection_error')` handler. It's perfectly fine to have multiple handlers for the same event; they will all execute. This second handler provides *even more detailed* logging for Engine.IO connection errors, specifically including details about the original HTTP request if available (URL, method, headers). This level of detail is extremely helpful when troubleshooting connection problems.

---

### Server Startup

```typescript
const PORT = Number(env.PORT || env.SOCKET_PORT || 3002)
```
Determines the port the server will listen on. It prioritizes:
1.  `env.PORT` (general application port)
2.  `env.SOCKET_PORT` (specific port for the socket server)
3.  Defaults to `3002` if neither environment variable is set.

```typescript
logger.info('Starting Socket.IO server...', {
  port: PORT,
  nodeEnv: env.NODE_ENV,
  hasDatabase: !!env.DATABASE_URL,
  hasAuth: !!env.BETTER_AUTH_SECRET,
})
```
Logs important configuration details when the server is about to start. This provides a quick overview of the server's environment and enabled features.

```typescript
httpServer.listen(PORT, '0.0.0.0', () => {
  logger.info(`Socket.IO server running on port ${PORT}`)
  logger.info(`🏥 Health check available at: http://localhost:${PORT}/health`)
})
```
*   `httpServer.listen(PORT, '0.0.0.0', ...)`: This is where the HTTP server (and by extension, the Socket.IO server) actually starts listening for incoming connections on the specified `PORT`. `'0.0.0.0'` means it will listen on all available network interfaces, making it accessible from outside the local machine (e.g., within a Docker container or from another computer on the network).
*   The callback function executes once the server has successfully started, logging confirmation and suggesting a health check URL.

```typescript
httpServer.on('error', (error) => {
  logger.error('❌ Server failed to start:', error)
  process.exit(1)
})
```
This is a critical error handler specifically for the `listen` call. If the server *fails to start* (e.g., the port is already in use by another application), it logs the error and then `process.exit(1)` immediately terminates the process with an error code, indicating a critical failure.

---

### Graceful Shutdown

These handlers ensure the server shuts down cleanly, preventing abrupt disconnections for clients and potential data loss.

```typescript
process.on('SIGINT', () => {
  logger.info('Shutting down Socket.IO server...')
  httpServer.close(() => {
    logger.info('Socket.IO server closed')
    process.exit(0)
  })
})
```
*   `process.on('SIGINT', ...)`: Catches the `SIGINT` signal, typically sent when a user presses `Ctrl+C` in the terminal.
*   `httpServer.close(...)`: Instructs the HTTP server to stop accepting new connections and close existing ones gracefully. The callback function executes once all connections are closed.
*   `process.exit(0)`: Exits the Node.js process with a success code (`0`).

```typescript
process.on('SIGTERM', () => {
  logger.info('Shutting down Socket.IO server...')
  httpServer.close(() => {
    logger.info('Socket.IO server closed')
    process.exit(0)
  })
})
```
*   `process.on('SIGTERM', ...)`: Catches the `SIGTERM` signal, typically sent by process managers (like `kill` command, `systemctl`, Docker, Kubernetes) to request a graceful termination.
*   The logic is identical to `SIGINT`, ensuring a consistent graceful shutdown procedure regardless of the trigger.

---

In summary, this file is a comprehensive boilerplate for a robust, real-time backend, carefully setting up communication channels, managing state, handling errors, and ensuring a smooth operational lifecycle.