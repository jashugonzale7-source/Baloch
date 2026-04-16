<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Match-3 Game Template</title>
    <style>
        body {
            margin: 0;
            padding: 20px;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            font-family: 'Arial', sans-serif;
        }
        .game-container {
            text-align: center;
            background: rgba(255,255,255,0.1);
            padding: 20px;
            border-radius: 20px;
            backdrop-filter: blur(10px);
            box-shadow: 0 8px 32px rgba(0,0,0,0.3);
        }
        canvas {
            border: 3px solid rgba(255,255,255,0.3);
            border-radius: 15px;
            background: linear-gradient(45deg, #2c3e50, #34495e);
            cursor: pointer;
        }
        .score {
            font-size: 24px;
            color: white;
            margin: 10px 0;
            font-weight: bold;
        }
        .game-over {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0,0,0,0.9);
            color: white;
            padding: 30px;
            border-radius: 15px;
            font-size: 24px;
            display: none;
        }
        button {
            background: linear-gradient(45deg, #ff6b6b, #feca57);
            border: none;
            padding: 12px 24px;
            font-size: 18px;
            border-radius: 25px;
            color: white;
            cursor: pointer;
            margin: 10px;
            font-weight: bold;
        }
        button:hover {
            transform: scale(1.05);
        }
    </style>
</head>
<body>
    <div class="game-container">
        <div class="score">Score: <span id="score">0</span></div>
        <canvas id="gameCanvas" width="400" height="500"></canvas>
        <div class="game-over" id="gameOver">
            <h2>Game Over!</h2>
            <p>Final Score: <span id="finalScore">0</span></p>
            <button onclick="restartGame()">Play Again</button>
        </div>
    </div>

    <script>
        class Match3Game {
            constructor() {
                this.canvas = document.getElementById('gameCanvas');
                this.ctx = this.canvas.getContext('2d');
                this.score = 0;
                this.scoreElement = document.getElementById('score');
                this.gameOverElement = document.getElementById('gameOver');
                this.finalScoreElement = document.getElementById('finalScore');
                
                // Game settings
                this.rows = 10;
                this.cols = 8;
                this.tileSize = 45;
                this.candyTypes = ['red', 'blue', 'green', 'yellow', 'purple', 'orange'];
                this.board = [];
                this.selectedTile = null;
                this.isAnimating = false;
                this.gameOver = false;
                
                this.init();
            }
            
            init() {
                this.generateBoard();
                this.render();
                this.canvas.addEventListener('click', (e) => this.handleClick(e));
                this.gameLoop();
            }
            
            generateBoard() {
                this.board = [];
                for (let r = 0; r < this.rows; r++) {
                    this.board[r] = [];
                    for (let c = 0; c < this.cols; c++) {
                        this.board[r][c] = this.getRandomCandy();
                    }
                }
                // Remove initial matches
                while (this.findMatches().length > 0) {
                    this.clearMatches();
                    this.dropCandies();
                    this.fillEmpty();
                }
            }
            
            getRandomCandy() {
                return this.candyTypes[Math.floor(Math.random() * this.candyTypes.length)];
            }
            
            render() {
                this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
                
                for (let r = 0; r < this.rows; r++) {
                    for (let c = 0; c < this.cols; c++) {
                        if (this.board[r][c]) {
                            this.drawCandy(c * this.tileSize + 5, r * this.tileSize + 5, this.board[r][c]);
                        }
                    }
                }
                
                // Highlight selected tile
                if (this.selectedTile) {
                    const x = this.selectedTile.c * this.tileSize + 2;
                    const y = this.selectedTile.r * this.tileSize + 2;
                    this.ctx.strokeStyle = '#fff';
                    this.ctx.lineWidth = 4;
                    this.ctx.strokeRect(x, y, this.tileSize, this.tileSize);
                }
            }
            
            drawCandy(x, y, type) {
                const colors = {
                    red: '#ff4757', blue: '#3742fa', green: '#2ed573',
                    yellow: '#ffa502', purple: '#8e44ad', orange: '#ff6348'
                };
                
                // Candy background
                this.ctx.fillStyle = colors[type];
                this.ctx.beginPath();
                this.ctx.arc(x + this.tileSize/2, y + this.tileSize/2, this.tileSize/2 - 3, 0, Math.PI * 2);
                this.ctx.fill();
                
                // Highlight
                const gradient = this.ctx.createRadialGradient(
                    x + this.tileSize/4, y + this.tileSize/4, 0,
                    x + this.tileSize/2, y + this.tileSize/2, this.tileSize/2
                );
                gradient.addColorStop(0, 'rgba(255,255,255,0.4)');
                gradient.addColorStop(1, 'rgba(255,255,255,0)');
                this.ctx.fillStyle = gradient;
                this.ctx.beginPath();
                this.ctx.arc(x + this.tileSize/2, y + this.tileSize/2, this.tileSize/2 - 3, 0, Math.PI * 2);
                this.ctx.fill();
                
                // Shine effect
                this.ctx.fillStyle = 'rgba(255,255,255,0.6)';
                this.ctx.beginPath();
                this.ctx.arc(x + this.tileSize/3, y + this.tileSize/3, 4, 0, Math.PI * 2);
                this.ctx.fill();
            }
            
            handleClick(e) {
                if (this.isAnimating || this.gameOver) return;
                
                const rect = this.canvas.getBoundingClientRect();
                const x = e.clientX - rect.left;
                const y = e.clientY - rect.top;
                
                const col = Math.floor(x / this.tileSize);
                const row = Math.floor(y / this.tileSize);
                
                if (col >= 0 && col < this.cols && row >= 0 && row < this.rows && this.board[row][col]) {
                    if (this.selectedTile) {
                        if (this.isAdjacent(this.selectedTile, {r: row, c: col})) {
                            this.swapCandies(this.selectedTile, {r: row, c: col});
                            this.selectedTile = null;
                        } else {
                            this.selectedTile = {r: row, c: col};
                        }
                    } else {
                        this.selectedTile = {r: row, c: col};
                    }
                    this.render();
                }
            }
            
            isAdjacent(tile1, tile2) {
                const dr = Math.abs(tile1.r - tile2.r);
                const dc = Math.abs(tile1.c - tile2.c);
                return (dr === 1 && dc === 0) || (dr === 0 && dc === 1);
            }
            
            swapCandies(tile1, tile2) {
                this.isAnimating = true;
                const temp = this.board[tile1.r][tile1.c];
                this.board[tile1.r][tile1.c] = this.board[tile2.r][tile2.c];
                this.board[tile2.r][tile2.c] = temp;
                
                setTimeout(() => {
                    if (this.findMatches().length === 0) {
                        // Swap back if no match
                        this.board[tile1.r][tile1.c] = this.board[tile2.r][tile2.c];
                        this.board[tile2.r][tile2.c] = temp;
                    } else {
                        this.processMatches();
                    }
                    this.isAnimating = false;
                }, 300);
            }
            
            findMatches() {
                const matches = [];
                
                // Horizontal matches
                for (let r = 0; r < this.rows; r++) {
                    for (let c = 0; c < this.cols - 2; c++) {
                        if (this.board[r][c] && 
                            this.board[r][c] === this.board[r][c+1] && 
                            this.board[r][c] === this.board[r][c+2]) {
                            matches.push({r, c}, {r, c: c+1}, {r, c: c+2});
                        }
                    }
                }
                
                // Vertical matches
                for (let c = 0; c < this.cols; c++) {
                    for (let r = 0; r < this.rows - 2; r++) {
                        if (this.board[r][c] && 
                            this.board[r][c] === this.board[r+1][c] && 
                            this.board[r][c] === this.board[r+2][c]) {
                            matches.push({r, c}, {r: r+1, c}, {r: r+2, c});
                        }
                    }
                }
                
                return matches;
            }
            
            processMatches() {
                const matches = this.findMatches();
                if (matches.length > 0) {

                this.score += matches.length * 10;
                    this.updateScore();
                    
                    matches.forEach(match => {
                        this.board[match.r][match.c] = null;
                    });
                    
                    setTimeout(() => {
                        this.dropCandies();
                        setTimeout(() => {
                            this.fillEmpty();
                            setTimeout(() => {
                                this.processMatches();
                            }, 200);
                        }, 200);
                    }, 200);
                } else {
                    this.isAnimating = false;
                    this.checkGameOver();
                }
            }
            
            dropCandies() {
                for (let c = 0; c < this.cols; c++) {
                    let writeRow = this.rows - 1;
                    for (let r = this.rows - 1; r >= 0; r--) {
                        if (this.board[r][c] !== null) {
                            this.board[writeRow][c] = this.board[r][c];
                            if (writeRow !== r) {
                                this.board[r][c] = null;
                            }
                            writeRow--;
                        }
                    }
                }
            }
            
            fillEmpty() {
                for (let r = 0; r < this.rows; r++) {
                    for (let c = 0; c < this.cols; c++) {
                        if (this.board[r][c] === null) {
                            this.board[r][c] = this.getRandomCandy();
                        }
                    }
                }
            }
            
            updateScore() {
                this.scoreElement.textContent = this.score;
            }
            
            checkGameOver() {
                const hasMoves = this.findPossibleMoves().length > 0;
                if (!hasMoves) {
                    this.gameOver = true;
                    this.finalScoreElement.textContent = this.score;
                    this.gameOverElement.style.display = 'block';
                }
            }
            
            findPossibleMoves() {
                const moves = [];
                for (let r = 0; r < this.rows; r++) {
                    for (let c = 0; c < this.cols; c++) {
                        // Check right and down
                        if (c < this.cols - 1) {
                            this.swapCandies({r, c}, {r, c: c+1});
                            if (this.findMatches().length > 0) {
                                moves.push([{r, c}, {r, c: c+1}]);
                            }
                            this.swapCandies({r, c}, {r, c: c+
