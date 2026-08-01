我原本以為透過python 可以直接拆解剪映的草稿檔的檔案結構，進而直接生成剪映草稿檔。 但是，我發現剪映在5.9版之後，就加密了剪映草稿檔，所以無法直接解密草稿檔。 但是，我又發現原來python，有以下這個函式庫 可安裝及調用。（轻量、灵活、易上手的Python剪映草稿生成及导出工具，构建全自动化视频剪辑/混剪流水线。本项目的CapCut版本正于 https://github.com/GuanYixuan/pyJianYingDraft） 在不更動原本已經完成的功能，及生成的檔案格式，及檔名的情況下。 在注入剪映草稿的方式，請改採用此方式（pyJianYingDraft），生成剪映草稿檔。 我希望未來也能將python 程式，分發給其它用戶使用。 是否能代入偵測python主程式，是否有安裝的用戶提醒？（請用戶自行安裝） 另外，需要同時安裝的python 函式庫，也一併自動幫用戶下載安裝（只要能自動安裝，不出發防毒軟體的，都自動安裝）。 其它所需要運作的軟體，不能自動安裝的，也能一併提醒用戶自行安裝。 還有，提醒用戶填寫VideoCaptioner的資料夾位址。

# -*- coding: utf-8 -*-
"""
text_match_injector.py
======================
自動化影片流水線：
  [1] 調用 VideoCaptioner runtime 生成對齊字幕 (subtitle.srt)
  [2] 將字幕與 slide 圖片附註做文字匹配，建立圖片時間軸
  [3] 使用 pyJianYingDraft 產生剪映 (JianyingPro) 草稿檔
 
使用方式：
    python text_match_injector.py
 
環境需求（程式會自動提示/安裝）：
    - Python 3.8 ~ 3.11 （建議）
    - 套件：pyJianYingDraft （自動安裝）
    - 剪映專業版 (JianyingPro) for Windows （請用戶自行安裝）
    - VideoCaptioner （請用戶自行安裝，並於首次執行時告訴程式位置）
 
最後更新：2026-06-09
"""
 
import base64
import hashlib
import importlib
import importlib.util
import json
import os
import re
import shutil
import subprocess
import sys
import tempfile
import time
import uuid
from datetime import datetime
from difflib import SequenceMatcher
 
# ============================================================
# 0. 基本常數
# ============================================================
CURRENT_SCRIPT_DIR = (
    os.path.dirname(os.path.abspath(__file__))
    if "__file__" in locals() and __file__
    else os.getcwd()
)
 
LOGGER = None
 
TEMPLATE_DRAFT_NAME = "敦煌大佛-手動匹配"
TRANSITION_MATERIAL_ID = "6914112332205396488"
EFFECT_GOLD_MATERIAL_ID = "7395457886899424547"
EFFECT_TYNDALL_MATERIAL_ID = "7399492418606828815"
TEMPLATE_SIGNATURES = {}
ENCODED_FIELD_MAP = {}
 
# 自動 pip 安裝的清單（不會觸發防毒的標準 pip 安裝）
REQUIRED_PACKAGES = [
    ("pyJianYingDraft", "pyJianYingDraft"),
]
 
