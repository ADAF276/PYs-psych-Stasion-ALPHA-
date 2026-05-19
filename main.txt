import sys
import math
import os
import tkinter as tk
from tkinter import filedialog, messagebox
from datetime import datetime
import subprocess
import threading  
import time  


import pymunk
pymunk.inf = math.inf

import PIL.ImageDraw
if not hasattr(PIL.ImageDraw.ImageDraw, 'multiline_textsize'):
    def multiline_textsize_palsu(self, text, font=None, spacing=4, direction=None, features=None):
        bbox = self.multiline_textbbox((0, 0), text, font=font, spacing=spacing, direction=direction, features=features)
        return (bbox[2] - bbox[0], bbox[3] - bbox[1])
    PIL.ImageDraw.ImageDraw.multiline_textsize = multiline_textsize_palsu

import arcade

SCREEN_WIDTH = 1280
SCREEN_HEIGHT = 720
SCREEN_TITLE = "PsychStation 5 - Psych Engine Standalone Edition"

def draw_rounded_rect(center_x, center_y, width, height, color, radius, outline=False, outline_width=2):
    points = []
    half_w = width / 2
    half_h = height / 2
    steps = 4 
    
    for i in range(steps + 1):
        ang = math.radians(0 + (i * 90 / steps))
        points.append((center_x + half_w - radius + radius * math.cos(ang), center_y + half_h - radius + radius * math.sin(ang)))
    for i in range(steps + 1):
        ang = math.radians(90 + (i * 90 / steps))
        points.append((center_x - half_w + radius + radius * math.cos(ang), center_y + half_h - radius + radius * math.sin(ang)))
    for i in range(steps + 1):
        ang = math.radians(180 + (i * 90 / steps))
        points.append((center_x - half_w + radius + radius * math.cos(ang), center_y - half_h + radius + radius * math.sin(ang)))
    for i in range(steps + 1):
        ang = math.radians(270 + (i * 90 / steps))
        points.append((center_x + half_w - radius + radius * math.cos(ang), center_y - half_h + radius + radius * math.sin(ang)))

    if outline:
        arcade.draw_polygon_outline(points, color, outline_width)
    else:
        arcade.draw_polygon_filled(points, color)


