import os
os.environ['PYGAME_HIDE_SUPPORT_PROMPT'] = "hide"

from ids_peak import ids_peak
import numpy as np
from PIL import Image
import ctypes
import io
from datetime import datetime, timedelta
import time
import threading
import multiprocessing
import pygame
import pygame.mixer
import sys
import keyboard
import msvcrt
import cv2
from queue import Empty
import logging
import traceback
import psutil
import queue  # Am Anfang der Datei hinzufügen

DEBUG = True
PROGRAM_NAME = "Kameraaufnahme"
VERSION = "1.92"
CREATION_DATE = "11.08.2025"

os.makedirs("Bilder", exist_ok=True)
os.makedirs("Videos", exist_ok=True)

if DEBUG:
    logging.basicConfig(filename='debug.log', level=logging.DEBUG, 
                        format='%(asctime)s - %(levelname)s - %(message)s',
                        encoding='utf-8')

def debug_print(*args, **kwargs):
    if DEBUG:
        message = ' '.join(map(str, args))
        logging.debug(message, **kwargs)

def log_system_info():
    debug_print("=== Systemumgebung ===")
    debug_print(f"Python: {sys.version.split()[0]}, OpenCV: {cv2.__version__}, NumPy: {np.__version__}")
    debug_print(f"OS: {os.name}-{sys.platform}, CPU Kerne: {multiprocessing.cpu_count()}")
    debug_print(f"RAM verfügbar: {psutil.virtual_memory().available / (1024**3):.2f}GB")

def log_camera_info(device):
    try:
        nodemap = device.RemoteDevice().NodeMaps()[0]
        debug_print(f"=== Kamera: {nodemap.FindNode('DeviceModelName').Value()} ===")
        debug_print(f"S/N: {nodemap.FindNode('DeviceSerialNumber').Value()}")
        debug_print(f"Firmware: {nodemap.FindNode('DeviceFirmwareVersion').Value()}")
    except Exception as e:
        debug_print(f"Fehler beim Lesen der Kamera-Details: {str(e)}")

def log_frame_info(camera_name, frame_count, total_frames, frame_time, buffer_size):
    if frame_count % 30 == 0:  # Nur alle 30 Frames
        debug_print(f"{camera_name}: Frame {frame_count}/{total_frames}, FPS: {frame_count/frame_time:.1f}")

def log_performance_metrics(start_time, operation_name):
    if operation_name.startswith(("Start", "Ende")):
        elapsed = time.perf_counter() - start_time
        ram = psutil.Process().memory_info().rss / (1024**2)
        debug_print(f"Performance [{operation_name}]: {elapsed:.1f}s, RAM: {ram:.1f}MB")

def log_video_details(filename, frame_count, actual_duration, expected_duration):
    debug_print(f"Video: {os.path.basename(filename)}, Frames: {frame_count}, "
                f"Dauer: {actual_duration:.1f}s/{expected_duration:.1f}s")

def log_exception(e, context=""):
    debug_print(f"Fehler in {context}: {type(e).__name__} - {str(e)}")
    if DEBUG:
        debug_print(traceback.format_exc())

def generate_sine_wave(frequency, duration, volume=0.8, sample_rate=44100):
    samples = np.arange(int(duration * sample_rate)) / sample_rate
    wave = np.sin(2 * np.pi * frequency * samples)
    return np.ascontiguousarray((np.array([wave, wave]).T * volume * 32767).astype(np.int16))

def initialize_audio():
    try:
        pygame.mixer.init(frequency=44100, size=-16, channels=2, buffer=2048)
        debug_print("Audio initialisiert")
    except Exception as e:
        log_exception(e, "Audio-Init")

