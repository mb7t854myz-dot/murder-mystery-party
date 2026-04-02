const socket = io();
let currentGameId = null;
let gameState = null;
let assignDraft = {};

// ─── Init ───────────────────────────────────────────────────────────────────
window.addEventListener('DOMContentLoaded', () => {
  loadGames();
  document.getElementById('new-name').addEventListener('keydown', e => { if (e.key === 'Enter') createGame(); });
  document.getElementById('role-name').addEventListener('keydown', e => { if (e.key === 'Enter') addRole(); });
  document.addEventListener('keydown', e => { if (e.key === 'Escape') document.querySelectorAll('.modal-overlay').forEach(m => m.style.display = 'none'); });
});

socket.on('game:state', state => {
  gameState = state;
  renderGame();
});

socket.on('error', msg => toast(msg, 'error'));

// ─── Game list ──────────────────────────────────────────────────────────────
async function loadGames() {
  const res = await fetch('/api/games');
  const games = await res.json();
  const el = document.getElementById('games-list');
  if (!games.length) { el.innerHTML = '<div class="empty">No games yet. Create your first mystery...</div>'; return; }
  games.sort((a, b) => b.createdAt - a.createdAt);
  el.innerHTML = games.map(g => `
    <div class="game-item" onclick="openGame('${g.id}')">
      <div class="game-icon">🔪</div>
      <div style="flex:1">
        <div class="game-name">${g.name}</div>
        <div class="game-meta">Code: <strong style="font-family:'Cinzel',serif;color:var(--accent)">${g.code}</strong> · ${g.playerCount} players · ${g.ended ? 'Ended' : 'Active'}</div>
      </div>
      <span class="badge ${g.ended ? 'badge-gray' : 'badge-teal'}">${g.ended ? 'Ended' : 'Active'}</span>
    </div>`).join('');
}

async function openGame(id) {
  currentGameId = id;
  socket.emit('gm:join', { gameId: id });
  document.getElementById('view-list').style.display = 'none';
  document.getElementById('view-game').style.display = 'block';
  document.getElementById('g-joinlink').textContent = window.location.origin + '/join?code=';
}

function backToList() {
  currentGameId = null;
  gameState = null;
  document.getElementById('view-game').style.display = 'none';
  document.getElementById('view-list').style.display = 'block';
  document.getElementById('nav-code').textContent = '';
  loadGames();
}

// ─── Create game ─────────────────────────────────────────────────────────────
async function createGame() {
  const name = document.getElementById('new-name').value.trim();
  const theme = document.getElementById('new-theme').value.trim();
  if (!name) { toast('Please enter a game name', 'error'); return; }
  const res = await fetch('/api/games', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ name, theme }) });
  const data = await res.json();
  closeModal('modal-create');
  document.getElementById('new-name').value = '';
  document.getElementById('new-theme').value = '';
  toast('Game created!', 'success');
  openGame(data.id);
}

