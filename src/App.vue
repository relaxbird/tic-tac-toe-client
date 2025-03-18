<template>
  <div class="container mt-4">
    <h1 class="text-center mb-4">Tic-Tac-Toe Online</h1>
    
    <!-- Username Input -->
    <div v-if="!username" class="username-form text-center">
      <div class="card mx-auto" style="max-width: 400px;">
        <div class="card-body">
          <h3 class="card-title">Enter Your Username</h3>
          <div class="form-group mb-3">
            <input 
              type="text" 
              class="form-control" 
              v-model="usernameInput" 
              placeholder="Username"
              @keyup.enter="setUsername"
            >
          </div>
          <button class="btn btn-primary" @click="setUsername">Play</button>
        </div>
      </div>
    </div>
    
    <!-- Room Selection -->
    <div v-else-if="!currentRoom" class="room-selection">
      <div class="row">
        <div class="col-md-3" v-for="room in rooms" :key="room.id">
          <div class="card mb-4">
            <div class="card-body text-center">
              <h5 class="card-title">Room {{ room.id }}</h5>
              <p class="card-text">
                Players: {{ room.playerCount }}/2<br>
                Status: {{ formatStatus(room.status) }}
              </p>
              <button 
                class="btn btn-success" 
                @click="joinRoom(room.id)"
                :disabled="room.playerCount >= 2 || room.status == 1"
              >
                Join Room
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <!-- Game Room -->
    <div v-else class="game-room">
      <div class="row">
        <!-- Game Info -->
        <div class="col-md-4">
          <div class="card mb-4">
            <div class="card-header">Game Info</div>
            <div class="card-body">
              <div class="game-info">
                <p><strong>Room:</strong> {{ currentRoom }}</p>
                <p><strong>Round:</strong> {{ roundsPlayed + 1 }}</p>
                <div v-if="players.length > 0">
                  <h5>Players:</h5>
                  <div v-for="player in players" :key="player.username" class="mb-2">
                    <div :class="{'font-weight-bold': player.username === currentTurn}">
                      {{ player.username }} ({{ player.mark }}) - {{ player.wins }} wins
                      <span v-if="player.username === currentTurn" class="badge bg-success">Current Turn</span>
                    </div>
                  </div>
                </div>
                <div v-else>
                  <p>Waiting for opponent...</p>
                </div>
              </div>
            </div>
          </div>
          
          <div class="card mb-4">
            <div class="card-header">Game Status</div>
            <div class="card-body">
              <p v-if="gameStatus === 0">Waiting for player...</p>
              <p v-else-if="gameStatus === 1">
                {{ currentTurn === username ? "Your turn!" : `Waiting for ${currentTurn}...` }}
              </p>
              <p v-else>Game finished</p>
              
              <div v-if="message" class="alert alert-info mt-3">
                {{ message }}
              </div>
            </div>
          </div>
        </div>
        
        <!-- Game Board -->
        <div class="col-md-8">
          <div class="game-board mx-auto" style="max-width: 400px;">
            <div class="row g-0" v-for="(row, rowIndex) in board" :key="rowIndex">
              <div 
                class="col-4 game-cell" 
                v-for="(cell, colIndex) in row" 
                :key="colIndex"
                @click="makeMove(rowIndex, colIndex)"
                :class="{
                  'cell-x': cell === 'X',
                  'cell-o': cell === 'O',
                  'clickable': gameStatus === 1 && currentTurn === username && !cell
                }"
              >
                <span v-if="cell === 'X'" class="mark">X</span>
                <span v-else-if="cell === 'O'" class="mark">O</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { HubConnectionBuilder, LogLevel } from '@microsoft/signalr';