def play_audio(audio_playing, abort_flag):
    """Spielt einen kontinuierlichen Ton während der Aufnahme"""
    try:
        pygame.mixer.init(frequency=44100, size=-16, channels=1, buffer=4096)
        debug_print("Audio-System initialisiert")
        
        # Generiere kontinuierlichen Ton
        frequency = 1000  # 1kHz Ton
        duration = 0.5    # 500ms Samples (wird in Schleife wiederholt)
        samples = generate_sine_wave(frequency, duration)
        sound = pygame.mixer.Sound(buffer=samples)
        channel = None
        
        while not abort_flag.value:
            if audio_playing.value:
                if not channel or not channel.get_busy():
                    debug_print("Audio-Status: START CONTINUOUS")
                    channel = sound.play(-1)  # -1 bedeutet Endlosschleife
            else:
                if channel and channel.get_busy():
                    debug_print("Audio-Status: STOP")
                    channel.stop()
                time.sleep(0.1)
                
    except Exception as e:
        debug_print(f"Fehler im Audio-Prozess: {str(e)}")
    finally:
        debug_print("Audio-Prozess wird beendet")
        pygame.mixer.quit()

def stop_audio(audio_process):
    if audio_process and audio_process.is_alive():
        audio_process.terminate()
        audio_process.join()
        debug_print("Audio-Prozess gestoppt")

def check_camera_availability(num_cameras):
    debug_print(f"Prüfe {num_cameras} Kameras")
    try:
        ids_peak.Library.Initialize()
        device_manager = ids_peak.DeviceManager.Instance()
        device_manager.Update()

        if device_manager.Devices().empty():
            return False, "Keine Kamera gefunden."

        if len(device_manager.Devices()) < num_cameras:
            return False, f"Nur {len(device_manager.Devices())} von {num_cameras} Kameras gefunden."

        for i in range(num_cameras):
            device = device_manager.Devices()[i].OpenDevice(ids_peak.DeviceAccessType_Control)
            log_camera_info(device)
            debug_print(f"Kamera {i+1} OK")

        return True, ""
    except ids_peak.Exception as e:
        if "GC_ERR_ACCESS_DENIED" in str(e):
            return False, "Kamerazugriff verweigert. IDS peak Cockpit geschlossen?"
        return False, f"Kamera-Fehler: {str(e)}"
    finally:
        ids_peak.Library.Close()
def capture_image(image_number, total_images, device_index, camera_name, abort_flag):
    start_time = time.perf_counter()
    debug_print(f"Starte Bildaufnahme {camera_name}")
    ids_peak.Library.Initialize()

    try:
        device_manager = ids_peak.DeviceManager.Instance()
        device_manager.Update()
        device = device_manager.Devices()[device_index].OpenDevice(ids_peak.DeviceAccessType_Control)
        nodemap_remote_device = device.RemoteDevice().NodeMaps()[0]
        
        # Setze die Kamera-Parameter
        nodemap_remote_device.FindNode("AcquisitionMode").SetCurrentEntry("SingleFrame")
        pixel_format_node = nodemap_remote_device.FindNode("PixelFormat")
        if "RGB8" in [entry.SymbolicValue() for entry in pixel_format_node.Entries()]:
            pixel_format_node.SetCurrentEntry("RGB8")

        width = nodemap_remote_device.FindNode("Width").Value()
        height = nodemap_remote_device.FindNode("Height").Value()
        payload_size = nodemap_remote_device.FindNode("PayloadSize").Value()

        datastream = device.DataStreams()[0].OpenDataStream()
        buffer = datastream.AllocAndAnnounceBuffer(payload_size)
        datastream.QueueBuffer(buffer)

        datastream.StartAcquisition()
        nodemap_remote_device.FindNode("AcquisitionStart").Execute()
        buffer = datastream.WaitForFinishedBuffer(10000)

        if buffer is None:
            raise Exception("Null-Buffer erhalten")

        raw_buffer = (ctypes.c_ubyte * buffer.Size()).from_address(int(buffer.BasePtr()))
        image_bytes = bytes(raw_buffer)
        image_data = np.frombuffer(image_bytes, dtype=np.uint8)
        image_data = image_data.reshape((height, width, 3))
        pil_image = Image.fromarray(image_data, mode='RGB')

        timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
        filename = os.path.join("Bilder", f"image_{image_number:06d}_{camera_name}_{timestamp}.jpg")
        pil_image.save(filename, format='JPEG')
        debug_print(f"Bild gespeichert: {filename}")

        datastream.QueueBuffer(buffer)
        nodemap_remote_device.FindNode("AcquisitionStop").Execute()
        datastream.StopAcquisition()
        datastream.Flush(ids_peak.DataStreamFlushMode_DiscardAll)
        datastream.RevokeBuffer(buffer)

    except Exception as e:
        log_exception(e, f"Bildaufnahme {camera_name}")
        raise
    finally:
        ids_peak.Library.Close()
        log_performance_metrics(start_time, f"Ende Bildaufnahme {camera_name}")