// ─── Render game ─────────────────────────────────────────────────────────────
function renderGame() {
  if (!gameState) return;
  const g = gameState;

  document.getElementById('g-name').textContent = g.name;
  document.getElementById('g-code').textContent = g.code;
  document.getElementById('nav-code').textContent = 'CODE: ' + g.code;
  document.getElementById('g-status').textContent = g.ended ? 'Ended' : 'Active';
  document.getElementById('g-joinlink').textContent = window.location.origin + '/join?code=' + g.code;

  const assigned = Object.values(g.roleAssignments || {}).filter(Boolean).length;
  document.getElementById('s-players').textContent = g.players.length;
  document.getElementById('s-roles').textContent = assigned + '/' + g.players.length;
  const totalVotes = Object.values(g.votes || {}).reduce((a, b) => a + b, 0);
  document.getElementById('s-votes').textContent = totalVotes;
  document.getElementById('player-count-badge').textContent = g.players.length + ' players';

  document.getElementById('voting-toggle').checked = g.votingOpen;
  document.getElementById('voting-label').textContent = g.votingOpen ? 'Voting is ON' : 'Voting is OFF';

  // Players
  const pl = document.getElementById('players-list');
  const colors = ['#c9a84c','#9b7fd4','#4cb89a','#c94c4c','#4c8bc9','#c97b4c'];
  if (!g.players.length) { pl.innerHTML = '<div class="empty">Waiting for players to join...</div>'; }
  else {
    pl.innerHTML = g.players.map((p, i) => {
      const role = (g.roleAssignments || {})[p] || '';
      const col = colors[i % colors.length];
      const init = p.split(' ').map(w => w[0]).join('').slice(0, 2).toUpperCase();
      return `<div class="player-row">
        <div class="avatar" style="background:${col}22;color:${col}">${init}</div>
        <div style="flex:1">
          <div class="player-name">${p}</div>
          <div class="player-sub">${role || 'No role assigned'}</div>
        </div>
        ${role ? `<span class="badge badge-purple" style="font-size:10px">${role}</span>` : ''}
        <button class="btn btn-sm btn-red" onclick="removePlayer('${p}')">Remove</button>
      </div>`;
    }).join('');
  }

  // Assign roles
  const al = document.getElementById('assign-list');
  if (!g.players.length) { al.innerHTML = '<div class="empty">No players yet</div>'; }
  else {
    if (!Object.keys(assignDraft).length) assignDraft = { ...(g.roleAssignments || {}) };
    al.innerHTML = g.players.map(p => {
      const sel = assignDraft[p] || '';
      return `<div class="assign-row">
        <div style="flex:1;font-size:14px;font-weight:500">${p}</div>
        <select class="assign-select" onchange="assignDraft['${p}']=this.value">
          <option value="">— No role —</option>
          ${(g.roles || []).map(r => `<option value="${r.name}" ${sel === r.name ? 'selected' : ''}>${r.name}</option>`).join('')}
        </select>
        <div style="font-size:11px;color:var(--text3);min-width:70px;text-align:right">${sel ? '✓ Set' : 'Unset'}</div>
      </div>`;
    }).join('');
  }

  // Role library
  const rl = document.getElementById('roles-library');
  if (!g.roles.length) { rl.innerHTML = '<div class="empty">No roles in library</div>'; }
  else {
    rl.innerHTML = g.roles.map(r => `
      <div class="player-row">
        <div style="flex:1">
          <div class="player-name">${r.name}</div>
          <div class="player-sub">${r.desc || 'No description'}</div>
        </div>
        <button class="btn btn-sm btn-red" onclick="removeRole('${r.id}')">Remove</button>
      </div>`).join('');
  }

  // Vote results (GM)
  const vr = document.getElementById('gm-vote-results');
  const votes = g.votes || {};
  const votable = g.votablePlayers || [];
  document.getElementById('total-votes-badge').textContent = totalVotes + ' votes';
  if (!votable.length) { vr.innerHTML = '<div class="empty">No suspects selected yet</div>'; }
  else {
    vr.innerHTML = votable.map(name => {
      const v = votes[name] || 0;
      const pct = totalVotes ? Math.round(v / totalVotes * 100) : 0;
      return `<div class="vote-bar-wrap">
        <div class="vote-bar-label">${name}</div>
        <div class="vote-bar-bg"><div class="vote-bar-fill" style="width:${pct}%"></div></div>
        <div class="vote-count">${v} (${pct}%)</div>
      </div>`;
    }).join('');
  }

  // Votable
  const vl = document.getElementById('votable-list');
  if (!g.players.length) { vl.innerHTML = '<div class="empty">No players yet</div>'; }
  else {
    vl.innerHTML = g.players.map(p => {
      const checked = votable.includes(p);
      return `<div class="votable-row" onclick="toggleVotable('${p}',${!checked})">
        <div class="votable-check ${checked ? 'checked' : ''}"></div>
        <div style="flex:1;font-size:14px">${p}</div>
        ${checked ? '<span class="badge badge-purple">Suspect</span>' : '<span class="badge badge-gray">Excluded</span>'}
      </div>`;
    }).join('');
  }

  // Activity
  const act = document.getElementById('activity-log');
  const log = g.activityLog || [];
  if (!log.length) { act.innerHTML = '<div class="empty">No activity yet</div>'; }
  else {
    act.innerHTML = log.map(a => `
      <div class="activity-item">
        <div class="activity-dot"></div>
        <div><div>${a.msg}</div><div class="activity-time">${a.time}</div></div>
      </div>`).join('');
  }
}

