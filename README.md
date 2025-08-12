""" Telegram Mines Bot - Simulation + Statistics File: mines_bot.py

Requirements: pip install python-telegram-bot==20.3

How to use:

1. Create a bot with @BotFather and get the TOKEN.


2. Fill TOKEN below or set environment variable TELEGRAM_BOT_TOKEN.


3. Run: python mines_bot.py


4. Host on a server (Heroku/Railway/VPS) for 24/7 availability.



This implementation:

5x5 board with 3 mines by default (configurable).

Uses InlineKeyboard for the grid; users click cells.

Stores simple per-user statistics in stats.json.

Keeps active games in memory while running.


Notes:

This is a training/simulation bot, not for real gambling or prediction.

You can extend persistence to a DB (SQLite/Postgres) if needed.


"""

import json import os import random import logging from typing import Dict, List, Tuple from pathlib import Path

from telegram import Update, InlineKeyboardMarkup, InlineKeyboardButton from telegram.ext import ( ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes, )

---------- Config ----------

TOKEN = os.getenv('TELEGRAM_BOT_TOKEN') or 'PUT_YOUR_TOKEN_HERE' ROWS = 5 COLS = 5 MINES = 3 STATS_FILE = Path('stats.json')

logging.basicConfig(level=logging.INFO) logger = logging.getLogger(name)

In-memory active games: user_id -> game state

active_games: Dict[int, Dict] = {}

---------- Helpers: storage ----------

def load_stats() -> Dict: if not STATS_FILE.exists(): return {} try: with STATS_FILE.open('r', encoding='utf-8') as f: return json.load(f) except Exception as e: logger.exception('Failed to load stats: %s', e) return {}

def save_stats(stats: Dict): try: with STATS_FILE.open('w', encoding='utf-8') as f: json.dump(stats, f, ensure_ascii=False, indent=2) except Exception as e: logger.exception('Failed to save stats: %s', e)

---------- Game logic ----------

def new_game(rows=ROWS, cols=COLS, mines=MINES) -> Dict: # Create a game state dict # Board coordinates: (r, c) with r in [0..rows-1] all_cells = [(r, c) for r in range(rows) for c in range(cols)] mine_positions = set(random.sample(all_cells, mines)) # Precompute neighbor counts for nicer UX neighbor_counts = {} for r in range(rows): for c in range(cols): count = 0 for dr in (-1, 0, 1): for dc in (-1, 0, 1): if dr == 0 and dc == 0: continue nr, nc = r + dr, c + dc if (nr, nc) in mine_positions: count += 1 neighbor_counts[(r, c)] = count

return {
    'rows': rows,
    'cols': cols,
    'mines': mines,
    'mine_positions': [list(x) for x in mine_positions],
    'opened': [],  # list of [r,c]
    'neighbor_counts': {f'{r},{c}': neighbor_counts[(r, c)] for r in range(rows) for c in range(cols)},
    'over': False,
    'won': False,
}

def is_mine(game: Dict, r: int, c: int) -> bool: return [r, c] in game['mine_positions']

def open_cell(game: Dict, r: int, c: int) -> Tuple[str, Dict]: # Returns ('mine'|'safe'|'already'|'end'), game if game['over']: return 'end', game key = [r, c] if key in game['opened']: return 'already', game if is_mine(game, r, c): game['opened'].append(key) game['over'] = True game['won'] = False return 'mine', game # Safe cell: open it game['opened'].append(key) # optional: auto-open neighbors when neighbor_count==0 (like Minesweeper) if game['neighbor_counts'].get(f'{r},{c}', 0) == 0: flood_fill_open(game, r, c) # Check win: all non-mine cells opened rows = game['rows']; cols = game['cols']; mines = game['mines'] total_cells = rows * cols if len(game['opened']) >= total_cells - mines: game['over'] = True game['won'] = True return 'win', game return 'safe', game

def flood_fill_open(game: Dict, r: int, c: int): # BFS to open neighbors with zero neighbor count rows, cols = game['rows'], game['cols'] stack = [(r, c)] while stack: cr, cc = stack.pop() for dr in (-1, 0, 1): for dc in (-1, 0, 1): nr, nc = cr + dr, cc + dc if 0 <= nr < rows and 0 <= nc < cols: if [nr, nc] not in game['opened'] and not is_mine(game, nr, nc): game['opened'].append([nr, nc]) if game['neighbor_counts'].get(f'{nr},{nc}', 0) == 0: stack.append((nr, nc))

---------- Telegram keyboards / rendering ----------

