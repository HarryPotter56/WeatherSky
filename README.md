import customtkinter as ctk
from tkinter import StringVar
import requests
from datetime import datetime
from PIL import Image
from io import BytesIO
import threading
from collections import defaultdict

# ---------------- CONFIG ----------------
API_KEY = "68418ce436132483c0bce07db5b9f435"  # Replace with your valid key
BASE_URL_CURRENT = "https://api.openweathermap.org/data/2.5/weather"
BASE_URL_FORECAST = "https://api.openweathermap.org/data/2.5/forecast"

# ---------------- APP SETUP ----------------
ctk.set_appearance_mode("light")
ctk.set_default_color_theme("blue")

win = ctk.CTk()
win.title("Weather Sky")
win.state("zoomed")

# ---------------- TITLE BAR ----------------
title_frame = ctk.CTkFrame(win, fg_color=("#4A90E2", "#2C2C2C"), corner_radius=0)
title_frame.pack(fill="x")

title_inner = ctk.CTkFrame(title_frame, fg_color="transparent")
title_inner.pack(pady=15)

title_icon_label = ctk.CTkLabel(title_inner, text="", image=None)
title_icon_label.pack(side="left", padx=10)

title_label = ctk.CTkLabel(title_inner, text="Weather Sky",
                           font=("Times New Roman", 36, "bold"),
                           text_color="white")
title_label.pack(side="left")

time_label = ctk.CTkLabel(title_frame, text="", font=("Arial", 16), text_color="white")
time_label.pack(pady=5)

def update_time():
    now = datetime.now().strftime("%I:%M %p, %A, %d %B %Y")
    time_label.configure(text=now)
    win.after(1000, update_time)

update_time()

# ---------------- CITY SELECTION ----------------
city_frame = ctk.CTkFrame(win, fg_color=("white", "#333"), corner_radius=20)
city_frame.pack(pady=20, padx=20, fill="x")

city_name = StringVar()
city_entry = ctk.CTkEntry(city_frame, placeholder_text="Enter city name (e.g. Lucknow)",
                          textvariable=city_name, font=("Times New Roman", 18), width=350)
city_entry.grid(row=0, column=0, padx=20, pady=20)

get_button = ctk.CTkButton(city_frame, text="Get Weather", font=("Times New Roman", 20, "bold"),
                           corner_radius=15, width=180, fg_color=("#4A90E2", "#1E90FF"),
                           hover_color=("#357ABD", "#104E8B"))
get_button.grid(row=0, column=1, padx=20)

# ---------------- CURRENT WEATHER ----------------
current_frame = ctk.CTkFrame(win, fg_color=("#DDEAF6", "#2D2D2D"), corner_radius=20)
current_frame.pack(pady=10, padx=20, fill="x")

w_label = ctk.CTkLabel(current_frame, text="Weather: --", font=("Times New Roman", 22))
w_label.grid(row=0, column=0, padx=20, pady=10)

desc_label = ctk.CTkLabel(current_frame, text="Description: --", font=("Times New Roman", 22))
desc_label.grid(row=1, column=0, padx=20, pady=10)

temp_label = ctk.CTkLabel(current_frame, text="Temperature: -- °C", font=("Times New Roman", 22))
temp_label.grid(row=0, column=1, padx=20, pady=10)

pressure_label = ctk.CTkLabel(current_frame, text="Pressure: -- hPa", font=("Times New Roman", 22))
pressure_label.grid(row=1, column=1, padx=20, pady=10)

current_icon_label = ctk.CTkLabel(current_frame, text="", image=None)
current_icon_label.grid(row=0, column=2, rowspan=2, padx=30, pady=10)

# ---------------- FORECAST ----------------
forecast_frame = ctk.CTkFrame(win, fg_color=("#EFF6FC", "#1C1C1C"), corner_radius=20)
forecast_frame.pack(pady=20, padx=20, fill="both", expand=True)

forecast_cards = []

def create_forecast_cards():
    for i in range(5):
        card = ctk.CTkFrame(forecast_frame, fg_color=("#FFFFFF", "#2D2D2D"), corner_radius=20)
        card.grid(row=0, column=i, padx=15, pady=20, sticky="nsew")
        forecast_frame.grid_columnconfigure(i, weight=1)

        day_label = ctk.CTkLabel(card, text="Day", font=("Arial", 18, "bold"))
        day_label.pack(pady=6)

        icon_label = ctk.CTkLabel(card, text="", image=None)
        icon_label.pack(pady=6)

        temp_label = ctk.CTkLabel(card, text="-- °C", font=("Arial", 16))
        temp_label.pack(pady=6)

        desc_label = ctk.CTkLabel(card, text="--", font=("Arial", 14))
        desc_label.pack(pady=6)

        forecast_cards.append({
            "day": day_label,
            "icon": icon_label,
            "temp": temp_label,
            "desc": desc_label,
            "icon_image": None
        })

