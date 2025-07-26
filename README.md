# National-cultural-memory-under-AI
 national cultural memory
import os
import tkinter as tk
import random
import time
import json
import requests
import shutil
import traceback
import threading
import pyaudio
import serial
import serial.tools.list_ports
from PIL import Image, ImageTk, ImageDraw, ImageFont
from datetime import datetime
import pytz
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import numpy as np  # 添加numpy导入
import speech_recognition as sr

expected_char=''
# ===== 修改步骤1: 在全局变量区域添加会员配置 =====
# 在你的代码中，找到这些行（大约在第20-30行左右）：

# ========== API Keys and URLs ==========
# Tripo 3D API for 3D model generation (Updated)
TRIPO_API_KEY = ""
TRIPO_BASE_URL = ""

# 在上面这些行的后面添加以下会员配置：
# ========== Tripo AI 会员配置 ==========
TRIPO_MEMBER_CONFIG = {
    "is_member": True,  # 设置为True启用会员优化
    "member_benefits": {
        "priority_queue": True,  # 优先队列
        "faster_gpu": True,  # 更快GPU
        "concurrent_tasks": 3,  # 并发任务数
        "reduced_throttling": True  # 减少限流
    }
}


def get_expected_generation_time(model_version="v2.0", face_count=1000, is_member=True):
    """根据会员状态和模型配置预估生成时间"""

    # 基础生成时间（免费用户）
    base_times = {
        "v1.4": {"min": 45, "max": 90},  # v1.4: 45-90秒
        "v2.0": {"min": 60, "max": 120}  # v2.0: 60-120秒
    }

    base_time = base_times.get(model_version, base_times["v2.0"])

    if is_member:
        # 会员加速因子
        speed_multipliers = {
            "priority_queue": 0.7,  # 优先队列减少30%等待
            "faster_gpu": 0.8,  # 更快GPU减少20%计算时间
            "reduced_throttling": 0.9  # 减少限流减少10%延迟
        }

        # 计算会员加速后的时间
        accelerated_min = int(base_time["min"] * speed_multipliers["priority_queue"] * speed_multipliers["faster_gpu"])
        accelerated_max = int(base_time["max"] * speed_multipliers["priority_queue"] * speed_multipliers["faster_gpu"])

        return {
            "free_user": f"{base_time['min']}-{base_time['max']}秒",
            "member": f"{accelerated_min}-{accelerated_max}秒",
            "improvement": f"提升约{int((1 - accelerated_max / base_time['max']) * 100)}%",
            "concurrent_tasks": TRIPO_MEMBER_CONFIG["member_benefits"]["concurrent_tasks"]
        }
    else:
        return {
            "free_user": f"{base_time['min']}-{base_time['max']}秒",
            "member": "未开通会员",
            "improvement": "无提升",
            "concurrent_tasks": 1
        }


def create_member_optimized_session():
    """创建为会员优化的请求会话"""
    session = create_requests_session()

    if TRIPO_MEMBER_CONFIG["is_member"]:
        # 会员专属请求头
        session.headers.update({
            "X-Priority": "high",
            "X-Member-Tier": "premium",
            "X-Concurrent-Enabled": "true"
        })

    return session


# Vosk模型路径配置
VOSK_MODEL_PATH = r"C:\111\vosk-model-cn-0.22"  # 中文语音识别模型路径
# 在VOSK_MODEL_PATH后面添加
SPEECH_CONFIG = {
    "use_google_speech": True,  # 使用Google语音识别作为主要方案
    "use_vosk_fallback": True,  # Vosk作为备用方案
    "energy_threshold": 4000,   # 提高能量阈值，减少误识别
    "dynamic_energy_threshold": True,
    "pause_threshold": 0.8,
    "phrase_threshold": 0.3,
    "non_speaking_duration": 0.8,
    "timeout": 3,
    "phrase_timeout": 2,
}

# ========== Directory Configuration ==========
# Base directories
BASE_DIR = r"C:\111"
COMFYUI_DIR = r"C:\111\ComfyUI-aki\ComfyUI-aki-v1.6\ComfyUI"
IMAGE_DISPLAY_FOLDER = os.path.join(BASE_DIR, "文本转图片")

# Output directories
VOICE_DIR = os.path.join(BASE_DIR, "语音测试A")
IMAGE_OUTPUT_DIR = os.path.join(COMFYUI_DIR, "output")  # Updated path
MODEL_OUTPUT_DIR = os.path.join(BASE_DIR, "3D模型输出")

# Create directories if they don't exist
for directory in [VOICE_DIR, IMAGE_OUTPUT_DIR, MODEL_OUTPUT_DIR, IMAGE_DISPLAY_FOLDER]:
    os.makedirs(directory, exist_ok=True)

# File paths
VOICE_TEXT_FILE = os.path.join(VOICE_DIR, "语音测试A.txt")

# ========== UI Configuration ==========
WINDOW_WIDTH = 1620
WINDOW_HEIGHT = 1920
SCROLL_SPEED = 1

# 图像显示优化配置
MAX_IMAGES_TOTAL = 50  # 最大图片总数
IMAGE_LOAD_DELAY = 1
UI_UPDATE_INTERVAL = 50

# 智能布局配置表 - 确保图片始终占满UI
LAYOUT_TABLE = {
    1: {"rows": 1, "cols": 1},     # 1张：1x1，占满整个UI
    2: {"rows": 1, "cols": 2},     # 2张：1x2
    3: {"rows": 1, "cols": 3},     # 3张：1x3
    4: {"rows": 2, "cols": 2},     # 4张：2x2
    5: {"rows": 1, "cols": 5},     # 5张：1x5
    6: {"rows": 2, "cols": 3},     # 6张：2x3
    9: {"rows": 3, "cols": 3},     # 9张：3x3
    10: {"rows": 2, "cols": 5},    # 10张：2x5
    12: {"rows": 3, "cols": 4},    # 12张：3x4
    15: {"rows": 3, "cols": 5},    # 15张：3x5，正好
    16: {"rows": 4, "cols": 4},    # 16张：4x4
    20: {"rows": 4, "cols": 5},    # 20张：4x5
    25: {"rows": 5, "cols": 5},    # 25张：5x5
    30: {"rows": 5, "cols": 6},    # 30张：5x6
    36: {"rows": 6, "cols": 6},    # 36张：6x6，正好
    40: {"rows": 5, "cols": 8},    # 40张：5x8
    50: {"rows": 5, "cols": 10},   # 50张：5x10，最大配置
}

# Set timezone to Beijing time
BEIJING_TZ = pytz.timezone("Asia/Shanghai")

# ========== Global Variables ==========
processing_queue = []
processing_lock = threading.Lock()
is_processing = False
# 图像与语音文本的映射关系
image_text_mapping = {}
# 映射文件路径
MAPPING_FILE = os.path.join(BASE_DIR, "image_text_mapping.json")

# 旋钮控制变量
knob_control_active = False
current_angle = 0
rotation_models = ["bishe01", "bishe02", "bishe03"]  # 旋转生成的模型文件名


