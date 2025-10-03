# pongashzer.exe
pong ta mère
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Pong — Jeu HTML5 (prêt pour GitHub Pages)</title>
  <meta name="description" content="Petite version de Pong en HTML/CSS/JS — glisser-déposer sur GitHub Pages (index.html)." />
  <style>
    /* Reset minimal */
    * { box-sizing: border-box; margin: 0; padding: 0; }
    html,body { height: 100%; }
    body {
      display: flex;
      align-items: center;
      justify-content: center;
      background: linear-gradient(180deg,#0f1724 0%, #071229 100%);
      color: #e6eef8;
      font-family: Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;
      padding: 20px;
    }

    .wrap {
      width: min(900px, 96vw);
      max-width: 960px;
      background: rgba(255,255,255,0.02);
      border: 1px solid rgba(255,255,255,0.04);
      border-radius: 12px;
      padding: 16px;
      box-shadow: 0 6px 30px rgba(2,6,23,0.6);
    }

    header {
      display:flex; align-items:center; justify-content:space-between; gap:12px; margin-bottom:12px;
    }
    h1 { font-size:18px; letter-spacing:0.6px; }
    .controls { display:flex; gap:8px; align-items:center; }

    button {
      background: linear-gradient(180deg, rgba(255,255,255,0.03), rgba(255,255,255,0.01));
      border: 1px solid rgba(255,255,255,0.06);
      color: #e6eef8;
      padding: 8px 10px;
      border-radius: 8px;
      cursor:pointer;
      font-size:14px;
    }
    button:active { transform: translateY(1px); }

    /* Canvas area */
    .stage {
      display:flex; gap:12px; align-items:center; justify-content:center;
      flex-direction:column;
    }

    canvas { background: radial-gradient(circle at 15% 10%, rgba(255,255,255,0.02), rgba(0,0,0,0)); border-radius:8px; max-width:100%; }

    .hud { display:flex; justify-content:space-between; margin-top:10px; font-size:16px; }
    .hint { font-size:13px; opacity:0.85; }

    .footer { margin-top:8px; display:flex; justify-content:space-between; gap:8px; align-items:center; }

    @media (max-width:640px){
      h1{ font-size:16px }
      .hud{ font-size:14px }
      button{ padding:6px 8px; font-size:13px }
    }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Pong — index.html (prêt pour GitHub Pages)</h1>
      <div class="controls">
        <button id="startBtn">Démarrer</button>
        <button id="pauseBtn">Pause</button>
        <button id="resetBtn">Réinitialiser</button>
      </div>
    </header>

    <main class="stage">
      <canvas id="pong" width="900" height="500" aria-label="Canvas du jeu Pong"></canvas>

      <div class="hud">
        <div class="score"><strong id="scoreLeft">0</strong> — <strong id="scoreRight">0</strong></div>
        <div class="hint">Touches: W/S &nbsp;et&nbsp; ↑/↓. Sur mobile: toucher côté gauche/droit pour bouger la raquette. Appuyez sur Espace pour relancer la balle.</div>
      </div>

      <div class="footer">
        <div>Mode: <select id="modeSelect"><option value="local">2 joueurs (local)</option><option value="ai">Joueur vs IA</option></select></div>
        <div>Vitesse: <input id="speedRange" type="range" min="1" max="6" value="3" /></div>
      </div>
    </main>
  </div>

  <script>
    // Simple Pong implementation — single file, no dépendances.
    (function(){
      const canvas = document.getElementById('pong');
      const ctx = canvas.getContext('2d');
      const startBtn = document.getElementById('startBtn');
      const pauseBtn = document.getElementById('pauseBtn');
      const resetBtn = document.getElementById('resetBtn');
      const scoreLeftEl = document.getElementById('scoreLeft');
      const scoreRightEl = document.getElementById('scoreRight');
      const modeSelect = document.getElementById('modeSelect');
      const speedRange = document.getElementById('speedRange');

      let W = canvas.width, H = canvas.height;
      let rafId = null;
      let running = false;

      function resizeCanvasToDisplay() {
        // keep canvas aspect while fitting width
        const maxW = Math.min(900, window.innerWidth - 80);
        const ratio = W / H;
        canvas.width = maxW;
        canvas.height = Math.round(maxW / ratio);
      }
      window.addEventListener('resize', resizeCanvasToDisplay);
      resizeCanvasToDisplay();

      // Game objects
      function makePaddle(x){
        return {
          x, y: canvas.height/2 - 50, w: 12, h: 100, speed: 6, dy: 0
        };
      }
      let left = makePaddle(20);
      let right = makePaddle(canvas.width - 32);
      let ball = { x: canvas.width/2, y: canvas.height/2, r: 8, vx: 6, vy: 3 };
      let score = { left:0, right:0 };

      function resetBall(direction){
        ball.x = canvas.width/2;
        ball.y = canvas.height/2;
        const speed = 4 + Number(speedRange.value);
        const angle = (Math.random() * Math.PI/3) - Math.PI/6; // -30..30deg
        ball.vx = (direction === 'left' ? -1 : 1) * speed * Math.cos(angle);
        ball.vy = speed * Math.sin(angle);
      }

      resetBall('right');

      // Controls
      const keys = {};
      window.addEventListener('keydown', e=>{
        keys[e.key.toLowerCase()] = true;
        if(e.code === 'Space') resetBall(Math.random()<0.5?'left':'right');
      });
      window.addEventListener('keyup', e=>{ keys[e.key.toLowerCase()] = false; });

      // Touch controls: touch left/right halves to move paddles
      canvas.addEventListener('touchstart', handleTouch);
      canvas.addEventListener('touchmove', handleTouch);
      canvas.addEventListener('touchend', ()=>{ left.dy=0; right.dy=0; });

      function handleTouch(e){
        e.preventDefault();
        const rect = canvas.getBoundingClientRect();
        for(const t of e.touches){
          const x = t.clientX - rect.left;
          const y = t.clientY - rect.top;
          if(x < rect.width/2){ // left side touch
            left.y = Math.max(0, Math.min(canvas.height - left.h, y - left.h/2));
          } else { // right side
            right.y = Math.max(0, Math.min(canvas.height - right.h, y - right.h/2));
          }
        }
      }

      // AI for right paddle when mode = ai
      function aiMove(paddle, targetY, difficulty=0.08){
        const center = paddle.y + paddle.h/2;
        const diff = targetY - center;
        paddle.y += diff * difficulty * (1 + Number(speedRange.value)/6);
        // clamp
        paddle.y = Math.max(0, Math.min(canvas.height - paddle.h, paddle.y));
      }

      function update(){
        // update paddles from keys
        // Left: w/s ; Right: ArrowUp/ArrowDown
        if(keys['w']) left.y -= left.speed * (1 + Number(speedRange.value)/6);
        if(keys['s']) left.y += left.speed * (1 + Number(speedRange.value)/6);
        if(keys['arrowup']) right.y -= right.speed * (1 + Number(speedRange.value)/6);
        if(keys['arrowdown']) right.y += right.speed * (1 + Number(speedRange.value)/6);

        // clamp
        left.y = Math.max(0, Math.min(canvas.height - left.h, left.y));
        right.y = Math.max(0, Math.min(canvas.height - right.h, right.y));

        // AI mode
        if(modeSelect.value === 'ai'){
          aiMove(right, ball.y, 0.12);
        }

        // ball movement
        ball.x += ball.vx;
        ball.y += ball.vy;

        // top/bottom collision
        if(ball.y - ball.r <= 0){ ball.y = ball.r; ball.vy *= -1; }
        if(ball.y + ball.r >= canvas.height){ ball.y = canvas.height - ball.r; ball.vy *= -1; }

        // paddle collisions
        function hitPaddle(p){
          return ball.x - ball.r < p.x + p.w && ball.x + ball.r > p.x &&
                 ball.y + ball.r > p.y && ball.y - ball.r < p.y + p.h;
        }

        if(hitPaddle(left) && ball.vx < 0){
          ball.x = left.x + left.w + ball.r;
          const rel = (ball.y - (left.y + left.h/2)) / (left.h/2); // -1..1
          const speed = Math.hypot(ball.vx, ball.vy);
          const newSpeed = Math.min(14, speed + 0.7);
          const angle = rel * Math.PI/4; // ±45deg
          ball.vx = Math.abs(newSpeed * Math.cos(angle));
          ball.vy = newSpeed * Math.sin(angle);
        }

        if(hitPaddle(right) && ball.vx > 0){
          ball.x = right.x - ball.r;
          const rel = (ball.y - (right.y + right.h/2)) / (right.h/2);
          const speed = Math.hypot(ball.vx, ball.vy);
          const newSpeed = Math.min(14, speed + 0.7);
          const angle = rel * Math.PI/4;
          ball.vx = -Math.abs(newSpeed * Math.cos(angle));
          ball.vy = newSpeed * Math.sin(angle);
        }

        // score out
        if(ball.x < -30){ score.right++; scoreRightEl.textContent = score.right; resetBall('right'); }
        if(ball.x > canvas.width + 30){ score.left++; scoreLeftEl.textContent = score.left; resetBall('left'); }
      }

      function drawNet(){
        const step = 18;
        ctx.fillStyle = 'rgba(230,238,248,0.12)';
        for(let y=0; y<canvas.height; y+=step){ ctx.fillRect(canvas.width/2 - 1, y+6, 2, step/2); }
      }

      function render(){
        // clear
        ctx.clearRect(0,0,canvas.width, canvas.height);

        // background subtle
        ctx.fillStyle = 'rgba(255,255,255,0.02)';
        ctx.fillRect(0,0,canvas.width, canvas.height);

        // net
        drawNet();

        // paddles
        ctx.fillStyle = '#e6eef8';
        ctx.fillRect(left.x, left.y, left.w, left.h);
        ctx.fillRect(right.x, right.y, right.w, right.h);

        // ball
        ctx.beginPath(); ctx.arc(ball.x, ball.y, ball.r, 0, Math.PI*2); ctx.fill();

        // scores (already in HUD) — draw small on canvas
        ctx.font = Math.round(canvas.width/20) + 'px system-ui, sans-serif';
        ctx.textAlign = 'center'; ctx.fillStyle = 'rgba(230,238,248,0.18)';
        ctx.fillText(score.left, canvas.width*0.25, 40);
        ctx.fillText(score.right, canvas.width*0.75, 40);
      }

      function frame(){
        update();
        render();
        rafId = requestAnimationFrame(frame);
      }

      // Public controls
      startBtn.addEventListener('click', ()=>{
        if(!running){ running = true; frame(); }
      });
      pauseBtn.addEventListener('click', ()=>{
        if(running){ running = false; cancelAnimationFrame(rafId); rafId = null; }
      });
      resetBtn.addEventListener('click', ()=>{
        score.left = 0; score.right = 0; scoreLeftEl.textContent = '0'; scoreRightEl.textContent = '0';
        resetBall(Math.random()<0.5?'left':'right'); render();
      });

      // ensure paddles reposition on canvas resize
      function syncPaddles(){
        left = makePaddle(20);
        right = makePaddle(canvas.width - 32);
      }
      window.addEventListener('resize', ()=>{ syncPaddles(); render(); });

      // Start automatically
      render();

      // Accessibility: allow clicking canvas to toggle pause
      canvas.addEventListener('click', ()=>{
        if(running){ pauseBtn.click(); } else { startBtn.click(); }
      });

      // Prevent text selection while playing
      document.addEventListener('selectstart', e=>{ if(running) e.preventDefault(); });

      // Friendly: show instructions in console
      console.log('Pong prêt — touches: W/S & ArrowUp/ArrowDown — Espace relance la balle.');

    })();
  </script>
</body>
</html>