// ─── Actions ─────────────────────────────────────────────────────────────────
function setTab(tab) {
  ['players', 'roles', 'voting', 'activity'].forEach(t => {
    document.getElementById('tab-' + t).style.display = t === tab ? 'block' : 'none';
  });
  document.querySelectorAll('.page-tab').forEach(btn => {
    btn.classList.toggle('active', btn.textContent.toLowerCase().trim() === tab);
  });
}

function saveAssignments() {
  socket.emit('gm:assignRoles', { gameId: currentGameId, assignments: assignDraft });
  toast('Roles saved!', 'success');
}

function clearAssignments() {
  assignDraft = {};
  socket.emit('gm:assignRoles', { gameId: currentGameId, assignments: {} });
  toast('Roles cleared', 'success');
}

function addRole() {
  const name = document.getElementById('role-name').value.trim();
  const desc = document.getElementById('role-desc').value.trim();
  if (!name) { toast('Enter a role name', 'error'); return; }
  socket.emit('gm:addRole', { gameId: currentGameId, name, desc });
  document.getElementById('role-name').value = '';
  document.getElementById('role-desc').value = '';
  closeModal('modal-role');
  toast('Role added', 'success');
}

function removeRole(roleId) {
  socket.emit('gm:removeRole', { gameId: currentGameId, roleId });
}

function removePlayer(name) {
  if (!confirm(`Remove ${name} from the game?`)) return;
  socket.emit('gm:removePlayer', { gameId: currentGameId, playerName: name });
}

function toggleVoting(open) {
  socket.emit('gm:toggleVoting', { gameId: currentGameId, open });
  toast(open ? 'Voting is now OPEN' : 'Voting is now CLOSED', 'success');
}

function resetVotes() {
  if (!confirm('Reset all votes?')) return;
  socket.emit('gm:resetVotes', { gameId: currentGameId });
  toast('Votes reset', 'success');
}

function toggleVotable(player, checked) {
  const votable = [...(gameState?.votablePlayers || [])];
  if (checked && !votable.includes(player)) votable.push(player);
  else if (!checked) { const i = votable.indexOf(player); if (i > -1) votable.splice(i, 1); }
  socket.emit('gm:setVotable', { gameId: currentGameId, votablePlayers: votable });
}

function selectAllVotable() {
  socket.emit('gm:setVotable', { gameId: currentGameId, votablePlayers: [...(gameState?.players || [])] });
}

function endGame() {
  if (!confirm('End this game? Players will be notified.')) return;
  socket.emit('gm:endGame', { gameId: currentGameId });
  toast('Game ended', 'success');
  setTimeout(backToList, 1000);
}

// ─── UI Helpers ──────────────────────────────────────────────────────────────
function showModal(id) {
  document.getElementById(id).style.display = 'flex';
  const first = document.querySelector(`#${id} input`);
  if (first) setTimeout(() => first.focus(), 50);
}
function closeModal(id) { document.getElementById(id).style.display = 'none'; }

function toast(msg, type = 'success') {
  const el = document.createElement('div');
  el.className = `toast ${type}`;
  el.textContent = msg;
  document.getElementById('toasts').appendChild(el);
  setTimeout(() => el.remove(), 3000);
}
