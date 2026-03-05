Interface graficzny do:

https://github.com/hamhobbypl/wavegenerator

#!/usr/bin/env python3
# MIT License
# Copyright (c) 2026 Maniek SP8KM HAMHOBBY.PL
# Generator CW z GUI (tylko WAV) – kompaktowy układ z amplitudą nad XYZ

import json
import math
import random
import time
import wave
import threading
import queue
from array import array
from pathlib import Path
import tkinter as tk
from tkinter import filedialog, messagebox, ttk

# ---------- słownik Morse'a ----------
MORSE = {
    "A": ".-",    "B": "-...",  "C": "-.-.",  "D": "-..",   "E": ".",
    "F": "..-.",  "G": "--.",   "H": "....",  "I": "..",    "J": ".---",
    "K": "-.-",   "L": ".-..",  "M": "--",    "N": "-.",    "O": "---",
    "P": ".--.",  "Q": "--.-",  "R": ".-.",   "S": "...",   "T": "-",
    "U": "..-",   "V": "...-",  "W": ".--",   "X": "-..-",  "Y": "-.--",
    "Z": "--..",
    "0": "-----", "1": ".----", "2": "..---", "3": "...--", "4": "....-",
    "5": ".....", "6": "-....", "7": "--...", "8": "---..", "9": "----.",
    ".": ".-.-.-", ",": "--..--", "?": "..--..", "/": "-..-.", "-": "-....-",
    "(": "-.--.",  ")": "-.--.-",
}

# ---------- funkcje pomocniczne ----------
def farnsworth_scale(wpm: float, fwpm: float) -> float:
    if fwpm >= wpm:
        return 1.0
    dit = 1.2 / wpm
    target_word_time = 60.0 / fwpm
    fixed_units = 31
    variable_units = 19
    fixed_time = fixed_units * dit
    remaining = target_word_time - fixed_time
    if remaining <= 0:
        return 1.0
    scale = remaining / (variable_units * dit)
    return max(scale, 1.0)

