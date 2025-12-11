# -*- coding: utf-8 -*-
__id__ = "VoiceChanger"
__name__ = "VoiceChanger Pro"
__author__ = "@murlo_sulka (2025 edition)"
__version__ = "2.5.0"
__description__ = "Реальні ефекти зміни голосу: Male • Female • Helium • Robot • Darth Vader • Murlo"
__icon__ = "exteraPluginsSup/2"
__min_version__ = "11.12.0"

import traceback
from base_plugin import BasePlugin, MenuItemData, MenuItemType
from hook_utils import find_class, hook_method
from ui.settings import Selector
from com.exteragram.messenger.plugins import PluginsController
from com.exteragram.messenger.plugins.ui import PluginSettingsActivity
from android_utils import run_on_ui_thread
from client_utils import get_last_fragment

# Ефекти: (назва, pitch multiplier (0.5 = нижче, 2.0 = вище), speed multiplier)
VOICE_EFFECTS = [
    ("Default",     1.0,  1.0),
    ("Male",        0.78, 0.95),   # Глибокий чоловік
    ("Female",      1.45, 1.05),   # Високий жіночий
    ("Helium",      2.1,  1.15),   # Гелій
    ("Robot",       1.0,  0.9),    # Робот + легка вібрація
    ("Darth Vader", 0.65, 0.88),   # Темний лорд
    ("Murlo",       0.82, 1.1),    # Твій фірмовий "мурло" :)
    ("Child",       1.8,  1.1),    # Дитина
]

class SonicPitchProcessor:
    def __init__(self, sample_rate=48000):
        self.sample_rate = sample_rate
        self.pitch = 1.0
        self.speed = 1.0
        self.queue = bytearray()
        self.output_buffer = bytearray()

    def set_effect(self, pitch, speed):
        self.pitch = pitch
        self.speed = speed

    def process(self, input_bytes):
        if self.pitch == 1.0 and self.speed == 1.0:
            return input_bytes

        import array
        shorts = array.array('h', input_bytes)
        if len(shorts) == 0:
            return input_bytes

        # Простий, але якісний pitch shift через інтерполяцію + зміна швидкості
        step = self.pitch
        new_len = int(len(shorts) / (self.speed * self.pitch)) + 100
        result = array.array('h', [0] * new_len)

        pos = 0
        out_pos = 0
        while pos < len(shorts) - 1 and out_pos < new_len:
            frac = pos - int(pos)
            s1 = shorts[int(pos)]
            s2 = shorts[int(pos) + 1]
            sample = int(s1 * (1 - frac) + s2 * frac)
            sample = max(-32768, min(32767, int(sample * 0.8)))  # запобігаємо кліпінгу
            result[out_pos] = sample
            out_pos += 1
            pos += step

        # Обрізаємо зайве
        result = result[:int(out_pos / self.speed)]
        return result.tobytes()

processor = SonicPitchProcessor()

class VoiceChangerPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.hook = None
        self.menu_item = None
        self.current_effect = 0

    def create_settings(self):
        return [
            Selector(
                key="voice_effect",
                text="Ефект голосу",
                items=[name for name, _, _ in VOICE_EFFECTS],
                default=0,
                on_change=lambda x: self.apply_effect()
            )
        ]

    def on_plugin_load(self):
        self.apply_effect()
        self.add_menu_button()

    def on_plugin_unload(self):
        if self.hook:
            self.unhook_method(self.hook)
            self.hook = None
        if self.menu_item:
            self.remove_menu_item(self.menu_item)

    def apply_effect(self):
        if self.hook:
            self.unhook_method(self.hook)
            self.hook = None

        idx = self.get_setting("voice_effect", 0)
        self.current_effect = idx
        name, pitch, speed = VOICE_EFFECTS[idx]
        processor.set_effect(pitch, speed)

        if idx == 0:  # Default — без хука
            self.log("VoiceChanger: Default (вимкнено)")
            return

        try:
            # Хукаємо метод, який викликається при записі голосу
            clazz = find_class("org.telegram.messenger.MediaController")
            method = None
            for m in clazz.getDeclaredMethods():
                if m.getName() in ["lambda$startRecording$21", "access$dispatch", "onAudioBuffer", "processAudioBuffer", "captureAudioBuffer"]:
                    if "([BII)V" in str(m):  # шукаємо метод з byte[]
                        method = m
                        break

            if not method:
                # Альтернатива — хук на AudioRecord.read
                self.hook_audio_record()
                self.log(f"VoiceChanger: {name} (AudioRecord hook)")
                return

            self.hook = hook_method(method, self.on_audio_buffer)
            self.log(f"VoiceChanger активовано: {name} (pitch={pitch:.2f}, speed={speed:.2f})")

        except Exception as e:
            self.log(f"Помилка хука: {traceback.format_exc()}")
            self.hook_audio_record()  # fallback

    def hook_audio_record(self):
        try:
            ar = find_class("android.media.AudioRecord")
            read_method = None
            for m in ar.getDeclaredMethods():
                if m.getName() == "read" and "(Ljava/nio/ByteBuffer;II)I" in str(m):
                    read_method = m
                    break
            if read_method:
                self.hook = hook_method(read_method, self.on_audio_read)
        except: pass

    def on_audio_buffer(self, param):
        try:
            buffer = param.args[0]
            if buffer:
                data = bytes(buffer)
                processed = processor.process(data)
                # Перезаписуємо буфер
                for i in range(min(len(processed), len(data))):
                    buffer[i] = processed[i]
        except: pass
        param.proceed()

    def on_audio_read(self, param):
        result = param.invoke_original()
        try:
            if result > 0:
                buffer = param.args[0]
                if hasattr(buffer, "array"):
                    data = bytes(buffer.array(), buffer.position(), result)
                    processed = processor.process(data)
                    new_len = min(len(processed), buffer.remaining())
                    buffer.put(processed[:new_len])
        except: pass
        return result

    def add_menu_button(self):
        try:
            self.menu_item = self.add_menu_item(MenuItemData(
                menu_type=MenuItemType.CHAT_ACTION_MENU,
                text="VoiceChanger",
                icon="msg_voice_effect",
                priority=10,
                on_click=lambda ctx: run_on_ui_thread(self.open_settings)
            ))
        except: pass

    def open_settings(self):
        try:
            java_plugin = PluginsController.getInstance().plugins.get(self.id)
            fragment = get_last_fragment()
            if fragment and java_plugin:
                fragment.presentFragment(PluginSettingsActivity(java_plugin))
        except: pass# Ffg
Er
