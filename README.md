import base64
import io
import os
import json
import threading
import logging
from socket import socket, AF_INET, SOCK_STREAM

from customtkinter import *
from tkinter import filedialog
from PIL import Image

logging.basicConfig(level=logging.INFO)

THEME_FILE = "theme_config.json"

LIGHT_THEME = {
    "bg": "white",
    "menu_bg": "lightyellow",
    "chat_fg": "lavender",
    "text": "black",
    "author": "#888be5",
    "button": "pink"
}
DARK_THEME = {
    "bg": "#23272e",
    "menu_bg": "#30363d",
    "chat_fg": "#2c2f38",
    "text": "white",
    "author": "#ffe966",
    "button": "#873b77"
}

class NetworkClient:
    def __init__(self, host, port, username, on_receive):
        self.host = host
        self.port = port
        self.username = username
        self.on_receive = on_receive
        self.sock = None
        self.running = False

    def connect(self):
        try:
            self.sock = socket(AF_INET, SOCK_STREAM)
            self.sock.connect((self.host, self.port))
            hello = f"TEXT@{self.username}@[SYSTEM] {self.username} приєднався(лась) до чату!\n"
            self.sock.send(hello.encode('utf-8'))
            self.running = True
            threading.Thread(target=self.recv_thread, daemon=True).start()
            logging.info("Підключено до сервера.")
        except Exception as e:
            logging.error(f"Не вдалося підключитися: {e}")
            self.on_receive(f"Не вдалося підключитися до сервера: {e}")

    def send(self, data):
        try:
            if self.sock and self.running:
                self.sock.sendall(data.encode())
        except Exception as e:
            logging.error(f"Помилка надсилання: {e}")

    def close(self):
        self.running = False
        if self.sock:
            try:
                self.sock.close()
                logging.info("З'єднання закрите.")
            except Exception as e:
                logging.error(f"Помилка закриття: {e}")

    def recv_thread(self):
        buffer = ""
        while self.running:
            try:
                chunk = self.sock.recv(4096)
                if not chunk:
                    break
                buffer += chunk.decode('utf-8', errors='ignore')
                while "\n" in buffer:
                    line, buffer = buffer.split("\n", 1)
                    self.on_receive(line.strip())
            except Exception as e:
                logging.error(f"Помилка прийому: {e}")
                break
        self.close()

class ThemeManager:
    """Збереження й завантаження теми"""
    def __init__(self):
        self.theme = LIGHT_THEME
        self.load_theme()

    def load_theme(self):
        if os.path.exists(THEME_FILE):
            try:
                with open(THEME_FILE, "r", encoding="utf-8") as f:
                    config = json.load(f)
                    if config.get("theme") == "dark":
                        self.theme = DARK_THEME
            except Exception:
                pass

    def set_theme(self, mode):
        if mode == "light":
            self.theme = LIGHT_THEME
        else:
            self.theme = DARK_THEME
        try:
            with open(THEME_FILE, "w", encoding="utf-8") as f:
                json.dump({"theme": mode}, f)
        except Exception:
            pass

    def get_theme(self):
        return self.theme

