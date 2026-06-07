# Exam Project – P2P Tic-Tac-Toe Game with Java RMI

This branch contains the updated version of the **Distributed Algorithms** exam project:
a **peer-to-peer** application for playing **tic-tac-toe** through:

- **Java RMI** for communication
- an **automatic peer discovery** system via UDP Multicast
- an optional **epidemic gossip** mechanism to quickly propagate membership
- a **distributed matchmaking** system with no central server
- game management based on a **logical token**
- a **rematch** protocol with a coordinator elected through symmetry breaking

---

## Project Structure

The main files are:

- `AutoPeerMain.java` – application entry point
- `PeerService.java` – RMI interface with the remote methods exposed by the peers
- `PeerImpl.java` – concrete peer implementation (matchmaking, gameplay, rematch)
- `Discovery.java` – peer discovery via UDP multicast + (optional) triggered gossip
- `GameBoard.java` – management of the 3×3 tic-tac-toe board

All files are located in the package:

```java
package gamep2p;
```

---

## System Model

Every process started through `AutoPeerMain` is a **distributed peer**:

* it exposes an RMI remote object
* it calls remote methods on other peers
* it takes part in **distributed discovery**
* it automatically enters **matchmaking**
* it plays through a **token**, which guarantees mutual exclusion over the turn
* it handles shutdown and rematch through a small consensus protocol

There is no central server or authority:
all peers are identical.

Each peer has a global identifier:

```
myId = host:port
```

used to:

* break symmetry during matchmaking,
* decide who starts the game,
* decide who coordinates the rematch phase,
* deterministically select the opponent.

---

## Files and Responsibilities

### `AutoPeerMain.java` – Peer Startup

Main responsibilities:

* argument parsing:

  ```bash
  java gamep2p.AutoPeerMain [host] [port]
  ```

* selecting a free port if none is specified

* peer creation:

  ```java
  PeerImpl peer = new PeerImpl(host, port);
  ```

* startup and binding in the RMI registry:

  * creating the registry on the port
  * registration:

    ```
    rmi://host:port/peer
    ```

* the main thread then stays idle (the peer uses its own threads)

---

### `PeerService.java` – RMI Interface

Defines the remote operations a peer can invoke on another.

### State Methods

* `boolean ping()`
* `String getId()`
* `boolean isInGame()`

### Matchmaking

* `boolean proposeMatch(String proposerId)`
* `void confirmMatch(String opponentId, boolean iStart, char symbol)`

### Gameplay

* `void receiveToken()`
* `void updateMove(int row, int col, char symbol, char result)`

### Rematch

* `boolean getRematchDecision()`
* `void startRematch(boolean iStart, char mySymbol)`
* `void noRematch()`

These are the only methods actually invoked over the network; this is why they appear in the Class Diagram.

---

### `GameBoard.java` – Tic-Tac-Toe Logic

Handles:

* move validation
* symbol placement
* state checking (`X`, `O`, `D`, ` `)
* user-friendly rendering of the board

It is entirely local and independent of the network.

---

### `Discovery.java` – Peer Discovery (HELLO + Optional Gossip)

Discovery maintains the list of known IDs through:

#### ✔ Periodic HELLO (always active)

Each peer sends:

```
HELLO <myId>
```

where `<myId>` is the `host:port` string.

The receiver updates the list of active peers.

#### ✔ Optional Gossip (enabled via a boolean in the constructor)

The features of the updated version are:

* it is **not** periodic
* it is sent **only when the view changes** (a new peer is discovered)
* it is **unicast gossip**, not multicast:

  * lower network usage
  * less congestion
* the payload contains a timestamp (`id;ts`) and is propagated as long as needed

There is also a **periodic cleaner** that removes peers that are no longer active.

Discovery is used as a "black box" by `PeerImpl` to obtain the up-to-date list of possible opponents.

---

### `PeerImpl.java` – Complete Distributed Protocol

This class contains all the peer logic:

### State

* identifier
* game board
* discovery
* remote opponent + RMI stub
* assigned symbols
* token (bool)
* rematch and automatic matchmaking
* thread scheduler

### Distributed Matchmaking

It periodically performs:

1. retrieval of the view from discovery
2. filtering of live and free peers (`ping`, `isInGame`)
3. symmetry breaking: the **lexicographic successor** is selected
4. sending the match proposal
5. if accepted → game setup

In addition, it:

* prevents immediate matches with the last opponent (`lastOpponentId`);
* handles failures during lookups and remote calls.

### Token-based Game

Whoever holds the token:

* can make the move
* communicates it via RMI
* passes the token to the opponent at the end

This is a simple and effective form of distributed mutual exclusion.

### Rematch (two-party mini-consensus)

The coordinator (the smaller ID):

1. collects both the local and the remote decision
2. if both are positive → `startRematch`
3. otherwise → `noRematch`

The non-coordinator simply provides its own decision and waits.

---

## Compilation and Execution

### Requirements

* Java 8+
* A local network supporting UDP Multicast

### Compilation

```bash
javac gamep2p/*.java
```

### Execution

```bash
java gamep2p.AutoPeerMain
```
or

```bash
java gamep2p.AutoPeerMain <host> [port]
```

Example:

```bash
java gamep2p.AutoPeerMain 192.168.1.20 5001
java gamep2p.AutoPeerMain 192.168.1.20 5002
```

## Author

**Simone Candiani**
Master's Degree in Computer Science – UNIMORE