export default {
  name: 'App',
  data() {
    return {
      connection: null,
      username: '',
      usernameInput: '',
      rooms: [],
      currentRoom: null,
      players: [],
      // Updated to match the backend's List<List<string>> structure
      board: [
        ['', '', ''],
        ['', '', ''],
        ['', '', '']
      ],
      currentTurn: null,
      gameStatus: 0, // 0: Waiting, 1: InProgress, 2: Finished
      roundsPlayed: 0,
      message: ''
    }
  },
  methods: {
    setUsername() {
      if (this.usernameInput.trim()) {
        this.username = this.usernameInput.trim();
        this.connectToSignalR();
      }
    },
    async connectToSignalR() {
      try {
        this.connection = new HubConnectionBuilder()
          .withUrl('https://tic-tac-toe-sob-api.runasp.net/gameHub')  // Update with your backend URL
          .configureLogging(LogLevel.Information)
          .build();
        
        // Set up event handlers
        this.setupSignalREvents();
        
        // Start connection
        await this.connection.start();
        console.log('Connected to SignalR Hub');
        
        // Get rooms
        await this.connection.invoke('GetRooms');
      } catch (err) {
        console.error('Error establishing connection: ', err);
        this.message = 'Failed to connect to game server. Please refresh and try again.';
      }
    },
    setupSignalREvents() {
      this.connection.on('ReceiveRooms', (roomList) => {
        this.rooms = roomList;
      });
      
      this.connection.on('RoomsUpdated', (roomList) => {
        this.rooms = roomList;
      });
      
      this.connection.on('PlayerJoined', (username) => {
        this.message = `${username} has joined the room!`;
      });
      
      this.connection.on('PlayerLeft', (username) => {
        this.message = `${username} has left the room.`;
      });
      
      this.connection.on('GameStarted', (gameData) => {
        this.updateGameState(gameData);
        this.message = 'Game started! Good luck!';
      });
      
      this.connection.on('BoardUpdated', (gameData) => {
        this.updateGameState(gameData);
        this.message = '';
      });
      
      this.connection.on('RoundEnded', (winner) => {
        this.message = `${winner} won the round!`;
      });
      
      this.connection.on('RoundDraw', () => {
        this.message = 'Round ended in a draw!';
      });
      
      this.connection.on('NextRound', (gameData) => {
        this.updateGameState(gameData);
        this.message = 'New round started!';
      });
      
      this.connection.on('GameEnded', (winner) => {
        this.gameStatus = 2;
        this.message = `Game Over! ${winner} won the game!`;
        
        // Reset room after delay
        setTimeout(() => {
          this.currentRoom = null;
          this.players = [];
          this.board = [
            ['', '', ''],
            ['', '', ''],
            ['', '', '']
          ];
          this.currentTurn = null;
          this.gameStatus = 0;
          this.roundsPlayed = 0;
          this.message = '';
          this.connection.invoke('GetRooms');
        }, 5000);
      });
      
      this.connection.on('Error', (errorMessage) => {
        this.message = `Error: ${errorMessage}`;
        console.error('Server error:', errorMessage);
      });
    },
    async joinRoom(roomId) {
      try {
        await this.connection.invoke('JoinRoom', roomId, this.username);
        this.currentRoom = roomId;
      } catch (err) {
        console.error('Error joining room: ', err);
        this.message = 'Failed to join room. Please try again.';
      }
    },
    async makeMove(row, col) {
      // Check if it's player's turn and cell is empty
      if (this.gameStatus !== 1 || this.currentTurn !== this.username || this.board[row][col]) {
        return;
      }
      
      try {
        await this.connection.invoke('MakeMove', {
          roomId: this.currentRoom,
          row: row,
          col: col
        });
      } catch (err) {
        console.error('Error making move: ', err);
        this.message = 'Failed to make move. Please try again.';
      }
    },
    updateGameState(gameData) {
      this.gameStatus = gameData.status;
      this.players = gameData.players;
      this.currentTurn = gameData.currentTurn;
      this.roundsPlayed = gameData.roundsPlayed;
      
      // Update board - directly assign the 2D array/list
      this.board = gameData.board;
    },
    formatStatus(status) {
      const statuses = ['Waiting', 'In Progress', 'Finished'];
      return statuses[status] || 'Unknown';
    }
  }
}
</script>

<style>
body {
  background-color: #f5f5f5;
}

.card {
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.game-board {
  border: 3px solid #333;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
}

.game-cell {
  height: 100px;
  border: 1px solid #333;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 48px;
  font-weight: bold;
}

.clickable {
  cursor: pointer;
  background-color: #f0f8ff;
}

.clickable:hover {
  background-color: #e0f0ff;
}

.cell-x {
  color: #dc3545;
}

.cell-o {
  color: #007bff;
}

.mark {
  animation: pop-in 0.3s;
}

@keyframes pop-in {
  0% { transform: scale(0); opacity: 0; }
  70% { transform: scale(1.2); opacity: 1; }
  100% { transform: scale(1); opacity: 1; }
}
</style>

