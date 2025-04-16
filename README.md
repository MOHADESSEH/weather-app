# weather-app
import customtkinter as ctk
from PIL import Image, ImageTk
import httpx
import asyncio
import threading
from io import BytesIO
from datetime import datetime
import sqlite3

API_KEY = "749f06ae54a8469dbdf152910251304"
DB_NAME = "weather_data.db"
LANGUAGES = {"فارسی": "fa", "English": "en"}

ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("blue")

class WeatherDatabase:
    def __init__(self, db_name):
        self.conn = sqlite3.connect(db_name)
        self.create_table()

    def create_table(self):
        with self.conn:
            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS history (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    city TEXT,
                    temperature TEXT,
                    condition TEXT,
                    time TEXT)""")

    def add_record(self, city, temperature, condition, time):
        with self.conn:
            self.conn.execute(
                "INSERT INTO history (city, temperature, condition, time) VALUES (?, ?, ?, ?)",
                (city, temperature, condition, time))

    def get_history(self):
        with self.conn:
            return self.conn.execute(
                "SELECT city, temperature, condition, time FROM history ORDER BY id DESC").fetchall()

class WeatherFetcher:
    @staticmethod
    async def get_weather(city, lang="fa"):
        url = f"http://api.weatherapi.com/v1/current.json?key={API_KEY}&q={city}&lang={lang}"
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            if response.status_code == 200:
                data = response.json()
                return {
                    "name": data["location"]["name"],
                    "country": data["location"]["country"],
                    "localtime": data["location"]["localtime"],
                    "temp_c": data["current"]["temp_c"],
                    "temp_f": data["current"]["temp_f"],
                    "feelslike_c": data["current"]["feelslike_c"],
                    "feelslike_f": data["current"]["feelslike_f"],
                    "humidity": data["current"]["humidity"],
                    "wind_kph": data["current"]["wind_kph"],
                    "pressure_mb": data["current"]["pressure_mb"],
                    "condition": data["current"]["condition"]["text"],
                    "icon_url": "http:" + data["current"]["condition"]["icon"]}
            else:
                return None

    @staticmethod
    def fetch_icon_image(url):
        response = httpx.get(url)
        image_data = BytesIO(response.content)
        image = Image.open(image_data).resize((64, 64))
        return ImageTk.PhotoImage(image)

class WeatherApp:
    def __init__(self):
        self.db = WeatherDatabase(DB_NAME)
        self.root = ctk.CTk()
        self.root.title("Weather App")
        self.root.geometry("500x700")

        self.setup_ui()
        self.load_history()
        self.root.mainloop()

    def setup_ui(self):
        self.city_entry = ctk.CTkEntry(self.root, width=300, placeholder_text="نام شهر را وارد کنید")
        self.city_entry.pack(pady=15)
        self.city_entry.insert(0, "Tehran")

        self.temp_unit_var = ctk.StringVar(value="°C")
        self.temp_menu = ctk.CTkOptionMenu(self.root, variable=self.temp_unit_var, values=["°C", "°F"])
        self.temp_menu.pack(pady=10)

        self.lang_var = ctk.StringVar(value="فارسی")
        self.lang_menu = ctk.CTkOptionMenu(self.root, variable=self.lang_var, values=list(LANGUAGES.keys()))
        self.lang_menu.pack(pady=10)

        ctk.CTkButton(self.root, text="نمایش وضعیت هوا", command=self.on_get_weather).pack(pady=20)

        self.icon_label = ctk.CTkLabel(self.root, text="")
        self.icon_label.pack(pady=10)

        self.result_label = ctk.CTkLabel(self.root, text="", wraplength=450, justify="right")
        self.result_label.pack(pady=10)

        ctk.CTkLabel(self.root, text="تاریخچه جست‌وجو:").pack(pady=10)
        self.history_box = ctk.CTkTextbox(self.root, height=200, width=450)
        self.history_box.pack(pady=10)

    def load_history(self):
        history = self.db.get_history()
        self.history_box.delete("1.0", ctk.END)
        for item in history:
            city, temp, condition, time = item
            self.history_box.insert("end", f"{city} - {temp} - {condition} - {time}\n")

    def on_get_weather(self):
        city = self.city_entry.get().strip()
        if city:
            temp_unit = self.temp_unit_var.get()
            lang = LANGUAGES[self.lang_var.get()]
            threading.Thread(target=lambda: asyncio.run(
                self.fetch_weather_and_display(city, temp_unit, lang))
                             ).start()

    async def fetch_weather_and_display(self, city, temp_unit, lang):
        try:
            weather = await WeatherFetcher.get_weather(city, lang)
            if weather:
                time = datetime.strptime(weather["localtime"], "%Y-%m-%d %H:%M")
                temp = weather["temp_c"] if temp_unit == "°C" else weather["temp_f"]
                feels = weather["feelslike_c"] if temp_unit == "°C" else weather["feelslike_f"]
                unit = temp_unit

                result = (
                    f"شهر: {weather['name']}, {weather['country']}\n"
                    f"زمان محلی: {time.strftime('%H:%M %d/%m/%Y')}\n"
                    f"دما: {temp} {unit} (احساس: {feels} {unit})\n"
                    f"وضعیت: {weather['condition']}\n"
                    f"رطوبت: {weather['humidity']}%\n"
                    f"باد: {weather['wind_kph']} km/h\n"
                    f"فشار هوا: {weather['pressure_mb']} mb")

                icon_img = WeatherFetcher.fetch_icon_image(weather["icon_url"])
                self.icon_label.configure(image=icon_img)
                self.icon_label.image = icon_img

                self.result_label.configure(text=result)

                self.db.add_record(weather['name'], f"{temp}{unit}", weather['condition'], time.strftime('%H:%M'))
                self.load_history()
            else:
                self.result_label.configure(text="شهر یافت نشد یا مشکلی در ارتباط وجود دارد.")
        except Exception as e:
            self.result_label.configure(text=f"خطا: {e}")

if __name__ == "__main__":
    app = WeatherApp()