def capture_video(video_number, video_length, device_index, camera_name, start_event, timestamp_queue, 
                 frame_queue, abort_flag, progress_queue, audio_playing):
    preview_interval = 10
    frames_written = 0
    datastream = None
    device = None
    
    try:
        # Library initialisieren
        ids_peak.Library.Initialize()
        debug_print(f"{camera_name}: IDS peak Library initialisiert")
        
        # Audio nur für Camera1 starten
        if camera_name == 'Camera1':
            audio_playing.value = True
            debug_print(f"{camera_name}: Audio für Videoaufnahme gestartet")
        
        # Device Manager
        device_manager = ids_peak.DeviceManager.Instance()
        device_manager.Update()
        
        # Device öffnen
        device = device_manager.Devices()[device_index].OpenDevice(ids_peak.DeviceAccessType_Control)
        nodemap_remote_device = device.RemoteDevice().NodeMaps()[0]
        
        # Setze einheitliche FPS für beide Kameras
        target_fps = 12.5
        fps_node = nodemap_remote_device.FindNode("AcquisitionFrameRate")
        if fps_node and fps_node.AccessStatus() == ids_peak.NodeAccessStatus_ReadWrite:
            fps_node.SetValue(target_fps)
            actual_fps = target_fps
            debug_print(f"{camera_name}: FPS auf {actual_fps:.2f} gesetzt")
        else:
            debug_print(f"{camera_name}: Konnte FPS nicht setzen, verwende Standard")
            actual_fps = fps_node.Value() if fps_node else 12.5

        # Berechne die Ziel-Frames mit einheitlichem Puffer
        target_frames = int(video_length * actual_fps * 1.02)  # 2% Puffer
        max_time = video_length * 1.05  # 5% mehr Zeit
        debug_print(f"{camera_name}: Ziel-Frames: {target_frames} bei {actual_fps:.2f} fps (mit 2% Puffer)")

        # Buffer initialisieren
        datastream = device.DataStreams()[0].OpenDataStream()
        num_buffers = datastream.NumBuffersAnnouncedMinRequired()
        payload_size = nodemap_remote_device.FindNode("PayloadSize").Value()

        buffers = []
        for _ in range(num_buffers):
            buffer = datastream.AllocAndAnnounceBuffer(payload_size)
            buffers.append(buffer)
            datastream.QueueBuffer(buffer)
        debug_print(f"{camera_name}: {num_buffers} Buffer allokiert")

        # Vorbereitung für Video-Aufnahme
        frames_written = 0
        actual_start_time = None
        timestamp = timestamp_queue.get() if timestamp_queue else datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
        filename = os.path.join("Videos", f"video_{video_number:06d}_{camera_name}_{timestamp}.mp4")
        
        # VideoWriter mit der erkannten Framerate initialisieren
        fourcc = cv2.VideoWriter_fourcc(*'mp4v')
        width = nodemap_remote_device.FindNode("Width").Value()
        height = nodemap_remote_device.FindNode("Height").Value()
        debug_print(f"{camera_name}: Erstelle VideoWriter mit {actual_fps:.2f} fps")
        out = cv2.VideoWriter(filename, fourcc, float(actual_fps), (width, height))
        if not out.isOpened():
            raise Exception(f"Konnte VideoWriter nicht öffnen")
        
        # Starte die Aufnahme
        try:
            datastream.StartAcquisition()
            nodemap_remote_device.FindNode("AcquisitionStart").Execute()
            debug_print(f"{camera_name}: Acquisition gestartet")
        except Exception as e:
            debug_print(f"{camera_name}: Fehler beim Starten der Acquisition: {str(e)}")
            raise

        # Überprüfe das aktuelle Pixelformat
        pixel_format_node = nodemap_remote_device.FindNode("PixelFormat")
        current_format = pixel_format_node.CurrentEntry().SymbolicValue()
        debug_print(f"{camera_name}: Pixelformat ist {current_format}")

        # Für das Video und Preview: BayerBG2BGR
        color_conversion = cv2.COLOR_BayerBG2BGR
        
        # Setze Belichtungszeit
        exposure_node = nodemap_remote_device.FindNode("ExposureTime")
        if exposure_node:
            current_exposure = exposure_node.Value()
            debug_print(f"{camera_name}: Aktuelle Belichtungszeit: {current_exposure}µs")
            if current_exposure < 10000:  # Wenn unter 10ms
                exposure_node.SetValue(10000)
                debug_print(f"{camera_name}: Belichtungszeit auf 10000µs gesetzt")
        
        while frames_written < target_frames and not abort_flag.value:
            try:
                buffer = datastream.WaitForFinishedBuffer(5000)
                if buffer is None:
                    debug_print(f"{camera_name}: Kein Buffer erhalten nach Timeout")
                    continue
                
                # Frame verarbeiten
                raw_buffer = (ctypes.c_ubyte * buffer.Size()).from_address(int(buffer.BasePtr()))
                frame_bayer = np.frombuffer(raw_buffer, dtype=np.uint8).reshape((height, width))
                
                # Video Frame
                frame_bgr = cv2.cvtColor(frame_bayer, color_conversion)
                ret = out.write(frame_bgr)
                debug_print(f"{camera_name}: Frame {frames_written + 1} geschrieben - Ret: {ret}")
                
                # Preview Frame - jetzt mit der gleichen Konvertierung
                if frame_queue and frames_written % preview_interval == 0:
                    try:
                        preview_frame = cv2.cvtColor(frame_bayer, color_conversion)  # Gleiche Konvertierung wie Video
                        if isinstance(preview_frame, np.ndarray):
                            frame_queue.put_nowait((preview_frame, camera_name))
                            debug_print(f"{camera_name}: Preview Frame {frames_written} in Queue")
                    except queue.Full:
                        debug_print(f"{camera_name}: Preview Queue voll - Frame übersprungen")
                    except Exception as e:
                        debug_print(f"{camera_name}: Preview-Fehler: {str(e)}")
                
                frames_written += 1
                
            finally:
                if buffer and not buffer.IsQueued():
                    try:
                        datastream.QueueBuffer(buffer)
                    except Exception as e:
                        debug_print(f"{camera_name}: Buffer Queue Fehler: {str(e)}")
                        
        debug_print(f"{camera_name}: Aufnahme beendet - {frames_written} Frames geschrieben")
        
    except Exception as e:
        debug_print(f"{camera_name}: Kritischer Fehler: {str(e)}")
        debug_print(f"{camera_name}: Stack Trace: {traceback.format_exc()}")
        
    finally:
        try:
            if datastream and datastream.IsGrabbing():
                datastream.StopAcquisition()
                debug_print(f"{camera_name}: Datastream gestoppt")
            if device:
                device.RemoteDevice().NodeMaps()[0].FindNode("AcquisitionStop").Execute()
                debug_print(f"{camera_name}: Acquisition gestoppt")
        except Exception as e:
            debug_print(f"{camera_name}: Fehler beim Cleanup: {str(e)}")
            
        try:
            ids_peak.Library.Close()
            debug_print(f"{camera_name}: IDS peak Library geschlossen")
        except Exception as e:
            debug_print(f"{camera_name}: Fehler beim Schließen der Library: {str(e)}")

        # Audio nur von Camera1 stoppen
        if camera_name == 'Camera1':
            audio_playing.value = False
            debug_print(f"{camera_name}: Audio für Videoaufnahme gestoppt")

