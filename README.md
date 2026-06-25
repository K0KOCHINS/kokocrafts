<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Kokocraft — a voxel sandbox</title>
<style>
  :root {
    --panel: rgba(16, 26, 38, 0.85);
    --ink: #eaf2f8;
    --muted: #9fb2c8;
    --accent: #3b8af0;     /* blue — the one bold note */
    --accent-ink: #ffffff; /* text that sits on the accent */
    --line: rgba(255,255,255,0.10);
    --field: rgba(0,0,0,0.28);
    --field-typeface: "Segoe UI", system-ui, -apple-system, sans-serif;
  }
  * { box-sizing: border-box; }
  html, body { margin: 0; height: 100%; overflow: hidden; background: #bcd4e6; font-family: var(--field-typeface); }
  #app { position: fixed; inset: 0; }
  canvas { display: block; }

  /* --- HUD: a quiet "field notes" panel; the world is the hero --- */
  .hud {
    position: fixed; top: 16px; left: 16px; z-index: 25;
    width: 260px; padding: 14px 16px;
    background: var(--panel); color: var(--ink);
    border: 1px solid var(--line); border-radius: 12px;
    backdrop-filter: blur(8px);
    box-shadow: 0 10px 30px rgba(0,0,0,0.25);
  }
  .hud h1 {
    margin: 0 0 2px; font-size: 15px; letter-spacing: 0.04em; font-weight: 700;
  }
  .hud h1 .mark { color: var(--accent); }
  .hud .dbg-tag { font-size: 10px; letter-spacing: 0.1em; color: var(--muted); text-transform: uppercase; margin-left: 6px; }
  .hud .sub { margin: 0 0 12px; font-size: 11px; color: var(--muted); letter-spacing: 0.06em; text-transform: uppercase; }

  .row { display: flex; align-items: center; gap: 8px; margin: 8px 0; font-size: 12px; color: var(--muted); }
  .row .val { color: var(--ink); margin-left: auto; font-variant-numeric: tabular-nums; }
  .biome { color: var(--accent); font-weight: 600; }

  .controls { margin-top: 12px; border-top: 1px solid var(--line); padding-top: 12px; }
  .controls label { display: block; font-size: 11px; color: var(--muted); margin-bottom: 6px; letter-spacing: 0.04em; }
  .seed-line { display: flex; gap: 8px; }
  input[type="number"] {
    flex: 1; min-width: 0; padding: 7px 9px; font-size: 13px; color: var(--ink);
    background: rgba(0,0,0,0.25); border: 1px solid var(--line); border-radius: 8px;
    font-variant-numeric: tabular-nums;
  }
  input[type="number"]:focus { outline: 2px solid var(--accent); outline-offset: 0; }
  button {
    padding: 7px 12px; font-size: 12px; font-weight: 600; color: var(--accent-ink);
    background: var(--accent); border: none; border-radius: 8px; cursor: pointer;
    letter-spacing: 0.02em;
  }
  button:hover { filter: brightness(1.06); }
  button:active { transform: translateY(1px); }

  .keys { margin-top: 12px; font-size: 11px; color: var(--muted); line-height: 1.7; }
  .keys kbd {
    display: inline-block; min-width: 18px; text-align: center;
    padding: 1px 5px; margin-right: 2px; border-radius: 5px;
    background: rgba(255,255,255,0.10); border: 1px solid var(--line); color: var(--ink);
    font-family: var(--field-typeface); font-size: 10px;
  }

  .crosshair {
    position: fixed; left: 50%; top: 50%; width: 5px; height: 5px; z-index: 4;
    transform: translate(-50%, -50%); border-radius: 50%;
    background: rgba(255,255,255,0.7); box-shadow: 0 0 0 1px rgba(0,0,0,0.4);
    pointer-events: none; opacity: 0;
  }
  /* tint the screen while the camera is below the waterline */
  .underwater {
    position: fixed; inset: 0; z-index: 3; pointer-events: none;
    background: radial-gradient(ellipse at center, rgba(30,90,140,0.25), rgba(12,45,80,0.55));
    opacity: 0; transition: opacity 0.18s ease;
  }

  /* --- pause / settings menus --- */
  .overlay {
    position: fixed; inset: 0; z-index: 20; display: none;
    align-items: center; justify-content: center;
    background: rgba(8, 14, 18, 0.55); backdrop-filter: blur(3px);
  }
  .overlay.show { display: flex; }
  .menu-panel {
    width: 340px; max-height: 88vh; overflow-y: auto;
    padding: 22px 22px 18px; background: var(--panel); color: var(--ink);
    border: 1px solid var(--line); border-radius: 14px;
    box-shadow: 0 18px 50px rgba(0,0,0,0.4);
  }
  .menu-panel.wide { width: 760px; max-width: 94vw; max-height: 92vh; overflow-y: auto; }
  .settings-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px 22px; margin-bottom: 6px; }
  .settings-grid .setting { margin: 0; }
  .toggle.locked, input.locked, input:disabled { opacity: 0.4; cursor: not-allowed; pointer-events: none; }
  .mgr-list { display: flex; flex-direction: column; gap: 8px; margin: 6px 0 16px; min-height: 60px; }
  .mgr-row { display: flex; align-items: center; justify-content: space-between; gap: 12px;
    padding: 12px 14px; background: rgba(255,255,255,0.05); border: 1px solid var(--line); border-radius: 10px; }
  .mgr-row .mgr-name { font-size: 16px; font-weight: 600; }
  .mgr-row .mgr-acts { display: flex; gap: 8px; flex-wrap: wrap; justify-content: flex-end; }
  .mgr-empty { padding: 24px; text-align: center; color: var(--muted); border: 1px dashed var(--line); border-radius: 10px; }
  .menu-title { margin: 0 0 4px; font-size: 20px; letter-spacing: 0.02em; }
  .menu-title .mark { color: var(--accent); }
  .menu-sub { margin: 0 0 16px; font-size: 11px; color: var(--muted); text-transform: uppercase; letter-spacing: 0.08em; }
  .menu-btn {
    display: block; width: 100%; margin: 8px 0; padding: 11px;
    font-size: 14px; font-weight: 600; color: var(--accent-ink); background: var(--accent);
    border: none; border-radius: 9px; cursor: pointer; letter-spacing: 0.02em;
  }
  .menu-btn.secondary { background: rgba(255,255,255,0.10); color: var(--ink); border: 1px solid var(--line); }
  .menu-btn:hover { filter: brightness(1.06); }
  .menu-btn:active { transform: translateY(1px); }
  .menu-note { margin-top: 10px; font-size: 12px; color: var(--accent); text-align: center; min-height: 14px; }

  .setting { margin: 14px 0; }
  .setting label { display: flex; justify-content: space-between; font-size: 12px; color: var(--muted); margin-bottom: 6px; }
  .setting label span { color: var(--ink); font-variant-numeric: tabular-nums; }
  .setting input[type="range"] { width: 100%; accent-color: var(--accent); }
  .toggle-row { display: flex; align-items: center; justify-content: space-between; }
  .toggle-row label { margin: 0; }
  .toggle {
    padding: 5px 14px; font-size: 12px; font-weight: 600; border-radius: 7px; cursor: pointer;
    background: rgba(255,255,255,0.10); color: var(--muted); border: 1px solid var(--line); min-width: 56px;
  }
  .toggle.on { background: var(--accent); color: var(--accent-ink); border-color: transparent; }
  .keybinds { display: grid; grid-template-columns: 1fr auto; gap: 6px 10px; align-items: center; }
  .keybinds .kb-name { font-size: 12px; color: var(--muted); }
  .kb-btn {
    padding: 4px 10px; font-size: 11px; font-family: var(--field-typeface); color: var(--ink);
    background: rgba(255,255,255,0.08); border: 1px solid var(--line); border-radius: 6px; cursor: pointer;
    min-width: 70px; text-align: center;
  }
  .kb-btn.listening { background: var(--accent); color: var(--accent-ink); }

  .fps {
    position: fixed; top: 14px; right: 16px; z-index: 6; display: none;
    padding: 4px 9px; font-family: var(--field-typeface); font-size: 12px; font-variant-numeric: tabular-nums;
    color: var(--ink); background: var(--panel); border: 1px solid var(--line); border-radius: 7px;
  }
  .fps.show { display: block; }

  /* in-game chat */
  .chat {
    position: fixed; left: 12px; bottom: 12px; z-index: 23; width: min(520px, 60vw);
    display: flex; flex-direction: column; gap: 6px; pointer-events: none;
  }
  .chat-log { display: flex; flex-direction: column; gap: 3px; opacity: 0; transition: opacity 0.3s; }
  .chat-log.show { opacity: 1; }
  .chat-line {
    align-self: flex-start; max-width: 100%; padding: 3px 8px; font-size: 13px; line-height: 1.35;
    color: var(--ink); background: rgba(8,14,20,0.62); border-radius: 6px; word-break: break-word;
  }
  .chat-line.system { color: var(--accent); }
  .chat-bar { display: none; align-items: center; gap: 6px; padding: 7px 10px; background: rgba(8,14,20,0.85);
    border: 1px solid var(--accent); border-radius: 8px; }
  .chat-bar.show { display: flex; }
  .chat-bar .prompt { color: var(--accent); font-weight: 700; }
  .chat-bar .text { color: var(--ink); font-size: 14px; white-space: pre-wrap; }
  .chat-bar .caret { width: 7px; height: 16px; background: var(--accent); animation: blink 1s steps(1) infinite; }
  @keyframes blink { 50% { opacity: 0; } }

  /* command hint shown while typing a slash command */
  .chat-hint { display: none; flex-direction: column; gap: 2px; padding: 8px 10px; margin-bottom: 4px;
    background: rgba(8,14,20,0.9); border: 1px solid var(--line); border-radius: 8px; font-size: 12px; color: var(--muted); }
  .chat-hint.show { display: flex; }
  .chat-hint .hint-title { color: var(--accent); font-weight: 700; margin-bottom: 2px; }

  /* hotbar */
  .hotbar { position: fixed; left: 50%; bottom: 14px; transform: translateX(-50%); z-index: 18;
    display: none; gap: 4px; padding: 4px; background: rgba(8,14,20,0.55); border-radius: 8px; }
  .hotbar.show { display: flex; }
  #statBars { position: fixed; left: 50%; bottom: 106px; transform: translateX(-50%); width: 454px; max-width: 96vw;
    z-index: 18; display: none; justify-content: space-between; pointer-events: none; }
  #statBars.show { display: flex; }
  #xpWrap { position: fixed; left: 50%; bottom: 76px; transform: translateX(-50%); width: 320px; max-width: 80vw;
    z-index: 18; display: none; flex-direction: column; align-items: center; pointer-events: none; }
  #xpWrap.show { display: flex; }
  #hurtFlash { position: fixed; inset: 0; z-index: 17; pointer-events: none; opacity: 0;
    background: radial-gradient(circle, rgba(170,0,0,0) 38%, rgba(170,0,0,0.6) 100%); transition: opacity 0.5s ease; }
  #hurtFlash.show { opacity: 1; transition: opacity 0s; }
  #heldName { position: fixed; left: 50%; bottom: 140px; transform: translateX(-50%); z-index: 19;
    color: #fff; font-weight: 600; font-size: 16px; text-shadow: 1px 1px 2px #000, -1px 1px 2px #000;
    pointer-events: none; opacity: 0; transition: opacity 0.6s ease; white-space: nowrap; }
  #heldName.show { opacity: 1; transition: opacity 0s; }
  #slotTip { position: fixed; z-index: 60; background: rgba(8,12,18,0.96); color: #fff; font-size: 13px;
    padding: 4px 9px; border: 1px solid var(--line); border-radius: 6px; pointer-events: none; display: none;
    transform: translate(-50%, -100%); white-space: nowrap; }
  #slotTip.show { display: block; }
  #xpLevel { color: #6dff4c; font-weight: 800; font-size: 14px; line-height: 1; margin-bottom: 2px;
    text-shadow: 1px 1px 0 #000, -1px 1px 0 #000, 1px -1px 0 #000, -1px -1px 0 #000; }
  #xpBar { width: 100%; height: 9px; background: rgba(8,14,20,0.7); border: 1px solid rgba(0,0,0,0.6);
    border-radius: 4px; overflow: hidden; }
  #xpFill { height: 100%; width: 0%; background: linear-gradient(#8dff66, #36b524); transition: width 0.15s ease; }
  .stat-row { display: flex; gap: 1px; }
  .stat-icon { position: relative; width: 18px; height: 18px; filter: drop-shadow(0 1px 1px rgba(0,0,0,0.6)); }
  .stat-icon .base, .stat-icon .fill { position: absolute; inset: 0; background-size: 18px 18px; background-repeat: no-repeat; }
  .stat-icon .fill { overflow: hidden; }
  .stat-icon .fill > i { display: block; width: 18px; height: 18px; background-size: 18px 18px; background-repeat: no-repeat; }
  .slot { position: relative; width: 46px; height: 46px; border: 2px solid var(--line); border-radius: 6px;
    background: rgba(255,255,255,0.05); box-sizing: border-box; cursor: pointer; }
  .slot.sel { border-color: var(--accent); box-shadow: 0 0 0 2px var(--accent); }
  .slot .icon { position: absolute; inset: 7px; border-radius: 3px; box-shadow: inset 0 -6px 8px rgba(0,0,0,0.25);
    image-rendering: pixelated; image-rendering: crisp-edges; background-size: cover; }
  .slot .count { position: absolute; right: 3px; bottom: 1px; font-size: 12px; font-weight: 700; color: #fff;
    text-shadow: 1px 1px 0 #000; font-variant-numeric: tabular-nums; }

  /* inventory overlay */
  .inv-panel { width: 520px; max-width: 94vw; max-height: 90vh; overflow-y: auto; padding: 20px;
    background: var(--panel); color: var(--ink); border: 1px solid var(--line); border-radius: 14px; box-shadow: 0 18px 50px rgba(0,0,0,0.45); }
  .inv-section { margin: 12px 0; }
  .inv-label { font-size: 11px; letter-spacing: 0.1em; text-transform: uppercase; color: var(--muted); margin-bottom: 6px; }
  .grid { display: grid; gap: 4px; }
  .invgrid { grid-template-columns: repeat(9, 1fr); }
  .hotrow  { grid-template-columns: repeat(9, 1fr); }
  .palette { grid-template-columns: repeat(9, 1fr); }
  .craft-row { display: flex; align-items: center; gap: 16px; flex-wrap: wrap; margin: 6px 0 12px; }
  .craft-row .inv-section { margin: 0; }
  .craft2 { grid-template-columns: repeat(2, 1fr); width: 100px; }
  .craft3 { grid-template-columns: repeat(3, 1fr); width: 150px; }
  .craftout { grid-template-columns: repeat(1, 1fr); width: 50px; }
  .craft-arrow { font-size: 24px; color: var(--muted); align-self: flex-end; padding-bottom: 6px; }
  .slot.out { border-color: var(--accent); background: rgba(59,138,240,0.10); cursor: pointer; }
  .grid .slot { width: 100%; aspect-ratio: 1 / 1; height: auto; }
  .slot.pal { border-style: dashed; }
  .carry { position: fixed; z-index: 40; width: 38px; height: 38px; border-radius: 4px; pointer-events: none;
    transform: translate(-50%, -50%); display: none; box-shadow: 0 2px 6px rgba(0,0,0,0.5);
    font-size: 12px; font-weight: 700; color: #fff; text-align: right; }

  /* --- full-screen title screens (home / selector / generator) --- */
  .screen {
    position: fixed; inset: 0; z-index: 22; display: none;
    flex-direction: column; align-items: center; justify-content: center;
    padding: 40px; box-sizing: border-box;
  }
  .screen.show { display: flex; }
  .screen::before {                                   /* darken the live 3D backdrop for legibility */
    content: ''; position: absolute; inset: 0;
    background: linear-gradient(180deg, rgba(8,14,18,0.30), rgba(8,14,18,0.62));
  }
  .screen > * { position: relative; }                 /* lift content above the dim layer */

  .brand { text-align: center; margin-bottom: 30px; }
  .brand h1 { margin: 0; font-size: 52px; letter-spacing: 0.01em; color: var(--ink); text-shadow: 0 4px 24px rgba(0,0,0,0.5); }
  .brand h1 .mark { color: var(--accent); }
  .brand p { margin: 6px 0 0; font-size: 12px; letter-spacing: 0.22em; text-transform: uppercase; color: var(--muted); }
  .credit-line { position: absolute; bottom: 14px; left: 0; right: 0; text-align: center; color: #000; font-size: 13px; font-weight: 600; }

  .home-buttons { display: flex; flex-direction: column; gap: 12px; width: 300px; }
  .home-row { display: flex; gap: 26px; align-items: flex-start; justify-content: center; }
  .home-player { display: flex; flex-direction: column; align-items: center; gap: 10px; }
  .home-player #menuSkin { width: 80px; height: 120px; image-rendering: pixelated;
    background: rgba(255,255,255,0.04); border: 1px solid var(--line); border-radius: 10px; }
  .home-player .big-btn { width: 120px; }
  .editor-wrap { display: flex; gap: 18px; margin: 14px 0; justify-content: center; }
  .editor-left { display: flex; flex-direction: column; gap: 10px; }
  .editor-tabs { display: flex; gap: 6px; flex-wrap: wrap; }
  .editor-tabs button { padding: 6px 10px; font-size: 12px; border-radius: 8px; border: 1px solid var(--line);
    background: rgba(255,255,255,0.05); color: var(--text); cursor: pointer; text-transform: capitalize; }
  .editor-tabs button.active { background: var(--accent); color: #08131a; border-color: var(--accent); font-weight: 700; }
  .editor-grid { display: grid; grid-template-columns: repeat(16, 16px); grid-template-rows: repeat(16, 16px);
    gap: 1px; background: var(--line); border: 2px solid var(--line); border-radius: 6px; width: max-content; touch-action: none; }
  .editor-grid div { width: 16px; height: 16px; cursor: crosshair; }
  .editor-right { display: flex; flex-direction: column; align-items: center; gap: 12px; }
  .editor-right #editorPreview { width: 96px; height: 144px; image-rendering: pixelated;
    background: rgba(255,255,255,0.04); border: 1px solid var(--line); border-radius: 10px; }
  .editor-palette { display: grid; grid-template-columns: repeat(8, 18px); gap: 4px; }
  .editor-palette button { width: 18px; height: 18px; border-radius: 4px; border: 1px solid rgba(0,0,0,0.4); cursor: pointer; padding: 0; }
  .editor-palette button.active { outline: 2px solid #fff; outline-offset: 1px; }
  .editor-colorlbl { font-size: 12px; color: var(--muted); display: flex; align-items: center; gap: 8px; }
  .editor-colorlbl input { width: 40px; height: 26px; padding: 0; border: 1px solid var(--line); background: none; border-radius: 6px; }
  .row-btns { display: flex; gap: 10px; justify-content: center; }
  .big-btn {
    width: 100%; padding: 15px; font-size: 16px; font-weight: 700; letter-spacing: 0.02em;
    color: var(--accent-ink); background: var(--accent); border: none; border-radius: 11px; cursor: pointer;
    box-shadow: 0 10px 28px rgba(0,0,0,0.32);
  }
  .big-btn.secondary { background: rgba(20,30,38,0.78); color: var(--ink); border: 1px solid var(--line); }
  .big-btn:hover { filter: brightness(1.07); }
  .big-btn:active { transform: translateY(1px); }

  .screen-card {
    width: 460px; max-width: 92vw; max-height: 70vh; overflow-y: auto;
    padding: 22px; background: var(--panel); border: 1px solid var(--line); border-radius: 14px;
    box-shadow: 0 18px 50px rgba(0,0,0,0.45);
  }
  .screen-card h2 { margin: 0 0 2px; font-size: 22px; color: var(--ink); }
  .row-btns { display: flex; gap: 8px; margin-top: 12px; }
  .row-btns .menu-btn { flex: 1; margin: 0; }
  .screen-card .sub2 { margin: 0 0 16px; font-size: 11px; letter-spacing: 0.12em; text-transform: uppercase; color: var(--muted); }

  .world-list { display: flex; flex-direction: column; gap: 10px; min-height: 90px; max-height: 46vh; overflow-y: auto; }
  .world-card {
    display: flex; justify-content: space-between; align-items: center; gap: 12px;
    padding: 12px 14px; background: rgba(255,255,255,0.05); border: 1px solid var(--line);
    border-radius: 10px; text-align: left;
  }
  .world-card:hover { border-color: var(--accent); background: rgba(59,138,240,0.12); }
  .world-card .wc-left { flex: 1; cursor: pointer; }
  .world-card .wc-name { font-size: 15px; font-weight: 600; color: var(--ink); }
  .world-card .wc-meta { font-size: 11px; color: var(--muted); margin-top: 3px; }
  .world-card .wc-actions { display: flex; gap: 6px; }
  .wc-btn { padding: 6px 12px; font-size: 12px; font-weight: 600; border-radius: 8px; cursor: pointer;
    border: 1px solid var(--line); background: var(--accent); color: var(--accent-ink); }
  .wc-btn.danger { background: transparent; color: #e6726b; border-color: rgba(230,114,107,0.5); }
  .wc-btn.danger:hover { background: rgba(230,114,107,0.15); }
  .srv-count { position: relative; font-size: 12px; color: var(--muted); cursor: help; padding: 4px 8px; border: 1px solid var(--line); border-radius: 8px; }
  .srv-count .srv-players { display: none; position: absolute; right: 0; bottom: 130%; z-index: 50; min-width: 130px;
    background: rgba(12,18,28,0.96); border: 1px solid var(--line); border-radius: 8px; padding: 8px 10px; text-align: left;
    color: var(--ink); font-size: 12px; white-space: nowrap; box-shadow: 0 8px 24px rgba(0,0,0,0.5); }
  .srv-count:hover .srv-players { display: block; }
  .srv-count .srv-players div { padding: 2px 0; }
  .srv-badge { font-size: 10px; color: var(--accent); text-transform: uppercase; letter-spacing: 0.08em; margin-left: 6px; }
  .srv-tabs { display: flex; gap: 8px; margin: 0 0 12px; }
  .srv-tab { flex: 1; padding: 9px 10px; border: 1px solid rgba(255,255,255,0.14); background: rgba(255,255,255,0.04); color: var(--muted); border-radius: 9px; cursor: pointer; font-size: 13px; font-weight: 600; }
  .srv-tab.active { background: var(--accent); color: #0d0f12; border-color: var(--accent); }
  .set-tabs { display: flex; gap: 8px; margin: 0 0 14px; }
  .set-tab { flex: 1; padding: 9px 10px; border: 1px solid rgba(255,255,255,0.14); background: rgba(255,255,255,0.04); color: var(--muted); border-radius: 9px; cursor: pointer; font-size: 13px; font-weight: 600; }
  .set-tab.active { background: var(--accent); color: #0d0f12; border-color: var(--accent); }
  .world-empty { padding: 24px; text-align: center; font-size: 13px; color: var(--muted); border: 1px dashed var(--line); border-radius: 10px; }

  #playerList {
    position: fixed; top: 40px; left: 50%; transform: translateX(-50%);
    min-width: 260px; max-width: 80vw; padding: 12px 14px; z-index: 40; display: none;
    background: rgba(12,18,28,0.82); border: 1px solid var(--line); border-radius: 10px;
    font-family: inherit; color: var(--ink); backdrop-filter: blur(2px);
  }
  #playerList.show { display: block; }
  #playerList .pl-title { font-size: 12px; letter-spacing: 0.1em; text-transform: uppercase; color: var(--muted); margin-bottom: 8px; text-align: center; }
  #playerList .pl-row { display: flex; justify-content: space-between; gap: 18px; padding: 4px 2px; font-size: 14px; }
  #playerList .pl-row .pl-name { font-weight: 600; }
  #playerList .pl-row .pl-ping { color: var(--muted); font-variant-numeric: tabular-nums; }
  #playerList .pl-row .pl-name .pl-tag { font-size: 10px; color: var(--accent); margin-left: 6px; text-transform: uppercase; letter-spacing: 0.08em; }

  .size-opts { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin: 4px 0 14px; }
  .size-opt {
    padding: 14px; border: 1px solid var(--line); border-radius: 10px; background: rgba(255,255,255,0.04);
    cursor: pointer; text-align: center;
  }
  .size-opt .so-title { font-size: 14px; font-weight: 700; color: var(--ink); }
  .size-opt .so-desc { font-size: 11px; color: var(--muted); margin-top: 4px; }
  .size-opt.active { border-color: var(--accent); background: rgba(59,138,240,0.16); }

  .field-row { margin: 12px 0; }
  .field-row label { display: flex; justify-content: space-between; font-size: 12px; color: var(--muted); margin-bottom: 6px; }
  .field-row label span { color: var(--ink); }
  .field-row input[type="number"] {
    width: 100%; box-sizing: border-box; padding: 9px 10px; font-family: var(--field-typeface); font-size: 14px;
    color: var(--ink); background: var(--field); border: 1px solid var(--line); border-radius: 8px;
  }
  .field-row input[type="range"] { width: 100%; accent-color: var(--accent); }

  .card-actions { display: flex; gap: 10px; margin-top: 18px; }
  .card-actions .menu-btn { margin: 0; }
  .card-actions .grow { flex: 1; }

  /* click-to-play catcher shown after a world finishes loading */
  .startcatch {
    position: fixed; inset: 0; z-index: 24; display: none; cursor: pointer;
    align-items: center; justify-content: center; background: rgba(8,14,18,0.34);
  }
  .startcatch.show { display: flex; }
  .startcatch span {
    padding: 14px 26px; font-size: 16px; font-weight: 700; color: var(--ink);
    background: var(--panel); border: 1px solid var(--line); border-radius: 12px; box-shadow: 0 10px 30px rgba(0,0,0,0.4);
  }

  /* world-generation loading screen */
  .loading {
    position: fixed; inset: 0; z-index: 30; display: none;
    flex-direction: column; align-items: center; justify-content: center;
    background: radial-gradient(ellipse at center, #14202b, #0a1016);
  }
  .loading.show { display: flex; }
  .loading h2 { margin: 0 0 4px; font-size: 22px; color: var(--ink); }
  .loading h2 .mark { color: var(--accent); }
  .loading p { margin: 0 0 18px; font-size: 12px; color: var(--muted); text-transform: uppercase; letter-spacing: 0.1em; }
  .loadbar { width: 300px; height: 10px; border-radius: 6px; background: rgba(255,255,255,0.10); border: 1px solid var(--line); overflow: hidden; }
  .loadbar > i { display: block; height: 100%; width: 0%; background: var(--accent); transition: width 0.12s linear; }
  .loadpct { margin-top: 10px; font-family: var(--field-typeface); font-size: 12px; color: var(--muted); font-variant-numeric: tabular-nums; }
</style>
</head>
<body>
<div id="app"></div>

<div class="hud" id="debug">
  <h1>Koko<span class="mark">craft</span> <span class="dbg-tag">F3 debug</span></h1>
  <div class="row">XYZ <span class="val" id="dbgPos">—</span></div>
  <div class="row">Biome <span class="val biome" id="dbgBiome">—</span></div>
  <div class="row">Time <span class="val" id="clock">—</span></div>
  <div class="row">Day <span class="val" id="dbgDay">—</span></div>
  <div class="row">Chunks <span class="val" id="dbgChunks">—</span></div>
  <div class="row">Blocks <span class="val" id="dbgBlocks">—</span></div>
  <div class="row">Entities <span class="val" id="dbgEntities">—</span></div>
  <div class="row">View <span class="val" id="dbgView">—</span></div>
  <div class="row">Mode <span class="val" id="dbgMode">—</span></div>
</div>

<div class="crosshair" id="crosshair"></div>
<div class="underwater" id="underwater"></div>
<div class="fps" id="fps">— fps</div>

<div id="playerList">
  <div class="pl-title">Players (<span id="plCount">1</span>)</div>
  <div id="plRows"></div>
</div>

<div class="chat" id="chat">
  <div class="chat-log" id="chatLog"></div>
  <div class="chat-hint" id="chatHint"></div>
  <div class="chat-bar" id="chatBar"><span class="prompt">&gt;</span><span class="text" id="chatText"></span><span class="caret"></span></div>
</div>

<div id="hurtFlash"></div>
<div id="heldName"></div>
<div id="slotTip"></div>
<div class="hotbar" id="hotbar"></div>
<div id="xpWrap">
  <div id="xpLevel">0</div>
  <div id="xpBar"><div id="xpFill"></div></div>
</div>
<div id="statBars">
  <div class="stat-row" id="foodRow"></div>
  <div class="stat-row" id="heartRow"></div>
</div>
<div class="overlay" id="deathScreen">
  <div class="menu-panel" style="text-align:center">
    <h2 class="menu-title" style="color:#e6726b">You Died</h2>
    <p class="menu-sub" id="deathMsg">the world goes dark…</p>
    <button class="menu-btn" id="btnRespawn">Respawn</button>
    <button class="menu-btn secondary" id="btnDeathQuit">Save &amp; Quit to Title</button>
  </div>
</div>

<div class="overlay" id="managerMenu">
  <div class="menu-panel" style="width:720px;max-width:94vw;max-height:90vh;overflow-y:auto">
    <h2 class="menu-title">Server Manager</h2>
    <p class="menu-sub">players in your world — teleport to them or remove them</p>
    <div id="managerList" class="mgr-list"></div>
    <button class="menu-btn" id="managerClose">Close (F8)</button>
  </div>
</div>

<div class="overlay" id="inventory">
  <div class="inv-panel">
    <h2 class="menu-title">Inventory</h2>
    <div class="craft-row">
      <div class="inv-section"><div class="inv-label">Crafting</div><div class="grid craft2" id="craft2grid"></div></div>
      <div class="craft-arrow">&#8594;</div>
      <div class="inv-section"><div class="inv-label">Result</div><div class="grid craftout" id="craft2out"></div></div>
    </div>
    <div class="inv-section"><div class="inv-label">Storage</div><div class="grid invgrid" id="invGrid"></div></div>
    <div class="inv-section"><div class="inv-label">Hotbar</div><div class="grid hotrow" id="invHotbar"></div></div>
    <div class="inv-section"><div class="inv-label">All blocks · Creative</div><div class="grid palette" id="palette"></div></div>
    <button class="menu-btn" id="invClose">Close (E)</button>
  </div>
</div>

<div class="overlay" id="tableMenu">
  <div class="inv-panel">
    <h2 class="menu-title">Crafting Table</h2>
    <div class="craft-row">
      <div class="inv-section"><div class="inv-label">Crafting (3×3)</div><div class="grid craft3" id="craft3grid"></div></div>
      <div class="craft-arrow">&#8594;</div>
      <div class="inv-section"><div class="inv-label">Result</div><div class="grid craftout" id="craft3out"></div></div>
    </div>
    <div class="inv-section"><div class="inv-label">Storage</div><div class="grid invgrid" id="tableInv"></div></div>
    <div class="inv-section"><div class="inv-label">Hotbar</div><div class="grid hotrow" id="tableHotbar"></div></div>
    <button class="menu-btn" id="tableClose">Close (E)</button>
  </div>
</div>
<div class="carry" id="carry"></div>

<div class="overlay" id="signEditor">
  <div class="menu-panel">
    <h2 class="menu-title">Edit Sign</h2>
    <p class="menu-sub">type a message, then save</p>
    <input class="field" id="signInput" type="text" maxlength="60" autocomplete="off" placeholder="your message">
    <div class="row-btns">
      <button class="menu-btn secondary" id="signCancel">Cancel</button>
      <button class="menu-btn" id="signSave">Save</button>
    </div>
  </div>
</div>

<div class="loading" id="loading">
  <h2>Koko<span class="mark">craft</span></h2>
  <p>generating terrain</p>
  <div class="loadbar"><i id="loadFill"></i></div>
  <div class="loadpct" id="loadPct">0%</div>
</div>

<div class="startcatch" id="startCatch"><span>Click to play</span></div>

<!-- HOME / title screen -->
<div class="screen" id="usernameMenu">
  <div class="screen-card">
    <div class="brand" style="margin-bottom:14px"><h1>Koko<span class="mark">craft</span></h1><p>voxel sandbox</p></div>
    <h2>Account</h2>
    <p class="sub2">log in, or create a new account — each account keeps its own worlds, settings &amp; skin</p>
    <input class="field" id="acctUser" type="text" placeholder="username" maxlength="16" autocomplete="off">
    <input class="field" id="acctPass" type="password" placeholder="password" maxlength="32" autocomplete="off" style="margin-top:8px">
    <div class="menu-note" id="acctStatus"></div>
    <div class="row-btns" style="margin-top:12px">
      <button class="menu-btn" id="acctLogin">Log In</button>
      <button class="menu-btn secondary" id="acctCreate">Create Account</button>
    </div>
  </div>
</div>

<div class="screen" id="homeMenu">
  <div class="brand"><h1>Koko<span class="mark">craft</span></h1><p>voxel sandbox</p></div>
  <div class="home-row">
    <div class="home-buttons">
      <button class="big-btn" id="hmPlay">Play</button>
      <button class="big-btn secondary" id="hmJoin">Join World</button>
      <button class="big-btn secondary" id="hmServers">Servers</button>
      <button class="big-btn secondary" id="hmSettings">Settings</button>
      <button class="big-btn secondary" id="hmHowto">How to Play</button>
    </div>
    <div class="home-player">
      <canvas id="menuSkin" width="80" height="120"></canvas>
      <button class="big-btn secondary" id="hmEditor">Editor</button>
    </div>
  </div>
  <div class="credit-line">This game is created by Claude Ai and ideas are made by Kokochins</div>
</div>

<!-- PLAYER / SKIN EDITOR -->
<div class="screen" id="editorMenu">
  <div class="screen-card" style="max-width:560px">
    <h2>Player Editor</h2>
    <p class="sub2">paint your skin · pick a part &amp; face, choose a colour, draw</p>
    <div class="editor-wrap">
      <div class="editor-left">
        <div class="editor-tabs" id="editorTabs"></div>
        <div class="editor-tabs" id="editorFaceTabs"></div>
        <div class="editor-grid" id="editorGrid"></div>
      </div>
      <div class="editor-right">
        <canvas id="editorPreview" width="96" height="144"></canvas>
        <div class="editor-palette" id="editorPalette"></div>
        <label class="editor-colorlbl">Colour <input type="color" id="editorColor" value="#222222"></label>
      </div>
    </div>
    <div class="row-btns">
      <button class="menu-btn secondary" id="editorCancel">Cancel</button>
      <button class="menu-btn secondary" id="editorReset">Reset</button>
      <button class="menu-btn" id="editorSave">Save</button>
    </div>
  </div>
</div>

<!-- OPEN TO LAN (host) -->
<div class="screen" id="lanHostMenu">
  <div class="screen-card">
    <h2>Open to LAN</h2>
    <p class="sub2">set a password — others type it to join you</p>
    <input class="field" id="lanHostPw" type="text" placeholder="world password" maxlength="24" autocomplete="off">
    <div class="menu-note" id="lanHostStatus"></div>
    <div class="row-btns">
      <button class="menu-btn secondary" id="lanHostCancel">Back</button>
      <button class="menu-btn" id="lanHostStart">Start Hosting</button>
    </div>
  </div>
</div>

<!-- SERVERS (open LAN worlds) -->
<div class="screen" id="serversMenu">
  <div class="screen-card">
    <h2>Servers</h2>
    <p class="sub2">creator servers are always online · player servers are open LAN worlds</p>
    <div class="srv-tabs">
      <button class="srv-tab active" id="srvTabCreator">Creator made</button>
      <button class="srv-tab" id="srvTabPlayer">Player made</button>
    </div>
    <div class="world-list" id="serverList"></div>
    <div class="menu-note" id="serversStatus"></div>
    <div class="row-btns">
      <button class="menu-btn secondary" id="serversBack">Back</button>
      <button class="menu-btn" id="serversRefresh">Refresh</button>
    </div>
  </div>
</div>

<!-- JOIN WORLD -->
<div class="screen" id="joinMenu">
  <div class="screen-card">
    <h2>Join World</h2>
    <p class="sub2">enter the host's password</p>
    <input class="field" id="joinPw" type="text" placeholder="world password" maxlength="24" autocomplete="off">
    <div class="menu-note" id="joinStatus"></div>
    <div class="row-btns">
      <button class="menu-btn secondary" id="joinCancel">Back</button>
      <button class="menu-btn" id="joinGo">Join</button>
    </div>
  </div>
</div>

<!-- WORLD SELECTOR -->
<div class="screen" id="selectorMenu">
  <div class="screen-card">
    <h2>Select World</h2>
    <p class="sub2">continue a world or make a new one</p>
    <div class="world-list" id="worldList"></div>
    <div class="card-actions">
      <button class="menu-btn secondary grow" id="selCancel">Cancel</button>
      <button class="menu-btn grow" id="selGenerate">Generate</button>
    </div>
  </div>
</div>

<!-- WORLD GENERATOR -->
<div class="screen" id="generatorMenu">
  <div class="screen-card">
    <h2>New World</h2>
    <p class="sub2">name it, pick a seed and size</p>

    <div class="field-row">
      <label for="genName">World name</label>
      <input id="genName" type="text" value="New World" maxlength="28" autocomplete="off" />
    </div>

    <div class="field-row">
      <label for="genSeed">World seed</label>
      <input id="genSeed" type="number" value="1337" />
    </div>

    <div class="size-opts">
      <div class="size-opt active" id="optInfinite" data-mode="infinite">
        <div class="so-title">Infinite</div>
        <div class="so-desc">endless terrain</div>
      </div>
      <div class="size-opt" id="optCustom" data-mode="custom">
        <div class="so-title">Custom size</div>
        <div class="so-desc">a bounded square</div>
      </div>
    </div>

    <div class="field-row" id="customSizeRow" style="display:none">
      <label>World size <span id="genSizeVal">256 blocks</span></label>
      <input type="range" id="genSize" min="64" max="1024" step="32" value="256">
    </div>

    <div class="field-row" style="justify-content:space-between;align-items:center">
      <label>Flat world <span style="font-size:11px;color:var(--muted)">all plains · no hills or oceans</span></label>
      <button class="toggle" id="genFlat" type="button">Off</button>
    </div>

    <label style="display:block;font-size:12px;color:var(--muted);margin:6px 0 6px">Game mode</label>
    <div class="size-opts">
      <div class="size-opt active" id="optCreative">
        <div class="so-title">Creative</div>
        <div class="so-desc">fly · free build · commands</div>
      </div>
      <div class="size-opt" id="optSurvival">
        <div class="so-title">Survival</div>
        <div class="so-desc">no commands (unless OP)</div>
      </div>
    </div>

    <div class="card-actions">
      <button class="menu-btn secondary grow" id="genCancel">Cancel</button>
      <button class="menu-btn grow" id="genPlay">Play</button>
    </div>
  </div>
</div>

<!-- HOW TO PLAY -->
<div class="screen" id="howtoMenu">
  <div class="screen-card">
    <h2>How to Play</h2>
    <p class="sub2">creative flight · break blocks</p>
    <div class="keys" style="position:static;font-size:13px;line-height:2">
      <div><kbd>W</kbd><kbd>A</kbd><kbd>S</kbd><kbd>D</kbd> move around</div>
      <div><kbd>Space</kbd> fly up &nbsp; <kbd>Shift</kbd> fly down</div>
      <div>move the mouse to look &nbsp; left-click to break a block</div>
      <div><kbd>F5</kbd> switch first / third person</div>
      <div><kbd>Esc</kbd> open the in-game menu</div>
    </div>
    <div class="card-actions"><button class="menu-btn grow" id="howtoBack">Back</button></div>
  </div>
</div>

<!-- IN-GAME PAUSE (Esc) -->
<div class="overlay" id="pauseMenu">
  <div class="menu-panel">
    <h2 class="menu-title">Paused</h2>
    <p class="menu-sub">Kokocraft</p>
    <button class="menu-btn" id="btnPlay">Resume</button>
    <button class="menu-btn secondary" id="btnLan">Open to LAN</button>
    <button class="menu-btn secondary" id="btnSettings">Settings</button>
    <button class="menu-btn secondary" id="btnSaveQuit">Save &amp; Quit to Title</button>
    <button class="menu-btn secondary" id="btnLogout">Log Out</button>
    <div class="menu-note" id="pauseNote"></div>
  </div>
</div>

<!-- SETTINGS (reached from home or pause; Save / Cancel) -->
<div class="overlay" id="settingsMenu">
  <div class="menu-panel wide">
    <h2 class="menu-title">Settings</h2>
    <p class="menu-sub">configure your game</p>

    <div class="set-tabs">
      <button class="set-tab active" id="setTabVideo">Video settings</button>
      <button class="set-tab" id="setTabGraphics">Graphics</button>
      <button class="set-tab" id="setTabKeys">Keybinds</button>
      <button class="set-tab" id="setTabHosting">Hosting</button>
    </div>

    <!-- VIDEO -->
    <div class="set-panel" id="setPanelVideo">
      <div class="settings-grid">
      <div class="setting">
        <label>Render distance <span id="rdVal">—</span></label>
        <input type="range" id="setRender" min="2" max="18" step="1">
      </div>
      <div class="setting">
        <label>Field of view <span id="fovVal">—</span></label>
        <input type="range" id="setFov" min="50" max="110" step="1">
      </div>
      <div class="setting">
        <label>Brightness <span id="brightVal">—</span></label>
        <input type="range" id="setBrightness" min="0" max="100" step="1">
      </div>
      <div class="setting">
        <label>Time of day <span id="timeVal">—</span></label>
        <input type="range" id="setTime" min="0" max="1439" step="1">
      </div>
      <div class="setting toggle-row">
        <label>FPS display</label>
        <button class="toggle" id="setFps">Off</button>
      </div>
      <div class="setting toggle-row">
        <label>Performance mode <span style="font-size:10px">low detail far away</span></label>
        <button class="toggle" id="setPerf">Off</button>
      </div>
      </div>
    </div>

    <!-- GRAPHICS -->
    <div class="set-panel" id="setPanelGraphics" style="display:none">
      <div class="settings-grid">
      <div class="setting toggle-row">
        <label>Textures <span style="font-size:10px">High = crisp HD · Low = lighter</span></label>
        <button class="toggle" id="setTexHi">High</button>
      </div>
      <div class="setting toggle-row">
        <label>Simple game <span style="font-size:10px">flat 1–2 colour blocks</span></label>
        <button class="toggle" id="setSimple">Off</button>
      </div>
      <div class="setting toggle-row">
        <label>Shadows <span style="font-size:10px">cast sun/moon shadows</span></label>
        <button class="toggle" id="setShadows">On</button>
      </div>
      <div class="setting toggle-row">
        <label>Smooth lighting</label>
        <button class="toggle" id="setSmooth">On</button>
      </div>
      <div class="setting toggle-row">
        <label>Reflections <span style="font-size:10px">reflective water (costs FPS)</span></label>
        <button class="toggle" id="setReflect">Off</button>
      </div>
      <div class="setting toggle-row">
        <label>Water flow <span style="font-size:10px">animated rippling water</span></label>
        <button class="toggle" id="setWaterFlow">On</button>
      </div>
      <div class="setting toggle-row">
        <label>Clouds <span style="font-size:10px">drifting clouds overhead</span></label>
        <button class="toggle" id="setClouds">On</button>
      </div>
      <div class="setting toggle-row">
        <label>Ore vision <span style="font-size:10px">x-ray: only ores &amp; grass show</span></label>
        <button class="toggle" id="setXray">Off</button>
      </div>
      <div class="setting toggle-row">
        <label>Fullbright <span style="font-size:10px">see in the dark</span></label>
        <button class="toggle" id="setBright">Off</button>
      </div>
      </div>
    </div>

    <!-- KEYBINDS -->
    <div class="set-panel" id="setPanelKeys" style="display:none">
      <div class="setting">
        <label>Keybinds <span style="font-size:10px">click, then press a key</span></label>
        <div class="keybinds" id="keybinds"></div>
      </div>
    </div>

    <!-- HOSTING -->
    <div class="set-panel" id="setPanelHosting" style="display:none">
      <div class="settings-grid">
      <div class="setting toggle-row">
        <label>Open world <span style="font-size:10px">list in Servers menu (no passcode)</span></label>
        <button class="toggle" id="setOpenWorld">Off</button>
      </div>
      <div class="setting toggle-row">
        <label>Operator (OP) <span style="font-size:10px">commands in any mode</span></label>
        <button class="toggle" id="setOp">Off</button>
      </div>
      </div>
    </div>

    <div class="card-actions">
      <button class="menu-btn secondary grow" id="setCancel">Cancel</button>
      <button class="menu-btn grow" id="setSave">Save</button>
    </div>
  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/peerjs/1.5.4/peerjs.min.js"></script>
<script>
/* ============================================================
   Kokocraft — a compact open-world voxel prototype.
   Everything is in one file on purpose so it's easy to read
   and to pull apart. Search for the SECTION markers below.
   ============================================================ */

// ---------- SECTION 1: scene, camera, renderer, light ----------
const app = document.getElementById('app');

const scene = new THREE.Scene();
// (sky background + fog are created and animated by the day/night system below)

const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 600);
camera.layers.enable(1);                          // main view shows water (layer 1); reflection camera will not

const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;   // soft, smooth shadow edges
app.appendChild(renderer.domElement);

// Soft ambient fill (sky/ground bounce). Colours + intensity change over the day.
const hemi = new THREE.HemisphereLight(0xbcd4e6, 0x4a4636, 0.7);
scene.add(hemi);

// The single celestial light: it IS the sun during the day and the moon at night.
// Its direction (and colour/intensity) is recomputed every frame from the time of
// day, so the sun's position genuinely drives how surfaces are lit.
const sunLight = new THREE.DirectionalLight(0xfff2dd, 1.0);
sunLight.castShadow = true;
sunLight.shadow.mapSize.set(2048, 2048);            // balanced shadow quality vs FPS
sunLight.shadow.bias = -0.0004;
sunLight.shadow.normalBias = 0.6;                   // tames acne on blocky faces
{
  const c = sunLight.shadow.camera, d = 60;         // frustum follows the player
  c.left = -d; c.right = d; c.top = d; c.bottom = -d; c.near = 1; c.far = 420;
  c.updateProjectionMatrix();
}
scene.add(sunLight);
const lightTarget = new THREE.Object3D();
scene.add(lightTarget);
sunLight.target = lightTarget;

// drifting clouds high above the world (toggleable in settings)
const cloudGroup = new THREE.Group(); scene.add(cloudGroup);
const CLOUD_Y = 118, CLOUD_SPAN = 340, CLOUD_WIND = 1.5;
(function buildClouds(){
  const mat = new THREE.MeshLambertMaterial({ color: 0xffffff, transparent: true, opacity: 0.85, depthWrite: false, fog: false });
  const rnd = (a, b) => a + Math.random() * (b - a);
  for (let i = 0; i < 18; i++){
    const puff = new THREE.Group();
    const n = 3 + ((Math.random() * 4) | 0);
    for (let j = 0; j < n; j++){
      const m = new THREE.Mesh(new THREE.BoxGeometry(rnd(12, 24), rnd(3, 5), rnd(12, 24)), mat);
      m.position.set(rnd(-12, 12), rnd(-1, 1), rnd(-12, 12)); m.castShadow = false; m.receiveShadow = false;
      puff.add(m);
    }
    puff.position.set(rnd(-CLOUD_SPAN/2, CLOUD_SPAN/2), CLOUD_Y + rnd(-5, 5), rnd(-CLOUD_SPAN/2, CLOUD_SPAN/2));
    cloudGroup.add(puff);
  }
})();
function updateClouds(dt){
  if (!settings.clouds){ cloudGroup.visible = false; return; }
  cloudGroup.visible = true;
  cloudGroup.position.x = camera.position.x; cloudGroup.position.z = camera.position.z;   // always overhead
  for (const puff of cloudGroup.children){
    puff.position.x += dt * CLOUD_WIND;                                                     // drift on the wind
    if (puff.position.x > CLOUD_SPAN/2) puff.position.x -= CLOUD_SPAN;
  }
}

// dynamic water reflections (opt-in): a cube camera captures the scene (sky, sun, terrain, player)
// while ignoring water itself; its texture is used as the water envMap when Reflections is enabled.
const reflectRT = new THREE.WebGLCubeRenderTarget(128, { generateMipmaps: true, minFilter: THREE.LinearMipmapLinearFilter });
const reflectCam = new THREE.CubeCamera(0.5, 600, reflectRT);   // only renders layer 0 -> excludes water (layer 1)
scene.add(reflectCam);

// ---------- SECTION 1b: sky, sun, moon, stars + day/night state ----------
const SKY_DIST = 380;                  // how far the celestial bodies sit from the camera
let timeOfDay = 0.30;                  // 0=midnight, .25=sunrise, .5=noon, .75=sunset
let dayCount = 1;                      // in-game days elapsed (shown in F3)
const DAY_SECONDS   = 20 * 60;         // real seconds of daylight (sunrise -> sunset)
const NIGHT_SECONDS = 10 * 60;         // real seconds of night  (sunset -> sunrise)
const sunDir = new THREE.Vector3();    // recomputed each frame
const moonDir = new THREE.Vector3();

// sky-colour keyframes
const SKY_NIGHT = new THREE.Color(0x0b1026);
const SKY_DAY   = new THREE.Color(0xa6caec);
const SKY_DUSK  = new THREE.Color(0xe98c5a);
const skyColor  = new THREE.Color(SKY_DAY);
scene.background = skyColor;                 // updated in place every frame
scene.fog = new THREE.FogExp2(SKY_DAY.getHex(), 0.012);

// small canvas-texture helpers (self-contained; used only for the sky)
function radialTexture(stops){
  const c = document.createElement('canvas'); c.width = c.height = 128;
  const x = c.getContext('2d'); const g = x.createRadialGradient(64, 64, 0, 64, 64, 64);
  stops.forEach(([o, col]) => g.addColorStop(o, col));
  x.fillStyle = g; x.fillRect(0, 0, 128, 128);
  return new THREE.CanvasTexture(c);
}

// Sun: a bright disc with an additive glow halo
const sunMesh = new THREE.Mesh(
  new THREE.SphereGeometry(16, 20, 20),
  new THREE.MeshBasicMaterial({ color: 0xfff3c0, fog: false })
);
const sunGlow = new THREE.Sprite(new THREE.SpriteMaterial({
  map: radialTexture([[0,'rgba(255,240,190,0.9)'],[0.3,'rgba(255,210,120,0.5)'],[1,'rgba(255,200,120,0)']]),
  transparent: true, blending: THREE.AdditiveBlending, depthWrite: false, fog: false
}));
sunGlow.scale.set(170, 170, 1);
scene.add(sunMesh, sunGlow);

// Moon: a pale cratered disc
const moonTex = (() => {
  const c = document.createElement('canvas'); c.width = c.height = 64; const x = c.getContext('2d');
  x.fillStyle = '#d9dee8'; x.fillRect(0, 0, 64, 64);
  x.fillStyle = '#bcc2d0';
  [[20,18,7],[40,34,9],[30,48,5],[48,16,4]].forEach(([cx,cy,r]) => { x.beginPath(); x.arc(cx,cy,r,0,7); x.fill(); });
  return new THREE.CanvasTexture(c);
})();
const moonMesh = new THREE.Mesh(
  new THREE.SphereGeometry(12, 20, 20),
  new THREE.MeshBasicMaterial({ map: moonTex, fog: false })
);
const moonGlow = new THREE.Sprite(new THREE.SpriteMaterial({
  map: radialTexture([[0,'rgba(200,215,245,0.5)'],[1,'rgba(200,215,245,0)']]),
  transparent: true, blending: THREE.AdditiveBlending, depthWrite: false, fog: false
}));
moonGlow.scale.set(70, 70, 1);
scene.add(moonMesh, moonGlow);

// Stars: points on a far sphere, faded in at night
const starGeo = new THREE.BufferGeometry();
{
  const N = 700, arr = new Float32Array(N * 3);
  for (let i = 0; i < N; i++){
    const u = Math.random() * 2 - 1, a = Math.random() * Math.PI * 2, r = Math.sqrt(1 - u*u);
    arr[i*3] = Math.cos(a)*r*SKY_DIST; arr[i*3+1] = Math.abs(u)*SKY_DIST; arr[i*3+2] = Math.sin(a)*r*SKY_DIST;
  }
  starGeo.setAttribute('position', new THREE.BufferAttribute(arr, 3));
}
const starField = new THREE.Points(starGeo, new THREE.PointsMaterial({
  color: 0xffffff, size: 1.6, sizeAttenuation: false, transparent: true, opacity: 0, fog: false, depthWrite: false
}));
scene.add(starField);

const _c1 = new THREE.Color(), _c2 = new THREE.Color();
const clamp01 = v => Math.max(0, Math.min(1, v));

// Recompute the whole sky/lighting state for the current time of day.
function updateSky(dt){
  if (ruleForceDay()){ timeOfDay = 0.5; }                // permanent daytime on this server
  const prevTime = timeOfDay;
  // daytime spans timeOfDay 0.25..0.75 (sunrise..sunset); advance each half at its own pace
  const isDay = (timeOfDay >= 0.25 && timeOfDay < 0.75);
  const rate  = isDay ? (0.5 / DAY_SECONDS) : (0.5 / NIGHT_SECONDS);
  if (!ruleForceDay()) timeOfDay = (timeOfDay + dt * rate) % 1;
  if (prevTime > 0.9 && timeOfDay < 0.1) dayCount++;     // wrapped past midnight = a new day

  // sun travels an arc; angle is 0 at the eastern horizon at sunrise
  const ang = timeOfDay * Math.PI * 2 - Math.PI / 2;
  sunDir.set(Math.cos(ang), Math.sin(ang), 0.33).normalize();
  moonDir.copy(sunDir).multiplyScalar(-1);
  const sunY = sunDir.y;

  const day  = clamp01((sunY + 0.12) / 0.45);          // 0 night ... 1 full day
  const glow = clamp01(1 - Math.abs(sunY) / 0.22);     // peaks at sunrise/sunset

  // sky + fog colour
  skyColor.copy(SKY_NIGHT).lerp(SKY_DAY, day).lerp(SKY_DUSK, glow * 0.55);
  scene.fog.color.copy(skyColor);

  // ambient fill follows the sky
  hemi.intensity = 0.22 + day * 0.5;
  hemi.color.copy(skyColor).lerp(_c1.set(0xffffff), 0.25 * day);

  // the directional light is the sun by day, the moon by night
  if (sunY > 0){
    sunLight.position.copy(benny.position).addScaledVector(sunDir, 140);
    sunLight.color.copy(_c1.set(0xffb070)).lerp(_c2.set(0xfff4e0), clamp01(sunY * 2.2));
    sunLight.intensity = clamp01(sunY * 1.8) * 1.15;
  } else {
    sunLight.position.copy(benny.position).addScaledVector(moonDir, 140);
    sunLight.color.set(0xaebfe6);
    sunLight.intensity = clamp01(-sunY * 1.4) * 0.28;
  }
  lightTarget.position.copy(benny.position);
  lightTarget.updateMatrixWorld();

  // underground gets darker: the deeper the eye sits below the local surface,
  // the less sunlight reaches it (a cheap stand-in for real light propagation)
  let caveFactor = 1;
  if (playing){
    const depth = heightAt(Math.floor(camera.position.x), Math.floor(camera.position.z)) - camera.position.y;
    if (depth > 0) caveFactor = Math.max(0.07, 1 - depth / 16);
  }
  hemi.intensity *= caveFactor;
  sunLight.intensity *= caveFactor;

  // smooth lighting: soft shadow edges + gentle fill; off = crisp, flatter, blockier
  if (settings.smoothLighting){
    sunLight.shadow.radius = 4;
  } else {
    sunLight.shadow.radius = 0;
    hemi.intensity = Math.min(1.1, hemi.intensity + 0.12);
  }

  // fullbright: see everywhere — overrides night AND underground darkness
  if (settings.fullbright){
    hemi.intensity = 1.25;
    sunLight.intensity = 0.5;
    sunLight.castShadow = false;
  } else {
    sunLight.castShadow = settings.shadows !== false;   // respect the Shadows toggle
  }
  hemi.intensity *= brightnessMul;                 // user brightness setting
  sunLight.intensity *= brightnessMul;

  // place the celestial bodies relative to the camera so they read as infinitely far
  sunMesh.position.copy(camera.position).addScaledVector(sunDir, SKY_DIST);
  sunGlow.position.copy(sunMesh.position);
  moonMesh.position.copy(camera.position).addScaledVector(moonDir, SKY_DIST);
  moonGlow.position.copy(moonMesh.position);
  sunMesh.visible = sunGlow.visible = sunDir.y > -0.12;
  moonMesh.visible = moonGlow.visible = moonDir.y > -0.12;
  starField.position.copy(camera.position);
  starField.material.opacity = clamp01(-sunY * 2 - 0.1);   // only at night

  // HUD clock
  const mins = Math.floor(timeOfDay * 24 * 60);
  const hh = String(Math.floor(mins / 60)).padStart(2, '0');
  const mm = String(mins % 60).padStart(2, '0');
  const clock = document.getElementById('clock');
  if (clock) clock.textContent = `${hh}:${mm}`;
}

// ---------- SECTION 2: seeded procedural noise ----------
// A tiny seeded value-noise + fBm. This is the heart of
// "custom world generation": same seed -> same world.
function fade(t){ return t*t*t*(t*(t*6-15)+10); }
function lerp(a,b,t){ return a+(b-a)*t; }

function makeNoise(seed){
  function rand(ix, iy){
    let h = (Math.imul(ix|0, 73856093) ^ Math.imul(iy|0, 19349663) ^ Math.imul(seed|0, 83492791)) >>> 0;
    h = Math.imul(h ^ (h >>> 13), 1274126177) >>> 0;
    return (h & 0xffffff) / 0x1000000;            // 0..1
  }
  function value2D(x, y){
    const xi = Math.floor(x), yi = Math.floor(y);
    const xf = x - xi, yf = y - yi;
    const u = fade(xf), v = fade(yf);
    const a = lerp(rand(xi, yi),   rand(xi+1, yi),   u);
    const b = lerp(rand(xi, yi+1), rand(xi+1, yi+1), u);
    return lerp(a, b, v);                          // 0..1
  }
  function fbm(x, y, octaves, freq, persist){
    let amp = 1, sum = 0, norm = 0, f = freq;
    for (let i = 0; i < octaves; i++){
      sum  += amp * value2D((x+i*131.7)*f, (y-i*57.3)*f);
      norm += amp; amp *= persist; f *= 2;
    }
    return sum / norm;                             // 0..1
  }
  return { fbm, value2D };
}

// ---------- SECTION 3: world parameters (infinite, chunked) ----------
const CHUNK   = 32;          // a chunk is CHUNK x CHUNK columns of blocks
const WORLD_H = 128;         // vertical block limit per column
const WATER_Y = 28;          // sea level (absolute block height) — raised by 1
const BASE_Y  = 12;          // lowest land height
const HILL    = 38;          // how tall terrain can rise above BASE_Y

const AIR = 0, GRASS = 1, DIRT = 2, SAND = 3, STONE = 4, SNOW_B = 5, WOOD = 6, LEAF = 7, WATER = 8;
const COAL_ORE = 9, IRON_ORE = 10, DIAMOND_ORE = 11;
const WOOD_PLANKS = 12, CRAFTING_TABLE = 13, SIGN = 14;
const CACTUS = 15, JUNGLE_WOOD = 16, JUNGLE_LEAF = 17, VINE = 18, SANDSTONE = 19, DRY_GRASS = 20;
const ROAD = 21, CONCRETE = 22, BRICK = 23, GLASS_B = 24;   // custom Koko City blocks
const isOpaque = v => v !== AIR && v !== WATER;   // water is see-through, like air, for exposure

// every chunk owns its own voxel array; chunks live in a Map keyed by "cx,cz"
const chunks = new Map();
const edits  = new Map();    // player changes: "wx,wy,wz" -> block id (survives reg/reload of a chunk)
const ckey = (cx, cz) => cx + ',' + cz;
const cIndex = (lx, y, lz) => (y * CHUNK + lz) * CHUNK + lx;     // local index within a chunk
const floorDiv = (a, b) => Math.floor(a / b);

// noise fields for the current world; (re)made whenever the seed changes
let elevNoise, moistNoise, forestNoise, caveNoise, tempNoise, wetNoise;
function seedWorld(seed){
  elevNoise   = makeNoise(seed);
  moistNoise  = makeNoise(seed ^ 0x9e3779b1);
  forestNoise = makeNoise(seed ^ 0x68bc21aa);
  caveNoise   = makeNoise(seed ^ 0x1b873593);
  tempNoise   = makeNoise(seed ^ 0x2c1b3a5d);
  wetNoise    = makeNoise(seed ^ 0x5f4d1c9b);
  villageCache.clear();
  if (typeof templeCache !== 'undefined') templeCache.clear();
  if (typeof mvCache    !== 'undefined') mvCache.clear();
  if (typeof ruinCache  !== 'undefined') ruinCache.clear();
  if (typeof fishCache  !== 'undefined') fishCache.clear();
  if (typeof gtCache    !== 'undefined') gtCache.clear();
}

// pseudo-3D cave field: average three rotated 2D samples so it varies in all axes.
// returns true where rock should be carved into open cave.
function caveCarve(wx, wy, wz){
  const f = 0.058;
  const n = (caveNoise.value2D(wx * f, wz * f)
           + caveNoise.value2D(wz * f + 31.7, wy * f * 1.6)
           + caveNoise.value2D(wy * f * 1.6 + 11.3, wx * f)) / 3;
  return n > 0.72;
}

// terrain surface height (absolute block y of the top solid block) at a world column
function heightAt(wx, wz){
  if (isFlatMode()) return FLAT_Y;                       // flat-family worlds: dead-flat plains
  const c = elevNoise.fbm(wx, wz, 4, 0.0042, 0.5);      // continentalness: very low freq -> big oceans & continents
  if (c < 0.46){                                         // ocean basin: deeper the further from shore
    const d = (0.46 - c) / 0.46;                         // 0 at shore .. 1 far out
    return Math.max(2, WATER_Y - 2 - Math.floor(d * 18));
  }
  const land = (c - 0.46) / 0.54;                        // 0 at shore .. 1 deep inland
  let h = WATER_Y + 1 + Math.floor(land * 5);            // gentle plains rise
  const ridge = moistNoise.fbm(wx, wz, 4, 0.012, 0.5);   // mountain ridges
  const mtn = Math.max(0, ridge - 0.46) / 0.54;          // 0..1 where ridges are high
  h += Math.floor(Math.pow(mtn, 1.15) * 56 * Math.min(1, land * 2.2));
  return Math.max(1, Math.min(WORLD_H - 14, h));
}

// look up the block at any world position; AIR for unloaded neighbours, solid below the world
function blockWorld(wx, wy, wz){
  if (wy < 0) return STONE;
  if (wy >= WORLD_H) return AIR;
  const cx = floorDiv(wx, CHUNK), cz = floorDiv(wz, CHUNK);
  const ch = chunks.get(ckey(cx, cz));
  if (!ch) return AIR;
  return ch.data[cIndex(wx - cx * CHUNK, wy, wz - cz * CHUNK)];
}
// an opaque block is drawn if any neighbour is non-opaque (air OR water), so seabed shows through water
function exposedOpaque(wx, wy, wz){
  return !isOpaque(blockWorld(wx+1,wy,wz)) || !isOpaque(blockWorld(wx-1,wy,wz)) || !isOpaque(blockWorld(wx,wy+1,wz)) ||
         !isOpaque(blockWorld(wx,wy-1,wz)) || !isOpaque(blockWorld(wx,wy,wz+1)) || !isOpaque(blockWorld(wx,wy,wz-1));
}
// a water block face is drawn only where it meets open air (its surface & shoreline)
function exposedWater(wx, wy, wz){
  return blockWorld(wx+1,wy,wz)===AIR || blockWorld(wx-1,wy,wz)===AIR || blockWorld(wx,wy+1,wz)===AIR ||
         blockWorld(wx,wy-1,wz)===AIR || blockWorld(wx,wy,wz+1)===AIR || blockWorld(wx,wy,wz-1)===AIR;
}

// stable ordering of biomes -> array index (used for textures)
const BIOME_ORDER = ['Ocean', 'Beach', 'Plains', 'Oak Forest', 'Mountains', 'Desert', 'Jungle', 'Savanna'];
const biomeIndex = name => BIOME_ORDER.indexOf(name);

// name the biome of a column from its surface height plus temperature + moisture fields
function biomeName(wx, wz, sY){
  if (isFlatMode()) return 'Plains';                     // flat-family worlds are pure plains
  if (sY === undefined) sY = heightAt(wx, wz);
  if (sY < WATER_Y - 1)        return 'Ocean';
  if (sY <= WATER_Y)           return 'Beach';
  if (sY >= WATER_Y + 10)      return 'Mountains';
  const temp = tempNoise.fbm(wx, wz, 2, 0.0055, 0.5);
  const wet  = wetNoise.fbm(wx, wz, 2, 0.0055, 0.5);
  if (temp > 0.6  && wet < 0.45) return 'Desert';      // hot + dry
  if (temp > 0.55 && wet > 0.62) return 'Jungle';      // hot + wet
  if (temp > 0.5  && wet < 0.52) return 'Savanna';     // warm + dryish
  const fo = forestNoise.fbm(wx, wz, 3, 0.011, 0.5);
  return fo > 0.55 ? 'Oak Forest' : 'Plains';
}

// ---------- SECTION 3b: procedural pixel textures (Minecraft-ish) ----------
// Each block type gets its own little 16x16 canvas texture, drawn with noise
// and a few hand-placed details, then sampled with NearestFilter for crisp pixels.
const clampByte = v => Math.max(0, Math.min(255, v | 0));
let TEX_RES = 32;          // procedural texture resolution: 32 = HD (default), 16 = low

function pixelTexture(paint){
  const S = TEX_RES;
  const cv = document.createElement('canvas'); cv.width = cv.height = S;
  const ctx = cv.getContext('2d');
  paint(ctx, S);
  const tex = new THREE.CanvasTexture(cv);
  tex.magFilter = THREE.NearestFilter;                 // crisp pixels up close
  tex.minFilter = THREE.LinearMipmapLinearFilter;      // smooth in the distance
  tex.wrapS = tex.wrapT = THREE.RepeatWrapping;
  tex.anisotropy = TEX_RES >= 32 ? 8 : 2;
  return tex;
}
// fill the whole tile with luminance-noisy variations of a base color
function noiseFill(ctx, S, base, spread){
  for (let y = 0; y < S; y++) for (let x = 0; x < S; x++){
    const n = (Math.random() - 0.5) * 2 * spread;
    ctx.fillStyle = `rgb(${clampByte(base[0]+n)},${clampByte(base[1]+n)},${clampByte(base[2]+n)})`;
    ctx.fillRect(x, y, 1, 1);
  }
}
function specks(ctx, S, count, color, alpha = 1){
  ctx.fillStyle = color; ctx.globalAlpha = alpha;
  for (let k = 0; k < count; k++) ctx.fillRect((Math.random()*S)|0, (Math.random()*S)|0, 1, 1);
  ctx.globalAlpha = 1;
}

function buildTextures(){
  const grass = base => pixelTexture((ctx, S) => {
    noiseFill(ctx, S, base, 16);
    specks(ctx, S, 26, 'rgba(40,70,30,0.55)');     // darker tufts
    specks(ctx, S, 22, 'rgba(190,220,140,0.5)');   // lighter blades
  });
  const sand = (base, ripple) => pixelTexture((ctx, S) => {
    noiseFill(ctx, S, base, 10);
    if (ripple) for (let y = 1; y < S; y += 3){     // faint dune ripples
      ctx.fillStyle = 'rgba(150,125,80,0.18)'; ctx.fillRect(0, y, S, 1);
    }
    specks(ctx, S, 18, 'rgba(120,95,55,0.4)');
  });
  const stone = base => pixelTexture((ctx, S) => {
    noiseFill(ctx, S, base, 18);
    ctx.strokeStyle = 'rgba(60,55,50,0.55)'; ctx.lineWidth = 1;   // a couple of cracks
    for (let k = 0; k < 3; k++){
      ctx.beginPath();
      let x = (Math.random()*S)|0, y = (Math.random()*S)|0;
      ctx.moveTo(x, y);
      for (let s = 0; s < 4; s++){ x += (Math.random()*5-2)|0; y += (Math.random()*5-2)|0; ctx.lineTo(x, y); }
      ctx.stroke();
    }
    specks(ctx, S, 16, 'rgba(200,196,190,0.4)');
  });
  const snow = pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [236, 242, 248], 8);
    specks(ctx, S, 10, 'rgba(255,255,255,0.9)');   // sparkle
    specks(ctx, S, 12, 'rgba(190,205,225,0.5)');   // faint blue shadow
  });
  const oceanTop = pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [104, 134, 132], 14);        // silty seabed
    specks(ctx, S, 20, 'rgba(60,80,80,0.5)');
    specks(ctx, S, 10, 'rgba(150,170,160,0.4)');
  });

  const dirt    = pixelTexture((ctx, S) => { noiseFill(ctx, S, [110, 80, 55], 16); specks(ctx, S, 22, 'rgba(70,48,30,0.5)'); specks(ctx, S, 12, 'rgba(150,115,80,0.4)'); });
  const stoneBody = stone([122, 116, 108]);
  const sandBody  = sand([205, 184, 138], false);
  const oceanBody = pixelTexture((ctx, S) => { noiseFill(ctx, S, [74, 96, 104], 12); specks(ctx, S, 18, 'rgba(45,60,68,0.5)'); });

  // top texture and body texture per biome index (matches BIOME_ORDER)
  const tops = [
    oceanTop,                 // 0 Ocean
    sand([224, 202, 150], false),  // 1 Beach
    sand([228, 200, 138], true),   // 2 Desert
    grass([150, 182, 96]),    // 3 Plains
    grass([92, 140, 78]),     // 4 Forest
    stone([142, 134, 124]),   // 5 Mountains
    snow,                     // 6 Snowy Peaks
  ];
  const bodies = [ oceanBody, sandBody, sandBody, dirt, dirt, stoneBody, stoneBody ];

  const bark = pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [110, 78, 48], 12);
    for (let x = 2; x < S; x += 5){ ctx.fillStyle = 'rgba(70,48,28,0.5)'; ctx.fillRect(x, 0, 1, S); } // vertical grain
  });
  const leaf = pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [70, 110, 55], 22);
    specks(ctx, S, 40, 'rgba(40,75,30,0.6)');
    specks(ctx, S, 24, 'rgba(120,160,80,0.5)');
  });

  return { tops, bodies, bark, leaf };
}
let TEX = buildTextures();

// wood planks: light brown horizontal boards with seams + shading
function makePlanksTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [180, 140, 94], 10);
    for (let y = 0; y < S; y++){
      if (y % 4 === 0){ ctx.fillStyle = 'rgba(92,62,34,0.7)'; ctx.fillRect(0, y, S, 1); }      // seam between boards
      else if (y % 4 === 1){ ctx.fillStyle = 'rgba(255,238,205,0.18)'; ctx.fillRect(0, y, S, 1); } // highlight under seam
    }
    for (let band = 0; band < S; band += 4){ const jx = ((band * 5) + 3) % S; ctx.fillStyle = 'rgba(92,62,34,0.4)'; ctx.fillRect(jx, band, 1, 4); } // staggered board joints
    specks(ctx, S, 16, 'rgba(122,84,46,0.35)');                                                  // grain flecks
  });
}
// crafting table top: planks with a grid + small tool marks
function makeCraftTopTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [172, 132, 86], 8);
    ctx.fillStyle = 'rgba(74,50,28,0.92)'; ctx.fillRect(0,0,S,1); ctx.fillRect(0,S-1,S,1); ctx.fillRect(0,0,1,S); ctx.fillRect(S-1,0,1,S); // frame
    ctx.fillStyle = 'rgba(74,50,28,0.7)'; ctx.fillRect(0,5,S,1); ctx.fillRect(0,10,S,1); ctx.fillRect(5,0,1,S); ctx.fillRect(10,0,1,S);   // grid
    ctx.fillStyle = 'rgba(40,40,46,0.85)'; for (let i = 0; i < 4; i++) ctx.fillRect(2+i, 2+i, 1, 1);   // saw mark (diagonal)
    ctx.fillStyle = 'rgba(150,150,160,0.85)'; ctx.fillRect(12,3,2,1); ctx.fillRect(3,12,1,2);          // tool glints
  });
}
// crafting table side: wooden panel with a toolbox/hammer detail
function makeCraftSideTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [152, 114, 74], 8);
    ctx.fillStyle = 'rgba(74,50,28,0.85)'; ctx.fillRect(0,0,S,1); ctx.fillRect(0,4,S,1);              // top plank edge
    ctx.fillStyle = 'rgba(255,238,205,0.15)'; ctx.fillRect(0,1,S,1);
    ctx.fillStyle = 'rgba(74,50,28,0.7)'; ctx.fillRect(1,5,1,S-6); ctx.fillRect(S-2,5,1,S-6);          // side frame
    ctx.fillStyle = 'rgba(60,60,68,0.85)'; ctx.fillRect(6,9,4,1); ctx.fillRect(7,9,1,5);               // hammer
    ctx.fillStyle = 'rgba(140,140,150,0.8)'; ctx.fillRect(6,8,4,1);
    specks(ctx, S, 12, 'rgba(112,78,44,0.35)');
  });
}
let PLANKS_TEX = makePlanksTex();
let CT_TOP = makeCraftTopTex();
let CT_SIDE = makeCraftSideTex();
// sign: a small board mounted on a post
function makeSignTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [120, 86, 52], 8);                       // post-dark wood base
    ctx.fillStyle = 'rgba(196,160,108,1)'; ctx.fillRect(1, 2, S-2, 7);    // light board
    ctx.fillStyle = 'rgba(255,238,205,0.25)'; ctx.fillRect(1, 2, S-2, 1);
    ctx.fillStyle = 'rgba(80,54,30,0.8)'; ctx.fillRect(1, 8, S-2, 1);     // board shadow
    ctx.fillStyle = 'rgba(90,62,34,0.9)'; ctx.fillRect((S>>1)-1, 9, 2, S-9); // post
  });
}
let SIGN_TEX = makeSignTex();

// --- biome blocks ---
function makeCactusTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [70, 130, 70], 14);
    for (let x = 1; x < S; x += Math.max(3, (S/5)|0)){ ctx.fillStyle = 'rgba(40,90,45,0.6)'; ctx.fillRect(x, 0, 1, S); }   // vertical ribs
    specks(ctx, S, 14, 'rgba(220,235,180,0.5)');                                                                          // tiny spines
  });
}
function makeJungleWoodTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [92, 64, 40], 12);
    for (let x = 2; x < S; x += 5){ ctx.fillStyle = 'rgba(54,36,22,0.55)'; ctx.fillRect(x, 0, 1, S); }
    specks(ctx, S, 18, 'rgba(120,150,80,0.25)');                                                                          // mossy tint
  });
}
function makeJungleLeafTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [40, 96, 42], 24);
    specks(ctx, S, 42, 'rgba(24,60,24,0.6)');
    specks(ctx, S, 22, 'rgba(90,150,70,0.5)');
  });
}
function makeVineTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [48, 100, 46], 18);
    for (let x = 2; x < S; x += 4){ ctx.fillStyle = 'rgba(30,70,30,0.7)'; ctx.fillRect(x, 0, 1, S); }
    specks(ctx, S, 16, 'rgba(80,140,70,0.5)');
  });
}
function makeSandstoneTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [222, 204, 150], 8);
    for (let y = 0; y < S; y += Math.max(3, (S/5)|0)){ ctx.fillStyle = 'rgba(180,158,108,0.5)'; ctx.fillRect(0, y, S, 1); }  // horizontal strata
    specks(ctx, S, 12, 'rgba(150,128,84,0.4)');
  });
}
function makeDryGrassTex(){
  return pixelTexture((ctx, S) => {
    noiseFill(ctx, S, [176, 168, 96], 14);
    specks(ctx, S, 26, 'rgba(120,110,56,0.5)');
    specks(ctx, S, 20, 'rgba(206,196,130,0.5)');
  });
}
let CACTUS_TEX = makeCactusTex(), JWOOD_TEX = makeJungleWoodTex(), JLEAF_TEX = makeJungleLeafTex();
let VINE_TEX = makeVineTex(), SANDSTONE_TEX = makeSandstoneTex(), DRYGRASS_TEX = makeDryGrassTex();

// ---------- SECTION 4: build (or rebuild) the world ----------
const _dummy = new THREE.Object3D();
const _color = new THREE.Color();

// shared cube + one material per block type (built once, reused on every rebuild)
const UNIT_CUBE = new THREE.BoxGeometry(1, 1, 1);
// signs render as a board on a post (not a full cube)
const SIGN_POST_GEO  = new THREE.BoxGeometry(0.14, 0.55, 0.14);
const SIGN_BOARD_GEO = new THREE.BoxGeometry(0.82, 0.46, 0.13);
// procedural ore texture: a stone base with coloured speckles
function makeOreTex(spotColor){
  const S = TEX_RES; const c = document.createElement('canvas'); c.width = c.height = S; const g = c.getContext('2d');
  for (let y = 0; y < S; y++) for (let x = 0; x < S; x++){ const v = 116 + ((x * 7 + y * 13) % 26); g.fillStyle = 'rgb(' + v + ',' + v + ',' + (v + 4) + ')'; g.fillRect(x, y, 1, 1); }
  g.fillStyle = spotColor; const sz = Math.max(2, (S / 8) | 0); const n = Math.round(7 * (S / 16));
  for (let k = 0; k < n; k++){ const x = (k * 53 + 3) % (S - 2) + 1, y = (k * 97 + 5) % (S - 2) + 1; g.fillRect(x, y, sz, sz); }
  const t = new THREE.CanvasTexture(c); t.magFilter = THREE.NearestFilter; t.minFilter = THREE.LinearMipmapLinearFilter; t.generateMipmaps = true; t.anisotropy = TEX_RES >= 32 ? 8 : 2; return t;
}
// a small sky-gradient cube map used as an environment for reflective water
function makeSkyEnv(){
  const faces = [];
  for (let i = 0; i < 6; i++){
    const c = document.createElement('canvas'); c.width = c.height = 32; const g = c.getContext('2d');
    if (i === 2){ g.fillStyle = '#bfe0ff'; g.fillRect(0,0,32,32); }                 // +Y (up): bright sky
    else if (i === 3){ g.fillStyle = '#3a526a'; g.fillRect(0,0,32,32); }            // -Y (down): dim
    else { const grd = g.createLinearGradient(0,0,0,32); grd.addColorStop(0,'#a6caec'); grd.addColorStop(0.55,'#cfe6fb'); grd.addColorStop(1,'#5b7fa0'); g.fillStyle = grd; g.fillRect(0,0,32,32); }
    faces.push(c);
  }
  const tex = new THREE.CubeTexture(faces); tex.needsUpdate = true; return tex;
}
const SKY_ENV = makeSkyEnv();
let ORE_COAL = makeOreTex('#161616'), ORE_IRON = makeOreTex('#c79a6d'), ORE_DIAMOND = makeOreTex('#54d7df');
const CT_SIDE_MAT = new THREE.MeshLambertMaterial({ map: CT_SIDE });
const CT_TOP_MAT  = new THREE.MeshLambertMaterial({ map: CT_TOP });
const CT_BOT_MAT  = new THREE.MeshLambertMaterial({ map: PLANKS_TEX });
const BLOCK_MAT = {
  [GRASS]:  new THREE.MeshLambertMaterial({ map: TEX.tops[3] }),
  [DIRT]:   new THREE.MeshLambertMaterial({ map: TEX.bodies[3] }),
  [SAND]:   new THREE.MeshLambertMaterial({ map: TEX.tops[1] }),
  [STONE]:  new THREE.MeshLambertMaterial({ map: TEX.tops[5] }),
  [SNOW_B]: new THREE.MeshLambertMaterial({ map: TEX.tops[6] }),
  [WOOD]:   new THREE.MeshLambertMaterial({ map: TEX.bark }),
  [LEAF]:   new THREE.MeshLambertMaterial({ map: TEX.leaf }),
  [COAL_ORE]:    new THREE.MeshLambertMaterial({ map: ORE_COAL }),
  [IRON_ORE]:    new THREE.MeshLambertMaterial({ map: ORE_IRON }),
  [DIAMOND_ORE]: new THREE.MeshLambertMaterial({ map: ORE_DIAMOND }),
  [WOOD_PLANKS]: new THREE.MeshLambertMaterial({ map: PLANKS_TEX }),
  [SIGN]: new THREE.MeshLambertMaterial({ map: SIGN_TEX }),
  [CACTUS]:      new THREE.MeshLambertMaterial({ map: CACTUS_TEX }),
  [JUNGLE_WOOD]: new THREE.MeshLambertMaterial({ map: JWOOD_TEX }),
  [JUNGLE_LEAF]: new THREE.MeshLambertMaterial({ map: JLEAF_TEX }),
  [VINE]:        new THREE.MeshLambertMaterial({ map: VINE_TEX }),
  [SANDSTONE]:   new THREE.MeshLambertMaterial({ map: SANDSTONE_TEX }),
  [DRY_GRASS]:   new THREE.MeshLambertMaterial({ map: DRYGRASS_TEX }),
  [ROAD]:        new THREE.MeshLambertMaterial({ color: 0x35383d }),   // asphalt
  [CONCRETE]:    new THREE.MeshLambertMaterial({ color: 0xb7bcc2 }),   // pale concrete
  [BRICK]:       new THREE.MeshLambertMaterial({ color: 0x9e4b3a }),   // red brick
  [GLASS_B]:     new THREE.MeshLambertMaterial({ color: 0x9fd4e8 }),   // window glass (opaque)
  // crafting table: per-face materials. BoxGeometry group order is +X,-X,+Y,-Y,+Z,-Z
  [CRAFTING_TABLE]: [CT_SIDE_MAT, CT_SIDE_MAT, CT_TOP_MAT, CT_BOT_MAT, CT_SIDE_MAT, CT_SIDE_MAT],
  [WATER]:  new THREE.MeshStandardMaterial({ color: 0x2f6f9e, transparent: true, opacity: 0.82,
                                            depthWrite: false, side: THREE.DoubleSide,
                                            metalness: 0.0, roughness: 1.0 }),
};
const BLOCK_TYPES = [GRASS, DIRT, SAND, STONE, SNOW_B, WOOD, LEAF, COAL_ORE, IRON_ORE, DIAMOND_ORE, WOOD_PLANKS, CRAFTING_TABLE, SIGN, CACTUS, JUNGLE_WOOD, JUNGLE_LEAF, VINE, SANDSTONE, DRY_GRASS, ROAD, CONCRETE, BRICK, GLASS_B];   // breakable, opaque

// faint ripple texture for water; its UV offset is scrolled each frame for a flowing look
function makeWaterTex(){
  const t = pixelTexture((ctx, S) => {
    for (let y = 0; y < S; y++) for (let x = 0; x < S; x++){
      const v = 235 + Math.round(20 * Math.sin((x + y) * 0.9) + 8 * Math.sin(x * 1.7));
      ctx.fillStyle = 'rgb(' + v + ',' + v + ',' + v + ')'; ctx.fillRect(x, y, 1, 1);
    }
    ctx.fillStyle = 'rgba(255,255,255,0.5)';
    for (let k = 0; k < S; k += 4) ctx.fillRect(0, (k + 1) % S, S, 1);   // gentle horizontal crests
  });
  t.wrapS = t.wrapT = THREE.RepeatWrapping; return t;
}
let WATER_TEX = makeWaterTex();
BLOCK_MAT[WATER].map = WATER_TEX; BLOCK_MAT[WATER].needsUpdate = true;
// gentle ocean waves: displace the water's top surface with two sine ripples in world space
const waterWave = { value: 0 };
BLOCK_MAT[WATER].onBeforeCompile = (shader) => {
  shader.uniforms.uWaveTime = waterWave;
  shader.vertexShader = 'uniform float uWaveTime;\n' + shader.vertexShader.replace(
    '#include <begin_vertex>',
    '#include <begin_vertex>\n if (normal.y > 0.5){ transformed.y += sin(position.x*0.7 + uWaveTime*1.7)*0.07 + sin(position.z*0.55 + uWaveTime*1.3)*0.07; }'
  );
};

// decorative grass tufts ("hairs") that sit on top of some grass blocks
function makeGrassHairTex(){
  const c = document.createElement('canvas'); c.width = c.height = 16; const g = c.getContext('2d');
  g.clearRect(0, 0, 16, 16);
  for (let b = 0; b < 11; b++){
    const x = 1 + ((b * 97) % 14), hgt = 6 + ((b * 53) % 8);
    g.strokeStyle = (b % 2) ? '#4f8f3a' : '#62ad4c'; g.lineWidth = 1;
    g.beginPath(); g.moveTo(x, 15); g.lineTo(x + ((b % 3) - 1), 15 - hgt); g.stroke();
  }
  const t = new THREE.CanvasTexture(c); t.magFilter = THREE.NearestFilter; t.minFilter = THREE.NearestFilter; return t;
}
const GRASS_HAIR_MAT = new THREE.MeshLambertMaterial({ map: makeGrassHairTex(), transparent: true, alphaTest: 0.5, side: THREE.DoubleSide });
const GRASS_HAIR_GEO = (function(){
  const g = new THREE.BufferGeometry(); const H = 0.55, W = 0.7;
  const p = [], uv = [], idx = [];
  const quad = (ax, az, bx, bz) => { const base = p.length / 3;
    p.push(ax,0,az, bx,0,bz, bx,H,bz, ax,H,az); uv.push(0,0, 1,0, 1,1, 0,1);
    idx.push(base,base+1,base+2, base,base+2,base+3); };
  quad(-W/2,-W/2, W/2,W/2); quad(-W/2,W/2, W/2,-W/2);          // two crossed blades
  g.setAttribute('position', new THREE.Float32BufferAttribute(p, 3));
  g.setAttribute('uv', new THREE.Float32BufferAttribute(uv, 2));
  g.setIndex(idx); g.computeVertexNormals(); return g;
})();

// crisp-near / smooth-far filtering + anisotropy: a real quality bump at distance & oblique angles
function applyTexFilter(){
  const maxAni = (renderer.capabilities && renderer.capabilities.getMaxAnisotropy) ? renderer.capabilities.getMaxAnisotropy() : 1;
  const ani = TEX_RES >= 32 ? maxAni : Math.min(2, maxAni);
  const tex = [];
  for (const t of BLOCK_TYPES){ const m = BLOCK_MAT[t]; if (!m) continue; (Array.isArray(m) ? m : [m]).forEach(mm => { if (mm.map) tex.push(mm.map); }); }
  if (WATER_TEX) tex.push(WATER_TEX);
  for (const m of tex){
    m.magFilter = THREE.NearestFilter;
    m.minFilter = (TEX_RES >= 32) ? THREE.LinearMipmapLinearFilter : THREE.NearestFilter;
    m.generateMipmaps = (TEX_RES >= 32); m.anisotropy = ani; m.needsUpdate = true;
  }
}
applyTexFilter();

// data-URL icons drawn straight from each block's own texture, for the hotbar / inventory
const ICON_URL = {};
function rebuildIcons(){
  for (const t of BLOCK_TYPES){
    const tex = BLOCK_MAT[t] && BLOCK_MAT[t].map;
    try { if (tex && tex.image && tex.image.toDataURL) ICON_URL[t] = tex.image.toDataURL(); } catch (e){}
  }
  try { ICON_URL[CRAFTING_TABLE] = CT_TOP.image.toDataURL(); } catch (e){}   // array material has no single .map
}
rebuildIcons();

// regenerate every world texture at the current TEX_RES and reassign maps (for the Low/High toggle)
function rebuildTextures(){
  try {
    TEX = buildTextures();
    PLANKS_TEX = makePlanksTex(); CT_TOP = makeCraftTopTex(); CT_SIDE = makeCraftSideTex(); SIGN_TEX = makeSignTex();
    ORE_COAL = makeOreTex('#161616'); ORE_IRON = makeOreTex('#c79a6d'); ORE_DIAMOND = makeOreTex('#54d7df');
    CACTUS_TEX = makeCactusTex(); JWOOD_TEX = makeJungleWoodTex(); JLEAF_TEX = makeJungleLeafTex();
    VINE_TEX = makeVineTex(); SANDSTONE_TEX = makeSandstoneTex(); DRYGRASS_TEX = makeDryGrassTex();
    WATER_TEX = makeWaterTex();
    const set = (t, map) => { if (BLOCK_MAT[t]){ BLOCK_MAT[t].map = map; BLOCK_MAT[t].needsUpdate = true; } };
    set(GRASS, TEX.tops[3]); set(DIRT, TEX.bodies[3]); set(SAND, TEX.tops[1]); set(STONE, TEX.tops[5]);
    set(SNOW_B, TEX.tops[6]); set(WOOD, TEX.bark); set(LEAF, TEX.leaf);
    set(COAL_ORE, ORE_COAL); set(IRON_ORE, ORE_IRON); set(DIAMOND_ORE, ORE_DIAMOND);
    set(WOOD_PLANKS, PLANKS_TEX); set(SIGN, SIGN_TEX);
    set(CACTUS, CACTUS_TEX); set(JUNGLE_WOOD, JWOOD_TEX); set(JUNGLE_LEAF, JLEAF_TEX);
    set(VINE, VINE_TEX); set(SANDSTONE, SANDSTONE_TEX); set(DRY_GRASS, DRYGRASS_TEX);
    CT_SIDE_MAT.map = CT_SIDE; CT_SIDE_MAT.needsUpdate = true;
    CT_TOP_MAT.map = CT_TOP; CT_TOP_MAT.needsUpdate = true;
    CT_BOT_MAT.map = PLANKS_TEX; CT_BOT_MAT.needsUpdate = true;
    BLOCK_MAT[WATER].map = WATER_TEX; BLOCK_MAT[WATER].needsUpdate = true;
    applyTexFilter(); rebuildIcons();
    if (typeof settings !== 'undefined' && settings.simpleGame) stripToSimple();   // keep flat blocks after a texture rebuild
    if (typeof updateHeldViewmodel === 'function') updateHeldViewmodel(true);       // refresh the in-hand block texture
    if (typeof refreshHotbar === 'function') refreshHotbar();
    if (typeof refreshOpenMenus === 'function') refreshOpenMenus();
  } catch (e){ console.warn('texture rebuild failed', e); }
}

// surface block vs underground block for a biome
function surfaceType(name){
  switch (name){
    case 'Plains': case 'Oak Forest': case 'Jungle': return GRASS;
    case 'Savanna':                   return DRY_GRASS;
    case 'Desert':                    return SAND;
    case 'Mountains':                 return STONE;
    default:                          return SAND;      // Ocean, Beach
  }
}
function bodyType(name){
  switch (name){
    case 'Plains': case 'Oak Forest': case 'Jungle': case 'Savanna': return DIRT;
    case 'Desert':                    return SANDSTONE;
    case 'Mountains':                 return STONE;
    default:                          return SAND;
  }
}

// ---- per-chunk meshing ----
// Each loaded chunk gets one InstancedMesh per opaque block type (only the
// exposed blocks) plus one merged water surface. Meshes are tagged with their
// chunk + cell list so a raycast hit maps straight back to a voxel.
function disposeChunkMeshes(ch){
  if (!ch.meshes) return;
  ch.meshes.forEach(m => { scene.remove(m); if (m.isInstancedMesh) m.dispose(); else m.geometry.dispose(); });
  ch.meshes = []; ch.opaque = []; ch.water = [];
}
function buildSignMeshes(ch, ox, oz, arr){
  const n = arr.length;
  const post  = new THREE.InstancedMesh(SIGN_POST_GEO,  BLOCK_MAT[SIGN], n);
  const board = new THREE.InstancedMesh(SIGN_BOARD_GEO, BLOCK_MAT[SIGN], n);
  for (let i = 0; i < n; i++){
    const c = arr[i];
    _dummy.scale.set(1, 1, 1); _dummy.rotation.set(0, 0, 0);
    _dummy.position.set(ox + c[0] + 0.5, c[1] + 0.275, oz + c[2] + 0.5); _dummy.updateMatrix(); post.setMatrixAt(i, _dummy.matrix);
    _dummy.position.set(ox + c[0] + 0.5, c[1] + 0.73,  oz + c[2] + 0.5); _dummy.updateMatrix(); board.setMatrixAt(i, _dummy.matrix);
  }
  post.instanceMatrix.needsUpdate = true; board.instanceMatrix.needsUpdate = true;
  for (const m of [post, board]){
    m.userData = { ch, cells: arr }; m.castShadow = true; m.receiveShadow = true; m.frustumCulled = false;
    scene.add(m); ch.meshes.push(m); ch.opaque.push(m);
  }
}
function meshChunk(ch){
  disposeChunkMeshes(ch);
  const ox = ch.cx * CHUNK, oz = ch.cz * CHUNK;
  const data = ch.data;
  const lists = {}; BLOCK_TYPES.forEach(t => lists[t] = []);
  const water = [];
  const hairs = [];
  const topY = Math.min(WORLD_H, ch.maxY || WORLD_H);
  // fast solidity test: read this chunk's own array directly (no Map lookup) for in-bounds cells,
  // only falling back to blockWorld at chunk borders. Map.get was the meshing bottleneck.
  const solidAt = (lx, y, lz) => {
    if (y < 0) return true; if (y >= WORLD_H) return false;
    if (lx >= 0 && lx < CHUNK && lz >= 0 && lz < CHUNK){ const v = data[cIndex(lx, y, lz)]; return v !== AIR && v !== WATER; }
    return isOpaque(blockWorld(ox + lx, y, oz + lz));
  };
  const airAt = (lx, y, lz) => {
    if (y < 0) return false; if (y >= WORLD_H) return true;
    if (lx >= 0 && lx < CHUNK && lz >= 0 && lz < CHUNK) return data[cIndex(lx, y, lz)] === AIR;
    return blockWorld(ox + lx, y, oz + lz) === AIR;
  };
  for (let y = 0; y < topY; y++)
    for (let lz = 0; lz < CHUNK; lz++)
      for (let lx = 0; lx < CHUNK; lx++){
        const t = data[cIndex(lx, y, lz)];
        if (t === AIR) continue;
        if (t === WATER){
          if (airAt(lx+1,y,lz)||airAt(lx-1,y,lz)||airAt(lx,y+1,lz)||airAt(lx,y-1,lz)||airAt(lx,y,lz+1)||airAt(lx,y,lz-1)) water.push([lx, y, lz]);
        } else if (!solidAt(lx+1,y,lz)||!solidAt(lx-1,y,lz)||!solidAt(lx,y+1,lz)||!solidAt(lx,y-1,lz)||!solidAt(lx,y,lz+1)||!solidAt(lx,y,lz-1)){
          lists[t].push([lx, y, lz]);
          if (t === GRASS && airAt(lx, y+1, lz) && hash3(ox+lx, y, oz+lz, 909) < 0.3333) hairs.push([lx, y+1, lz]);
        }
      }
  ch.meshes = []; ch.opaque = []; ch.water = [];
  BLOCK_TYPES.forEach(t => {
    const arr = lists[t]; const n = arr.length; if (!n) return;
    if (t === SIGN){ buildSignMeshes(ch, ox, oz, arr); return; }   // signs aren't cubes
    const mesh = new THREE.InstancedMesh(UNIT_CUBE, BLOCK_MAT[t], n);
    for (let i = 0; i < n; i++){
      const c = arr[i];
      _dummy.position.set(ox + c[0] + 0.5, c[1] + 0.5, oz + c[2] + 0.5); _dummy.scale.set(1, 1, 1);
      _dummy.updateMatrix(); mesh.setMatrixAt(i, _dummy.matrix);
    }
    mesh.instanceMatrix.needsUpdate = true;
    mesh.userData = { ch, cells: arr };
    mesh.castShadow = true; mesh.receiveShadow = true;
    mesh.frustumCulled = false;                 // we frustum-cull per chunk ourselves (see cullChunks)
    scene.add(mesh); ch.meshes.push(mesh); ch.opaque.push(mesh);
  });
  if (water.length) buildChunkWater(ch, water);
  if (hairs.length){
    const hm = new THREE.InstancedMesh(GRASS_HAIR_GEO, GRASS_HAIR_MAT, hairs.length);
    for (let i = 0; i < hairs.length; i++){
      const c = hairs[i];
      _dummy.position.set(ox + c[0] + 0.5, c[1], oz + c[2] + 0.5);
      _dummy.rotation.set(0, (hash3(ox+c[0], c[1], oz+c[2], 13) * 1.57), 0);
      _dummy.scale.set(1, 1, 1); _dummy.updateMatrix(); hm.setMatrixAt(i, _dummy.matrix);
    }
    _dummy.rotation.set(0, 0, 0);
    hm.instanceMatrix.needsUpdate = true;
    hm.castShadow = false; hm.receiveShadow = false;
    hm.frustumCulled = false;                   // culled per chunk by cullChunks
    scene.add(hm); ch.meshes.push(hm);          // decoration: drawn + culled, but not a raycast/break target
  }
  ch.visibleNow = undefined;                     // re-evaluate frustum visibility next cull
  ch.built = true;
}

// merged water surface for one chunk: only the faces that touch open air
function chunkLoaded(wx, wz){ return chunks.has(ckey(floorDiv(wx, CHUNK), floorDiv(wz, CHUNK))); }
// a vertical water face is open only if the neighbour chunk is loaded AND that block is air;
// toward an unloaded chunk we assume the water continues, so no wall (keeps water seamless across chunks)
function waterSideOpen(wx, wy, wz){ return chunkLoaded(wx, wz) && blockWorld(wx, wy, wz) === AIR; }
function buildChunkWater(ch, cells){
  const ox = ch.cx * CHUNK, oz = ch.cz * CHUNK;
  const pos = [], nor = [], ind = []; let v = 0;
  const quad = (a,b,c,d,n) => { [a,b,c,d].forEach(p => { pos.push(p[0],p[1],p[2]); nor.push(n[0],n[1],n[2]); }); ind.push(v,v+1,v+2,v,v+2,v+3); v += 4; };
  for (let k = 0; k < cells.length; k++){
    const lx = cells[k][0], y = cells[k][1], lz = cells[k][2];
    const wx = ox + lx, wz = oz + lz;
    const x0 = wx, x1 = wx + 1, y0 = y, y1 = y + 1, z0 = wz, z1 = wz + 1;
    if (blockWorld(wx,y+1,wz)===AIR)  quad([x0,y1,z0],[x1,y1,z0],[x1,y1,z1],[x0,y1,z1],[0,1,0]);
    if (blockWorld(wx,y-1,wz)===AIR)  quad([x0,y0,z1],[x1,y0,z1],[x1,y0,z0],[x0,y0,z0],[0,-1,0]);
    if (waterSideOpen(wx+1,y,wz))     quad([x1,y0,z0],[x1,y0,z1],[x1,y1,z1],[x1,y1,z0],[1,0,0]);
    if (waterSideOpen(wx-1,y,wz))     quad([x0,y0,z1],[x0,y0,z0],[x0,y1,z0],[x0,y1,z1],[-1,0,0]);
    if (waterSideOpen(wx,y,wz+1))     quad([x1,y0,z1],[x0,y0,z1],[x0,y1,z1],[x1,y1,z1],[0,0,1]);
    if (waterSideOpen(wx,y,wz-1))     quad([x0,y0,z0],[x1,y0,z0],[x1,y1,z0],[x0,y1,z0],[0,0,-1]);
  }
  if (!pos.length) return;
  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.Float32BufferAttribute(pos, 3));
  geo.setAttribute('normal',   new THREE.Float32BufferAttribute(nor, 3));
  geo.setIndex(ind);
  const mesh = new THREE.Mesh(geo, BLOCK_MAT[WATER]);
  mesh.renderOrder = 1;
  mesh.frustumCulled = false;                      // culled per chunk by cullChunks
  mesh.layers.set(1);                              // reflection (cube) camera ignores layer 1 -> no water-in-water feedback
  scene.add(mesh); ch.meshes.push(mesh); ch.water.push(mesh);
}

// --- breaking blocks (creative: instant) ---
const raycaster = new THREE.Raycaster();
raycaster.far = 9;                          // reach distance, in blocks
const _ndc = new THREE.Vector2(0, 0);       // centre of screen (the crosshair)
const _frustum = new THREE.Frustum(), _projScreen = new THREE.Matrix4(), _fvec = new THREE.Vector3(), _box3 = new THREE.Box3();
// edits queue a re-mesh for next frame instead of rebuilding inline (stops the break/place stutter)
const dirtyMesh = new Set();
function markChunkDirty(cx, cz){ const ch = chunks.get(ckey(cx, cz)); if (ch && ch.built) dirtyMesh.add(ch); }
function flushDirtyMeshes(budget){
  let n = 0;
  for (const ch of dirtyMesh){
    if (n >= budget) break;
    dirtyMesh.delete(ch);
    if (chunks.has(ckey(ch.cx, ch.cz))) meshChunk(ch);
    n++;
  }
}
function setBlockWorld(wx, wy, wz, type){
  const cx = floorDiv(wx, CHUNK), cz = floorDiv(wz, CHUNK);
  const ch = chunks.get(ckey(cx, cz)); if (!ch) return;
  ch.data[cIndex(wx - cx*CHUNK, wy, wz - cz*CHUNK)] = type;
  if (wy + 1 > (ch.maxY || 0)) ch.maxY = Math.min(WORLD_H, wy + 1);
  edits.set(wx + ',' + wy + ',' + wz, type);          // remember the change for save / re-gen
  dirtyMesh.add(ch);
  const lx = wx - cx*CHUNK, lz = wz - cz*CHUNK;        // re-mesh neighbours if on a border
  if (lx === 0)         markChunkDirty(cx-1, cz);
  if (lx === CHUNK - 1) markChunkDirty(cx+1, cz);
  if (lz === 0)         markChunkDirty(cx, cz-1);
  if (lz === CHUNK - 1) markChunkDirty(cx, cz+1);
}
// ---- flowing water ----
// Placed/disturbed water spreads DOWN until it rests on a block, then sideways
// (N/S/E/W, never up), up to MAX_FLOW blocks from its source. Budgeted per frame.
const MAX_FLOW = 6;
const flowQueue = [];
const flowDist = new Map();
function flowKey(x, y, z){ return x + ',' + y + ',' + z; }
function seedFlow(wx, wy, wz, dist){ flowDist.set(flowKey(wx, wy, wz), dist || 0); flowQueue.push([wx, wy, wz]); }
function flowInto(wx, wy, wz, dist){
  if (wy < 1 || wy >= WORLD_H) return;
  if (blockWorld(wx, wy, wz) !== AIR) return;            // only flow into empty space
  const cx = floorDiv(wx, CHUNK), cz = floorDiv(wz, CHUNK);
  if (!chunks.has(ckey(cx, cz))) return;                 // don't flow into unloaded chunks
  setBlockWorld(wx, wy, wz, WATER);
  if (net.active) netSendEdit(wx, wy, wz, WATER);
  seedFlow(wx, wy, wz, dist);
}
// when a block is removed or water is placed, let nearby water re-flow into the gap
function nudgeWaterAround(wx, wy, wz){
  const around = [[0,1,0],[0,-1,0],[1,0,0],[-1,0,0],[0,0,1],[0,0,-1]];
  for (const [dx,dy,dz] of around){
    const x = wx+dx, y = wy+dy, z = wz+dz;
    if (blockWorld(x, y, z) === WATER) seedFlow(x, y, z, 0);
  }
}
function updateWaterFlow(){
  let budget = 48;                                       // gentle: a few dozen cells per tick
  while (flowQueue.length && budget-- > 0){
    const cell = flowQueue.shift();
    const wx = cell[0], wy = cell[1], wz = cell[2];
    if (blockWorld(wx, wy, wz) !== WATER){ flowDist.delete(flowKey(wx, wy, wz)); continue; }
    if (blockWorld(wx, wy-1, wz) === AIR){ flowInto(wx, wy-1, wz, 0); continue; }   // fall straight down
    const d = flowDist.get(flowKey(wx, wy, wz)) || 0;    // blocked below -> spread sideways within range
    if (d < MAX_FLOW){
      flowInto(wx+1, wy, wz, d+1); flowInto(wx-1, wy, wz, d+1);
      flowInto(wx, wy, wz+1, d+1); flowInto(wx, wy, wz-1, d+1);
    }
  }
}
function raycastBlock(){
  const targets = [];
  // reach (9 blocks) is less than a chunk, so only the player's chunk + its 8 neighbours can be hit.
  // Raycasting every loaded chunk's InstancedMesh tested millions of instances -> the per-click freeze.
  const pcx = floorDiv(Math.floor(benny.position.x), CHUNK), pcz = floorDiv(Math.floor(benny.position.z), CHUNK);
  for (let dz = -1; dz <= 1; dz++) for (let dx = -1; dx <= 1; dx++){
    const ch = chunks.get(ckey(pcx + dx, pcz + dz));
    if (ch && ch.opaque) for (const m of ch.opaque) targets.push(m);
  }
  raycaster.setFromCamera(_ndc, camera);
  const hits = raycaster.intersectObjects(targets, false);
  if (!hits.length) return null;
  const h = hits[0], ud = h.object.userData;
  const cell = ud.cells[h.instanceId]; if (!cell) return null;
  const wx = ud.ch.cx*CHUNK + cell[0], wy = cell[1], wz = ud.ch.cz*CHUNK + cell[2];
  const t = ud.ch.data[cIndex(cell[0], cell[1], cell[2])];
  if (t === AIR || t === WATER) return null;
  let n = { x:0, y:0, z:0 };
  if (h.face) n = { x: Math.round(h.face.normal.x), y: Math.round(h.face.normal.y), z: Math.round(h.face.normal.z) };
  return { wx, wy, wz, type: t, normal: n };
}

// deterministic 0..1 hash from a world column + the current seed (tree placement, layer jitter)
let worldSeed = 1337;
let worldMode = 'normal';     // 'normal' | 'flat' | 'koko' | '67' (flat-family worlds are all plains, no hills/oceans)
const FLAT_Y = WATER_Y + 2;   // ground level for flat-family worlds
const isFlatMode = () => worldMode !== 'normal';
function hash01(wx, wz, salt){
  let h = Math.imul(wx | 0, 73856093) ^ Math.imul(wz | 0, 19349663) ^ Math.imul(worldSeed | 0, 83492791) ^ Math.imul(salt | 0, 2654435761);
  h = Math.imul(h ^ (h >>> 15), 2246822519) >>> 0;
  return (h & 0xffffff) / 0x1000000;
}
function hash3(wx, wy, wz, salt){
  let h = Math.imul(wx | 0, 73856093) ^ Math.imul(wy | 0, 19349663) ^ Math.imul(wz | 0, 83492791) ^ Math.imul((worldSeed ^ salt) | 0, 2654435761);
  h = Math.imul(h ^ (h >>> 15), 2246822519) >>> 0;
  return (h & 0xffffff) / 0x1000000;
}
function treeAt(wx, wz){
  const sY = heightAt(wx, wz);
  if (sY <= WATER_Y) return null;                       // land only
  if (inVillage(wx, wz)) return null;                   // keep the village clearing tree-free
  const name = biomeName(wx, wz, sY);
  let dens = 0, kind = 'oak';
  if (name === 'Oak Forest')      dens = 0.16;
  else if (name === 'Jungle'){    dens = 0.13; kind = 'jungle'; }
  else if (name === 'Savanna')    dens = 0.012;          // hardly any trees
  else dens = 0;                                          // Plains (no trees), Desert, etc.
  if (dens === 0 || hash01(wx, wz, 7) >= dens) return null;
  const trunkH = kind === 'jungle' ? (7 + Math.floor(hash01(wx, wz, 19) * 5))   // 7..11 tall
                                   : (4 + Math.floor(hash01(wx, wz, 19) * 3));   // 4..6
  return { sY, trunkH, kind };
}
function cactusAt(wx, wz){
  const sY = heightAt(wx, wz);
  if (sY <= WATER_Y || inVillage(wx, wz)) return null;
  if (biomeName(wx, wz, sY) !== 'Desert') return null;
  if (hash01(wx, wz, 33) >= 0.018) return null;          // sparse
  return { sY, h: 2 + Math.floor(hash01(wx, wz, 34) * 2) };   // 2..3 tall
}
// rare sandstone temple ruins in the desert (one candidate per big region)
const TEMPLE_REGION = 768;
const templeCache = new Map();
function templeAt(rx, rz){
  const key = rx + ',' + rz;
  if (templeCache.has(key)) return templeCache.get(key);
  let t = null;
  if (hash01(rx, rz, 951) < 0.55){
    const wx = rx * TEMPLE_REGION + 120 + Math.floor(hash01(rx, rz, 952) * 400);
    const wz = rz * TEMPLE_REGION + 120 + Math.floor(hash01(rx, rz, 953) * 400);
    const sY = heightAt(wx, wz);
    if (sY > WATER_Y && biomeName(wx, wz, sY) === 'Desert') t = { wx, wz, sY };
  }
  templeCache.set(key, t); return t;
}
function stampTemple(data, ox, oz, T){
  const R = 4, H = 5;
  for (let k = 0; k < H; k++){
    const r = R - k; if (r < 0) break;
    for (let dx = -r; dx <= r; dx++) for (let dz = -r; dz <= r; dz++){
      const edge = (Math.abs(dx) === r || Math.abs(dz) === r);
      if (k > 0 && !edge) continue;                       // hollow upper layers, solid base
      const lx = T.wx + dx - ox, lz = T.wz + dz - oz, y = T.sY + 1 + k;
      if (lx < 0 || lx >= CHUNK || lz < 0 || lz >= CHUNK || y < 0 || y >= WORLD_H) continue;
      data[cIndex(lx, y, lz)] = SANDSTONE;
    }
  }
}

// ---------- villages (flat, tree-free plains settlements) ----------
const VREGION = 320;          // one candidate village per this-sized region of the world
const VRADIUS = 60;           // flattened, tree-free clearing radius (4x the old area)
const villageCache = new Map();
// the (possibly null) village anchor for a region cell; cached, validity needs flat Plains land
function villageCenter(rx, rz){
  const key = rx + ',' + rz; if (villageCache.has(key)) return villageCache.get(key);
  let v = null;
  if (hash01(rx, rz, 555) < 0.25){
    const gap = VRADIUS + 8, span = VREGION - 2 * gap;          // keep the whole clearing inside one region
    const wx = rx * VREGION + gap + Math.floor(hash01(rx, rz, 556) * span);
    const wz = rz * VREGION + gap + Math.floor(hash01(rx, rz, 557) * span);
    const level = heightAt(wx, wz);
    if (level >= WATER_Y + 1 && biomeName(wx, wz, level) === 'Plains') v = { wx, wz, level, rx, rz };
  }
  villageCache.set(key, v); return v;
}
// the village covering a column, or null (only its own region can reach it, since VRADIUS < edge gap)
function inVillage(wx, wz){
  if (isFlatMode()) return null;
  const v = villageCenter(floorDiv(wx, VREGION), floorDiv(wz, VREGION));
  if (!v) return null;
  const dx = wx - v.wx, dz = wz - v.wz;
  return (dx*dx + dz*dz <= VRADIUS*VRADIUS) ? v : null;
}
// ---- building stampers: each writes only the cells that fall inside this chunk ----
function stampHouse(put, bx, bz, L){
  const w = 8, d = 7, h = 5;
  for (let x = 0; x < w; x++) for (let z = 0; z < d; z++){ put(bx+x, L, bz+z, WOOD); put(bx+x, L+h, bz+z, WOOD); }
  for (let y = 1; y < h; y++) for (let x = 0; x < w; x++) for (let z = 0; z < d; z++)
    if (x===0 || x===w-1 || z===0 || z===d-1) put(bx+x, L+y, bz+z, WOOD);
  put(bx + (w>>1), L+1, bz, AIR); put(bx + (w>>1), L+2, bz, AIR);     // doorway
  put(bx, L+2, bz+2, AIR); put(bx+w-1, L+2, bz+2, AIR);              // windows
  put(bx, L+2, bz+4, AIR); put(bx+w-1, L+2, bz+4, AIR);
}
function stampSmith(put, bx, bz, L){
  const w = 7, d = 6, h = 5;
  for (let x = 0; x < w; x++) for (let z = 0; z < d; z++){ put(bx+x, L, bz+z, STONE); put(bx+x, L+h, bz+z, WOOD); }
  for (let y = 1; y < h; y++) for (let x = 0; x < w; x++) for (let z = 0; z < d; z++)
    if (x===0 || x===w-1 || z===0 || z===d-1) put(bx+x, L+y, bz+z, STONE);
  put(bx + (w>>1), L+1, bz, AIR); put(bx + (w>>1), L+2, bz, AIR);     // doorway
  // forge out front
  put(bx+w, L+1, bz+1, STONE); put(bx+w, L+2, bz+1, COAL_ORE);
  put(bx+w+1, L+1, bz+1, IRON_ORE); put(bx+w, L+1, bz+2, COAL_ORE);
}
function stampFarm(put, bx, bz, L){
  const w = 11, d = 9;
  for (let x = 0; x < w; x++) for (let z = 0; z < d; z++){
    if (x===0 || x===w-1 || z===0 || z===d-1){ put(bx+x, L+1, bz+z, WOOD); }   // fence posts
    else {
      const water = (x === (w>>1));
      put(bx+x, L, bz+z, water ? WATER : DIRT);
      if (!water) put(bx+x, L+1, bz+z, LEAF);                                   // crops
    }
  }
}
function stampVillage(data, cox, coz, v){
  const L = v.level, cx = v.wx, cz = v.wz;
  const put = (wx, wy, wz, t) => {
    const lx = wx - cox, lz = wz - coz;
    if (lx < 0 || lx >= CHUNK || lz < 0 || lz >= CHUNK || wy < 1 || wy >= WORLD_H) return;
    data[cIndex(lx, wy, lz)] = t;
  };
  const houseSlots = [[-30,-22],[-8,-26],[16,-24],[-34,6],[-14,10],[10,14]];      // 6 homes
  for (const s of houseSlots) stampHouse(put, cx + s[0], cz + s[1], L);
  stampSmith(put, cx + 24, cz - 6, L); stampSmith(put, cx + 28, cz + 14, L);      // 2 blacksmiths
  stampFarm(put, cx - 22, cz + 22, L); stampFarm(put, cx + 4, cz - 4, L);         // 2 farms
}

// build one chunk's voxel data from the noise fields (+ any saved edits)
// KOKOSMP's deterministic world gets a medieval village at spawn on flat ground,
// with a big castle on a hill to the north. Everything sits on solid, flattened ground.
const KOKO_SEED = 4242421;
// ---- Extreme Challenge parkour (deterministic, over the void) ----
const EXTREME_SEED = 91919191;
const PARK_Y = 64;                            // platform height
const PARK_VOID_Y = PARK_Y - 22;             // falling below this sends you back to the start circle
const PARK_START_R = 5;                       // start plank-circle radius
function buildParkour(){
  let s = (EXTREME_SEED ^ 0x9e3779b9) >>> 0;
  const rnd = () => { s = (s * 1664525 + 1013904223) >>> 0; return s / 4294967296; };
  const plats = [];
  let x = 0, y = PARK_Y, z = PARK_START_R;          // first jump lands 2 past the circle edge
  const N = 140;                                    // a long course
  for (let i = 0; i < N; i++){
    const dz = 2;                                   // always step forward (toward +z)
    const dx = ((rnd() * 3) | 0) - 1;               // wander -1..+1 sideways (≤1 keeps jumps fair)
    let dy = 0; const rr = rnd();
    if (rr < 0.18) dy = -1;                          // step down sometimes
    else if (rr > 0.90 && dx === 0) dy = 1;          // rare step up, only when not also moving sideways
    x += dx; z += dz; y += dy;
    if (y < PARK_Y - 10) y = PARK_Y - 10;
    if (y > PARK_Y + 6)  y = PARK_Y + 6;
    plats.push({ x, y, z });
  }
  return { plats, finish: plats[plats.length - 1] };
}
const PARKOUR = buildParkour();
function stampParkour(data, ox, oz){
  let top = 1;
  const put = (wx, wy, wz, t) => { const lx = wx - ox, lz = wz - oz; if (lx < 0 || lx >= CHUNK || lz < 0 || lz >= CHUNK || wy < 1 || wy >= WORLD_H) return; data[cIndex(lx, wy, lz)] = t; if (wy + 1 > top) top = wy + 1; };
  for (let dx = -PARK_START_R; dx <= PARK_START_R; dx++) for (let dz = -PARK_START_R; dz <= PARK_START_R; dz++)
    if (dx*dx + dz*dz <= PARK_START_R*PARK_START_R) put(dx, PARK_Y, dz, WOOD_PLANKS);      // round plank circle
  for (const p of PARKOUR.plats) put(p.x, p.y, p.z, WOOD_PLANKS);                            // course platforms
  const f = PARKOUR.finish;                                                                  // finish pad (diamond, 3x3)
  for (let dx = -1; dx <= 1; dx++) for (let dz = -1; dz <= 1; dz++) put(f.x + dx, f.y, f.z + dz, DIAMOND_ORE);
  return top;
}
// ---- Koko City: a deterministic 1000x1000 city of roads + buildings on flat ground ----
const CITY_HALF = 500, CITY_Y = WATER_Y + 2, CITY_GRID = 28, CITY_ROAD = 6;
function stampKokoCity(data, ox, oz){
  let top = CITY_Y + 1;
  const put = (lx, wy, lz, t) => { if (wy < 0 || wy >= WORLD_H) return; data[cIndex(lx, wy, lz)] = t; if (wy + 1 > top) top = wy + 1; };
  for (let lz = 0; lz < CHUNK; lz++) for (let lx = 0; lx < CHUNK; lx++){
    const wx = ox + lx, wz = oz + lz;
    for (let y = 0; y <= CITY_Y; y++) put(lx, y, lz, y === CITY_Y ? GRASS : (y >= CITY_Y - 3 ? DIRT : STONE));   // flat ground
    if (Math.abs(wx) > CITY_HALF || Math.abs(wz) > CITY_HALF) continue;                                          // outside the city: open grass
    const gx = ((wx % CITY_GRID) + CITY_GRID) % CITY_GRID, gz = ((wz % CITY_GRID) + CITY_GRID) % CITY_GRID;
    if (gx < CITY_ROAD || gz < CITY_ROAD){                                                                       // streets + sidewalks
      const sidewalk = (gx === CITY_ROAD - 1 || gz === CITY_ROAD - 1);
      put(lx, CITY_Y, lz, sidewalk ? CONCRETE : ROAD);
      if (gx === ((CITY_ROAD/2)|0) && gz === ((CITY_ROAD/2)|0)){ for (let y = 1; y <= 4; y++) put(lx, CITY_Y+y, lz, STONE); put(lx, CITY_Y+5, lz, GLASS_B); }  // lamp post at corners
      continue;
    }
    const lotX = gx - CITY_ROAD, lotZ = gz - CITY_ROAD, lotW = CITY_GRID - CITY_ROAD;        // building lot (22x22)
    const cellX = Math.floor(wx / CITY_GRID), cellZ = Math.floor(wz / CITY_GRID);
    const kind = hash01(cellX, cellZ, 9100);
    if (kind < 0.16) continue;                                                                // park lot: open grass
    if (lotX < 1 || lotZ < 1 || lotX > lotW - 2 || lotZ > lotW - 2) continue;                 // 1-block margin around the building
    const H = 6 + Math.floor(hash01(cellX, cellZ, 9101) * 26);                                // 6..31 storeys tall
    const mat = (hash01(cellX, cellZ, 9102) < 0.5) ? CONCRETE : BRICK;
    put(lx, CITY_Y, lz, CONCRETE);                                                           // floor slab
    const perim = (lotX === 1 || lotZ === 1 || lotX === lotW - 2 || lotZ === lotW - 2);
    if (perim){
      const door = (lotZ === 1 && lotX === (lotW >> 1));
      for (let y = 1; y <= H; y++){
        if (door && y <= 2) continue;                                                        // doorway
        const window = (y % 3 === 2) && (((wx * 3 + wz * 7) & 3) === 0);
        put(lx, CITY_Y + y, lz, window ? GLASS_B : mat);
      }
    }
    put(lx, CITY_Y + H + 1, lz, mat);                                                         // flat roof
  }
  return top;
}
// ---- Koko PvP Hub: a floating sky-island lobby with an 8x8 drop-hole over a walled arena ----
const HUB_ISLAND_R = 20, HUB_ARENA_R = 40;       // arena is twice the island's size
const HUB_SKY_Y = 96, HUB_ARENA_Y = 32;          // island top high in the sky; arena floor far below
const HUB_HOLE = 4;                              // central 8x8 drop hole (x,z in -4..3)
function hubInHole(wx, wz){ return wx >= -HUB_HOLE && wx <= HUB_HOLE - 1 && wz >= -HUB_HOLE && wz <= HUB_HOLE - 1; }
function stampPvpHub(data, ox, oz){
  let top = HUB_SKY_Y + 3;
  const put = (lx, wy, lz, t) => { if (wy < 0 || wy >= WORLD_H) return; data[cIndex(lx, wy, lz)] = t; if (wy + 1 > top) top = wy + 1; };
  for (let lz = 0; lz < CHUNK; lz++) for (let lx = 0; lx < CHUNK; lx++){
    const wx = ox + lx, wz = oz + lz, d = Math.sqrt(wx*wx + wz*wz);
    // ---- arena floor far below (plains + little hills, walled by a mountain rim) ----
    if (d <= HUB_ARENA_R){
      let g = HUB_ARENA_Y + Math.floor(elevNoise.fbm(wx, wz, 3, 0.05, 0.5) * 4);   // little rolling hills
      let mountain = false;
      if (d > HUB_ARENA_R - 12){ const tt = (d - (HUB_ARENA_R - 12)) / 12; g = HUB_ARENA_Y + Math.floor(tt*tt * 30) + 2; mountain = true; }  // mountain ring
      for (let y = 1; y <= g && y < WORLD_H; y++){
        const isTop = (y === g);
        const b = isTop ? (mountain && g > HUB_ARENA_Y + 16 ? STONE : GRASS) : (y >= g - 3 ? DIRT : STONE);
        put(lx, y, lz, b);
      }
    }
    // ---- floating sky-island lobby (with the central drop hole) ----
    if (d <= HUB_ISLAND_R && !hubInHole(wx, wz)){
      const thick = 4 + Math.floor((1 - d / HUB_ISLAND_R) * 5);                    // domed underside, thicker at centre
      for (let k = 0; k < thick; k++){ const y = HUB_SKY_Y - k; put(lx, y, lz, k === 0 ? GRASS : (k < 3 ? DIRT : STONE)); }
      if (d > HUB_ISLAND_R - 1.5){ put(lx, HUB_SKY_Y + 1, lz, WOOD); put(lx, HUB_SKY_Y + 2, lz, WOOD); }   // safety railing around the rim
    }
  }
  return top;
}
const TOWN = {
  baseY: WATER_Y + 4,                       // flat village ground level
  villR: 74,                                // village half-extent (flattened square) around origin
  hillX: 0, hillZ: 64,                      // castle hill centre, north of the village
  hillR: 42, hillTopR: 22, hillPeak: 16,    // big hill dome (flat top of radius hillTopR)
  stairHalf: 3,                             // grand staircase half-width (south face)
  castHalf: 16, keepHalf: 6,                // castle curtain-wall + central keep footprints
  fountainR: 4                              // fountain basin half-size (village centre)
};
const TOWN_HOUSES = [                        // [x, z, half] medieval houses on the flat village ground
  [-30,-12,4],[-46,4,4],[-30,16,4],[-15,18,4],[15,18,4],[30,16,4],[46,4,4],[30,-12,4],
  [-50,-26,4],[-30,-40,4],[30,-40,4],[50,-26,4],[-14,-48,4],[14,-48,4],
  [-62,10,3],[62,10,3],[-58,-10,3],[58,-10,3],[-44,-46,3],[44,-46,3]
];
function townHillH(wx, wz){
  const dx = wx - TOWN.hillX, dz = wz - TOWN.hillZ, d = Math.sqrt(dx*dx + dz*dz);
  if (d > TOWN.hillR) return 0;
  if (d <= TOWN.hillTopR) return TOWN.hillPeak;                       // flat hilltop for the castle
  const f = 1 - (d - TOWN.hillTopR) / (TOWN.hillR - TOWN.hillTopR);
  return Math.round(TOWN.hillPeak * f * f);                           // smooth grassy skirt
}
// one column of a hollow, enterable medieval house
function stampHouseCol(set, lx, lz, dx, dz, half, baseY){
  const ax = Math.abs(dx), az = Math.abs(dz);
  set(lx, baseY, lz, WOOD_PLANKS);                                    // floor
  const corner = (ax === half && az === half), onWall = (ax === half || az === half);
  if (corner){
    for (let y = baseY+1; y <= baseY+4; y++) set(lx, y, lz, WOOD);    // timber-frame corner posts
  } else if (onWall){
    const door = (dz === half && ax === 0);                          // door faces south
    const win  = (((ax === half) && (az % 2 === 0) && az <= half-2) || ((az === half) && (ax % 2 === 0) && ax <= half-2));
    for (let y = baseY+1; y <= baseY+4; y++){
      if (door && y <= baseY+2){ set(lx, y, lz, AIR); continue; }
      if (win && y === baseY+2){ set(lx, y, lz, GLASS_B); continue; }
      set(lx, y, lz, WOOD_PLANKS);
    }
  } else {
    for (let y = baseY+1; y <= baseY+4; y++) set(lx, y, lz, AIR);     // hollow interior — walk right in
  }
  set(lx, baseY+5, lz, BRICK);                                        // tiled roof
  if (ax <= half-1 && az <= half-1) set(lx, baseY+6, lz, BRICK);      // slight raised peak
}
// one column of the massive hilltop castle (dx,dz are relative to the hill centre)
function stampCastleCol(set, lx, lz, dx, dz, hy){
  const ax = Math.abs(dx), az = Math.abs(dz), H = TOWN.castHalf, K = TOWN.keepHalf;
  if (ax > H || az > H) return;                                       // outside the walls
  set(lx, hy, lz, STONE);                                             // cobbled courtyard / castle floor
  const corner = (ax >= H-1 && az >= H-1), onWall = (ax === H || az === H);
  if (corner){                                                        // four tall corner towers
    for (let y = hy+1; y <= hy+12; y++) set(lx, y, lz, STONE);
    if (((dx+dz) & 1) === 0) set(lx, hy+13, lz, STONE);               // battlements
    return;
  }
  if (onWall){                                                        // sandstone curtain wall
    const southGate = (dz === -H && ax <= 1);                         // grand gate faces the stairs (south)
    for (let y = hy+1; y <= hy+7; y++){
      if (southGate && y <= hy+4){ set(lx, y, lz, AIR); continue; }
      set(lx, y, lz, SANDSTONE);
    }
    if (!southGate && ((dx+dz) & 1) === 0) set(lx, hy+8, lz, SANDSTONE);
    return;
  }
  if (ax <= K && az <= K){                                            // central keep (tall, hollow, enterable)
    const kWall = (ax === K || az === K), keepDoor = (dz === -K && ax <= 1);
    if (kWall){
      for (let y = hy+1; y <= hy+14; y++){
        if (keepDoor && y <= hy+3){ set(lx, y, lz, AIR); continue; }
        if ((y === hy+6 || y === hy+10) && (((ax === K) && (az % 3 === 0)) || ((az === K) && (ax % 3 === 0)))){ set(lx, y, lz, GLASS_B); continue; }
        set(lx, y, lz, STONE);
      }
      if (((dx+dz) & 1) === 0) set(lx, hy+15, lz, STONE);             // keep battlements
    } else {
      set(lx, hy+14, lz, WOOD_PLANKS);                               // keep ceiling (floors below stay open)
    }
  }
}
function stampKokoTown(data, ox, oz){
  const T = TOWN, baseY = T.baseY, span = T.hillR - T.hillTopR;
  let top = baseY + 1;
  const set = (lx, y, lz, b) => { if (lx >= 0 && lx < CHUNK && lz >= 0 && lz < CHUNK && y >= 0 && y < WORLD_H) data[cIndex(lx, y, lz)] = b; };
  for (let lz = 0; lz < CHUNK; lz++) for (let lx = 0; lx < CHUNK; lx++){
    const wx = ox + lx, wz = oz + lz;
    const dxh = wx - T.hillX, dzh = wz - T.hillZ, dh = Math.sqrt(dxh*dxh + dzh*dzh);
    // grand staircase up the south face of the hill
    const onStair = (Math.abs(dxh) <= T.stairHalf && wz >= T.hillZ - T.hillR && wz <= T.hillZ - T.hillTopR);
    let domeH = townHillH(wx, wz);
    if (onStair){
      const run = wz - (T.hillZ - T.hillR);
      domeH = Math.max(0, Math.min(T.hillPeak, Math.round(run / span * T.hillPeak)));
    }
    const inVill = (Math.abs(wx) <= T.villR && wz >= -T.villR && wz <= T.hillZ + T.hillR);
    if (!inVill && domeH === 0) continue;                            // natural terrain elsewhere
    const gy = baseY + domeH;
    for (let y = gy + 1; y < WORLD_H; y++) data[cIndex(lx, y, lz)] = AIR;
    const surf = onStair ? STONE : GRASS;
    for (let y = 0; y <= gy && y < WORLD_H; y++) data[cIndex(lx, y, lz)] = (y === gy) ? surf : (y >= gy - 3 ? DIRT : STONE);
    if (gy + 1 > top) top = gy + 1;
    if (onStair && Math.abs(dxh) === T.stairHalf){ set(lx, gy + 1, lz, STONE); if (gy + 2 > top) top = gy + 2; }  // stair railings

    // ---- flat-ground village features ----
    if (domeH === 0 && inVill){
      if (Math.abs(wx) <= 2 || Math.abs(wz) <= 2) data[cIndex(lx, gy, lz)] = STONE;   // cobbled crossroads
      const fr = T.fountainR, ax = Math.abs(wx), az = Math.abs(wz);
      if (ax <= fr && az <= fr){                                      // central water fountain
        set(lx, baseY, lz, STONE);
        if (ax === fr || az === fr){ set(lx, baseY+1, lz, STONE); set(lx, baseY+2, lz, STONE); }     // basin rim
        else set(lx, baseY+1, lz, WATER);                                                            // pool
        if (ax <= 1 && az <= 1){ for (let y = baseY+1; y <= baseY+3; y++) set(lx, y, lz, STONE); set(lx, baseY+4, lz, WATER); }  // spout
        if (baseY + 5 > top) top = baseY + 5;
      }
      for (const h of TOWN_HOUSES){
        const dx = wx - h[0], dz = wz - h[1], half = h[2];
        if (Math.abs(dx) <= half && Math.abs(dz) <= half){ stampHouseCol(set, lx, lz, dx, dz, half, baseY); if (baseY + 7 > top) top = baseY + 7; break; }
      }
    }
    // ---- castle on the hilltop ----
    if (dh <= T.hillTopR + 0.5){
      const hy = baseY + T.hillPeak;
      stampCastleCol(set, lx, lz, dxh, dzh, hy);
      if (hy + 16 > top) top = hy + 16;
    }
  }
  return top;
}
// ---- extra world structures (normal worlds only), each placed on flat land ----
// sample terrain heights around a point; return the level if it's flat enough, else null
function groundFlat(wx, wz, half, maxVar){
  let lo = heightAt(wx, wz), hi = lo;
  for (const [dx,dz] of [[-half,-half],[half,-half],[-half,half],[half,half],[half,0],[-half,0],[0,half],[0,-half]]){
    const h = heightAt(wx+dx, wz+dz); if (h < lo) lo = h; if (h > hi) hi = h;
  }
  if (hi - lo > (maxVar == null ? 2 : maxVar)) return null;
  return Math.round((lo + hi) / 2);
}
// massive village: plains, ~1/7 of regions, 10+ houses + a central castle
const MV_REGION = 440, MV_RADIUS = 86, mvCache = new Map();
function massiveVillageAt(rx, rz){
  if (isFlatMode()) return null;
  const key = rx + ',' + rz; if (mvCache.has(key)) return mvCache.get(key);
  let v = null;
  if (hash01(rx, rz, 7001) < 1/7){
    const gap = MV_RADIUS + 10, span = MV_REGION - 2*gap;
    const wx = rx*MV_REGION + gap + Math.floor(hash01(rx, rz, 7002) * span);
    const wz = rz*MV_REGION + gap + Math.floor(hash01(rx, rz, 7003) * span);
    if (biomeName(wx, wz) === 'Plains'){ const lvl = groundFlat(wx, wz, 24, 6); if (lvl != null) v = { wx, wz, level: lvl }; }
  }
  mvCache.set(key, v); return v;
}
function stampMVCastle(put, cx, cz, L){
  const H = 12, KH = 4, wallH = 7, towerH = 11;
  for (let dx = -H; dx <= H; dx++) for (let dz = -H; dz <= H; dz++){
    const ax = Math.abs(dx), az = Math.abs(dz), wx = cx+dx, wz = cz+dz;
    const onWall = (ax === H || az === H), corner = (ax >= H-1 && az >= H-1);
    if (corner){ for (let y = 1; y <= towerH; y++) put(wx, L+y, wz, STONE); }
    else if (onWall){
      const gate = (dz === H && ax <= 1);
      for (let y = 1; y <= wallH; y++){ if (gate && y <= 4) continue; put(wx, L+y, wz, SANDSTONE); }
      if (((wx+wz) & 1) === 0) put(wx, L+wallH+1, wz, SANDSTONE);
    }
    if (ax <= KH && az <= KH){
      if (ax === KH || az === KH){ const gk = (dz === KH && ax <= 1); for (let y = 1; y <= wallH+2; y++){ if (gk && y <= 3) continue; put(wx, L+y, wz, STONE); } }
      put(wx, L+wallH+3, wz, SANDSTONE);
    }
  }
}
function stampMassiveVillage(data, cox, coz, v){
  const L = v.level, cx = v.wx, cz = v.wz, R = 80;
  const put = (wx, wy, wz, t) => { const lx = wx-cox, lz = wz-coz; if (lx<0||lx>=CHUNK||lz<0||lz>=CHUNK||wy<1||wy>=WORLD_H) return; data[cIndex(lx, wy, lz)] = t; };
  for (let lz = 0; lz < CHUNK; lz++) for (let lx = 0; lx < CHUNK; lx++){          // flatten the grounds
    const wx = cox+lx, wz = coz+lz; if (Math.abs(wx-cx) > R || Math.abs(wz-cz) > R) continue;
    for (let y = L+1; y < WORLD_H; y++) data[cIndex(lx, y, lz)] = AIR;
    for (let y = 0; y <= L; y++) data[cIndex(lx, y, lz)] = (y === L) ? GRASS : (y >= L-3 ? DIRT : STONE);
  }
  for (let gx = -3; gx <= 3; gx++) for (let gz = -3; gz <= 3; gz++){              // 10+ houses on a grid
    if (Math.abs(gx) <= 1 && Math.abs(gz) <= 1) continue;                         // keep the centre for the castle
    if (hash01(cx+gx*100, cz+gz*100, 7100) < 0.8) stampHouse(put, cx + gx*22 - 4, cz + gz*22 - 3, L);
  }
  stampMVCastle(put, cx, cz, L);
}
// ruined building: anywhere except jungle/desert, broken stone walls
const RUIN_REGION = 224, ruinCache = new Map();
function ruinAt(rx, rz){
  if (isFlatMode()) return null;
  const key = rx + ',' + rz; if (ruinCache.has(key)) return ruinCache.get(key);
  let v = null;
  if (hash01(rx, rz, 7201) < 0.5){
    const wx = rx*RUIN_REGION + 24 + Math.floor(hash01(rx, rz, 7202) * (RUIN_REGION-48));
    const wz = rz*RUIN_REGION + 24 + Math.floor(hash01(rx, rz, 7203) * (RUIN_REGION-48));
    const b = biomeName(wx, wz);
    if (b !== 'Jungle' && b !== 'Desert' && b !== 'Ocean' && b !== 'Beach'){ const lvl = groundFlat(wx, wz, 7, 2); if (lvl != null) v = { wx, wz, level: lvl }; }
  }
  ruinCache.set(key, v); return v;
}
function stampRuin(data, cox, coz, v){
  const L = v.level, cx = v.wx, cz = v.wz, half = 5;
  const put = (wx, wy, wz, t) => { const lx = wx-cox, lz = wz-coz; if (lx<0||lx>=CHUNK||lz<0||lz>=CHUNK||wy<1||wy>=WORLD_H) return; data[cIndex(lx, wy, lz)] = t; };
  for (let dx = -half; dx <= half; dx++) for (let dz = -half; dz <= half; dz++){
    const wx = cx+dx, wz = cz+dz, edge = (Math.abs(dx) === half || Math.abs(dz) === half);
    put(wx, L, wz, STONE);                                                        // cracked floor
    if (edge){                                                                    // crumbling walls (random height, gaps)
      const hh = 1 + Math.floor(hash01(wx, wz, 7300) * 4);
      for (let y = 1; y <= hh; y++){ if (hash01(wx, wz*3+y, 7301) < 0.78) put(wx, L+y, wz, hash01(wx,y,wz,7302) < 0.3 ? COAL_ORE : STONE); }
    }
  }
}
// fishing village: on big oceans, huts on stilts above the water
const FISH_REGION = 288, fishCache = new Map();
function fishingVillageAt(rx, rz){
  if (isFlatMode()) return null;
  const key = rx + ',' + rz; if (fishCache.has(key)) return fishCache.get(key);
  let v = null;
  if (hash01(rx, rz, 7401) < 1/7){
    const wx = rx*FISH_REGION + 40 + Math.floor(hash01(rx, rz, 7402) * (FISH_REGION-80));
    const wz = rz*FISH_REGION + 40 + Math.floor(hash01(rx, rz, 7403) * (FISH_REGION-80));
    // require deep, open ocean: the centre and all around must be ocean
    let ocean = true;
    for (const [dx,dz] of [[0,0],[26,0],[-26,0],[0,26],[0,-26]]) if (biomeName(wx+dx, wz+dz) !== 'Ocean') ocean = false;
    if (ocean) v = { wx, wz };
  }
  fishCache.set(key, v); return v;
}
function stampFishingVillage(data, cox, coz, v){
  const cx = v.wx, cz = v.wz, deck = WATER_Y;                                     // platforms sit at the water surface
  const put = (wx, wy, wz, t) => { const lx = wx-cox, lz = wz-coz; if (lx<0||lx>=CHUNK||lz<0||lz>=CHUNK||wy<1||wy>=WORLD_H) return; data[cIndex(lx, wy, lz)] = t; };
  const hut = (hx, hz) => {                                                       // a stilted plank hut over the water
    for (let x = -2; x <= 2; x++) for (let z = -2; z <= 2; z++) put(hx+x, deck, hz+z, WOOD_PLANKS);   // deck
    for (const [sx,sz] of [[-2,-2],[2,-2],[-2,2],[2,2]]) for (let y = deck-1; y > WATER_Y-5; y--) put(hx+sx, y, hz+sz, WOOD);  // stilts down into water
    for (let x = -2; x <= 2; x++) for (let z = -2; z <= 2; z++){ const edge = (Math.abs(x)===2||Math.abs(z)===2); if (edge) for (let y = 1; y <= 3; y++) put(hx+x, deck+y, hz+z, WOOD_PLANKS); }
    put(hx, deck+1, hz-2, AIR); put(hx, deck+2, hz-2, AIR);                       // door
    for (let x = -2; x <= 2; x++) for (let z = -2; z <= 2; z++) put(hx+x, deck+4, hz+z, WOOD);          // roof
  };
  hut(cx, cz); hut(cx+8, cz+2); hut(cx-7, cz+4); hut(cx+2, cz-8); hut(cx-5, cz-6);
  for (let x = cx-7; x <= cx+8; x++) put(x, deck, cz, WOOD_PLANKS);               // a connecting boardwalk
}
// giant tree: plains, ~1/50 of small regions
const GT_REGION = 110, gtCache = new Map();
function giantTreeAt(rx, rz){
  if (isFlatMode()) return null;
  const key = rx + ',' + rz; if (gtCache.has(key)) return gtCache.get(key);
  let v = null;
  if (hash01(rx, rz, 7501) < 1/50){
    const wx = rx*GT_REGION + 20 + Math.floor(hash01(rx, rz, 7502) * (GT_REGION-40));
    const wz = rz*GT_REGION + 20 + Math.floor(hash01(rx, rz, 7503) * (GT_REGION-40));
    if (biomeName(wx, wz) === 'Plains'){ const lvl = groundFlat(wx, wz, 8, 2); if (lvl != null) v = { wx, wz, level: lvl }; }
  }
  gtCache.set(key, v); return v;
}
function stampGiantTree(data, cox, coz, v){
  const L = v.level, cx = v.wx, cz = v.wz, trunkH = 26, R = 11;
  const put = (wx, wy, wz, t, onlyair) => { const lx = wx-cox, lz = wz-coz; if (lx<0||lx>=CHUNK||lz<0||lz>=CHUNK||wy<1||wy>=WORLD_H) return; if (onlyair && data[cIndex(lx,wy,lz)] !== AIR) return; data[cIndex(lx, wy, lz)] = t; };
  for (let y = 1; y <= trunkH; y++) for (let dx = -1; dx <= 1; dx++) for (let dz = -1; dz <= 1; dz++) put(cx+dx, L+y, cz+dz, WOOD);   // fat 3x3 trunk
  const cyc = L + trunkH;                                                          // canopy centre
  for (let dx = -R; dx <= R; dx++) for (let dy = -6; dy <= 6; dy++) for (let dz = -R; dz <= R; dz++){
    const dd = Math.sqrt(dx*dx + (dy*1.5)*(dy*1.5) + dz*dz);
    if (dd > R) continue;
    if (hash3(cx+dx, cyc+dy, cz+dz, 7600) < 0.18) continue;                        // a few natural gaps
    put(cx+dx, cyc+dy, cz+dz, LEAF, true);
  }
}
// secret ".67" world: a giant "67" monument in blocks at spawn
const DIG67 = { '6': ["01110","10000","10000","11110","10001","10001","01110"], '7': ["11111","00001","00010","00100","00100","01000","01000"] };
function stamp67(data, ox, oz){
  const S = 3, baseY = FLAT_Y + 1, z0 = 8, gap = 4; let top = baseY;
  let xCursor = -((5*S*2 + gap) >> 1);
  for (const d of ['6','7']){
    const pat = DIG67[d];
    for (let row = 0; row < 7; row++) for (let col = 0; col < 5; col++){
      if (pat[row][col] !== '1') continue;
      for (let sx = 0; sx < S; sx++) for (let sy = 0; sy < S; sy++) for (let sz = 0; sz < 2; sz++){
        const wx = xCursor + col*S + sx, wy = baseY + (6-row)*S + sy, wz = z0 + sz;
        const lx = wx-ox, lz = wz-oz;
        if (lx>=0 && lx<CHUNK && lz>=0 && lz<CHUNK && wy>=0 && wy<WORLD_H){ data[cIndex(lx, wy, lz)] = DIAMOND_ORE; if (wy+1 > top) top = wy+1; }
      }
    }
    xCursor += 5*S + gap;
  }
  return top;
}
function genChunkData(cx, cz){
  const data = new Uint8Array(CHUNK * CHUNK * WORLD_H);
  const ox = cx * CHUNK, oz = cz * CHUNK;
  let maxFill = 1;                                                  // highest filled layer (+1) for cheaper meshing
  if (worldMode === 'extreme' || worldMode === 'kokocity' || worldMode === 'pvphub'){    // built worlds: stamp then edits
    let top = (worldMode === 'extreme') ? stampParkour(data, ox, oz) : (worldMode === 'kokocity') ? stampKokoCity(data, ox, oz) : stampPvpHub(data, ox, oz);
    if (top > maxFill) maxFill = top;
    edits.forEach((val, k) => { const p = k.split(','); const ex = +p[0], ey = +p[1], ez = +p[2];
      if (floorDiv(ex, CHUNK) === cx && floorDiv(ez, CHUNK) === cz && ey >= 0 && ey < WORLD_H){ data[cIndex(ex - ox, ey, ez - oz)] = val; if (ey + 1 > maxFill) maxFill = ey + 1; } });
    return { data, maxY: Math.min(WORLD_H, maxFill + 1) };
  }
  for (let lz = 0; lz < CHUNK; lz++){
    for (let lx = 0; lx < CHUNK; lx++){
      const wx = ox + lx, wz = oz + lz;
      let sY = heightAt(wx, wz);
      let name = biomeName(wx, wz, sY);
      const vil = inVillage(wx, wz);
      if (vil){ sY = vil.level; name = 'Plains'; }                 // flatten the village clearing
      const surf = surfaceType(name), body = bodyType(name);
      const dirtN = hash01(wx, wz, 3) < 0.5 ? 3 : 4;               // 3-4 sub-surface layers
      for (let y = 0; y <= sY && y < WORLD_H; y++){
        const depth = sY - y;                                      // 0 = top block
        let block = depth === 0 ? surf : depth <= dirtN ? body : STONE;
        if (block === STONE){                                      // scatter ores by depth + rarity
          if (y < 14 && hash3(wx, y, wz, 101) < 0.004)            block = DIAMOND_ORE;   // deepest, rarest
          else if (depth > 12 && hash3(wx, y, wz, 102) < 0.012)   block = IRON_ORE;      // a bit deeper than coal
          else if (depth > 4 && hash3(wx, y, wz, 103) < 0.03)     block = COAL_ORE;      // near the surface
        }
        // caves: hollow out the rock underground, leaving the surface shell intact
        if (y > 1 && depth > 2 && y < sY && !isFlatMode() && caveCarve(wx, y, wz)) block = AIR;
        data[cIndex(lx, y, lz)] = block;
      }
      if (sY + 1 > maxFill) maxFill = sY + 1;
      if (sY < WATER_Y - 1){                                       // ocean basin -> water up to sea level
        for (let y = sY + 1; y < WATER_Y && y < WORLD_H; y++) data[cIndex(lx, y, lz)] = WATER;
        if (WATER_Y > maxFill) maxFill = WATER_Y;
      }
      const cac = cactusAt(wx, wz);                                // desert cacti (single column)
      if (cac){ for (let y = cac.sY + 1; y <= cac.sY + cac.h && y < WORLD_H; y++) data[cIndex(lx, y, lz)] = CACTUS; if (cac.sY + cac.h + 1 > maxFill) maxFill = cac.sY + cac.h + 1; }
    }
  }
  // trees: scan one column past each edge so canopies cross chunk borders seamlessly
  if (!isFlatMode()) for (let oxx = -1; oxx <= CHUNK; oxx++){
    for (let ozz = -1; ozz <= CHUNK; ozz++){
      const t = treeAt(ox + oxx, oz + ozz); if (!t) continue;
      const jungle = t.kind === 'jungle';
      const woodB = jungle ? JUNGLE_WOOD : WOOD, leafB = jungle ? JUNGLE_LEAF : LEAF;
      const R = jungle ? 3 : 2;                                    // jungle canopies are wider
      if (oxx >= 0 && oxx < CHUNK && ozz >= 0 && ozz < CHUNK)      // trunk lives in its own chunk
        for (let y = t.sY + 1; y <= t.sY + t.trunkH && y < WORLD_H; y++) data[cIndex(oxx, y, ozz)] = woodB;
      const leafC = t.sY + t.trunkH + 1;                          // canopy centre, just above the trunk
      if (leafC + 3 > maxFill) maxFill = leafC + 3;
      // irregular blob: dense core with hash-thinned gaps, plus a few leaves sticking out
      for (let dx = -R; dx <= R; dx++)
        for (let dy = -1; dy <= 2; dy++)
          for (let dz = -R; dz <= R; dz++){
            const r = Math.abs(dx) + Math.abs(dy) + Math.abs(dz);
            if (r === 0) continue;
            const h = hash3(ox + oxx + dx, leafC + dy, oz + ozz + dz, 4500);
            let keep;
            if (r <= R) keep = h > 0.14;          // dense core, a few natural gaps
            else if (r === R + 1) keep = h > 0.6; // protruding leaves stick out (~40%)
            else keep = false;
            if (!keep) continue;
            const lx = oxx + dx, lz = ozz + dz, y = leafC + dy;
            if (lx < 0 || lx >= CHUNK || lz < 0 || lz >= CHUNK || y < 0 || y >= WORLD_H) continue;
            if (data[cIndex(lx, y, lz)] === AIR) data[cIndex(lx, y, lz)] = leafB;
            // jungle: hang a short vine off the underside of edge leaves
            if (jungle && dy <= 0 && r >= R && (hash3(ox + oxx + dx, y, oz + ozz + dz, 8800) > 0.55)){
              for (let vy = y - 1; vy >= y - (1 + ((hash3(ox+oxx+dx, vy, oz+ozz+dz, 8801) * 3) | 0)) && vy > t.sY; vy--){
                if (lx < 0 || lx >= CHUNK || lz < 0 || lz >= CHUNK || vy < 0 || vy >= WORLD_H) break;
                if (data[cIndex(lx, vy, lz)] === AIR) data[cIndex(lx, vy, lz)] = VINE; else break;
              }
            }
          }
      // an occasional tall sprig poking out the top
      if (oxx >= 0 && oxx < CHUNK && ozz >= 0 && ozz < CHUNK && hash3(ox + oxx, leafC + 3, oz + ozz, 71) > 0.5){
        const y = leafC + 3; if (y < WORLD_H && data[cIndex(oxx, y, ozz)] === AIR) data[cIndex(oxx, y, ozz)] = leafB;
      }
    }
  }
  // ---- structures (normal worlds only) ----
  if (!isFlatMode()){
    const corners = [[ox,oz],[ox+CHUNK-1,oz],[ox,oz+CHUNK-1],[ox+CHUNK-1,oz+CHUNK-1]];
    const regSet = (size) => { const s = new Set(); corners.forEach(([px,pz]) => s.add(floorDiv(px,size)+','+floorDiv(pz,size))); return s; };
    // procedural villages
    regSet(VREGION).forEach(k => { const p = k.split(','); const v = villageCenter(+p[0], +p[1]); if (v){ stampVillage(data, ox, oz, v); if (v.level + 8 > maxFill) maxFill = v.level + 8; } });
    // desert temples
    regSet(TEMPLE_REGION).forEach(k => { const p = k.split(','); const T = templeAt(+p[0], +p[1]); if (T){ stampTemple(data, ox, oz, T); if (T.sY + 7 > maxFill) maxFill = T.sY + 7; } });
    // massive villages (plains, 1/7) — houses + castle
    regSet(MV_REGION).forEach(k => { const p = k.split(','); const m = massiveVillageAt(+p[0], +p[1]); if (m){ stampMassiveVillage(data, ox, oz, m); if (m.level + 18 > maxFill) maxFill = m.level + 18; } });
    // ruined buildings (not jungle/desert/ocean)
    regSet(RUIN_REGION).forEach(k => { const p = k.split(','); const r = ruinAt(+p[0], +p[1]); if (r){ stampRuin(data, ox, oz, r); if (r.level + 6 > maxFill) maxFill = r.level + 6; } });
    // fishing villages (open ocean, on the water)
    regSet(FISH_REGION).forEach(k => { const p = k.split(','); const fv = fishingVillageAt(+p[0], +p[1]); if (fv){ stampFishingVillage(data, ox, oz, fv); if (WATER_Y + 6 > maxFill) maxFill = WATER_Y + 6; } });
    // giant trees (plains, 1/50)
    regSet(GT_REGION).forEach(k => { const p = k.split(','); const g = giantTreeAt(+p[0], +p[1]); if (g){ stampGiantTree(data, ox, oz, g); if (g.level + 40 > maxFill) maxFill = g.level + 40; } });
  }
  // secret ".67" world: a giant "67" monument at spawn
  if (worldMode === '67'){ const t67 = stamp67(data, ox, oz); if (t67 > maxFill) maxFill = t67; }
  // KOKOSMP medieval village + castle hill (deterministic, only in the koko-seed world), stamped before edits
  if (worldSeed === KOKO_SEED){
    const x0 = ox, x1 = ox + CHUNK - 1, z0 = oz, z1 = oz + CHUNK - 1;
    if (x1 >= -TOWN.villR - 4 && x0 <= TOWN.villR + 4 && z1 >= -TOWN.villR - 4 && z0 <= TOWN.hillZ + TOWN.hillR + 4){
      const ctop = stampKokoTown(data, ox, oz); if (ctop > maxFill) maxFill = ctop;
    }
  }
  // re-apply any player edits that fall inside this chunk
  edits.forEach((val, k) => {
    const p = k.split(','); const ex = +p[0], ey = +p[1], ez = +p[2];
    if (floorDiv(ex, CHUNK) === cx && floorDiv(ez, CHUNK) === cz && ey >= 0 && ey < WORLD_H){
      data[cIndex(ex - ox, ey, ez - oz)] = val; if (ey + 1 > maxFill) maxFill = ey + 1;
    }
  });
  return { data, maxY: Math.min(WORLD_H, maxFill + 1) };
}

function ensureChunk(cx, cz){
  const k = ckey(cx, cz);
  let ch = chunks.get(k);
  if (!ch){ const g = genChunkData(cx, cz); ch = { cx, cz, data: g.data, maxY: g.maxY, meshes: [], opaque: [], water: [], built: false }; chunks.set(k, ch); }
  return ch;
}

// ---- streaming load / unload around the player ----
let renderChunks = 8;                 // render distance in chunks (set from settings)
let chunksDirty = true;
let frameTick = 0;
let autosaveT = 20;
let lobbyBeatT = 12;          // re-advertise the open world to the directory periodically
const buildQueue = [];
const chebyshev = (a, b) => Math.max(Math.abs(a), Math.abs(b));
const world = { finite: false, half: 256 };   // custom worlds are a bounded square: x,z in [-half, half]

function chunkInBounds(cx, cz){
  if (!world.finite) return true;
  const minC = floorDiv(-world.half, CHUNK), maxC = floorDiv(world.half - 1, CHUNK);
  return cx >= minC && cx <= maxC && cz >= minC && cz <= maxC;
}
function activeRadius(){ return orbiting ? Math.min(renderChunks, 5) : renderChunks; }   // cheap backdrop vs full world

function refreshChunkQueue(){
  const pcx = floorDiv(Math.floor(benny.position.x), CHUNK);
  const pcz = floorDiv(Math.floor(benny.position.z), CHUNK);
  const r = activeRadius();
  chunks.forEach((ch, k) => {                                   // unload chunks outside the radius
    if (chebyshev(ch.cx - pcx, ch.cz - pcz) > r + 1){ disposeChunkMeshes(ch); chunks.delete(k); }
  });
  buildQueue.length = 0;                                        // queue missing chunks, nearest first
  for (let dz = -r; dz <= r; dz++)
    for (let dx = -r; dx <= r; dx++){
      const cx = pcx + dx, cz = pcz + dz;
      if (!chunkInBounds(cx, cz)) continue;                     // respect a custom world's edge
      const ch = chunks.get(ckey(cx, cz));
      if (!ch || !ch.built) buildQueue.push([cx, cz, dx*dx + dz*dz]);
    }
  buildQueue.sort((a, b) => a[2] - b[2]);
  chunksDirty = false;
}
function processChunkQueue(budget){
  let done = 0;
  while (buildQueue.length && done < budget){
    const item = buildQueue.shift();
    const ch = ensureChunk(item[0], item[1]);
    if (!ch.built) meshChunk(ch);
    done++;
  }
  return buildQueue.length;
}
function targetChunkCount(){ const n = renderChunks * 2 + 1; return n * n; }

// performance mode LOD: only chunks near the player cast shadows
const DETAIL_NEAR = 4;
function applyDetail(){
  const pcx = floorDiv(Math.floor(benny.position.x), CHUNK);
  const pcz = floorDiv(Math.floor(benny.position.z), CHUNK);
  chunks.forEach(ch => {
    const lowFar = settings.lowDetail && chebyshev(ch.cx - pcx, ch.cz - pcz) > DETAIL_NEAR;
    if (ch.opaque) for (const m of ch.opaque) m.castShadow = !lowFar;
  });
}

// correct per-chunk frustum culling: a chunk's InstancedMeshes share a unit-cube geometry, so
// three.js can't cull them by extent — we test each chunk's real box against the camera frustum
// and toggle its meshes' visibility ourselves, so only chunks actually in view are drawn.
function cullChunks(){
  camera.updateMatrixWorld();
  _projScreen.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
  _frustum.setFromProjectionMatrix(_projScreen);
  chunks.forEach(ch => {
    if (!ch.meshes || !ch.meshes.length) return;
    const ox = ch.cx * CHUNK, oz = ch.cz * CHUNK;
    _box3.min.set(ox, 0, oz); _box3.max.set(ox + CHUNK, ch.maxY || WORLD_H, oz + CHUNK);
    const vis = _frustum.intersectsBox(_box3);
    if (ch.visibleNow === vis) return;                 // only touch meshes when it actually changes
    ch.visibleNow = vis;
    for (const m of ch.meshes) m.visible = vis;
  });
}
function setAllChunksVisible(v){
  chunks.forEach(ch => { if (!ch.meshes) return; ch.visibleNow = v; for (const m of ch.meshes) m.visible = v; });
}

// reseed and queue a brand-new world (the loading screen drains the queue)
function startNewWorld(seed){
  worldSeed = seed; seedWorld(seed);
  edits.clear(); clearInventory(); clearMobs(); clearSigns(); resetVitals(); resetXp();
  chunks.forEach(ch => disposeChunkMeshes(ch)); chunks.clear();
  placeBenny();
  chunksDirty = true; refreshChunkQueue();
}

function placeBenny(){
  if (worldMode === 'extreme'){ benny.position.set(0.5, PARK_Y + 1, 0.5); return; }          // start circle
  if (worldMode === 'kokocity'){ benny.position.set(0.5, CITY_Y + 1, 0.5); return; }          // city centre
  if (worldMode === 'pvphub'){ benny.position.set(10.5, HUB_SKY_Y + 1, 10.5); return; }        // on the sky island, clear of the hole
  if (worldSeed === KOKO_SEED){ benny.position.set(TOWN.hillX + 0.5, TOWN.baseY + TOWN.hillPeak + 2, TOWN.hillZ - 11); yaw = Math.PI; benny.rotation.y = Math.PI; return; }   // castle courtyard, looking out over the village
  const sY = Math.max(heightAt(0, 0), WATER_Y);
  benny.position.set(0.5, sY + 4, 0.5);          // hover just above the surface at the origin
}
function biomeAt(wx, wz){ return biomeName(Math.floor(wx), Math.floor(wz)); }

// ---------- SECTION 5: Benny (the character you control) ----------
// 2 blocks tall, roughly 1 block thick. Feet sit at the group origin (y = 0),
// head tops out at y = 2.0.
function makeBenny(){
  const g = new THREE.Group();
  const part = (w, h, d, color, x, y, z) => {
    const m = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), new THREE.MeshLambertMaterial({ color }));
    m.position.set(x, y, z); g.add(m); return m;
  };
  const PANTS = 0x33405c, SHIRT = 0xcf6a48, SKIN = 0xf2c79e, HAIR = 0x4a3322, BOOT = 0x241c14;

  const legL = part(0.26, 0.80, 0.42, PANTS, -0.14, 0.40, 0);   // left leg   (0.00–0.80)
  const legR = part(0.26, 0.80, 0.42, PANTS,  0.14, 0.40, 0);   // right leg
  part(0.30, 0.16, 0.46, BOOT, -0.14, 0.08, 0.03);              // left boot
  part(0.30, 0.16, 0.46, BOOT,  0.14, 0.08, 0.03);              // right boot
  part(0.58, 0.70, 0.40, SHIRT, 0,    1.15, 0);                 // torso      (0.80–1.50)
  const armL = part(0.20, 0.70, 0.34, SHIRT, -0.39, 1.15, 0);   // left arm
  const armR = part(0.20, 0.70, 0.34, SHIRT,  0.39, 1.15, 0);   // right arm
  part(0.50, 0.50, 0.50, SKIN, 0, 1.75, 0);                     // head       (1.50–2.00)
  part(0.54, 0.16, 0.54, HAIR, 0, 1.96, 0);                     // hair on top
  part(0.54, 0.18, 0.10, HAIR, 0, 1.78, -0.24);                 // hair at back

  g.userData.limbs = { legL, legR, armL, armR };
  return g;
}
const benny = makeBenny();
benny.traverse(o => { if (o.isMesh){ o.castShadow = true; o.receiveShadow = true; } });
scene.add(benny);
const BENNY_EYE = 1.72;             // first-person eye height (near the head)
let stretch = 0;   // current squash/stretch amount, eased every frame

// ---------- SECTION 5b: first-person hand viewmodel ----------
// Benny's bare (skin) arm pinned to the camera, bottom-RIGHT like Minecraft's
// main hand. Held mostly upright with a slight lean to the left, and rooted
// BELOW the screen edge so it reads as coming up from the bottom (never floats).
// Routed through the transparent pass + high render order + no depth test, so it
// always draws on top — even over the translucent water plane.
const offhand = new THREE.Group();
const OFFHAND_BASE = new THREE.Vector3(0.42, -1.04, -1.0);  // base sits below the screen bottom
(function buildOffhand(){
  const SKIN = 0xf2c79e;
  const mat = new THREE.MeshLambertMaterial({
    color: SKIN, transparent: true, fog: false,  // transparent pass = drawn after water
    depthTest: false, depthWrite: false           // never clipped by world geometry
  });
  // a single skin block, rising from the pivot (local y = 0 is the bottom)
  const arm = new THREE.Mesh(new THREE.BoxGeometry(0.24, 0.82, 0.24), mat);
  arm.position.y = 0.41; arm.renderOrder = 9999; arm.castShadow = false;
  offhand.add(arm);
})();
offhand.rotation.set(0.05, 0.0, 0.14);     // upright, leaning a little to the left
offhand.position.copy(OFFHAND_BASE);
camera.add(offhand);
scene.add(camera);                          // so the camera's children get rendered

// held-item viewmodel: shows the equipped block/item in the hand (it's parented to the
// offhand group, so it bobs and swings right along with the arm)
const heldBlockMat = new THREE.MeshLambertMaterial({ color: 0xffffff, transparent: true, fog: false, depthTest: false, depthWrite: false });
const heldFlatMat  = new THREE.MeshBasicMaterial({ transparent: true, fog: false, depthTest: false, depthWrite: false, alphaTest: 0.35, side: THREE.DoubleSide });
const heldBlockMesh = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.5, 0.5), heldBlockMat);
const heldFlatMesh  = new THREE.Mesh(new THREE.PlaneGeometry(0.64, 0.64), heldFlatMat);
heldBlockMesh.renderOrder = 10000; heldFlatMesh.renderOrder = 10000;
heldBlockMesh.castShadow = false; heldFlatMesh.castShadow = false;
heldBlockMesh.position.set(0.0, 0.7, -0.18); heldBlockMesh.rotation.set(0.35, 0.7, 0.0);
heldFlatMesh.position.set(0.0, 0.64, -0.16); heldFlatMesh.rotation.set(0.1, -0.25, 0.55);
heldBlockMesh.visible = false; heldFlatMesh.visible = false;
offhand.add(heldBlockMesh); offhand.add(heldFlatMesh);
const _vmTex = {};
function vmTexFor(type){
  if (_vmTex[type]) return _vmTex[type];
  const url = ICON_URL[type]; if (!url) return null;
  const img = new Image(); const tex = new THREE.Texture(img);
  tex.magFilter = THREE.NearestFilter; tex.minFilter = THREE.NearestFilter;
  img.onload = () => { tex.needsUpdate = true; }; img.src = url;
  _vmTex[type] = tex; return tex;
}
let _heldType = undefined;
function updateHeldViewmodel(force){
  const s = (typeof selectedStack === 'function') ? selectedStack() : null;
  const type = s ? s.type : null;
  if (!force && type === _heldType) return;
  _heldType = type;
  if (type == null || type === AIR){ heldBlockMesh.visible = false; heldFlatMesh.visible = false; return; }
  if (isBlockType(type)){
    const m = BLOCK_MAT[type];
    const map = Array.isArray(m) ? (m[0] && m[0].map) : (m && m.map);
    heldBlockMat.map = map || null;
    heldBlockMat.color.set(map ? 0xffffff : (BLOCK_COLOR[type] || '#cccccc'));
    heldBlockMat.needsUpdate = true;
    heldBlockMesh.visible = true; heldFlatMesh.visible = false;
  } else {
    const tex = vmTexFor(type);
    heldFlatMat.map = tex || null; heldFlatMat.needsUpdate = true;
    heldFlatMesh.visible = !!tex; heldBlockMesh.visible = false;
  }
}
let bobT = 0;                               // walking-bob phase
let swingT = 0;                             // first-person swing timer (counts down)
const SWING_DUR = 0.30;
function swingHand(){ if (swingT <= 0) swingT = SWING_DUR; }   // start a swing if not mid-swing

// ---------- SECTION 6: controls, settings & pause menu ----------
const keys = {};
let yaw = 0, pitch = -0.10;
let firstPerson = true;          // start in first person; the view key toggles
const FLY_SPEED = 12;            // creative flight speed (blocks / second)
let currentSeed = 1337;
let username = 'Player';
let account = null;          // active account id (lowercased username); set on login
let activeWorldId = null;          // which saved world we're playing (null = a joined/guest world: not saved)
let currentWorldName = 'New World';
let kokoSMP = false;               // true while in the always-on default server (everyone OP, no kicking)
let isExtreme = false;             // true while on the Extreme Challenge parkour server
let isKokoCity = false;            // true while on the Koko City server
let isPvpHub = false;              // true while on the Koko PvP Hub server
let parkourWon = false;            // set once the player reaches the finish (so we announce only once)
// per-server rules
const ruleNoBuild   = () => isExtreme || isKokoCity || isPvpHub;   // can't break or place blocks
const ruleNoPVP     = () => isExtreme;                             // can't attack other players (PvP Hub allows it)
const ruleNoCmds    = () => isExtreme || isPvpHub;                 // commands disabled
const ruleForceDay  = () => isExtreme;                             // permanent daytime
const ruleNoHunger  = () => isExtreme || isKokoCity;               // hunger never drains (PvP Hub keeps hunger so beef matters)
const ruleNoOP      = () => isExtreme || isPvpHub;                 // no operator powers
const ruleNoFall    = () => isExtreme || isPvpHub;                 // no fall damage
const ruleNoCreative= () => isPvpHub;                             // survival only — creative locked out

// rebindable controls
const DEFAULT_BINDS = { forward:'KeyW', back:'KeyS', left:'KeyA', right:'KeyD', up:'Space', down:'ShiftLeft', view:'F5', chat:'KeyT' };
let keybinds = { ...DEFAULT_BINDS };
const BIND_LABELS = { forward:'Move forward', back:'Move back', left:'Move left', right:'Move right', up:'Fly up', down:'Fly down', view:'Toggle view', chat:'Open chat' };

// tweakable settings
const settings = { renderDistance: 8, fov: 70, fps: false, fullbright: false, smoothLighting: true, lowDetail: false, op: false, xray: false, reflections: false, openWorld: false, waterFlow: true, texHi: true, brightness: 50, clouds: true, simpleGame: false, shadows: true };
let gamemode = 'creative';      // 'creative' or 'survival'

const crosshair    = document.getElementById('crosshair');
const underwaterEl = document.getElementById('underwater');
const fpsEl        = document.getElementById('fps');
const canvas = renderer.domElement;
const MOUSE_SENSITIVITY = 0.0022;
const MAX_PITCH = 1.4;

let paused = true;          // in-game pause (Esc) while playing
let playing = false;        // true once a world is launched (false on title screens)
let orbiting = true;        // slow cinematic camera orbit over the title backdrop
let listeningBind = null;   // which action is waiting to be rebound

// --- screen management ---
const el = id => document.getElementById(id);
const SCREENS  = ['usernameMenu','homeMenu','editorMenu','selectorMenu','generatorMenu','howtoMenu','lanHostMenu','joinMenu','serversMenu'];
const OVERLAYS = ['pauseMenu','settingsMenu'];
const startCatch = el('startCatch');
const debugEl = document.getElementById('debug');
let showDebug = false;
let lostFocus = false;             // true while F2 has freed the cursor (no pause, no UI)
function refreshDebug(){ debugEl.style.display = (playing && showDebug) ? '' : 'none'; }
const pauseNote = el('pauseNote');

function hideAllMenus(){
  SCREENS.forEach(id => el(id).classList.remove('show'));
  OVERLAYS.forEach(id => el(id).classList.remove('show'));
  startCatch.classList.remove('show');
}
function showScreen(name){ hideAllMenus(); if (name) el(name).classList.add('show'); }
function showPause(){ hideAllMenus(); el('pauseMenu').classList.add('show'); }

// --- mouse / pointer-lock ---
startCatch.addEventListener('click', () => canvas.requestPointerLock());
canvas.addEventListener('contextmenu', (e) => e.preventDefault());
canvas.addEventListener('mousedown', (e) => {
  // after F2 freed the cursor, a click re-grabs the game (no pause)
  if (playing && !paused && !dead && !uiBlocking() && document.pointerLockElement !== canvas){ canvas.requestPointerLock(); return; }
  if (!playing || paused || uiBlocking() || document.pointerLockElement !== canvas) return;
  if (e.button === 0){                                      // left = attack player / mob / break block
    const bt = raycastBlock();
    const bd = bt ? Math.hypot(camera.position.x-(bt.wx+0.5), camera.position.y-(bt.wy+0.5), camera.position.z-(bt.wz+0.5)) : Infinity;
    const ph = net.active ? raycastPlayer() : null;
    const mh = raycastMob();
    const pd = (ph && ph.dist <= 4.2) ? ph.dist : Infinity;     // players have a melee reach
    const md = mh ? mh.dist : Infinity;
    if (pd < bd && pd <= md){ attackPlayer(ph.id); }
    else if (md < bd && md < pd){ damageMob(mh.mob); }
    else if (gamemode === 'creative'){ doBreak(bt); swingHand(); }   // instant
    else { mineHeld = true; swingHand(); }                            // survival: hold to mine
  } else if (e.button === 2){                              // right = eat / open table / sign / place
    const sel = selectedStack();
    if (sel && isFood(sel.type)){ eatSelected(); return; }
    const tgt = raycastBlock();
    const bt = tgt ? blockWorld(tgt.wx, tgt.wy, tgt.wz) : AIR;
    if (bt === CRAFTING_TABLE) openTableMenu();
    else if (bt === SIGN) openSignEditor(tgt.wx, tgt.wy, tgt.wz);
    else placeEquipped();
  }
});
canvas.addEventListener('mouseup', (e) => { if (e.button === 0){ mineHeld = false; mineKey = null; mineProgress = 0; hideCrack(); } });

document.addEventListener('pointerlockchange', () => {
  const locked = document.pointerLockElement === canvas;
  crosshair.style.opacity = locked ? '1' : '0';
  if (locked){ paused = false; lostFocus = false; if (chatOpen) closeChat(); hideAllMenus(); }
  else if (playing){
    if (dead) return;                     // death screen is showing; don't pause over it
    if (invOpen || tableOpen || signEditOpen || managerOpen) return;     // menus intentionally release the pointer
    if (lostFocus){ lostFocus = false; return; }   // F2: free the cursor without pausing / any UI
    if (chatOpen) closeChat();
    paused = true; showPause();
  }
});

document.addEventListener('mousemove', (e) => {
  if (invOpen || tableOpen){ const c = el('carry'); if (c && carried){ c.style.left = e.clientX + 'px'; c.style.top = e.clientY + 'px'; } return; }
  if (document.pointerLockElement !== canvas || chatOpen) return;
  yaw   -= e.movementX * MOUSE_SENSITIVITY;
  pitch -= e.movementY * MOUSE_SENSITIVITY;
  pitch = Math.max(-MAX_PITCH, Math.min(MAX_PITCH, pitch));
});

addEventListener('keydown', (e) => {
  if (signEditOpen) return;                  // the sign text input handles its own keys
  if (dead) return;                          // death screen is up; ignore gameplay keys
  if (listeningBind){                       // capture a key for rebinding
    e.preventDefault();
    if (e.code !== 'Escape') keybinds[listeningBind] = e.code;
    listeningBind = null; buildKeybindUI();
    return;
  }
  if (chatOpen){                            // typing in chat
    if (e.code === 'Enter') submitChat();
    else if (e.code === 'Backspace'){ chatInput = chatInput.slice(0, -1); chatText.textContent = chatInput; updateChatHint(); }
    else if (e.key && e.key.length === 1){ chatInput += e.key; chatText.textContent = chatInput; updateChatHint(); }
    e.preventDefault();
    return;
  }
  if (invOpen){                             // inventory open: only E / Escape do anything here
    if (e.code === 'KeyE' || e.code === 'Escape'){ closeInventory(); e.preventDefault(); }
    return;
  }
  if (tableOpen){                           // crafting table open
    if (e.code === 'KeyE' || e.code === 'Escape'){ closeTableMenu(); e.preventDefault(); }
    return;
  }
  if (managerOpen){                         // server manager open
    if (e.code === 'F8' || e.code === 'Escape'){ closeManager(); e.preventDefault(); }
    return;
  }
  if (e.code === 'F8'){                      // host: open the server manager
    e.preventDefault();
    if (playing && net.active && (net.isHost || kokoSMP)) openManager();
    return;
  }
  // open chat (T) while playing
  if (playing && !paused && e.code === keybinds.chat && document.pointerLockElement === canvas){
    openChat(); updateChatHint(); e.preventDefault(); return;
  }
  // open inventory (E) while playing
  if (playing && !paused && e.code === 'KeyE' && document.pointerLockElement === canvas){
    openInventory(); e.preventDefault(); return;
  }
  // select a hotbar slot with the number keys
  if (playing && /^Digit[1-9]$/.test(e.code)){ selIndex = +e.code.slice(5) - 1; refreshHotbar(); showHeldName(); }
  // Q: drop one of the held item (others can pick it up online)
  if (playing && !paused && !dead && !uiBlocking() && e.code === 'KeyQ' && document.pointerLockElement === canvas){ dropSelected(); e.preventDefault(); return; }
  // TAB: show the player list (instead of pausing), held down
  if (e.code === 'Tab'){
    e.preventDefault();
    if (playing && !paused){ refreshPlayerList(); el('playerList').classList.add('show'); }
    return;
  }
  // F2: free the mouse cursor without opening any menu
  if (e.code === 'F2'){
    e.preventDefault();
    if (playing && !paused && document.pointerLockElement === canvas){ lostFocus = true; document.exitPointerLock(); }
    return;
  }
  keys[e.code] = true;
  if (e.code === 'F3'){ e.preventDefault(); if (playing){ showDebug = !showDebug; refreshDebug(); } }
  if (e.code === keybinds.view){
    if (e.code === 'F5') e.preventDefault();   // F5 would reload the page
    if (playing && !paused) firstPerson = !firstPerson;
  }
});
addEventListener('keyup', (e) => { keys[e.code] = false; if (e.code === 'Tab') el('playerList').classList.remove('show'); });

// --- home screen ---
el('hmPlay').addEventListener('click', () => { buildWorldList(); showScreen('selectorMenu'); });
el('hmSettings').addEventListener('click', () => openSettings('home'));
el('hmHowto').addEventListener('click', () => showScreen('howtoMenu'));
el('howtoBack').addEventListener('click', () => showScreen('homeMenu'));
el('hmEditor').addEventListener('click', openEditor);
el('editorCancel').addEventListener('click', cancelEditor);
el('editorReset').addEventListener('click', resetEditor);
el('editorSave').addEventListener('click', saveEditor);
el('editorColor').addEventListener('input', e => { editState.color = e.target.value; buildEditorPalette(); });
addEventListener('pointerup', () => { editPainting = false; });

// --- world selector ---
el('selCancel').addEventListener('click', () => showScreen('homeMenu'));
el('selGenerate').addEventListener('click', () => openGenerator());
function openGenerator(){
  el('genSeed').value = Math.floor(Math.random() * 1e6);   // fresh random seed each time
  el('genName').value = 'New World';
  showScreen('generatorMenu');
}
function worldIndex(){ try { return JSON.parse(localStorage.getItem(acctKey('worldIndex'))) || []; } catch (e){ return []; } }
function writeWorldIndex(arr){ try { localStorage.setItem(acctKey('worldIndex'), JSON.stringify(arr)); } catch (e){} }
function loadWorldMeta(id){ try { return JSON.parse(localStorage.getItem(acctKey('world_' + id))); } catch (e){ return null; } }
function deleteWorld(id){
  try { localStorage.removeItem(acctKey('world_' + id)); } catch (e){}
  writeWorldIndex(worldIndex().filter(w => w.id !== id));
  buildWorldList();
}
function buildWorldList(){
  const list = el('worldList'); list.innerHTML = '';
  const idx = worldIndex().sort((a, b) => (b.updated || 0) - (a.updated || 0));
  if (!idx.length){
    const empty = document.createElement('div'); empty.className = 'world-empty';
    empty.textContent = 'No worlds yet \u2014 press Generate to make one.';
    list.appendChild(empty); return;
  }
  idx.forEach(w => {
    const card = document.createElement('div'); card.className = 'world-card';
    const left = document.createElement('div'); left.className = 'wc-left';
    const size = w.finite ? (w.half * 2) + ' blocks' : 'infinite';
    left.innerHTML = '<div class="wc-name"></div><div class="wc-meta">seed ' + w.seed + ' \u00B7 ' + size + '</div>';
    left.querySelector('.wc-name').textContent = w.name || 'New World';
    left.addEventListener('click', () => continueSavedWorld(w.id));
    const acts = document.createElement('div'); acts.className = 'wc-actions';
    const play = document.createElement('button'); play.className = 'wc-btn'; play.textContent = 'Play';
    play.addEventListener('click', () => continueSavedWorld(w.id));
    const del = document.createElement('button'); del.className = 'wc-btn danger'; del.textContent = 'Delete';
    del.addEventListener('click', () => { if (confirm('Delete "' + (w.name || 'this world') + '"? This cannot be undone.')) deleteWorld(w.id); });
    acts.appendChild(play); acts.appendChild(del);
    card.appendChild(left); card.appendChild(acts);
    list.appendChild(card);
  });
}

// --- world generator ---
let genMode = 'infinite';
function setGenMode(m){
  genMode = m;
  el('optInfinite').classList.toggle('active', m === 'infinite');
  el('optCustom').classList.toggle('active', m === 'custom');
  el('customSizeRow').style.display = m === 'custom' ? '' : 'none';
}
el('optInfinite').addEventListener('click', () => setGenMode('infinite'));
el('optCustom').addEventListener('click', () => setGenMode('custom'));
el('genSize').addEventListener('input', () => { el('genSizeVal').textContent = el('genSize').value + ' blocks'; });
el('genCancel').addEventListener('click', () => showScreen('selectorMenu'));
el('genPlay').addEventListener('click', launchFromGenerator);

let genGamemode = 'creative';
function setGenGamemode(m){
  genGamemode = m;
  el('optCreative').classList.toggle('active', m === 'creative');
  el('optSurvival').classList.toggle('active', m === 'survival');
}
el('optCreative').addEventListener('click', () => setGenGamemode('creative'));
el('optSurvival').addEventListener('click', () => setGenGamemode('survival'));
let genFlat = false;
el('genFlat').addEventListener('click', () => { genFlat = !genFlat; const b = el('genFlat'); b.textContent = genFlat ? 'On' : 'Off'; b.classList.toggle('on', genFlat); });

function launchFromGenerator(){
  let seed = parseInt(el('genSeed').value, 10);
  if (!Number.isFinite(seed)) seed = Math.floor(Math.random() * 1e6);
  currentSeed = seed;
  const nameRaw = (el('genName').value || '').trim();
  currentWorldName = nameRaw.slice(0, 28) || 'New World';
  const nameLow = nameRaw.toLowerCase();
  if (nameLow === '.67')        worldMode = '67';            // secret: giant "67" monument world
  else if (nameLow === '.koko') worldMode = 'koko';         // secret: flat plains, only Koko mobs
  else if (genFlat)             worldMode = 'flat';         // flat world option
  else                          worldMode = 'normal';
  if (worldMode !== 'normal'){ world.finite = false; }       // secret/flat worlds are infinite plains
  activeWorldId = 'w' + Date.now().toString(36) + Math.floor(Math.random() * 1e4).toString(36);
  gamemode = genGamemode;
  if (worldMode === 'normal' && genMode === 'custom'){ world.finite = true; world.half = Math.max(32, (+el('genSize').value) / 2); }
  else if (worldMode === 'normal') world.finite = false;
  orbiting = false;
  applyRenderDistance();
  startNewWorld(seed);              // reseed, clear, place player, queue spawn area
  saveGame();                       // register the world immediately so it shows in the list
  beginLoad(enterPlay);            // loading screen -> click to play
}
function continueSavedWorld(id){
  const meta = loadWorldMeta(id); if (!meta) return;
  activeWorldId = id;
  currentWorldName = meta.name || 'New World';
  currentSeed = meta.seed; timeOfDay = meta.time != null ? meta.time : timeOfDay;
  firstPerson = meta.firstPerson != null ? meta.firstPerson : true; yaw = meta.yaw || 0; pitch = meta.pitch != null ? meta.pitch : -0.10;
  Object.assign(settings, meta.settings || {});
  keybinds = { ...DEFAULT_BINDS, ...(meta.keybinds || {}) };
  world.finite = meta.world ? !!meta.world.finite : false;
  world.half = (meta.world && meta.world.half) || 256;
  gamemode = meta.gamemode === 'survival' ? 'survival' : 'creative';
  worldMode = meta.worldMode || 'normal';                       // flat / secret world type
  edits.clear(); if (Array.isArray(meta.edits)) for (const [k, v] of meta.edits) edits.set(k, v);
  clearSigns(); if (Array.isArray(meta.signs)) for (const [k, txt] of meta.signs) setSignText(k, txt);
  health = (typeof meta.health === 'number') ? meta.health : maxHealth;
  food = (typeof meta.food === 'number') ? meta.food : maxFood;
  xpLevel = (typeof meta.xpLevel === 'number') ? meta.xpLevel : 0;
  xpInto = (typeof meta.xpInto === 'number') ? meta.xpInto : 0;
  dead = false; fallPeakY = null; statT = 0; refreshStatBars(); refreshXpBar();
  clearInventory(); clearMobs();
  if (meta.invHotbar) loadInv(hotbar, meta.invHotbar);
  if (meta.invMain) loadInv(invMain, meta.invMain);
  selIndex = meta.selIndex || 0; refreshHotbar();
  orbiting = false;
  worldSeed = currentSeed; seedWorld(currentSeed);
  chunks.forEach(ch => disposeChunkMeshes(ch)); chunks.clear();
  applyRenderDistance(); syncSettingsUI();
  benny.position.set(meta.px != null ? meta.px : 0.5, meta.py != null ? meta.py : 40, meta.pz != null ? meta.pz : 0.5);
  chunksDirty = true; refreshChunkQueue();
  beginLoad(enterPlay);
}

// after a world loads, one click grabs the mouse and play begins
function enterPlay(){
  playing = true; paused = true; orbiting = false; dead = false;
  benny.visible = true; vy = 0; mineHeld = false; fallPeakY = null;
  chunksDirty = true;                 // top up to full render distance now that we're playing
  hideAllMenus(); el('deathScreen').classList.remove('show'); refreshDebug(); refreshHotbar(); refreshStatBars();
  el('hotbar').classList.add('show');
  startCatch.classList.add('show');
}
// Save & Quit -> back to the title screen with the live backdrop
function quitToTitle(skipSave){
  if (!skipSave) saveGame();
  netShutdown();                      // host: notifies joiners (they get kicked); joiner: disconnects
  playing = false; paused = true; orbiting = true; invOpen = false; chatOpen = false; tableOpen = false; signEditOpen = false; dead = false; managerOpen = false;
  if (document.pointerLockElement === canvas) document.exitPointerLock();
  offhand.visible = false; benny.visible = false;
  clearMobs(); clearSigns();
  el('hotbar').classList.remove('show'); el('inventory').classList.remove('show'); el('chatBar').classList.remove('show');
  el('tableMenu').classList.remove('show'); el('signEditor').classList.remove('show'); el('deathScreen').classList.remove('show'); el('statBars').classList.remove('show'); el('managerMenu').classList.remove('show');
  refreshDebug();
  startBackdrop();
  showScreen('homeMenu');
}

// --- in-game pause buttons ---
el('btnPlay').addEventListener('click', () => canvas.requestPointerLock());
el('btnSettings').addEventListener('click', () => openSettings('pause'));
el('btnSaveQuit').addEventListener('click', () => quitToTitle());
el('btnLogout').addEventListener('click', () => {
  if (playing) quitToTitle();                       // save current world + leave any session
  account = null; username = 'Player';              // end the account session
  el('acctPass').value = ''; el('acctStatus').textContent = '';
  try { const last = localStorage.getItem('kokoLastUser'); if (last) el('acctUser').value = last; } catch (e){}
  showScreen('usernameMenu');                       // back to the login screen
});
el('btnRespawn').addEventListener('click', respawn);
el('btnDeathQuit').addEventListener('click', () => { el('deathScreen').classList.remove('show'); quitToTitle(); });

// --- settings (Save / Cancel, snapshot-based) ---
let settingsReturn = 'home', settingsSnapshot = null;
function openSettings(from){
  settingsReturn = from;
  settingsSnapshot = { settings: { ...settings }, keybinds: { ...keybinds }, time: timeOfDay };
  syncSettingsUI();
  setSettingsTab('video');
  updateSettingLocks();
  hideAllMenus(); el('settingsMenu').classList.add('show');
}
function closeSettings(save){
  if (save){ persistSettings(); }
  else {
    Object.assign(settings, settingsSnapshot.settings);
    keybinds = { ...settingsSnapshot.keybinds };
    timeOfDay = settingsSnapshot.time;
    applyRenderDistance(); applyFov(); applyFps(); applyBright(); applySmooth(); applyPerf(); applyOp(); applyXray(); applyReflections(); applyDetail(); applyOpenWorld(); applyWaterFlow(); applyTexQuality(); applyBrightness(); applyClouds(); applyShadows(); applySimpleGame();
  }
  if (settingsReturn === 'pause') showPause(); else showScreen('homeMenu');
}
el('setSave').addEventListener('click', () => closeSettings(true));
el('setCancel').addEventListener('click', () => closeSettings(false));

// --- settings widgets ---
const setRender = el('setRender');
const setFov    = el('setFov');
const setTime   = el('setTime');
const setFpsBtn = el('setFps');
const setBright = el('setBright');
const setSmooth = el('setSmooth');
const setPerf   = el('setPerf');

// in LAN, only the host controls the clock, OP, and Open World (KOKOSMP also pins OP on for all)
function settingLocked(which){
  if (which === 'op' && kokoSMP) return true;
  return net.active && !net.isHost;
}
function updateSettingLocks(){
  [['setTime','time'],['setOp','op'],['setOpenWorld','openWorld']].forEach(([id, which]) => {
    const e = el(id); if (!e) return;
    const locked = settingLocked(which);
    e.classList.toggle('locked', locked);
    if (e.tagName === 'INPUT') e.disabled = locked;
  });
}

setRender.addEventListener('input', () => { settings.renderDistance = +setRender.value; applyRenderDistance(); chunksDirty = true; });
setFov.addEventListener('input',    () => { settings.fov = +setFov.value; applyFov(); });
el('setBrightness').addEventListener('input', () => { settings.brightness = +el('setBrightness').value; applyBrightness(); });
el('setClouds').addEventListener('click', () => { settings.clouds = !settings.clouds; applyClouds(); });
setTime.addEventListener('input',   () => { if (settingLocked('time')){ setTime.value = Math.floor(timeOfDay * 1440); return; } timeOfDay = (+setTime.value) / 1440; el('timeVal').textContent = clockLabel(); });
setFpsBtn.addEventListener('click', () => { settings.fps = !settings.fps; applyFps(); });
setBright.addEventListener('click', () => { settings.fullbright = !settings.fullbright; applyBright(); });
setSmooth.addEventListener('click', () => { settings.smoothLighting = !settings.smoothLighting; applySmooth(); });
setPerf.addEventListener('click',   () => { settings.lowDetail = !settings.lowDetail; applyPerf(); applyDetail(); });
el('setOp').addEventListener('click', () => { if (settingLocked('op')) return; settings.op = !settings.op; applyOp(); });
el('setOpenWorld').addEventListener('click', () => { if (settingLocked('openWorld')) return; settings.openWorld = !settings.openWorld; applyOpenWorld(); if (net.active && net.isHost){ if (settings.openWorld) lobbyAdvertise(); else lobbyUnadvertise(); } });
el('setWaterFlow').addEventListener('click', () => { settings.waterFlow = !settings.waterFlow; applyWaterFlow(); });
el('setTexHi').addEventListener('click', () => { settings.texHi = !settings.texHi; applyTexQuality(); });
el('setXray').addEventListener('click', () => { settings.xray = !settings.xray; applyXray(); });
el('setReflect').addEventListener('click', () => { settings.reflections = !settings.reflections; applyReflections(); });
el('setSimple').addEventListener('click', () => { settings.simpleGame = !settings.simpleGame; applySimpleGame(); });
el('setShadows').addEventListener('click', () => { settings.shadows = !settings.shadows; applyShadows(); });

// settings tabs (Video / Graphics / Keybinds)
function setSettingsTab(tab){
  [['video','setTabVideo','setPanelVideo'],['graphics','setTabGraphics','setPanelGraphics'],['keys','setTabKeys','setPanelKeys'],['hosting','setTabHosting','setPanelHosting']].forEach(([t, btn, panel]) => {
    el(btn).classList.toggle('active', t === tab);
    el(panel).style.display = (t === tab) ? '' : 'none';
  });
}
el('setTabVideo').addEventListener('click', () => setSettingsTab('video'));
el('setTabGraphics').addEventListener('click', () => setSettingsTab('graphics'));
el('setTabKeys').addEventListener('click', () => setSettingsTab('keys'));
el('setTabHosting').addEventListener('click', () => setSettingsTab('hosting'));

// Simple game: replace detailed block textures with flat 1–2 colour blocks
function stripToSimple(){
  for (const t of BLOCK_TYPES){ const m = BLOCK_MAT[t]; if (!m || Array.isArray(m)) continue; m.map = null; m.color.set(BLOCK_COLOR[t] || '#999999'); m.needsUpdate = true; }
}
function applySimpleGame(){
  const b = el('setSimple'); if (b){ b.textContent = settings.simpleGame ? 'On' : 'Off'; b.classList.toggle('on', settings.simpleGame); }
  if (settings.simpleGame){ stripToSimple(); }
  else { rebuildTextures(); for (const t of BLOCK_TYPES){ const m = BLOCK_MAT[t]; if (m && !Array.isArray(m) && m.map) m.color.set(0xffffff); } applyXray(); applyReflections(); }
}
// Shadows on/off
function applyShadows(){
  const b = el('setShadows'); if (b){ b.textContent = settings.shadows ? 'On' : 'Off'; b.classList.toggle('on', settings.shadows); }
  const on = settings.shadows !== false;
  renderer.shadowMap.enabled = on;
  sunLight.castShadow = on;
  scene.traverse(o => { if (o.isMesh && o.material){ (Array.isArray(o.material) ? o.material : [o.material]).forEach(mm => { if (mm) mm.needsUpdate = true; }); } });
}

// reflective water (opt-in). off = plain blue (no reflection, not white); on = reflects the live scene
function applyReflections(){
  const w = BLOCK_MAT[WATER]; const b = el('setReflect');
  if (b){ b.textContent = settings.reflections ? 'On' : 'Off'; b.classList.toggle('on', settings.reflections); }
  if (settings.reflections){
    w.envMap = reflectRT.texture; w.envMapIntensity = 0.55; w.metalness = 0.35; w.roughness = 0.34; w.color.setHex(0x274f6e);
  } else {
    w.envMap = null; w.metalness = 0.0; w.roughness = 1.0; w.color.setHex(0x2f6f9e);
  }
  w.needsUpdate = true;
}

// ore vision: hide ordinary blocks (keep them raycastable so they still break); ores + grass stay solid
const XRAY_HIDE = [DIRT, SAND, STONE, SNOW_B, WOOD, LEAF];
function applyXray(){
  const b = el('setXray'); if (b){ b.textContent = settings.xray ? 'On' : 'Off'; b.classList.toggle('on', settings.xray); }
  for (const t of XRAY_HIDE){
    const m = BLOCK_MAT[t]; if (!m) continue;
    if (settings.xray){ m.transparent = true; m.opacity = 0; m.depthWrite = false; }
    else { m.transparent = false; m.opacity = 1; m.depthWrite = true; }
    m.needsUpdate = true;
  }
}

function applyRenderDistance(){
  renderChunks = Math.max(2, Math.min(18, Math.round(settings.renderDistance) || 8));  // guard against stale values
  settings.renderDistance = renderChunks;
  const blocks = renderChunks * CHUNK;                                    // how far you can see, in blocks
  camera.far = Math.max(blocks + CHUNK, SKY_DIST + 100);                  // keep sun/moon visible
  camera.updateProjectionMatrix();
  scene.fog.density = 1.1 / blocks;                                       // fog fades terrain near the edge
  el('rdVal').textContent = renderChunks + ' chunks';
  setRender.value = renderChunks;
}
function applyFov(){
  camera.fov = settings.fov; camera.updateProjectionMatrix();
  el('fovVal').textContent = settings.fov + '\u00B0';
  setFov.value = settings.fov;
}
function applyFps(){ setFpsBtn.textContent = settings.fps ? 'On' : 'Off'; setFpsBtn.classList.toggle('on', settings.fps); fpsEl.classList.toggle('show', settings.fps); }
function applyBright(){ setBright.textContent = settings.fullbright ? 'On' : 'Off'; setBright.classList.toggle('on', settings.fullbright); }
function applySmooth(){ setSmooth.textContent = settings.smoothLighting ? 'On' : 'Off'; setSmooth.classList.toggle('on', settings.smoothLighting); }
function applyPerf(){ setPerf.textContent = settings.lowDetail ? 'On' : 'Off'; setPerf.classList.toggle('on', settings.lowDetail); }
function applyOp(){ const b = el('setOp'); b.textContent = settings.op ? 'On' : 'Off'; b.classList.toggle('on', settings.op); }
function applyOpenWorld(){ const b = el('setOpenWorld'); if (b){ b.textContent = settings.openWorld ? 'On' : 'Off'; b.classList.toggle('on', settings.openWorld); } }
function applyWaterFlow(){ const b = el('setWaterFlow'); if (b){ b.textContent = settings.waterFlow ? 'On' : 'Off'; b.classList.toggle('on', settings.waterFlow); } }
let brightnessMul = 1.0;
function applyBrightness(){
  const v = (settings.brightness == null) ? 50 : settings.brightness;
  brightnessMul = 0.5 + (v / 100) * 1.0;                 // 0%→0.5×, 50%→1.0×, 100%→1.5×
  const e = el('setBrightness'); if (e) e.value = v;
  const lab = el('brightVal'); if (lab) lab.textContent = v + '%';
}
function applyClouds(){
  const b = el('setClouds'); if (b){ b.textContent = settings.clouds ? 'On' : 'Off'; b.classList.toggle('on', settings.clouds); }
  if (typeof cloudGroup !== 'undefined' && cloudGroup) cloudGroup.visible = !!settings.clouds;
}
function applyTexQuality(){
  const b = el('setTexHi'); if (b){ b.textContent = settings.texHi ? 'High' : 'Low'; b.classList.toggle('on', settings.texHi); }
  const want = settings.texHi ? 32 : 16;
  if (want !== TEX_RES){ TEX_RES = want; rebuildTextures(); }
}
function clockLabel(){ const m = Math.floor(timeOfDay * 1440); return String((m / 60) | 0).padStart(2,'0') + ':' + String(m % 60).padStart(2,'0'); }

function prettyKey(code){
  if (code === 'Space') return 'Space';
  if (code.startsWith('Shift')) return 'Shift';
  if (code.startsWith('Control')) return 'Ctrl';
  return code.replace('Key','').replace('Digit','').replace('Arrow','');
}
function buildKeybindUI(){
  const box = el('keybinds'); box.innerHTML = '';
  Object.keys(BIND_LABELS).forEach(act => {
    const name = document.createElement('div'); name.className = 'kb-name'; name.textContent = BIND_LABELS[act];
    const btn = document.createElement('button'); btn.className = 'kb-btn';
    btn.textContent = listeningBind === act ? '...' : prettyKey(keybinds[act]);
    if (listeningBind === act) btn.classList.add('listening');
    btn.addEventListener('click', () => { listeningBind = act; buildKeybindUI(); });
    box.appendChild(name); box.appendChild(btn);
  });
}
function syncSettingsUI(){
  applyRenderDistance(); applyFov(); applyFps(); applyBright(); applySmooth(); applyPerf(); applyOp(); applyXray(); applyReflections();
  applyOpenWorld(); applyWaterFlow(); applyTexQuality(); applyBrightness(); applyClouds(); applyShadows(); applySimpleGame();
  setTime.value = Math.floor(timeOfDay * 1440);
  el('timeVal').textContent = clockLabel();
  buildKeybindUI();
}

// --- persist just the settings/keybinds (separate from a world save) ---
function persistSettings(){
  try { localStorage.setItem(acctKey('settings'), JSON.stringify({ settings, keybinds })); } catch (err){}
}
function loadSettings(){
  try {
    const raw = localStorage.getItem(acctKey('settings')); if (!raw) return;
    const s = JSON.parse(raw);
    Object.assign(settings, s.settings || {});
    keybinds = { ...DEFAULT_BINDS, ...(s.keybinds || {}) };
  } catch (err){}
}

// --- live title backdrop: a real slice of the game with a slow orbiting camera ---
const BACKDROP_SEED = 20250607;
let orbitAngle = 0;
function startBackdrop(){
  world.finite = false; orbiting = true;
  worldMode = "normal"; worldSeed = BACKDROP_SEED; seedWorld(BACKDROP_SEED);
  edits.clear();
  chunks.forEach(ch => disposeChunkMeshes(ch)); chunks.clear();
  benny.position.set(8, Math.max(heightAt(8, 8), WATER_Y) + 6, 8);
  benny.visible = false; offhand.visible = false;
  orbitAngle = 0; chunksDirty = true; refreshChunkQueue();
}
function orbitCamera(dt){
  orbitAngle += dt * 0.05;                       // slow drift
  const tx = 8, tz = 8, ty = WATER_Y + 4;
  const R = 60, H = WATER_Y + 30;
  camera.position.set(tx + Math.cos(orbitAngle) * R, H, tz + Math.sin(orbitAngle) * R);
  camera.lookAt(tx, ty, tz);
}

// --- save / load (best-effort; uses localStorage where available) ---
// An infinite world can't be stored whole, so we save the seed (the world is
// deterministic) plus the player's state and any blocks they've changed.
function saveGame(){
  if (!activeWorldId) return false;        // joined/guest worlds aren't saved locally
  try {
    localStorage.setItem(acctKey('world_' + activeWorldId), JSON.stringify({
      name: currentWorldName, seed: currentSeed, time: timeOfDay, firstPerson, yaw, pitch,
      px: benny.position.x, py: benny.position.y, pz: benny.position.z,
      settings, keybinds, world: { finite: world.finite, half: world.half }, gamemode, worldMode,
      invHotbar: serializeInv(hotbar), invMain: serializeInv(invMain), selIndex,
      edits: Array.from(edits.entries()),
      health, food, xpLevel, xpInto,
      signs: Array.from(signs.entries()).map(([k, r]) => [k, r.text])
    }));
    const idx = worldIndex().filter(w => w.id !== activeWorldId);
    idx.push({ id: activeWorldId, name: currentWorldName, seed: currentSeed, finite: world.finite, half: world.half, updated: Date.now() });
    writeWorldIndex(idx);
    return true;
  } catch (err){ return false; }
}
function loadSaveMeta(){ return activeWorldId ? loadWorldMeta(activeWorldId) : null; }

// ---------- in-game chat + commands ----------
const chatLogEl = el('chatLog'), chatBar = el('chatBar'), chatText = el('chatText');
let chatOpen = false, chatInput = '';
const chatLines = [];
let chatHideTimer = null;

function renderChatLog(){
  chatLogEl.innerHTML = '';
  chatLines.slice(-8).forEach(L => {
    const d = document.createElement('div');
    d.className = 'chat-line' + (L.system ? ' system' : '');
    const m = !L.system && /^(kokochins): ([\s\S]*)$/i.exec(L.text);   // rainbow the kokochins name
    if (m){
      const nm = m[1];
      for (let i = 0; i < nm.length; i++){ const s = document.createElement('span'); s.textContent = nm[i]; s.style.color = 'hsl(' + Math.round(i/nm.length*360) + ',95%,62%)'; d.appendChild(s); }
      d.appendChild(document.createTextNode(': ' + m[2]));
    } else d.textContent = L.text;
    chatLogEl.appendChild(d);
  });
}
function showChatLog(){
  chatLogEl.classList.add('show');
  if (chatHideTimer) clearTimeout(chatHideTimer);
  if (!chatOpen) chatHideTimer = setTimeout(() => chatLogEl.classList.remove('show'), 6000);
}
function addChatLine(text, system){
  chatLines.push({ text, system: !!system });
  if (chatLines.length > 50) chatLines.shift();
  renderChatLog(); showChatLog();
}
function openChat(){
  chatOpen = true; chatInput = '';
  for (const k in keys) keys[k] = false;           // drop any held movement keys
  chatText.textContent = '';
  chatBar.classList.add('show'); chatLogEl.classList.add('show');
  if (chatHideTimer) clearTimeout(chatHideTimer);
  updateChatHint();
}
function closeChat(){
  chatOpen = false; chatInput = ''; chatText.textContent = '';
  chatBar.classList.remove('show'); showChatLog(); updateChatHint();
}
function submitChat(){
  const msg = chatInput.trim();
  if (msg){
    if (msg[0] === '/') runCommand(msg.slice(1));
    else { addChatLine(username + ': ' + msg); netSendChat(msg); }
  }
  closeChat();
}

const gamemodeLabel = () => gamemode === 'survival' ? 'Survival' : 'Creative';
function parseGm(s){ s = (s || '').toLowerCase(); if (s === 'creative' || s === 'c' || s === '1') return 'creative'; if (s === 'survival' || s === 's' || s === '0') return 'survival'; return null; }
const canUseCommands = () => !ruleNoCmds() && (!net.active || net.isHost || settings.op || gamemode === 'creative');

function runCommand(raw){
  const t = raw.trim().split(/\s+/);
  const cmd = (t[0] || '').toLowerCase();

  // /kick <username> — host only, works regardless of game mode
  if (cmd === 'kick'){
    if (kokoSMP){ addChatLine('Kicking is disabled on KOKOSMP.', true); return; }
    if (!net.active || !net.isHost){ addChatLine('Only the host can kick players.', true); return; }
    const target = t.slice(1).join(' ').trim();
    if (!target){ addChatLine('Usage: /kick <username>', true); return; }
    let found = null;
    netNames.forEach((name, id) => { if ((name || '').toLowerCase() === target.toLowerCase()) found = id; });
    if (!found){ addChatLine('No player named "' + target + '" is connected.', true); return; }
    kickPlayerById(found);
    addChatLine('Kicked ' + target + '.', true);
    return;
  }

  if (!canUseCommands()){ addChatLine('Commands only work in Creative, or with OP enabled in Settings.', true); return; }

  // /go to <x> <z> <y>   or   /go to @<player>   (also accepts /goto)
  if ((cmd === 'go' && (t[1] || '').toLowerCase() === 'to') || cmd === 'goto'){
    const a = cmd === 'goto' ? t.slice(1) : t.slice(2);
    if (a[0] && a[0][0] === '@'){                          // teleport to another player
      const target = a.join(' ').slice(1).trim();
      if (target.toLowerCase() === username.toLowerCase()){ addChatLine('That\u2019s you — you\u2019re already there.', true); return; }
      let found = null;
      netNames.forEach((nm, id) => { if ((nm || '').toLowerCase() === target.toLowerCase()) found = id; });
      if (!found){ addChatLine('No player named "' + target + '" is here.', true); return; }
      const rp = remotePlayers.get(found);
      if (!rp){ addChatLine('Can\u2019t locate ' + netNames.get(found) + ' yet.', true); return; }
      benny.position.set(rp.x, rp.y + 0.1, rp.z); vy = 0; chunksDirty = true;
      addChatLine('Teleported to ' + netNames.get(found), true);
      return;
    }
    const x = parseFloat(a[0]), z = parseFloat(a[1]), y = parseFloat(a[2]);
    if (![x, y, z].every(Number.isFinite)){ addChatLine('Usage: /go to <x> <z> <y>  or  /go to @player', true); return; }
    benny.position.set(x, y, z);
    vy = 0;
    chunksDirty = true;
    addChatLine('Teleported to x ' + x + ', z ' + z + ', y ' + y, true);
    return;
  }
  // /time set <ticks | keyword>   (host-only in LAN)
  if (cmd === 'time' && (t[1] || '').toLowerCase() === 'set'){
    if (net.active && !net.isHost){ addChatLine('Only the host can change the time.', true); return; }
    setTimeFromArg((t[2] || '').toLowerCase()); return;
  }
  // /gamemode <creative|survival|c|s>  (OP only)
  if (cmd === 'gamemode'){
    if (!settings.op && net.active && !net.isHost){ addChatLine('Only operators can change game mode here.', true); return; }
    // /gamemode @<player> <mode>  — yourself, or (as host) another player
    if ((t[1] || '')[0] === '@'){
      const gm = parseGm(t[2]); if (!gm){ addChatLine('Usage: /gamemode @player <creative|survival>  (or c / s)', true); return; }
      const target = t[1].slice(1).trim();
      if (target.toLowerCase() === username.toLowerCase()){           // change your own mode
        gamemode = gm; refreshHotbar(); addChatLine('Game mode set to ' + gamemodeLabel(), true); return;
      }
      if (!net.active){ addChatLine('No player named "' + target + '" is here.', true); return; }
      if (!net.isHost){ addChatLine('Only the host can set another player\u2019s game mode.', true); return; }
      let found = null; netNames.forEach((nm, id) => { if ((nm || '').toLowerCase() === target.toLowerCase()) found = id; });
      if (!found){ addChatLine('No player named "' + target + '" is here.', true); return; }
      const conn = net.conns.get(found);
      if (conn) netSend(conn, { t: 'setgm', gm });
      addChatLine('Set ' + netNames.get(found) + '\u2019s game mode to ' + (gm === 'creative' ? 'Creative' : 'Survival'), true);
      return;
    }
    const g = parseGm(t[1]); if (!g){ addChatLine('Usage: /gamemode <creative|survival>  (or @player, c, s)', true); return; }
    gamemode = g; refreshHotbar();
    addChatLine('Game mode set to ' + gamemodeLabel(), true);
    return;
  }
  // /summon <mob>  — spawn a mob at your feet
  if (cmd === 'summon'){
    const name = (t[1] || '').toLowerCase();
    if (!MOB_DEF[name]){ addChatLine('Usage: /summon <pig|sheep|cow|chicken|fish|villager|koko>', true); return; }
    const fx = benny.position.x + Math.sin(yaw) * 1.5, fz = benny.position.z + Math.cos(yaw) * 1.5;
    const m = spawnMob(name, fx, benny.position.y, fz);
    addChatLine(m ? ('Summoned a ' + name) : 'Too many mobs already.', true);
    return;
  }
  // /give <item name> [count]
  if (cmd === 'give'){
    const rest = t.slice(1);
    let count = 1; if (rest.length && Number.isFinite(parseInt(rest[rest.length-1], 10)) && /^\d+$/.test(rest[rest.length-1])) count = Math.max(1, Math.min(999, parseInt(rest.pop(), 10)));
    const wanted = rest.join(' ').toLowerCase().replace(/\s+/g, ' ').trim();
    if (!wanted){ addChatLine('Usage: /give <item> [count]   e.g. /give diamond ore 5', true); return; }
    const id = lookupItem(wanted);
    if (id == null){ addChatLine('No item named "' + wanted + '"', true); return; }
    addItem(id, count);
    addChatLine('Gave ' + count + ' x ' + itemName(id), true);
    return;
  }
  addChatLine('Unknown command: /' + cmd, true);
}
// resolve a typed name (case/space/underscore-insensitive) to a block or item id
function lookupItem(name){
  const norm = s => s.toLowerCase().replace(/[\s_]+/g, '');
  const target = norm(name);
  for (const [id, nm] of Object.entries(BLOCK_NAME)) if (norm(nm) === target) return +id;
  for (const [id, nm] of Object.entries(ITEM_NAME))  if (norm(nm) === target) return +id;
  // also accept "<mob> egg" / "<mob> spawn egg"
  for (const [eid, mob] of Object.entries(EGG_TO_MOB)) if (target === norm(mob+'egg') || target === norm(mob+'spawnegg')) return +eid;
  return null;
}

function setTimeFromArg(a){
  const kw = { dawn:0, sunrise:0, day:1000, morning:1000, noon:6000, midday:6000, sunset:12000, dusk:12000, night:13000, midnight:18000 };
  let ticks;
  if (a in kw) ticks = kw[a];
  else {
    const n = parseFloat(a);
    if (!Number.isFinite(n)){ addChatLine('Usage: /time set <0-24000 | day | noon | night | midnight>', true); return; }
    ticks = ((n % 24000) + 24000) % 24000;
  }
  timeOfDay = ((ticks / 24000) + 0.25) % 1;          // tick 0 = sunrise on our clock
  el('timeVal').textContent = clockLabel();
  addChatLine('Set the time to ' + a, true);
}

// ---------- SECTION 6c: items, hotbar, inventory, breaking & placing ----------
const BLOCK_NAME  = { [GRASS]:'Grass', [DIRT]:'Dirt', [SAND]:'Sand', [STONE]:'Stone', [SNOW_B]:'Snow', [WOOD]:'Wood', [LEAF]:'Leaves', [COAL_ORE]:'Coal Ore', [IRON_ORE]:'Iron Ore', [DIAMOND_ORE]:'Diamond Ore', [WOOD_PLANKS]:'Wood Planks', [CRAFTING_TABLE]:'Crafting Table', [SIGN]:'Sign', [CACTUS]:'Cactus', [JUNGLE_WOOD]:'Jungle Wood', [JUNGLE_LEAF]:'Jungle Leaves', [VINE]:'Vine', [SANDSTONE]:'Sandstone', [DRY_GRASS]:'Dry Grass', [ROAD]:'Asphalt', [CONCRETE]:'Concrete', [BRICK]:'Brick', [GLASS_B]:'Glass', [WATER]:'Water' };
const BLOCK_COLOR = { [GRASS]:'#6aa84f', [DIRT]:'#8a6a43', [SAND]:'#ded2a0', [STONE]:'#8f8f8f', [SNOW_B]:'#eef3f6', [WOOD]:'#9a6f43', [LEAF]:'#4f7f3f', [COAL_ORE]:'#3a3a3a', [IRON_ORE]:'#b98c5e', [DIAMOND_ORE]:'#56cdd6', [WOOD_PLANKS]:'#b9905a', [CRAFTING_TABLE]:'#9a6f43', [SIGN]:'#c4a06c', [CACTUS]:'#4f8f4f', [JUNGLE_WOOD]:'#5c4028', [JUNGLE_LEAF]:'#2f6e2f', [VINE]:'#3f7e3f', [SANDSTONE]:'#ddca96', [DRY_GRASS]:'#b0a860', [ROAD]:'#35383d', [CONCRETE]:'#b7bcc2', [BRICK]:'#9e4b3a', [GLASS_B]:'#9fd4e8' };
const HARDNESS    = { [GRASS]:0.55, [DIRT]:0.5, [SAND]:0.5, [STONE]:1.6, [SNOW_B]:0.4, [WOOD]:1.2, [LEAF]:0.25, [COAL_ORE]:2.2, [IRON_ORE]:2.6, [DIAMOND_ORE]:3.2, [WOOD_PLANKS]:1.0, [CRAFTING_TABLE]:1.2, [SIGN]:0.7 };  // seconds (survival)
const STACK_MAX   = 64;

// ---- non-block items: foods (mob drops) + spawn eggs ----
const BACON = 100, MUTTON = 101, BEEF = 102, CHICKEN_I = 103, FISH_I = 104;
const STICK = 105, WOODEN_AXE = 130, WOODEN_SWORD = 131, WOODEN_PICKAXE = 132;
const STONE_AXE = 133, STONE_SWORD = 134, STONE_PICKAXE = 135, IRON_AXE = 136, IRON_SWORD = 137, IRON_PICKAXE = 138;
const EGG_PIG = 110, EGG_SHEEP = 111, EGG_COW = 112, EGG_CHICKEN = 113, EGG_FISH = 114, EGG_VILLAGER = 115, EGG_KOKO = 116;
const ITEM_NAME = {
  [BACON]:'Bacon', [MUTTON]:'Mutton', [BEEF]:'Beef', [CHICKEN_I]:'Chicken', [FISH_I]:'Fish',
  [STICK]:'Stick', [WOODEN_AXE]:'Wooden Axe', [WOODEN_SWORD]:'Wooden Sword', [WOODEN_PICKAXE]:'Wooden Pickaxe',
  [STONE_AXE]:'Stone Axe', [STONE_SWORD]:'Stone Sword', [STONE_PICKAXE]:'Stone Pickaxe',
  [IRON_AXE]:'Iron Axe', [IRON_SWORD]:'Iron Sword', [IRON_PICKAXE]:'Iron Pickaxe',
  [EGG_PIG]:'Pig Spawn Egg', [EGG_SHEEP]:'Sheep Spawn Egg', [EGG_COW]:'Cow Spawn Egg',
  [EGG_CHICKEN]:'Chicken Spawn Egg', [EGG_FISH]:'Fish Spawn Egg', [EGG_VILLAGER]:'Villager Spawn Egg', [EGG_KOKO]:'Koko Spawn Egg'
};
const ITEM_COLOR = {
  [BACON]:'#d98b8b', [MUTTON]:'#e7d6cf', [BEEF]:'#9b3b2f', [CHICKEN_I]:'#e8c98e', [FISH_I]:'#9fc3cf',
  [STICK]:'#9a6f43', [WOODEN_AXE]:'#b9905a', [WOODEN_SWORD]:'#b9905a', [WOODEN_PICKAXE]:'#b9905a',
  [STONE_AXE]:'#8f8f8f', [STONE_SWORD]:'#8f8f8f', [STONE_PICKAXE]:'#8f8f8f',
  [IRON_AXE]:'#d8d8d8', [IRON_SWORD]:'#d8d8d8', [IRON_PICKAXE]:'#d8d8d8',
  [EGG_PIG]:'#f4a9c0', [EGG_SHEEP]:'#e8e8e8', [EGG_COW]:'#5b4636',
  [EGG_CHICKEN]:'#f1e7c6', [EGG_FISH]:'#3f7fa6', [EGG_VILLAGER]:'#9c6b3f', [EGG_KOKO]:'#d6b48a'
};
const EGG_TO_MOB = { [EGG_PIG]:'pig', [EGG_SHEEP]:'sheep', [EGG_COW]:'cow', [EGG_CHICKEN]:'chicken', [EGG_FISH]:'fish', [EGG_VILLAGER]:'villager', [EGG_KOKO]:'koko' };
const EGG_LIST = [EGG_PIG, EGG_SHEEP, EGG_COW, EGG_CHICKEN, EGG_FISH, EGG_VILLAGER, EGG_KOKO];
const isBlockType = t => t in BLOCK_MAT && t !== WATER;
const isEgg       = t => t in EGG_TO_MOB;
function itemName(t){ return BLOCK_NAME[t] || ITEM_NAME[t] || ('item ' + t); }
// procedural icons for items (so the hotbar shows little drawings, not flat colour)
function makeItemIcon(base, egg){
  const c = document.createElement('canvas'); c.width = c.height = 16; const g = c.getContext('2d');
  if (egg){
    g.fillStyle = base; g.beginPath(); g.ellipse(8, 9, 5, 6.5, 0, 0, Math.PI*2); g.fill();
    g.fillStyle = 'rgba(0,0,0,0.28)'; for (let k = 0; k < 6; k++) g.fillRect(4 + (k*5%7), 5 + (k*7%9), 2, 2);
  } else {
    g.fillStyle = base; g.fillRect(3, 4, 10, 8);
    g.fillStyle = 'rgba(255,255,255,0.25)'; g.fillRect(4, 5, 8, 2);
    g.fillStyle = 'rgba(0,0,0,0.22)'; g.fillRect(3, 11, 10, 1);
  }
  return c.toDataURL();
}
for (const t of [BACON, MUTTON, BEEF, CHICKEN_I, FISH_I]) { try { ICON_URL[t] = makeItemIcon(ITEM_COLOR[t], false); } catch(e){} }
for (const t of EGG_LIST) { try { ICON_URL[t] = makeItemIcon(ITEM_COLOR[t], true); } catch(e){} }
// stick + tool icons (pixel drawings, diagonal like Minecraft). `mat` tints the head/blade:
// 'wood' = brown, 'stone' = grey, 'iron' = light steel. Handles are always wood.
const TOOL_MATS = {
  wood:  { head: '#b9905a', hi: '#caa36a', sh: '#8a6a3a' },
  stone: { head: '#8f8f8f', hi: '#a8a8a8', sh: '#6f6f6f' },
  iron:  { head: '#d8d8d8', hi: '#efefef', sh: '#a9a9a9' }
};
function makeToolIcon(kind, mat){
  const c = document.createElement('canvas'); c.width = c.height = 16; const g = c.getContext('2d');
  const handle = '#9a6f43', dark = '#6e4d2a';
  const m = TOOL_MATS[mat] || TOOL_MATS.wood;
  const px = (x, y, w, h, col) => { g.fillStyle = col; g.fillRect(x, y, w, h); };
  if (kind === 'stick'){
    for (let i = 0; i < 6; i++){ px(11 - i, 3 + i, 2, 2, handle); px(11 - i, 3 + i, 1, 1, dark); }
  } else if (kind === 'sword'){
    px(4, 10, 3, 3, handle); px(4, 10, 3, 1, dark);                          // wooden hilt + guard
    for (let i = 0; i < 7; i++){ px(7 + i, 9 - i, 2, 2, m.head); px(7 + i, 9 - i, 1, 1, m.sh); }   // blade (tinted)
    px(12, 3, 2, 2, m.hi);                                                   // tip glint
  } else if (kind === 'pickaxe'){
    for (let i = 0; i < 7; i++){ px(11 - i, 4 + i, 2, 2, handle); px(11 - i, 4 + i, 1, 1, dark); }   // handle
    px(4, 2, 9, 2, m.head); px(4, 2, 9, 1, m.hi); px(4, 4, 2, 2, m.head); px(11, 4, 2, 2, m.head);    // curved head
  } else if (kind === 'axe'){
    for (let i = 0; i < 7; i++){ px(11 - i, 4 + i, 2, 2, handle); px(11 - i, 4 + i, 1, 1, dark); }   // handle
    px(8, 2, 5, 5, m.head); px(8, 2, 1, 5, m.hi); px(7, 3, 1, 3, m.head);                            // blade head
  }
  return c.toDataURL();
}
try { ICON_URL[STICK] = makeToolIcon('stick', 'wood'); } catch(e){}
const TOOL_ICONS = [
  [WOODEN_AXE,'axe','wood'], [WOODEN_SWORD,'sword','wood'], [WOODEN_PICKAXE,'pickaxe','wood'],
  [STONE_AXE,'axe','stone'], [STONE_SWORD,'sword','stone'], [STONE_PICKAXE,'pickaxe','stone'],
  [IRON_AXE,'axe','iron'],   [IRON_SWORD,'sword','iron'],   [IRON_PICKAXE,'pickaxe','iron']
];
for (const [id, kind, mat] of TOOL_ICONS){ try { ICON_URL[id] = makeToolIcon(kind, mat); } catch(e){} }
try { if (WATER_TEX && WATER_TEX.image && WATER_TEX.image.toDataURL) ICON_URL[WATER] = WATER_TEX.image.toDataURL(); } catch(e){}
const TOOL_LIST = [
  WOODEN_AXE, WOODEN_SWORD, WOODEN_PICKAXE,
  STONE_AXE, STONE_SWORD, STONE_PICKAXE,
  IRON_AXE, IRON_SWORD, IRON_PICKAXE, STICK
];
// the creative inventory palette: blocks, water, tools, then spawn eggs
const CREATIVE_PALETTE = BLOCK_TYPES.concat([WATER]).concat(TOOL_LIST).concat(EGG_LIST);

// inventory data: 9 hotbar slots + 36 main slots; each slot is null or {type,count}
const HOTBAR_N = 9, INV_N = 36;
const hotbar = new Array(HOTBAR_N).fill(null);
const invMain = new Array(INV_N).fill(null);
let selIndex = 0;          // selected hotbar slot
let carried = null;        // stack held on the cursor while the inventory is open

// ---- crafting ----
const craft2 = new Array(4).fill(null);    // 2x2 grid (inventory)
const craft3 = new Array(9).fill(null);    // 3x3 grid (crafting table)
const RECIPES = [
  { shapeless: true, need: [WOOD], out: { type: WOOD_PLANKS, count: 4 } },                       // 1 wood -> 4 planks
  { rows: [[WOOD_PLANKS, WOOD_PLANKS], [WOOD_PLANKS, WOOD_PLANKS]], out: { type: CRAFTING_TABLE, count: 1 } }, // 4 planks -> table
  { rows: [[WOOD_PLANKS], [WOOD_PLANKS]], out: { type: STICK, count: 4 } },                       // 2 planks -> 4 sticks
  // 3x3 tool recipes (1 = stick, 3 = planks)
  { rows: [[WOOD_PLANKS, WOOD_PLANKS], [STICK, WOOD_PLANKS], [STICK, null]], out: { type: WOODEN_AXE, count: 1 } },     // #33 / #13 / #1#
  { rows: [[WOOD_PLANKS], [WOOD_PLANKS], [STICK]], out: { type: WOODEN_SWORD, count: 1 } },                            // #3# / #3# / #1#
  { rows: [[WOOD_PLANKS, WOOD_PLANKS, WOOD_PLANKS], [null, STICK, null], [null, STICK, null]], out: { type: WOODEN_PICKAXE, count: 1 } }, // 333 / #1# / #1#
  // stone tools (4 = stone)
  { rows: [[STONE, STONE], [STICK, STONE], [STICK, null]], out: { type: STONE_AXE, count: 1 } },                       // #44 / #14 / #1#
  { rows: [[STONE], [STONE], [STICK]], out: { type: STONE_SWORD, count: 1 } },                                         // #4# / #4# / #1#
  { rows: [[STONE, STONE, STONE], [null, STICK, null], [null, STICK, null]], out: { type: STONE_PICKAXE, count: 1 } }, // 444 / #1# / #1#
  // iron tools (5 = iron ore)
  { rows: [[IRON_ORE, IRON_ORE], [STICK, IRON_ORE], [STICK, null]], out: { type: IRON_AXE, count: 1 } },               // #55 / #15 / #1#
  { rows: [[IRON_ORE], [IRON_ORE], [STICK]], out: { type: IRON_SWORD, count: 1 } },                                    // #5# / #5# / #1#
  { rows: [[IRON_ORE, IRON_ORE, IRON_ORE], [null, STICK, null], [null, STICK, null]], out: { type: IRON_PICKAXE, count: 1 } }, // 555 / #1# / #1#
];
// match a grid (length dim*dim) against the recipe list; returns {type,count} or null
function craftMatch(grid, dim){
  const items = grid.filter(s => s).map(s => s.type);
  if (!items.length) return null;
  for (const r of RECIPES){                      // shapeless recipes
    if (!r.shapeless) continue;
    if (items.length === r.need.length){
      const a = items.slice().sort(), b = r.need.slice().sort();
      if (a.every((v, i) => v === b[i])) return r.out;
    }
  }
  let minR = dim, maxR = -1, minC = dim, maxC = -1;     // shaped: trim to bounding box
  for (let r = 0; r < dim; r++) for (let c = 0; c < dim; c++)
    if (grid[r*dim+c]){ if (r<minR)minR=r; if (r>maxR)maxR=r; if (c<minC)minC=c; if (c>maxC)maxC=c; }
  if (maxR < 0) return null;
  const h = maxR-minR+1, w = maxC-minC+1;
  for (const rec of RECIPES){
    if (!rec.rows) continue;
    if (rec.rows.length !== h || rec.rows[0].length !== w) continue;
    let ok = true;
    for (let r = 0; r < h && ok; r++) for (let c = 0; c < w; c++){
      const want = rec.rows[r][c] || null;
      const got = grid[(minR+r)*dim+(minC+c)];
      if (want !== (got ? got.type : null)){ ok = false; break; }
    }
    if (ok) return rec.out;
  }
  return null;
}
function craftConsume(grid){
  for (let i = 0; i < grid.length; i++){ const s = grid[i]; if (s && s.count !== Infinity){ s.count--; if (s.count <= 0) grid[i] = null; } }
}
function clickCraftOutput(grid, dim){
  const out = craftMatch(grid, dim); if (!out) return;
  if (carried == null) carried = { type: out.type, count: out.count };
  else if (carried.type === out.type && carried.count !== Infinity) carried.count += out.count;
  else return;                                   // hands full with a different item
  craftConsume(grid);
  refreshOpenMenus(); refreshHotbar(); renderCarry();
}
function makeOutputSlot(grid, dim){
  const out = craftMatch(grid, dim);
  const cell = document.createElement('div'); cell.className = 'slot out';
  if (out){
    const ic = document.createElement('div'); ic.className = 'icon'; setIcon(ic, out.type); cell.appendChild(ic);
    if (out.count > 1){ const c = document.createElement('span'); c.className = 'count'; c.textContent = String(out.count); cell.appendChild(c); }
  }
  cell.addEventListener('click', () => clickCraftOutput(grid, dim));
  return cell;
}
function refreshOpenMenus(){ if (invOpen) refreshInventory(); if (tableOpen) refreshTableMenu(); }
// drop any items left in a crafting grid into the world at the player's feet, then clear it
function dumpCraftGrid(grid){
  for (let i = 0; i < grid.length; i++){
    const s = grid[i]; if (!s) continue;
    const n = s.count === Infinity ? 1 : s.count;
    for (let k = 0; k < n && k < 12; k++) spawnDrop(s.type, Math.floor(benny.position.x), Math.floor(benny.position.y), Math.floor(benny.position.z));
    grid[i] = null;
  }
}
let tableOpen = false;
function refreshTableMenu(){
  const g = el('craft3grid'); if (!g) return;
  g.innerHTML = ''; for (let i = 0; i < 9; i++) g.appendChild(makeSlot(craft3, i));
  const o = el('craft3out'); o.innerHTML = ''; o.appendChild(makeOutputSlot(craft3, 3));
  const inv = el('tableInv'); inv.innerHTML = ''; for (let i = 0; i < INV_N; i++) inv.appendChild(makeSlot(invMain, i));
  const hb = el('tableHotbar'); hb.innerHTML = ''; for (let i = 0; i < HOTBAR_N; i++){ const c = makeSlot(hotbar, i); if (i === selIndex) c.classList.add('sel'); hb.appendChild(c); }
}
function openTableMenu(){
  tableOpen = true; refreshTableMenu(); renderCarry();
  el('tableMenu').classList.add('show');
  if (document.pointerLockElement === canvas) document.exitPointerLock();
}
function closeTableMenu(){
  tableOpen = false; el("tableMenu").classList.remove("show"); hideSlotTip();
  dumpCraftGrid(craft3);
  if (carried){ addItem(carried.type, carried.count === Infinity ? 1 : carried.count); carried = null; renderCarry(); }
  refreshHotbar();
  canvas.requestPointerLock();
}

function slotCountLabel(s){ return (!s || s.count === Infinity || s.count <= 1) ? '' : String(s.count); }
function setIcon(elm, type){
  const url = ICON_URL[type];
  if (url){ elm.style.backgroundImage = 'url(' + url + ')'; elm.style.backgroundSize = 'cover'; elm.style.backgroundColor = 'transparent'; }
  else { elm.style.backgroundImage = 'none'; elm.style.background = BLOCK_COLOR[type] || ITEM_COLOR[type] || '#999'; }
}

// add `count` of `type`, stacking then filling empty slots (hotbar first). Returns leftover.
function addItem(type, count){
  count = count == null ? 1 : count;
  const arrs = [hotbar, invMain];
  for (const arr of arrs) for (let i = 0; i < arr.length && count > 0; i++){
    const s = arr[i];
    if (s && s.type === type && s.count !== Infinity && s.count < STACK_MAX){
      const add = Math.min(STACK_MAX - s.count, count); s.count += add; count -= add;
    }
  }
  for (const arr of arrs) for (let i = 0; i < arr.length && count > 0; i++){
    if (!arr[i]){ const add = Math.min(STACK_MAX, count); arr[i] = { type, count: add }; count -= add; }
  }
  refreshHotbar(); if (invOpen) refreshInventory();
  return count;
}
// creative: give an endless stack of a block
function giveCreative(type){
  const arrs = [hotbar, invMain];
  for (const arr of arrs) for (let i = 0; i < arr.length; i++) if (arr[i] && arr[i].type === type){ arr[i].count = Infinity; refreshHotbar(); if (invOpen) refreshInventory(); return; }
  for (const arr of arrs) for (let i = 0; i < arr.length; i++) if (!arr[i]){ arr[i] = { type, count: Infinity }; refreshHotbar(); if (invOpen) refreshInventory(); return; }
}
function selectedStack(){ return hotbar[selIndex]; }
function clearInventory(){ hotbar.fill(null); invMain.fill(null); craft2.fill(null); craft3.fill(null); selIndex = 0; carried = null; refreshHotbar(); }
function serializeInv(arr){ return arr.map(s => s ? [s.type, s.count === Infinity ? -1 : s.count] : 0); }
function loadInv(arr, data){ for (let i = 0; i < arr.length; i++){ const d = data && data[i]; arr[i] = (d && d !== 0) ? { type: d[0], count: d[1] === -1 ? Infinity : d[1] } : null; } }
function consumeSelected(){
  const s = hotbar[selIndex]; if (!s) return;
  if (gamemode === 'survival' && s.count !== Infinity){ s.count--; if (s.count <= 0) hotbar[selIndex] = null; refreshHotbar(); }
}

// ---------- health, food (hunger) & dying ----------
const FOOD_SET = new Set([BACON, MUTTON, BEEF, CHICKEN_I, FISH_I]);
const FOOD_RESTORE = { [BACON]: 6, [MUTTON]: 6, [BEEF]: 8, [CHICKEN_I]: 6, [FISH_I]: 5 };   // hunger points (each icon = 2)
const isFood = t => FOOD_SET.has(t);
function svgURL(svg){ return 'url("data:image/svg+xml;utf8,' + encodeURIComponent(svg) + '")'; }
const HEART_PATH = 'M9 16C9 16 1 11 1 5.6 1 3.1 3 1.2 5.4 1.2 6.9 1.2 8.2 2 9 3.2 9.8 2 11.1 1.2 12.6 1.2 15 1.2 17 3.1 17 5.6 17 11 9 16 9 16Z';
const HEART_FULL = svgURL('<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 18 18"><path d="' + HEART_PATH + '" fill="#e23b3b" stroke="#7a1414" stroke-width="1.4"/></svg>');
const HEART_EMPTY = svgURL('<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 18 18"><path d="' + HEART_PATH + '" fill="#39202c" stroke="#23141c" stroke-width="1.4"/></svg>');
const FOOD_FULL = svgURL('<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 18 18"><ellipse cx="7" cy="8" rx="5" ry="6" fill="#c8803a" stroke="#5b3a1a" stroke-width="1.2"/><rect x="9.5" y="11" width="6.5" height="3" rx="1.5" fill="#f0e3c6" stroke="#5b3a1a" stroke-width="1"/></svg>');
const FOOD_EMPTY = svgURL('<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 18 18"><ellipse cx="7" cy="8" rx="5" ry="6" fill="#2e241a" stroke="#1a140d" stroke-width="1.2"/><rect x="9.5" y="11" width="6.5" height="3" rx="1.5" fill="#3a342a" stroke="#1a140d" stroke-width="1"/></svg>');
function buildStatRow(row, val, fullURL, emptyURL){
  if (!row) return; row.innerHTML = '';
  for (let i = 0; i < 10; i++){
    const frac = Math.max(0, Math.min(1, val / 2 - i));      // each icon represents 2 points (full / half / empty)
    const icon = document.createElement('div'); icon.className = 'stat-icon';
    const base = document.createElement('div'); base.className = 'base'; base.style.backgroundImage = emptyURL;
    const fill = document.createElement('div'); fill.className = 'fill'; fill.style.width = (frac * 18) + 'px';
    const inner = document.createElement('i'); inner.style.backgroundImage = fullURL;
    fill.appendChild(inner); icon.appendChild(base); icon.appendChild(fill); row.appendChild(icon);
  }
}
function refreshStatBars(){
  buildStatRow(el('heartRow'), health, HEART_FULL, HEART_EMPTY);
  buildStatRow(el('foodRow'), food, FOOD_FULL, FOOD_EMPTY);
}
// ---- experience: earned from kills + mining ore ----
let xpLevel = 0, xpInto = 0;
function xpNeed(l){ return 4 + l * 2; }
function refreshXpBar(){
  const need = xpNeed(xpLevel);
  const f = el('xpFill'); if (f) f.style.width = Math.max(0, Math.min(100, (xpInto / need) * 100)) + '%';
  const lv = el('xpLevel'); if (lv) lv.textContent = xpLevel;
}
function addXp(n){
  xpInto += n;
  while (xpInto >= xpNeed(xpLevel)){ xpInto -= xpNeed(xpLevel); xpLevel++; }
  refreshXpBar();
}
function resetXp(){ xpLevel = 0; xpInto = 0; refreshXpBar(); }
function resetVitals(){ health = maxHealth; food = maxFood; dead = false; fallPeakY = null; statT = 0; refreshStatBars(); }
function respawnToCircle(){ benny.position.set(0.5, PARK_Y + 1.2, 0.5); vy = 0; resetVitals(); fallPeakY = null; chunksDirty = true; }
function parkourTick(){
  if (benny.position.y < PARK_VOID_Y){ respawnToCircle(); return; }   // fell into the void -> back to start
  if (!parkourWon){
    const f = PARKOUR.finish;
    const ddx = benny.position.x - (f.x + 0.5), ddz = benny.position.z - (f.z + 0.5), ddy = benny.position.y - f.y;
    if (ddx*ddx + ddz*ddz < 4 && ddy > -1 && ddy < 3.5){ parkourWon = true; announceSys(username + ' completed the Extreme Challenge!'); }
  }
}
function damagePlayer(amount, cause){
  if (gamemode !== 'survival' || dead || amount <= 0) return;
  health = Math.max(0, health - amount);
  refreshStatBars();
  flashSelfHurt();                                   // red screen flash
  if (net.active) netSendHurt();                     // make my avatar flash red for everyone
  if (health <= 0) die(cause);
}
function flashSelfHurt(){
  const e = el('hurtFlash'); if (!e) return;
  e.classList.add('show');
  requestAnimationFrame(() => requestAnimationFrame(() => e.classList.remove('show')));
}
function flashAvatar(id){
  const rp = remotePlayers.get(id); if (!rp) return;
  const mats = [];
  rp.group.traverse(o => { if (o.isMesh && o.material && o.material.color){ mats.push(o.material); o.material.color.setHex(0xff5050); } });
  setTimeout(() => { mats.forEach(m => m.color.setHex(0xffffff)); }, 400);
}
function netSendHurt(){
  if (!net.active) return;
  if (net.isHost) broadcast({ t:'hurt', id:'host' }); else netSend(net.hostConn, { t:'hurt' });
}
const DEATH_CAUSE = { fall: 'hit the ground too hard', void: 'fell out of the world', starve: 'starved to death' };
function dropInventoryOnDeath(){
  const px = benny.position.x, py = benny.position.y + 0.5, pz = benny.position.z;
  const spill = (s) => { if (!s) return; const ang = Math.random() * 6.283; spawnDrop(s.type, 0, 0, 0, { count: s.count === Infinity ? 1 : s.count, x: px, y: py, z: pz, vx: Math.cos(ang) * 2.5, vy: 3, vz: Math.sin(ang) * 2.5 }); };
  hotbar.forEach(spill); invMain.forEach(spill); if (carried) spill(carried);
  clearInventory();
}
// PvP Hub kit: a sword + 64 beef, refreshed on every (re)spawn
function giveHubLoadout(){
  clearInventory();
  hotbar[0] = { type: WOODEN_SWORD, count: 1 };
  hotbar[1] = { type: BEEF, count: 64 };
  selIndex = 0; refreshHotbar();
}
function netSendDied(){ if (!net.active) return; if (net.isHost) broadcast({ t:'died', id:'host' }); else netSend(net.hostConn, { t:'died' }); }
function netSendResp(){ if (!net.active) return; if (net.isHost) broadcast({ t:'resp', id:'host' }); else netSend(net.hostConn, { t:'resp' }); }
function die(cause){
  if (worldMode === 'extreme'){ respawnToCircle(); return; }   // parkour: just bounce back to the start circle
  dead = true;
  const reason = (cause === 'player') ? ('was slain by ' + (lastAttacker || 'another player')) : (DEATH_CAUSE[cause] || 'died');
  el('deathMsg').textContent = (cause === 'player') ? ('You were slain by ' + (lastAttacker || 'another player') + '.') : ('You ' + reason + '.');
  el('deathScreen').classList.add('show');
  el('statBars').classList.remove('show'); el('xpWrap').classList.remove('show');
  if (worldMode !== 'pvphub') dropInventoryOnDeath();        // your items spill on the ground (kept on the PvP Hub)
  if (document.pointerLockElement === canvas) document.exitPointerLock();
  if (net.active){ announceSys(username + ' ' + reason); netSendDied(); }   // message + hide your body for others
}
function eatSelected(){
  if (gamemode !== 'survival') return;                       // creative doesn't need food
  const s = selectedStack(); if (!s || !isFood(s.type)) return;
  if (food >= maxFood && health >= maxHealth) return;        // already full
  const fv = FOOD_RESTORE[s.type] || 4;
  food = Math.min(maxFood, food + fv);
  health = Math.min(maxHealth, health + Math.ceil(fv / 2));   // eating also heals
  if (s.count !== Infinity){ s.count--; if (s.count <= 0) hotbar[selIndex] = null; refreshHotbar(); }
  refreshStatBars();
}
function respawn(){
  resetVitals();
  resetXp();                                                 // experience is lost on death
  vy = 0; mineHeld = false;
  placeBenny();                                              // back to the world spawn
  if (worldMode === 'pvphub') giveHubLoadout();              // fresh sword + 64 beef on respawn
  el('deathScreen').classList.remove('show');
  chunksDirty = true; refreshChunkQueue();
  if (net.active) netSendResp();                             // show your body to others again
  canvas.requestPointerLock();
}
// hunger now drains from activity (jumps + sprinting), not idle time
function loseHunger(n){ if (ruleNoHunger() || gamemode !== 'survival' || dead) return; if (food > 0){ food = Math.max(0, food - n); refreshStatBars(); } }
function onJump(){ if (gamemode !== 'survival') return; jumpCount++; if (jumpCount >= 5){ jumpCount = 0; loseHunger(1); } }   // every 5 jumps
function sprintDrain(dt){ sprintAcc += dt; if (sprintAcc >= 2){ sprintAcc = 0; loseHunger(1); } }                          // ~1 per 2s sprinting
function updateVitals(dt){
  if (gamemode !== 'survival' || dead) return;
  statT += dt;
  if (statT >= 4){
    statT = 0;
    if (food >= 16 && health < maxHealth) health = Math.min(maxHealth, health + 1);   // regen when well-fed
    else if (food <= 0 && health > 1) health = Math.max(1, health - 1);               // starving (never fatal on its own)
    refreshStatBars();
  }
}

// ---- hotbar UI ----
function refreshHotbar(){
  const bar = el('hotbar'); if (!bar) return;
  bar.innerHTML = '';
  for (let i = 0; i < HOTBAR_N; i++){
    const s = hotbar[i];
    const cell = document.createElement('div');
    cell.className = 'slot' + (i === selIndex ? ' sel' : '');
    if (s){
      const ic = document.createElement('div'); ic.className = 'icon'; setIcon(ic, s.type);
      cell.appendChild(ic);
      const lab = slotCountLabel(s); if (lab){ const c = document.createElement('span'); c.className = 'count'; c.textContent = lab; cell.appendChild(c); }
    }
    bar.appendChild(cell);
  }
}

// ---- inventory overlay UI ----
function makeSlot(arr, i){
  const s = arr[i];
  const cell = document.createElement('div'); cell.className = 'slot';
  if (s){
    const ic = document.createElement('div'); ic.className = 'icon'; setIcon(ic, s.type); cell.appendChild(ic);
    const lab = slotCountLabel(s); if (lab){ const c = document.createElement('span'); c.className = 'count'; c.textContent = lab; cell.appendChild(c); }
    cell.addEventListener('mouseenter', () => showSlotTip(cell, itemName(s.type)));   // name on hover
    cell.addEventListener('mousemove', () => showSlotTip(cell, itemName(s.type)));
    cell.addEventListener('mouseleave', hideSlotTip);
  }
  cell.addEventListener('click', () => { hideSlotTip(); clickSlot(arr, i); });
  cell.addEventListener('contextmenu', (e) => { e.preventDefault(); rightClickSlot(arr, i); });
  return cell;
}
function showSlotTip(cell, name){
  const tip = el('slotTip'); if (!tip) return;
  const r = cell.getBoundingClientRect();
  tip.textContent = name;
  tip.style.left = (r.left + r.width / 2) + 'px';
  tip.style.top  = (r.top - 6) + 'px';
  tip.classList.add('show');
}
function hideSlotTip(){ const tip = el('slotTip'); if (tip) tip.classList.remove('show'); }
// the held item's name shows above the hotbar for ~4s when you switch slots, then fades
let heldNameT = null;
function showHeldName(){
  const e = el('heldName'); if (!e) return;
  const s = hotbar[selIndex];
  if (!s){ e.classList.remove('show'); return; }
  e.textContent = itemName(s.type);
  e.classList.add('show');
  if (heldNameT) clearTimeout(heldNameT);
  heldNameT = setTimeout(() => e.classList.remove('show'), 4000);
}
function refreshInventory(){
  const inv = el('invGrid'), hb = el('invHotbar'), pal = el('palette');
  if (!inv) return;
  inv.innerHTML = ''; for (let i = 0; i < INV_N; i++) inv.appendChild(makeSlot(invMain, i));
  hb.innerHTML  = ''; for (let i = 0; i < HOTBAR_N; i++){ const c = makeSlot(hotbar, i); if (i === selIndex) c.classList.add('sel'); hb.appendChild(c); }
  const cg = el('craft2grid');
  if (cg){ cg.innerHTML = ''; for (let i = 0; i < 4; i++) cg.appendChild(makeSlot(craft2, i)); }
  const co = el('craft2out'); if (co){ co.innerHTML = ''; co.appendChild(makeOutputSlot(craft2, 2)); }
  pal.innerHTML = '';
  CREATIVE_PALETTE.forEach(t => {
    const cell = document.createElement('div'); cell.className = 'slot pal';
    const ic = document.createElement('div'); ic.className = 'icon'; setIcon(ic, t); cell.appendChild(ic);
    cell.title = itemName(t);
    cell.addEventListener('click', () => { if (gamemode === 'creative') giveCreative(t); });
    pal.appendChild(cell);
  });
}
// click-to-move: pick up / drop / swap / merge a stack between any two slots
function clickSlot(arr, i){
  const s = arr[i];
  if (carried == null){ if (s){ carried = s; arr[i] = null; } }
  else {
    if (!s) { arr[i] = carried; carried = null; }
    else if (s.type === carried.type && s.count !== Infinity && carried.count !== Infinity){
      const add = Math.min(STACK_MAX - s.count, carried.count); s.count += add; carried.count -= add; if (carried.count <= 0) carried = null;
    } else { arr[i] = carried; carried = s; }
  }
  refreshOpenMenus(); refreshHotbar(); renderCarry();
}
// right-click: empty hand picks up half a stack; full hand drops one item into the slot
function rightClickSlot(arr, i){
  const s = arr[i];
  if (carried == null){
    if (s && s.count !== Infinity && s.count > 1){
      const half = Math.ceil(s.count / 2);
      carried = { type: s.type, count: half };
      s.count -= half; if (s.count <= 0) arr[i] = null;
    } else if (s){ carried = s; arr[i] = null; }          // single (or infinite) stack: take it all
  } else {
    if (!s){
      arr[i] = { type: carried.type, count: 1 };
      if (carried.count !== Infinity){ carried.count--; if (carried.count <= 0) carried = null; }
    } else if (s.type === carried.type && s.count !== Infinity && s.count < STACK_MAX){
      s.count++;
      if (carried.count !== Infinity){ carried.count--; if (carried.count <= 0) carried = null; }
    }
  }
  refreshOpenMenus(); refreshHotbar(); renderCarry();
}
function renderCarry(){
  const c = el('carry'); if (!c) return;
  if (carried){ c.style.display = 'block'; setIcon(c, carried.type); c.style.backgroundSize = 'cover'; c.textContent = slotCountLabel(carried); }
  else c.style.display = 'none';
}

let invOpen = false;
function openInventory(){
  invOpen = true;
  refreshInventory(); renderCarry();
  el('inventory').classList.add('show');
  if (document.pointerLockElement === canvas) document.exitPointerLock();
}
function closeInventory(){
  invOpen = false;
  el('inventory').classList.remove('show'); hideSlotTip();
  dumpCraftGrid(craft2);                        // leftover crafting items drop at the player's feet
  if (carried){ addItem(carried.type, carried.count === Infinity ? 1 : carried.count); carried = null; renderCarry(); }
  canvas.requestPointerLock();
}
el('invClose').addEventListener('click', closeInventory);
el('tableClose').addEventListener('click', closeTableMenu);
el('managerClose').addEventListener('click', closeManager);
el('signSave').addEventListener('click', saveSignEditor);
el('signCancel').addEventListener('click', closeSignEditor);
el('signInput').addEventListener('keydown', (e) => {
  e.stopPropagation();                                  // let the input type normally
  if (e.code === 'Enter') saveSignEditor();
  else if (e.code === 'Escape') closeSignEditor();
});

// ---- breaking & placing ----
const drops = [];
const MAX_DROPS = 40, DROP_LIFE = 45;        // bound the count + lifetime so rapid breaking never tanks FPS
const itemDropMat = {};                       // small coloured cubes for non-block item drops
function dropMaterial(type){
  if (isBlockType(type)) return BLOCK_MAT[type];
  if (!itemDropMat[type]) itemDropMat[type] = new THREE.MeshLambertMaterial({ color: new THREE.Color(ITEM_COLOR[type] || '#cccccc') });
  return itemDropMat[type];
}
let dropSeq = 0;
function newDropId(){ return (account || 'g') + '-' + (Date.now() % 100000) + '-' + (dropSeq++); }
function spawnDrop(type, wx, wy, wz, opts){
  opts = opts || {};
  if (drops.length >= MAX_DROPS){ const old = drops.shift(); scene.remove(old.mesh); }
  const m = new THREE.Mesh(UNIT_CUBE, dropMaterial(type)); m.scale.set(0.28, 0.28, 0.28);
  const x = (opts.x != null) ? opts.x : wx + 0.5, y = (opts.y != null) ? opts.y : wy + 0.4, z = (opts.z != null) ? opts.z : wz + 0.5;
  m.position.set(x, y, z); m.castShadow = false; m.receiveShadow = false;
  m.frustumCulled = true; scene.add(m);                       // only drawn when on-screen
  const d = { id: opts.id || newDropId(), mesh: m, type, count: opts.count || 1, x, y, z, vx: opts.vx || 0, vy: (opts.vy != null) ? opts.vy : 2.2, vz: opts.vz || 0, age: 0 };
  drops.push(d);
  if (net.active && !opts.remote) netSendDrop(d);            // share the drop so other players see/grab it
  return d;
}
function removeDropById(id){ for (let i = drops.length - 1; i >= 0; i--){ if (drops[i].id === id){ scene.remove(drops[i].mesh); drops.splice(i, 1); return; } } }
function updateDrops(dt){
  const now = performance.now();
  for (let i = drops.length - 1; i >= 0; i--){
    const d = drops[i];
    d.age += dt;
    if (d.age > DROP_LIFE){ scene.remove(d.mesh); drops.splice(i, 1); continue; }
    d.vy -= 20 * dt; d.y += d.vy * dt; d.x += d.vx * dt; d.z += d.vz * dt; d.vx *= 0.9; d.vz *= 0.9;
    if (isOpaque(blockWorld(Math.floor(d.x), Math.floor(d.y - 0.2), Math.floor(d.z)))){ d.y = Math.floor(d.y - 0.2) + 1.18; d.vy = 0; }
    d.mesh.position.set(d.x, d.y + Math.sin(now / 300 + i) * 0.05, d.z);
    d.mesh.rotation.y += dt * 1.6;
    const dx = d.x - benny.position.x, dz = d.z - benny.position.z, dy = d.y - (benny.position.y + 0.9);
    if (d.age > 0.5 && dx*dx + dy*dy + dz*dz < 1.7){       // walk over a drop to collect it
      addItem(d.type, d.count || 1);
      if (net.active) netSendPickup(d.id);                  // tell others it's gone
      scene.remove(d.mesh); drops.splice(i, 1);
    }
  }
}
function dropSelected(){
  const s = selectedStack(); if (!s) return;
  if (isPvpHub && (s.type === WOODEN_SWORD || s.type === BEEF)) return;   // sword & beef are locked here
  const type = s.type;
  consumeSelected();                                         // remove one from the held slot
  const ex = benny.position.x + Math.sin(yaw) * 0.6, ey = benny.position.y + 1.0, ez = benny.position.z + Math.cos(yaw) * 0.6;
  spawnDrop(type, 0, 0, 0, { x: ex, y: ey, z: ez, vx: Math.sin(yaw) * 4, vy: 3, vz: Math.cos(yaw) * 4 });
}
function floodWaterDown(wx, wy, wz){
  // water with nothing under it falls down, filling the gap until it meets a solid block
  let y = wy, guard = 0;
  while (y >= 0 && guard < 96 && blockWorld(wx, y, wz) === AIR){
    setBlockWorld(wx, y, wz, WATER);
    netSendEdit(wx, y, wz, WATER);
    y--; guard++;
  }
}
function doBreak(tgt){
  if (!tgt) return;
  if (ruleNoBuild()) return;                            // breaking disabled on this server
  removeSignAt(signKey(tgt.wx, tgt.wy, tgt.wz));        // a broken sign loses its text
  const XP_ORE = { [COAL_ORE]: 1, [IRON_ORE]: 3, [DIAMOND_ORE]: 7 };
  const isOre = (tgt.type === COAL_ORE || tgt.type === IRON_ORE || tgt.type === DIAMOND_ORE);
  const harvest = !isOre || isPickaxe(heldTool());     // ore only drops (and grants XP) when mined with a pickaxe
  if (harvest){
    if (XP_ORE[tgt.type]) addXp(XP_ORE[tgt.type]);     // mining ore grants experience
    spawnDrop(tgt.type, tgt.wx, tgt.wy, tgt.wz);
  }
  setBlockWorld(tgt.wx, tgt.wy, tgt.wz, AIR);
  netSendEdit(tgt.wx, tgt.wy, tgt.wz, AIR);
  nudgeWaterAround(tgt.wx, tgt.wy, tgt.wz);             // nearby water flows into the new gap (down + sideways)
  hideCrack();
}
function blockOverlapsPlayer(wx, wy, wz){
  const px = benny.position.x, py = benny.position.y, pz = benny.position.z;
  return (wx + 1 > px - 0.3 && wx < px + 0.3) && (wy + 1 > py && wy < py + 1.8) && (wz + 1 > pz - 0.3 && wz < pz + 0.3);
}
function placeEquipped(){
  if (ruleNoBuild()) return;                            // placing disabled on this server
  const s = selectedStack(); if (!s) return;
  const tgt = raycastBlock(); if (!tgt) return;
  const px = tgt.wx + tgt.normal.x, py = tgt.wy + tgt.normal.y, pz = tgt.wz + tgt.normal.z;
  if (py < 0 || py >= WORLD_H) return;
  if (isEgg(s.type)){                                   // right-click a spawn egg -> place the mob
    spawnMob(EGG_TO_MOB[s.type], px + 0.5, py, pz + 0.5);
    consumeSelected();
    return;
  }
  if (!isBlockType(s.type) && s.type !== WATER) return;  // foods/tools aren't placeable
  const cur = blockWorld(px, py, pz);
  if (cur !== AIR && cur !== WATER) return;             // can place into air or water (replaces the water)
  if (s.type !== WATER && blockOverlapsPlayer(px, py, pz)) return;
  setBlockWorld(px, py, pz, s.type);
  netSendEdit(px, py, pz, s.type);
  if (s.type === WATER) seedFlow(px, py, pz, 0);        // start the flow from the placed source
  consumeSelected();
}

// ---------- signs: placeable boards you can type a message on ----------
const signs = new Map();                         // "wx,wy,wz" -> { text, sprite }
function signKey(wx, wy, wz){ return wx + ',' + wy + ',' + wz; }
function makeSignSprite(text){
  const c = document.createElement('canvas'); c.width = 256; c.height = 128;
  const g = c.getContext('2d');
  g.fillStyle = 'rgba(150,110,66,0.96)'; g.fillRect(0, 0, 256, 128);
  g.strokeStyle = 'rgba(74,50,28,1)'; g.lineWidth = 6; g.strokeRect(3, 3, 250, 122);
  g.fillStyle = '#2c1c0e'; g.font = 'bold 30px sans-serif'; g.textAlign = 'center'; g.textBaseline = 'middle';
  const words = (text || '').split(/\s+/); const lines = []; let cur = '';
  for (const w of words){ const test = cur ? cur + ' ' + w : w; if (g.measureText(test).width > 232 && cur){ lines.push(cur); cur = w; } else cur = test; }
  if (cur) lines.push(cur); if (!lines.length) lines.push('');
  const show = lines.slice(0, 4); const lh = 32; const y0 = 64 - (show.length - 1) * lh / 2;
  show.forEach((ln, i) => g.fillText(ln, 128, y0 + i * lh));
  const tex = new THREE.CanvasTexture(c);
  const sp = new THREE.Sprite(new THREE.SpriteMaterial({ map: tex, depthTest: true, transparent: true }));
  sp.scale.set(1.5, 0.75, 1);
  return sp;
}
function setSignText(key, text){
  let rec = signs.get(key);
  const p = key.split(',').map(Number);
  if (!rec){ rec = { text: '', sprite: null }; signs.set(key, rec); }
  rec.text = text;
  if (rec.sprite) scene.remove(rec.sprite);
  rec.sprite = makeSignSprite(text);
  rec.sprite.position.set(p[0] + 0.5, p[1] + 1.05, p[2] + 0.5);
  scene.add(rec.sprite);
}
function removeSignAt(key){
  const rec = signs.get(key);
  if (rec){ if (rec.sprite) scene.remove(rec.sprite); signs.delete(key); }
}
function clearSigns(){ signs.forEach(r => { if (r.sprite) scene.remove(r.sprite); }); signs.clear(); }

let signEditOpen = false, signEditKey = null;
function openSignEditor(wx, wy, wz){
  signEditKey = signKey(wx, wy, wz);
  signEditOpen = true;
  const rec = signs.get(signEditKey);
  el('signInput').value = rec ? rec.text : '';
  el('signEditor').classList.add('show');
  if (document.pointerLockElement === canvas) document.exitPointerLock();
  setTimeout(() => { try { el('signInput').focus(); } catch (e){} }, 0);
}
function saveSignEditor(){
  if (!signEditOpen) return;
  const text = (el('signInput').value || '').slice(0, 60);
  setSignText(signEditKey, text);
  netSendSign(signEditKey, text);
  closeSignEditor();
}
function closeSignEditor(){
  signEditOpen = false;
  el('signEditor').classList.remove('show');
  canvas.requestPointerLock();
}
function netSendSign(key, text){
  if (!net.active) return;
  const m = { t: 'sign', key, text };
  if (net.isHost) broadcast(m); else netSend(net.hostConn, m);
}

// crack overlay (breaking animation)
let crackMesh = null; const crackTex = [];
function buildCrackTextures(){
  for (let stage = 0; stage < 6; stage++){
    const c = document.createElement('canvas'); c.width = c.height = 16; const g = c.getContext('2d');
    g.clearRect(0, 0, 16, 16); g.strokeStyle = 'rgba(0,0,0,0.85)'; g.lineWidth = 1;
    const lines = stage * 3;
    for (let k = 0; k < lines; k++){
      const x = (k * 53 % 16), y = (k * 97 % 16);
      g.beginPath(); g.moveTo(x, y); g.lineTo((x + 4 + (k%5)) % 16, (y + 5 + (k%4)) % 16); g.stroke();
    }
    const tx = new THREE.CanvasTexture(c); tx.magFilter = THREE.NearestFilter; tx.minFilter = THREE.NearestFilter;
    crackTex.push(tx);
  }
  crackMesh = new THREE.Mesh(new THREE.BoxGeometry(1.02, 1.02, 1.02),
    new THREE.MeshBasicMaterial({ transparent: true, opacity: 0.7, depthWrite: false, polygonOffset: true, polygonOffsetFactor: -1 }));
  crackMesh.visible = false; scene.add(crackMesh);
}
function showCrack(tgt, prog){
  if (!crackMesh) return;
  const stage = Math.max(0, Math.min(crackTex.length - 1, Math.floor(prog * crackTex.length)));
  crackMesh.material.map = crackTex[stage]; crackMesh.material.needsUpdate = true;
  crackMesh.position.set(tgt.wx + 0.5, tgt.wy + 0.5, tgt.wz + 0.5);
  crackMesh.visible = true;
}
function hideCrack(){ if (crackMesh) crackMesh.visible = false; }

// ---- survival physics state ----
let vy = 0, onGround = false, mineHeld = false;
let kbx = 0, kbz = 0;                        // horizontal knockback impulse (from being hit)
let health = 20, maxHealth = 20, food = 20, maxFood = 20;   // 10 hearts, 10 drumsticks (each icon = 2 points)
let dead = false, fallPeakY = null, statT = 0;
let jumpCount = 0, sprintAcc = 0;     // hunger now drains from jumping and sprinting
let lastAttacker = null, lastAttackT = 0;
const PVP_DMG = 4;                    // damage per melee hit on another player (2 hearts)
let mineKey = null, mineProgress = 0;
const PR = 0.3, PH = 1.8, GRAVITY = 26, JUMP_V = 8.6, WALK = 5.4;
const SWIM_UP = 26, SWIM_SINK = 3, SWIM_BUOY = 8;   // paddle-up accel, gentle sink, float-to-surface buoyancy
function collide(px, py, pz){
  const x0 = Math.floor(px - PR), x1 = Math.floor(px + PR);
  const y0 = Math.floor(py),       y1 = Math.floor(py + PH - 0.001);
  const z0 = Math.floor(pz - PR), z1 = Math.floor(pz + PR);
  for (let x = x0; x <= x1; x++) for (let y = y0; y <= y1; y++) for (let z = z0; z <= z1; z++)
    if (isOpaque(blockWorld(x, y, z))) return true;
  return false;
}

const uiBlocking = () => chatOpen || invOpen || tableOpen || signEditOpen || managerOpen;

// ---- chat command list (shown while typing a '/') ----
const COMMAND_HELP = [
  '/go to <x> <z> <y>  — teleport to a spot (or /go to @player)',
  '/time set <0-24000 | day | night | noon | midnight>',
  '/gamemode <creative | survival>  — (OP only)',
  '/give <item> [count]  — e.g. /give stone 10, /give pig egg',
  '/summon <mob>  — spawn pig, sheep, cow, chicken, fish or villager',
  '/kick <username>  — host: remove a player'
];
function updateChatHint(){
  const hint = el('chatHint'); if (!hint) return;
  if (chatOpen && chatInput[0] === '/'){
    hint.innerHTML = '<div class="hint-title">Commands</div>' + COMMAND_HELP.map(c => '<div>' + c + '</div>').join('');
    hint.classList.add('show');
  } else hint.classList.remove('show');
}

// ---------- SECTION 6d: mobs (passive animals + villagers) ----------
const mobs = [];
const MAX_MOBS = Infinity;          // no mob cap
const villagerSpawned = new Set();
let spawnTimer = 2;

function mPart(group, w, h, d, color, x, y, z){
  const m = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), new THREE.MeshLambertMaterial({ color }));
  m.position.set(x, y, z); m.castShadow = true; m.receiveShadow = true; group.add(m); return m;
}
function buildPig(){ const g = new THREE.Group();
  g.userData.body = mPart(g, 0.9, 0.6, 1.2, 0xe48aa0, 0, 0.55, 0);
  g.userData.head = mPart(g, 0.5, 0.5, 0.45, 0xe48aa0, 0, 0.66, 0.78);
  mPart(g, 0.24, 0.14, 0.16, 0xcf6f88, 0, 0.6, 1.02);
  const legs = []; for (const sx of [-0.28,0.28]) for (const sz of [-0.4,0.4]) legs.push(mPart(g, 0.2, 0.45, 0.2, 0x8a5566, sx, 0.22, sz));
  g.userData.legs = legs; return g; }
function buildSheep(){ const g = new THREE.Group();
  g.userData.body = mPart(g, 1.0, 0.85, 1.15, 0xeef0ee, 0, 0.7, 0);
  g.userData.head = mPart(g, 0.45, 0.5, 0.45, 0xd9c9b6, 0, 0.78, 0.72);
  const legs = []; for (const sx of [-0.32,0.32]) for (const sz of [-0.42,0.42]) legs.push(mPart(g, 0.2, 0.5, 0.2, 0x44372c, sx, 0.25, sz));
  g.userData.legs = legs; return g; }
function buildCow(){ const g = new THREE.Group();
  g.userData.body = mPart(g, 1.0, 0.85, 1.35, 0x5b4636, 0, 0.75, 0);
  mPart(g, 1.02, 0.4, 0.6, 0xe8e2d6, 0, 0.7, 0.2);
  g.userData.head = mPart(g, 0.5, 0.5, 0.5, 0x4c3a2c, 0, 0.85, 0.88);
  const legs = []; for (const sx of [-0.34,0.34]) for (const sz of [-0.48,0.48]) legs.push(mPart(g, 0.22, 0.55, 0.22, 0x3c2e22, sx, 0.27, sz));
  g.userData.legs = legs; return g; }
function buildChicken(){ const g = new THREE.Group();
  g.userData.body = mPart(g, 0.4, 0.42, 0.55, 0xf3f3f0, 0, 0.42, 0);
  g.userData.head = mPart(g, 0.3, 0.3, 0.3, 0xf3f3f0, 0, 0.66, 0.22);
  mPart(g, 0.12, 0.1, 0.14, 0xe2a32a, 0, 0.62, 0.4);
  mPart(g, 0.18, 0.16, 0.06, 0xd23b34, 0, 0.8, 0.2);
  const legs = []; for (const sx of [-0.12,0.12]) legs.push(mPart(g, 0.08, 0.32, 0.08, 0xe2a32a, sx, 0.16, 0));
  g.userData.legs = legs; return g; }
function buildFish(){ const g = new THREE.Group();
  g.userData.body = mPart(g, 0.28, 0.34, 0.7, 0x4f93b8, 0, 0, 0);
  mPart(g, 0.06, 0.3, 0.28, 0x3f7fa6, 0, 0, -0.42);
  g.userData.legs = null; return g; }
function buildVillager(){ const g = new THREE.Group();
  g.userData.body = mPart(g, 0.5, 0.9, 0.32, 0x6f5235, 0, 0.95, 0);
  g.userData.head = mPart(g, 0.5, 0.5, 0.5, 0xc89b73, 0, 1.6, 0);
  mPart(g, 0.14, 0.18, 0.2, 0xb07d52, 0, 1.55, 0.3);                 // big nose
  const arms = []; arms.push(mPart(g, 0.16, 0.7, 0.2, 0x5d4530, -0.33, 0.95, 0)); arms.push(mPart(g, 0.16, 0.7, 0.2, 0x5d4530, 0.33, 0.95, 0));
  const legs = []; for (const sx of [-0.13,0.13]) legs.push(mPart(g, 0.2, 0.55, 0.22, 0x3d3a44, sx, 0.27, 0));
  g.userData.legs = legs; return g; }
// Koko: a big fat pug, cow-sized — huge body, little head, stubby legs, a curled tail, name tag.
function buildKoko(){ const g = new THREE.Group();
  const FAWN = 0xd6b48a, LIGHT = 0xe6cda8, DARK = 0x3a2f28, NOSE = 0x1c150d;
  g.userData.body = mPart(g, 1.3, 1.0, 1.65, FAWN, 0, 0.82, 0);       // big fat body
  mPart(g, 1.32, 0.42, 1.05, LIGHT, 0, 0.5, 0.12);                    // pale belly
  g.userData.head = mPart(g, 0.52, 0.46, 0.46, FAWN, 0, 1.2, 0.98);   // little head
  mPart(g, 0.44, 0.36, 0.16, DARK, 0, 1.12, 1.2);                     // dark pug face
  mPart(g, 0.16, 0.13, 0.1, NOSE, 0, 1.1, 1.3);                       // squashed nose
  mPart(g, 0.16, 0.22, 0.06, DARK, -0.21, 1.4, 0.95);                 // floppy ears
  mPart(g, 0.16, 0.22, 0.06, DARK, 0.21, 1.4, 0.95);
  const legs = []; for (const sx of [-0.44,0.44]) for (const sz of [-0.58,0.58]) legs.push(mPart(g, 0.3, 0.34, 0.3, 0xc9a679, sx, 0.17, sz));  // little stubby legs
  g.userData.legs = legs;
  mPart(g, 0.16, 0.16, 0.16, FAWN, 0,    1.25, -0.88);               // curled tail
  mPart(g, 0.16, 0.16, 0.16, FAWN, 0.17, 1.34, -0.8);
  mPart(g, 0.16, 0.16, 0.16, FAWN, 0.07, 1.45, -0.66);
  const label = makeNameSprite('Koko'); label.position.set(0, 2.0, 0); g.add(label);   // "Koko" label
  return g; }

const MOB_DEF = {
  pig:     { hearts:4, drop:BACON,     speed:1.9, hw:0.45, h:0.95, land:true,  build:buildPig },
  sheep:   { hearts:4, drop:MUTTON,    speed:1.6, hw:0.5,  h:1.25, land:true,  build:buildSheep,    eats:true },
  cow:     { hearts:5, drop:BEEF,      speed:1.5, hw:0.5,  h:1.3,  land:true,  build:buildCow,      eats:true },
  chicken: { hearts:3, drop:CHICKEN_I, speed:1.8, hw:0.28, h:0.7,  land:true,  build:buildChicken },
  fish:    { hearts:2, drop:FISH_I,    speed:2.0, hw:0.3,  h:0.5,  land:false, build:buildFish },
  villager:{ hearts:7, drop:null,      speed:1.4, hw:0.32, h:1.85, land:true,  build:buildVillager, villager:true },
  koko:    { hearts:12, drop:BEEF,     speed:1.1, hw:0.62, h:1.35, land:true,  build:buildKoko },
};

function spawnMob(type, x, y, z, home){
  const def = MOB_DEF[type]; if (!def || mobs.length >= MAX_MOBS) return null;
  const group = def.build(); group.position.set(x, y, z); scene.add(group);
  const mob = { type, def, group, x, y, z, vy:0, yaw:Math.random()*6.28, pitch:0, onGround:false,
    health:def.hearts, flash:0, state:'idle', stateT:0.5+Math.random()*1.5, walkPhase:Math.random()*6,
    legs:group.userData.legs||null, head:group.userData.head||null, body:group.userData.body||null,
    home:home||null, parts:[] };
  group.traverse(o => { if (o.isMesh){ o.userData.mobRef = mob; mob.parts.push({ mat:o.material, base:o.material.color.getHex() }); } });
  mobs.push(mob); return mob;
}
function setMobColor(mob, red){ for (const p of mob.parts) p.mat.color.setHex(red ? 0xff3030 : p.base); }
function removeMob(mob){ scene.remove(mob.group); const i = mobs.indexOf(mob); if (i >= 0) mobs.splice(i, 1); }
function clearMobs(){ for (const m of mobs) scene.remove(m.group); mobs.length = 0; villagerSpawned.clear(); }
function heldTool(){ const s = hotbar[selIndex]; return s ? s.type : null; }
function isPickaxe(t){ return t === WOODEN_PICKAXE || t === STONE_PICKAXE || t === IRON_PICKAXE; }
function isAxe(t){ return t === WOODEN_AXE || t === STONE_AXE || t === IRON_AXE; }
function isSword(t){ return t === WOODEN_SWORD || t === STONE_SWORD || t === IRON_SWORD; }
function pickMul(t){ return t === IRON_PICKAXE ? 8 : t === STONE_PICKAXE ? 5 : 3; }   // higher tiers mine faster
function axeMul(t){ return t === IRON_AXE ? 6 : t === STONE_AXE ? 4 : 3; }
function swordDmg(t){ return t === IRON_SWORD ? 6 : t === STONE_SWORD ? 4 : t === WOODEN_SWORD ? 3 : 1; }   // hearts vs mobs
function swordPvp(t){ return t === IRON_SWORD ? 10 : t === STONE_SWORD ? 8 : t === WOODEN_SWORD ? 6 : PVP_DMG; }
// the right tool mines its block faster; sword hits harder (handled in damage fns)
function mineSpeedFor(type){
  const t = heldTool();
  const woody  = (type === WOOD || type === WOOD_PLANKS || type === CRAFTING_TABLE || type === JUNGLE_WOOD || type === SIGN);
  const stoney = (type === STONE || type === COAL_ORE || type === IRON_ORE || type === DIAMOND_ORE || type === SANDSTONE);
  if (isAxe(t) && woody) return axeMul(t);        // axe chops wood faster (by tier)
  if (isPickaxe(t) && stoney) return pickMul(t);  // pickaxe mines stone/ores faster (by tier)
  return 1;
}
function damageMob(mob){
  const dmg = swordDmg(heldTool());   // sword tier sets the hit power
  mob.health -= dmg; mob.flash = 0.3; setMobColor(mob, true);
  swingHand();                                          // first-person swing
  let dx = mob.x - benny.position.x, dz = mob.z - benny.position.z;   // knock the mob away from you
  const l = Math.hypot(dx, dz) || 1; mob.kbx = (dx / l) * 7; mob.kbz = (dz / l) * 7;
  if (mob.def && mob.def.land){ mob.vy = Math.max(mob.vy || 0, 4.2); mob.onGround = false; }
  if (mob.health <= 0){
    if (mob.def.drop != null) spawnDrop(mob.def.drop, Math.floor(mob.x), Math.floor(mob.y) + 1, Math.floor(mob.z));
    addXp(2);                                          // kills grant experience
    removeMob(mob);
  }
}
function raycastMob(){
  const list = []; for (const mob of mobs) mob.group.traverse(o => { if (o.isMesh) list.push(o); });
  if (!list.length) return null;
  raycaster.setFromCamera(_ndc, camera);
  const hits = raycaster.intersectObjects(list, false);
  if (!hits.length || !hits[0].object.userData.mobRef) return null;
  return { mob: hits[0].object.userData.mobRef, dist: hits[0].distance };
}
function raycastPlayer(){
  if (!net.active || !remotePlayers.size) return null;
  const list = []; const owner = new Map();
  remotePlayers.forEach((rp, id) => rp.group.traverse(o => { if (o.isMesh){ list.push(o); owner.set(o, id); } }));
  if (!list.length) return null;
  raycaster.setFromCamera(_ndc, camera);
  const hits = raycaster.intersectObjects(list, false);
  if (!hits.length) return null;
  return { id: owner.get(hits[0].object), dist: hits[0].distance };
}
function mobSolid(px, py, pz, def){
  const hw = def.hw, h = def.h;
  const x0 = Math.floor(px-hw), x1 = Math.floor(px+hw), y0 = Math.floor(py), y1 = Math.floor(py+h-0.01), z0 = Math.floor(pz-hw), z1 = Math.floor(pz+hw);
  for (let x = x0; x <= x1; x++) for (let y = y0; y <= y1; y++) for (let z = z0; z <= z1; z++) if (isOpaque(blockWorld(x, y, z))) return true;
  return false;
}
function tryMoveMob(mob, dx, dz, def){
  let moved = false;
  if (!mobSolid(mob.x+dx, mob.y, mob.z, def)){ mob.x += dx; moved = true; }
  else if (mob.onGround && !mobSolid(mob.x+dx, mob.y+1, mob.z, def)){ mob.x += dx; mob.y += 1; moved = true; }  // step up
  if (!mobSolid(mob.x, mob.y, mob.z+dz, def)){ mob.z += dz; moved = true; }
  else if (mob.onGround && !mobSolid(mob.x, mob.y+1, mob.z+dz, def)){ mob.z += dz; mob.y += 1; moved = true; }
  return moved;
}
function animLegs(mob, moving){
  if (!mob.legs) return;
  const a = moving ? Math.sin(mob.walkPhase) * 0.6 : mob.legs[0].rotation.x * 0.8;
  mob.legs.forEach((lg, i) => { lg.rotation.x = a * ((i % 2) ? 1 : -1); });
}
function updateMob(mob, dt){
  const def = mob.def;
  if (mob.flash > 0){ mob.flash -= dt; if (mob.flash <= 0) setMobColor(mob, false); }
  if (def.land){
    mob.stateT -= dt;
    if (mob.stateT <= 0){
      const r = Math.random();
      if (def.eats && r < 0.22){ mob.state = 'eat'; mob.stateT = 2 + Math.random()*2; }
      else if (r < 0.5){ mob.state = 'idle'; mob.stateT = 1 + Math.random()*2; }
      else { mob.state = 'walk'; mob.yaw = Math.random()*6.283; mob.stateT = 2 + Math.random()*3; }
    }
    // villagers keep close to home
    if (def.villager && mob.home){
      const dxh = mob.x - mob.home.x, dzh = mob.z - mob.home.z;
      if (dxh*dxh + dzh*dzh > 42*42){ mob.yaw = Math.atan2(mob.home.x - mob.x, mob.home.z - mob.z); mob.state = 'walk'; mob.stateT = 2.5; }
    }
    let moving = false;
    if (mob.state === 'walk'){
      const sp = def.speed * dt;
      if (tryMoveMob(mob, Math.sin(mob.yaw)*sp, Math.cos(mob.yaw)*sp, def)) moving = true;
      else mob.yaw = Math.random()*6.283;
    }
    if (mob.kbx || mob.kbz){                              // ride out the knockback impulse
      tryMoveMob(mob, mob.kbx * dt, mob.kbz * dt, def);
      const decay = Math.pow(0.0008, dt); mob.kbx *= decay; mob.kbz *= decay;
      if (Math.abs(mob.kbx) < 0.3) mob.kbx = 0;
      if (Math.abs(mob.kbz) < 0.3) mob.kbz = 0;
      moving = true;
    }
    mob.vy -= GRAVITY * dt;
    const ny = mob.y + mob.vy * dt;
    if (!mobSolid(mob.x, ny, mob.z, def)){ mob.y = ny; mob.onGround = false; }
    else { if (mob.vy < 0) mob.onGround = true; mob.vy = 0; }
    if (mob.y < -40){ removeMob(mob); return; }
    if (moving) mob.walkPhase += dt * 9;
    animLegs(mob, moving);
    if (mob.head) mob.head.rotation.x = (mob.state === 'eat') ? 0.6 + Math.sin(performance.now()/120)*0.15 : 0;   // grazing animation
    mob.group.rotation.y = mob.yaw;
  } else {                                            // fish: free swimming inside water
    mob.stateT -= dt;
    if (mob.stateT <= 0){ mob.yaw = Math.random()*6.283; mob.pitch = (Math.random()-0.5)*0.7; mob.stateT = 1 + Math.random()*2; }
    const sp = def.speed * dt;
    const nx = mob.x + Math.sin(mob.yaw)*sp, nz = mob.z + Math.cos(mob.yaw)*sp, ny = mob.y + Math.sin(mob.pitch)*sp*0.6;
    if (blockWorld(Math.floor(nx), Math.floor(mob.y), Math.floor(nz)) === WATER){ mob.x = nx; mob.z = nz; } else mob.yaw += 1.57;
    if (blockWorld(Math.floor(mob.x), Math.floor(ny), Math.floor(mob.z)) === WATER){ mob.y = ny; } else mob.pitch = -mob.pitch;
    mob.walkPhase += dt * 6; if (mob.body) mob.body.rotation.y = Math.sin(mob.walkPhase) * 0.3;
    mob.group.rotation.y = mob.yaw;
  }
  mob.group.position.set(mob.x, mob.y, mob.z);
}
function updateMobs(dt){
  const reach = renderChunks * CHUNK + 40;
  for (let i = mobs.length - 1; i >= 0; i--){
    const mob = mobs[i];
    const dx = mob.x - benny.position.x, dz = mob.z - benny.position.z;
    if (dx*dx + dz*dz > reach*reach){ removeMob(mob); continue; }   // outside the loaded world
    updateMob(mob, dt);
  }
}
function columnTop(wx, wz){                            // highest solid block top, or null if unloaded/none
  if (!chunkLoaded(wx, wz)) return null;
  for (let y = Math.min(WORLD_H - 1, 110); y >= 1; y--) if (isOpaque(blockWorld(wx, y, wz))) return y + 1;
  return null;
}
function updateSpawns(dt){
  if (worldMode === 'extreme' || worldMode === 'kokocity' || worldMode === 'pvphub') return;   // no mobs in built worlds
  spawnTimer -= dt;
  if (spawnTimer <= 0){
    spawnTimer = 1.5;
    for (let attempt = 0; attempt < 2; attempt++){
    if (mobs.length < MAX_MOBS){
      const ang = Math.random()*6.283, dist = 20 + Math.random() * Math.min(70, renderChunks*CHUNK*0.5);
      const wx = Math.floor(benny.position.x + Math.cos(ang)*dist), wz = Math.floor(benny.position.z + Math.sin(ang)*dist);
      const sY = heightAt(wx, wz), b = biomeName(wx, wz, sY);
      if (b === 'Ocean'){
        if (chunkLoaded(wx, wz) && blockWorld(wx, WATER_Y-2, wz) === WATER){ for (let j = 0; j < 2; j++) spawnMob('fish', wx+0.5+(Math.random()-0.5), WATER_Y-2, wz+0.5+(Math.random()-0.5)); }
      } else if (sY > WATER_Y){
        const top = columnTop(wx, wz);
        if (top != null){
          const kinds = (worldMode === 'koko') ? ['koko'] : ['pig','sheep','cow','chicken'];
          const k = kinds[(Math.random()*kinds.length)|0];
          const n = 2 + ((Math.random()*3)|0);                 // herds of 2..4
          for (let j = 0; j < n; j++) spawnMob(k, wx+0.5+(Math.random()-0.5), top, wz+0.5+(Math.random()-0.5));
        }
      }
    }
    }
    // villagers when the player is near an un-populated village
    const rx = floorDiv(Math.floor(benny.position.x), VREGION), rz = floorDiv(Math.floor(benny.position.z), VREGION);
    const v = isFlatMode() ? null : villageCenter(rx, rz);
    if (v){
      const ddx = benny.position.x - v.wx, ddz = benny.position.z - v.wz;
      const key = rx + ',' + rz;
      if (ddx*ddx + ddz*ddz < (VRADIUS+30)*(VRADIUS+30) && !villagerSpawned.has(key)){
        villagerSpawned.add(key);
        const n = 8 + ((Math.random()*5)|0);                 // 8..12 villagers
        for (let j = 0; j < n; j++){
          const ox = (Math.random()-0.5)*40, oz = (Math.random()-0.5)*40;
          spawnMob('villager', v.wx+ox, v.level+1, v.wz+oz, { x:v.wx, z:v.wz });
        }
      }
    }
  }
}

// ---------- SECTION 7: the loop ----------
const clock = new THREE.Clock();

// a wireframe box showing exactly where a held block will be placed
const placePreview = new THREE.LineSegments(
  new THREE.EdgesGeometry(new THREE.BoxGeometry(1.004, 1.004, 1.004)),
  new THREE.LineBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.55 }));
placePreview.visible = false; placePreview.frustumCulled = false; scene.add(placePreview);
// a black outline around whatever block you're looking at (always shown, even while holding a block)
const blockHighlight = new THREE.LineSegments(
  new THREE.EdgesGeometry(new THREE.BoxGeometry(1.012, 1.012, 1.012)),
  new THREE.LineBasicMaterial({ color: 0x000000, transparent: true, opacity: 0.85 }));
blockHighlight.visible = false; blockHighlight.frustumCulled = false; scene.add(blockHighlight);
function updatePlacePreview(){
  let show = false, hl = false;
  if (playing && !paused && !dead && !uiBlocking() && document.pointerLockElement === canvas){
    const tgt = raycastBlock();
    if (tgt){
      blockHighlight.position.set(tgt.wx + 0.5, tgt.wy + 0.5, tgt.wz + 0.5); hl = true;   // outline the looked-at block
      const s = selectedStack();
      if (s && isBlockType(s.type)){
        const px = tgt.wx + tgt.normal.x, py = tgt.wy + tgt.normal.y, pz = tgt.wz + tgt.normal.z;
        const cur = blockWorld(px, py, pz);
        if (py >= 0 && py < WORLD_H && (cur === AIR || cur === WATER) && !blockOverlapsPlayer(px, py, pz)){
          placePreview.position.set(px + 0.5, py + 0.5, pz + 0.5); show = true;
        }
      }
    }
  }
  blockHighlight.visible = hl;
  placePreview.visible = show;
}

function update(dt){
  if (paused && !net.active) return;             // solo: the pause menu freezes the world. online: keep it live.
  let moving = false;
  updateVitals(dt);

  if (!paused && !uiBlocking() && !dead){
    const forwardX = Math.sin(yaw), forwardZ = Math.cos(yaw);
    const rightX   = -Math.cos(yaw), rightZ  = Math.sin(yaw);
    let f = 0, r = 0;
    if (keys[keybinds.forward]) f += 1;
    if (keys[keybinds.back])    f -= 1;
    if (keys[keybinds.right])   r += 1;
    if (keys[keybinds.left])    r -= 1;
    moving = !!(f || r);
    const bcx = floorDiv(Math.floor(benny.position.x), CHUNK), bcz = floorDiv(Math.floor(benny.position.z), CHUNK);

    if (gamemode === 'survival'){
      const fx = Math.floor(benny.position.x), fz = Math.floor(benny.position.z);
      const headWater = blockWorld(fx, Math.floor(benny.position.y + 0.9), fz) === WATER;
      const feetWater = blockWorld(fx, Math.floor(benny.position.y + 0.1), fz) === WATER;
      const submerged = headWater || feetWater;          // in water if either feet or head are under
      const sprinting = (keys.ControlLeft || keys.ControlRight) && f > 0 && !submerged;   // Ctrl + forward
      const sp = (submerged ? WALK * 0.7 : (sprinting ? WALK * 1.6 : WALK)) * dt;
      let dx = 0, dz = 0;
      if (f || r){ const l = Math.hypot(f, r); f /= l; r /= l; dx = (forwardX*f + rightX*r) * sp; dz = (forwardZ*f + rightZ*r) * sp; benny.rotation.y = Math.atan2(dx, dz); }
      const movedX = dx && !collide(benny.position.x + dx, benny.position.y, benny.position.z);
      const movedZ = dz && !collide(benny.position.x, benny.position.y, benny.position.z + dz);
      if (movedX) benny.position.x += dx;
      if (movedZ) benny.position.z += dz;
      if (sprinting && (dx || dz) && onGround) sprintDrain(dt);     // sprinting burns hunger
      if (submerged){                                        // swimming: float to the surface, hold Space to rise
        if (keys[keybinds.up]) vy += SWIM_UP * dt;           // paddle upward
        else if (headWater) vy += SWIM_BUOY * dt;            // buoyancy lifts you toward the surface
        else vy -= SWIM_SINK * dt;                           // at the surface, settle gently
        // climb-out assist: swimming into a shore lifts you up onto it
        if ((dx || dz) && (!movedX || !movedZ)) vy = Math.max(vy, 5.5);
        vy *= 0.86;                                           // water drag
        vy = Math.max(-1.2, Math.min(5.5, vy));
        onGround = false;
      } else {
        vy -= GRAVITY * dt;                                   // gravity in air
      }
      const ny = benny.position.y + vy * dt;
      if (!collide(benny.position.x, ny, benny.position.z)){ benny.position.y = ny; if (!submerged) onGround = false; }
      else { if (vy < 0) onGround = true; vy = 0; }
      // fall-damage bookkeeping: remember the highest point while airborne, hurt on landing
      if (submerged) fallPeakY = null;                        // water breaks your fall
      else if (!onGround) fallPeakY = (fallPeakY === null) ? benny.position.y : Math.max(fallPeakY, benny.position.y);
      else if (fallPeakY !== null){
        const fall = fallPeakY - benny.position.y;
        if (fall > 2 && !ruleNoFall()){ const dmg = Math.round((fall - 2) * 2); if (dmg > 0) damagePlayer(dmg, 'fall'); }  // ~1 heart per block past 2
        fallPeakY = null;
      }
      if (!submerged && keys[keybinds.up] && onGround){ vy = JUMP_V; onGround = false; onJump(); }   // Space = jump on land
      if (benny.position.y < -40){ benny.position.y = 90; vy = 0; fallPeakY = null; damagePlayer(maxHealth, 'void'); }   // fell out of the world
    } else {
      const sprintFly = (keys.ControlLeft || keys.ControlRight) ? 1.8 : 1;
      const sp = FLY_SPEED * sprintFly * dt;                  // creative free flight (Ctrl = faster)
      if (f || r){ const l = Math.hypot(f, r); f /= l; r /= l;
        benny.position.x += (forwardX*f + rightX*r) * sp;
        benny.position.z += (forwardZ*f + rightZ*r) * sp;
        benny.rotation.y = Math.atan2((forwardX*f + rightX*r), (forwardZ*f + rightZ*r));
      }
      let vyMove = 0;
      if (keys[keybinds.up])   vyMove += 1;
      if (keys[keybinds.down]) vyMove -= 1;
      benny.position.y = Math.max(0.5, Math.min(WORLD_H - 0.5, benny.position.y + vyMove * sp));
    }

    // knockback from being hit — slides you back, decays quickly, respects walls
    if (kbx || kbz){
      const kdx = kbx * dt, kdz = kbz * dt;
      if (kdx && !collide(benny.position.x + kdx, benny.position.y, benny.position.z)) benny.position.x += kdx;
      if (kdz && !collide(benny.position.x, benny.position.y, benny.position.z + kdz)) benny.position.z += kdz;
      const decay = Math.pow(0.0008, dt);
      kbx *= decay; kbz *= decay;
      if (Math.abs(kbx) < 0.3) kbx = 0;
      if (Math.abs(kbz) < 0.3) kbz = 0;
    }
    if (world.finite){
      const lim = world.half - 0.5;
      benny.position.x = Math.max(-lim, Math.min(lim, benny.position.x));
      benny.position.z = Math.max(-lim, Math.min(lim, benny.position.z));
    }
    if (floorDiv(Math.floor(benny.position.x), CHUNK) !== bcx ||
        floorDiv(Math.floor(benny.position.z), CHUNK) !== bcz) chunksDirty = true;
  }

  // survival mining: hold right mouse to wear a block down
  if (gamemode === 'survival' && mineHeld && !ruleNoBuild() && !uiBlocking() && document.pointerLockElement === canvas){
    const tgt = raycastBlock();
    if (tgt){
      const key = tgt.wx + ',' + tgt.wy + ',' + tgt.wz;
      if (key !== mineKey){ mineKey = key; mineProgress = 0; }
      swingHand();                                        // keep swinging while mining
      mineProgress += dt * mineSpeedFor(tgt.type) / (HARDNESS[tgt.type] || 1);   // right tool mines faster
      showCrack(tgt, mineProgress);
      if (mineProgress >= 1){ doBreak(tgt); mineKey = null; mineProgress = 0; }
    } else { mineKey = null; mineProgress = 0; hideCrack(); }
  }

  updateDrops(dt);
  updateMobs(dt);
  updateSpawns(dt);
  updateWaterFlow();                                    // spread placed/disturbed water
  if (worldMode === 'extreme' && playing && !dead) parkourTick();
  autosaveT -= dt; if (autosaveT <= 0){ autosaveT = 20; saveGame(); }   // periodic autosave keeps items/position safe
  if (net.active && net.isHost && settings.openWorld){ lobbyBeatT -= dt; if (lobbyBeatT <= 0){ lobbyBeatT = 12; lobbyAdvertise(); lobbyUpdate(); } }  // keep our open world in the directory
  el('statBars').classList.toggle('show', playing && !paused && !dead && gamemode === 'survival' && !uiBlocking());
  el('xpWrap').classList.toggle('show', playing && !paused && !dead && !uiBlocking());
  updatePlacePreview();
  if (net.active){
    netAcc += dt; if (netAcc >= 0.05){ netAcc = 0; sendState(); }         // ~20 transform updates/sec
    pingAcc += dt; if (pingAcc >= 2){ pingAcc = 0; netPing(); }            // round-trip ping every 2s
    if (net.isHost){ timeAcc += dt; if (timeAcc >= 5){ timeAcc = 0; broadcast({ t:'time', time: timeOfDay }); } }  // host owns the clock
    updateRemotePlayers(dt);
  }

  // camera — first person (default) or third person, toggled with F5
  const eyeY = benny.position.y + BENNY_EYE;
  if (firstPerson){
    benny.visible = false;                 // don't let his own body block the view
    camera.position.set(benny.position.x, eyeY, benny.position.z);
    const lx = Math.sin(yaw) * Math.cos(pitch);
    const ly = Math.sin(pitch);
    const lz = Math.cos(yaw) * Math.cos(pitch);
    camera.lookAt(benny.position.x + lx, eyeY + ly, benny.position.z + lz);
  } else {
    benny.visible = true;                  // third-person: orbit behind Benny
    const dist = 7;
    const cx = Math.sin(yaw) * Math.cos(pitch) * dist;
    const cz = Math.cos(yaw) * Math.cos(pitch) * dist;
    const cy = Math.sin(-pitch) * dist;
    camera.position.set(benny.position.x - cx, eyeY + cy, benny.position.z - cz);
    camera.lookAt(benny.position.x, eyeY, benny.position.z);
  }

  // offhand viewmodel: only in first person, with a subtle bob while moving
  offhand.visible = firstPerson;
  if (firstPerson){
    updateHeldViewmodel();                              // keep the in-hand item in sync with the hotbar
    if (moving) bobT += dt * 9; else bobT *= 0.9;       // settle when idle
    let swing = 0;
    if (swingT > 0){ swingT -= dt; swing = Math.sin((1 - Math.max(0, swingT) / SWING_DUR) * Math.PI); }  // 0→1→0
    offhand.rotation.set(0.05 - swing * 1.15, -swing * 0.25, 0.14);    // swing down & across
    offhand.position.set(
      OFFHAND_BASE.x + Math.cos(bobT * 0.5) * 0.012 - swing * 0.04,
      OFFHAND_BASE.y + Math.sin(bobT) * 0.018 - swing * 0.10,
      OFFHAND_BASE.z + swing * 0.16
    );
  }

  // tint the view only when the camera is actually inside a water block
  const inWater = blockWorld(Math.floor(camera.position.x), Math.floor(camera.position.y), Math.floor(camera.position.z)) === WATER;
  underwaterEl.style.opacity = inWater ? '1' : '0';

  // advance sun/moon, sky colour and lighting for this frame
  updateSky(dt);

  // F3 debug readouts (only worth updating while the panel is visible)
  if (showDebug){
    el('dbgPos').textContent    = Math.floor(benny.position.x) + ', ' + Math.floor(benny.position.y) + ', ' + Math.floor(benny.position.z);
    el('dbgBiome').textContent  = (blockWorld(Math.floor(camera.position.x), Math.floor(camera.position.y), Math.floor(camera.position.z)) === WATER) ? 'Underwater' : biomeAt(benny.position.x, benny.position.z);
    el('dbgDay').textContent    = String(dayCount);
    el('dbgChunks').textContent = chunks.size + ' loaded';
    _projScreen.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
    _frustum.setFromProjectionMatrix(_projScreen);
    let vis = 0, totalB = 0, shownB = 0;
    chunks.forEach(ch => {
      if (!ch.opaque || !ch.opaque.length) return;
      let c = 0; for (const m of ch.opaque) c += m.count;
      totalB += c;
      const ox = ch.cx*CHUNK, oz = ch.cz*CHUNK;
      _box3.min.set(ox, 0, oz); _box3.max.set(ox + CHUNK, ch.maxY || WORLD_H, oz + CHUNK);
      if (_frustum.intersectsBox(_box3)) shownB += c;
    });
    el('dbgBlocks').textContent = shownB.toLocaleString() + ' shown / ' + totalB.toLocaleString();
    for (const m of mobs){ _fvec.set(m.x, m.y + 0.5, m.z); if (_frustum.containsPoint(_fvec)) vis++; }
    for (const d of drops){ _fvec.set(d.x, d.y, d.z); if (_frustum.containsPoint(_fvec)) vis++; }
    el('dbgEntities').textContent = vis + ' shown / ' + (mobs.length + drops.length) + ' total';
    el('dbgView').textContent   = firstPerson ? 'First person' : 'Third person';
    el('dbgMode').textContent   = gamemodeLabel() + (settings.op ? ' · OP' : '');
  }
}

let fpsTimer = 0, fpsFrames = 0;
function updateFps(dt){
  fpsTimer += dt; fpsFrames++;
  if (fpsTimer >= 0.5){ fpsEl.textContent = Math.round(fpsFrames / fpsTimer) + ' fps'; fpsTimer = 0; fpsFrames = 0; }
}

// --- loading screen (drains the chunk build queue before play) ---
const loadingEl = document.getElementById('loading');
const loadFill  = document.getElementById('loadFill');
const loadPct   = document.getElementById('loadPct');
let loading = false, loadTarget = 1, afterLoad = null;
function beginLoad(after){
  afterLoad = after || null;
  loadTarget = Math.max(1, buildQueue.length);
  loading = true;
  hideAllMenus(); loadingEl.classList.add('show');
  updateLoadBar();
}
function updateLoadBar(){
  const pct = Math.max(0, Math.min(100, Math.round((loadTarget - buildQueue.length) / loadTarget * 100)));
  loadFill.style.width = pct + '%'; loadPct.textContent = pct + '%';
}
function finishLoad(){
  loading = false; loadingEl.classList.remove('show');
  const f = afterLoad; afterLoad = null; if (f) f();
}

function animate(){
  requestAnimationFrame(animate);
  const dt = Math.min(clock.getDelta(), 0.05);

  if (loading){
    processChunkQueue(8);                       // crunch through generation
    updateLoadBar();
    if (!buildQueue.length) finishLoad();
  } else if (playing){
    if (chunksDirty) refreshChunkQueue();       // player crossed a chunk / changed render distance
    processChunkQueue(2);                        // stream a couple of fresh chunks per frame
    flushDirtyMeshes(2);                          // spread out edit re-meshes (no break stutter)
    frameTick++;
    if (settings.lowDetail && (frameTick % 12 === 0)) applyDetail();   // LOD pass only matters in performance mode
    update(dt);                                  // update() freezes local control while paused (but keeps online worlds live)
  } else {                                       // title screens: live orbiting backdrop
    if (chunksDirty) refreshChunkQueue();
    processChunkQueue(4);
    orbitCamera(dt);
    updateSky(dt);
  }
  if (settings.fps) updateFps(dt);
  waterWave.value += dt;                          // animate ocean waves
  if (settings.waterFlow && WATER_TEX){ WATER_TEX.offset.y = (WATER_TEX.offset.y + dt * 0.06) % 1; WATER_TEX.offset.x = (WATER_TEX.offset.x + dt * 0.018) % 1; }
  if (playing && settings.reflections && (frameTick % 16 === 0)){   // refresh the reflection occasionally
    setAllChunksVisible(true);                                       // reflect the whole scene, not just the main view
    const wasBenny = benny.visible; benny.visible = true;            // include the player in the reflection
    reflectCam.position.set(camera.position.x, WATER_Y + 0.3, camera.position.z);
    reflectCam.update(renderer, scene);
    benny.visible = wasBenny;
  }
  if (playing || orbiting) updateClouds(dt);     // drifting clouds overhead
  if (playing) cullChunks();                     // draw only the chunks actually inside the camera's view
  renderer.render(scene, camera);               // keep drawing so menus/loading sit over the scene
}

// ---------- SECTION 8: wiring + boot ----------
addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
addEventListener('beforeunload', () => { if (playing) saveGame(); });   // don't lose progress on tab close

// boot: load saved settings, build the live title backdrop, and show the home screen.
// ---------- SECTION 9: multiplayer (WebRTC via PeerJS) ----------
// Host "opens to LAN" with a password -> a peer id is derived from it. Others "join world"
// with the same password to connect peer-to-peer. The host is authoritative for the world.
const net = { peer: null, conns: new Map(), hostConn: null, isHost: false, active: false, password: '' };
const remotePlayers = new Map();
let netAcc = 0;
let pingAcc = 0;
let timeAcc = 0;

function peerIdFor(pw){
  let h = 5381; const s = 'kokocraft|' + pw;
  for (let i = 0; i < s.length; i++) h = ((h << 5) + h + s.charCodeAt(i)) >>> 0;
  return 'kokocraft' + h.toString(36);
}
function hostStatus(s){ const e = el('lanHostStatus'); if (e) e.textContent = s; }
function joinStatus(s){ const e = el('joinStatus'); if (e) e.textContent = s; }

// ---------- player skins (per-part 16x16 textures you can paint) ----------
const SKIN_PARTS = ['head', 'body', 'arms', 'legs'];
const SKIN_FACES = ['front', 'back', 'right', 'left', 'top', 'bottom'];   // each cuboid part has 6 paintable faces
const FACE_ORDER = ['right', 'left', 'top', 'bottom', 'front', 'back'];   // BoxGeometry material index order (+X,-X,+Y,-Y,+Z,-Z)
function makePartCanvas(paint){ const c = document.createElement('canvas'); c.width = c.height = 16; const g = c.getContext('2d'); g.imageSmoothingEnabled = false; if (paint) paint(g); return c; }
function defaultSkin(){
  // each part = 6 face canvases; the front face gets the detail, the rest a base colour
  const facesFor = (base, frontPaint, topPaint) => {
    const mk = (paint) => makePartCanvas(g => { g.fillStyle = base; g.fillRect(0,0,16,16); if (paint) paint(g); });
    return { front: mk(frontPaint), back: mk(), right: mk(), left: mk(), top: mk(topPaint), bottom: mk() };
  };
  return {
    head: facesFor('#c89b73',
      g => { g.fillStyle='#2a1d12'; g.fillRect(4,7,2,2); g.fillRect(10,7,2,2); g.fillStyle='#a06a44'; g.fillRect(6,11,4,1); },   // eyes + mouth (front)
      g => { g.fillStyle='#6a4a2c'; g.fillRect(0,0,16,16); }),                                                                   // hair (top)
    body: facesFor('#3b7a8a', g => { g.fillStyle='#2f6270'; g.fillRect(0,12,16,4); }, null),
    arms: facesFor('#c89b73', g => { g.fillStyle='#3b7a8a'; g.fillRect(0,0,16,5); }, null),
    legs: facesFor('#39406b', g => { g.fillStyle='#2a2f50'; g.fillRect(0,12,16,4); }, null),
  };
}
const DEFAULT_SKIN = defaultSkin();
let mySkin = defaultSkin();
function serializeSkin(skin){
  const o = {};
  SKIN_PARTS.forEach(p => { o[p] = {}; SKIN_FACES.forEach(f => { try { o[p][f] = skin[p][f].toDataURL(); } catch (e){ o[p][f] = null; } }); });
  return o;
}
function skinTex(src){
  if (!src) src = DEFAULT_SKIN.head.front;
  if (typeof src === 'string'){ const img = new Image(); const t = new THREE.Texture(img); t.magFilter = THREE.NearestFilter; t.minFilter = THREE.NearestFilter; img.onload = () => { t.needsUpdate = true; }; img.src = src; return t; }
  const t = new THREE.CanvasTexture(src); t.magFilter = THREE.NearestFilter; t.minFilter = THREE.NearestFilter; return t;
}
// a cuboid part textured per-face (6 materials, BoxGeometry order)
function mPartFaces(group, w, h, d, faces, x, y, z){
  faces = faces || DEFAULT_SKIN.head;
  const mats = FACE_ORDER.map(f => new THREE.MeshLambertMaterial({ map: skinTex(faces[f] || faces.front) }));
  const m = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), mats);
  m.position.set(x, y, z); m.castShadow = true; m.receiveShadow = true; group.add(m); return m;
}
const remoteSkins = new Map();           // id -> serialized skin ({part:{face:dataURL}})
function setRemoteSkin(id, data){ if (!data) return; remoteSkins.set(id, data); if (remotePlayers.has(id)) rebuildAvatar(id); }
function rebuildAvatar(id){
  const rp = remotePlayers.get(id); if (!rp) return;
  scene.remove(rp.group);
  rp.group = buildAvatar(remoteSkins.get(id) || DEFAULT_SKIN);
  rp.group.position.set(rp.x, rp.y, rp.z); rp.group.rotation.y = rp.yaw;
  if (netNames.has(id)){ rp.label = makeNameSprite(netNames.get(id)); rp.group.add(rp.label); }
  applyCape(rp);
  scene.add(rp.group);
}
// draw a little front-facing paper-doll of a skin into a target canvas (uses each part's FRONT face)
function renderSkinPreview(target, skin){
  if (!target) return; const g = target.getContext('2d'); g.imageSmoothingEnabled = false;
  g.clearRect(0, 0, target.width, target.height);
  const S = Math.floor(target.width / 16);                 // pixel scale
  const fc = (part) => (skin[part] && skin[part].front) || skin[part];   // tolerate old single-canvas skins
  const blit = (src, cx, cy, w, h) => { try { g.drawImage(src, cx*S, cy*S, w*S, h*S); } catch (e){} };
  blit(fc('head'), 4, 0, 8, 8);                            // head
  blit(fc('body'), 4, 8, 8, 8);                            // torso
  blit(fc('arms'), 1, 8, 3, 8); blit(fc('arms'), 12, 8, 3, 8); // arms
  blit(fc('legs'), 4, 16, 4, 7); blit(fc('legs'), 8, 16, 4, 7); // legs
}

function buildAvatar(skinSrc){
  skinSrc = skinSrc || DEFAULT_SKIN;
  const g = new THREE.Group();
  g.userData.head = mPartFaces(g, 0.5, 0.5, 0.5, skinSrc.head, 0, 1.6, 0);
  mPartFaces(g, 0.5, 0.6, 0.28, skinSrc.body, 0, 1.05, 0);
  g.userData.arms = [mPartFaces(g, 0.16, 0.6, 0.22, skinSrc.arms, -0.33, 1.05, 0), mPartFaces(g, 0.16, 0.6, 0.22, skinSrc.arms, 0.33, 1.05, 0)];
  g.userData.legs = [mPartFaces(g, 0.2, 0.7, 0.24, skinSrc.legs, -0.13, 0.35, 0), mPartFaces(g, 0.2, 0.7, 0.24, skinSrc.legs, 0.13, 0.35, 0)];
  return g;
}
// ---- skin editor UI ----
const EDIT_PALETTE = ['#c89b73','#a06a44','#6a4a2c','#2a1d12','#ffffff','#cccccc','#888888','#222222',
  '#3b7a8a','#39406b','#7a3b3b','#c75d5d','#d8a34a','#4caf6a','#7a4fb0','#000000'];
const FACE_LABEL = { front:'Front', back:'Back', right:'Right', left:'Left', top:'Top', bottom:'Bottom' };
let editState = { part:'head', face:'front', color:'#222222' };
let editPainting = false, editSnapshot = null;
function buildEditorTabs(){
  const t = el('editorTabs'); t.innerHTML = '';
  SKIN_PARTS.forEach(p => {
    const b = document.createElement('button'); b.textContent = p; b.classList.toggle('active', p === editState.part);
    b.addEventListener('click', () => { editState.part = p; buildEditorTabs(); renderEditorGrid(); });
    t.appendChild(b);
  });
  buildEditorFaceTabs();
}
function buildEditorFaceTabs(){
  const t = el('editorFaceTabs'); if (!t) return; t.innerHTML = '';
  SKIN_FACES.forEach(f => {
    const b = document.createElement('button'); b.textContent = FACE_LABEL[f]; b.classList.toggle('active', f === editState.face);
    b.addEventListener('click', () => { editState.face = f; buildEditorFaceTabs(); renderEditorGrid(); });
    t.appendChild(b);
  });
}
function buildEditorPalette(){
  const wrap = el('editorPalette'); wrap.innerHTML = '';
  EDIT_PALETTE.forEach(col => {
    const b = document.createElement('button'); b.style.background = col; b.classList.toggle('active', col.toLowerCase() === editState.color.toLowerCase());
    b.addEventListener('click', () => { editState.color = col; el('editorColor').value = col; buildEditorPalette(); });
    wrap.appendChild(b);
  });
}
function editCanvas(){ const part = mySkin[editState.part]; return (part && part[editState.face]) ? part[editState.face] : (part.front || part); }
function renderEditorGrid(){
  const grid = el('editorGrid'); grid.innerHTML = '';
  const ctx = editCanvas().getContext('2d');
  let data; try { data = ctx.getImageData(0, 0, 16, 16).data; } catch (e){ data = null; }
  for (let y = 0; y < 16; y++) for (let x = 0; x < 16; x++){
    const cell = document.createElement('div');
    if (data){ const i = (y*16 + x) * 4; cell.style.background = 'rgba(' + data[i] + ',' + data[i+1] + ',' + data[i+2] + ',' + (data[i+3]/255) + ')'; }
    cell.addEventListener('pointerdown', e => { e.preventDefault(); editPainting = true; paintAt(x, y, cell); });
    cell.addEventListener('pointerenter', () => { if (editPainting) paintAt(x, y, cell); });
    grid.appendChild(cell);
  }
}
function paintAt(x, y, cell){
  const ctx = editCanvas().getContext('2d');
  ctx.fillStyle = editState.color; ctx.fillRect(x, y, 1, 1);
  if (cell) cell.style.background = editState.color;
  renderSkinPreview(el('editorPreview'), mySkin);
}
function applySkinData(skin, data, cb){            // draw saved dataURLs back onto a skin's canvases (handles old single-image format)
  const jobs = [];
  SKIN_PARTS.forEach(p => {
    const pd = data && data[p]; if (!pd) return;
    if (typeof pd === 'string') SKIN_FACES.forEach(f => jobs.push([p, f, pd]));       // old format: one image for every face
    else SKIN_FACES.forEach(f => { if (pd[f]) jobs.push([p, f, pd[f]]); });
  });
  if (!jobs.length){ if (cb) cb(); return; }
  let n = 0; const done = () => { if (++n >= jobs.length && cb) cb(); };
  jobs.forEach(([p, f, url]) => {
    const img = new Image();
    img.onload = () => { const g = skin[p][f].getContext('2d'); g.clearRect(0, 0, 16, 16); g.drawImage(img, 0, 0, 16, 16); done(); };
    img.onerror = done; img.src = url;
  });
}
function openEditor(){
  editSnapshot = serializeSkin(mySkin);            // remember for Cancel
  editState.part = 'head'; editState.face = 'front';
  buildEditorTabs(); buildEditorPalette(); renderEditorGrid();
  renderSkinPreview(el('editorPreview'), mySkin);
  showScreen('editorMenu');
}
function saveEditor(){
  try { localStorage.setItem(acctKey('skin'), JSON.stringify(serializeSkin(mySkin))); } catch (e){}
  renderMenuSkin();
  if (net.active) netSendSkin();                   // let everyone see the new skin live
  showScreen('homeMenu');
}
function cancelEditor(){
  if (editSnapshot) applySkinData(mySkin, editSnapshot, renderMenuSkin);
  showScreen('homeMenu');
}
function resetEditor(){ mySkin = defaultSkin(); renderEditorGrid(); renderSkinPreview(el('editorPreview'), mySkin); }
function renderMenuSkin(){ renderSkinPreview(el('menuSkin'), mySkin); }
function netSendSkin(){
  if (!net.active) return;
  const skin = serializeSkin(mySkin);
  if (net.isHost) broadcast({ t:'skin', id:'host', skin });
  else netSend(net.hostConn, { t:'skin', skin });
}
function netSendDrop(d){
  if (!net.active) return;
  const m = { t:'drop', id: d.id, type: d.type, count: d.count || 1, x: d.x, y: d.y, z: d.z, vx: d.vx, vy: d.vy, vz: d.vz };
  if (net.isHost) broadcast(m); else netSend(net.hostConn, m);
}
function netSendPickup(id){
  if (!net.active) return;
  const m = { t:'pickup', id };
  if (net.isHost) broadcast(m); else netSend(net.hostConn, m);
}
// ---- player-vs-player combat ----
function attackPlayer(id){
  if (!id || !net.active) return;
  if (ruleNoPVP()) return;                        // attacking disabled on this server
  const now = performance.now();
  if (now - lastAttackT < 350) return;            // melee cooldown
  lastAttackT = now;
  const dmg = swordPvp(heldTool());   // sword tier sets PvP hit power
  netSendHit(id, dmg);
  swingHand();                                                // first-person swing
}
function netSendHit(id, dmg){
  if (!net.active) return;
  const ax = benny.position.x, az = benny.position.z;       // so the target knows which way to fly
  if (net.isHost){
    if (id === 'host') return;
    const conn = net.conns.get(id); if (conn) netSend(conn, { t:'hit', dmg, by: username, ax, az });
  } else {
    netSend(net.hostConn, { t:'hit', target: id, dmg, by: username, ax, az });   // host routes it to the target
  }
}
function applyHit(dmg, by, ax, az){
  if (gamemode !== 'survival' || dead) return;     // creative players take no damage
  lastAttacker = by || 'another player';
  if (ax != null && az != null){                   // get knocked back, away from the attacker
    let dx = benny.position.x - ax, dz = benny.position.z - az;
    const l = Math.hypot(dx, dz) || 1; dx /= l; dz /= l;
    kbx = dx * 9; kbz = dz * 9; vy = Math.max(vy, 5.2);
  }
  damagePlayer(dmg || PVP_DMG, 'player');
}

function avatarColor(id){ let h = 0; for (let i = 0; i < id.length; i++) h = (h*31 + id.charCodeAt(i)) >>> 0; return new THREE.Color().setHSL((h % 360)/360, 0.6, 0.55).getHex(); }

const netNames = new Map();
const pings = new Map();
function refreshPlayerList(){
  const rows = el('plRows'); rows.innerHTML = '';
  const entries = [{ name: username, you: true, host: net.active && net.isHost, ping: null }];
  remotePlayers.forEach((rp, id) => entries.push({ name: netNames.get(id) || 'Player', host: id === 'host', ping: pings.get(id) }));
  el('plCount').textContent = entries.length;
  entries.forEach(en => {
    const r = document.createElement('div'); r.className = 'pl-row';
    const n = document.createElement('div'); n.className = 'pl-name'; n.textContent = en.name;
    const tag = en.you ? (en.host ? 'you · host' : 'you') : (en.host ? 'host' : '');
    if (tag){ const t = document.createElement('span'); t.className = 'pl-tag'; t.textContent = tag; n.appendChild(t); }
    const p = document.createElement('div'); p.className = 'pl-ping';
    p.textContent = en.you ? '—' : (en.ping != null ? en.ping + ' ms' : '…');
    r.appendChild(n); r.appendChild(p); rows.appendChild(r);
  });
}
function makeNameSprite(name){
  const c = document.createElement('canvas'); c.width = 256; c.height = 64; const g = c.getContext('2d');
  g.fillStyle = 'rgba(0,0,0,0.5)'; g.fillRect(0, 0, 256, 64);
  g.font = 'bold 30px sans-serif'; g.textAlign = 'center'; g.textBaseline = 'middle';
  const shown = name.slice(0, 16);
  if (name.toLowerCase() === 'kokochins'){                    // special: rainbow username
    const total = g.measureText(shown).width; let x = 128 - total/2;
    for (let i = 0; i < shown.length; i++){
      const ch = shown[i], w = g.measureText(ch).width;
      g.fillStyle = 'hsl(' + Math.round(i / shown.length * 360) + ',95%,60%)';
      g.textAlign = 'left'; g.fillText(ch, x, 33); x += w;
    }
  } else { g.fillStyle = '#fff'; g.fillText(shown, 128, 33); }
  const tex = new THREE.CanvasTexture(c);
  const sp = new THREE.Sprite(new THREE.SpriteMaterial({ map: tex, depthTest: false, transparent: true }));
  sp.scale.set(1.7, 0.42, 1); sp.position.set(0, 2.25, 0);
  return sp;
}
function setRemoteName(id, name){
  netNames.set(id, name);
  const rp = remotePlayers.get(id); if (!rp) return;
  if (rp.label) rp.group.remove(rp.label);
  rp.label = makeNameSprite(name); rp.group.add(rp.label);
  applyCape(rp);
}
// the "Owner" cape — a blue cape with black text, worn by anyone named kokochins
let CAPE_TEX = null;
function makeCapeTex(){
  const c = document.createElement('canvas'); c.width = 64; c.height = 96; const g = c.getContext('2d');
  g.fillStyle = '#244fd6'; g.fillRect(0, 0, 64, 96);                       // blue cape
  g.strokeStyle = '#16266f'; g.lineWidth = 4; g.strokeRect(2, 2, 60, 92);
  g.fillStyle = '#000'; g.font = 'bold 15px sans-serif'; g.textAlign = 'center'; g.textBaseline = 'middle';
  g.fillText('Owner', 32, 50);                                              // black "Owner" text
  const t = new THREE.CanvasTexture(c); t.magFilter = THREE.NearestFilter; t.minFilter = THREE.LinearMipmapLinearFilter; return t;
}
function addCape(group){
  if (group.userData.cape) return;
  if (!CAPE_TEX) CAPE_TEX = makeCapeTex();
  const cape = new THREE.Mesh(new THREE.BoxGeometry(0.46, 0.72, 0.06), new THREE.MeshLambertMaterial({ map: CAPE_TEX }));
  cape.position.set(0, 1.0, -0.2);                                          // on the back, hanging from the shoulders
  cape.rotation.x = 0.14;                                                   // slight drape
  cape.castShadow = true; group.add(cape); group.userData.cape = cape;
}
function removeCape(group){ if (group.userData.cape){ group.remove(group.userData.cape); group.userData.cape = null; } }
function applyCape(rp){
  if (!rp || !rp.group) return;
  const nm = netNames.get(rp.id) || '';
  if (nm.toLowerCase() === 'kokochins') addCape(rp.group); else removeCape(rp.group);
}
function applyRemoteState(id, m){
  let rp = remotePlayers.get(id);
  if (!rp){
    const group = buildAvatar(remoteSkins.get(id) || DEFAULT_SKIN); scene.add(group);
    rp = { id, group, x:m.x, y:m.y, z:m.z, tx:m.x, ty:m.y, tz:m.z, yaw:m.ry||0, tyaw:m.ry||0, pitch:m.rp||0, phase:0, label:null };
    remotePlayers.set(id, rp);
    if (netNames.has(id)){ rp.label = makeNameSprite(netNames.get(id)); group.add(rp.label); }
    applyCape(rp);
  }
  rp.tx = m.x; rp.ty = m.y; rp.tz = m.z; rp.tyaw = m.ry || 0; rp.pitch = m.rp || 0;
}
function removeRemote(id){ const rp = remotePlayers.get(id); if (rp){ scene.remove(rp.group); remotePlayers.delete(id); } }
function updateRemotePlayers(dt){
  remotePlayers.forEach(rp => {
    const ox = rp.x, oz = rp.z;
    rp.x += (rp.tx - rp.x) * Math.min(1, dt * 12);
    rp.y += (rp.ty - rp.y) * Math.min(1, dt * 12);
    rp.z += (rp.tz - rp.z) * Math.min(1, dt * 12);
    let d = rp.tyaw - rp.yaw; while (d > Math.PI) d -= 2*Math.PI; while (d < -Math.PI) d += 2*Math.PI;
    rp.yaw += d * Math.min(1, dt * 12);
    rp.group.position.set(rp.x, rp.y, rp.z);
    rp.group.rotation.y = rp.yaw;
    if (rp.group.userData.head) rp.group.userData.head.rotation.x = -rp.pitch * 0.6;
    const moved = Math.hypot(rp.x - ox, rp.z - oz);
    rp.phase += moved * 6;
    const sw = Math.sin(rp.phase) * 0.5;
    const lg = rp.group.userData.legs, ar = rp.group.userData.arms;
    if (lg){ lg[0].rotation.x = sw; lg[1].rotation.x = -sw; }
    if (ar){ ar[0].rotation.x = -sw; ar[1].rotation.x = sw; }
  });
}

function netSend(conn, obj){ try { if (conn && conn.open) conn.send(obj); } catch (e){} }
function broadcast(obj, exceptId){ net.conns.forEach((conn, id) => { if (id !== exceptId) netSend(conn, obj); }); }
function sendState(){
  if (!net.active) return;
  const m = { t:'s', x: benny.position.x, y: benny.position.y, z: benny.position.z, ry: yaw, rp: pitch };
  if (net.isHost) broadcast(Object.assign({ id:'host' }, m));
  else netSend(net.hostConn, m);
}
function netSendEdit(x, y, z, b){
  if (!net.active) return;
  const m = { t:'edit', x, y, z, b };
  if (net.isHost) broadcast(m); else netSend(net.hostConn, m);
}
function netSendChat(msg){
  if (!net.active) return;
  const m = { t:'chat', name: username, msg };
  if (net.isHost) broadcast(m); else netSend(net.hostConn, m);
}
function netSendSys(msg){                                   // system line (joins/leaves/deaths)
  if (!net.active) return;
  if (net.isHost) broadcast({ t:'sys', msg }); else netSend(net.hostConn, { t:'sys', msg });
}
function announceSys(msg){ addChatLine(msg, true); showChatLog(); netSendSys(msg); }
function kickPlayerById(id){
  const nm = netNames.get(id) || 'Player';
  const conn = net.conns.get(id);
  if (conn){ netSend(conn, { t:'kick' }); setTimeout(() => { try { conn.close(); } catch (e){} }, 120); }
  net.conns.delete(id); removeRemote(id); netNames.delete(id); pings.delete(id);
  if (managerOpen) refreshManager();
  return nm;
}
function promoteOp(id){
  if (!net.isHost || ruleNoOP()) return;
  const conn = net.conns.get(id); if (conn) netSend(conn, { t:'setop', op:true });
  announceSys(username + ' promoted ' + (netNames.get(id) || 'a player') + ' to OP.');
}
function deOp(id){
  if (!net.isHost) return;
  const conn = net.conns.get(id); if (conn) netSend(conn, { t:'setop', op:false });
  announceSys(username + ' removed OP from ' + (netNames.get(id) || 'a player') + '.');
}
// hand the world host over to another player (always-on servers only). The target claims the world id
// immediately while everyone else migrates with the usual stagger, so the chosen player wins the race.
function transferHostTo(id){
  if (!net.isHost || !kokoSMP) return;
  const conn = net.conns.get(id); if (!conn) return;
  const nm = netNames.get(id) || 'a player';
  netSend(conn, { t:'becomehost' });                 // chosen player takes over now
  broadcast({ t:'bye' }, id);                         // the rest migrate (staggered); skip the new host
  announceSys(username + ' made ' + nm + ' the world host.');
  closeManager();
  setTimeout(() => {                                  // step down and rejoin as a normal player
    try { if (net.peer) net.peer.destroy(); } catch (e){}
    net.peer = null; net.hostConn = null; net.conns.clear();
    remotePlayers.forEach((rp, rid) => removeRemote(rid)); netNames.clear(); pings.clear();
    net.isHost = false; net.active = false; migrating = false;
    setTimeout(serverJoinExisting, 700);             // give the new host time to claim the id
  }, 250);
}
// claim the always-on world id immediately (no stagger) — used by the chosen "Make Host" target
function serverBecomeHostNow(){
  if (!kokoSMP || typeof Peer === 'undefined') return;
  if (migrating) return; migrating = true;
  try { if (net.peer) net.peer.destroy(); } catch (e){}
  net.peer = null; net.hostConn = null; net.conns.clear();
  remotePlayers.forEach((rp, id) => removeRemote(id)); netNames.clear(); pings.clear();
  net.isHost = true; net.active = false;
  let peer; try { peer = new Peer(peerIdFor(smpCode)); } catch (e){ migrating = false; return; }
  net.peer = peer;
  peer.on('open', () => { net.active = true; migrating = false; lobbyAdvertise(); addChatLine('You are now the world host.', true); showChatLog(); });
  peer.on('connection', conn => setupHostConn(conn));
  peer.on('error', e => {
    if (e && e.type === 'unavailable-id'){ try { peer.destroy(); } catch (e2){} net.peer = null; net.isHost = false; migrating = false; serverJoinExisting(); }
    else { net.isHost = false; net.active = false; migrating = false; }
  });
}
// rejoin whichever always-on server we're on (koko or extreme)
function serverJoinExisting(){ kokoJoinExisting(); }

// ---- host server-manager menu (F8) ----
let managerOpen = false;
function refreshManager(){
  const list = el('managerList'); if (!list) return;
  list.innerHTML = '';
  if (!remotePlayers.size){ const e = document.createElement('div'); e.className = 'mgr-empty'; e.textContent = 'No other players are connected yet.'; list.appendChild(e); return; }
  remotePlayers.forEach((rp, id) => {
    const row = document.createElement('div'); row.className = 'mgr-row';
    const nm = document.createElement('div'); nm.className = 'mgr-name'; nm.textContent = netNames.get(id) || 'Player';
    const acts = document.createElement('div'); acts.className = 'mgr-acts';
    const tp = document.createElement('button'); tp.className = 'wc-btn'; tp.textContent = 'Teleport';
    tp.addEventListener('click', () => { const r = remotePlayers.get(id); if (r){ benny.position.set(r.x, r.y + 0.1, r.z); vy = 0; chunksDirty = true; addChatLine('Teleported to ' + (netNames.get(id) || 'player'), true); closeManager(); } });
    const cre = document.createElement('button'); cre.className = 'wc-btn'; cre.textContent = 'Creative';
    cre.addEventListener('click', () => setPlayerGamemode(id, 'creative'));
    const sur = document.createElement('button'); sur.className = 'wc-btn'; sur.textContent = 'Survival';
    sur.addEventListener('click', () => setPlayerGamemode(id, 'survival'));
    acts.appendChild(tp); if (!ruleNoCreative()) acts.appendChild(cre); acts.appendChild(sur);
    if (net.isHost && !ruleNoOP()){                            // host-only: promote to OP (off on Extreme)
      const op = document.createElement('button'); op.className = 'wc-btn'; op.textContent = 'Make OP';
      op.addEventListener('click', () => promoteOp(id));
      acts.appendChild(op);
      const deop = document.createElement('button'); deop.className = 'wc-btn'; deop.textContent = 'De-OP';
      deop.addEventListener('click', () => deOp(id));
      acts.appendChild(deop);
    }
    if (net.isHost && kokoSMP){                                 // host-only on always-on servers: hand off hosting
      const mh = document.createElement('button'); mh.className = 'wc-btn'; mh.textContent = 'Make Host';
      mh.addEventListener('click', () => transferHostTo(id));
      acts.appendChild(mh);
    }
    if (net.isHost && !kokoSMP){                                // kicking stays host-only and is off on KOKOSMP
      const kick = document.createElement('button'); kick.className = 'wc-btn danger'; kick.textContent = 'Kick';
      kick.addEventListener('click', () => { const name = netNames.get(id) || 'Player'; kickPlayerById(id); addChatLine('Kicked ' + name + '.', true); });
      acts.appendChild(kick);
    }
    row.appendChild(nm); row.appendChild(acts); list.appendChild(row);
  });
}
function openManager(){
  if (!net.active || !(net.isHost || kokoSMP)) return;          // host always; everyone on KOKOSMP
  managerOpen = true; refreshManager();
  el('managerMenu').classList.add('show');
  if (document.pointerLockElement === canvas) document.exitPointerLock();
}
function closeManager(){
  managerOpen = false; el('managerMenu').classList.remove('show');
  if (playing && !dead) canvas.requestPointerLock();
}
function setPlayerGamemode(id, gm){
  if (!net.active) return;
  if (gm === 'creative' && ruleNoCreative()) return;     // survival-only server
  if (net.isHost){
    const myId = net.peer && net.peer.id;
    if (id === 'host' || id === myId){ gamemode = gm; refreshHotbar(); }      // (host targeting itself)
    else { const conn = net.conns.get(id); if (conn) netSend(conn, { t: 'setgm', gm }); }
  } else {
    netSend(net.hostConn, { t: 'setgmreq', target: id, gm });                 // joiner asks the host to relay
  }
  addChatLine('Set ' + (netNames.get(id) || 'player') + '\u2019s game mode to ' + (gm === 'creative' ? 'Creative' : 'Survival'), true);
}
function netPing(){
  if (!net.active) return;
  const ts = Date.now();
  if (net.isHost) broadcast({ t:'ping', ts }); else netSend(net.hostConn, { t:'ping', ts });
}
function applyRemoteEdit(x, y, z, b){
  edits.set(x + ',' + y + ',' + z, b);                 // record even if the chunk isn't loaded yet
  if (b === AIR) removeSignAt(signKey(x, y, z));        // a broken sign elsewhere loses its label here too
  const ch = chunks.get(ckey(floorDiv(x, CHUNK), floorDiv(z, CHUNK)));
  if (ch) setBlockWorld(x, y, z, b);
}

function onData(conn, d){
  if (!d || !d.t) return;
  if (d.t === 'welcome'){ netLoadWorld(d); return; }
  if (d.t === 'winit'){                                   // heavy state that follows the tiny welcome
    if (Array.isArray(d.players)) for (const [pid, pname] of d.players) setRemoteName(pid, pname);
    if (d.hostSkin) setRemoteSkin('host', d.hostSkin);
    if (Array.isArray(d.playerSkins)) for (const [pid, sk] of d.playerSkins) setRemoteSkin(pid, sk);
    if (Array.isArray(d.signs)) for (const [k, txt] of d.signs) setSignText(k, txt);
    if (Array.isArray(d.edits)) for (const [k, b] of d.edits){     // re-applies on built chunks, bakes into the rest
      const p = k.split(','); applyRemoteEdit(+p[0], +p[1], +p[2], b);
    }
    return;
  }
  if (d.t === 'hello'){                                   // host learns a joiner's name
    setRemoteName(conn.peer, d.name || 'Player');
    if (d.skin) setRemoteSkin(conn.peer, d.skin);
    if (net.isHost){ broadcast({ t:'name', id: conn.peer, name: d.name || 'Player' }, conn.peer); if (d.skin) broadcast({ t:'skin', id: conn.peer, skin: d.skin }, conn.peer); lobbyUpdate(); announceSys((d.name || 'Player') + ' joined the world'); if (managerOpen) refreshManager(); }
    return;
  }
  if (d.t === 'name'){ setRemoteName(d.id, d.name || 'Player'); return; }
  if (d.t === 's'){
    const id = d.id || conn.peer;
    applyRemoteState(id, d);
    if (net.isHost) broadcast(Object.assign({ id }, d), conn.peer);   // relay to the other players
    return;
  }
  if (d.t === 'edit'){
    applyRemoteEdit(d.x, d.y, d.z, d.b);
    if (net.isHost) broadcast(d, conn.peer);
    return;
  }
  if (d.t === 'chat'){
    addChatLine((d.name || 'Player') + ': ' + d.msg); showChatLog();
    if (net.isHost) broadcast(d, conn.peer);
    return;
  }
  if (d.t === 'sign'){ setSignText(d.key, d.text); if (net.isHost) broadcast(d, conn.peer); return; }
  if (d.t === 'sys'){ addChatLine(d.msg, true); showChatLog(); if (net.isHost) broadcast(d, conn.peer); return; }
  if (d.t === 'time'){ if (!net.isHost) timeOfDay = d.time; return; }
  if (d.t === 'setgm'){ if (d.gm === 'creative' && ruleNoCreative()) return; gamemode = d.gm === 'survival' ? 'survival' : 'creative'; addChatLine('The host set your game mode to ' + gamemodeLabel(), true); showChatLog(); refreshHotbar(); return; }
  if (d.t === 'setgmreq'){                                  // a player asked (via host) to change someone's mode
    if (net.isHost){
      const myId = net.peer && net.peer.id;
      if (d.target === 'host' || d.target === myId){ gamemode = d.gm === 'survival' ? 'survival' : 'creative'; refreshHotbar(); }
      else { const c = net.conns.get(d.target); if (c) netSend(c, { t: 'setgm', gm: d.gm }); }
    }
    return;
  }
  if (d.t === 'skin'){
    if (net.isHost){ setRemoteSkin(conn.peer, d.skin); broadcast({ t:'skin', id: conn.peer, skin: d.skin }, conn.peer); }
    else if (d.id){ setRemoteSkin(d.id, d.skin); }
    return;
  }
  if (d.t === 'hit'){
    if (net.isHost && d.target){
      const myId = net.peer && net.peer.id;
      if (d.target === 'host' || d.target === myId) applyHit(d.dmg, d.by, d.ax, d.az);
      else { const c = net.conns.get(d.target); if (c) netSend(c, { t:'hit', dmg: d.dmg, by: d.by, ax: d.ax, az: d.az }); }
    } else {
      applyHit(d.dmg, d.by, d.ax, d.az);
    }
    return;
  }
  if (d.t === 'drop'){
    if (!drops.some(x => x.id === d.id)) spawnDrop(d.type, 0, 0, 0, { id: d.id, count: d.count || 1, x: d.x, y: d.y, z: d.z, vx: d.vx, vy: d.vy, vz: d.vz, remote: true });
    if (net.isHost) broadcast(d, conn.peer);
    return;
  }
  if (d.t === 'pickup'){ removeDropById(d.id); if (net.isHost) broadcast(d, conn.peer); return; }
  if (d.t === 'died'){                                      // a player died — hide their body
    const id = d.id || conn.peer; const rp = remotePlayers.get(id);
    if (rp){ rp.dead = true; rp.group.visible = false; }
    if (net.isHost) broadcast({ t:'died', id }, conn.peer);
    return;
  }
  if (d.t === 'resp'){                                      // a player respawned — show their body
    const id = d.id || conn.peer; const rp = remotePlayers.get(id);
    if (rp){ rp.dead = false; rp.group.visible = true; }
    if (net.isHost) broadcast({ t:'resp', id }, conn.peer);
    return;
  }
  if (d.t === 'hurt'){
    if (net.isHost){ const id = d.id || conn.peer; flashAvatar(id); broadcast({ t:'hurt', id }, conn.peer); }
    else if (d.id){ flashAvatar(d.id); }
    return;
  }
  if (d.t === 'ping'){ netSend(conn, { t:'pong', ts: d.ts }); return; }
  if (d.t === 'pong'){ pings.set(net.isHost ? conn.peer : 'host', Date.now() - d.ts); return; }
  if (d.t === 'setop'){
    if (net.isHost) return;
    if (d.op === false){ settings.op = false; applyOp(); addChatLine('Your operator status was removed.', true); showChatLog(); }
    else if (!ruleNoOP()){ settings.op = true; applyOp(); addChatLine('You were promoted to OP.', true); showChatLog(); }
    return;
  }
  if (d.t === 'becomehost'){ if (!net.isHost && kokoSMP) serverBecomeHostNow(); return; }
  if (d.t === 'kick'){ if (!net.isHost) netKicked('You were kicked from the world.'); return; }
  if (d.t === 'bye'){ if (!net.isHost){ if (kokoSMP && playing) kokoMigrateHost(); else netKicked('The host closed the world.'); } return; }
}

function setupHostConn(conn){
  conn.on('open', () => {
    net.conns.set(conn.peer, conn);
    // 1) tiny "welcome" — just enough to generate + enter the world instantly (always fits one packet)
    netSend(conn, { t:'welcome', seed: currentSeed, gm: gamemode, time: timeOfDay,
      finite: world.finite, half: world.half,
      hx: benny.position.x, hy: benny.position.y, hz: benny.position.z,
      hostName: username, worldName: currentWorldName, koko: kokoSMP, mode: worldMode, extreme: isExtreme });
    // 2) heavier state (edits, other players, skins, signs) — applied live after the joiner is already in.
    //    Sent as a separate message so a big payload can never stall the join itself.
    netSend(conn, { t:'winit', edits: Array.from(edits.entries()),
      players: Array.from(netNames.entries()),
      hostSkin: serializeSkin(mySkin), playerSkins: Array.from(remoteSkins.entries()),
      signs: Array.from(signs.entries()).map(([k, r]) => [k, r.text]) });
    hostStatus('A player joined! Connected: ' + net.conns.size);
    lobbyUpdate();
  });
  conn.on('data', d => onData(conn, d));
  conn.on('close', () => { const nm = netNames.get(conn.peer) || 'Player'; net.conns.delete(conn.peer); removeRemote(conn.peer); netNames.delete(conn.peer); pings.delete(conn.peer); hostStatus('A player left. Connected: ' + net.conns.size); lobbyUpdate(); announceSys(nm + ' left the world'); if (managerOpen) refreshManager(); });
  conn.on('error', () => {});
}

function netHost(pw){
  if (typeof Peer === 'undefined'){ hostStatus('Networking unavailable (PeerJS did not load — needs internet).'); return; }
  if (!pw){ hostStatus('Enter a password first.'); return; }
  netShutdown();
  net.isHost = true; net.password = pw;
  hostStatus('Starting…');
  const peer = new Peer(peerIdFor(pw)); net.peer = peer;
  peer.on('open', () => { net.active = true; hostStatus('Hosting! Others can Join with this password. Keep playing — press Back.'); if (settings.openWorld) lobbyAdvertise(); });
  peer.on('connection', conn => setupHostConn(conn));
  peer.on('error', e => {
    if (e && e.type === 'unavailable-id') hostStatus('That password is already in use. Try a different one.');
    else hostStatus('Error: ' + (e && e.type));
    net.isHost = false; net.active = false;
  });
}
function netJoin(pw){
  if (typeof Peer === 'undefined'){ joinStatus('Networking unavailable (PeerJS did not load — needs internet).'); return; }
  if (!pw){ joinStatus('Enter the host\u2019s password.'); return; }
  netShutdown();
  net.isHost = false;
  joinStatus('Connecting…');
  const peer = new Peer(); net.peer = peer;
  peer.on('open', () => {
    const conn = peer.connect(peerIdFor(pw), { reliable: true }); net.hostConn = conn;
    conn.on('open', () => { joinStatus('Connected — loading the world…'); netSend(conn, { t:'hello', name: username, skin: serializeSkin(mySkin) }); });
    conn.on('data', d => onData(conn, d));
    conn.on('close', () => netKicked('Disconnected from host.'));
    conn.on('error', () => joinStatus('Could not reach that world.'));
    setTimeout(() => { if (!net.active) joinStatus('No world found for that password (is the host hosting?).'); }, 9000);
  });
  peer.on('error', e => joinStatus((e && e.type === 'peer-unavailable') ? 'No world found for that password.' : ('Error: ' + (e && e.type))));
}
function netLoadWorld(d){
  net.active = true; net.isHost = false;
  activeWorldId = null;              // a joined world isn't saved to this player's world list
  kokoSMP = !!d.koko;
  isExtreme = (d.mode === 'extreme'); isKokoCity = (d.mode === 'kokocity'); isPvpHub = (d.mode === 'pvphub');
  if (isExtreme){ smpCode = EXTREME.code; smpSeed = EXTREME.seed; smpName = EXTREME.name; }
  if (isKokoCity){ smpCode = KOKO_CITY.code; smpSeed = KOKO_CITY.seed; smpName = KOKO_CITY.name; }
  if (isPvpHub){ smpCode = PVP_HUB.code; smpSeed = PVP_HUB.seed; smpName = PVP_HUB.name; }
  currentWorldName = d.worldName || (d.hostName ? (d.hostName + "'s world") : 'Joined world');
  if (kokoSMP && !isExtreme && !isPvpHub){ settings.op = true; applyOp(); }   // OP on KOKOSMP & Koko City (not Extreme / PvP Hub)
  worldMode = (d.mode === 'extreme' || d.mode === 'kokocity' || d.mode === 'pvphub') ? d.mode : 'normal'; currentSeed = d.seed; worldSeed = d.seed; seedWorld(d.seed);
  edits.clear(); if (Array.isArray(d.edits)) for (const [k, v] of d.edits) edits.set(k, v);
  gamemode = d.gm === 'survival' ? 'survival' : 'creative';
  timeOfDay = d.time != null ? d.time : timeOfDay;
  world.finite = !!d.finite; world.half = d.half || 256;
  clearInventory(); clearMobs(); resetVitals(); resetXp();
  chunks.forEach(ch => disposeChunkMeshes(ch)); chunks.clear();
  applyRenderDistance();
  benny.position.set((d.hx || 0.5) + 1.5, (d.hy || 40) + 1, (d.hz || 0.5));
  if (worldMode === 'extreme'){ benny.position.set(0.5, PARK_Y + 1, 0.5); parkourWon = false; }
  if (worldMode === 'kokocity'){ benny.position.set(0.5, CITY_Y + 1, 0.5); }
  if (worldMode === 'pvphub'){ benny.position.set(10.5, HUB_SKY_Y + 1, 10.5); gamemode = 'survival'; giveHubLoadout(); }
  yaw = 0; pitch = -0.1; orbiting = false;
  if (d.hostName) setRemoteName('host', d.hostName);
  if (d.hostSkin) setRemoteSkin('host', d.hostSkin);
  if (Array.isArray(d.players)) for (const [pid, pname] of d.players) setRemoteName(pid, pname);
  if (Array.isArray(d.playerSkins)) for (const [pid, sk] of d.playerSkins) setRemoteSkin(pid, sk);
  clearSigns(); if (Array.isArray(d.signs)) for (const [k, txt] of d.signs) setSignText(k, txt);
  chunksDirty = true; refreshChunkQueue();
  joinStatus('');
  beginLoad(enterPlay);
}
function netKicked(msg){
  netShutdown();
  if (playing) quitToTitle(true);
  showScreen('joinMenu'); joinStatus(msg);
}
function netShutdown(){
  if (net.isHost) broadcast({ t:'bye' });
  lobbyUnadvertise();
  net.conns.forEach(conn => { try { conn.close(); } catch (e){} });
  net.conns.clear();
  if (net.hostConn){ try { net.hostConn.close(); } catch (e){} net.hostConn = null; }
  if (net.peer){ try { net.peer.destroy(); } catch (e){} net.peer = null; }
  remotePlayers.forEach(rp => scene.remove(rp.group)); remotePlayers.clear(); netNames.clear(); pings.clear();
  net.active = false; net.isHost = false; kokoSMP = false; isExtreme = false; isKokoCity = false; isPvpHub = false;
}

// ---------- open-server directory ("lobby") + Servers menu + the always-on KOKOSMP world ----------
const KOKO = { name: 'KOKOSMP', code: 'KOKOSMP', seed: KOKO_SEED };
const EXTREME = { name: 'Extreme Challenge', code: 'EXTREMECHALLENGE', seed: EXTREME_SEED };
const KOKO_CITY = { name: 'Koko City', code: 'KOKOCITY', seed: 70707070 };
const PVP_HUB = { name: 'Koko PvP Hub', code: 'KOKOPVPHUB', seed: 55512345 };
// the active always-on server (KOKOSMP or Extreme Challenge) — used by the shared host/migration code
let smpCode = KOKO.code, smpSeed = KOKO.seed, smpName = KOKO.name;
const LOBBY_ID = 'kokocraftlobbyv1';                 // a fixed, well-known peer id used as a directory
const lobby = { peer: null, isLobby: false, conn: null, servers: new Map() };

function serverInfo(){
  return { name: currentWorldName || 'World', code: net.password,
           count: net.conns.size + 1, players: [username].concat(Array.from(netNames.values())) };
}
// advertise this world in the directory (host + Open World on). Best-effort over the public broker.
function lobbyAdvertise(){
  if (typeof Peer === 'undefined' || !net.active || !net.isHost || !net.password) return;
  if (lobby.isLobby){ lobby.servers.set(net.password, Object.assign(serverInfo(), { _self: true })); return; }
  if (lobby.conn && lobby.conn.open){ netSend(lobby.conn, { t: 'register', srv: serverInfo() }); return; }
  const probe = new Peer();
  probe.on('open', () => {
    const c = probe.connect(LOBBY_ID, { reliable: true }); let ok = false;
    c.on('open', () => { ok = true; lobby.peer = probe; lobby.conn = c; netSend(c, { t: 'register', srv: serverInfo() }); });
    c.on('error', () => {});
    setTimeout(() => { if (!ok){ try { probe.destroy(); } catch (e){} becomeLobby(true); } }, 3500);
  });
  probe.on('error', () => becomeLobby(true));
}
function lobbyUpdate(){       // push fresh count/players to the directory
  if (!net.isHost || !settings.openWorld || !net.password) return;
  if (lobby.isLobby){ if (lobby.servers.has(net.password)) lobby.servers.set(net.password, Object.assign(serverInfo(), { _self: true })); return; }
  if (lobby.conn && lobby.conn.open) netSend(lobby.conn, { t: 'update', code: net.password, count: net.conns.size + 1, players: [username].concat(Array.from(netNames.values())) });
}
function lobbyUnadvertise(){
  try {
    if (lobby.isLobby){ if (net.password) lobby.servers.delete(net.password); }
    else if (lobby.conn && lobby.conn.open && net.password) netSend(lobby.conn, { t: 'unregister', code: net.password });
  } catch (e){}
  if (lobby.peer){ try { lobby.peer.destroy(); } catch (e){} }
  lobby.peer = null; lobby.conn = null; lobby.isLobby = false; lobby.servers.clear();
}
// claim the directory id and serve the server list to others
function becomeLobby(advertiseSelf){
  if (typeof Peer === 'undefined') return;
  const p = new Peer(LOBBY_ID);
  p.on('open', () => { lobby.peer = p; lobby.isLobby = true; if (advertiseSelf) lobby.servers.set(net.password, Object.assign(serverInfo(), { _self: true })); });
  p.on('connection', conn => {
    conn.on('data', d => {
      if (!d || !d.t) return;
      if (d.t === 'register'){ const s = d.srv; s._conn = conn.peer; lobby.servers.set(s.code, s); }
      else if (d.t === 'update'){ const s = lobby.servers.get(d.code); if (s){ s.count = d.count; s.players = d.players; } }
      else if (d.t === 'unregister'){ lobby.servers.delete(d.code); }
      else if (d.t === 'list'){ netSend(conn, { t: 'servers', list: Array.from(lobby.servers.values()).map(s => ({ name: s.name, code: s.code, count: s.count, players: s.players })) }); }
    });
    conn.on('close', () => { lobby.servers.forEach((v, k) => { if (v._conn === conn.peer) lobby.servers.delete(k); }); });
    conn.on('error', () => {});
  });
  p.on('error', e => { if (e && e.type === 'unavailable-id'){ lobby.isLobby = false; lobbyAdvertise(); } });
}
// fetch the directory list (always includes the KOKOSMP default), then call cb(list)
function queryServers(cb){
  const base = [{ name: KOKO.name, code: KOKO.code, count: null, players: [], koko: true },
                { name: EXTREME.name, code: EXTREME.code, count: null, players: [], koko: true, extreme: true },
                { name: KOKO_CITY.name, code: KOKO_CITY.code, count: null, players: [], koko: true, city: true },
                { name: PVP_HUB.name, code: PVP_HUB.code, count: null, players: [], koko: true, pvp: true }];
  const notBase = s => s.code !== KOKO.code && s.code !== EXTREME.code && s.code !== KOKO_CITY.code && s.code !== PVP_HUB.code;
  if (typeof Peer === 'undefined'){ cb(base); return; }
  // if this client IS the directory (it became the lobby while hosting open), answer from local state
  if (lobby.isLobby){
    const local = Array.from(lobby.servers.values()).map(s => ({ name: s.name, code: s.code, count: s.count, players: s.players }));
    cb(base.concat(local.filter(notBase))); return;
  }
  const probe = new Peer(); let done = false;
  const finish = (list) => { if (done) return; done = true; try { probe.destroy(); } catch (e){} cb(list); };
  probe.on('open', () => {
    const c = probe.connect(LOBBY_ID, { reliable: true });
    c.on('open', () => netSend(c, { t: 'list' }));
    c.on('data', d => { if (d && d.t === 'servers') finish(base.concat((d.list || []).filter(notBase))); });
    c.on('error', () => {});
  });
  probe.on('error', () => finish(base));
  setTimeout(() => finish(base), 6500);                  // give the public broker time to answer
}

// join a world directly by its code (used by the Servers menu — no passcode typed)
function netJoinByCode(code, opts){
  opts = opts || {};
  if (typeof Peer === 'undefined'){ serversStatus('Networking unavailable (needs internet).'); return; }
  netShutdown();
  net.isHost = false; net.password = code;
  serversStatus('Connecting…');
  const peer = new Peer(); net.peer = peer;
  peer.on('open', () => {
    const conn = peer.connect(peerIdFor(code), { reliable: true }); net.hostConn = conn; let ok = false;
    conn.on('open', () => { ok = true; serversStatus('Connected — loading…'); netSend(conn, { t: 'hello', name: username, skin: serializeSkin(mySkin) }); });
    conn.on('data', d => onData(conn, d));
    conn.on('close', () => { if (kokoSMP && playing) kokoMigrateHost(); else netKicked('Disconnected from host.'); });
    conn.on('error', () => {});
    setTimeout(() => { if (!net.active){ try { peer.destroy(); } catch (e){} net.peer = null;       // connected but never got the world from the host
      if (opts.hostIfMissing) hostKoko(); else { serversStatus('Couldn\u2019t load that world — the host didn\u2019t respond. Try again.'); showScreen('serversMenu'); } } }, 9000);
  });
  peer.on('error', e => {
    if (e && e.type === 'peer-unavailable' && opts.hostIfMissing){ try { peer.destroy(); } catch (e2){} net.peer = null; hostKoko(); }
    else serversStatus('Could not reach that server.');
  });
}
// host the always-on KOKOSMP world (fixed seed). Whoever arrives first holds the id;
// everyone else automatically joins them, so it's one shared online world (no manual host pick).
// Enter KOKOSMP reliably: load the deterministic world LOCALLY and start playing immediately,
// so entry never depends on a network handshake (this is what was erroring/hanging before).
// THEN, in the background, best-effort link players together (claim the shared id, or join whoever has it).
function enterKoko(){
  netShutdown();
  kokoSMP = true; isExtreme = false; isKokoCity = false; isPvpHub = false; smpCode = KOKO.code; smpSeed = KOKO.seed; smpName = KOKO.name;
  activeWorldId = null; currentWorldName = KOKO.name;
  settings.op = true; settings.openWorld = true; applyOp(); applyOpenWorld();
  worldMode = "normal"; currentSeed = KOKO.seed; worldSeed = KOKO.seed; gamemode = 'creative'; world.finite = false;
  orbiting = false; applyRenderDistance();
  startNewWorld(KOKO.seed);            // deterministic world — same castle for everyone, no networking needed
  beginLoad(enterPlay);               // you are in-world right away (guaranteed, no error screen)
  kokoConnect();                      // background: try to see other players too
}
// Extreme Challenge: an always-on parkour world over the void. Same shared-host machinery as KOKOSMP,
// but a deterministic parkour course; dying drops you back to the start circle; finishing announces a win.
function enterExtreme(){
  netShutdown();
  kokoSMP = true; isExtreme = true; isKokoCity = false; isPvpHub = false; smpCode = EXTREME.code; smpSeed = EXTREME.seed; smpName = EXTREME.name;
  activeWorldId = null; currentWorldName = EXTREME.name;
  settings.op = false; settings.openWorld = true; applyOp(); applyOpenWorld();
  worldMode = "extreme"; currentSeed = EXTREME.seed; worldSeed = EXTREME.seed; gamemode = 'survival'; world.finite = false;
  parkourWon = false; orbiting = false; applyRenderDistance();
  startNewWorld(EXTREME.seed);
  beginLoad(enterPlay);
  kokoConnect();
}
// Koko City: an always-on 1000x1000 city. Same shared-host machinery; no building, commands allowed, no hunger.
function enterKokoCity(){
  netShutdown();
  kokoSMP = true; isExtreme = false; isKokoCity = true; isPvpHub = false; smpCode = KOKO_CITY.code; smpSeed = KOKO_CITY.seed; smpName = KOKO_CITY.name;
  activeWorldId = null; currentWorldName = KOKO_CITY.name;
  settings.op = true; settings.openWorld = true; applyOp(); applyOpenWorld();      // OP so everyone can use commands
  worldMode = "kokocity"; currentSeed = KOKO_CITY.seed; worldSeed = KOKO_CITY.seed; gamemode = 'creative'; world.finite = false;
  orbiting = false; applyRenderDistance();
  startNewWorld(KOKO_CITY.seed);
  beginLoad(enterPlay);
  kokoConnect();
}
// Koko PvP Hub: a floating sky-island lobby; drop through the hole into a walled arena. Survival, no build/fall/commands.
function enterPvpHub(){
  netShutdown();
  kokoSMP = true; isExtreme = false; isKokoCity = false; isPvpHub = true; smpCode = PVP_HUB.code; smpSeed = PVP_HUB.seed; smpName = PVP_HUB.name;
  activeWorldId = null; currentWorldName = PVP_HUB.name;
  settings.op = false; settings.openWorld = true; applyOp(); applyOpenWorld();
  worldMode = "pvphub"; currentSeed = PVP_HUB.seed; worldSeed = PVP_HUB.seed; gamemode = 'survival'; world.finite = false;
  orbiting = false; applyRenderDistance();
  startNewWorld(PVP_HUB.seed);
  giveHubLoadout();                                  // spawn with a sword + 64 beef
  beginLoad(enterPlay);
  kokoConnect();
}
// background linking — never blocks entry, never shows an error if it fails (you just play solo on the shared seed)
function kokoConnect(){
  if (typeof Peer === 'undefined') return;             // offline / preview: stay solo on the shared world
  let peer;
  try { peer = new Peer(peerIdFor(smpCode)); } catch (e){ return; }
  net.peer = peer; net.isHost = true; net.password = smpCode;
  peer.on('open', () => { net.active = true; lobbyAdvertise(); addChatLine('Linked to KOKOSMP (hosting).', true); });
  peer.on('connection', conn => setupHostConn(conn));
  peer.on('error', e => {
    if (e && e.type === 'unavailable-id'){             // someone already holds the world — join them (still in-world)
      try { peer.destroy(); } catch (e2){} net.peer = null; net.isHost = false; kokoJoinExisting();
    } else { net.isHost = false; net.active = false; }  // any other failure: stay solo, no popup
  });
}
function kokoJoinExisting(){
  let peer;
  try { peer = new Peer(); } catch (e){ return; }
  net.peer = peer; net.isHost = false; net.password = smpCode;
  peer.on('open', () => {
    const conn = peer.connect(peerIdFor(smpCode), { reliable: true }); net.hostConn = conn;
    conn.on('open', () => { net.active = true; netSend(conn, { t:'hello', name: username, skin: serializeSkin(mySkin) }); addChatLine('Linked to KOKOSMP.', true); });
    conn.on('data', d => onData(conn, d));
    conn.on('close', () => { if (kokoSMP && playing) kokoMigrateHost(); });   // keep the world alive on host loss
    conn.on('error', () => {});
  });
  peer.on('error', () => { net.active = false; });     // stay solo on failure
}
function hostKoko(){
  if (typeof Peer === 'undefined'){ serversStatus('Networking unavailable (needs internet).'); return; }
  serversStatus('Connecting to KOKOSMP…');
  netShutdown();
  net.isHost = true; net.password = KOKO.code;
  kokoSMP = true; activeWorldId = null; currentWorldName = KOKO.name;
  settings.op = true; settings.openWorld = true; applyOp(); applyOpenWorld();
  currentSeed = KOKO.seed; gamemode = 'creative'; world.finite = false;
  const peer = new Peer(peerIdFor(KOKO.code)); net.peer = peer;
  peer.on('open', () => {                       // we successfully claimed the world — host it
    net.active = true;
    orbiting = false; applyRenderDistance();
    startNewWorld(KOKO.seed);
    lobbyAdvertise();
    beginLoad(enterPlay);
  });
  peer.on('connection', conn => setupHostConn(conn));
  peer.on('error', e => {
    if (e && e.type === 'unavailable-id'){      // someone is already hosting KOKOSMP — join them instead
      try { peer.destroy(); } catch (e2){} net.peer = null; net.isHost = false;
      netJoinByCode(KOKO.code, {});
    } else { net.isHost = false; net.active = false; serversStatus('Could not reach KOKOSMP.'); }
  });
}
// when the current KOKOSMP host leaves, a remaining player claims the world id and keeps it alive.
// everyone races for it; the winner hosts (no reload), the rest auto-rejoin the new host.
let migrating = false;
function kokoMigrateHost(){
  if (!kokoSMP) { netKicked('Disconnected from host.'); return; }
  if (migrating) return;                                   // don't run twice if bye + close both fire
  migrating = true;
  addChatLine('Host left — keeping KOKOSMP alive…', true); showChatLog();
  try { if (net.peer) net.peer.destroy(); } catch (e){}
  net.peer = null; net.hostConn = null; net.conns.clear();
  remotePlayers.forEach((rp, id) => removeRemote(id)); netNames.clear(); pings.clear();
  net.isHost = true; net.active = false;
  // small random stagger so every joiner doesn't grab the freed id at the exact same instant
  setTimeout(() => {
    let peer; try { peer = new Peer(peerIdFor(smpCode)); } catch (e){ migrating = false; return; }
    net.peer = peer;
    peer.on('open', () => {                       // we won the race — host the same world (no reload)
      net.active = true; migrating = false; lobbyAdvertise();
      addChatLine('You are now hosting KOKOSMP. The world stays up.', true); showChatLog();
    });
    peer.on('connection', conn => setupHostConn(conn));
    peer.on('error', e => {
      if (e && e.type === 'unavailable-id'){      // someone else took over first — quietly rejoin them
        try { peer.destroy(); } catch (e2){} net.peer = null; net.isHost = false; migrating = false;
        kokoJoinExisting();
      } else { net.isHost = false; net.active = false; migrating = false; }   // stay solo on the shared world, no kick
    });
  }, 200 + Math.floor(Math.random() * 700));
}
function serversStatus(s){ const e = el('serversStatus'); if (e) e.textContent = s; }
let serverTab = 'creator';                        // 'creator' (always-on) | 'player' (LAN/open worlds)
function setServerTab(tab){
  serverTab = tab;
  el('srvTabCreator').classList.toggle('active', tab === 'creator');
  el('srvTabPlayer').classList.toggle('active', tab === 'player');
  refreshServerList();
}
function refreshServerList(){
  const list = el('serverList'); if (!list) return;
  list.innerHTML = ''; serversStatus(serverTab === 'creator' ? 'Loading creator servers…' : 'Searching for open LAN worlds…');
  queryServers(servers => {
    const creators = servers.filter(s => s.koko), players = servers.filter(s => !s.koko);
    const shown = serverTab === 'creator' ? creators : players;
    serversStatus(serverTab === 'creator' ? '' : (players.length ? '' : 'No open LAN worlds found. Open a world to LAN to host one.'));
    list.innerHTML = '';
    if (!shown.length){ const e = document.createElement('div'); e.className = 'mgr-empty'; e.textContent = serverTab === 'creator' ? 'No creator servers available.' : 'No player-made worlds are online right now.'; list.appendChild(e); return; }
    shown.forEach(srv => {
      const card = document.createElement('div'); card.className = 'world-card';
      const left = document.createElement('div'); left.className = 'wc-left';
      const nm = document.createElement('div'); nm.className = 'wc-name'; nm.textContent = srv.name;
      if (srv.koko){ const b = document.createElement('span'); b.className = 'srv-badge'; b.textContent = srv.extreme ? 'challenge' : (srv.city ? 'city' : (srv.pvp ? 'pvp' : 'default')); nm.appendChild(b); }
      const meta = document.createElement('div'); meta.className = 'wc-meta'; meta.textContent = srv.extreme ? 'parkour over the void' : (srv.city ? '1000×1000 city' : (srv.pvp ? 'sky-island PvP arena' : (srv.koko ? 'always online' : 'open world')));
      left.appendChild(nm); left.appendChild(meta);
      left.addEventListener('click', () => joinFromServers(srv));
      const acts = document.createElement('div'); acts.className = 'wc-actions';
      const cnt = document.createElement('div'); cnt.className = 'srv-count';
      cnt.textContent = (srv.count != null ? srv.count : '—') + ' \u25CF';
      const pop = document.createElement('div'); pop.className = 'srv-players';
      const names = (srv.players && srv.players.length) ? srv.players : (srv.koko ? ['(join to populate)'] : ['(no players)']);
      names.forEach(p => { const d = document.createElement('div'); d.textContent = p; pop.appendChild(d); });
      cnt.appendChild(pop); acts.appendChild(cnt);
      const play = document.createElement('button'); play.className = 'wc-btn'; play.textContent = 'Join';
      play.addEventListener('click', () => joinFromServers(srv));
      acts.appendChild(play);
      card.appendChild(left); card.appendChild(acts); list.appendChild(card);
    });
  });
}
function joinFromServers(srv){
  if (srv.extreme) enterExtreme();                  // parkour challenge server
  else if (srv.city) enterKokoCity();               // city server
  else if (srv.pvp) enterPvpHub();                  // pvp hub server
  else if (srv.koko) enterKoko();                   // local-first entry — always works, world stays up
  else netJoinByCode(srv.code, {});
}

// menu wiring
el('hmJoin').addEventListener('click', () => { joinStatus(''); el('joinPw').value = ''; showScreen('joinMenu'); });
el('hmServers').addEventListener('click', () => { showScreen('serversMenu'); setServerTab('creator'); });
el('srvTabCreator').addEventListener('click', () => setServerTab('creator'));
el('srvTabPlayer').addEventListener('click', () => setServerTab('player'));
el('serversBack').addEventListener('click', () => { netShutdown(); showScreen('homeMenu'); });
el('serversRefresh').addEventListener('click', () => refreshServerList());
el('joinCancel').addEventListener('click', () => { netShutdown(); showScreen('homeMenu'); });
el('joinGo').addEventListener('click', () => netJoin(el('joinPw').value.trim()));
el('btnLan').addEventListener('click', () => { el('lanHostPw').value = net.password || ''; hostStatus(net.active && net.isHost ? 'Already hosting with this password.' : ''); showScreen('lanHostMenu'); });
el('lanHostCancel').addEventListener('click', () => showPause());
el('lanHostStart').addEventListener('click', () => netHost(el('lanHostPw').value.trim()));

// username gate (shown before the main menu)
// ---- accounts: each username+password is an isolated profile (worlds/settings/skin scoped to it) ----
function acctKey(suffix){ return 'kokoA::' + (account || 'guest') + '::' + suffix; }
function hashPass(s){ let h = 5381; s = String(s); for (let i = 0; i < s.length; i++) h = ((h * 33) ^ s.charCodeAt(i)) >>> 0; return h.toString(36); }
function loadAccounts(){ try { return JSON.parse(localStorage.getItem('kokoAccounts')) || {}; } catch (e){ return {}; } }
function saveAccounts(o){ try { localStorage.setItem('kokoAccounts', JSON.stringify(o)); } catch (e){} }
function acctMsg(t){ el('acctStatus').textContent = t || ''; }
function setCurrentAccount(name){
  account = name.toLowerCase();
  username = name;
  try { localStorage.setItem('kokoLastUser', name); } catch (e){}
  mySkin = defaultSkin();                       // start from defaults, then load this account's saved data
  loadSettings();
  try { const sk = localStorage.getItem(acctKey('skin')); if (sk) applySkinData(mySkin, JSON.parse(sk), renderMenuSkin); } catch (e){}
  syncSettingsUI();
  renderMenuSkin();
}
function doLogin(){
  const name = el('acctUser').value.trim().slice(0, 16);
  const pass = el('acctPass').value;
  if (!name){ acctMsg('Enter your username.'); return; }
  const accts = loadAccounts(); const rec = accts[name.toLowerCase()];
  if (!rec){ acctMsg('No account with that username — create one below.'); return; }
  if (rec.hash !== hashPass(pass)){ acctMsg('Wrong password.'); return; }
  acctMsg(''); setCurrentAccount(rec.name); showScreen('homeMenu');
}
function doCreate(){
  const name = el('acctUser').value.trim().slice(0, 16);
  const pass = el('acctPass').value;
  if (!name){ acctMsg('Choose a username.'); return; }
  if (pass.length < 3){ acctMsg('Password must be at least 3 characters.'); return; }
  const accts = loadAccounts();
  if (accts[name.toLowerCase()]){ acctMsg('That username is taken — choose another.'); return; }
  accts[name.toLowerCase()] = { name, hash: hashPass(pass) };
  saveAccounts(accts);
  acctMsg(''); setCurrentAccount(name); showScreen('homeMenu');
}
el('acctLogin').addEventListener('click', doLogin);
el('acctCreate').addEventListener('click', doCreate);
el('acctPass').addEventListener('keydown', e => { if (e.code === 'Enter') doLogin(); });
el('acctUser').addEventListener('keydown', e => { if (e.code === 'Enter') el('acctPass').focus(); });

(function boot(){
  syncSettingsUI();              // apply default render distance / fov / textures for the menu backdrop
  buildCrackTextures();          // breaking-animation overlay
  refreshHotbar();               // empty hotbar to start
  refreshDebug();                // F3 debug panel starts hidden
  startBackdrop();              // generate a scenic world for the menu background
  renderMenuSkin();              // default skin preview until an account loads
  try { const last = localStorage.getItem('kokoLastUser'); if (last) el('acctUser').value = last; } catch (e){}
  showScreen('usernameMenu');    // log in or create an account before the main menu
  animate();
})();

</script>
</body>
</html>
