# WIFIMONITOR.PY
Monitoramento wifi

import tkinter as tk
from tkinter import ttk, scrolledtext, messagebox
import sqlite3
import time
import threading
import subprocess
import platform
import re
from datetime import datetime
from collections import deque
from ping3 import ping  # pip install ping3

# --- CONFIGURAÇÕES ---
DB_NAME = "monitoramento.db"
ALVO_PING = "8.8.8.8"  # Google DNS
INTERVALO = 1          # Reduzi para 1s para o gráfico ficar mais fluido
TIMEOUT_PING = 2       # Segundos
PING_ALTO_THRESHOLD = 150 # Ms

class WifiMonitorGUI(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Monitor de Rede - Gravador")
        self.geometry("420x650") # Aumentei um pouco a altura
        self.resizable(False, False)
        
        # Variáveis de Controle
        self.monitoring = False
        self.thread = None
        self.last_ping = 0.0
        self.is_offline = False
        self.use_system_ping = False 

        # Dados para o Gráfico (Histórico dos últimos 40 pontos)
        self.ping_history = deque([0]*40, maxlen=40)

        # Configuração de Estilo (Dark Mode)
        self.colors = {
            'bg': '#2b2b2b', 'fg': '#ffffff',
            'panel': '#3c3f41', 'green': '#5cb85c',
            'red': '#d9534f', 'yellow': '#f0ad4e',
            'graph_bg': '#1e1e1e', 'line': '#00ffff'
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

    def setup_ui(self):
        # --- Painel Principal de Status ---
        self.status_frame = ttk.Frame(self, style='Status.TFrame')
        self.status_frame.pack(fill='x', padx=10, pady=10)

        self.lbl_status_text = tk.Label(self.status_frame, text="PARADO", 
                                        font=('Segoe UI', 20, 'bold'),
                                        bg=self.colors['panel'], fg='#999')
        self.lbl_status_text.pack(pady=(10, 2))

        self.lbl_ping_val = tk.Label(self.status_frame, text="-- ms", 
                                     font=('Segoe UI', 30, 'bold'),
                                     bg=self.colors['panel'], fg='#999')
        self.lbl_ping_val.pack(pady=(0, 10))

        # --- Botão de Ação ---
        self.btn_action = ttk.Button(self, text="INICIAR MONITORAMENTO", command=self.toggle_monitoring)
        self.btn_action.pack(fill='x', padx=20, pady=5, ipady=5)

        # --- TELA PING (Gráfico) ---
        lbl_graph = ttk.Label(self, text="Visualização em Tempo Real:", font=('Segoe UI', 9))
        lbl_graph.pack(anchor='w', padx=10, pady=(10, 0))

        self.canvas_height = 100
        self.canvas_width = 380
        self.canvas = tk.Canvas(self, height=self.canvas_height, width=self.canvas_width, 
                                bg=self.colors['graph_bg'], highlightthickness=0)
        self.canvas.pack(padx=10, pady=5)
        
        # Linhas de grade do gráfico
        self.draw_grid()

        # --- Log Visual ---
        lbl_log = ttk.Label(self, text="Log de Eventos:", font=('Segoe UI', 9))
        lbl_log.pack(anchor='w', padx=10, pady=(10, 0))

        self.txt_log = scrolledtext.ScrolledText(self, height=10, font=('Consolas', 8),
                                                 bg='#1e1e1e', fg='#ccc', borderwidth=0)
        self.txt_log.pack(fill='both', expand=True, padx=10, pady=5)
        self.txt_log.tag_config('error', foreground=self.colors['red'])
        self.txt_log.tag_config('success', foreground=self.colors['green'])
        self.txt_log.tag_config('warning', foreground=self.colors['yellow'])
        self.txt_log.tag_config('info', foreground='#888')

    def draw_grid(self):
        """Desenha linhas de referência no gráfico"""
        self.canvas.delete("grid")
        # Linha de 100ms (Limite aceitável)
        y_100 = self.canvas_height - (100 / 200 * self.canvas_height) 
        self.canvas.create_line(0, y_100, self.canvas_width, y_100, fill='#333', dash=(2, 4), tags="grid")

    def update_graph(self):
        """Redesenha o gráfico com base no self.ping_history"""
        self.canvas.delete("line")
        
        points = []
        w = self.canvas_width
        h = self.canvas_height
        num_points = len(self.ping_history)
        step = w / (num_points - 1) if num_points > 1 else w

        # Normalização: Assumimos max ping visual de 300ms para o gráfico não ficar achatado
        max_visual_ping = 300 

        for i, ping_val in enumerate(self.ping_history):
            x = i * step
            
            if ping_val == -1: # Código para Timeout/Offline
                y = 0 # Topo do gráfico (ou poderia ser base, mas topo chama atenção)
                color = self.colors['red']
            else:
                # Inverter Y (0 é topo no Canvas)
                # Se ping > 300, ele bate no teto
                normalized_ping = min(ping_val, max_visual_ping)
                y = h - (normalized_ping / max_visual_ping * h)
                color = self.colors['line']
                
                if ping_val > PING_ALTO_THRESHOLD: color = self.colors['yellow']
            
            points.append(x)
            points.append(y)
            
            # Desenha ponto individual (opcional, mas ajuda a ver)
            self.canvas.create_oval(x-1, y-1, x+1, y+1, fill=color, outline=color, tags="line")

        if len(points) >= 4:
            # Desenha a linha conectando os pontos
            self.canvas.create_line(points, fill=self.colors['line'], width=2, tags="line", smooth=True)

    def setup_database(self):
        try:
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS ping_logs (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    data_hora DATETIME,
                    status TEXT,
                    ping REAL,
                    ping_anterior REAL
                )
            ''')
            conn.commit()
            conn.close()
            self.log_message("Sistema", "Banco de dados conectado.", "info")
        except Exception as e:
            messagebox.showerror("Erro de BD", str(e))

    def log_event_db(self, status, ping_atual=None):
        try:
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            data_atual = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            cursor.execute('''
                INSERT INTO ping_logs (data_hora, status, ping, ping_anterior)
                VALUES (?, ?, ?, ?)
            ''', (data_atual, status, ping_atual, self.last_ping))
            conn.commit()
            conn.close()
        except Exception:
            pass 

    def log_message(self, source, message, tag='info'):
        timestamp = datetime.now().strftime('%H:%M:%S')
        self.txt_log.insert(tk.END, f"[{timestamp}] {message}\n", tag)
        self.txt_log.see(tk.END)

    def update_status_display(self, state, ping_ms=None):
        if state == "offline":
            self.lbl_status_text.config(text="OFFLINE", fg=self.colors['red'])
            self.lbl_ping_val.config(text="TIMEOUT", fg=self.colors['red'])
            self.ping_history.append(-1) # -1 representa falha no gráfico
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
            self.ping_history.clear()
            self.ping_history.extend([0]*40)
        
        self.update_graph()

    def toggle_monitoring(self):
        if not self.monitoring:
            self.monitoring = True
            self.btn_action.config(text="PARAR MONITORAMENTO")
            self.log_message("Sistema", "Monitoramento Iniciado...", "info")
            self.thread = threading.Thread(target=self.run_monitor, daemon=True)
            self.thread.start()
        else:
            self.monitoring = False
            self.btn_action.config(text="INICIAR MONITORAMENTO")
            self.log_message("Sistema", "Parando...", "info")
            self.update_status_display("stopped")

    def get_system_ping(self, host):
        param = '-n' if platform.system().lower() == 'windows' else '-c'
        timeout_param = '-w' if platform.system().lower() == 'windows' else '-W'
        timeout_val = '1000' if platform.system().lower() == 'windows' else '1'
        command = ['ping', param, '1', timeout_param, timeout_val, host]
        try:
            creation_flags = 0
            if platform.system().lower() == 'windows':
                creation_flags = subprocess.CREATE_NO_WINDOW
            output = subprocess.check_output(command, stderr=subprocess.STDOUT, creationflags=creation_flags).decode('utf-8', errors='ignore')
            match = re.search(r'(tempo|time)[=<]\s*([\d\.]+)', output, re.IGNORECASE)
            if match: return float(match.group(2))
            return 0.0 
        except subprocess.CalledProcessError:
            return None 
        except Exception:
            return None

    def run_monitor(self):
        while self.monitoring:
            try:
                r = None
                if not self.use_system_ping:
                    try:
                        r = ping(ALVO_PING, timeout=TIMEOUT_PING, unit='ms')
                    except OSError:
                        self.use_system_ping = True
                        self.after(0, self.log_message, "Aviso", "Modo Admin falhou. Usando Ping do Sistema.", "warning")
                        continue 
                else:
                    r = self.get_system_ping(ALVO_PING)

                # Lógica de Análise
                if r is None or r is False:
                    if not self.is_offline:
                        self.is_offline = True
                        self.log_event_db("internet_fall")
                        self.after(0, self.log_message, "Rede", "🚨 Conexão Perdida!", "error")
                    self.after(0, self.update_status_display, "offline")
                else:
                    ping_ms = round(r, 1)
                    if self.is_offline:
                        self.is_offline = False
                        self.log_event_db("reconnection", ping_ms)
                        self.after(0, self.log_message, "Rede", f"✅ Reconectado! ({ping_ms}ms)", "success")
                    elif ping_ms > PING_ALTO_THRESHOLD:
                        self.log_event_db("ping_fall", ping_ms)
                        self.after(0, self.log_message, "Rede", f"⚠️ Latência Alta: {ping_ms}ms", "warning")
                        self.after(0, self.update_status_display, "unstable", ping_ms)
                    else:
                        self.after(0, self.update_status_display, "online", ping_ms)
                    
                    self.last_ping = ping_ms
            except Exception as e:
                self.after(0, self.log_message, "Erro", str(e), "error")
                time.sleep(5)

            time.sleep(INTERVALO)

if __name__ == "__main__":
    app = WifiMonitorGUI()
    app.mainloop()