class PsychStationPS5(arcade.Window):
    def __init__(self):
        # Tambahkan konfigurasi agar window bisa di-resize dan mendukung fullscreen
        super().__init__(SCREEN_WIDTH, SCREEN_HEIGHT, SCREEN_TITLE, resizable=True)
        
        self.tk_root = tk.Tk()
        self.tk_root.withdraw()
        
        
        self.state = "LOADING"       
        self.loading_time = 0.0      
        self.loading_angle = 0       
        
        # Variabel Transisi Fade
        self.fade_alpha = 0          
        self.fade_direction = "FADE_OUT_LOADING_AWAL" 
        
        
        self.press_delay_timer = 0.0
        self.is_pressing = False
        
        
        self.target_exe_name = ""    
        self.game_is_running = False 
        
       
        self.mod_list = [
            {
                "title": "Add Game",
                "desc": "Click or press Enter to import your game folder.",
                "logo": None,
                "exe_path": "", 
                "folder_path": "",
                "is_button": True,
                "color": (45, 48, 65)
            }
        ]
        
        self.current_selection = 0
        self.scroll_x = 0
        self.target_scroll_x = 0
        self.card_scale = 1.0
        
        self.bg_color_top = (15, 18, 28)
        self.bg_color_bottom = (4, 5, 8)
        
        self.base_card_w = 180       
        self.base_card_h = 180       
        self.card_radius = 25   
        self.spacing = 30
        self.start_y = 160

    def background_launcher_thread(self, exe_path, folder_path, exe_name):
        try:
            subprocess.Popen([exe_path], cwd=folder_path, shell=True)
            
            
            while True:
                output = os.popen(f'tasklist /FI "IMAGENAME eq {exe_name}"').read()
                if exe_name.lower() in output.lower():
                    self.game_is_running = True 
                    break
                time.sleep(0.2)
        except Exception as e:
            print(f"Error thread background: {e}")
            self.game_is_running = True 

    def on_draw(self):
        self.clear()
        
        
        width, height = self.get_size()
        cx, cy = width // 2, height // 2
        
        # ==========================================
        # LAYAR 1: INTRO LOADING SCREEN (BOOTING AWAL)
        # ==========================================
        if self.state == "LOADING":
            arcade.draw_rectangle_filled(cx, cy, width, height, (8, 10, 15))
            
            for dx in [-2, 0, 2]:
                for dy in [-2, 0, 2]:
                    arcade.draw_text("PYs", cx + dx, cy + 30 + dy, arcade.color.WHITE, 55, font_name="arial", bold=True, anchor_x="center", anchor_y="center")
            
            arcade.draw_arc_outline(cx, cy - 60, 40, 40, arcade.color.WHITE, self.loading_angle, self.loading_angle + 270, border_width=4)
            
            if self.fade_alpha > 0:
                arcade.draw_rectangle_filled(cx, cy, width, height, (8, 10, 15, int(self.fade_alpha)))
            return 


        if self.state == "MENU":
            for y in range(0, height, 4):
                pct = y / height
                r = int(self.bg_color_bottom[0] + (self.bg_color_top[0] - self.bg_color_bottom[0]) * pct)
                g = int(self.bg_color_bottom[1] + (self.bg_color_top[1] - self.bg_color_bottom[1]) * pct)
                b = int(self.bg_color_bottom[2] + (self.bg_color_top[2] - self.bg_color_bottom[2]) * pct)
                arcade.draw_rectangle_filled(cx, y + 2, width, 4, (r, g, b))
                
            arcade.draw_text("Games", 60, height - 50, arcade.color.WHITE, 16, bold=True)
            arcade.draw_text("Media", 140, height - 50, arcade.color.LIGHT_GRAY, 16)
            
            waktu_sekarang = datetime.now().strftime("%I:%M %p")
            arcade.draw_text(waktu_sekarang, width - 120, height - 50, arcade.color.WHITE, 16)
            
            selected_mod = self.mod_list[self.current_selection]
            arcade.draw_text(selected_mod["title"], 60, cy + 40, arcade.color.WHITE, 36, bold=True)
            arcade.draw_text(selected_mod["desc"], 60, cy - 10, arcade.color.LIGHT_GRAY, 14)
            
            if "is_button" in selected_mod:
                arcade.draw_text("Import Game / Press Enter Or Click", 60, cy - 40, (112, 128, 144), 12, bold=True)
            else:
                arcade.draw_text("Start Game / Press Enter Or Click", 60, cy - 40, (112, 128, 144), 12, bold=True)
            
            for i, mod in enumerate(self.mod_list):
                mod["render_x"] = 140 + (i * (self.base_card_w + self.spacing)) + self.scroll_x
                
                if i == self.current_selection:
                    current_w = self.base_card_w * self.card_scale
                    current_h = self.base_card_h * self.card_scale
                    current_radius = self.card_radius * self.card_scale
                    
                    draw_rounded_rect(center_x=mod["render_x"], center_y=self.start_y, width=current_w + 14, height=current_h + 14, color=arcade.color.WHITE, radius=current_radius + 4, outline=True, outline_width=4)
                    opacity = 255
                else:
                    current_w = self.base_card_w
                    current_h = self.base_card_h
                    current_radius = self.card_radius
                    opacity = 140
                    
                c = mod["color"]
                warna_kartu = (c[0], c[1], c[2], opacity)
                
                draw_rounded_rect(center_x=mod["render_x"], center_y=self.start_y, width=current_w, height=current_h, color=warna_kartu, radius=current_radius, outline=False)
                draw_rounded_rect(center_x=mod["render_x"], center_y=self.start_y, width=current_w, height=current_h, color=(255, 255, 255, 40), radius=current_radius, outline=True, outline_width=2)
                
                if "is_button" in mod:
                    ukuran_tambah = int(55 * (self.card_scale if i == self.current_selection else 1.0))
                    arcade.draw_text("+", mod["render_x"], self.start_y, (180, 190, 210), ukuran_tambah, font_name="arial", anchor_x="center", anchor_y="center")
                else:
                    ukuran_tanya = int(50 * (self.card_scale if i == self.current_selection else 1.0))
                    if mod["logo"]:
                        try:
                            arcade.draw_texture_rectangle(mod["render_x"], self.start_y, current_w - 16, current_h - 16, mod["logo"])
                        except Exception:
                            arcade.draw_text("?", mod["render_x"], self.start_y, arcade.color.WHITE, ukuran_tanya, font_name="arial", bold=True, anchor_x="center", anchor_y="center")
                    else:
                        arcade.draw_text("?", mod["render_x"], self.start_y, arcade.color.WHITE, ukuran_tanya, font_name="arial", bold=True, anchor_x="center", anchor_y="center")

            if self.fade_alpha > 0:
                arcade.draw_rectangle_filled(cx, cy, width, height, (8, 10, 15, int(self.fade_alpha)))
            return


        if self.state == "LAUNCHING":
            arcade.draw_rectangle_filled(cx, cy, width, height, (5, 6, 10))
            
            # Spinner Muter Bersih Tanpa Text PYs
            arcade.draw_arc_outline(
                center_x=cx, center_y=cy, 
                width=60, height=60, color=arcade.color.WHITE, 
                start_angle=self.loading_angle, end_angle=self.loading_angle + 270, 
                border_width=5
            )
            
            if self.fade_alpha > 0:
                arcade.draw_rectangle_filled(cx, cy, width, height, (5, 6, 10, int(self.fade_alpha)))
            return

    def on_update(self, delta_time):
        self.loading_angle -= 6 

        # 1. LOGIKA BOOTING AWAL
        if self.state == "LOADING":
            self.loading_time += delta_time
            if self.loading_time >= 2.2 and self.fade_direction == "FADE_OUT_LOADING_AWAL":
                self.fade_alpha += (255 - self.fade_alpha) * 0.15
                if self.fade_alpha > 250:
                    self.fade_alpha = 255
                    self.state = "MENU" 
                    self.fade_direction = "FADE_IN_HOME_AWAL"
            return 

        if self.state == "MENU" and self.fade_direction == "FADE_IN_HOME_AWAL":
            self.fade_alpha += (0 - self.fade_alpha) * 0.12
            if self.fade_alpha < 5:
                self.fade_alpha = 0
                self.fade_direction = "NONE"


        if self.state == "MENU" and self.is_pressing:
            self.press_delay_timer += delta_time
            if self.press_delay_timer >= 0.15: 
                self.is_pressing = False
                self.press_delay_timer = 0.0
                self.fade_direction = "MENUTUP_MENU_MENUJU_LAUNCH"


        if self.state == "MENU" and self.fade_direction == "MENUTUP_MENU_MENUJU_LAUNCH":
            self.fade_alpha += (255 - self.fade_alpha) * 0.15
            if self.fade_alpha > 250:
                self.fade_alpha = 255
                self.state = "LAUNCHING"  
                self.fade_direction = "MUNCULKAN_SPINNER_LAUNCH"
                
                selected = self.mod_list[self.current_selection]
                self.target_exe_name = os.path.basename(selected["exe_path"])
                self.game_is_running = False
                
                t = threading.Thread(target=self.background_launcher_thread, args=(selected["exe_path"], selected["folder_path"], self.target_exe_name))
                t.daemon = True
                t.start()

        if self.state == "LAUNCHING":
            if self.fade_alpha > 0 and self.fade_direction == "MUNCULKAN_SPINNER_LAUNCH":
                self.fade_alpha += (0 - self.fade_alpha) * 0.15
                if self.fade_alpha < 5:
                    self.fade_alpha = 0
            
            # Jika background thread membaca game .exe aktif -> Picu Fade Out
            if self.game_is_running and self.fade_direction == "MUNCULKAN_SPINNER_LAUNCH":
                self.fade_direction = "MENUTUP_LAUNCH_MENUJU_HOME"

            if self.fade_direction == "MENUTUP_LAUNCH_MENUJU_HOME":
                self.fade_alpha += (255 - self.fade_alpha) * 0.15
                if self.fade_alpha > 250:
                    self.fade_alpha = 255
                    self.state = "MENU"
                    self.fade_direction = "BERSIHKAN_LAYAR_DASHBOARD"
            return

        if self.state == "MENU" and self.fade_direction == "BERSIHKAN_LAYAR_DASHBOARD":
            self.fade_alpha += (0 - self.fade_alpha) * 0.12
            if self.fade_alpha < 5:
                self.fade_alpha = 0
                self.fade_direction = "NONE"

        self.scroll_x += (self.target_scroll_x - self.scroll_x) * 0.15
        
        if self.card_scale < 1.0 and not self.is_pressing:
            self.card_scale += (1.0 - self.card_scale) * 0.15
            if self.card_scale > 0.99:
                self.card_scale = 1.0

    def on_key_press(self, key, modifiers):
        # FITUR BARU: Deteksi Tombol F11 untuk gonta-ganti Layar Fullscreen / Windowed
        if key == arcade.key.F11:
            self.set_fullscreen(not self.fullscreen)
            return

        if self.state in ["LOADING", "LAUNCHING"] or self.is_pressing or self.fade_direction != "NONE":
            return 
            
        if key == arcade.key.RIGHT:
            if self.current_selection < len(self.mod_list) - 1:
                self.current_selection += 1
                self.target_scroll_x = -self.current_selection * (self.base_card_w + self.spacing)
                self.card_scale = 1.0
                    
        elif key == arcade.key.LEFT:
            if self.current_selection > 0:
                self.current_selection -= 1
                self.target_scroll_x = -self.current_selection * (self.base_card_w + self.spacing)
                self.card_scale = 1.0
                    
        elif key == arcade.key.ENTER:
            self.eksekusi_klik_kartu()

    def on_mouse_press(self, x, y, button, modifiers):
        if self.state in ["LOADING", "LAUNCHING"] or self.is_pressing or self.fade_direction != "NONE":
            return 
            
        if button == 1:
            for i, mod in enumerate(self.mod_list):
                if "render_x" in mod:
                    kiri = mod["render_x"] - (self.base_card_w / 2)
                    kanan = mod["render_x"] + (self.base_card_w / 2)
                    bawah = self.start_y - (self.base_card_h / 2)
                    atas = self.start_y + (self.base_card_h / 2)
                    
                    if kiri <= x <= kanan and bawah <= y <= atas:
                        self.current_selection = i
                        self.target_scroll_x = -self.current_selection * (self.base_card_w + self.spacing)
                        self.eksekusi_klik_kartu()
                        return

    def eksekusi_klik_kartu(self):
        selected = self.mod_list[self.current_selection]
        if "is_button" in selected:
            self.pilih_folder_game()
        else:
            self.card_scale = 0.82
            self.is_pressing = True
            self.press_delay_timer = 0.0

    def pilih_folder_game(self):
        folder_terpilih = filedialog.askdirectory(title="Select Psych Engine Game Folder")
        
        if folder_terpilih:
            path_folder_mods = os.path.join(folder_terpilih, "mods")
            if not os.path.exists(path_folder_mods) or not os.path.isdir(path_folder_mods):
                messagebox.showerror("Import Failed!", "Missing 'mods' folder inside!")
                return

            file_exe_target = None
            for file in os.listdir(folder_terpilih):
                if file.endswith(".exe") and "crash" not in file.lower():
                    file_exe_target = os.path.join(folder_terpilih, file)
                    break
            
            if not file_exe_target:
                messagebox.showerror("Import Failed!", "No game executable found!")
                return

            nama_folder = os.path.basename(folder_terpilih)
            kemungkinan_nama_logo = ['icon.ico', 'icon.png', 'pack.png', 'logo.png', 'thumbnail.png']
            found_texture = None
            
            for nama_file in kemungkinan_nama_logo:
                path_gambar = os.path.join(folder_terpilih, nama_file)
                if os.path.exists(path_gambar):
                    try:
                        found_texture = arcade.load_texture(path_gambar)
                        break
                    except Exception:
                        pass
            
            warna_kartu_baru = (70, 83, 99)
            
            data_game_baru = {
                "title": nama_folder.upper(),
                "desc": f"Engine Executable: {os.path.basename(file_exe_target)}\nPath: {folder_terpilih}",
                "logo": found_texture,
                "exe_path": file_exe_target,     
                "folder_path": folder_terpilih,  
                "color": warna_kartu_baru
            }
            
            self.mod_list.insert(len(self.mod_list) - 1, data_game_baru)
            self.current_selection = len(self.mod_list) - 2
            self.target_scroll_x = -self.current_selection * (self.base_card_w + self.spacing)

if __name__ == "__main__":
    app = PsychStationPS5()
    arcade.run()