def preview_window(frame_queue, abort_flag):
    try:
        while not abort_flag.value:
            try:
                frame, camera_name = frame_queue.get_nowait()
                
                # Prüfe ob frame ein numpy array ist
                if not isinstance(frame, np.ndarray):
                    debug_print(f"Preview: Ungültiges Frame-Format empfangen: {type(frame)}")
                    continue
                    
                # Skalierung
                scale_percent = 25
                width = int(frame.shape[1] * scale_percent / 100)
                height = int(frame.shape[0] * scale_percent / 100)
                dim = (width, height)
                
                # Frame skalieren
                resized = cv2.resize(frame, dim, interpolation=cv2.INTER_AREA)
                
                # Fenster anzeigen
                cv2.imshow(f'Preview {camera_name}', resized)
                cv2.waitKey(1)
                
            except queue.Empty:
                cv2.waitKey(100)  # Warte 100ms wenn keine Frames verfügbar
                continue
                
    except Exception as e:
        debug_print(f"Preview: Kritischer Fehler: {str(e)}")
        debug_print(f"Preview: Stack Trace: {traceback.format_exc()}")
        
    finally:
        cv2.destroyAllWindows()

def get_user_input(prompt, default):
    user_input = input(f"{prompt} [{default}]: ")
    return user_input if user_input else default

