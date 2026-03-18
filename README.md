Interface graficzny do:

https://github.com/hamhobbypl/wavegenerator

Na podstawie kodu:(c) 2026 Maniek SP8KM HAMHOBBY.PL — MIT License

Przeznaczony do treningu nasłuchu CW (kodu Morse'a) – pozwala tworzyć nieskończenie wiele wariantów ćwiczeń na podstawie jednego pliku źródłowego JSON.

## Zmiany

18.03.2026 - LOSUJ SEKCJE I WPISY (niestety na razie wybór razem).

## Funkcje

- Wczytywanie struktury ćwiczeń z pliku JSON.
- Losowa kolejność sekcji oraz słów w każdej sekcji.
- Regulacja prędkości znakowej (WPM) i prędkości Farnswortha (FWPM).
- Ustawienia częstotliwości tonu, amplitudy, częstotliwości próbkowania.
- Definiowanie przerw:
  - **X** – przerwa po każdym tokenie (słowie) w nagłówku.
  - **Y** – przerwa po każdym słowie w linii.
  - **Z** – przerwa po całej linii (zastosowanie: dłuższy odstęp między liniami).
- Automatyczne wygładzanie (ramp) sygnału – eliminacja kliknięć.
- Pasek postępu i informacje o stanie.
- Zapis do pliku WAV (mono, 16-bit).

## Build

Aby zbudować wersję:

## Windows
```bash
pyinstaller --onefile --windowed --name cw_generator.exe cw_gui.py
```

## macOS
```bash
pyinstaller --onefile --windowed --name cw_generator cw_gui.py
```