def gen_tone(sr: int, freq: float, duration_s: float, amp: float = 0.35, ramp_s: float = 0.005) -> array:
    n = int(round(duration_s * sr))
    if n <= 0:
        return array('h')
    ramp = int(round(ramp_s * sr))
    ramp = min(ramp, n // 2)
    out = array('h')
    two_pi_f = 2.0 * math.pi * freq
    for i in range(n):
        t = i / sr
        x = math.sin(two_pi_f * t)
        if ramp > 0:
            if i < ramp:
                w = 0.5 - 0.5 * math.cos(math.pi * i / ramp)
            elif i >= n - ramp:
                j = n - 1 - i
                w = 0.5 - 0.5 * math.cos(math.pi * j / ramp)
            else:
                w = 1.0
        else:
            w = 1.0
        v = int(max(-1.0, min(1.0, amp * w * x)) * 32767)
        out.append(v)
    return out

def gen_silence(sr: int, duration_s: float) -> array:
    n = int(round(duration_s * sr))
    return array('h', [0] * max(0, n))

def add_samples(dst: array, src: array):
    dst.extend(src)

def cw_emit_token(token: str, sr: int, freq: float, wpm: float, fwpm: float, amp: float = 0.35) -> array:
    dit = 1.2 / wpm
    scale = farnsworth_scale(wpm, fwpm)
    intra = 1 * dit
    dah = 3 * dit
    inter_char = 3 * dit * scale
    out = array('h')
    token = token.upper()
    first_char = True
    for ch in token:
        code = MORSE.get(ch)
        if not code:
            add_samples(out, gen_silence(sr, inter_char))
            first_char = True
            continue
        if not first_char:
            add_samples(out, gen_silence(sr, inter_char))
        for ei, e in enumerate(code):
            add_samples(out, gen_tone(sr, freq, dit if e == "." else dah, amp=amp))
            if ei != len(code) - 1:
                add_samples(out, gen_silence(sr, intra))
        first_char = False
    return out

def parse_section_header(header: str) -> list[str]:
    return [t for t in header.strip().split() if t]

def split_wordline(raw: str) -> list[str]:
    s = raw.replace("[  ]", " ")
    return [t for t in s.strip().split() if t]

def estimate_total_steps(data) -> int:
    steps = 0
    for hdr, entries in data:
        steps += 1
        hdr_tokens = parse_section_header(hdr)
        steps += len(hdr_tokens)
        steps += len(entries)
        for raw in entries:
            steps += len(split_wordline(raw))
    return max(1, steps)

def render_one_pass(
    json_path: Path,
    sr: int,
    freq: float,
    wpm: float,
    fwpm: float,
    X: float,
    Y: float,
    Z: float,
    amp: float = 0.35,
    end_silence: float = 0.8,
    progress_callback=None,
) -> array:
    data = json.loads(json_path.read_text(encoding="utf-8"))
    sections = [(hdr, entries) for hdr, entries in data]
    random.shuffle(sections)
    out = array('h')
    z_after_line = max(0.0, Z - Y)

    for hdr, entries in sections:
        if progress_callback:
            progress_callback(1)
        hdr_tokens = parse_section_header(hdr)
        for tok in hdr_tokens:
            add_samples(out, cw_emit_token(tok, sr, freq, wpm, fwpm, amp=amp))
            add_samples(out, gen_silence(sr, X))
            if progress_callback:
                progress_callback(1)
        entries = list(entries)
        random.shuffle(entries)
        for raw in entries:
            if progress_callback:
                progress_callback(1)
            words = split_wordline(raw)
            for w in words:
                add_samples(out, cw_emit_token(w, sr, freq, wpm, fwpm, amp=amp))
                add_samples(out, gen_silence(sr, Y))
                if progress_callback:
                    progress_callback(1)
            add_samples(out, gen_silence(sr, z_after_line))
    add_samples(out, gen_silence(sr, end_silence))
    return out

def write_wav_mono16(path: Path, samples: array, sr: int):
    with wave.open(str(path), "wb") as wf:
        wf.setnchannels(1)
        wf.setsampwidth(2)
        wf.setframerate(sr)
        wf.writeframes(samples.tobytes())

# ---------- GUI ----------
class CwGeneratorApp:
    def __init__(self, root):
        self.root = root
        self.root.title("hamhobby.pl Generator CW")
        self.root.resizable(False, False)

        # Zmienne
        self.json_path = tk.StringVar()
        self.out_path = tk.StringVar(value="cw_losowo.wav")
        self.wpm = tk.DoubleVar(value=20.0)
        self.fwpm = tk.DoubleVar(value=12.0)
        self.freq = tk.DoubleVar(value=600.0)
        self.sr = tk.IntVar(value=44100)
        self.x = tk.DoubleVar(value=1.0)
        self.y = tk.DoubleVar(value=1.0)
        self.z = tk.DoubleVar(value=3.0)
        self.amp = tk.DoubleVar(value=0.35)
        self.end_silence = tk.DoubleVar(value=0.8)

        # Kolejka do komunikacji z wątkiem
        self.queue = queue.Queue()

        self.create_widgets()
        self.update_progress_from_queue()

    def create_widgets(self):
        # Ramka na pliki
        frame_files = ttk.LabelFrame(self.root, text="Pliki", padding=5)
        frame_files.grid(row=0, column=0, padx=5, pady=5, sticky="ew")

        ttk.Label(frame_files, text="Plik JSON:").grid(row=0, column=0, sticky="w")
        ttk.Entry(frame_files, textvariable=self.json_path, width=40).grid(row=0, column=1, padx=5)
        ttk.Button(frame_files, text="Przeglądaj", command=self.browse_json).grid(row=0, column=2)

        ttk.Label(frame_files, text="Plik wyjściowy WAV:").grid(row=1, column=0, sticky="w", pady=5)
        ttk.Entry(frame_files, textvariable=self.out_path, width=40).grid(row=1, column=1, padx=5)
        ttk.Button(frame_files, text="Zapisz jako", command=self.browse_out).grid(row=1, column=2)

        # Ramka parametrów – 4 wiersze, amplituda nad XYZ
        frame_params = ttk.LabelFrame(self.root, text="Parametry", padding=5)
        frame_params.grid(row=1, column=0, padx=5, pady=5, sticky="ew")

        # Wiersz 0: WPM i FWPM
        ttk.Label(frame_params, text="WPM:").grid(row=0, column=0, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.wpm, width=8).grid(row=0, column=1, padx=2, pady=2)
        ttk.Label(frame_params, text="FWPM:").grid(row=0, column=2, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.fwpm, width=8).grid(row=0, column=3, padx=2, pady=2)

        # Wiersz 1: Częstotliwość i Sample rate
        ttk.Label(frame_params, text="Częstotliwość (Hz):").grid(row=1, column=0, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.freq, width=8).grid(row=1, column=1, padx=2, pady=2)
        ttk.Label(frame_params, text="Sample rate:").grid(row=1, column=2, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.sr, width=8).grid(row=1, column=3, padx=2, pady=2)

        # Wiersz 2: Amplituda i Cisza końcowa (wyżej)
        ttk.Label(frame_params, text="Amplituda (0-1):").grid(row=2, column=0, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.amp, width=8).grid(row=2, column=1, padx=2, pady=2)
        ttk.Label(frame_params, text="Cisza końcowa (s):").grid(row=2, column=2, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.end_silence, width=8).grid(row=2, column=3, padx=2, pady=2)

        # Wiersz 3: Przerwa X, Y, Z (niżej)
        ttk.Label(frame_params, text="Przerwa X (s):").grid(row=3, column=0, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.x, width=8).grid(row=3, column=1, padx=2, pady=2)
        ttk.Label(frame_params, text="Y (s):").grid(row=3, column=2, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.y, width=8).grid(row=3, column=3, padx=2, pady=2)
        ttk.Label(frame_params, text="Z (s):").grid(row=3, column=4, sticky="w", padx=2, pady=2)
        ttk.Entry(frame_params, textvariable=self.z, width=8).grid(row=3, column=5, padx=2, pady=2)

        # Ramka postępu
        frame_progress = ttk.Frame(self.root, padding=5)
        frame_progress.grid(row=2, column=0, padx=5, pady=5, sticky="ew")

        self.progress_bar = ttk.Progressbar(frame_progress, mode='determinate', length=400)
        self.progress_bar.pack(fill=tk.X, pady=5)

        self.status_label = ttk.Label(frame_progress, text="Gotowy")
        self.status_label.pack()

        # Przycisk Generuj
        self.generate_btn = ttk.Button(self.root, text="Generuj plik WAV", command=self.start_generation)
        self.generate_btn.grid(row=3, column=0, pady=10)

    def browse_json(self):
        filename = filedialog.askopenfilename(filetypes=[("JSON files", "*.json"), ("All files", "*.*")])
        if filename:
            self.json_path.set(filename)

    def browse_out(self):
        filename = filedialog.asksaveasfilename(defaultextension=".wav", filetypes=[("WAV files", "*.wav")])
        if filename:
            self.out_path.set(filename)

    def start_generation(self):
        if not self.json_path.get():
            messagebox.showerror("Błąd", "Wybierz plik JSON")
            return
        if not Path(self.json_path.get()).exists():
            messagebox.showerror("Błąd", "Plik JSON nie istnieje")
            return

        self.generate_btn.config(state=tk.DISABLED)
        self.status_label.config(text="Generowanie...")
        self.progress_bar['value'] = 0

        thread = threading.Thread(target=self.generate_thread, daemon=True)
        thread.start()

    def generate_thread(self):
        try:
            json_path = Path(self.json_path.get())
            data = json.loads(json_path.read_text(encoding="utf-8"))
            total_steps = estimate_total_steps(data)

            self.queue.put(('max', total_steps))
            self.queue.put(('status', 'Generowanie...'))

            step_count = 0
            def progress_callback(n=1):
                nonlocal step_count
                step_count += n
                self.queue.put(('step', step_count))

            samples = render_one_pass(
                json_path=json_path,
                sr=self.sr.get(),
                freq=self.freq.get(),
                wpm=self.wpm.get(),
                fwpm=self.fwpm.get(),
                X=self.x.get(),
                Y=self.y.get(),
                Z=self.z.get(),
                amp=self.amp.get(),
                end_silence=self.end_silence.get(),
                progress_callback=progress_callback,
            )

            out_path = Path(self.out_path.get())
            write_wav_mono16(out_path, samples, self.sr.get())

            self.queue.put(('done', f"Zapisano: {out_path.name}"))

        except Exception as e:
            self.queue.put(('error', str(e)))

    def update_progress_from_queue(self):
        try:
            while True:
                msg = self.queue.get_nowait()
                if msg[0] == 'max':
                    self.progress_bar['maximum'] = msg[1]
                elif msg[0] == 'step':
                    self.progress_bar['value'] = msg[1]
                elif msg[0] == 'status':
                    self.status_label.config(text=msg[1])
                elif msg[0] == 'done':
                    self.status_label.config(text=msg[1])
                    self.generate_btn.config(state=tk.NORMAL)
                    messagebox.showinfo("Sukces", msg[1])
                elif msg[0] == 'error':
                    self.status_label.config(text="Błąd")
                    self.generate_btn.config(state=tk.NORMAL)
                    messagebox.showerror("Błąd", msg[1])
        except queue.Empty:
            pass
        finally:
            self.root.after(100, self.update_progress_from_queue)

def main():
    root = tk.Tk()
    app = CwGeneratorApp(root)
    root.mainloop()

if __name__ == "__main__":
    main()