def update_progress_bar(current, total, elapsed_time, remaining_time, light_status, live_status):
    """Aktualisiert die Fortschrittsanzeige"""
    width = 20
    progress = int(width * current / total)
    bar = '█' * progress + '-' * (width - progress)
    percent = current / total * 100
    
    # Entferne die Doppelung von "Licht" und "Live"
    print(f'\rVideo {current:3d} von {total:3d} [{bar}] {percent:5.1f}% | '
          f'Zeit: {format_time(elapsed_time)} | Rest: {format_time(remaining_time)} | '
          f'{light_status} | {live_status}', end='', flush=True)

def format_time(time_delta):
    """Formatiert eine Zeitdauer in HH:MM:SS"""
    if isinstance(time_delta, (int, float)):
        seconds = int(time_delta)
    else:  # timedelta
        seconds = int(time_delta.total_seconds())
    hours = seconds // 3600
    minutes = (seconds % 3600) // 60
    seconds = seconds % 60
    return f"{hours:02d}:{minutes:02d}:{seconds:02d}"

def capture_for_all_cameras(cameras, image_number, total_images, start_number, is_video, video_length, 
                          abort_flag, live_view_active, frame_queues, progress_queue, audio_playing):
    processes = []
    start_event = multiprocessing.Event()
    timestamp_queue = multiprocessing.Queue()
    timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    
    # Starte Prozesse für alle Kameras
    for i, camera in enumerate(cameras):
        process = multiprocessing.Process(
            target=capture_video if is_video else capture_image,
            args=(start_number + image_number - 1, video_length if is_video else total_images,
                  camera['device_index'], camera['name'], start_event if is_video else None,
                  timestamp_queue if is_video else None, frame_queues[i], abort_flag,
                  progress_queue if is_video else None, audio_playing)
        )
        processes.append(process)
        process.start()
        debug_print(f"Prozess für {camera['name']} gestartet")
    
    if is_video:
        # Gib allen Kameras Zeit zum Initialisieren
        time.sleep(1.0)
        debug_print("Sende Zeitstempel an alle Kameras")
        for _ in cameras:
            timestamp_queue.put(timestamp)
        debug_print("Starte Aufnahme")
        start_event.set()
    
    # Warte auf die Prozesse
    capture_timeout = video_length + 5 if is_video else 5  # Nur 5 Sekunden extra für die Aufnahme
    start_wait = time.perf_counter()
    
    # Warte zunächst auf die Aufnahme
    while any(p.is_alive() for p in processes):
        if time.perf_counter() - start_wait > capture_timeout:
            debug_print("Aufnahmezeit abgelaufen")
            break
        time.sleep(0.1)
    
    # Gib zusätzliche Zeit für das Schreiben der Videos
    write_timeout = 15  # 15 Sekunden für das Schreiben
    start_write = time.perf_counter()
    
    while any(p.is_alive() for p in processes):
        if time.perf_counter() - start_write > write_timeout:
            debug_print("Schreibzeit abgelaufen")
            break
        time.sleep(0.1)
    
    # Beende Prozesse
    for process in processes:
        if process.is_alive():
            debug_print(f"Beende Prozess {process.name}")
            process.terminate()
            process.join(timeout=1)
    
    # Überprüfe Ergebnis
    if is_video:
        for camera in cameras:
            final_filename = os.path.join("Videos", f"video_{start_number + image_number - 1:06d}_{camera['name']}_{timestamp}.mp4")
            
            if os.path.exists(final_filename):
                debug_print(f"Video {final_filename} erfolgreich erstellt")
            else:
                debug_print(f"Keine Videodatei für {camera['name']} gefunden")

    debug_print(f"Aufnahme {image_number} von {total_images} abgeschlossen")

