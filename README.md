# smart-audio-QR-code-scanner
audio QR code scanner with Intelligent content detection
import qrcode
import cv2
import numpy as np
from pyzbar.pyzbar import decode
import webbrowser
import time
import re
import os
import requests
from bs4 import BeautifulSoup
from gtts import gTTS
import pygame
import nltk

nltk.download('punkt')
nltk.download('averaged_perceptron_tagger')
nltk.download('maxent_ne_chunker')
nltk.download('words')

from nltk import word_tokenize, pos_tag, ne_chunk
from nltk.tree import Tree
from urllib.parse import urlparse

def is_url(string):
    try:
        result = urlparse(string)
        return all([result.scheme, result.netloc])
    except:
        return False

def speak(text):
    tts = gTTS(text=text, lang='en', slow=False)
    audio_file = "speech.mp3"
    tts.save(audio_file)
    pygame.mixer.init()
    pygame.mixer.music.load(audio_file)
    pygame.mixer.music.play()
    while pygame.mixer.music.get_busy():
        time.sleep(0.1)
    pygame.mixer.quit()
    os.remove(audio_file)

def speak_long_text(text, chunk_size=900):
    chunks = [text[i:i+chunk_size] for i in range(0, len(text), chunk_size)]
    for chunk in chunks:
        speak(chunk)

def play_audio_from_url(url):
    try:
        r = requests.get(url)
        with open("temp_audio.mp3", "wb") as f:
            f.write(r.content)
        speak("Playing the audio file.")
        pygame.mixer.init()
        pygame.mixer.music.load("temp_audio.mp3")
        pygame.mixer.music.play()
        while pygame.mixer.music.get_busy():
            time.sleep(0.1)
        pygame.mixer.quit()
        os.remove("temp_audio.mp3")
    except:
        speak("Could not play audio from the URL.")

def extract_key_info(text):
    emails = re.findall(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}", text)
    phones = re.findall(r"\+?\d[\d\s\-().]{7,}\d", text)
    names = re.findall(r"\b[A-Z][a-z]+(?:\s[A-Z][a-z]+)*\b", text)
    return {
        "emails": list(set(emails)),
        "phones": list(set(phones)),
        "names": list(set(names))
    }

def read_website_content(url):
    try:
        response = requests.get(url)
        response.raise_for_status()
        soup = BeautifulSoup(response.text, 'html.parser')
        text = ' '.join([p.get_text() for p in soup.find_all('p')])
        if text:
            print("🔍 Extracting key info...")
            info = extract_key_info(text)
            if info["names"]:
                speak("Names found on the website include: " + ', '.join(info["names"][:3]))
            if info["emails"]:
                speak("Email addresses found are: " + ', '.join(info["emails"][:3]))
            if info["phones"]:
                speak("Phone numbers found are: " + ', '.join(info["phones"][:3]))
            speak("Now reading the content.")
            speak_long_text(text)
        else:
            speak("Sorry, no readable text found on the website.")
    except requests.exceptions.RequestException as e:
        print("Error fetching website content:", e)
        speak("Could not fetch website content.")

def process_qr_data(qr_data):
    print(f'Scanned QR Code Data: {qr_data}')
    if is_url(qr_data):
        speak("Website detected")
    elif qr_data.lower().endswith(('.mp3', '.wav', '.ogg')):
        speak("Audio file link detected")
    elif qr_data.isdigit():
        speak("Number detected")
    elif "@" in qr_data:
        speak("Email detected")
    else:
        speak("Text message detected")
    speak(f"The QR code contains: {qr_data}")
    if qr_data.lower().endswith(('.mp3', '.wav', '.ogg')):
        play_audio_from_url(qr_data)
    elif is_url(qr_data):
        webbrowser.open(qr_data)
        read_website_content(qr_data)
    else:
        webbrowser.open(f"https://www.google.com/search?q={qr_data}")

def scan_qr_from_webcam():
    cap = cv2.VideoCapture(0)
    scanned_data = set()
    print("📷 Webcam started. Show a QR code to the camera.")
    while True:
        success, frame = cap.read()
        if not success:
            break
        for code in decode(frame):
            qr_data = code.data.decode('utf-8')
            if qr_data not in scanned_data:
                scanned_data.add(qr_data)
                process_qr_data(qr_data)
        cv2.imshow('QR Code Scanner - Press Q to Quit', frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    scan_qr_from_webcam()
