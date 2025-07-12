#!/usr/bin/env python3
import os, sys, subprocess
from pydub import AudioSegment
import whisper

VERSION = "1.1"
TEMP_WAV = "temp_transcribir.wav"
OUTPUT_FILE = "transcripcion.txt"

def verificar_ffmpeg():
    try:
        subprocess.run(["ffmpeg", "-version"], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, check=True)
        return True
    except:
        return False

def mostrar_ayuda():
    print(f"""
Transcribir + Fácil v{VERSION}
Uso: arrastra o ejecuta:
    transcribir_facil.exe <ruta_audio>

Ejemplo:
    transcribir_facil.exe "C:\\...\\audio.opus"

Requisitos:
  • FFmpeg instalado (o portable en la misma carpeta)
  • Python 3.8+, whisper y pydub:
      pip install -U openai-whisper pydub

¿Cómo funciona?
  1. Convierte audio a WAV
  2. Transcribe con Whisper
  3. Marca con * palabras dudosas (palabras de 1 letra)
  4. Guarda en {OUTPUT_FILE}

Si algo falla, revisa los requisitos y vuelve a intentar :)
""")

def marcar_palabras(texto):
    res = []
    for w in texto.split():
        # Conservar signos de puntuación
        limpio = "".join(ch for ch in w if ch.isalnum())
        
        # Marcar solo palabras de 1 letra (no números)
        if len(limpio) == 1 and limpio.isalpha():
            res.append(w + '*')
        else:
            res.append(w)
    return " ".join(res)

def transcribir(archivo_audio):
    # CORRECCIÓN: Paréntesis bien cerrados
    if not os.path.exists(archivo_audio):
        print(f"[ERROR] No existe: {archivo_audio}")
        return

    if not verificar_ffmpeg():
        print("[ERROR] FALTA FFmpeg. Asegúrate de instalarlo o tener el ejecutable en la misma carpeta.")
        return

    try:
        print("🔄 Convirtiendo a WAV...")
        audio = AudioSegment.from_file(archivo_audio)
        audio.export(TEMP_WAV, format="wav", codec="pcm_s16le")

        print("⏳ Transcribiendo... (La primera vez descargará el modelo)")
        model = whisper.load_model("base")
        result = model.transcribe(TEMP_WAV)

        texto = result["text"].strip()
        marcado = marcar_palabras(texto)

        with open(OUTPUT_FILE, "w", encoding="utf-8") as f:
            f.write(marcado)

        print(f"✅ Transcripción completa. Revisa `{OUTPUT_FILE}`")
        if "*" in marcado:
            print("⚠️ Las palabras marcadas con * son de una letra (pueden ser dudosas).")
    except Exception as e:
        print(f"[ERROR CRÍTICO] No se pudo procesar: {str(e)}")
        import traceback
        traceback.print_exc()
    finally:
        if os.path.exists(TEMP_WAV):
            os.remove(TEMP_WAV)

if __name__ == "__main__":
    if len(sys.argv) < 2 or sys.argv[1].lower() in ("-h", "--help"):
        mostrar_ayuda()
    else:
        transcribir(sys.argv[1])