def render_board_markup(game: Dict, reveal_all=False) -> InlineKeyboardMarkup: rows = game['rows']; cols = game['cols'] keyboard = [] opened = [tuple(x) for x in game['opened']] mines = [tuple(x) for x in game['mine_positions']] for r in range(rows): row_buttons = [] for c in range(cols): if (r, c) in opened or reveal_all: if (r, c) in mines: text = '💣' else: cnt = game['neighbor_counts'].get(f'{r},{c}', 0) text = str(cnt) if cnt > 0 else '▫️' # disabled button (no callback) btn = InlineKeyboardButton(text, callback_data='noop') else: # hidden btn = InlineKeyboardButton('◻️', callback_data=f'open:{r}:{c}') row_buttons.append(btn) keyboard.append(row_buttons) return InlineKeyboardMarkup(keyboard)

---------- Command handlers ----------

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE): text = ( "Bienvenue dans Mines Bot 🎲\n\n" "Commandes:\n" "/jouer - lancer une partie (5x5, 3 mines par défaut)\n" "/stats - voir tes statistiques\n" "/help - aide rapide\n\n" "Ceci est une simulation / entraînement." ) await update.message.reply_text(text)

async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE): text = ( "Aide rapide:\n" "- /jouer pour commencer une nouvelle partie.\n" "- Clique sur les cases pour les ouvrir.\n" "- Si tu ouvres une mine, la partie est terminée.\n" "- Tes stats sont sauvegardées localement.\n" ) await update.message.reply_text(text)

async def jouer(update: Update, context: ContextTypes.DEFAULT_TYPE): user_id = update.effective_user.id game = new_game(rows=ROWS, cols=COLS, mines=MINES) active_games[user_id] = game kb = render_board_markup(game) await update.message.reply_text('Nouvelle partie: bonne chance !', reply_markup=kb)

async def stats_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE): user_id = str(update.effective_user.id) stats = load_stats() u = stats.get(user_id, {'total': 0, 'wins': 0, 'losses': 0, 'cells_opened': 0}) text = ( f"Tes statistiques:\n" f"Parties jouées: {u.get('total',0)}\n" f"Victoires: {u.get('wins',0)}\n" f"Défaites: {u.get('losses',0)}\n" f"Cases ouvertes (total): {u.get('cells_opened',0)}\n" ) await update.message.reply_text(text)

---------- Callback for button presses ----------

async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE): query = update.callback_query await query.answer() user_id = query.from_user.id data = query.data if data == 'noop': return if not data.startswith('open:'): await query.edit_message_text('Commande inconnue.') return parts = data.split(':') if len(parts) != 3: await query.edit_message_text('Données invalides.') return r = int(parts[1]); c = int(parts[2]) game = active_games.get(user_id) if not game: await query.edit_message_text('Aucune partie en cours. Lance une nouvelle partie avec /jouer') return result, game = open_cell(game, r, c) # Update in-memory active_games[user_id] = game

# Update stats depending on result
stats = load_stats()
uid = str(user_id)
if uid not in stats:
    stats[uid] = {'total': 0, 'wins': 0, 'losses': 0, 'cells_opened': 0}

if result == 'already':
    # just refresh board
    kb = render_board_markup(game)
    try:
        await query.edit_message_reply_markup(reply_markup=kb)
    except Exception:
        pass
    return

if result == 'mine':
    stats[uid]['total'] += 1
    stats[uid]['losses'] += 1
    stats[uid]['cells_opened'] += len(game['opened'])
    save_stats(stats)
    # reveal full board
    kb = render_board_markup(game, reveal_all=True)
    await query.edit_message_text('💥 Boom ! Tu as trouvé une mine. Partie terminée.', reply_markup=kb)
    # remove active game
    active_games.pop(user_id, None)
    return

if result == 'win':
    stats[uid]['total'] += 1
    stats[uid]['wins'] += 1
    stats[uid]['cells_opened'] += len(game['opened'])
    save_stats(stats)
    kb = render_board_markup(game, reveal_all=True)
    await query.edit_message_text('🎉 Bravo — tu as vidé le plateau sans mines !', reply_markup=kb)
    active_games.pop(user_id, None)
    return

# safe hit
# update cells opened count only at the end (on win/lose) to avoid double counting if user opens many cells
kb = render_board_markup(game)
try:
    await query.edit_message_reply_markup(reply_markup=kb)
except Exception:
    # fallback: edit message text
    await query.edit_message_text('Case ouverte — continue !', reply_markup=kb)

---------- Main ----------

def main(): if TOKEN == 'PUT_YOUR_TOKEN_HERE' or not TOKEN: print('Erreur : mets ton token dans la variable TOKEN ou dans l'env TELEGRAM_BOT_TOKEN') return

app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(CommandHandler('start', start))
app.add_handler(CommandHandler('help', help_cmd))
app.add_handler(CommandHandler('jouer', jouer))
app.add_handler(CommandHandler('stats', stats_cmd))
app.add_handler(CallbackQueryHandler(button_callback))

print('Bot démarré...')
app.run_polling()

if name == 'main': main()

# Mines_bot.py