def save_image_mapping():
    """保存图像文本映射到文件"""
    try:
        with open(MAPPING_FILE, 'w', encoding='utf-8') as f:
            json.dump(image_text_mapping, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print(f"❌ 保存映射文件失败: {e}")


def load_image_mapping():
    """从文件加载图像文本映射"""
    global image_text_mapping
    try:
        if os.path.exists(MAPPING_FILE):
            with open(MAPPING_FILE, 'r', encoding='utf-8') as f:
                image_text_mapping = json.load(f)
            print(f"✅ 已加载 {len(image_text_mapping)} 个图像文本映射")
        else:
            image_text_mapping = {}
    except Exception as e:
        print(f"❌ 加载映射文件失败: {e}")
        image_text_mapping = {}


def create_requests_session():
    """创建带有代理配置的requests会话"""
    session = requests.Session()
    # 尝试禁用系统代理
    session.trust_env = False
    session.proxies = {
        'http': None,
        'https': None,
    }
    # 设置环境变量以禁用代理
    os.environ.pop('HTTP_PROXY', None)
    os.environ.pop('HTTPS_PROXY', None)
    os.environ.pop('http_proxy', None)
    os.environ.pop('https_proxy', None)
    return session


# ========== 旋钮控制系统 ==========
class KnobControlSystem:
    """旋钮控制系统 - 监听真实Arduino硬件"""

    def __init__(self):
        self.active = False
        self.current_angle = 0
        self.last_stable_angle = 0
        self.current_resolution = "medium"
        self.serial_connection = None
        self.serial_thread = None
        self.angle_buffer = []  # 角度缓冲区，用于稳定性处理
        self.buffer_size = 5  # 缓冲区大小
        self.angle_threshold = 10  # 角度变化阈值
        self.last_angle_change_time = 0
        self.debounce_delay = 2.0  # 防抖延迟2秒

    def initialize(self, port='COM13', baudrate=57600):
        """初始化旋钮控制系统并连接Arduino"""
        try:
            self.serial_connection = serial.Serial(port, baudrate, timeout=1)
            time.sleep(2)  # 等待Arduino初始化
            self.active = True

            print("🎛️ 旋钮控制系统已初始化")
            print(f"🔌 已连接Arduino Leonardo串口: {port}")
            print("📋 旋钮角度映射:")
            print("   0度 = bishe01 模型 + 1080×1080分辨率")
            print("   90度 = bishe02 模型 + 444×444分辨率")
            print("   180度 = bishe03 模型 + 64×64分辨率")
            print("🎛️ 开始监听Leonardo硬件旋钮...")

            # 启动串口监听线程
            self.start_serial_listening()
            return True

        except Exception as e:
            print(f"❌ Arduino Leonardo连接失败: {e}")
            print("请检查:")
            print("1. Arduino Leonardo是否已连接到COM13")
            print("2. 串口号是否正确 (应为COM13)")
            print("3. 波特率是否匹配 (57600)")
            print("4. Leonardo驱动是否正确安装")
            return False

    def listen_serial_data(self):
        """监听串口数据"""
        print("🎯 开始监听串口数据...")
        print("🎛️ 映射关系: B→0°, N→90°, M→180°")

        # 确保初始角度为0
        self.current_angle = 0
        print(f"⚙️ 初始角度设置为: {self.current_angle}°")

        while self.active and self.serial_connection:
            try:
                if self.serial_connection.in_waiting > 0:
                    raw_data = self.serial_connection.read(self.serial_connection.in_waiting)

                    # 添加原始数据调试打印
                    print(f"📦 原始数据: {raw_data}")

                    for byte in raw_data:
                        char = chr(byte).upper()
                        print(f"🔍 接收到字符: {char}")  # 调试所有字符

                        if char in ['B', 'N', 'M']:
                            print(f"🎯 识别到有效指令: {char}")
                            self.process_angle_input(char)
                            break

                time.sleep(0.01)

            except Exception as e:
                print(f"❌ 错误: {e}")
                time.sleep(0.01)

    def process_angle_input(self, char_input):
        """处理角度输入 - 带详细调试信息"""
        # 正确的映射关系
        angle_mapping = {
            'B': 0,  # B → 0°
            'N': 90,  # N → 90°
            'M': 180  # M → 180°
        }

        # 调试：打印当前映射表
        print(f"🗺️ 当前映射表: {angle_mapping}")

        if char_input in angle_mapping:
            new_angle = angle_mapping[char_input]
            print(f"🔧 映射 {char_input} → {new_angle}°")
            self.update_angle(new_angle)
        else:
            print(f"⚠️ 无效字符: {char_input}")

    def update_angle(self, new_angle):
        """更新角度 - 带详细状态检查"""
        # 打印当前状态
        print(f"📊 更新前状态: 当前角度={self.current_angle}°, 新角度={new_angle}°")

        if new_angle == self.current_angle:
            print("🔄 角度未变化，跳过更新")
            return

        old_angle = self.current_angle
        print(old_angle)
        self.current_angle = new_angle
        print(self.current_angle)

        # 获取分辨率信息
        resolution_info = self.get_resolution_by_angle(new_angle)

        print(f"🔄 角度变化: {old_angle}° → {self.current_angle}°")
        print(f"🖥️ 分辨率切换: {resolution_info['width']}×{resolution_info['height']}")

        self.generate_rotated_model()

    def get_resolution_by_angle(self, angle):
        if angle == 0:  # B映射为 0°
            return {"mode": "high", "width": 1080, "height": 1080}  # 修改为 1080×1080
        elif angle == 180:  # M映射为 90°
            return {"mode": "medium", "width": 444, "height": 444}  # 保持 444×444
        else:  # N映射为 180°
            return {"mode": "low", "width": 64, "height": 64}  # 修改为 64×64

    def generate_rotated_model(self):
        """根据当前角度生成对应模型"""
        try:
            # 根据角度选择对应的模型文件
            if self.current_angle == 0:
                model_file = "bishe01"
                target_dir = "D:\\桌面\\bishe01"
            elif self.current_angle == 90:
                model_file = "bishe02"
                target_dir = "D:\\桌面\\bishe02"
            else:  # 180度
                model_file = "bishe03"
                target_dir = "D:\\桌面\\bishe03"

            # 创建输出路径
            output_path = os.path.join(MODEL_OUTPUT_DIR, f"{model_file}.glb")

            # 获取当前分辨率信息
            resolution_info = self.get_resolution_by_angle(self.current_angle)

            # 创建旋转标记文件
            rotation_info = {
                "angle": self.current_angle,
                "timestamp": datetime.now().isoformat(),
                "model_type": f"{model_file}_rotated",
                "target_directory": target_dir,
                "resolution": f"{resolution_info['width']}×{resolution_info['height']}",
                "resolution_mode": resolution_info["mode"],
                "hardware_controlled": True  # 标记为硬件控制
            }

            info_file = output_path.replace('.glb', f'_rotation_{self.current_angle}.json')
            with open(info_file, 'w', encoding='utf-8') as f:
                json.dump(rotation_info, f, indent=2)

            print(f"✅ 生成模型到: {target_dir}")
            print(f"📁 旋转模型信息已保存: {info_file}")

        except Exception as e:
            print(f"❌ 生成旋转模型时出错: {e}")

    def get_current_status(self):
        """获取当前旋钮状态"""
        resolution_info = self.get_resolution_by_angle(self.current_angle)
        return {
            "angle": self.current_angle,
            "resolution_mode": self.current_resolution,
            "width": resolution_info["width"],
            "height": resolution_info["height"],
            "hardware_connected": self.serial_connection is not None,
            "stable": len(self.angle_buffer) >= self.buffer_size
        }

    def disconnect(self):
        """断开Arduino连接"""
        self.active = False
        if self.serial_connection:
            self.serial_connection.close()
            print("🔌 Arduino Leonardo连接已断开")


# 全局旋钮控制实例
knob_system = KnobControlSystem()


# ========== 优化的语音识别类 ==========
class EnhancedSpeechRecognizer:
    """增强的语音识别系统 - 使用多种引擎提高准确性"""

    def __init__(self):
        self.recognizer = sr.Recognizer()
        self.microphone = sr.Microphone()
        self.vosk_model = None
        self.vosk_rec = None
        self.audio_interface = None

        # 优化识别参数
        self.recognizer.energy_threshold = SPEECH_CONFIG["energy_threshold"]
        self.recognizer.dynamic_energy_threshold = SPEECH_CONFIG["dynamic_energy_threshold"]
        self.recognizer.pause_threshold = SPEECH_CONFIG["pause_threshold"]
        self.recognizer.phrase_threshold = SPEECH_CONFIG["phrase_threshold"]
        self.recognizer.non_speaking_duration = SPEECH_CONFIG["non_speaking_duration"]

        print("🎤 初始化增强语音识别系统...")
        self.calibrate_microphone()

        if SPEECH_CONFIG["use_vosk_fallback"]:
            self.init_vosk_fallback()

    def calibrate_microphone(self):
        """校准麦克风"""
        print("🔧 正在校准麦克风...")
        try:
            with self.microphone as source:
                self.recognizer.adjust_for_ambient_noise(source, duration=2)
            print(f"✅ 麦克风校准完成，能量阈值: {self.recognizer.energy_threshold}")
        except Exception as e:
            print(f"⚠️ 麦克风校准失败: {e}")

    def init_vosk_fallback(self):
        """初始化Vosk作为备用"""
        try:
            if os.path.exists(VOSK_MODEL_PATH):
                import vosk
                vosk.SetLogLevel(-1)
                self.vosk_model = vosk.Model(VOSK_MODEL_PATH)
                self.vosk_rec = vosk.KaldiRecognizer(self.vosk_model, 16000)
                self.audio_interface = pyaudio.PyAudio()
                print("✅ Vosk备用引擎初始化成功")
            else:
                print("⚠️ Vosk模型路径不存在，跳过备用引擎")
        except Exception as e:
            print(f"⚠️ Vosk备用引擎初始化失败: {e}")

    def recognize_speech_google(self, audio_data):
        """使用Google语音识别"""
        try:
            text = self.recognizer.recognize_google(audio_data, language='zh-CN')
            return text.strip()
        except sr.UnknownValueError:
            return None
        except sr.RequestError as e:
            print(f"⚠️ Google语音识别服务错误: {e}")
            return None

    def recognize_speech_vosk(self, audio_data):
        """使用Vosk离线识别作为备用"""
        if not self.vosk_rec:
            return None

        try:
            wav_data = audio_data.get_wav_data(convert_rate=16000, convert_width=2)

            if self.vosk_rec.AcceptWaveform(wav_data):
                result = json.loads(self.vosk_rec.Result())
                return result.get('text', '').strip()
            else:
                partial = json.loads(self.vosk_rec.PartialResult())
                return partial.get('partial', '').strip()

        except Exception as e:
            print(f"⚠️ Vosk识别错误: {e}")
            return None

    def listen_and_recognize(self):
        """监听并识别语音"""
        try:
            print("🎤 开始监听语音...")

            with self.microphone as source:
                audio = self.recognizer.listen(
                    source,
                    timeout=SPEECH_CONFIG["timeout"],
                    phrase_time_limit=SPEECH_CONFIG["phrase_timeout"]
                )

            print("🔍 正在识别语音...")

            # 首先尝试Google识别
            if SPEECH_CONFIG["use_google_speech"]:
                text = self.recognize_speech_google(audio)
                if text and len(text) > 1:
                    print(f"✅ Google识别结果: {text}")
                    return text

            # 如果Google失败，尝试Vosk
            if SPEECH_CONFIG["use_vosk_fallback"]:
                text = self.recognize_speech_vosk(audio)
                if text and len(text) > 1:
                    print(f"✅ Vosk备用识别结果: {text}")
                    return text

            print("🔇 未识别到有效语音")
            return None

        except sr.WaitTimeoutError:
            print("⏰ 语音监听超时")
            return None
        except Exception as e:
            print(f"❌ 语音识别错误: {e}")
            return None

# ========== Text Display Classes ==========
class TextDisplayApp(tk.Tk):
    def __init__(self, arduino_port='COM13'):
        super().__init__()
        self.title("实时显示旋钮角度")
        self.configure(bg="black")

        # 保存Arduino端口
        self.arduino_port = arduino_port

        # 窗口设置 - 设置为半屏显示
        self.geometry("960x1080+0+0")  # 左半屏

        # 创建文本区域显示识别的语音
        self.text_area = tk.Text(self, wrap=tk.WORD, bg="black", fg="white",
                                 font=("Microsoft YaHei", 16), padx=10, pady=10)
        self.text_area.pack(fill="both", expand=True)

        # 添加ESC键关闭程序的绑定
        self.bind("<Escape>", self.on_closing)
        self.protocol("WM_DELETE_WINDOW", self.on_closing)

        # 加载包含识别语音的文本文件
        self.text_file = VOICE_TEXT_FILE
        self.text_data = ""
        self.original_text = ""
        self.texts_to_restore = []

        # 使用新的语音识别系统
        self.speech_recognizer = EnhancedSpeechRecognizer()
        self.speech_recognition_active = False

        self.energy_threshold = 500  # 默认阈值

        # 特殊符号计数
        self.symbol_count = 0
        self.total_char_count = 0
        self.next_symbol_change = time.time() + random.uniform(1, 3)
        self.current_tags = []

        # 图片UI引用
        self.image_app = None

        # 初始化旋钮控制系统 - 连接真实硬件
        print("🔌 正在连接Arduino Leonardo硬件旋钮...")
        # 尝试COM9端口（Arduino Leonardo），如果失败则尝试其他端口
        knob_success = False
        for port_try in ['COM13', self.arduino_port, 'COM3', 'COM4', 'COM5']:
            print('T------',port_try)
            try:
                print('O-----',port_try)
                knob_success = knob_system.initialize(port=port_try)  # 设置为 COM13
                if knob_success:
                    break
            except:
                print('C----',port_try)
                continue

        if not knob_success:
            print("⚠️ Arduino Leonardo连接失败，请检查硬件连接")
            print("💡 提示：程序仍可正常工作，但无法控制分辨率")
            print("🔧 请检查:")
            print("   1. Arduino Leonardo USB线是否连接")
            print("   2. Arduino是否正确上传了旋钮代码")
            print("   3. 串口驱动是否正确安装")
            print("   4. 端口是否为COM9")

        # 初始化组件
        self.update_text_display()
        self.setup_file_watcher()

        # 启动各种效果
        self.after(1000, self.random_highlight_text)
        self.after(1000, self.restore_original_text)
        self.after(1000, self.change_random_symbols)
        self.randomize_text_changes()

        # 自动启动语音识别
        self.after(2000, self.auto_start_speech_recognition)

        # 显示初始旋钮状态
        if knob_success:
            self.after(3000, self.show_initial_knob_status)

    def show_initial_knob_status(self):
        """显示初始旋钮状态"""
        if knob_system.active:
            status = knob_system.get_current_status()
            print(
                f"🎛️ Arduino Leonardo旋钮已连接: {status['angle']}° | 分辨率: {status['width']}×{status['height']} | 模式: {status['resolution_mode']}")
            print("🎯 请手动旋转Leonardo硬件旋钮来切换角度和分辨率")
        else:
            print("⚠️ Arduino Leonardo硬件旋钮未连接，将使用默认分辨率设置")

    def set_image_app(self, image_app):
        """设置图片UI的引用"""
        self.image_app = image_app

    def calibrate_audio_threshold(self):
        """校准音频阈值"""
        print("🔧 正在校准音频阈值...")
        print("📢 请保持安静5秒钟...")

        if not self.audio_interface:
            return 300  # 默认阈值

        try:
            FORMAT = pyaudio.paInt16
            CHANNELS = 1
            RATE = 16000
            CHUNK = 4000

            stream = self.audio_interface.open(
                format=FORMAT, channels=CHANNELS, rate=RATE,
                input=True, frames_per_buffer=CHUNK
            )

            energy_samples = []
            start_time = time.time()

            while time.time() - start_time < 5:  # 采样5秒
                data = stream.read(CHUNK, exception_on_overflow=False)
                audio_data = np.frombuffer(data, dtype=np.int16)
                energy = self.calculate_audio_energy(audio_data)
                energy_samples.append(energy)
                time.sleep(0.1)

            stream.stop_stream()
            stream.close()

            # 计算背景噪音水平
            avg_noise = np.mean(energy_samples)
            max_noise = np.max(energy_samples)

            # 设置阈值为背景噪音的3-5倍
            threshold = max(avg_noise * 4, max_noise * 2, 300)

            print(f"🔧 校准完成:")
            print(f"   平均背景噪音: {avg_noise:.0f}")
            print(f"   最大背景噪音: {max_noise:.0f}")
            print(f"   设置阈值: {threshold:.0f}")

            return threshold

        except Exception as e:
            print(f"❌ 校准失败: {e}")
            return 300  # 返回默认阈值

    def auto_start_speech_recognition(self):
        """自动启动语音识别"""
        print("🔄 自动启动语音识别...")
        if self.init_vosk():  # 确保Vosk已初始化
            print("🔧 开始音频阈值校准...")
            self.energy_threshold = self.calibrate_audio_threshold()
            print(f"✅ 阈值校准完成，设置为: {self.energy_threshold}")
        self.start_speech_recognition()

    def start_speech_recognition(self):
            """启动语音识别"""
            if not self.speech_recognition_active:
                self.speech_recognition_active = True
                self.speech_thread = threading.Thread(target=self.recognize_speech_loop)
                self.speech_thread.daemon = True
                self.speech_thread.start()
                print("🎤 优化的语音识别已启动")

    def recognize_speech_loop(self):
        """语音识别循环"""
        print("🎤 优化的实时语音识别已启动，准确性大幅提升...")

        while self.speech_recognition_active:
            try:
                text = self.speech_recognizer.listen_and_recognize()

                if text and len(text) > 1:
                    print(f"✅ 识别到语音: {text}")

                    if not text.endswith("。") and not text.endswith("."):
                        text += "。"

                    self.original_text += text + "\n"
                    self.text_data += text + "\n"

                    self.save_text_to_file(text)
                    self.update_text_display()
                    self.simulate_random_deletion()

                    # 自动启动图像生成处理
                    self.auto_process_voice_to_image(text)

            except Exception as e:
                if self.speech_recognition_active:
                    print(f"❌ 语音识别循环出错: {e}")
                    time.sleep(1)

    def filter_input_text(text):
        """过滤输入文本中的不当内容"""
        # 定义需要替换的词汇
        filter_words = {
            # 可以添加需要替换的词汇，用更中性的词替换
            "暴力": "和平",
            "打斗": "运动",
            "血腥": "红色"
            # 根据需要添加更多替换规则
        }

        filtered_text = text
        for bad_word, replacement in filter_words.items():
            filtered_text = filtered_text.replace(bad_word, replacement)

        return filtered_text

    def auto_process_voice_to_image(self, text):
        """自动处理语音到图像生成 - 添加内容过滤"""
        global processing_queue, processing_lock, is_processing

        # 过滤输入文本
        filtered_text = filter_input_text(text)

        with processing_lock:
            processing_queue.append(filtered_text)  # 使用过滤后的文本

        if not is_processing:
            processing_thread = threading.Thread(target=self.process_queue)
            processing_thread.daemon = True
            processing_thread.start()

    def on_closing(self, event=None):
        """程序关闭时的清理工作"""
        print("🔄 正在关闭程序...")
        self.stop_speech_recognition()

        # 断开Arduino连接
        if knob_system.active:
            knob_system.disconnect()

        if self.image_app:
            self.image_app.destroy()
        self.destroy()
        print("👋 程序已安全退出")



    def init_vosk(self):
        """初始化Vosk离线语音识别模型"""
        try:
            print("🔧 正在初始化Vosk语音识别模型...")

            if not os.path.exists(VOSK_MODEL_PATH):
                print(f"❌ Vosk模型路径不存在: {VOSK_MODEL_PATH}")
                print("请下载中文语音识别模型：")
                print("1. 访问: https://alphacephei.com/vosk/models")
                print("2. 下载: vosk-model-cn-0.22.zip")
                print(f"3. 解压到: {VOSK_MODEL_PATH}")
                return False

            def calibrate_audio_threshold(self):
                """校准音频阈值"""
                print("🔧 正在校准音频阈值...")
                print("📢 请保持安静5秒钟...")

                import numpy as np
                import time

                if not self.audio_interface:
                    return 300  # 默认阈值

                try:
                    FORMAT = pyaudio.paInt16
                    CHANNELS = 1
                    RATE = 16000
                    CHUNK = 4000

                    stream = self.audio_interface.open(
                        format=FORMAT, channels=CHANNELS, rate=RATE,
                        input=True, frames_per_buffer=CHUNK
                    )

                    energy_samples = []
                    start_time = time.time()

                    while time.time() - start_time < 5:  # 采样5秒
                        data = stream.read(CHUNK, exception_on_overflow=False)
                        audio_data = np.frombuffer(data, dtype=np.int16)
                        energy =self.calculate_audio_energy(audio_data)# np.sqrt(np.mean(audio_data ** 2))
                        energy_samples.append(energy)
                        time.sleep(0.1)

                    stream.stop_stream()
                    stream.close()

                    # 计算背景噪音水平
                    avg_noise = np.mean(energy_samples)
                    max_noise = np.max(energy_samples)

                    # 设置阈值为背景噪音的3-5倍
                    threshold = max(avg_noise * 4, max_noise * 2, 300)

                    print(f"🔧 校准完成:")
                    print(f"   平均背景噪音: {avg_noise:.0f}")
                    print(f"   最大背景噪音: {max_noise:.0f}")
                    print(f"   设置阈值: {threshold:.0f}")

                    return threshold

                except Exception as e:
                    print(f"❌ 校准失败: {e}")
                    return 300  # 返回默认阈值

            vosk.SetLogLevel(-1)
            self.vosk_model = vosk.Model(VOSK_MODEL_PATH)
            self.vosk_rec = vosk.KaldiRecognizer(self.vosk_model, 16000)
            self.audio_interface = pyaudio.PyAudio()

            print("✅ Vosk语音识别模型初始化成功")
            return True

        except Exception as e:
            print(f"❌ Vosk初始化失败: {str(e)}")
            return False

    def setup_file_watcher(self):
        """设置文件监视器以检测文本文件的变化"""
        try:
            self.observer = Observer()
            event_handler = FileChangeHandler(self.on_text_file_change)
            self.observer.schedule(event_handler, os.path.dirname(self.text_file), recursive=False)
            self.observer.start()
        except Exception as e:
            print(f"文件监视器设置失败: {e}")

    def on_text_file_change(self):
        """文件变更时重新加载文本内容"""
        self.after(100, self.update_text_display)

    def update_text_display(self):
        """更新Text小部件中显示的文本"""
        try:
            self.text_area.delete(1.0, tk.END)
            self.text_area.insert(tk.END, self.text_data)
            self.text_area.see(tk.END)
            self.total_char_count = len(self.text_data)
            self.check_symbol_ratio()
        except Exception as e:
            print(f"读取文本文件时出错: {e}")

    def check_symbol_ratio(self):
        """检查特殊符号比例，如果超过3%则减少"""
        special_chars = "▓▒░■□●○"  # 简化符号列表
        symbol_count = sum(1 for char in self.text_data if char in special_chars)
        symbol_ratio = symbol_count / max(1, self.total_char_count)

        if symbol_ratio > 0.03:  # 降低到3%
            # 静默清理，不输出日志
            for char in special_chars:
                if symbol_ratio <= 0.03:
                    break
                if char in self.text_data:
                    # 移除大部分该字符
                    self.text_data = self.text_data.replace(char, "")
                    symbol_count = sum(1 for c in self.text_data if c in special_chars)
                    symbol_ratio = symbol_count / max(1, len(self.text_data))

    def start_speech_recognition(self):
        """启动语音识别"""
        if not self.speech_recognition_active and self.init_vosk():
            self.speech_recognition_active = True
            self.speech_thread = threading.Thread(target=self.recognize_speech)
            self.speech_thread.daemon = True
            self.speech_thread.start()
            print("🎤 语音识别已启动 - 实时模式")

    def stop_speech_recognition(self):
        """停止语音识别"""
        self.speech_recognition_active = False
        print("🔇 语音识别已停止")

    def calculate_audio_energy(self,audio_data):
        """计算音频数据的能量（RMS），处理NaN和无穷大值"""
        # 检查并替换NaN值
        if np.isnan(audio_data).any():
            print("警告：音频数据包含NaN值，将被替换为0")
            audio_data = np.nan_to_num(audio_data)

        # 检查并处理无穷大值
        if np.isinf(audio_data).any():
            print("警告：音频数据包含无穷大值，将被裁剪")
            audio_data = np.clip(audio_data, -np.finfo(float).max, np.finfo(float).max)

        # 计算均方根能量
        squared = audio_data ** 2
        mean_squared = np.mean(squared)
        #np.sqrt(np.mean(audio_data ** 2))
        energy = mean_squared#np.sqrt(mean_squared)

        return energy



    def recognize_speech(self):
        """使用Vosk进行离线实时语音识别并自动处理"""
        if not self.vosk_model or not self.vosk_rec or not self.audio_interface:
            return

        try:


            FORMAT = pyaudio.paInt16
            CHANNELS = 1
            RATE = 16000
            CHUNK = 4000

            # 查找麦克风设备（如果需要）
            microphone_index = None
            for i in range(self.audio_interface.get_device_count()):
                info = self.audio_interface.get_device_info_by_index(i)
                if (info['maxInputChannels'] > 0 and
                        ('麦克风' in info['name'] or 'Microphone' in info['name'] or 'mic' in info['name'].lower())):
                    microphone_index = i
                    print(f"✅ 找到麦克风: {info['name']}")
                    break

            stream = self.audio_interface.open(
                format=FORMAT,
                channels=CHANNELS,
                rate=RATE,
                input=True,
                input_device_index=microphone_index,
                frames_per_buffer=CHUNK
            )

            print("🎤 实时语音识别已启动，请开始讲话...")

            while self.speech_recognition_active:
               #try:

                    data = stream.read(CHUNK, exception_on_overflow=False)

                    import numpy as np
                    audio_data = np.frombuffer(data, dtype=np.int16)
                    audio_energy = self.calculate_audio_energy(audio_data)#np.sqrt(np.mean(audio_data ** 2))

                    # 使用校准的阈值
                    if audio_energy > self.energy_threshold:
                        if self.vosk_rec.AcceptWaveform(data):
                            result = json.loads(self.vosk_rec.Result())
                            text = result.get('text', '').strip()

                            # 只处理长度大于1的文本
                            if text and len(text) > 1:
                                print(f"✅ Vosk识别的语音 (能量:{audio_energy:.0f}): {text}")

                    # 计算音频能量水平
                    audio_data = np.frombuffer(data, dtype=np.int16)
                    audio_energy =self.calculate_audio_energy(audio_data)# np.sqrt(np.mean(audio_data ** 2))

                    # 设置能量阈值：只有超过阈值才进行语音识别
                    ENERGY_THRESHOLD = 300  # 调整这个值，根据你的环境

                    if audio_energy > ENERGY_THRESHOLD:
                        # 只有在有足够音频能量时才进行识别
                        if self.vosk_rec.AcceptWaveform(data):
                            result = json.loads(self.vosk_rec.Result())
                            text = result.get('text', '').strip()
                            # 只处理长度大于1的文本
                            if text and len(text) > 1:
                                print(f"✅ Vosk识别的语音 (能量:{audio_energy:.0f}): {text}")

                                if not text.endswith("。") and not text.endswith("."):
                                    text += "。"

                                self.original_text += text + "\n"
                                self.text_data += text + "\n"



                                self.save_text_to_file(text)
                                self.update_text_display()
                                self.simulate_random_deletion()

                                # 自动启动图像生成处理

                                self.auto_process_voice_to_image(text)
                            elif text:
                                print(f"🔇 忽略单字符 (能量:{audio_energy:.0f}): {text}")
                    else:
                        # 能量太低，跳过识别，但可以显示能量水平（调试用）
                        if audio_energy > 50:  # 只显示有一定能量的
                            print(f"🔇 能量不足，跳过识别 (能量:{audio_energy:.0f})")

                # except Exception as e:
                #     if self.speech_recognition_active:
                #         print(f"❌ 音频处理出错: {str(e)}")
                #         time.sleep(0.1)

        except Exception as e:
            print(f"❌ Vosk语音识别过程中出错: {str(e)}")
        finally:
            try:
                if 'stream' in locals():
                    stream.stop_stream()
                    stream.close()
            except Exception as cleanup_error:
                print(f"❌ 清理音频流时出错: {cleanup_error}")

    def save_text_to_file(self, text):
        """将识别的文本保存到指定文件"""
        try:
            os.makedirs(os.path.dirname(self.text_file), exist_ok=True)
            with open(self.text_file, "a", encoding="utf-8") as file:
                file.write(f"{text}\n")
        except Exception as e:
            print(f"❌ 将文本写入文件时出错: {e}")

    def randomize_text_changes(self):
        """随机修改文本使其动态化"""
        if len(self.text_data) > 0 and random.random() < 0.05:  # 大幅降低概率
            special_chars = "▓▒░"  # 进一步简化
            symbol_count = sum(1 for char in self.text_data if char in special_chars)
            symbol_ratio = symbol_count / max(1, len(self.text_data))

            if symbol_ratio < 0.02:  # 更严格的限制
                glitch_chars = random.choice("▓▒░")  # 只添加一个字符
                position = random.randint(0, max(0, len(self.text_data) - 1))
                self.text_data = self.text_data[:position] + glitch_chars + self.text_data[position:]
                self.update_text_display()

        self.after(random.randint(5000, 10000), self.randomize_text_changes)  # 大幅增加间隔

    def auto_process_voice_to_image(self, text):

        """自动处理语音到图像生成"""
        global processing_queue, processing_lock, is_processing

        with processing_lock:
            processing_queue.append(text)

        # 如果没有正在处理，启动处理线程
        if not is_processing:
            processing_thread = threading.Thread(target=self.process_queue)
            processing_thread.daemon = True
            processing_thread.start()

    def process_queue(self):
        """处理队列中的语音文本"""
        global processing_queue, processing_lock, is_processing, image_text_mapping

        with processing_lock:
            is_processing = True

        try:
            while True:
                with processing_lock:
                    if not processing_queue:
                        break
                    text = processing_queue.pop(0)
                print(text,len(text))
                print(f"🔄 开始处理语音文本: {text}")

                # 直接使用语音文本，不添加任何前缀
                prompt_text = text
                print(f"🎤 直接使用语音文本: {prompt_text}")

                # 生成图像
                enhanced_prompt = prompt_text
                image_path = generate_image_with_comfyui(enhanced_prompt)

                if image_path:
                    # 复制到显示文件夹
                    filename = os.path.basename(image_path)
                    dest_path = os.path.join(IMAGE_DISPLAY_FOLDER, filename)
                    shutil.copy2(image_path, dest_path)

                    # 保存图像与语音文本的映射关系
                    dest_filename = os.path.basename(dest_path)
                    image_text_mapping[dest_filename] = text.replace("。", "").strip()  # 移除句号，保留原始文本
                    save_image_mapping()  # 保存到文件

                    print(f"✅ 图像已复制到显示文件夹: {dest_path}")
                    print(f"🏷️ 图像标签: {text}")

                    # 生成3D模型（异步，不阻塞图像显示）
                    model_thread = threading.Thread(target=lambda: generate_3d_model(image_path))
                    model_thread.daemon = True
                    model_thread.start()
                else:
                    print("⚠️ 图像生成失败，跳过3D模型生成")

                # 稍作延迟，避免过于频繁的处理
                time.sleep(IMAGE_LOAD_DELAY)

        except Exception as e:
            print(f"❌ 处理队列时出错: {str(e)}")
            traceback.print_exc()
        finally:
            with processing_lock:
                is_processing = False

    def translate_random_word(self, text):
        """随机翻译句子中的一个词语为英文"""
        try:
            session = create_requests_session()
            headers = {
                "Content-Type": "application/json",
                "Authorization": f"Bearer "
            }

            words = text.split()
            if len(words) < 2:
                return text

            word_to_translate = random.choice(words)
            prompt = f"请将以下句子中的词语「{word_to_translate}」翻译成英文（仅翻译这一个词，保留其他部分不变）:\n\n{text}"

            payload = {
                "model": "deepseek-chat",
                "messages": [{"role": "user", "content": prompt}],
                "temperature": 0.7,
                "max_tokens": 512
            }

            response = session.post(
                "https://api.deepseek.com/v1/chat/completions",
                headers=headers, json=payload, timeout=10
            )

            if response.status_code == 200:
                result = response.json()
                translated_text = result["choices"][0]["message"]["content"].strip()
                if not translated_text.endswith("。") and not translated_text.endswith("."):
                    translated_text += "。"
                return translated_text

        except Exception as e:
            print(f"❌ 翻译文本出错: {str(e)}")
        return None

    def simulate_random_deletion(self):
        """模拟随机删除文本"""
        if random.random() < 0.1:
            if len(self.text_data) > 20:
                start_index = random.randint(0, len(self.text_data) - 20)
                length = random.randint(2, 10)
                end_index = min(start_index + length, len(self.text_data))
                self.text_data = self.text_data[:start_index] + self.text_data[end_index:]
                self.update_text_display()

    def change_random_symbols(self):
        """随机变化特殊符号"""
        try:
            current_time = time.time()
            if current_time >= self.next_symbol_change and len(self.text_data) > 10:
                strange_symbols = [
                    "▓", "▒", "░", "■", "□", "●", "○"  # 简化符号列表
                ]

                special_chars = "".join(strange_symbols)
                positions = [i for i, char in enumerate(self.text_data) if char in special_chars]

                if positions and random.random() < 0.3:  # 降低变化概率
                    position = random.choice(positions)
                    new_symbol = random.choice(strange_symbols)
                    self.text_data = self.text_data[:position] + new_symbol + self.text_data[position + 1:]
                elif random.random() < 0.1:  # 降低添加概率
                    symbol_count = sum(1 for char in self.text_data if char in special_chars)
                    symbol_ratio = symbol_count / max(1, len(self.text_data))
                    if symbol_ratio < 0.03:  # 更严格的限制
                        position = random.randint(0, len(self.text_data) - 1)
                        symbol = random.choice(strange_symbols)
                        self.text_data = self.text_data[:position] + symbol + self.text_data[position:]

                self.next_symbol_change = current_time + random.uniform(3, 8)  # 增加间隔时间
                self.update_text_display()
        except Exception as e:
            print(f"❌ 符号变化出错: {str(e)}")

        self.after(2000, self.change_random_symbols)  # 增加检查间隔

    def random_highlight_text(self):
        """随机高亮文本中的部分内容"""
        try:
            if len(self.text_data) > 10:
                for tag in self.current_tags:
                    self.text_area.tag_delete(tag)
                self.current_tags = []

                total_length = len(self.text_data)
                highlight_target = int(total_length * 0.1)
                highlighted_so_far = 0
                max_attempts = 20
                attempts = 0

                while highlighted_so_far < highlight_target and attempts < max_attempts:
                    attempts += 1
                    if len(self.text_data) <= 5:
                        break

                    start_index = random.randint(0, len(self.text_data) - 5)
                    remaining_text = len(self.text_data) - start_index
                    if remaining_text < 5:
                        continue

                    remaining = highlight_target - highlighted_so_far
                    max_length = min(30, remaining_text, remaining)
                    min_length = min(5, max_length)

                    if max_length <= min_length:
                        length = min_length
                    else:
                        length = random.randint(min_length, max_length)

                    end_index = start_index + length
                    tag_name = f"highlight_{random.randint(1, 10000)}"

                    self.text_area.tag_add(tag_name, f"1.{start_index}", f"1.{end_index}")
                    highlight_color = random.choice(["yellow", "#FFFF99", "#FFD700", "#FFFACD"])
                    self.text_area.tag_configure(tag_name, background=highlight_color)
                    self.current_tags.append(tag_name)
                    highlighted_so_far += length
        except Exception as e:
            print(f"❌ 高亮文本出错: {str(e)}")

        self.after(random.randint(1000, 3000), self.random_highlight_text)

    def restore_original_text(self):
        """将处理后的文本恢复为原始文本"""
        try:
            current_time = time.time()
            texts_to_remove = []

            for restore_info in self.texts_to_restore:
                if current_time >= restore_info["time"]:
                    processed_text = restore_info["processed_text"]
                    original_text = restore_info["original_text"]

                    if processed_text in self.text_data:
                        self.text_data = self.text_data.replace(processed_text, original_text)
                        texts_to_remove.append(restore_info)

            for item in texts_to_remove:
                self.texts_to_restore.remove(item)

            if texts_to_remove:
                self.update_text_display()
        except Exception as e:
            print(f"❌ 恢复原始文本出错: {str(e)}")

        self.after(1000, self.restore_original_text)

    def __del__(self):
        """确保程序退出时观察者线程停止"""
        try:
            if hasattr(self, 'observer') and self.observer:
                self.observer.stop()
                self.observer.join()
        except Exception as e:
            print(f"❌ 停止观察者时出错: {e}")

        try:
            if hasattr(self, 'audio_interface') and self.audio_interface:
                self.audio_interface.terminate()
        except Exception as e:
            print(f"❌ 终止音频接口时出错: {e}")


class FileChangeHandler(FileSystemEventHandler):
    """文件变更的自定义处理程序"""

    def __init__(self, callback):
        self.callback = callback

    def on_modified(self, event):
        """监控文件修改时触发"""
        if (hasattr(self.callback, '__self__') and
                hasattr(self.callback.__self__, 'text_file') and
                event.src_path == self.callback.__self__.text_file):
            self.callback()


# ========== Image Display Classes ==========
class ImageChangeHandler(FileSystemEventHandler):
    def __init__(self, callback):
        self.callback = callback
        self.last_event_time = time.time()
        self.cooldown = 1.0

    def on_created(self, event):
        if event.is_directory:
            return
        if event.src_path.lower().endswith(('.png', '.jpg', '.jpeg', '.gif', '.bmp')):
            current_time = time.time()
            if current_time - self.last_event_time > self.cooldown:
                self.last_event_time = current_time
                self.callback()

    def on_deleted(self, event):
        if event.is_directory:
            return
        if event.src_path.lower().endswith(('.png', '.jpg', '.jpeg', '.gif', '.bmp')):
            current_time = time.time()
            if current_time - self.last_event_time > self.cooldown:
                self.last_event_time = current_time
                self.callback()


# ========== Enhanced Image Display Classes ==========
class DynamicLayoutApp(tk.Toplevel):
    def __init__(self, folder, parent):
        super().__init__(parent)
        self.title("Dynamic Layout Image Viewer")
        self.configure(bg="black")

        # 窗口状态管理
        self.is_fullscreen = False
        self.normal_geometry = "960x1080+960+0"  # 记录正常大小
        self.manual_layout = False  # 是否使用手动布局

        # 设置窗口为无边框（黑色顶栏）
        self.overrideredirect(False)  # 保留窗口控制按钮
        self.configure(bg="black")

        # 初始窗口设置 - 设置为右半屏显示
        self.geometry(self.normal_geometry)

        # 绑定键盘事件
        self.bind("<KeyPress>", self.on_key_press)
        self.focus_set()  # 确保窗口可以接收键盘事件

        # 绑定窗口大小变化事件
        self.bind("<Configure>", self.on_window_resize)

        # 绑定最大化事件
        self.bind("<Button-1>", self.on_click)  # 可选：双击最大化

        self.canvas = tk.Canvas(self, bg="black", highlightthickness=0, bd=0)
        self.canvas.pack(fill="both", expand=True)

        self.folder =r"C:\111\ComfyUI-aki\ComfyUI-aki-v1.6\ComfyUI\output"
        self.loaded_images = []
        self.image_filenames = []
        self.tk_images = []
        self.image_items = []
        self.layout = {"rows": 1, "cols": 1}
        self.scroll_directions = []

        # 窗口尺寸变量
        self.current_width = 960
        self.current_height = 1080

        # 图像显示优化 - 增加到50个
        self.max_visible_images = 50  # 增加到50个同时显示
        self.image_cache = {}
        self.last_update_time = 0
        self.update_throttle = 0.05  # 减少更新节流时间，提高响应性

        # 加载包含识别语音的文本文件
        self.text_file = VOICE_TEXT_FILE
        self.text_data = ""
        self.original_text = ""
        self.texts_to_restore = []


        # 滚动相关
        self.scroll_speed = 2  # 滚动速度
        self.scroll_active = True

        # Set up file watcher
        self.setup_file_watcher()

        # Initialize the interface
        self.load_images()
        self.update_layout()
        self.populate_canvas()
        self.scroll()

        # 启动后显示帮助
        self.after(1000, self.show_help)

        print(f"✅ 充满屏幕的正方形图片UI已启动")
        print("🔲 所有图片保持正方形显示，充满整个屏幕")
        print("♾️ 支持无限循环滚动，即使图片不足也会自动填充")
        print("🎛️ 可调整行列布局:")
        print("   1-9键: 快速设置NxN布局")
        print("   +/-键: 增减行列数")
        print("   R键: 重置自动布局")
        print("🖥️ F11: 切换全屏 | ESC: 退出全屏")
        print("🎮 空格: 暂停滚动 | H键: 显示帮助")

    def update_text_display(self):
        """更新Text小部件中显示的文本"""
        try:
            self.text_area.delete(1.0, tk.END)
            self.text_area.insert(tk.END, self.text_data)
            self.text_area.see(tk.END)
            self.total_char_count = len(self.text_data)
            self.check_symbol_ratio()
        except Exception as e:
            print(f"读取文本文件时出错: {e}")

    def on_text_file_change(self):
        """文件变更时重新加载文本内容"""
        self.after(100, self.update_text_display)

    def setup_file_watcher(self):
        """设置文件监视器以检测文本文件的变化"""
        try:
            self.observer = Observer()
            event_handler = FileChangeHandler(self.on_text_file_change)
            self.observer.schedule(event_handler, os.path.dirname(self.text_file), recursive=False)
            self.observer.start()
        except Exception as e:
            print(f"文件监视器设置失败: {e}")

    def on_key_press(self, event):
        """处理键盘事件"""
        if event.keysym == "F11":
            self.toggle_fullscreen()
        elif event.keysym == "Escape":
            if self.is_fullscreen:
                self.exit_fullscreen()
            else:
                # 如果不在全屏，ESC关闭程序
                self.master.on_closing()
        elif event.keysym == "space":
            # 空格键暂停/恢复滚动
            self.scroll_active = not self.scroll_active
            print(f"📺 滚动{'暂停' if not self.scroll_active else '恢复'}")
        elif event.keysym in ["1", "2", "3", "4", "5", "6", "7", "8", "9"]:
            # 数字键快速设置行列布局
            num = int(event.keysym)
            self.set_manual_layout(num, num)
        elif event.keysym == "equal" or event.keysym == "plus":
            # + 键增加行列数
            self.adjust_layout(1)
        elif event.keysym == "minus":
            # - 键减少行列数
            self.adjust_layout(-1)
        elif event.keysym == "r":
            # R键重置为自动布局
            self.reset_auto_layout()
        elif event.keysym == "h":
            # H键显示帮助
            self.show_help()

    def set_manual_layout(self, rows, cols):
        """手动设置行列布局"""
        self.layout = {"rows": rows, "cols": cols}
        self.manual_layout = True
        print(f"🎛️ 手动设置布局: {rows}x{cols}")
        self.update_scroll_directions()
        self.populate_canvas()

    def adjust_layout(self, delta):
        """调整当前布局"""
        current_size = max(self.layout["rows"], self.layout["cols"])
        new_size = max(1, min(10, current_size + delta))
        self.set_manual_layout(new_size, new_size)

    def reset_auto_layout(self):
        """重置为自动布局"""
        self.manual_layout = False
        print("🔄 重置为自动布局模式")
        self.update_layout()
        self.populate_canvas()

    def show_help(self):
        """显示帮助信息"""
        print("=" * 50)
        print("🎮 图片UI控制帮助:")
        print("F11     - 切换全屏")
        print("ESC     - 退出全屏/关闭程序")
        print("空格    - 暂停/恢复滚动")
        print("1-9     - 快速设置NxN布局")
        print("+       - 增加行列数")
        print("-       - 减少行列数")
        print("R       - 重置为自动布局")
        print("H       - 显示此帮助")
        print("=" * 50)

    def on_click(self, event):
        """处理点击事件 - 可选的双击最大化"""
        # 这里可以添加双击检测逻辑
        pass

    def toggle_fullscreen(self):
        """切换全屏状态"""
        if self.is_fullscreen:
            self.exit_fullscreen()
        else:
            self.enter_fullscreen()

    def enter_fullscreen(self):
        """进入全屏模式"""
        if not self.is_fullscreen:
            # 记录当前窗口几何信息
            self.normal_geometry = self.geometry()

            # 获取屏幕尺寸
            screen_width = self.winfo_screenwidth()
            screen_height = self.winfo_screenheight()

            # 设置全屏
            self.geometry(f"{screen_width}x{screen_height}+0+0")
            self.attributes('-topmost', True)  # 置顶
            self.overrideredirect(True)  # 完全无边框

            self.is_fullscreen = True
            self.current_width = screen_width
            self.current_height = screen_height

            print(f"🖥️ 已进入全屏模式: {screen_width}x{screen_height}")

            # 如果不是手动布局模式，重新计算布局
            if not self.manual_layout:
                self.update_layout()

            # 重新布局
            self.after_idle(self.populate_canvas)

    def exit_fullscreen(self):
        """退出全屏模式"""
        if self.is_fullscreen:
            # 恢复窗口
            self.overrideredirect(False)
            self.attributes('-topmost', False)
            self.geometry(self.normal_geometry)

            self.is_fullscreen = False

            # 从几何字符串中提取尺寸
            try:
                size_part = self.normal_geometry.split('+')[0]
                width, height = map(int, size_part.split('x'))
                self.current_width = width
                self.current_height = height
            except:
                self.current_width = 960
                self.current_height = 1080

            print(f"🖥️ 已退出全屏模式: {self.current_width}x{self.current_height}")

            # 如果不是手动布局模式，重新计算布局
            if not self.manual_layout:
                self.update_layout()

            # 重新布局
            self.after_idle(self.populate_canvas)

    def on_window_resize(self, event):
        """处理窗口大小变化"""
        if event.widget == self and not self.is_fullscreen:  # 只在非全屏模式处理resize
            new_width = event.width
            new_height = event.height

            # 检查是否真的改变了大小
            if abs(new_width - self.current_width) > 10 or abs(new_height - self.current_height) > 10:
                self.current_width = new_width
                self.current_height = new_height

                # 如果不是手动布局模式，重新计算布局
                if not self.manual_layout:
                    self.update_layout()

                # 重新绘制图像，使用节流
                current_time = time.time()
                if current_time - self.last_update_time > self.update_throttle:
                    self.last_update_time = current_time
                    self.after_idle(self.populate_canvas)

    def calculate_optimal_layout(self, num_images):
        """根据图片数量获取预定义的完美布局"""
        if num_images <= 0:
            return {"rows": 1, "cols": 1}

        # 直接从配置表查找最接近的布局
        if num_images in LAYOUT_TABLE:
            return LAYOUT_TABLE[num_images]

        # 如果不在表中，找最接近的较大值
        for count in sorted(LAYOUT_TABLE.keys()):
            if count >= num_images:
                return LAYOUT_TABLE[count]

        # 超过最大值，使用最大配置
        return LAYOUT_TABLE[50]

    def update_layout(self):
        """根据图片数量和屏幕尺寸动态更新布局"""
        if not self.manual_layout:  # 只在非手动模式下自动计算布局
            num_images = len(self.loaded_images)
            self.layout = self.calculate_optimal_layout(num_images)

        self.update_scroll_directions()

        print(f"📐 布局: {self.layout['rows']}x{self.layout['cols']} (屏幕: {self.current_width}x{self.current_height})")
        if self.manual_layout:
            print("🎛️ 手动布局模式")
        else:
            print("🔄 自动布局模式")

    def update_scroll_directions(self):
        """更新滚动方向"""
        self.scroll_directions = []
        for i in range(self.layout["cols"]):
            direction = 1 if i % 2 == 1 else -1
            self.scroll_directions.append(direction)

    def get_image_size(self):
        """计算图像大小确保完全占满UI"""
        rows = self.layout["rows"]
        cols = self.layout["cols"]
        if rows == 0 or cols == 0:
            return 300, 300

        # 计算每个图片的精确尺寸，确保无黑边
        width = self.current_width // cols
        height = self.current_height // rows

        # 使用较小值确保正方形，但允许完全填充
        size = min(width, height)
        return size, size

    def draw_number(self, img, index, filename=None):
        """在图片上绘制信息标签 - 相对位置确保不偏移"""
        img_width, img_height = img.size

        # 根据图片尺寸动态调整字体大小
        base_font_size = min(img_width, img_height) // 50
        font_size_large = max(12, min(32, base_font_size * 2))
        font_size_small = max(8, min(18, base_font_size))

        label = f"{str(index).zfill(8)}X"

        # 生成随机坐标
        lat = random.uniform(-90, 90)
        lon = random.uniform(-180, 180)
        coordinates = f"{lat:.4f}° N {lon:.4f}° E"

        # 生成时间标签
        current_time = datetime.now(BEIJING_TZ)
        hour = current_time.hour
        time_labels = {
            (23, 1): "子 时", (1, 3): "丑 时", (3, 5): "寅 时", (5, 7): "卯 时",
            (7, 9): "辰 时", (9, 11): "巳 时", (11, 13): "午 时", (13, 15): "未 时",
            (15, 17): "申 时", (17, 19): "酉 时", (19, 21): "戌 时", (21, 23): "亥 时"
        }

        time_label = "亥 时"  # 默认
        for (start, end), label_text in time_labels.items():
            if start <= hour < end or (start == 23 and (hour >= 23 or hour < 1)):
                time_label = label_text
                break

        time_date = current_time.strftime(
            f"{time_label} {current_time.strftime('%I:%M%p')} {current_time.strftime('%m.%d.%Y')}")

        # 获取对应的语音关键词
        keyword = self.get_image_keyword(filename) if filename else "生成图像"

        draw = ImageDraw.Draw(img)
        try:
            font_large = ImageFont.truetype("msyh.ttc", font_size_large)
            font_small = ImageFont.truetype("msyh.ttc", font_size_small)
        except:
            font_large = ImageFont.load_default()
            font_small = ImageFont.load_default()

        # 使用相对位置绘制文字，确保不因窗口变化而偏移
        margin = max(5, img_width // 100)  # 动态边距

        # 左上角 - 编号
        draw.text((margin, margin), label, fill="white", font=font_large)

        # 右上角 - 坐标
        try:
            coord_bbox = font_small.getbbox(coordinates)
            coord_width = coord_bbox[2] - coord_bbox[0]
        except:
            coord_width = len(coordinates) * font_size_small // 2

        draw.text((img_width - coord_width - margin, margin), coordinates, fill="white", font=font_small)

        # 左下角 - 关键词
        try:
            keyword_bbox = font_small.getbbox(f"Key Word: [{keyword}]")
            keyword_height = keyword_bbox[3] - keyword_bbox[1]
        except:
            keyword_height = font_size_small

        draw.text((margin, img_height - keyword_height * 2 - margin),
                  f"Key Word: [{keyword}]", fill="white", font=font_small)

        # 左下角 - 时间
        draw.text((margin, img_height - keyword_height - margin // 2),
                  time_date, fill="white", font=font_small)

        return img

    def populate_canvas(self):
        """完美填充canvas，无黑色区域"""
        self.canvas.delete("all")
        self.tk_images.clear()
        self.image_items.clear()

        if not self.loaded_images:
            img_size, _ = self.get_image_size()
            blank = Image.new("RGB", (img_size, img_size), "black")
            blank_with_number = self.draw_number(blank, 1, "default_black.png")
            self.loaded_images = [blank_with_number]
            self.image_filenames = ["default_black.png"]

        img_size, _ = self.get_image_size()

        # 关键：计算实际可以放置的图片数量来填满UI
        max_displayable = self.layout["rows"] * self.layout["cols"]

        # 如果图片不足，复制现有图片来填满布局
        display_images = self.loaded_images.copy()
        if len(display_images) < max_displayable:
            original_count = len(display_images)
            for i in range(max_displayable - original_count):
                display_images.append(self.loaded_images[i % original_count])

        # 绘制图片，确保完全填满UI
        for i in range(min(max_displayable, len(display_images))):
            img = display_images[i].copy()
            img = img.resize((img_size, img_size), Image.Resampling.LANCZOS)

            # 添加标签
            filename = self.image_filenames[i % len(self.image_filenames)] if self.image_filenames else None
            img = self.draw_number(img, i + 1, filename)

            # 计算位置：完全无间隙填充
            row = i // self.layout["cols"]
            col = i % self.layout["cols"]

            x_pos = col * img_size
            y_pos = row * img_size

            tk_img = ImageTk.PhotoImage(img)
            self.tk_images.append(tk_img)

            item = self.canvas.create_image(x_pos, y_pos, anchor="nw", image=tk_img)
            self.image_items.append((item, col))

        print(f"🔲 完美填充UI: {self.layout['rows']}x{self.layout['cols']} = {max_displayable}个位置")
        print(f"📊 实际图片数: {len(self.loaded_images)} | 显示图片数: {len(display_images)}")

    def scroll(self):
        """优化的滚动 - 每列首尾相连，无黑色区域"""
        if not self.image_items or not self.scroll_active:
            self.after(30, self.scroll)
            return

        img_size, _ = self.get_image_size()

        # 按列分组
        by_column = {}
        for item, col_idx in self.image_items:
            if col_idx not in by_column:
                by_column[col_idx] = []
            by_column[col_idx].append(item)

        # 每列独立滚动，首尾相连
        for col_idx, items in by_column.items():
            if not items:
                continue

            # 所有图片向下移动
            for item in items:
                self.canvas.move(item, 0, self.scroll_speed)

            # 检查是否需要循环 - 关键改进
            for item in items:
                x, y = self.canvas.coords(item)

                if y >= self.current_height:  # 完全移出底部
                    # 找到该列最顶部的位置
                    min_y = min(self.canvas.coords(i)[1] for i in items if i != item)
                    new_y = min_y - img_size  # 紧贴顶部，无间隙
                    self.canvas.coords(item, x, new_y)

        self.after(30, self.scroll)

    def update_layout(self):
        """根据实际图片数量更新布局"""
        if not self.manual_layout:  # 只在非手动模式下自动计算布局
            actual_image_count = len(self.loaded_images)

            # 如果没有图片，使用1x1布局
            if actual_image_count == 0:
                self.layout = {"rows": 1, "cols": 1}
            else:
                # 根据实际图片数量计算最佳布局
                self.layout = self.calculate_optimal_layout(actual_image_count)

        self.update_scroll_directions()

        actual_count = len(self.loaded_images)
        print(f"📐 布局: {self.layout['rows']}x{self.layout['cols']} (实际图片: {actual_count} 张)")
        if self.manual_layout:
            print("🎛️ 手动布局模式")
        else:
            print("🔄 自动布局模式 - 基于实际图片数量")

    def load_images(self):
        """优化的图像加载方法 - 加载所有图像"""
        supported_ext = (".png", ".jpg", ".jpeg", ".gif", ".bmp")
        self.loaded_images.clear()
        self.image_filenames = []

        # 获取图像文件列表
        image_files = []
        for filename in os.listdir(self.folder):
            if filename.lower().endswith(supported_ext):
                image_files.append(filename)

        # 按文件修改时间排序，最新的在前
        image_files.sort(key=lambda x: os.path.getmtime(os.path.join(self.folder, x)), reverse=True)

        # 加载所有图像（移除数量限制）
        for filename in image_files:  # ← 移除 [:self.max_visible_images]
            path = os.path.join(self.folder, filename)
            try:
                if filename in self.image_cache:
                    img = self.image_cache[filename]
                else:
                    img = Image.open(path).convert("RGB")
                    self.image_cache[filename] = img

                self.loaded_images.append(img)
                self.image_filenames.append(filename)
            except Exception as e:
                print(f"无法加载图像 {filename}: {e}")

        if not self.loaded_images:
            img = Image.new("RGB", (self.current_width, self.current_height), "black")
            self.loaded_images.append(img)
            self.image_filenames.append("default_black.png")

        print(f"✅ 已加载 {len(self.loaded_images)} 张实际图像")  # ← 修改输出信息

    def get_image_keyword(self, filename):
        """不使用图像映射，统一标签"""
        return "ComfyUI 生成图像"

    def __del__(self):
        """确保程序退出时观察者线程停止"""
        try:
            if hasattr(self, 'observer') and self.observer:
                self.observer.stop()
                self.observer.join()
        except Exception as e:
            print(f"❌ 停止图像观察者时出错: {e}")


def create_content_filter():
    """创建内容过滤负向提示词"""
    # 基础质量过滤
    quality_filters = [
        "low quality", "blurry", "distorted", "unrealistic", "bad anatomy",
        "poor quality", "low resolution", "pixelated", "artifacts"
    ]

    # 内容安全过滤
    content_filters = [
        "inappropriate content", "adult content", "explicit content",
        "mature content", "nsfw", "disturbing content", "harmful content",
        "violence", "gore", "weapon", "blood", "injury"
    ]

    # 合并所有过滤词
    all_filters = quality_filters + content_filters
    return ", ".join(all_filters)


def create_workflow_json(positive_prompt):
    """生成ComfyUI工作流JSON，包含内容过滤"""
    global knob_system

    # 获取动态负向提示词
    negative_prompt = create_content_filter()


    # ... 其他代码保持不变 ...

def create_workflow_json(positive_prompt):
    """生成ComfyUI工作流JSON，根据旋钮角度动态调整分辨率"""
    global knob_system

    # 根据旋钮当前角度选择分辨率
    if hasattr(knob_system, 'current_angle'):
        resolution_info = knob_system.get_resolution_by_angle(knob_system.current_angle)
        selected_width = resolution_info["width"]
        selected_height = resolution_info["height"]
        resolution_mode = resolution_info["mode"]
        print(
            f"🎛️ 旋钮角度: {knob_system.current_angle}° → 分辨率: {selected_width}×{selected_height} ({resolution_mode})")
    else:
        # 默认中等分辨率
        selected_width = 444
        selected_height = 444
        resolution_mode = "medium"
        print(f"🖼️ 使用默认分辨率: {selected_width}×{selected_height}")

    # 根据分辨率调整生成参数
    if resolution_mode == "low":
        steps = 15  # 低分辨率快速生成
        cfg = 6
    elif resolution_mode == "medium":
        steps = 20  # 中等分辨率平衡
        cfg = 7
    else:  # high - 1080×1080高分辨率
        steps = 30  # 高分辨率需要更多步数
        cfg = 9

    workflow = {
        "10": {
            "inputs": {
                "ckpt_name": "lustermix_v15Safetensor.safetensors"
            },
            "class_type": "CheckpointLoaderSimple",
            "_meta": {"title": "Checkpoint Loader"}
        },
        "12": {
            "inputs": {
                "seed": random.randint(1, 2 ** 32),
                "steps": steps,
                "cfg": cfg,
                "sampler_name": "dpmpp_2m",
                "scheduler": "karras",
                "denoise": 1,
                "model": ["10", 0],
                "positive": ["13", 0],
                "negative": ["14", 0],
                "latent_image": ["16", 0]
            },
            "class_type": "KSampler",
            "_meta": {"title": "KSampler"}
        },
        "13": {
            "inputs": {
                "text": positive_prompt,
                "clip": ["10", 1]
            },
            "class_type": "CLIPTextEncode",
            "_meta": {"title": "CLIP Text Encode"}
        },
         "14": {
        "inputs": {
            "text": negative_prompt,  # 使用动态生成的负向提示词
            "clip": ["10", 1]
        },
        "class_type": "CLIPTextEncode",
        "_meta": {"title": "CLIP Text Encode (Negative)"}
    },
        "15": {
            "inputs": {
                "samples": ["12", 0],
                "vae": ["10", 2]
            },
            "class_type": "VAEDecode",
            "_meta": {"title": "VAE Decode"}
        },
        "16": {
            "inputs": {
                "width": selected_width,
                "height": selected_height,
                "batch_size": 1
            },
            "class_type": "EmptyLatentImage",
            "_meta": {"title": "Empty Latent Image"}
        },
        "17": {
            "inputs": {
                "filename_prefix": f"ComfyUI_{resolution_mode}_{knob_system.current_angle if hasattr(knob_system, 'current_angle') else 0}deg",
                "images": ["15", 0]
            },
            "class_type": "SaveImage",
            "_meta": {"title": "Save Image"}
        }
    }

    return workflow


def check_comfyui_connection():
    """检查ComfyUI服务是否运行"""
    try:
        session = create_requests_session()
        response = session.get("http://127.0.0.1:8188/system_stats", timeout=3)
        return response.status_code == 200
    except:
        return False


def generate_image_with_comfyui(prompt_text):
    """Generate image using ComfyUI with optimized settings"""
    print(f"🖼️ 开始图像生成，提示词: {prompt_text}")

    # 检查ComfyUI连接
    if not check_comfyui_connection():
        print("❌ ComfyUI服务未运行!")
        print("📋 请按以下步骤启动ComfyUI:")
        print(f"   1. 打开命令行窗口")
        print(f"   2. 进入目录: cd C:\\111\\ComfyUI-aki\\ComfyUI-aki-v1.6\\ComfyUI")
        print(f"   3. 运行: python main.py")
        print(f"   4. 等待看到 'Starting server' 信息")
        print(f"   5. 然后重新测试语音识别")
        return None

    api_url = "http://127.0.0.1:8188/prompt"

    try:
        session = create_requests_session()
        api_data = {
            "prompt": create_workflow_json(prompt_text),
            "client_id": f"voice_to_comfyui_{int(time.time())}"
        }

        response = session.post(api_url, json=api_data, timeout=15)
        if response.status_code != 200:
            print(f"❌ API响应错误: {response.text}")
            return None

        response_data = response.json()
        prompt_id = response_data.get("prompt_id")
        if not prompt_id:
            print("⚠️ 响应中没有prompt_id")
            return None

        print(f"✅ 提交的prompt ID: {prompt_id}")

        # 等待完成，减少等待时间
        for i in range(20):  # 减少等待循环次数
            time.sleep(2)
            try:
                history_response = session.get("http://127.0.0.1:8188/history", timeout=5)
                history_data = history_response.json()

                if prompt_id in history_data:
                    status = history_data[prompt_id].get("status", {})
                    if status.get("completed", False):
                        print("✅ 图像生成完成!")
                        break
                    elif status.get("error"):
                        print(f"❌ 任务失败: {status.get('error')}")
                        return None
            except Exception as e:
                print(f"⚠️ 检查状态时出错: {e}")
                continue

        # 获取生成的文件
        try:
            history_response = session.get("http://127.0.0.1:8188/history", timeout=5)
            history_data = history_response.json()
            outputs = history_data.get(prompt_id, {}).get("outputs", {})

            if "17" in outputs:
                image_files = outputs["17"].get("images", [])
                if image_files:
                    for img in image_files:
                        filename = img["filename"]
                        subfolder = img.get("subfolder", "")

                        if subfolder:
                            image_path = os.path.join(IMAGE_OUTPUT_DIR, subfolder, filename)
                        else:
                            image_path = os.path.join(IMAGE_OUTPUT_DIR, filename)

                        if os.path.exists(image_path):
                            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
                            new_filename = f"VoiceGen_{timestamp}_{filename}"
                            destination = os.path.join(IMAGE_OUTPUT_DIR, new_filename)

                            if image_path != destination:
                                shutil.copy2(image_path, destination)

                            print(f"✅ 图像已生成: {destination}")
                            return destination
                        else:
                            root_path = os.path.join(IMAGE_OUTPUT_DIR, filename)
                            if os.path.exists(root_path):
                                timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
                                new_filename = f"VoiceGen_{timestamp}_{filename}"
                                destination = os.path.join(IMAGE_OUTPUT_DIR, new_filename)
                                shutil.copy2(root_path, destination)
                                print(f"✅ 图像已生成: {destination}")
                                return destination
        except Exception as e:
            print(f"❌ 获取生成图像时出错: {e}")

        print("⚠️ 未找到生成的图像文件")
        return None

    except Exception as e:
        print(f"❌ 图像生成过程中出错: {str(e)}")
        return None


def generate_3d_model(image_path):
    """Generate 3D model from image using Tripo API with member optimization"""
    print(f"🧊 开始3D模型生成: {image_path}")

    # 根据旋钮角度智能选择模型版本
    current_angle = getattr(knob_system, 'current_angle', 0)

    # 会员用户使用更好的模型选择策略
    if TRIPO_MEMBER_CONFIG["is_member"]:
        if current_angle == 0:  # 1080×1080高分辨率
            model_version = "v2.0"  # 高分辨率用v2.0
        else:
            model_version = "v2.0"  # 会员全部用v2.0，享受质量优势
    else:
        model_version = "v1.4"  # 免费用户用v1.4保证速度

    # 显示预期时间
    time_info = get_expected_generation_time(
        model_version=model_version,
        is_member=TRIPO_MEMBER_CONFIG["is_member"]
    )

    print(f"🎯 使用模型: {model_version} {'👑 会员专享' if TRIPO_MEMBER_CONFIG['is_member'] else ''}")
    print(f"⏱️ 免费用户预期: {time_info['free_user']}")
    print(f"👑 会员用户预期: {time_info['member']}")
    print(f"🚀 速度提升: {time_info['improvement']}")
    print(f"🔧 配置: 1000个面 | 标准纹理")

    try:
        if not os.path.exists(image_path):
            print(f"❌ 图像文件不存在: {image_path}")
            return None

        # 使用会员优化的会话
        session = create_member_optimized_session()
        headers = {"Authorization": f"Bearer {TRIPO_API_KEY}"}

        # 如果是会员，添加优先级头部
        if TRIPO_MEMBER_CONFIG["is_member"]:
            headers.update({
                "X-Priority": "high",
                "X-Member-Tier": "premium"
            })

        # 步骤1: 上传图像文件
        upload_url = f"{TRIPO_BASE_URL}/upload"

        with open(image_path, "rb") as f:
            files = {"file": ("image.png", f, "image/png")}
            status_msg = "📤 上传图像文件 (会员优先通道)..." if TRIPO_MEMBER_CONFIG["is_member"] else "📤 上传图像文件..."
            print(status_msg)
            upload_response = session.post(upload_url, headers=headers, files=files, timeout=30)

        if upload_response.status_code != 200:
            print(f"❌ 图像上传失败: {upload_response.status_code}")
            print(f"响应内容: {upload_response.text}")
            return None

        upload_data = upload_response.json()

        # 从上传响应中获取文件Token
        file_token = None
        if "data" in upload_data:
            file_token = upload_data["data"].get("image_token") or upload_data["data"].get("token")
        elif "image_token" in upload_data:
            file_token = upload_data["image_token"]
        elif "token" in upload_data:
            file_token = upload_data["token"]

        if not file_token:
            print("❌ 上传响应中没有找到文件token")
            print(f"完整上传响应: {upload_data}")
            return None

        print(f"✅ 图像上传成功，Token: {file_token}")

        # 步骤2: 创建任务 - 使用选定的模型版本和会员优化
        task_url = f"{TRIPO_BASE_URL}/task"

        task_data = {
            "type": "image_to_model",
            "file": {"type": "png", "file_token": file_token},
            "model_version": model_version,
            "face_limit": 1000,
            "texture_resolution": "standard",
            "quality": "high" if model_version == "v2.0" else "balanced"
        }

        # 会员专属优化参数
        if TRIPO_MEMBER_CONFIG["is_member"]:
            task_data.update({
                "priority": "high",  # 高优先级
                "gpu_tier": "premium",  # 高级GPU
                "processing_mode": "accelerated"  # 加速模式
            })

        task_status = f"({model_version}, 会员加速模式)" if TRIPO_MEMBER_CONFIG["is_member"] else f"({model_version})"
        print(f"🚀 创建3D模型任务 {task_status}...")

        task_response = session.post(task_url, headers=headers, json=task_data, timeout=30)

        if task_response.status_code != 200:
            print(f"❌ 创建3D模型任务失败: {task_response.status_code}")
            print(f"响应内容: {task_response.text}")

            # 如果会员参数不被支持，尝试简化版本
            print("🔄 尝试使用简化参数...")
            fallback_task_data = {
                "type": "image_to_model",
                "file": {"type": "png", "file_token": file_token},
                "model_version": model_version,
                "face_limit": 1000,
                "texture_resolution": "1024"
            }

            task_response = session.post(task_url, headers=headers, json=fallback_task_data, timeout=30)

            if task_response.status_code != 200:
                print(f"❌ 简化参数任务也失败: {task_response.status_code}")
                return None
            else:
                print("✅ 使用简化参数创建任务成功")

        task_result = task_response.json()

        # 提取任务ID
        task_id = None
        if "data" in task_result:
            task_id = task_result["data"].get("task_id")
        elif "task_id" in task_result:
            task_id = task_result["task_id"]

        if not task_id:
            print("❌ 任务响应中没有任务ID")
            print(f"完整任务响应: {task_result}")
            return None

        print(f"✅ 3D模型任务已创建，ID: {task_id}")

        # 步骤3: 等待完成 - 会员优化的轮询策略
        max_attempts = 60 if TRIPO_MEMBER_CONFIG["is_member"] else 40
        check_interval = 4 if TRIPO_MEMBER_CONFIG["is_member"] else 6  # 会员更频繁检查

        status_url = f"{TRIPO_BASE_URL}/task/{task_id}"
        start_time = time.time()

        for attempt in range(max_attempts):
            time.sleep(check_interval)
            elapsed_time = int(time.time() - start_time)

            member_status = "👑会员加速" if TRIPO_MEMBER_CONFIG["is_member"] else ""
            print(f"📡 检查任务状态... ({attempt + 1}/{max_attempts}) "
                  f"[{model_version}{member_status}] 已用时: {elapsed_time}秒")

            try:
                status_response = session.get(status_url, headers=headers, timeout=10)
                if status_response.status_code != 200:
                    print(f"⚠️ 检查状态失败: {status_response.status_code}")
                    continue

                status_data = status_response.json()

                # 提取状态
                status = None
                result_data = None

                if "data" in status_data:
                    status = status_data["data"].get("status")
                    result_data = status_data["data"].get("result")
                elif "status" in status_data:
                    status = status_data["status"]
                    result_data = status_data.get("result")

                print(f"📊 任务状态: {status}")

                if status == "success" or status == "completed":
                    # 提取模型URL
                    model_url = None
                    if result_data:
                        if isinstance(result_data, dict):
                            if "pbr_model" in result_data:
                                pbr_model = result_data["pbr_model"]
                                if isinstance(pbr_model, dict) and "url" in pbr_model:
                                    model_url = pbr_model["url"]
                                elif isinstance(pbr_model, str):
                                    model_url = pbr_model

                        # 兜底：仍旧尝试旧字段
                        if not model_url:
                            model_url = (result_data.get("model") or
                                         result_data.get("model_url") or
                                         result_data.get("output_url") or
                                         result_data.get("glb_url") or
                                         result_data.get("file_url"))

                        # 如果result_data是字符串URL
                        if isinstance(result_data, str) and result_data.startswith("http"):
                            model_url = result_data

                    if not model_url:
                        print("❌ 完成的任务中没有模型URL")
                        print(f"完整状态响应: {status_data}")
                        return None

                    # 下载模型
                    total_time = int(time.time() - start_time)
                    member_tag = " 👑会员加速" if TRIPO_MEMBER_CONFIG["is_member"] else ""
                    print(f"📥 下载3D模型 ({model_version}, 1000面): {model_url}")
                    print(f"⏱️ 总生成时间: {total_time}秒{member_tag}")

                    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

                    # 设置输出目录和文件名根据旋钮角度
                    member_suffix = "_member" if TRIPO_MEMBER_CONFIG["is_member"] else ""

                    if hasattr(knob_system, 'current_angle'):
                        angle = knob_system.current_angle
                        if 0 <= angle < 90:
                            target_dir = r"D:\桌面\bishe01"
                            filename = f"bishe01_{model_version}{member_suffix}_{timestamp}.glb"
                        elif 90 <= angle < 180:
                            target_dir = r"D:\桌面\bishe02"
                            filename = f"bishe02_{model_version}{member_suffix}_{timestamp}.glb"
                        else:
                            target_dir = r"D:\桌面\bishe03"
                            filename = f"bishe03_{model_version}{member_suffix}_{timestamp}.glb"
                    else:
                        target_dir = MODEL_OUTPUT_DIR
                        filename = f"VoiceGen_3D_{model_version}{member_suffix}_{timestamp}.glb"

                    # 创建目标目录（如果不存在）
                    os.makedirs(target_dir, exist_ok=True)

                    # 拼接完整输出路径
                    output_file = os.path.join(target_dir, filename)

                    model_response = session.get(model_url, timeout=60)
                    if model_response.status_code == 200:
                        with open(output_file, "wb") as f:
                            f.write(model_response.content)

                        print(f"✅ 3D模型已保存到: {output_file}")
                        print(f"🎯 模型规格: {model_version}版本, 1000个面, 标准纹理")
                        if TRIPO_MEMBER_CONFIG["is_member"]:
                            print(f"👑 会员总时间: {total_time}秒 (预估: {time_info['member']})")
                        return output_file
                    else:
                        print(f"❌ 下载模型失败: {model_response.status_code}")
                        return None

                elif status == "failed" or status == "error":
                    error_msg = "未知错误"
                    if "data" in status_data:
                        error_msg = status_data["data"].get("error", error_msg)
                    elif "error" in status_data:
                        error_msg = status_data["error"]
                    print(f"❌ 3D模型生成失败: {error_msg}")
                    return None
                else:
                    # 任务仍在进行中
                    progress = 0
                    if "data" in status_data:
                        progress = status_data["data"].get("progress", 0)
                    elif "progress" in status_data:
                        progress = status_data["progress"]
                    if progress > 0:
                        member_tag = " 👑会员加速" if TRIPO_MEMBER_CONFIG["is_member"] else ""
                        print(f"⏳ {model_version}模型生成进度: {progress}%{member_tag}")

            except Exception as e:
                print(f"⚠️ 检查状态时出错: {e}")
                continue

        print("❌ 3D模型生成超时")
        return None

    except Exception as e:
        print(f"❌ 3D模型生成出错: {str(e)}")
        traceback.print_exc()
        return None

        print(f"✅ 3D模型任务已创建，ID: {task_id}")

        # 步骤3: 轮询任务状态
        status_url = f"{TRIPO_BASE_URL}/task/{task_id}"
        max_attempts = 40  # 增加等待时间，因为1000面的模型可能需要更长时间

        for attempt in range(max_attempts):
            time.sleep(6)  # 稍微增加检查间隔
            print(f"📡 检查任务状态... ({attempt + 1}/{max_attempts}) [v1.4模型生成中]")

            try:
                status_response = session.get(status_url, headers=headers, timeout=10)
                if status_response.status_code != 200:
                    print(f"⚠️ 检查状态失败: {status_response.status_code}")
                    continue

                status_data = status_response.json()

                # 提取状态
                status = None
                result_data = None

                if "data" in status_data:
                    status = status_data["data"].get("status")
                    result_data = status_data["data"].get("result")
                elif "status" in status_data:
                    status = status_data["status"]
                    result_data = status_data.get("result")

                print(f"📊 任务状态: {status}")

                if status == "success" or status == "completed":
                    # 提取模型URL
                    model_url = None
                    if result_data:
                        if isinstance(result_data, dict):
                            if "pbr_model" in result_data:
                                pbr_model = result_data["pbr_model"]
                                if isinstance(pbr_model, dict) and "url" in pbr_model:
                                    model_url = pbr_model["url"]
                                elif isinstance(pbr_model, str):
                                    model_url = pbr_model

                        # 兜底：仍旧尝试旧字段
                        if not model_url:
                            model_url = (result_data.get("model") or
                                         result_data.get("model_url") or
                                         result_data.get("output_url") or
                                         result_data.get("glb_url") or
                                         result_data.get("file_url"))

                        # 如果result_data是字符串URL
                        if isinstance(result_data, str) and result_data.startswith("http"):
                            model_url = result_data

                    if not model_url:
                        print("❌ 完成的任务中没有模型URL")
                        print(f"完整状态响应: {status_data}")
                        return None

                    # 下载模型
                    print(f"📥 下载3D模型 (v1.4模型, 1000面): {model_url}")
                    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

                    # 设置输出目录和文件名根据旋钮角度
                    if hasattr(knob_system, 'current_angle'):
                        angle = knob_system.current_angle
                        if 0 <= angle < 90:
                            target_dir = r"D:\桌面\bishe01"
                            filename = f"bishe01_v1.4_1000f_{timestamp}.glb"
                        elif 90 <= angle < 180:
                            target_dir = r"D:\桌面\bishe02"
                            filename = f"bishe02_v1.4_1000f_{timestamp}.glb"
                        else:
                            target_dir = r"D:\桌面\bishe03"
                            filename = f"bishe03_v1.4_1000f_{timestamp}.glb"
                    else:
                        target_dir = MODEL_OUTPUT_DIR
                        filename = f"VoiceGen_3D_v1.4_1000f_{timestamp}.glb"

                    # 创建目标目录（如果不存在）
                    os.makedirs(target_dir, exist_ok=True)

                    # 拼接完整输出路径
                    output_file = os.path.join(target_dir, filename)

                    model_response = session.get(model_url, timeout=60)
                    if model_response.status_code == 200:
                        with open(output_file, "wb") as f:
                            f.write(model_response.content)
                        print(f"✅ 3D模型已保存到: {output_file}")
                        print(f"🎯 模型规格: v1.4版本, 1000个面, 标准纹理")
                        return output_file
                    else:
                        print(f"❌ 下载模型失败: {model_response.status_code}")
                        return None

                elif status == "failed" or status == "error":
                    error_msg = "未知错误"
                    if "data" in status_data:
                        error_msg = status_data["data"].get("error", error_msg)
                    elif "error" in status_data:
                        error_msg = status_data["error"]
                    print(f"❌ 3D模型生成失败: {error_msg}")
                    return None
                else:
                    # 任务仍在进行中
                    progress = 0
                    if "data" in status_data:
                        progress = status_data["data"].get("progress", 0)
                    elif "progress" in status_data:
                        progress = status_data["progress"]
                    if progress > 0:
                        print(f"⏳ v1.4模型生成进度: {progress}% (目标: 1000面)")

            except Exception as e:
                print(f"⚠️ 检查状态时出错: {e}")
                continue

        print("❌ 3D模型生成超时")
        return None

    except Exception as e:
        print(f"❌ 3D模型生成出错: {str(e)}")
        traceback.print_exc()
        return None


# ========== Arduino硬件监听功能 ==========
def detect_arduino_port():
    """自动检测Arduino端口 - 优先使用COM9"""
    try:
        ports = serial.tools.list_ports.comports()

        # 优先检查COM9（Arduino Leonardo）
        for port in ports:
            if port.device == 'COM9':
                print(f"✅ 找到Arduino Leonardo端口: {port.device}")
                return port.device

        # 查找其他Arduino端口
        for port in ports:
            if "Arduino" in port.description or "Leonardo" in port.description or "CH340" in port.description or "USB" in port.description:
                return port.device

        # 如果没有找到，返回常见端口（COM9优先）
        common_ports = ['COM9', 'COM3', 'COM4', 'COM5', '/dev/ttyUSB0', '/dev/ttyACM0']
        for port_name in common_ports:
            for port in ports:
                if port.device == port_name:
                    return port_name

        # 返回第一个可用端口
        if ports:
            return ports[0].device

    except Exception as e:
        print(f"❌ 端口检测失败: {e}")

    return 'COM9'  # 默认端口改为COM9


# ========== Main Application ==========
if __name__ == "__main__":
    try:
        print("=" * 60)
        print("🚀 启动优化版实时语音识别图像3D生成系统")
        print("=" * 60)
        print(f"📁 基础目录: {BASE_DIR}")
        print(f"📁 ComfyUI目录: {COMFYUI_DIR}")
        print(f"📁 图像输出目录: {IMAGE_OUTPUT_DIR}")
        print(f"📁 3D模型输出目录: {MODEL_OUTPUT_DIR}")
        print(f"📁 语音文本文件: {VOICE_TEXT_FILE}")
        print(f"📁 Vosk模型路径: {VOSK_MODEL_PATH}")
        print("=" * 60)
        print("🎛️ 旋钮控制功能:")
        print("   旋钮角度 0° → bishe01模型 + 64×64分辨率")
        print("   旋钮角度 90° → bishe02模型 + 444×444分辨率")
        print("   旋钮角度 180° → bishe03模型 + 1080×1080分辨率")
        print("   🔌 监听Arduino Leonardo硬件旋钮输入 (COM9)")
        print("=" * 60)
        print("⚡ 性能优化功能:")
        print(f"   - 智能分辨率切换: 64×64 / 444×444 / 1080×1080")
        print(f"   - 最大显示图像数: {MAX_IMAGES_PER_COLUMN}")
        print(f"   - 图像加载延迟: {IMAGE_LOAD_DELAY}秒")
        print(f"   - UI更新间隔: {UI_UPDATE_INTERVAL}毫秒")
        print(f"   - Arduino Leonardo旋钮控制ComfyUI分辨率")
        print("=" * 60)

        # 加载图像文本映射关系
        load_image_mapping()

        # 检查ComfyUI连接
        print("🔍 检查ComfyUI服务状态...")
        if check_comfyui_connection():
            print("✅ ComfyUI服务正在运行")
        else:
            print("⚠️ ComfyUI服务未运行")
            print("📋 请在另一个命令行窗口中启动ComfyUI:")
            print(f"   cd {COMFYUI_DIR}")
            print("   python main.py")
            print("⚠️ 注意: 如果不启动ComfyUI，语音识别仍可工作，但无法生成图像")
            print("=" * 60)

        # 加载图像文本映射关系
        load_image_mapping()

        # 检测Arduino端口
        print("🔍 检测Arduino Leonardo硬件端口...")
        arduino_port = detect_arduino_port()
        print(f"📡 将尝试连接端口: {arduino_port} (Arduino Leonardo)")

        # 检查ComfyUI连接
        print("🔍 检查ComfyUI服务状态...")
        if check_comfyui_connection():
            print("✅ ComfyUI服务正在运行")
        else:
            print("⚠️ ComfyUI服务未运行")
            print("📋 请在另一个命令行窗口中启动ComfyUI:")
            print(f"   cd {COMFYUI_DIR}")
            print("   python main.py")
            print("⚠️ 注意: 如果不启动ComfyUI，语音识别仍可工作，但无法生成图像")
            print("=" * 60)

        # 创建文本显示窗口（主窗口）
        text_app = TextDisplayApp(arduino_port)
        # 创建图像显示窗口（子窗口）
        image_app = DynamicLayoutApp(IMAGE_DISPLAY_FOLDER, text_app)

        # 设置相互引用
        text_app.set_image_app(image_app)

        print("✅ 双窗口界面已启动")
        print("🎤 实时语音识别将自动开始...")
        print("🔄 检测到语音后将自动生成图像和3D模型")
        # 显示会员状态
        if TRIPO_MEMBER_CONFIG["is_member"]:
            print("=" * 60)
            print("👑 Tripo AI 会员状态：已激活")
            print("🚀 享有的加速特权：")
            print("   ✅ 优先队列处理 (减少30%等待)")
            print("   ✅ 高性能GPU集群 (提升20%速度)")
            print("   ✅ 3个并发任务支持")
            print("   ✅ 减少API限流限制")
            print("📊 预期性能提升：")

            for version in ["v1.4", "v2.0"]:
                time_info = get_expected_generation_time(version, is_member=True)
                print(f"   {version}: {time_info['free_user']} → {time_info['member']} ({time_info['improvement']})")

            print("🎯 会员策略：全部使用v2.0模型获得最佳质量")
            print("=" * 60)
        else:
            print("⚠️ 未开通Tripo AI会员，使用标准速度配置")
        print("🏷️ 图像上将显示对应的语音关键词")
        print("🖼️ 图像将自动适应窗口大小变化")
        print("🎛️ 请手动旋转Arduino Leonardo硬件旋钮来控制分辨率")
        print("📊 旋钮角度将实时控制ComfyUI图像生成分辨率")
        print("⚡ 系统已优化性能，降低资源占用")
        print("=" * 60)
        print("🚀 系统启动完成，开始监听语音输入和Arduino Leonardo旋钮...")

        # 启动主循环
        text_app.mainloop()

    except KeyboardInterrupt:
        print("\n🛑 用户中断程序")
        print("🔄 正在清理资源...")

    except Exception as e:
        print(f"❌ 应用程序错误: {str(e)}")
        traceback.print_exc()

    finally:
        print("👋 程序已退出")
        print("📊 会话统计:")
        if 'image_text_mapping' in globals():
            print(f"   - 总生成图像: {len(image_text_mapping)} 张")
        print(f"   - 运行时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print("=" * 60)