def get_parameters():
    try:
        num_cameras = int(input("Anzahl der Kameras (1 oder 2) [2]: ") or "2")
        if num_cameras not in [1, 2]:
            raise ValueError("Ungültige Anzahl Kameras")
        
        video_length = int(input("Videolänge (Sekunden) [10]: ") or "10")
        if video_length < 1:
            raise ValueError("Ungültige Videolänge")
            
        interval = int(input("Intervall (Sekunden) [15]: ") or "15")
        if interval <= video_length:
            raise ValueError("Intervall muss größer als Videolänge sein")
            
        total_time = int(input("Gesamtzeit (Minuten) [2]: ") or "2")
        if total_time < 1:
            raise ValueError("Ungültige Gesamtzeit")
            
        return {
            'num_cameras': num_cameras,
            'mode': 'video',  # Fest auf 'video' gesetzt
            'video_length': video_length,
            'start_number': 1,  # Fest auf 1 gesetzt
            'interval': interval,
            'total_time': total_time
        }
    except ValueError as e:
        print(f"\nFehler: {e}")
        return None

def configure_camera(device):
    try:
        nodemap = device.RemoteDevice().NodeMaps()[0]
        
        # Belichtungszeit setzen
        exposure = nodemap.FindNode('ExposureTime')
        current_exposure = exposure.Value()
        debug_print(f"Aktuelle Belichtungszeit: {current_exposure}µs")
        
        if current_exposure < 10000:  # Wenn zu niedrig
            exposure.SetValue(10000)  # 10ms Mindestbelichtung
            debug_print(f"Belichtungszeit korrigiert auf: {exposure.Value()}µs")
            
        # Gain prüfen
        gain = nodemap.FindNode('Gain')
        debug_print(f"Aktueller Gain: {gain.Value()}")
        
    except Exception as e:
        debug_print(f"Fehler bei Kamera-Konfiguration: {str(e)}")
        raise

