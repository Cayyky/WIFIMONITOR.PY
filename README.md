#!/usr/bin/env python3
import tkinter as tk
from tkinter import ttk, scrolledtext, messagebox
import sqlite3
import time
import threading
import subprocess
import re
from datetime import datetime
from collections import deque
from ping3 import ping

# --- CONFIGURAÇÕES ---
DB_NAME = "monitoramento.db"
DEFAULT_TARGET = "8.8.8.8"
INTERVALO = 1
TIMEOUT_PING = 2
PING_ALTO_THRESHOLD = 150

class WifiMonitorGUI(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Monitor de Rede - Tentando")
        self.geometry("420x780")
        self.resizable(False, False)
        
        # Variáveis de Controle
        self.monitoring = False
        self.thread = None
        self.last_ping = 0.0
        self.is_offline = False
        self.use_system_ping = False 
        self.current_target = DEFAULT_TARGET
        
        # Variáveis de Opções
        self.var_sound = tk.BooleanVar(value=True)
        self.var_notify = tk.BooleanVar(value=True)
        self.var_top = tk.BooleanVar(value=False)

        self.ping_history = deque([0]*40, maxlen=40)

        self.colors = {
            'bg': '#2b2b2b', 'fg': '#ffffff',
            'panel': '#3c3f41', 'green': '#5cb85c',
            'red': '#d9534f', 'yellow': '#f0ad4e',
            'graph_bg': '#1e1e1e', 'line': '#00ffff',
            'entry_bg': '#1e1e1e'
        }
        self.configure_style()
        self.setup_ui()
        self.setup_database()

    def configure_style(self):
        style = ttk.Style(self)
        style.theme_use('clam')
        self.configure(background=self.colors['bg'])
        style.configure('.', background=self.colors['bg'], foreground=self.colors['fg'])
        style.configure('TLabel', background=self.colors['bg'], foreground=self.colors['fg'])
        style.configure('TButton', font=('Segoe UI', 10, 'bold'))
        style.configure('Status.TFrame', background=self.colors['panel'])
        style.configure('TEntry', fieldbackground=self.colors['entry_bg'], foreground='white', insertcolor='white')
        style.configure('TCheckbutton', background=self.colors['bg'], foreground=self.colors['fg'], font=('Segoe UI', 10))
        style.map('TCheckbutton', background=[('active', self.colors['bg'])], indicatorcolor=[('selected', self.colors['green'])])

    def setup_ui(self):
        # Painel Status
        self.status_frame = ttk.Frame(self, style='Status.TFrame')
        self.status_frame.pack(fill='x', padx=10, pady=10)

        self.lbl_status_text = tk.Label(self.status_frame, text="PARADO", font=('Segoe UI', 20, 'bold'), bg=self.colors['panel'], fg='#999')
        self.lbl_status_text.pack(pady=(10, 2))

        self.lbl_ping_val = tk.Label(self.status_frame, text="-- ms", font=('Segoe UI', 30, 'bold'), bg=self.colors['panel'], fg='#999')
        self.lbl_ping_val.pack(pady=(0, 10))

        # Config IP
        config_frame = ttk.Frame(self)
        config_frame.pack(fill='x', padx=20, pady=(0, 5))
        ttk.Label(config_frame, text="IP Alvo:").pack(side='left', padx=(0, 5))
        self.entry_target = ttk.Entry(config_frame)
        self.entry_target.pack(side='left', fill='x', expand=True)
        self.entry_target.insert(0, DEFAULT_TARGET)

        # Opções Extras
        opts_frame = ttk.LabelFrame(self, text="Opções Extras", padding=10)
        opts_frame.pack(fill='x', padx=20, pady=10)
        
        ttk.Checkbutton(opts_frame, text="Notificações (notify-send)", variable=self.var_notify).pack(anchor='w')
        ttk.Checkbutton(opts_frame, text="Alertas Sonoros (Bip)", variable=self.var_sound).pack(anchor='w')
        ttk.Checkbutton(opts_frame, text="Sempre no Topo", variable=self.var_top, command=self.toggle_topmost).pack(anchor='w')

        # Botão Ação
        self.btn_action = ttk.Button(self, text="INICIAR MONITORAMENTO", command=self.toggle_monitoring)
        self.btn_action.pack(fill='x', padx=20, pady=5, ipady=5)

        # Gráfico
        lbl_graph = ttk.Label(self, text="Visualização em Tempo Real:", font=('Segoe UI', 9))
        lbl_graph.pack(anchor='w', padx=10, pady=(5, 0))
        self.canvas_height = 100
        self.canvas_width = 380
        self.canvas = tk.Canvas(self, height=self.canvas_height, width=self.canvas_width, bg=self.colors['graph_bg'], highlightthickness=0)
        self.canvas.pack(padx=10, pady=5)
        self.draw_grid()

        # Log
        lbl_log = ttk.Label(self, text="Log de Eventos:", font=('Segoe UI', 9))
        lbl_log.pack(anchor='w', padx=10, pady=(5, 0))
        self.txt_log = scrolledtext.ScrolledText(self, height=8, font=('Consolas', 8), bg='#1e1e1e', fg='#ccc', borderwidth=0)
        self.txt_log.pack(fill='both', expand=True, padx=10, pady=5)
        self.txt_log.tag_config('error', foreground=self.colors['red'])
        self.txt_log.tag_config('success', foreground=self.colors['green'])
        self.txt_log.tag_config('warning', foreground=self.colors['yellow'])
        self.txt_log.tag_config('info', foreground='#888')

    def toggle_topmost(self):
        """Alterna modo 'Sempre no topo'"""
        self.attributes('-topmost', self.var_top.get())

    def play_sound(self, tipo):
        """Toca sons do sistema"""
        if not self.var_sound.get(): return
        
        try:
            # Tenta usar o 'beep' do sistema (Funciona na maioria dos terminais Linux)
            if tipo == "error":
                self.bell() # Som padrão
            elif tipo == "success":
                self.bell()
        except:
            pass

    def send_notification(self, title, message, urgency="normal"):
        """Envia notificação nativa do Linux (libnotify)"""
        if not self.var_notify.get(): return
        
        try:
            # Ícones padrão do Linux Mint/Ubuntu
            icon = "network-error" if urgency == "critical" else "network-transmit-receive"
            subprocess.run(['notify-send', '-u', urgency, '-i', icon, title, message])
        except FileNotFoundError:
            self.log_message("Erro", "Instale 'libnotify-bin' para ver notificações", "warning")

    def draw_grid(self):
        self.canvas.delete("grid")
        y_100 = self.canvas_height - (100 / 200 * self.canvas_height) 
        self.canvas.create_line(0, y_100, self.canvas_width, y_100, fill='#333', dash=(2, 4), tags="grid")

    def update_graph(self):
        self.canvas.delete("line")
        points = []
        w, h = self.canvas_width, self.canvas_height
        num = len(self.ping_history)
        step = w / (num - 1) if num > 1 else w
        max_vis = 300 

        for i, val in enumerate(self.ping_history):
            x = i * step
            if val == -1:
                y = 0; color = self.colors['red']
            else:
                norm = min(val, max_vis)
                y = h - (norm / max_vis * h)
                color = self.colors['line'] if val <= PING_ALTO_THRESHOLD else self.colors['yellow']
            
            points.extend([x, y])
            self.canvas.create_oval(x-1, y-1, x+1, y+1, fill=color, outline=color, tags="line")

        if len(points) >= 4:
            self.canvas.create_line(points, fill=self.colors['line'], width=2, tags="line", smooth=True)

    def setup_database(self):
        try:
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute('''CREATE TABLE IF NOT EXISTS ping_logs (id INTEGER PRIMARY KEY AUTOINCREMENT, data_hora DATETIME, status TEXT, ping REAL, ping_anterior REAL)''')
            conn.commit(); conn.close()
        except Exception as e: messagebox.showerror("Erro de BD", str(e))

    def log_event_db(self, status, ping_atual=None):
        try:
            conn = sqlite3.connect(DB_NAME); cursor = conn.cursor()
            cursor.execute('''INSERT INTO ping_logs (data_hora, status, ping, ping_anterior) VALUES (?, ?, ?, ?)''', 
                          (datetime.now().strftime('%Y-%m-%d %H:%M:%S'), status, ping_atual, self.last_ping))
            conn.commit(); conn.close()
        except: pass 

    def log_message(self, source, message, tag='info'):
        ts = datetime.now().strftime('%H:%M:%S')
        self.txt_log.insert(tk.END, f"[{ts}] {message}\n", tag)
        self.txt_log.see(tk.END)

    def update_status_display(self, state, ping_ms=None):
        if state == "offline":
            self.lbl_status_text.config(text="OFFLINE", fg=self.colors['red'])
            self.lbl_ping_val.config(text="TIMEOUT", fg=self.colors['red'])
            self.ping_history.append(-1)
        elif state == "online":
            self.lbl_status_text.config(text="ONLINE", fg=self.colors['green'])
            self.lbl_ping_val.config(text=f"{ping_ms} ms", fg=self.colors['green'])
            self.ping_history.append(ping_ms)
        elif state == "unstable":
            self.lbl_status_text.config(text="INSTÁVEL", fg=self.colors['yellow'])
            self.lbl_ping_val.config(text=f"{ping_ms} ms", fg=self.colors['yellow'])
            self.ping_history.append(ping_ms)
        elif state == "stopped":
            self.lbl_status_text.config(text="PARADO", fg='#999')
            self.lbl_ping_val.config(text="-- ms", fg='#999')
            self.ping_history.clear(); self.ping_history.extend([0]*40)
        self.update_graph()

    def get_system_ping(self, host):
        """Ping otimizado para Linux"""
        # -c 1: Envia apenas 1 pacote
        # -W 1: Espera no máximo 1 segundo pela resposta
        cmd = ['ping', '-c', '1', '-W', '1', host]
        try:
            out = subprocess.check_output(cmd, stderr=subprocess.STDOUT).decode('utf-8', errors='ignore')
            # Regex para capturar o tempo (time=14.5 ms)
            match = re.search(r'time=([\d\.]+)', out)
            return float(match.group(1)) if match else 0.0
        except subprocess.CalledProcessError:
            return None # Offline
        except Exception:
            return None

    def toggle_monitoring(self):
        if not self.monitoring:
            target = self.entry_target.get().strip()
            if not target: messagebox.showwarning("Aviso", "Digite um IP válido."); return
            
            self.current_target = target
            self.monitoring = True
            self.btn_action.config(text="PARAR MONITORAMENTO")
            self.entry_target.config(state='disabled')
            self.log_message("Sistema", f"Monitorando: {self.current_target}", "info")
            self.thread = threading.Thread(target=self.run_monitor, daemon=True); self.thread.start()
        else:
            self.monitoring = False
            self.btn_action.config(text="INICIAR MONITORAMENTO")
            self.entry_target.config(state='normal')
            self.log_message("Sistema", "Parando...", "info")
            self.update_status_display("stopped")

    def run_monitor(self):
        while self.monitoring:
            try:
                r = None
                # Tenta usar o ping3 (requer sudo), se falhar usa o ping do sistema
                if not self.use_system_ping:
                    try: r = ping(self.current_target, timeout=TIMEOUT_PING, unit='ms')
                    except OSError:
                        self.use_system_ping = True
                        self.after(0, self.log_message, "Aviso", "Sem Root. Usando 'ping' do Linux.", "warning")
                        continue 
                else: r = self.get_system_ping(self.current_target)

                if r is None or r is False: # OFFLINE
                    if not self.is_offline:
                        self.is_offline = True
                        self.log_event_db("internet_fall")
                        self.after(0, self.log_message, "Rede", "🚨 Conexão Perdida!", "error")
                        # Ações extras
                        self.after(0, self.send_notification, "Internet Caiu!", f"Sem conexão com {self.current_target}", "critical")
                        self.after(0, self.play_sound, "error")
                    self.after(0, self.update_status_display, "offline")
                else: # ONLINE
                    ping_ms = round(r, 1)
                    if self.is_offline:
                        self.is_offline = False
                        self.log_event_db("reconnection", ping_ms)
                        self.after(0, self.log_message, "Rede", f"✅ Reconectado! ({ping_ms}ms)", "success")
                        # Ações extras
                        self.after(0, self.send_notification, "Internet Voltou!", f"Conexão restabelecida. Ping: {ping_ms}ms", "normal")
                        self.after(0, self.play_sound, "success")
                    elif ping_ms > PING_ALTO_THRESHOLD:
                        self.log_event_db("ping_fall", ping_ms)
                        self.after(0, self.log_message, "Rede", f"⚠️ Latência Alta: {ping_ms}ms", "warning")
                        self.after(0, self.update_status_display, "unstable", ping_ms)
                    else:
                        self.after(0, self.update_status_display, "online", ping_ms)
                    self.last_ping = ping_ms
            except Exception as e:
                self.after(0, self.log_message, "Erro", str(e), "error"); time.sleep(5)
            time.sleep(INTERVALO)

if __name__ == "__main__":
    app = WifiMonitorGUI()
    app.mainloop()