create_forecast_cards()

# ---------------- ICON LOADER ----------------
def load_weather_icon(icon_code):
    try:
        url = f"http://openweathermap.org/img/wn/{icon_code}@2x.png"
        response = requests.get(url, timeout=5)
        if response.status_code == 200:
            image = Image.open(BytesIO(response.content)).convert("RGBA")
            return ctk.CTkImage(light_image=image, dark_image=image, size=(60, 60))
        else:
            print(f"Failed to fetch icon: {response.status_code} for code {icon_code}")
    except Exception as e:
        print("Icon load error:", e)
    return None

# ---------------- STATUS BAR ----------------
status_label = ctk.CTkLabel(win, text="Waiting for update...",
                            font=("Arial", 12), text_color="gray")
status_label.pack(side="bottom", pady=5)

# ---------------- FORECAST LOGIC (ONE PER DAY, NEAREST TO MIDDAY) ----------------
def get_daily_forecasts(forecast_data):
    days = defaultdict(list)
    for f in forecast_data["list"]:
        date = f["dt_txt"].split()[0]
        days[date].append(f)
    daily = []
    for date, items in list(days.items())[:5]:
        midday = min(items, key=lambda x: abs(int(x["dt_txt"].split()[1][:2]) - 12))
        daily.append(midday)
    return daily

# ---------------- WEATHER FUNCTION ----------------
def get_weather():
    city = city_name.get().strip()
    if not city:
        status_label.configure(text="Please enter a valid city name.")
        return

    get_button.configure(state="disabled")
    status_label.configure(text="Loading weather data...")

    try:
        # CURRENT WEATHER
        current_params = {"q": city, "appid": API_KEY, "units": "metric"}
        current_resp = requests.get(BASE_URL_CURRENT, params=current_params, timeout=10)
        current_data = current_resp.json()

        if current_data.get("cod") != 200:
            w_label.configure(text="Error: " + current_data.get("message", ""))
            desc_label.configure(text="Description: --")
            temp_label.configure(text="Temperature: -- °C")
            pressure_label.configure(text="Pressure: -- hPa")
            current_icon_label.configure(image=None)
            title_icon_label.configure(image=None)
            status_label.configure(text="City not found or API error.")
            get_button.configure(state="normal")
            return

        w_label.configure(text=f"Weather: {current_data['weather'][0]['main']}")
        desc_label.configure(text=f"Description: {current_data['weather'][0]['description']}")
        temp_label.configure(text=f"Temperature: {current_data['main']['temp']} °C")
        pressure_label.configure(text=f"Pressure: {current_data['main']['pressure']} hPa")

        icon_code = current_data['weather'][0]['icon']
        icon_image = load_weather_icon(icon_code)
        current_icon_label.configure(image=icon_image)
        current_icon_label.image = icon_image
        title_icon_label.configure(image=icon_image)
        title_icon_label.image = icon_image

        # FORECAST
        forecast_params = {"q": city, "appid": API_KEY, "units": "metric"}
        forecast_resp = requests.get(BASE_URL_FORECAST, params=forecast_params, timeout=10)
        forecast_data = forecast_resp.json()

        if forecast_data.get("cod") != "200":
            for card in forecast_cards:
                card["day"].configure(text="Day")
                card["temp"].configure(text="-- °C")
                card["desc"].configure(text="--")
                card["icon"].configure(image=None)
            status_label.configure(text="Forecast not found or API error.")
            get_button.configure(state="normal")
            return

        forecasts = get_daily_forecasts(forecast_data)

        for i, f in enumerate(forecasts):
            dt_txt = f["dt_txt"]
            day_name = datetime.strptime(dt_txt, "%Y-%m-%d %H:%M:%S").strftime("%a")
            temp = f["main"]["temp"]
            desc = f["weather"][0]["main"]
            icon_code = f["weather"][0]["icon"]
            icon_image = load_weather_icon(icon_code)

            forecast_cards[i]["day"].configure(text=day_name)
            forecast_cards[i]["temp"].configure(text=f"{temp:.1f} °C")
            forecast_cards[i]["desc"].configure(text=desc)
            forecast_cards[i]["icon"].configure(image=icon_image)
            forecast_cards[i]["icon"].image = icon_image
            forecast_cards[i]["icon_image"] = icon_image

        status_label.configure(text=f"Last updated: {datetime.now().strftime('%I:%M %p, %b %d, %Y')}")
    except Exception as e:
        status_label.configure(text="Error fetching weather data.")
        print("Weather fetch error:", e)
    finally:
        get_button.configure(state="normal")

def get_weather_threaded():
    threading.Thread(target=get_weather, daemon=True).start()

get_button.configure(command=get_weather_threaded)

# ---------------- RUN APP ----------------
win.mainloop()