# Kamera-spezifische Einstellungen
def initialize_camera(device, camera_name):
    try:
        nodemap = device.RemoteDevice().NodeMaps()[0]
        
        # Belichtungszeit prüfen und setzen
        exposure = nodemap.FindNode('ExposureTime')
        current_exposure = exposure.Value()
        debug_print(f"{camera_name}: Aktuelle Belichtungszeit: {current_exposure}µs")
        
        if current_exposure < 10000:  # Wenn unter 10ms
            target_exposure = 10000
            exposure.SetValue(target_exposure)
            debug_print(f"{camera_name}: Belichtungszeit auf {target_exposure}µs gesetzt")
            
        # Gain prüfen
        gain = nodemap.FindNode('Gain')
        current_gain = gain.Value()
        debug_print(f"{camera_name}: Aktueller Gain: {current_gain}")
        
        if current_gain < 1.0:  # Wenn Gain zu niedrig
            target_gain = 1.0
            gain.SetValue(target_gain)
            debug_print(f"{camera_name}: Gain auf {target_gain} gesetzt")
            
    except Exception as e:
        debug_print(f"{camera_name}: Fehler bei Kamera-Initialisierung: {str(e)}")
        raise

def main():
    multiprocessing.freeze_support()
    log_system_info()
    
    # Shared Variables
    audio_playing = multiprocessing.Value('b', False)
    continuous_audio = multiprocessing.Value('b', True)
    abort_flag = multiprocessing.Value('b', False)
    live_view_active = multiprocessing.Value('b', True)
    
    # Initialize audio_process at the start
    audio_process = None
    
    try:
        print(f"{PROGRAM_NAME} v{VERSION}\nErstelldatum: {CREATION_DATE}\n")
        
        params = get_parameters()
        if params is None:
            return

        num_cameras = params['num_cameras']
        camera_ok, error = check_camera_availability(num_cameras)
        if not camera_ok:
            print(error)
            return

        capture_mode = params['mode']
        video_length = params['video_length']
        start_number = params['start_number']
        interval_seconds = params['interval']
        total_minutes = params['total_time']

        interval = timedelta(seconds=interval_seconds)
        total_time = timedelta(minutes=total_minutes)
        total_captures = int(total_time.total_seconds() // interval.total_seconds())

        capture_start_time = datetime.now().replace(microsecond=0) + timedelta(seconds=1)
        end_time = capture_start_time + total_time

        print(f"\nStart: {capture_start_time.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"Ende:  {end_time.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"Geplante Aufnahmen pro Kamera: {total_captures}\n")
        print("<L> = Live-Ansicht, <T> = Ton")

        initialize_audio()
        cameras = [{'device_index': i, 'name': f'Camera{i+1}'} for i in range(num_cameras)]
        frame_queues = [multiprocessing.Queue(maxsize=2) for _ in cameras]
        progress_queue = multiprocessing.Queue()

        # Starte Preview-Prozesse für beide Kameras
        preview_processes = []
        for i, camera in enumerate(cameras):
            preview_process = multiprocessing.Process(
                target=preview_window, 
                args=(frame_queues[i], abort_flag)
            )
            preview_process.start()
            preview_processes.append(preview_process)

        # Zeige sofort den initialen Status an
        current_time = datetime.now()
        last_update_time = current_time  # Initialisiere last_update_time
        update_progress_bar(1, total_captures, 0, total_time, "Licht an", "Live an")

        if continuous_audio.value:
            audio_process = multiprocessing.Process(target=play_audio, args=(audio_playing, abort_flag))
            audio_process.start()

        # Dann die Schleife für die Aufnahmen
        debug_print(f"Starte Hauptschleife mit {total_captures} Aufnahmen")
        for current_capture in range(total_captures):
            if abort_flag.value:
                debug_print("Abort-Flag gesetzt, beende Schleife")
                break

            capture_start = time.perf_counter()
            debug_print(f"\nStarte Aufnahme {current_capture + 1} von {total_captures}")
            
            # Starte die Aufnahme
            capture_for_all_cameras(cameras, current_capture + 1, total_captures, start_number,
                                  capture_mode == "video", video_length, abort_flag,
                                  live_view_active, frame_queues, progress_queue, audio_playing)

            capture_duration = time.perf_counter() - capture_start
            debug_print(f"Aufnahme {current_capture + 1} dauerte {capture_duration:.1f}s")

            # Berechne die verbleibende Wartezeit unter Berücksichtigung der Aufnahmedauer
            next_capture_time = capture_start_time + interval * (current_capture + 1)
            wait_time = (next_capture_time - datetime.now()).total_seconds()
            
            if wait_time > 0:
                debug_print(f"Warte {wait_time:.1f}s bis zur nächsten Aufnahme")
                time.sleep(wait_time)
            else:
                debug_print("Wartezeit bereits berschritten")

            # Warte bis zur nächsten geplanten Aufnahme
            debug_print(f"Warte bis zur nächsten Aufnahme um {next_capture_time}")
            while datetime.now() < next_capture_time:
                # Prüfe Benutzereingaben
                if msvcrt.kbhit():
                    key = msvcrt.getch().decode().lower()
                    if key == 'l':
                        live_view_active.value = not live_view_active.value
                        debug_print(f"Live-Ansicht {'aktiviert' if live_view_active.value else 'deaktiviert'}")
                    elif key == 't':
                        continuous_audio.value = not continuous_audio.value
                        debug_print(f"Audio {'aktiviert' if continuous_audio.value else 'deaktiviert'}")
                        if continuous_audio.value and not audio_process:
                            audio_process = multiprocessing.Process(target=play_audio, 
                                                                     args=(audio_playing, abort_flag))
                            audio_process.start()
                        elif not continuous_audio.value and audio_process:
                            stop_audio(audio_process)
                            audio_process = None

                # Aktualisiere die Anzeige alle 100ms
                if (datetime.now() - last_update_time).total_seconds() >= 0.1:
                    current_time = datetime.now()
                    elapsed = max(current_time - capture_start_time, timedelta(seconds=1))
                    remaining = max(end_time - current_time, timedelta(0))
                    progress = current_capture / total_captures
                    
                    update_progress_bar(current_capture + 1, total_captures, elapsed, remaining, "Licht an", "Live an")
                    last_update_time = current_time
                    debug_print(f"Progress aktualisiert: {current_capture + 1}/{total_captures} - Verbleibend: {remaining}")
                
                time.sleep(0.05)

            if abort_flag.value:
                break

            # Überprüfe, ob wir noch im Zeitplan sind
            if datetime.now() > next_capture_time:
                debug_print(f"Warnung: Zeitplan überschritten bei Aufnahme {current_capture + 1}")
                # Korrigiere den Zeitplan
                missed_intervals = int((datetime.now() - next_capture_time).total_seconds() / interval.total_seconds()) + 1
                next_capture_time = datetime.now() + interval
                debug_print(f"Nächste Aufnahme verschoben auf {next_capture_time}")

            # Aktualisiere die Anzeige ein letztes Mal für diese Aufnahme
            current_time = datetime.now()
            elapsed = max(current_time - capture_start_time, timedelta(seconds=1))
            remaining = max(end_time - current_time, timedelta(0))
            update_progress_bar(current_capture + 1, total_captures, elapsed, remaining, "Licht an", "Live an")

    except KeyboardInterrupt:
        debug_print("Programm durch Benutzer beendet")
    finally:
        abort_flag.value = True
        if audio_process:
            stop_audio(audio_process)

        # Beende alle Preview-Prozesse
        for preview_process in preview_processes:
            preview_process.join(timeout=5)
            if preview_process.is_alive():
                preview_process.terminate()
        
        cv2.destroyAllWindows()
        pygame.mixer.quit()

        print("\nAufnahme beendet.")

if __name__ == "__main__":
    main()