MATERIAL_ID_HINTS = {
    "transition_overlap": TRANSITION_MATERIAL_ID,
    "effect_gold_dust": EFFECT_GOLD_MATERIAL_ID,
    "effect_tyndall_light": EFFECT_TYNDALL_MATERIAL_ID,
}
 
 
# ============================================================
# 1. 環境檢查：Python 版本 + 自動安裝依賴
# ============================================================
def _print_welcome_banner():
    print("=" * 64)
    print("         Liao Factory - 影片自動化流水線 v5.0")
    print(f" [TIME] {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f" [CWD ] {CURRENT_SCRIPT_DIR}")
    print("=" * 64)
 
 
def check_python_version():
    """檢查 Python 版本，若過舊則提醒用戶升級。"""
    ver = sys.version_info
    print(f" [ENV] Python 版本：{ver.major}.{ver.minor}.{ver.micro}")
    if ver.major < 3 or (ver.major == 3 and ver.minor < 8):
        print(" [WARN] Python 版本過舊，建議升級到 Python 3.8 或更高版本。")
        print("        下載：https://www.python.org/downloads/")
        return False
    if ver.major == 3 and ver.minor >= 13:
        print(" [INFO] Python 3.13+ 可能與部分 UI 自動化函式庫不相容。")
        print("        若僅需產生草稿檔（不需自動匯出）則不受影響。")
    return True
 
 
def _pip_install(package_name, upgrade=False):
    """用當前直譯器執行 pip install。"""
    cmd = [sys.executable, "-m", "pip", "install"]
    if upgrade:
        cmd.append("--upgrade")
    cmd.append(package_name)
    print(f" [INSTALL] 正在安裝/升級：{package_name}")
    try:
        subprocess.check_call(cmd, stdout=sys.stdout, stderr=sys.stderr)
        return True
    except subprocess.CalledProcessError as exc:
        print(f" [ERROR] pip 安裝失敗 ({package_name}): {exc}")
        return False
 
 
def ensure_dependencies():
    """
    確保所需的 Python 套件已安裝。
    若未安裝則自動透過 pip 安裝（標準 pip 路徑，不會觸發防毒）。
    """
    print("\n---------------- 1. 檢查 Python 套件依�ite賴 ----------------")
    all_ok = True
    for import_name, pip_name in REQUIRED_PACKAGES:
        spec = importlib.util.find_spec(import_name)
        if spec is None:
            print(f" [MISSING] {import_name} 未安裝，將自動安裝……")
            if not _pip_install(pip_name):
                all_ok = False
        else:
            print(f" [OK] {import_name} 已安裝")
    return all_ok
 
 
# ============================================================
# 2. 環境檢查：剪映 / VideoCaptioner（需用戶手動安裝）
# ============================================================
def guess_jianying_draft_root():
    """嘗試猜測剪映草稿根目錄。"""
    candidates = []
    local_appdata = os.environ.get("LOCALAPPDATA") or os.path.expanduser(
        "~/AppData/Local"
    )
    # 5.x/6.x 內建預設位置
    candidates.append(os.path.join(local_appdata, "JianyingPro", "User Data", "Projects", "com.lveditor.draft"))
    # 手動設定的外部路徑，常見是「JianyingPro Drafts」
    # 嘗試掃描常用磁碟
    for drive in ("D:", "E:", "F:"):
        candidates.append(os.path.join(drive, "JianyingPro Drafts"))
        candidates.append(os.path.join(drive, "剪映草稿"))
    for path in candidates:
        try:
            if os.path.isdir(path):
                return path
        except Exception:
            pass
    return ""
 
 
def check_jianying_installed():
    """檢查剪映是否「可能」安裝，並嘗試定位草稿目錄。"""
    print("\n---------------- 2. 檢查剪映 (JianyingPro) ----------------")
    # 檢查常見安裝路徑
    install_candidates = [
        os.path.join(os.environ.get("ProgramFiles", "C:/Program Files"), "JianyingPro", "JianyingPro.exe"),
        os.path.join(os.environ.get("ProgramFiles(x86)", "C:/Program Files (x86)"), "JianyingPro", "JianyingPro.exe"),
        os.path.join(os.environ.get("LOCALAPPDATA", ""), "JianyingPro", "JianyingPro.exe"),
    ]
    installed = any(os.path.isfile(p) for p in install_candidates)
 
    draft_root = guess_jianying_draft_root()
 
    if installed and draft_root:
        print(f" [OK] 剪映似乎已安裝，草稿根目錄：{draft_root}")
        return True, draft_root
 
    print(" [提示] 未偵測到剪映安裝路徑。")
    print("        若尚未安裝，請至以下網址下載剪映專業版 (Windows)：")
    print("        https://www.capcut.cn/  或  https://lv.ulikecam.com/")
    print("        安裝後，請先執行一次剪映，讓它建立預設的草稿目錄。")
    if draft_root:
        print(f"        （已發現一個可用草稿目錄：{draft_root}）")
        return False, draft_root
    print("        若您已自訂草稿目錄，請在剪映中「全域設定→草稿位置」查詢，")
    print("        並稍後於提示時手動輸入。")
    return False, ""
 
 
def _prompt_user_for_vc_location():
    """互動式詢問 VideoCaptioner 安裝目錄，並儲存到 vc_location.txt。"""
    print("\n [提示] 請告訴我 VideoCaptioner 的安裝目錄。")
    print("        通常是類似 C:\\Program Files\\VideoCaptioner 或")
    print("        D:\\VideoCaptioner 的路徑（目錄中要有 VideoCaptioner.exe）。")
    print("        若尚未安裝，請至這裡下載：")
    print("        https://github.com/weifengdq/VideoCaptioner/releases")
    print("")
    for _attempt in range(3):
        sys.stdout.write("  請輸入 VideoCaptioner 目錄路徑 (直接 Enter 跳過): ")
        sys.stdout.flush()
        try:
            line = sys.stdin.readline().strip().strip('"').strip("'")
        except Exception:
            line = ""
        if not line:
            print(" [INFO] 跳過 VideoCaptioner 路徑設定（僅生成字幕時需要）。")
            return ""
        if not os.path.isdir(line):
            print(f" [WARN] 目錄不存在：{line}，請再試一次。")
            continue
        # 嘗試在該目錄找到 VideoCaptioner.exe
        expected_exe = os.path.join(line, "VideoCaptioner.exe")
        if not os.path.isfile(expected_exe):
            # 也可能放在子目錄
            alt = os.path.join(line, "VideoCaptioner", "VideoCaptioner.exe")
            if os.path.isfile(alt):
                line = os.path.dirname(alt)
            else:
                print(f" [WARN] 在該目錄找不到 VideoCaptioner.exe，仍會儲存您的路徑。")
        try:
            with open(os.path.join(CURRENT_SCRIPT_DIR, "vc_location.txt"), "w", encoding="utf-8") as fh:
                fh.write(line)
            print(f" [OK] 已儲存 VideoCaptioner 路徑到 vc_location.txt：{line}")
            return line
        except Exception as exc:
            print(f" [ERROR] 無法寫入 vc_location.txt：{exc}")
            return line
    return ""
 
 
def check_videocaptioner():
    """
    確認 VideoCaptioner 的位置。
    優先讀取 CURRENT_SCRIPT_DIR/vc_location.txt；
    若無效或不存在，則提示用戶輸入。
    """
    print("\n---------------- 3. 檢查 VideoCaptioner ----------------")
    config_path = os.path.join(CURRENT_SCRIPT_DIR, "vc_location.txt")
    if os.path.isfile(config_path):
        try:
            with open(config_path, "r", encoding="utf-8") as fh:
                saved_path = fh.read().strip().strip('"').strip("'")
            if saved_path and os.path.isdir(saved_path):
                print(f" [OK] 從 vc_location.txt 讀取 VideoCaptioner 路徑：{saved_path}")
                return saved_path
            print(f" [WARN] vc_location.txt 內容無效：{saved_path}")
        except Exception as exc:
            print(f" [WARN] 無法讀取 vc_location.txt：{exc}")
 
    # 嘗試常見安裝位置
    for guess in (
        os.path.join(os.environ.get("ProgramFiles", "C:/Program Files"), "VideoCaptioner"),
        os.path.join(os.environ.get("ProgramFiles(x86)", "C:/Program Files (x86)"), "VideoCaptioner"),
        os.path.join(os.environ.get("LOCALAPPDATA", ""), "VideoCaptioner"),
        os.path.join(CURRENT_SCRIPT_DIR, "VideoCaptioner"),
    ):
        if os.path.isfile(os.path.join(guess, "VideoCaptioner.exe")):
            print(f" [OK] 在以下位置發現 VideoCaptioner：{guess}")
            try:
                with open(config_path, "w", encoding="utf-8") as fh:
                    fh.write(guess)
            except Exception:
                pass
            return guess
 
    # 找不到，提示用戶
    return _prompt_user_for_vc_location()
 
 
def run_environment_checks():
    """統一的環境檢查流程。回傳 (draft_root, vc_root)。"""
    _print_welcome_banner()
    ok_py = check_python_version()
    ok_dep = ensure_dependencies()
    ok_jy, draft_root = check_jianying_installed()
    vc_root = check_videocaptioner()
 
    if not ok_dep:
        print("\n [ERROR] 必要的 Python 套件安裝失敗，請手動執行：")
        print("         pip install pyJianYingDraft")
        sys.exit(2)
 
    print("\n---------------- 環境檢查結束 ----------------")
    return draft_root, vc_root
 
 
# ============================================================
# 3. 原有工具函式（保留，檔名/格式不變）
# ============================================================
class FactoryLogger(object):
    def __init__(self, basedir, log_dir_name="factory_logs"):
        log_dir = os.path.join(basedir, log_dir_name)
        os.makedirs(log_dir, exist_ok=True)
        timestamp = datetime.now().strftime("%Y%m%d%H%M%S")
        logfilename = f"draft_injection_{timestamp}.log"
        self.log_path = os.path.join(log_dir, logfilename)
        self.terminal = sys.stdout
        self.log = open(self.log_path, "w", encoding="utf-8")
 
    def write(self, message):
        if message is None:
            return
        try:
            self.terminal.write(message)
        except Exception:
            pass
        try:
            self.log.write(message)
            self.log.flush()
        except Exception:
            pass
 
    def flush(self):
        try:
            self.terminal.flush()
        except Exception:
            pass
        try:
            self.log.flush()
        except Exception:
            pass
 
 
def init_logging(base_dir):
    global LOGGER
    if sys.platform == "win32":
        for stream_name in ("stdout", "stdin", "stderr"):
            stream = getattr(sys, stream_name, None)
            if stream and hasattr(stream, "reconfigure"):
                try:
                    stream.reconfigure(encoding="utf-8")
                except Exception:
                    pass
    LOGGER = FactoryLogger(base_dir)
    sys.stdout = LOGGER
    sys.stderr = LOGGER
 
 
def find_vc_executable(current_dir):
    """
    優先從 CURRENT_SCRIPT_DIR/vc_location.txt 讀取；
    其次嘗試常見路徑。
    """
    config_filename = "vc_location.txt"
    # 優先使用環境檢查時儲存的位置（已在 run_environment_checks 時寫入）
    config_path = os.path.join(CURRENT_SCRIPT_DIR, config_filename)
    if os.path.exists(config_path):
        try:
            with open(config_path, "r", encoding="utf-8") as handle:
                external_path = handle.read().strip().strip('"').strip("'")
            if external_path and os.path.isdir(external_path):
                # 嘗試找到 VideoCaptioner.exe
                exe = os.path.join(external_path, "VideoCaptioner.exe")
                if os.path.isfile(exe):
                    print(f" [CONFIG] 載入 VideoCaptioner 位置：{exe}")
                    return exe
        except Exception as exc:
            print(f" [WARN] 讀取 vc_location.txt 失敗：{exc}")
 
    # 接著嘗試 current_dir 內的本機副本
    for name in ("VideoCaptioner.exe", "vc.exe"):
        local_path = os.path.join(current_dir, name)
        if os.path.isfile(local_path):
            return local_path
    sub = os.path.join(current_dir, "VideoCaptioner", "VideoCaptioner.exe")
    if os.path.isfile(sub):
        return sub
    return ""
 
 
def find_vc_runtime_python(vc_executable_path):
    if not vc_executable_path:
        return ""
    vc_root = os.path.dirname(vc_executable_path)
    runtime_python = os.path.join(vc_root, "runtime", "python.exe")
    if os.path.isfile(runtime_python):
        return runtime_python
    return ""
 
 
def read_text_file(file_path):
    if not os.path.exists(file_path):
        return ""
    with open(file_path, "r", encoding="utf-8", errors="ignore") as handle:
        return handle.read()
 
 
def read_json_file(file_path, default=None):
    if not os.path.exists(file_path):
        return {} if default is None else default
    with open(file_path, "r", encoding="utf-8", errors="ignore") as handle:
        return json.load(handle)
 
 
def write_json_file(file_path, data):
    with open(file_path, "w", encoding="utf-8") as handle:
        json.dump(data, handle, ensure_ascii=False, separators=(",", ":"))
 
 
def compute_sha256_bytes(raw_bytes):
    return hashlib.sha256(raw_bytes).hexdigest()
 
 
def count_files_in_dir(folder_path):
    count = 0
    for _, _, file_names in os.walk(folder_path):
        count += len(file_names)
    return count
 
 
def build_output_draft_name(model_folder, timestamp=None):
    ts = timestamp or datetime.now().strftime("%Y%m%d%H%M")
    return f"Draft_{model_folder}_{ts}"
 
 
# ============================================================
# 4. 文字匹配與 SRT 解析（保留原演算法）
# ============================================================
def clean_text(text):
    return re.sub(r"[^\w\u4e00-\u9fff]+", "", text or "").lower()
 
 
def detect_language_code(script_text):
    if not script_text:
        return ""
    cjk_count = len(re.findall(r"[\u4e00-\u9fff]", script_text))
    latin_count = len(re.findall(r"[A-Za-z]", script_text))
    if cjk_count >= latin_count:
        return "zh"
    if latin_count > 0:
        return "en"
    return ""
 
 
def srt_time_to_us(time_str):
    time_str = time_str.replace(",", ":")
    parts = time_str.split(":")
    return (
        ((int(parts[0]) * 3600) + (int(parts[1]) * 60) + int(parts[2])) * 1000 + int(parts[3])
    ) * 1000
 
 
def build_even_image_timeline(image_list, total_duration):
    if not image_list:
        return []
    if total_duration <= 0:
        total_duration = len(image_list) * 5 * 1000 * 1000
 
    base_duration = max(total_duration // len(image_list), 200000)
    image_timeline = []
    current_time = 0
    for index, image_name in enumerate(image_list):
        duration = base_duration
        if index == len(image_list) - 1:
            duration = max(total_duration - current_time, 200000)
        image_timeline.append(
            {"image": image_name, "start": current_time, "duration": duration}
        )
        current_time += duration
    return image_timeline
 
 
def text_match_score(clean_srt_text, clean_asset_text):
    if not clean_srt_text or not clean_asset_text:
        return 0.0
    if clean_srt_text in clean_asset_text or clean_asset_text in clean_srt_text:
        return 1.0
 
    shorter, longer = sorted((clean_srt_text, clean_asset_text), key=len)
    ratio = SequenceMatcher(None, shorter, longer).ratio()
 
    window = max(6, min(18, len(shorter) // 2))
    chunk_hits = 0
    chunk_total = 0
    step = max(1, window // 2)
    for index in range(0, max(len(shorter) - window + 1, 1), step):
        chunk = shorter[index: index + window]
        if len(chunk) < 4:
            continue
        chunk_total += 1
        if chunk in longer:
            chunk_hits += 1
 
    coverage = (chunk_hits / chunk_total) if chunk_total else 0.0
    return max(ratio, coverage)
 
 
def parse_srt_by_text_matching(asset_folder, srt_file, image_list):
    """
    解析 subtitle.srt，並與 asset_folder 中各圖片對應的 .txt 附註做匹配，
    回傳 (srt_segments, image_timeline, total_duration)。
    """
    image_library = {}
    for file_name in image_list:
        txt_path = os.path.join(asset_folder, os.path.splitext(file_name)[0] + ".txt")
        image_library[file_name] = clean_text(read_text_file(txt_path))
 
    with open(srt_file, "r", encoding="utf-8", errors="ignore") as handle:
        srt_raw = handle.read().strip()
 
    blocks = re.split(r"\r?\n\r?\n+", srt_raw) if srt_raw else []
 
    srt_segments = []
    matched_timestamps = []
    total_duration = 0
 
    for block in blocks:
        lines = block.splitlines()
        if len(lines) < 3:
            continue
 
        time_line = lines[1].strip()
        time_match = re.search(
            r"(\d{2}:\d{2}:\d{2},\d{3})\s+-->\s+(\d{2}:\d{2}:\d{2},\d{3})", time_line
        )
        if not time_match:
            continue
 
        start_us = srt_time_to_us(time_match.group(1))
        end_us = srt_time_to_us(time_match.group(2))
        total_duration = max(total_duration, end_us)
 
        srt_text = "".join(lines[2:]).strip()
        clean_srt_text = clean_text(srt_text)
        srt_segments.append(
            {"start": start_us, "duration": max(end_us - start_us, 1), "text": srt_text}
        )
 
        best_image = None
        best_score = 0.0
        for image_name, image_text in image_library.items():
            score = text_match_score(clean_srt_text, image_text)
            if score > best_score:
                best_score = score
                best_image = image_name
 
        min_threshold = 0.60 if len(clean_srt_text) < 10 else 0.35
        if best_image and best_score >= min_threshold:
            matched_timestamps.append(
                {
                    "image": best_image,
                    "start": start_us,
                    "end": end_us,
                    "score": round(best_score, 3),
                }
            )
            print(
                f" [MATCHED] 字幕對應到素材：{best_image} (score={best_score:.3f}) -> '{srt_text}'"
            )
 
    if not matched_timestamps:
        print(" [WARN] 字幕與 slide 附註之間未找到可靠文字匹配，使用均分圖片時間軸。")
        return srt_segments, build_even_image_timeline(image_list, total_duration), total_duration
 
    image_timeline = []
    timeline_map = {img: {"start": None, "end": None} for img in image_list}
 
    for match in matched_timestamps:
        image_name = match["image"]
        if timeline_map[image_name]["start"] is None:
            timeline_map[image_name]["start"] = match["start"]
        timeline_map[image_name]["end"] = match["end"]
 
    current_time = 0
    for index, image_name in enumerate(image_list):
        if index == 0 and timeline_map[image_name]["start"] is None:
            timeline_map[image_name]["start"] = 0
 
        next_valid_start = total_duration
        for next_image in image_list[index + 1:]:
            if timeline_map[next_image]["start"] is None:
                continue
            next_valid_start = timeline_map[next_image]["start"]
            break
 
        if timeline_map[image_name]["start"] is None:
            timeline_map[image_name]["start"] = current_time
        if (
            timeline_map[image_name]["end"] is None
            or timeline_map[image_name]["end"] < next_valid_start
        ):
            timeline_map[image_name]["end"] = next_valid_start
        if timeline_map[image_name]["end"] <= timeline_map[image_name]["start"]:
            timeline_map[image_name]["end"] = timeline_map[image_name]["start"] + 200000
 
        duration = timeline_map[image_name]["end"] - timeline_map[image_name]["start"]
        image_timeline.append(
            {
                "image": image_name,
                "start": timeline_map[image_name]["start"],
                "duration": duration,
            }
        )
        current_time = timeline_map[image_name]["end"]
 
    return srt_segments, image_timeline, total_duration
 
 
# ============================================================
# 5. VideoCaptioner runtime 字幕生成（保留原邏輯）
# ============================================================
def build_vc_runtime_helper_script(vc_root, media_path, script_path, output_srt, language_code):
    vc_root_json = json.dumps(vc_root, ensure_ascii=False)
    media_path_json = json.dumps(media_path, ensure_ascii=False)
    script_path_json = json.dumps(script_path, ensure_ascii=False)
    output_srt_json = json.dumps(output_srt, ensure_ascii=False)
    language_code_json = json.dumps(language_code, ensure_ascii=False)
    return (
        "# -*- coding: utf-8 -*-\n"
        "import json, os, sys, tempfile, traceback\n"
        "from pathlib import Path\n"
        f"VC_ROOT = Path({vc_root_json})\n"
        f"MEDIA_PATH = Path({media_path_json})\n"
        f"SCRIPT_PATH = Path({script_path_json})\n"
        f"OUTPUT_SRT = Path({output_srt_json})\n"
        f"LANGUAGE_CODE = {language_code_json}\n"
        "sys.path.insert(0, str(VC_ROOT))\n"
        "import app.config\n"
        "from app.core.bk_asr.transcribe import transcribe\n"
        "from app.core.entities import TranscribeConfig, TranscribeModelEnum\n"
        "from app.core.utils.video_utils import video2audio\n"
        "\n"
        "def load_settings():\n"
        "    settings_path = VC_ROOT / 'AppData' / 'settings.json'\n"
        "    if not settings_path.exists():\n"
        "        return {}\n"
        "    return json.loads(settings_path.read_text(encoding='utf-8'))\n"
        "\n"
        "def enum_by_value(enum_cls, raw_value, default):\n"
        "    for item in enum_cls:\n"
        "        if item.value == raw_value:\n"
        "            return item\n"
        "    return default\n"
        "\n"
        "def build_configs(settings, prompt_text):\n"
        "    transcribe_settings = settings.get('Transcribe', {})\n"
        "    whisper_settings = settings.get('Whisper', {})\n"
        "    whisper_api_settings = settings.get('WhisperAPI', {})\n"
        "    faster_settings = settings.get('FasterWhisper', {})\n"
        "    configured_model = enum_by_value(TranscribeModelEnum,\n"
        "        transcribe_settings.get('TranscribeModel'), TranscribeModelEnum.BIJIAN)\n"
        "    ordered_models = [configured_model]\n"
        "    for candidate in (TranscribeModelEnum.BIJIAN, TranscribeModelEnum.JIANYING):\n"
        "        if candidate not in ordered_models:\n"
        "            ordered_models.append(candidate)\n"
        "    common_kwargs = {\n"
        "        'transcribe_language': LANGUAGE_CODE or '',\n"
        "        'use_asr_cache': False,\n"
        "        'need_word_time_stamp': False,\n"
        "        'whisper_model': whisper_settings.get('WhisperModel', 'tiny'),\n"
        "        'whisper_api_key': whisper_api_settings.get('WhisperApiKey', ''),\n"
        "        'whisper_api_base': whisper_api_settings.get('WhisperApiBase', ''),\n"
        "        'whisper_api_model': whisper_api_settings.get('WhisperApiModel', ''),\n"
        "        'whisper_api_prompt': prompt_text,\n"
        "        'faster_whisper_program': faster_settings.get('Program', 'faster-whisper-xxl.exe'),\n"
        "        'faster_whisper_model': faster_settings.get('Model', 'tiny'),\n"
        "        'faster_whisper_model_dir': str(VC_ROOT / 'AppData' / 'models'),\n"
        "        'faster_whisper_device': faster_settings.get('Device', 'cuda'),\n"
        "        'faster_whisper_vad_filter': bool(faster_settings.get('VadFilter', True)),\n"
        "        'faster_whisper_vad_threshold': float(faster_settings.get('VadThreshold', 0.4)),\n"
        "        'faster_whisper_vad_method': faster_settings.get('VadMethod', 'silero_v4'),\n"
        "        'faster_whisper_ff_mdx_kim2': bool(faster_settings.get('FfMdxKim2', False)),\n"
        "        'faster_whisper_one_word': bool(faster_settings.get('OneWord', True)),\n"
        "        'faster_whisper_prompt': prompt_text,\n"
        "    }\n"
        "    return [TranscribeConfig(transcribe_model=m, **common_kwargs) for m in ordered_models]\n"
        "\n"
        "temp_wav_path = None\n"
        "try:\n"
        "    script_text = SCRIPT_PATH.read_text(encoding='utf-8', errors='ignore')\n"
        "    prompt_text = script_text.strip()[:1200]\n"
        "    print(f'[VC] runtime root: {VC_ROOT}', flush=True)\n"
        "    print(f'[VC] source media: {MEDIA_PATH}', flush=True)\n"
        "    print(f'[VC] output srt: {OUTPUT_SRT}', flush=True)\n"
        "    fd, temp_wav_path = tempfile.mkstemp(prefix='vc_align_', suffix='.wav')\n"
        "    os.close(fd)\n"
        "    print('[VC] converting media to wav...', flush=True)\n"
        "    if not video2audio(str(MEDIA_PATH), output=temp_wav_path):\n"
        "        raise RuntimeError('VideoCaptioner video2audio returned False')\n"
        "    settings = load_settings()\n"
        "    last_error = None\n"
        "    for config in build_configs(settings, prompt_text):\n"
        "        try:\n"
        "            print('[VC] trying ASR model:', config.transcribe_model.value, 'language=', config.transcribe_language or 'auto', flush=True)\n"
        "            asr_data = transcribe(temp_wav_path, config)\n"
        "            OUTPUT_SRT.parent.mkdir(parents=True, exist_ok=True)\n"
        "            asr_data.to_srt(save_path=str(OUTPUT_SRT))\n"
        "            print(f'[VC] subtitle saved: {OUTPUT_SRT}', flush=True)\n"
        "            raise SystemExit(0)\n"
        "        except Exception as exc:\n"
        "            last_error = exc\n"
        "            print(f'[VC][WARN] model failed: {config.transcribe_model.value} -> {exc}', flush=True)\n"
        "    print(f'[VC][ERROR] all ASR attempts failed: {last_error}', flush=True)\n"
        "    if last_error:\n"
        "        traceback.print_exception(type(last_error), last_error, last_error.__traceback__)\n"
        "    raise SystemExit(1)\n"
        "finally:\n"
        "    if temp_wav_path and os.path.exists(temp_wav_path):\n"
        "        try: os.unlink(temp_wav_path)\n"
        "        except Exception: pass\n"
    )
 
 
def generate_srt_with_videocaptioner_runtime(vc_executable_path, media_path, script_path, output_srt):
    runtime_python = find_vc_runtime_python(vc_executable_path)
    if not runtime_python:
        raise FileNotFoundError(
            "找不到 VideoCaptioner 的 runtime/python.exe。"
            "請重新安裝 VideoCaptioner 或更新 vc_location.txt。"
        )
 
    vc_root = os.path.dirname(vc_executable_path)
    script_text = read_text_file(script_path)
    language_code = detect_language_code(script_text)
 
    helper_fd, helper_script_path = tempfile.mkstemp(
        prefix="vc_runtime_bridge_", suffix=".py", text=True
    )
    os.close(helper_fd)
 
    with open(helper_script_path, "w", encoding="utf-8") as handle:
        handle.write(
            build_vc_runtime_helper_script(
                vc_root=vc_root,
                media_path=media_path,
                script_path=script_path,
                output_srt=output_srt,
                language_code=language_code,
            )
        )
 
    env = os.environ.copy()
    env["PYTHONIOENCODING"] = "utf-8"
    env["PYTHONDONTWRITEBYTECODE"] = "1"
    creationflags = subprocess.CREATE_NO_WINDOW if os.name == "nt" else 0
    cmd = [runtime_python, helper_script_path]
 
    print(f" [INFO] VC runtime bridge -> {runtime_python}")
    print(f" [INFO] 推測的字幕語言 -> {language_code or 'auto'}")
 
    try:
        process = subprocess.Popen(
            cmd,
            cwd=vc_root,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            encoding="utf-8",
            errors="replace",
            env=env,
            creationflags=creationflags,
        )
 
        start_time = time.time()
        max_wait_seconds = 60 * 30
 
        while True:
            output_line = process.stdout.readline() if process.stdout else ""
            if output_line:
                sys.stdout.write(output_line)
                sys.stdout.flush()
            elif process.poll() is not None:
                break
 
            if time.time() - start_time > max_wait_seconds:
                process.kill()
                raise TimeoutError("VideoCaptioner runtime 執行逾時 (30 分鐘)。")
 
        return_code = process.wait()
        if return_code != 0:
            raise RuntimeError(
                f"VideoCaptioner runtime 結束代碼異常 ({return_code})。"
            )
 
    finally:
        try:
            os.remove(helper_script_path)
        except OSError:
            pass
 
 
# ============================================================
# 6. 草稿產生：改用 pyJianYingDraft
# ============================================================
def inject_timeline_into_jianying_template(
    asset_folder,
    sub_model_folder,
    image_list,
    audio_file,
    srt_segments,
    image_timeline,
    total_duration,
    draft_root="",
):
    """
    使用 pyJianYingDraft 產生剪映草稿。
 
    策略：
      1) 若 draft_root 目錄中存在名為 TEMPLATE_DRAFT_NAME 的未加密模板草稿，
         則 duplicate_as_template 保留模板的轉場/特效/濾鏡；
      2) 否則從頭建立一個新草稿（1920x1080），加入圖片軌、音軌、字幕軌。
    """
    # 嘗試匯入 pyJianYingDraft
    try:
        import pyJianYingDraft as draft_pkg
        from pyJianYingDraft import trange
    except ImportError as exc:
        raise RuntimeError(
            f"pyJianYingDraft 尚未安裝，請先執行：pip install pyJianYingDraft  ({exc})"
        )
 
    if not draft_root:
        draft_root = guess_jianying_draft_root()
    if not draft_root or not os.path.isdir(draft_root):
        raise RuntimeError(
            "找不到剪映草稿目錄，請先安裝/開啟一次剪映，或手動指定草稿根目錄。"
        )
 
    timestamp = datetime.now().strftime("%Y%m%d%H%M")
    project_name = build_output_draft_name(sub_model_folder, timestamp)
 
    print("\n=================== 草稿注入 (pyJianYingDraft) ===================")
    print(f" [INFO] 草稿根目錄：{draft_root}")
    print(f" [INFO] 新草稿名稱：{project_name}")
    print(f" [INFO] 圖片數量={len(image_list)}, 字幕片段={len(srt_segments)}, 總時長={total_duration/1_000_000:.2f}s")
 
    # 使用 DraftFolder 管理
    draft_folder = draft_pkg.DraftFolder(draft_root)
 
    # ---- 策略 A：嘗試使用模板（若存在且可載入） ----
    script = None
    use_template = False
    try:
        # 若模板草稿存在，先以它為基底
        template_candidates = [
            os.path.join(draft_root, TEMPLATE_DRAFT_NAME),
        ]
        for tc in template_candidates:
            if os.path.isdir(tc):
                print(f" [INFO] 發現模板草稿：{TEMPLATE_DRAFT_NAME}，嘗試以模板方式載入……")
                try:
                    script = draft_folder.duplicate_as_template(TEMPLATE_DRAFT_NAME, project_name)
                    use_template = True
                    print(f" [OK] 已以模板 '{TEMPLATE_DRAFT_NAME}' 為基底建立草稿。")
                    break
                except Exception as exc:
                    print(f" [WARN] 模板載入失敗（可能為 6+ 加密版本）：{exc}")
                    script = None
    except Exception as exc:
        print(f" [WARN] 模板載入發生意外：{exc}")
        script = None
 
    # ---- 策略 B：從零建立 ----
    if script is None:
        print(" [INFO] 從零建立新草稿 (1920x1080)。")
        script = draft_folder.create_draft(project_name, 1920, 1080, allow_replace=True)
 
    # 若從模板建立，可能已匯入一些模板軌道；在此加入新軌道用於我們的素材
    # —— 模板軌道的限制也許會在未來 pyJianYingDraft 版本中逐步放寬 ——
    script.add_track(draft_pkg.TrackType.video)   # 圖片/影片軌
    script.add_track(draft_pkg.TrackType.audio)   # 音軌
    script.add_track(draft_pkg.TrackType.text)    # 字幕軌
 
    # (a) 加入圖片片段
    print(" [STEP] 加入圖片片段到時間軸……")
    for item in image_timeline or []:
        image_path = os.path.join(asset_folder, item["image"])
        if not os.path.isfile(image_path):
            print(f" [WARN] 圖片不存在，跳過：{image_path}")
            continue
        start_us = int(item.get("start", 0))
        duration_us = int(item.get("duration", 0))
        if duration_us <= 0:
            continue
        try:
            # 使用 ImageMaterial / VideoSegment 都可，pyJianYingDraft 會自動判斷
            seg = draft_pkg.VideoSegment(
                image_path,
                trange(start_us, duration_us),  # (start, duration) 均以微秒為單位
            )
            script.add_segment(seg)
            print(f"   + 圖片：{item['image']}  @ {start_us/1_000_000:.2f}s  長 {duration_us/1_000_000:.2f}s")
        except Exception as exc:
            print(f" [WARN] 加入圖片片段失敗 ({item['image']}): {exc}")
 
    # (b) 加入音檔片段
    if audio_file and os.path.isfile(audio_file):
        print(" [STEP] 加入音檔片段……")
        try:
            # 若已有總時長，則以總時長為音軌持續時間；否則使用整個音檔
            audio_duration = int(total_duration) if total_duration > 0 else None
            if audio_duration is not None:
                seg = draft_pkg.AudioSegment(audio_file, trange(0, audio_duration), volume=1.0)
            else:
                seg = draft_pkg.AudioSegment(audio_file)  # 全長
            script.add_segment(seg)
            print(f"   + 音檔：{os.path.basename(audio_file)}")
        except Exception as exc:
            print(f" [WARN] 加入音檔片段失敗：{exc}")
 
    # (c) 加入字幕片段（文字）
    if srt_segments:
        print(" [STEP] 加入字幕片段……")
        for srt_item in srt_segments:
            text_content = (srt_item.get("text") or "").strip()
            if not text_content:
                continue
            start_us = int(srt_item.get("start", 0))
            duration_us = int(srt_item.get("duration", 0))
            if duration_us <= 0:
                continue
            try:
                text_seg = draft_pkg.TextSegment(
                    text_content,
                    trange(start_us, duration_us),
                    style=draft_pkg.TextStyle(size=48, color=(1.0, 1.0, 1.0)),
                    clip_settings=draft_pkg.ClipSettings(transform_y=-0.8),  # 畫面下方
                )
                script.add_segment(text_seg)
            except Exception as exc:
                # 某些 pyJianYingDraft 版本的 API 略有差異，做一次備援
                try:
                    text_seg = draft_pkg.TextSegment(text_content, trange(start_us, duration_us))
                    script.add_segment(text_seg)
                except Exception as exc2:
                    print(f" [WARN] 加入字幕片段失敗 ({text_content[:20]}...): {exc2}")
 
    # ---- 儲存 ----
    try:
        script.save()
    except Exception as exc:
        raise RuntimeError(f"儲存草稿失敗：{exc}")
 
    # 計算草稿資料夾位置
    draft_path = os.path.join(draft_root, project_name)
 
    print(f"\n [SUCCESS] 草稿已成功儲存至：{draft_path}")
    print(f"           使用模板：{'是 (' + TEMPLATE_DRAFT_NAME + ')' if use_template else '否（從零建立）'}")
    print(f"           現在可打開剪映查看草稿。")
 
    return {
        "project_name": project_name,
        "draft_path": draft_path,
        "template_used": use_template,
        "template_name": TEMPLATE_DRAFT_NAME if use_template else "",
    }
 
 
# ============================================================
# 7. 主程式
# ============================================================
def main():
    # 7.1 環境檢查（含自動安裝依賴、提示用戶安裝剪映/VideoCaptioner）
    draft_root, vc_root = run_environment_checks()
 
    # 7.2 初始化記錄
    init_logging(CURRENT_SCRIPT_DIR)
    if LOGGER:
        print(f" [LOG ] {LOGGER.log_path}")
 
    # 7.3 選擇 model1 / model2
    print("\n------------------------------------------------------")
    print("  請選擇要處理的素材目錄：")
    print("  [1] shared_cargo/model1 (PPT 匯出)")
    print("  [2] shared_cargo/model2 (程式提取)")
    print("------------------------------------------------------")
 
    track_choice = ""
    while track_choice not in ("1", "2"):
        sys.stdout.write("  輸入選項 (1 或 2) 後按 Enter：")
        sys.stdout.flush()
        try:
            line = sys.stdin.readline()
            if not line:
                return
            track_choice = line.strip()
        except KeyboardInterrupt:
            print("\n [INFO] 使用者取消。")
            return
        except Exception:
            track_choice = "1"
 
    sub_model_folder = f"model{track_choice}"
    asset_folder = os.path.join(CURRENT_SCRIPT_DIR, "shared_cargo", sub_model_folder)
    print(f"\n [LOAD] 目標素材目錄：{asset_folder}")
 
    os.makedirs(asset_folder, exist_ok=True)
 
    audio_file = os.path.join(asset_folder, "audio.mp4")
    txt_file = os.path.join(asset_folder, "script.txt")
    srt_file = os.path.join(asset_folder, "subtitle.srt")
 
    if not os.path.isfile(txt_file):
        print(f" [ERROR] 找不到 {txt_file} (script.txt)，無法繼續。")
        return
 
    image_list = sorted(
        [
            fn for fn in os.listdir(asset_folder)
            if fn.lower().endswith((".jpg", ".jpeg", ".png"))
        ]
    )
    if not image_list:
        print(f" [WARN] {asset_folder} 中未發現 slide 圖片（.jpg/.jpeg/.png），草稿可能僅有音訊與字幕。")
 
    has_audio = os.path.isfile(audio_file)
    srt_segments = []
    image_timeline = []
    total_duration = 0
 
    # 7.4 若有音檔，則產生字幕；否則以靜音模式（均分圖片時長）
    if has_audio:
        print(f"\n [STEP 1] 透過 VideoCaptioner runtime 產生 subtitle.srt ……")
        vc_exe = find_vc_executable(CURRENT_SCRIPT_DIR)
        if not vc_exe:
            print(" [ERROR] 找不到 VideoCaptioner.exe，請確認 vc_location.txt 或重新安裝。")
            print("         本步驟跳過，將改用靜音模式（均分圖片時長）。")
            has_audio = False
        else:
            try:
                generate_srt_with_videocaptioner_runtime(
                    vc_executable_path=vc_exe,
                    media_path=audio_file,
                    script_path=txt_file,
                    output_srt=srt_file,
                )
                print("\n [OK] Step 1 完成：subtitle.srt 已生成。")
            except FileNotFoundError as exc:
                print(f"\n [ERROR] {exc}")
                has_audio = False
            except Exception as exc:
                print(f"\n [ERROR] Step 1 失敗：{exc}")
                print("         請查看上方 runtime 輸出或 log 檔除錯。")
                has_audio = False
 
        if has_audio and not os.path.isfile(srt_file):
            print(" [ERROR] VideoCaptioner 執行結束但未產生 subtitle.srt。")
            has_audio = False
 
    if has_audio and os.path.isfile(srt_file):
        print(f"\n [STEP 2] 文字匹配並建立圖片時間軸……")
        srt_segments, image_timeline, total_duration = parse_srt_by_text_matching(
            asset_folder, srt_file, image_list
        )
    else:
        print("\n [INFO] 未啟用字幕生成（靜音模式），依賴 slide 附註的靜態時長分配。")
        if image_list:
            total_duration = len(image_list) * 5 * 1000 * 1000
            image_timeline = build_even_image_timeline(image_list, total_duration)
 
    # 7.5 以 pyJianYingDraft 注入草稿
    print("\n [STEP 3] 使用 pyJianYingDraft 產生剪映草稿……")
    result = inject_timeline_into_jianying_template(
        asset_folder=asset_folder,
        sub_model_folder=sub_model_folder,
        image_list=image_list,
        audio_file=audio_file if has_audio else "",
        srt_segments=srt_segments,
        image_timeline=image_timeline,
        total_duration=total_duration,
        draft_root=draft_root,
    )
 
    print("\n=================== 全部步驟完成 ===================")
    print(f" [OUT] 素材目錄    ：{asset_folder}")
    print(f" [OUT] 字幕檔      ：{srt_file if (has_audio and os.path.isfile(srt_file)) else '(未生成)'}")
    print(f" [OUT] 剪映草稿路徑：{result['draft_path']}")
    print(f" [OUT] 草稿名稱    ：{result['project_name']}")
    if result.get("template_used"):
        print(f" [OUT] 使用模板    ：{result['template_name']}")
    else:
        print(f" [OUT] 使用模板    ：否（從零建立 1920x1080）")
    if LOGGER:
        print(f" [OUT] 完整 log    ：{LOGGER.log_path}")
 
 
if __name__ == "__main__":
    main()