class MainWindow(CTk):
    """
    Чат-клієнт з перемиканням теми (нічний/день режим).
    """
    USER_COLORS = ["#80b7f0", "#e7af33", "#7be7a8", "#f6a5d3", "#ed8867"]

    def __init__(self):
        super().__init__()
        self.theme_manager = ThemeManager()
        self.theme = self.theme_manager.get_theme()
        self.geometry('820x600')
        self.title("PyChat Client")
        self.username = "настя"
        self.users_colors = {}

        # GUI елементи
        self.menu_frame = CTkFrame(self, width=30, height=300, fg_color=self.theme["menu_bg"])
        self.menu_frame.pack_propagate(False)
        self.menu_frame.place(x=0, y=0)
        self.is_show_menu = False
        self.speed_animate_menu = -20
        self.btn = CTkButton(self, text='▶️', fg_color=self.theme["button"], hover_color=self.theme["menu_bg"],
                             command=self.toggle_show_menu, width=30)
        self.btn.place(x=0, y=0)

        # Темний/світлий режим
        self.theme_button = CTkButton(self, text="🌞/🌚", fg_color="#bbbbbb", command=self.toggle_theme, width=30)
        self.theme_button.place(x=40, y=0)

        # Поле чату
        self.chat_field = CTkScrollableFrame(self, fg_color=self.theme["bg"])
        self.chat_field.place(x=0, y=0)

        # Ввід та кнопки
        self.message_entry = CTkEntry(self, placeholder_text='Введіть повідомлення:', height=40)
        self.message_entry.place(x=0, y=0)
        self.send_button = CTkButton(self, text='🠖', width=50, height=40, fg_color=self.theme["button"],
                                    command=self.send_message)
        self.send_button.place(x=0, y=0)
        self.open_img_button = CTkButton(self, text='📂', fg_color="blue", hover_color=self.theme["button"],
                                         width=50, height=40, command=self.open_image)
        self.open_img_button.place(x=0, y=0)
        self.after(50, self.adaptive_ui)

        self.add_message("Демонстрація зображення", CTkImage(Image.open('5713_1.jpg'), size=(180, 180)))

        # Мережа через NetworkClient
        self.network = NetworkClient('localhost', 8080, self.username, self.handle_line)
        self.network.connect()

        # Закриваємо сокет у разі виходу
        self.protocol("WM_DELETE_WINDOW", self.on_close)

    def toggle_show_menu(self):
        if self.is_show_menu:
            self.is_show_menu = False
            self.speed_animate_menu *= -1
            self.btn.configure(text='▶️')
            self.show_menu()
        else:
            self.is_show_menu = True
            self.speed_animate_menu *= -1
            self.btn.configure(text='◀️')
            self.show_menu()
            self.menu_label = CTkLabel(self.menu_frame, text='Імʼя', text_color=self.theme["author"])
            self.menu_label.pack(pady=30)
            self.menu_entry = CTkEntry(self.menu_frame, placeholder_text="Ваш новий нік ...")
            self.menu_entry.pack()
            self.save_button = CTkButton(self.menu_frame, text="Зберегти👌", fg_color=self.theme["author"],
                                         hover_color=self.theme["button"], command=self.save_name)
            self.save_button.pack()

    def show_menu(self):
        self.menu_frame.configure(width=self.menu_frame.winfo_width() + self.speed_animate_menu)
        if not self.menu_frame.winfo_width() >= 200 and self.is_show_menu:
            self.after(10, self.show_menu)
        elif self.menu_frame.winfo_width() >= 60 and not self.is_show_menu:
            self.after(10, self.show_menu)
            for item in ['menu_label', 'menu_entry', 'save_button']:
                if hasattr(self, item):
                    getattr(self, item).destroy()

    def save_name(self):
        new_name = self.menu_entry.get().strip()
        if new_name:
            self.username = new_name
            self.network.username = new_name
            self.add_message(f"Ваш новий нік😝: {self.username}")

    def adaptive_ui(self):
        self.menu_frame.configure(height=self.winfo_height(), fg_color=self.theme["menu_bg"])
        self.chat_field.place(x=self.menu_frame.winfo_width())
        self.chat_field.configure(width=self.winfo_width() - self.menu_frame.winfo_width() - 20,
                                 height=self.winfo_height() - 50, fg_color=self.theme["bg"])
        self.send_button.place(x=self.winfo_width() - 50, y=self.winfo_height() - 40)
        self.send_button.configure(fg_color=self.theme["button"])
        self.theme_button.place(x=40, y=0)
        self.message_entry.place(x=self.menu_frame.winfo_width(), y=self.send_button.winfo_y())
        self.message_entry.configure(
            width=self.winfo_width() - self.menu_frame.winfo_width() - 110)
        self.open_img_button.place(x=self.winfo_width()-105, y=self.send_button.winfo_y())
        self.open_img_button.configure(hover_color=self.theme["button"])
        self.after(50, self.adaptive_ui)

    def add_message(self, message, img=None, author=None):
        color = self.get_user_color(author) if author else self.theme["text"]
        message_frame = CTkFrame(self.chat_field, fg_color=self.theme["chat_fg"])
        message_frame.pack(pady=6, anchor='w', fill="x")
        wrapleng_size = self.winfo_width() - self.menu_frame.winfo_width() - 40

        if not img:
            CTkLabel(message_frame, text=message, wraplength=wrapleng_size,
                     text_color=color, justify='left').pack(padx=14, pady=5)
        else:
            CTkLabel(message_frame, text=message, wraplength=wrapleng_size,
                     text_color=color, image=img, compound='top',
                     justify='left').pack(padx=12, pady=5)

        # Автопрокрутка чату вниз
        self.chat_field._parent_canvas.yview_moveto(1)

    def get_user_color(self, username):
        if username not in self.users_colors:
            idx = len(self.users_colors) % len(self.USER_COLORS)
            self.users_colors[username] = self.USER_COLORS[idx] if self.theme == LIGHT_THEME else self.theme["author"]
        return self.users_colors[username]

    def send_message(self):
        message = self.message_entry.get()
        if message:
            self.add_message(f"{self.username}: {message}", author=self.username)
            data = f"TEXT@{self.username}@{message}\n"
            self.network.send(data)
        self.message_entry.delete(0, END)
        self.message_entry.focus_set()

    def handle_line(self, line):
        if not line:
            return
        parts = line.split("@", 3)
        msg_type = parts[0]

        if msg_type == "TEXT":
            if len(parts) >= 3:
                author = parts[1]
                message = parts[2]
                self.add_message(f"{author}: {message}", author=author)
        elif msg_type == "IMAGE":
            if len(parts) >= 4:
                author = parts[1]
                filename = parts[2]
                b64_img = parts[3]
                try:
                    img_data = base64.b64decode(b64_img)
                    pil_img = Image.open(io.BytesIO(img_data))
                    ctk_img = CTkImage(pil_img, size=(180, 180))
                    self.add_message(f"{author} надіслав(ла) зображення: {filename}", img=ctk_img, author=author)
                except Exception as e:
                    self.add_message(f"Помилка відображення зображення: {e}")
        else:
            self.add_message(line)

    def open_image(self):
        file_name = filedialog.askopenfilename()
        if not file_name:
            return
        try:
            with open(file_name, "rb") as f:
                raw = f.read()
            b64_data = base64.b64encode(raw).decode()
            short_name = os.path.basename(file_name)
            data = f"IMAGE@{self.username}@{short_name}@{b64_data}\n"
            self.network.send(data)
            self.add_message('', CTkImage(light_image=Image.open(file_name), size=(180, 180)), author=self.username)
        except Exception as e:
            self.add_message(f"Не вдалося надіслати зображення: {e}")

    def toggle_theme(self):
        # Перемикаємо тему
        current_mode = "dark" if self.theme == LIGHT_THEME else "light"
        self.theme_manager.set_theme(current_mode)
        self.theme = self.theme_manager.get_theme()
        # Оновити всі кольори
        self.configure(bg_color=self.theme["bg"])
        self.menu_frame.configure(fg_color=self.theme["menu_bg"])
        self.chat_field.configure(fg_color=self.theme["bg"])
        self.send_button.configure(fg_color=self.theme["button"])
        self.open_img_button.configure(hover_color=self.theme["button"])
        self.btn.configure(fg_color=self.theme["button"], hover_color=self.theme["menu_bg"])
        # Оновити кольори для нового режиму (авторів)
        for k in self.users_colors.keys():
            self.users_colors[k] = self.get_user_color(k)

        self.add_message(f"Тема перемикнута на {'dark' if current_mode == 'light' else 'light'} mode!", author="SYSTEM")

    def on_close(self):
        self.network.close()
        self.destroy()

if __name__ == "__main__":
    win = MainWindow()
    win.mainloop()
