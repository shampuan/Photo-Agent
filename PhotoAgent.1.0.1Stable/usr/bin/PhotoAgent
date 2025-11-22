#!/usr/bin/env python3

# Aydınlatan ve geliştiren bilimin adıyla:

import sys
import os
import hashlib
import shutil
import stat
from datetime import datetime
import getpass
from urllib.parse import quote
import subprocess
import platform
import json
import configparser 

# --- PYQT5 İMPORTLARI ---
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLineEdit, QLabel, QListWidget, QTableWidget,
    QTableWidgetItem, QHeaderView, QGroupBox, QCheckBox, QProgressBar,
    QAbstractItemView, QFileDialog, QMessageBox, QTextEdit,
    QRadioButton, QButtonGroup, QAbstractButton, QTabWidget,
    QFileIconProvider, QMenu, QListWidgetItem, QSizePolicy 
)
from PyQt5.QtCore import Qt, pyqtSignal as Signal, pyqtSlot as Slot, QThread, QLocale, QSize, QFileInfo
from PyQt5.QtGui import QColor, QBrush, QIcon, QPixmap, QFont, QGuiApplication
# PyQt5'te QtSvg genellikle ayrı bir pakettedir. Bu yüzden belirtmek zorundayım. 
try:
    from PyQt5.QtSvg import QSvgRenderer 
except ImportError:
    # Eğer QtSvg yoksa, kodu çalıştırmaya devam et.
    QSvgRenderer = None
# --- PYQT5 İMPORTLARI SONU ---

# --- GNOME/Qt Platform Plugin Fix ---
os.environ['QT_QPA_PLATFORM'] = 'xcb' 
# ------------------------------------

# --- GLOBAL DİL VERİLERİ ve PROGRAM ADI GÜNCELLEMESİ ---
DEFAULT_LANG = "en"
CURRENT_LANG = DEFAULT_LANG

# Dil verileri için global değişken
_texts = {}

# FİLTRE SABİTLERİ 
EXTENSION_FILTERS = {
    # Minimal sürümde sadece 'image' filtresi zorunludur.
    "image": [".jpg", ".jpeg", ".png", ".gif", ".bmp", ".tiff", ".webp", ".svg", ".ico", ".raw"],
}

# --- KARANLIK TEMA QSS TANIMI (Aynı Kaldı çünkü QT5de sistem temasına tam uyum yok böyle kalsın) ---
DARK_THEME_QSS = """
/* Genel Widget Ayarları */
QWidget {
    background-color: #2b2b2b;
    color: #ffffff;
}
QMainWindow {
    background-color: #2b2b2b;
}

/* Başlık Etiketi (Photo Agent Yazısı) - Mavi */
QLabel#logo_text_label { 
    color: #0078D4;
}

/* Butonlar */
QPushButton {
    background-color: #3e3e3e;
    border: 1px solid #555555;
    border-radius: 4px;
    padding: 5px 10px;
    color: #ffffff;
}
QPushButton:hover {
    background-color: #505050;
}
QPushButton:pressed {
    background-color: #606060;
}

/* Giriş Alanları, Listeler ve Tablolar */
QLineEdit, QListWidget, QTableWidget {
    background-color: #3c3c3c;
    border: 1px solid #555555;
    color: #ffffff;
    selection-background-color: #0078D4; /* Seçili öğe arka planı */
}

/* Tablo Başlıkları */
QHeaderView::section {
    background-color: #454545;
    color: #ffffff;
    padding: 4px;
    border: 1px solid #555555;
}

/* Sekmeler (QTabWidget) */
QTabWidget::pane { /* Sekme çerçevesi */
    border: 1px solid #555555;
    top: -1px;
    background: #2b2b2b;
}
QTabBar::tab {
    background: #3e3e3e;
    border: 1px solid #555555;
    border-bottom-color: #2b2b2b; 
    border-top-left-radius: 4px;
    border-top-right-radius: 4px;
    padding: 5px;
    min-width: 80px;
}
QTabBar::tab:selected {
    background: #2b2b2b;
    border-bottom: 2px solid #0078D4; 
    color: #ffffff;
}

/* İlerleme Çubuğu */
QProgressBar {
    border: 1px solid #555555;
    border-radius: 5px;
    text-align: center;
    background-color: #3c3c3c;
    color: #ffffff;
}
QProgressBar::chunk {
    background-color: #0078D4; 
    border-radius: 5px;
}

/* Özel Buton Stilleri */
#add_dir_btn { 
    background-color: #555555;
}
#add_dir_btn:hover {
    background-color: #6a6a6a;
}
"""
# --- KARANLIK TEMA QSS SONU ---

def load_language_files():
    """Dil dosyalarını (.ini) program dizinindeki 'languages' klasöründen veya sistem yolundan dinamik olarak yükler."""
    global _texts
    _texts = {}
    
    # Python dosyasının mutlak yolunu al
    script_path = os.path.abspath(__file__)
    script_dir = os.path.dirname(script_path)
    
    # Öncelik sırasına göre arama yolları, program gerekli dil ikon vb öğeleri bu yollardan çeksin diye
    search_paths = [
        # 1. Programın bulunduğu dizindeki 'languages' klasörü
        os.path.join(script_dir, 'languages'),
        # 2. Programın bulunduğu dizin (INI dosyaları doğrudan burada olabilir)
        script_dir,
        # 3. Sistem çapındaki '/usr/share/PhotoAgent/languages'
        '/usr/share/PhotoAgent/languages'
    ]

    config = configparser.ConfigParser()
    loaded_lang_codes = set() # Yüklenen dilleri takip etmek için set

    for path in search_paths:
        if os.path.isdir(path):
            # Dizin içindeki tüm .ini dosyalarını tara
            for file_name in os.listdir(path):
                if file_name.endswith('.ini'):
                    file_path = os.path.join(path, file_name)
                    
                    try:
                        # ConfigParser, bir sonraki okumada önceki içeriği temizlemez, bu yüzden yeniden başlatırız veya okumadan sonra temizleriz.
                        # En güvenlisi, her dosya için yeni bir ConfigParser kullanmaktır.
                        temp_config = configparser.ConfigParser()
                        temp_config.read(file_path, encoding='utf-8')
                        
                        # [META] bölümünden dil kodunu al
                        if 'META' in temp_config and 'LANG_CODE' in temp_config['META']:
                            lang_code = temp_config['META']['LANG_CODE'].lower()
                            
                            # Eğer bu dil daha önce yüklenmediyse ve [TEXTS] bölümü varsa
                            if lang_code not in loaded_lang_codes and 'TEXTS' in temp_config:
                                # [TEXTS] bölümündeki tüm çevirileri _texts sözlüğüne ekle
                                # dict() kullanımı, ConfigParser'ın dahili yapısını kopyalar.
                                _texts[lang_code] = dict(temp_config['TEXTS'])
                                loaded_lang_codes.add(lang_code)
                                
                    except Exception:
                        # Hata durumunda diğer dosyaları yüklemeye devam et.
                        pass
                        
    # Eğer varsayılan dil (en) yüklenemezse (dosya bulunamazsa),
    # programın çökmemesi için boş bir sözlük tanımla.
    if DEFAULT_LANG not in _texts:
        _texts[DEFAULT_LANG] = {}

def get_available_languages():
    """Yüklenen tüm dillerin listesini döndürür."""
    # Sadece [TEXTS] bölümünü başarıyla yüklediğimiz dilleri döndürürüz.
    return list(_texts.keys())

def get_text(key, lang=None):
    """Belirtilen anahtar için çeviriyi döndürür. Bulunamazsa 'en' veya hata anahtarını döndürür."""
    if lang is None:
        lang = CURRENT_LANG
    
    key_lower = key.lower() # INI okuması keys'i küçük harf yapar
    
    # 1. İstenen dilde ara
    if lang in _texts and key_lower in _texts[lang]:
        return _texts[lang][key_lower]
    
    # 2. Varsayılan dilde ('en') ara
    if lang != DEFAULT_LANG and DEFAULT_LANG in _texts and key_lower in _texts[DEFAULT_LANG]:
        return _texts[DEFAULT_LANG][key_lower]
    
    # 3. Bulunamadı hatası
    return f"[{key}]"

def save_language_preference(lang):
    """Kullanıcının dil tercihini kaydeder."""
    config_dir = os.path.expanduser('~/.photoagent')
    config_file = os.path.join(config_dir, 'settings.ini')
    
    try:
        os.makedirs(config_dir, exist_ok=True)
        config = configparser.ConfigParser()
        # Eğer dosya varsa, mevcut ayarları okur
        if os.path.exists(config_file):
             config.read(config_file, encoding='utf-8')
             
        if 'PREFERENCES' not in config:
            config['PREFERENCES'] = {}
            
        config['PREFERENCES']['language'] = lang
        
        with open(config_file, 'w', encoding='utf-8') as f:
            config.write(f)
    except Exception as e:
        print(f"Dil tercihi kaydedilemedi: {e}")

def load_language_preference():
    """Kaydedilen dil tercihini yükler."""
    config_file = os.path.join(os.path.expanduser('~/.photoagent'), 'settings.ini')
    
    if os.path.exists(config_file):
        try:
            config = configparser.ConfigParser()
            config.read(config_file, encoding='utf-8')
            
            if 'PREFERENCES' in config and 'language' in config['PREFERENCES']:
                # Yüklenen dilin geçerli diller arasında olup olmadığını kontrol et
                loaded_lang = config['PREFERENCES']['language']
                if loaded_lang in get_available_languages():
                    return loaded_lang
        except Exception as e:
            print(f"Dil tercihi yüklenemedi: {e}")
    
    return DEFAULT_LANG

# ----------------------------------------------------------------------
# 0. YARDIMCI FONKSİYONLAR (Burası aynı kaldı değişikliğe gerek yok)
# ----------------------------------------------------------------------

def _find_icon_path(icon_name="dup2.png"):
    current_working_dir = os.getcwd()
    path_in_cwd = os.path.join(current_working_dir, icon_name)
    if os.path.exists(path_in_cwd):
        return path_in_cwd

    system_path = os.path.join("/usr/share/PhotoAgent/", icon_name)
    if os.path.exists(system_path):
        return system_path
    
    # Örnek bir yedek ikon yolu (mevcut sistemde yoksa)
    # return "/usr/share/icons/hicolor/scalable/apps/photoagent.svg" Henüz yok, muhtemelen hata verecektir. 

    return None

def open_path_in_os(path):
    system = platform.system()
    try:
        if system == "Windows":
            subprocess.run(['start', '', path], shell=True, check=True)
        elif system == "Darwin":
            subprocess.run(['open', path], check=True)
        elif system == "Linux":
            subprocess.run(['xdg-open', path], check=True)
        else:
            return False
        return True
    except subprocess.CalledProcessError as e:
        print(f"HATA: Dosya/Klasör açılamadı: {e}")
        return False
    except FileNotFoundError:
        print(f"HATA: İşletim sistemi komutu bulunamadı.")
        return False

def get_mount_point(path):
    """Verilen dosya yolunun bağlı olduğu mount noktasını bulur."""
    path = os.path.abspath(path)
    if platform.system() == "Linux" or platform.system() == "Darwin":
        while not os.path.ismount(path):
            parent = os.path.dirname(path)
            if parent == path: 
                break
            path = parent
        return path
    elif platform.system() == "Windows":
        drive, tail = os.path.splitdrive(path)
        if drive:
            return drive + os.path.sep
        return path 
    else:
        return os.path.abspath(os.path.sep)

# ----------------------------------------------------------------------
# 1. HASH VE TARAMA MANTIĞI (Aynı kaldı)
# ----------------------------------------------------------------------

def calculate_md5(filepath, chunk_size=4096):
    """Büyük dosyalar için belleği yormadan MD5 hash'ini hesaplar."""
    hasher = hashlib.md5()
    try:
        with open(filepath, 'rb') as file:
            while True:
                chunk = file.read(chunk_size)
                if not chunk:
                    break
                hasher.update(chunk)
        return hasher.hexdigest()
    except IOError:
        return None

def format_size(size):
    """Bayt cinsinden boyutu okunabilir formata çevirir."""
    for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
        if size < 1024.0:
            return f"{size:3.1f} {unit}"
        size /= 1024.0
    return f"{size:3.1f} PB"

class WorkerThread(QThread):
    # PySide6'daki Signal yerine PyQt5'teki pyqtSignal (Signal olarak yeniden adlandırıldı)
    progress_updated = Signal(int)
    status_message = Signal(str)
    scan_finished = Signal(list)

    def __init__(self, target_dirs, options, parent=None):
        super().__init__(parent)
        self.target_dirs = target_dirs
        self.options = options
        self._is_running = True

    def run(self):
        # --- FİLTRELEME İŞLEMİNİN HAZIRLANMASI BAŞLANGICI (Sadece Görsel) ---
        allowed_extensions = {ext.lower() for ext in EXTENSION_FILTERS.get("image", [])}
        is_filtering_active = True
        
        # Hidden/System Dosya Kontrolü
        ignore_hidden = self.options["ignore"]["ignore_system_hidden"]
        # --- FİLTRELEME İŞLEMİNİN HAZIRLANMASI SONU ---

        self.status_message.emit(get_text("status_scanning"))
        all_files_by_size = {}
        total_files = 0

        for base_dir in self.target_dirs:
            if not self._is_running: return

            for root, dirs, files in os.walk(base_dir):
                if not self._is_running: return

                # Gizli dizinleri atlama
                if ignore_hidden:
                    dirs[:] = [d for d in dirs if not d.startswith('.')]

                for file_name in files:
                    full_path = os.path.join(root, file_name)

                    try:
                        file_stats = os.stat(full_path)
                        file_size = file_stats.st_size
                    except:
                        continue

                    if self.options["ignore"]["ignore_zero_byte"] and file_size == 0:
                        continue
                        
                    # Gizli Dosya Kontrolü
                    if ignore_hidden and (file_name.startswith('.') or 
                                          (platform.system() == "Windows" and 
                                           file_stats.st_file_attributes & stat.FILE_ATTRIBUTE_HIDDEN)):
                        continue

                    # UZANTI FİLTRELEME UYGULAMASI
                    if is_filtering_active:
                        file_ext = os.path.splitext(file_name)[1].lower()
                        if file_ext not in allowed_extensions:
                            continue

                    if file_size not in all_files_by_size:
                        all_files_by_size[file_size] = []
                    all_files_by_size[file_size].append(full_path)
                    total_files += 1

        candidate_groups = {size: paths for size, paths in all_files_by_size.items() if len(paths) > 1}
        total_candidates = sum(len(paths) for paths in candidate_groups.values())

        if total_candidates == 0:
            # {0} dosya bulundu
            self.status_message.emit(get_text("status_finished_none").format(total_files))
            self.scan_finished.emit([])
            return

        # {0} aday için hash hesaplanıyor...
        self.status_message.emit(get_text("status_hashing").format(total_candidates))

        files_by_hash = {}
        processed_count = 0

        for size, file_paths in candidate_groups.items():
            for file_path in file_paths:
                if not self._is_running: return

                processed_count += 1
                progress = int((processed_count / total_candidates) * 100)
                self.progress_updated.emit(progress)
                # Hashleniyor: {0}
                self.status_message.emit(get_text("status_hashing_file").format(os.path.basename(file_path)))

                file_hash = calculate_md5(file_path)

                if not file_hash:
                    continue

                if file_hash:
                    if file_hash not in files_by_hash:
                        files_by_hash[file_hash] = []
                    files_by_hash[file_hash].append(file_path)

        final_duplicates = []
        for file_hash, file_paths in files_by_hash.items():
            if len(file_paths) > 1:
                try:
                    file_size_bytes = os.stat(file_paths[0]).st_size
                    final_duplicates.append({
                        "hash": file_hash,
                        "size_bytes": file_size_bytes, 
                        "size": format_size(file_size_bytes),
                        "files": file_paths
                    })
                except:
                    continue

        # Tarama bitti. {0} kopya grubu bulundu.
        self.status_message.emit(get_text("status_finished").format(len(final_duplicates)))

        self.progress_updated.emit(100)
        self.scan_finished.emit(final_duplicates)

    def stop(self):
        self._is_running = False

# ----------------------------------------------------------------------
# 2. FAKE TRASH YÖNETİM SINIFI (Aynı Kaldı)
# ----------------------------------------------------------------------

class FakeTrashManager:
    """Sahte Çöp Kutusu dizinlerini ve metadata dosyasını DİSK BAZLI yönetir."""

    def __init__(self):
        self.base_config_dir = os.path.join(os.path.expanduser('~'), '.photoagent')

    def _get_trash_paths(self, filepath):
        """Dosyanın bulunduğu diske (mount noktasına) göre çöp dizini ve metadata yollarını döndürür."""
        
        mount_point = get_mount_point(filepath) 
        # Program adıyla uyumlu yeni Fake Trash yolu
        trash_dir = os.path.join(mount_point, '.Photoagent-Trash-1000') 
        metadata_path = os.path.join(trash_dir, 'trashdata.json')
        
        return trash_dir, metadata_path

    def _setup_disk_dirs(self, trash_dir):
        """Belirtilen diske ait çöp dizinini ve metadata dosyasını oluşturur."""
        metadata_path = os.path.join(trash_dir, 'trashdata.json')
        try:
            os.makedirs(trash_dir, exist_ok=True)
            if not os.path.exists(metadata_path):
                with open(metadata_path, 'w', encoding='utf-8') as f:
                    json.dump([], f, indent=4)
        except Exception as e:
            print(f"HATA: Disk Bazlı Fake Trash dizinleri oluşturulamadı: {e}")

    def _load_metadata(self, metadata_path):
        """Belirtilen metadata dosyasını okur ve JSON listesi döndürür."""
        try:
            with open(metadata_path, 'r', encoding='utf-8') as f:
                return json.load(f)
        except (FileNotFoundError, json.JSONDecodeError):
            return []

    def _save_metadata(self, data, metadata_path):
        """Metadata listesini belirtilen dosyaya yazar."""
        try:
            with open(metadata_path, 'w', encoding='utf-8') as f:
                json.dump(data, f, indent=4)
            return True
        except Exception as e:
            print(f"HATA: Metadata yazılamadı: {e}")
            return False

    def move_to_trash(self, filepath, file_size_bytes):
        """Dosyayı kendi diskindeki FakeTrash'a taşır ve metadata kaydını oluşturur."""
        original_path = os.path.abspath(filepath)
        
        trash_dir, metadata_path = self._get_trash_paths(original_path)
        self._setup_disk_dirs(trash_dir) 
        
        target_filename = os.path.basename(original_path)
        
        counter = 0
        name, ext = os.path.splitext(target_filename)
        postfix = int(datetime.now().timestamp() * 1000) 
        trash_filename = f"{name}_{postfix}{ext}"

        while os.path.exists(os.path.join(trash_dir, trash_filename)):
            counter += 1
            trash_filename = f"{name}_{postfix}_{counter}{ext}"

        target_path = os.path.join(trash_dir, trash_filename)
        
        try:
            shutil.move(original_path, target_path)

            metadata = self._load_metadata(metadata_path)
            new_entry = {
                "trash_filename": trash_filename,
                "original_path": original_path,
                "deletion_date": datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                "size": format_size(file_size_bytes),
                "size_bytes": file_size_bytes,
                "trash_dir": trash_dir 
            }
            metadata.append(new_entry)
            self._save_metadata(metadata, metadata_path)
            
            return True
        except Exception as e:
            print(f"Taşıma Hatası (Disk Bazlı): {e}")
            return False

    def get_trash_files(self):
        """Bu metot artık kullanılmayacak veya FakeTrashApp tarafından yönetilecek."""
        raise NotImplementedError("Disk bazlı yönetim nedeniyle bu metot artık DuplicateFinderApp tarafından yönetilmelidir.")

    def restore_file(self, trash_filename, original_path, trash_dir):
        """Dosyayı FakeTrash'tan orijinal konumuna geri yükler."""
        trash_file_path = os.path.join(trash_dir, trash_filename)
        metadata_path = os.path.join(trash_dir, 'trashdata.json')
        
        if not os.path.exists(trash_file_path):
            return False 

        try:
            original_dir = os.path.dirname(original_path)
            os.makedirs(original_dir, exist_ok=True)

            shutil.move(trash_file_path, original_path)

            metadata = self._load_metadata(metadata_path)
            metadata = [item for item in metadata if not (item["trash_filename"] == trash_filename and item["original_path"] == original_path)]
            self._save_metadata(metadata, metadata_path)
            
            return True
        except Exception as e:
            print(f"Geri Yükleme Hatası: {e}")
            return False

    def purge_file(self, trash_filename, original_path, trash_dir):
        """Dosyayı FakeTrash'tan kalıcı olarak siler (diskten siler)."""
        trash_file_path = os.path.join(trash_dir, trash_filename)
        metadata_path = os.path.join(trash_dir, 'trashdata.json')
        
        try:
            if os.path.exists(trash_file_path):
                os.remove(trash_file_path)

            metadata = self._load_metadata(metadata_path)
            metadata = [item for item in metadata if not (item["trash_filename"] == trash_filename and item["original_path"] == original_path)]
            self._save_metadata(metadata, metadata_path)
            
            return True
        except Exception as e:
            print(f"Kalıcı Silme Hatası: {e}")
            return False


# ----------------------------------------------------------------------
# 3. ANA PENCERE (PhotoFinderApp)
# ----------------------------------------------------------------------

class PhotoFinderApp(QMainWindow):

    GROUP_COLORS = [QColor("#3cb5ff"), QColor("#d7b981")] # Renkler koyu temaya uygun (açık mavi, kahverengi)

    def __init__(self):
        global CURRENT_LANG
        super().__init__()

        self.icon_path = _find_icon_path()
        self.trash_manager = FakeTrashManager() 
        self.duplicate_data = [] 
        
        # Daha minimalist bir boyut
        self.setGeometry(100, 100, 850, 650) 
        self.worker_thread = None

        if self.icon_path:
            self.setWindowIcon(QIcon(self.icon_path))
        else:
            self.setWindowIcon(QIcon(":/qt-project.org/qmessagebox/images/information.png"))

        # <<< PYQT5 İÇİN SVG KONTROLÜ >>>
        if QSvgRenderer is not None:
            try:
                # QSvgRenderer'ı sadece çağrılamak bile arka planda gerekli ayarları yapabilir.
                _svg_renderer_check = QSvgRenderer()
            except Exception as e:
                print(f"QSvgRenderer başlatılamadı: {e}")
                pass
        # <<< PYQT5 SVG KONTROLÜ SONU >>>


        self.central_widget = QWidget()
        self.setCentralWidget(self.central_widget)
        
        self._setup_ui()
        self._setup_dir_management() 
        self._connect_signals()      
        
        # Dil dosyalarını yükle (INI okuma mekanizması aktif)
        load_language_files()
        
        # <<< KRİTİK: DİL YÜKLEME VE ARAYÜZ GÜNCELLEME >>>
        saved_lang = load_language_preference()
        self._update_gui_texts(saved_lang) 
        # <<< KRİTİK SONU >>>

    # <<< YARDIMCI METOT: DOSYA İKONUNU GETİRME (Aynı Kaldı) >>>
    def _get_file_icon(self, file_path):
        """Dosya yoluna göre sistemin varsayılan dosya ikonunu döndürür."""
        file_info = QFileInfo(file_path)
        provider = QFileIconProvider()
        return provider.icon(file_info)

    # <<< YENİ METOT: THUMBNAIL OLUŞTURMA (Aynı Kaldı) >>>
    def _create_thumbnail(self, file_path, size=80): # DEFAULT BOYUT 80'E DÜŞÜRÜLDÜ
        """Dosya yolundan QPixmap oluşturur ve yeniden boyutlandırır."""
        try:
            pixmap = QPixmap(file_path)
            if pixmap.isNull():
                return self._get_file_icon(file_path).pixmap(size, size) 
                
            return pixmap.scaled(size, size, Qt.KeepAspectRatio, Qt.SmoothTransformation)
        except Exception:
            # Hata durumunda (e.g., dosya kilitli) varsayılan ikon
            return self._get_file_icon(file_path).pixmap(size, size) 
    # <<< YENİ METOT SONU >>>


    @Slot()
    def _update_gui_texts(self, lang):
        """Arayüzdeki tüm metinleri seçilen dile göre günceller."""
        global CURRENT_LANG
        CURRENT_LANG = lang

        try:
            self.setWindowTitle(get_text("title", lang))

            # Logo
            if self.icon_path:
                scaled_size = 80 
                pixmap = QPixmap(self.icon_path)
                self.logo_icon_label.setPixmap(pixmap.scaled(scaled_size, scaled_size, Qt.KeepAspectRatio, Qt.SmoothTransformation))
                self.logo_icon_label.setVisible(True)
            else:
                 self.logo_icon_label.setVisible(False)
            
            # QLabel stilini korumak için objectName eklenmiştir (QSS için)
            self.logo_text_label.setObjectName("logo_text_label")
            self.logo_text_label.setText(get_text("title", lang))

            # Dizin Seçimi
            self.dir_input.setPlaceholderText(get_text("select_dir_placeholder", lang))
            self.add_dir_btn.setText(get_text("add_dir", lang))
            
            # Sekmeler
            self.tab_widget.setTabText(0, get_text("tab_scan", lang))
            self.tab_widget.setTabText(1, get_text("tab_trash", lang))
            
            # Sonuçlar Listesi
            self.found_label.setText(get_text("found_duplicates", lang))
            
            # Tarama/Silme Butonları (Sonuçlar sekmesinde)
            self.delete_button.setText(get_text("delete_selected", lang))
            # Delete butonunun rengi QSS tarafından yönetilir.
            self.delete_button.setStyleSheet("background-color: #f44336; color: white; font-weight: bold; min-height: 35px;") 

            # Tarama Butonu ve Durum
            is_scanning = self.worker_thread and self.worker_thread.isRunning()
            
            # Scan Butonunun rengi QSS tarafından yönetilir, sadece metin ve arka plan rengi burada kalır.
            scan_button_style = "font-weight: bold; min-height: 35px;"

            # Buton metinlerini güncellemek için mevcut durumu kontrol et
            current_start_text = self.start_button.text()
            
            if is_scanning:
                 self.start_button.setText(get_text("cancel_scan", lang))
                 self.start_button.setStyleSheet(scan_button_style + "background-color: #FFA500;")
            elif current_start_text == get_text("start_scan", lang) or current_start_text == get_text("rescan", lang):
                 # Eğer yeni dildeki metinle eşleşmiyorsa, bir şey yapma (zaten doğru ayarlanmış)
                 pass
            elif current_start_text in [get_text("start_scan", c) for c in get_available_languages()]:
                 # Eğer önceki dildeki "Start Scan" ise yeni dildeki "Start Scan" yap
                 self.start_button.setText(get_text("start_scan", lang))
                 self.start_button.setStyleSheet(scan_button_style + "background-color: #4CAF50;")
            elif current_start_text in [get_text("rescan", c) for c in get_available_languages()]:
                 # Eğer önceki dildeki "Rescan" ise yeni dildeki "Rescan" yap
                 self.start_button.setText(get_text("rescan", lang))
                 self.start_button.setStyleSheet(scan_button_style + "background-color: #4CAF50;")
            else:
                 # Varsayılan başlangıç
                 self.start_button.setText(get_text("start_scan", lang))
                 self.start_button.setStyleSheet(scan_button_style + "background-color: #4CAF50;")
            
            
            self.language_button.setText(get_text("language", lang))
            self.about_button.setText(get_text("about", lang))

            # Fake Trash Tablosu
            # KRİTİK: Tablo başlıkları güncellenmeli.
            self.trash_table.setHorizontalHeaderLabels(["", get_text("trash_col_file", lang), get_text("trash_col_original_path", lang), get_text("trash_col_deletion_date", lang)])
            self.restore_button.setText(get_text("trash_restore", lang))
            self.purge_button.setText(get_text("trash_purge", lang))
            self.select_all_trash_button.setText(get_text("select_all", lang))
            self.unselect_all_trash_button.setText(get_text("unselect_all", lang))


            current_status = self.status_label.text()
            # Durum etiketini prefix ile yeniden oluştur
            # Tüm dillerdeki "Ready to scan" veya "Taramaya hazır" gibi durumları kontrol etmek zor olduğundan,
            # sadece prefix'i güncellemek en güvenli yaklaşımdır.
            current_prefix = get_text("status_prefix", lang)
            if ":" in current_status:
                status_message_without_prefix = current_status.split(":", 1)[1].strip()
                # Eğer mesajda eski dildeki "Ready to scan" varsa, yeni dildeki "Ready to scan" ile değiştir
                if status_message_without_prefix.lower().strip() in [get_text("status_ready", c).lower().strip() for c in get_available_languages()]:
                     self.status_label.setText(f'{current_prefix}: {get_text("status_ready", lang)}')
                else:
                    # Diğer karmaşık mesajlar için sadece prefix'i güncelle
                    self.status_label.setText(f'{current_prefix}: {status_message_without_prefix}')
            else:
                self.status_label.setText(f'{current_prefix}: {get_text("status_ready", lang)}')

            # Eğer çöp kutusu sekmesindeysek tabloyu yeniden yükle (Başlıklar değiştiği için)
            if self.tab_widget.currentIndex() == 1:
                self.update_trash_tab()

        except AttributeError:
            # UI öğeleri henüz oluşturulmamışsa (init sırasında ilk çağrı) hata verme
            pass

    @Slot()
    def _show_language_menu(self):
        """Dil seçimi menüsünü açar."""
        menu = QMenu(self)
        available_langs = get_available_languages()
        
        # Dil kodlarını tam adlara çevirme sözlüğü. (INI dosyalarındaki [META] bölümünden çekilebilir, ancak basitleştirildi)
        lang_names = {
            'en': 'English', 'tr': 'Türkçe', 'de': 'Deutsch', 'fr': 'Français', 'es': 'Español',
            'it': 'Italiano', 'pt': 'Português', 'ru': 'Русский', 'zh': '中文', 'ja': '日本語',
            'ko': '한국어', 'ar': 'العربية'
        }
        
        for lang_code in available_langs:
            lang_name = lang_names.get(lang_code, lang_code.upper())
            action = menu.addAction(lang_name)
            action.setData(lang_code)
            if lang_code == CURRENT_LANG:
                action.setCheckable(True)
                action.setChecked(True)
        
        # Butonun hemen altında açılması için pozisyon hesapla
        action = menu.exec(self.language_button.mapToGlobal(self.language_button.rect().bottomLeft()))
        if action:
            selected_lang = action.data()
            save_language_preference(selected_lang)
            self._update_gui_texts(selected_lang)

    def _setup_dir_management(self):
        # Sadece dizin seçme butonu kaldı.
        self.add_dir_btn.clicked.connect(self._open_dir_dialog)
        self.dir_input.setText(os.path.expanduser('~')) # Varsayılan olarak Ev Dizini

    @Slot()
    def _open_dir_dialog(self):
        dialog_title = get_text("add_dir").replace("...", "").strip()
        # Mevcut dizin veya ev dizini ile aç
        home_dir = self.dir_input.text() or os.path.expanduser('~')
        selected_dir = QFileDialog.getExistingDirectory(self, dialog_title, home_dir)
        if selected_dir:
            self.dir_input.setText(selected_dir)


    def _setup_ui(self):
        
        # -----------------------------------------------------
        # 1. ÜST ARAÇ ÇUBUĞU (Dizin Seçimi, Logo, Butonlar)
        # -----------------------------------------------------
        top_bar_widget = QWidget()
        top_bar_layout = QHBoxLayout(top_bar_widget)
        top_bar_layout.setContentsMargins(5, 5, 5, 5)
        top_bar_layout.setAlignment(Qt.AlignLeft | Qt.AlignVCenter)

        # LOGO
        self.logo_icon_label = QLabel()
        self.logo_text_label = QLabel()
        self.logo_text_label.setObjectName("logo_text_label") # QSS için
        
        logo_text_container = QWidget()
        logo_v_layout = QVBoxLayout(logo_text_container)
        logo_v_layout.setContentsMargins(0, 0, 0, 0)
        logo_v_layout.setSpacing(0)
        
        # Stil QSS'e taşındı ancak öncelik için burada tutulabilir.
        self.logo_text_label.setStyleSheet("font-size: 20px; font-weight: bold; margin-left: 5px;") 
        self.subtitle_label = QLabel("Duplicate Photo Finder and Cleaner")
        self.subtitle_label.setStyleSheet("font-size: 12px; font-style: italic; color: #aaaaaa; margin-left: 5px;") # Koyu temaya uygun renk

        
        # DİZİN SEÇİMİ
        dir_widget = QWidget()
        dir_layout = QHBoxLayout(dir_widget)
        dir_layout.setContentsMargins(0, 0, 0, 0)
        self.dir_input = QLineEdit()
        self.dir_input.setPlaceholderText(get_text("select_dir_placeholder")) 
        self.dir_input.setReadOnly(True) 
        self.add_dir_btn = QPushButton(get_text("add_dir"))
        self.add_dir_btn.setObjectName("add_dir_btn") # QSS için
        self.add_dir_btn.setFixedSize(130, 35) 
        self.dir_input.setMinimumWidth(300)
        dir_layout.addWidget(self.dir_input)
        dir_layout.addWidget(self.add_dir_btn)

        # EYLEM BUTONLARI (Dil ve Hakkında)
        self.language_button = QPushButton()
        self.about_button = QPushButton()
        self.language_button.setFixedSize(90, 35) 
        self.about_button.setFixedSize(90, 35) 

        # top_bar_layout'a bileşenleri ekle
        top_bar_layout.addWidget(self.logo_icon_label)
        
        logo_v_layout.addWidget(self.logo_text_label)
        logo_v_layout.addWidget(self.subtitle_label)
        
        top_bar_layout.addWidget(logo_text_container)
        
        top_bar_layout.addSpacing(20)
        top_bar_layout.addWidget(dir_widget)
        top_bar_layout.addStretch(1) # Boşluk
        top_bar_layout.addWidget(self.language_button)
        top_bar_layout.addWidget(self.about_button)
        
        # -----------------------------------------------------
        # 2. SEKMELER (Tarama Sonuçları ve Fake Trash)
        # -----------------------------------------------------
        self.tab_widget = QTabWidget() 
        
        # 2.1 TARAMA SONUÇLARI SEKME İÇERİĞİ (LİSTE GÖRÜNÜMÜ)
        scan_results_page = QWidget()
        results_layout = QVBoxLayout(scan_results_page)
        self.found_label = QLabel()
        
        # QListWidget
        self.results_list = QListWidget() 
        self.results_list.setViewMode(QListWidget.ListMode)      
        self.results_list.setResizeMode(QListWidget.Adjust)
        self.results_list.setIconSize(QSize(80, 80))             
        self.results_list.setSelectionMode(QAbstractItemView.ExtendedSelection) 
        self.results_list.setSpacing(5)                          
        
        # Scan/Rescan butonu ve Delete butonu için yatay düzen
        action_buttons_layout = QHBoxLayout()
        action_buttons_layout.addStretch(1) # Butonları sağa yaslamak için

        # 1. Scan/Rescan Butonu 
        self.start_button = QPushButton() 
        self.start_button.setMinimumHeight(35)
        self.start_button.setSizePolicy(QSizePolicy.Minimum, QSizePolicy.Fixed)
        
        # 2. Sahte Çöpe Gönder butonu 
        self.delete_button = QPushButton()
        self.delete_button.setEnabled(False)
        self.delete_button.setMinimumHeight(35)
        self.delete_button.setSizePolicy(QSizePolicy.Minimum, QSizePolicy.Fixed)
        self.delete_button.setStyleSheet("background-color: #f44336; color: white; font-weight: bold; min-height: 35px;")

        action_buttons_layout.addWidget(self.start_button)
        action_buttons_layout.addWidget(self.delete_button)

        results_layout.addWidget(self.found_label)
        results_layout.addWidget(self.results_list) 
        results_layout.addLayout(action_buttons_layout) # Yatay düzeni ekle
        
        # 2.2 FAKE TRASH SEKME İÇERİĞİ (Aynı kaldı)
        fake_trash_page = QWidget()
        trash_layout = QVBoxLayout(fake_trash_page)
        self.trash_table = QTableWidget()
        self.trash_table.setColumnCount(4)
        # Başlıklar _update_gui_texts'te ayarlanacak
        self.trash_table.setHorizontalHeaderLabels(["", get_text("trash_col_file"), get_text("trash_col_original_path"), get_text("trash_col_deletion_date")])
        self.trash_table.horizontalHeader().setSectionResizeMode(0, QHeaderView.ResizeToContents)
        self.trash_table.horizontalHeader().setSectionResizeMode(2, QHeaderView.Stretch)
        self.trash_table.horizontalHeader().setSectionResizeMode(3, QHeaderView.ResizeToContents)
        self.trash_table.setSelectionBehavior(QAbstractItemView.SelectRows)
        self.trash_table.setEditTriggers(QAbstractItemView.NoEditTriggers)
        
        select_buttons_layout = QHBoxLayout()
        self.select_all_trash_button = QPushButton()
        self.unselect_all_trash_button = QPushButton()
        self.select_all_trash_button.setStyleSheet("background-color: #4CAF50; color: white; font-weight: bold; height: 30px;")
        self.unselect_all_trash_button.setStyleSheet("background-color: #2196F3; color: white; font-weight: bold; height: 30px;")
        select_buttons_layout.addWidget(self.select_all_trash_button)
        select_buttons_layout.addWidget(self.unselect_all_trash_button)

        trash_buttons_layout = QHBoxLayout()
        self.restore_button = QPushButton()
        self.purge_button = QPushButton()
        self.restore_button.setEnabled(False)
        self.purge_button.setEnabled(False)
        self.restore_button.setStyleSheet("background-color: #3f51b5; color: white; font-weight: bold; height: 30px;") 
        self.purge_button.setStyleSheet("background-color: #e53935; color: white; font-weight: bold; height: 30px;") 
        
        trash_buttons_layout.addWidget(self.restore_button)
        trash_buttons_layout.addWidget(self.purge_button)
        
        trash_layout.addLayout(select_buttons_layout)
        trash_layout.addWidget(self.trash_table)
        trash_layout.addLayout(trash_buttons_layout)
        
        self.tab_widget.addTab(scan_results_page, get_text("tab_scan")) 
        self.tab_widget.addTab(fake_trash_page, get_text("tab_trash"))
        
        # -----------------------------------------------------
        # 3. DURUM VE İLERLEME ÇUBUĞU (Ortak Alt Alan)
        # -----------------------------------------------------
        status_layout_widget = QWidget()
        status_layout = QVBoxLayout(status_layout_widget)
        status_layout.setContentsMargins(5, 0, 5, 5)
        
        status_label_layout = QHBoxLayout()
        self.status_label = QLabel()
        self.status_label.setMinimumWidth(300)

        status_label_layout.addWidget(self.status_label, 1)

        progress_bar_layout = QHBoxLayout()
        self.progress_bar = QProgressBar()
        self.progress_bar.setRange(0, 100)

        progress_bar_layout.addWidget(self.progress_bar)

        status_layout.addLayout(status_label_layout)
        status_layout.addLayout(progress_bar_layout)
        
        # -----------------------------------------------------
        # 4. ANA DÜZEN
        # -----------------------------------------------------
        main_v_layout = QVBoxLayout(self.central_widget)
        main_v_layout.addWidget(top_bar_widget) 
        main_v_layout.addWidget(self.tab_widget, 1) 
        main_v_layout.addWidget(status_layout_widget) 


    def _connect_signals(self):
        self.start_button.clicked.connect(self._start_scan)
        self.delete_button.clicked.connect(self._delete_files_to_fake_trash) 
        self.language_button.clicked.connect(self._show_language_menu) # DİL BAĞLANTISI
        self.about_button.clicked.connect(self._show_about)
        self.results_list.itemDoubleClicked.connect(self._handle_list_double_click) 
        
        # Fake Trash Sekmesi
        self.tab_widget.currentChanged.connect(self._handle_tab_change)
        self.restore_button.clicked.connect(self._restore_selected_files)
        self.purge_button.clicked.connect(self._purge_selected_files)
        self.select_all_trash_button.clicked.connect(self._select_all_trash_files) 
        self.unselect_all_trash_button.clicked.connect(self._unselect_all_trash_files) 
        
        self.trash_table.cellDoubleClicked.connect(self._handle_trash_double_click)

    @Slot(int)
    def _handle_tab_change(self, index):
        """Sekme değiştiğinde Fake Trash sekmesini günceller."""
        if self.tab_widget.tabText(index) == get_text("tab_trash"):
            self.update_trash_tab()

    @Slot(QListWidgetItem)
    def _handle_list_double_click(self, item):
        """Tarama Sonuçları listesinde çift tıklama (Klasörü açar)."""
        try:
            full_path = item.data(Qt.UserRole)
            if not full_path: return
            
            folder_path = os.path.dirname(full_path).rstrip(os.path.sep)
            
            path_to_open = folder_path 
            self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_opening_folder")}: {folder_path}')
            
            if open_path_in_os(path_to_open):
                self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_ready")}')
            else:
                self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_error_open")}: \'{os.path.basename(path_to_open)}\'')
                
        except Exception as e:
            self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_error_general")}: {get_text("status_error_double_click")}. ({e})')
            
    @Slot(int, int)
    def _handle_trash_double_click(self, row, column):
        """Fake Trash tablosunda çift tıklama (Orijinal Klasör Yolu açılır)."""
        try:
            if column == 2: # Orijinal Yol sütunu
                original_path_item = self.trash_table.item(row, 2)
                if not original_path_item: return

                original_path = original_path_item.text()
                
                folder_path_to_open = os.path.dirname(original_path).rstrip(os.path.sep)

                if open_path_in_os(folder_path_to_open):
                    self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_ready")}')
                else:
                    self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_error_open")}: \'{folder_path_to_open}\'')
                    
        except Exception as e:
            self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_error_general")}: {get_text("status_error_trash_double_click")}. ({e})')


    @Slot()
    def _show_about(self):
        about_dialog = QMessageBox(self)
        about_dialog.setWindowTitle(get_text("about"))

        if self.icon_path:
            # QPixmap'i karanlık temaya uygun olarak yeniden boyutlandır
            pixmap = QPixmap(self.icon_path)
            about_dialog.setIconPixmap(pixmap.scaled(64, 64, Qt.KeepAspectRatio, Qt.SmoothTransformation))
        else:
            about_dialog.setIcon(QMessageBox.Information)

        # HTML içeriği QSS'ten bağımsız olarak açık renkte kalabilir, koyu arka plana uyum için font rengi beyaz/açık gri yapıldı.
        about_text = f"""
        <html>
        <head>
            <style>
                body {{ color: #cccccc; background-color: #2b2b2b; }}
                h3 {{ color: #0078D4; }}
                ul {{ list-style-type: none; padding-left: 0; }}
                li {{ margin-bottom: 5px; }}
                a {{ color: #3cb5ff; }}
            </style>
        </head>
        <body>
        <h3>{get_text("about_title")}</h3>
        <ul>
            <li><b>{get_text("about_version")}:</b> 1.0.1</li>
            <li><b>{get_text("about_license")}:</b> GPLv3</li>
            <li><b>{get_text("about_lang")}:</b> Python3 / PyQt5</li>
            <li><b>{get_text("about_dev")}:</b> A. Serhat KILIÇOĞLU (shampuan)</li>
            <li><b>Github:</b> <a href="https://www.github.com/shampuan">www.github.com/shampuan</a></li>
        </ul>
        <hr>
        <p>{get_text("about_purpose")}</p>
        <p><b>{get_text("about_warranty")}</b></p>
        <p>{get_text("about_copyright")} &copy; 2025 A. Serhat KILIÇOĞLU</p>
        </body>
        </html>
        """

        about_dialog.setText(about_text)
        about_dialog.exec()

    @Slot()
    def _start_scan(self):
        
        if self.worker_thread and self.worker_thread.isRunning():
            self.worker_thread.stop()
            self.worker_thread.wait()
            self.start_button.setText(get_text("start_scan"))
            self.start_button.setStyleSheet("background-color: #4CAF50; color: white; font-weight: bold; min-height: 35px;")
            self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_canceled")}')
            return

        # Minimalist sürümde ayarlar sabit: MD5 + Boyut eşleşmesi. Sadece Görsel dosyalar.
        match_options = {
            "content": True, 
            "size": True,    
            "name": False,
            "extension": False,
        }

        # Gizli/Sistem dosyalarını yoksayma varsayılan olarak TRUE
        ignore_options = {
            "ignore_zero_byte": True,
            "ignore_system_hidden": True, # Hardcode edildi: Gizli/Sistem dosyalarını yoksay.
        }
        
        filter_options = {
            "all": False, "audio": False, "video": False,
            "image": True, 
            "text": False, "office": False, "pdf": False, "archive": False,
            "custom": False, "custom_extensions": ""
        }

        options = {"match": match_options, "ignore": ignore_options, "filter": filter_options}

        target_dir = self.dir_input.text().strip()
        if not target_dir or not os.path.exists(target_dir):
            self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_error_dir")}')
            return
            
        target_dirs = [target_dir]

        self.results_list.clear() 
        self.progress_bar.setValue(0)
        self.delete_button.setEnabled(False)
        self.start_button.setText(get_text("cancel_scan"))
        self.start_button.setStyleSheet("background-color: #FFA500; color: white; font-weight: bold; min-height: 35px;")

        self.worker_thread = WorkerThread(target_dirs, options)
        self.worker_thread.progress_updated.connect(self._update_progress)
        self.worker_thread.status_message.connect(self._update_status)
        self.worker_thread.scan_finished.connect(self._display_results)
        self.worker_thread.finished.connect(self._scan_finished_cleanup)
        self.worker_thread.start()

    @Slot(int)
    def _update_progress(self, value):
        self.progress_bar.setValue(value)

    @Slot(str)
    def _update_status(self, message):
        # Durum mesajları worker thread'den geldiği için sadece mesajı yazdırırız.
        # get_text("status_prefix") eklenir.
        self.status_label.setText(f'{get_text("status_prefix")}: {message.split(":", 1)[1].strip() if ":" in message else message}')

    @Slot()
    def _scan_finished_cleanup(self):
        self.start_button.setText(get_text("rescan"))
        self.start_button.setStyleSheet("background-color: #4CAF50; color: white; font-weight: bold; min-height: 35px;")

    @Slot(list)
    def _display_results(self, duplicate_groups):
        
        self.results_list.clear()
        self.duplicate_data = duplicate_groups 
        
        total_files_found = sum(len(group['files']) for group in duplicate_groups)
        total_duplicates = total_files_found - len(duplicate_groups)

        self.found_label.setText(get_text("found_duplicates").format(len(duplicate_groups), total_duplicates))
        
        # BELLEK TÜKETİMİNİ ENGELLEMEK İÇİN LİMİT (2000 dosya/öğe)
        MAX_DISPLAY_ITEMS = 2000
        items_displayed_count = 0
        
        for group_index, group in enumerate(duplicate_groups):
            group_color = self.GROUP_COLORS[group_index % len(self.GROUP_COLORS)]

            for file_index, file_path in enumerate(group["files"]):
                
                if items_displayed_count >= MAX_DISPLAY_ITEMS:
                    break # Sınır aşıldı, iç döngüyü kır.
                
                # 1. Gerekli verileri al
                file_name = os.path.basename(file_path)
                # folder_path = os.path.dirname(file_path) 
                
                # 2. Thumbnail ve İkon oluştur
                thumbnail = self._create_thumbnail(file_path, size=80) 
                list_item = QListWidgetItem()
                list_item.setIcon(QIcon(thumbnail))
                
                # Açıklama metni (Dosya adı, boyut, yol ve HASH)
                item_text = f"{file_name}\n({group['size']})\n{os.path.dirname(file_path)}\nHash: {group['hash']}"
                list_item.setText(item_text)
                list_item.setToolTip(f"{file_name}\n{os.path.dirname(file_path)}\nHash: {group['hash']}")
                
                # 3. Öğeye özel verileri sakla 
                list_item.setData(Qt.UserRole, file_path)        # Tam yolu sakla
                list_item.setData(Qt.UserRole + 1, group["size_bytes"]) # Boyut (bytes)
                list_item.setData(Qt.UserRole + 2, group["hash"])       # Hash
                
                # 4. Arka plan rengini ayarla
                list_item.setBackground(QBrush(group_color))

                # Tüm öğeler seçilebilir ve işaretlenebilir olmalı
                list_item.setFlags(list_item.flags() | Qt.ItemIsUserCheckable | Qt.ItemIsSelectable)
                
                # 5. Checkbox durumunu ayarla: Tümü işaretsiz 
                list_item.setCheckState(Qt.CheckState.Unchecked) 
                

                self.results_list.addItem(list_item)
                items_displayed_count += 1
            
            if items_displayed_count >= MAX_DISPLAY_ITEMS:
                break # Sınır aşıldı, dış döngüyü kır.

        # UYARI MESAJI
        if items_displayed_count < total_files_found:
            QMessageBox.warning(
                self, 
                get_text("warning_title"), 
                get_text("warning_too_many_results").format(MAX_DISPLAY_ITEMS, total_files_found)
            )

        is_any_file = self.results_list.count() > 0
        self.delete_button.setEnabled(is_any_file)
        self.tab_widget.setCurrentIndex(0) 

    def _remove_deleted_rows(self, deleted_files_paths):
        
        deleted_set = set(deleted_files_paths)
        
        for i in range(self.results_list.count() - 1, -1, -1):
            item = self.results_list.item(i)
            full_path = item.data(Qt.UserRole)
            
            if full_path in deleted_set:
                self.results_list.takeItem(i)

        if self.results_list.count() == 0:
            self.delete_button.setEnabled(False)
            
    # <<< FAKE TRASH KULLANIMI (Aynı Kaldı) >>>
    @Slot()
    def _delete_files_to_fake_trash(self):
        """Seçilen dosyaları Sahte Çöp Kutusu'na taşır."""
        selected_files = []
        
        # QListWidget'taki işaretli öğeleri al
        for i in range(self.results_list.count()):
            item = self.results_list.item(i)
            # Yalnızca işaretli ve kullanıcı tarafından işaretlenebilir olanlar dikkate alınır
            if (item.flags() & Qt.ItemIsUserCheckable) and item.checkState() == Qt.CheckState.Checked:
                full_path = item.data(Qt.UserRole)
                size_bytes = item.data(Qt.UserRole + 1)
                selected_files.append({"path": full_path, "size_bytes": size_bytes})

        if not selected_files:
            self.status_label.setText(f'{get_text("status_prefix")}: {get_text("trash_error_select")}')
            return

        reply = QMessageBox.question(
            self,
            get_text("delete_confirm_title"),
            get_text("delete_confirm_text").format(len(selected_files)),
            QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No,
            QMessageBox.StandardButton.No
        )

        if reply == QMessageBox.StandardButton.No:
            self.status_label.setText(f'{get_text("status_prefix")}: {get_text("trash_canceled")}')
            return

        moved_count = 0
        error_count = 0
        moved_paths = []

        self.status_label.setText(f'{get_text("status_prefix")}: {get_text("delete_selected")}')

        for file_data in selected_files:
            if self.trash_manager.move_to_trash(file_data["path"], file_data["size_bytes"]):
                moved_count += 1
                moved_paths.append(file_data["path"])
            else:
                error_count += 1

        self._remove_deleted_rows(moved_paths)
        self.update_trash_tab() 

        if error_count == 0:
            final_message = get_text("trash_success").format(moved_count)
            QMessageBox.information(self, get_text("delete_confirm_title"), final_message)
        else:
            final_message = get_text("trash_error").format(moved_count, error_count)
            QMessageBox.warning(self, get_text("delete_confirm_title"), final_message)

        self.status_label.setText(f'{get_text("status_prefix")}: {final_message}')
        
    # <<< FAKE TRASH SEKMESİ YÖNETİMİ (Aynı Kaldı) >>>
    @Slot()
    def update_trash_tab(self):
        """Fake Trash sekmesindeki tabloyu günceller (DİSK BAZLI)."""
        all_trash_data = []

        target_dirs = [self.dir_input.text()] # Tek dizinden al
        target_dirs.append(os.path.expanduser('~')) 

        known_mount_points = set()
        for d in target_dirs:
            try:
                if d and os.path.exists(d):
                    known_mount_points.add(get_mount_point(d))
            except Exception as e:
                continue

        for mount_point in known_mount_points:
            # Yeni Fake Trash yolu
            trash_dir = os.path.join(mount_point, '.Photoagent-Trash-1000') 
            metadata_path = os.path.join(trash_dir, 'trashdata.json')

            if os.path.exists(metadata_path):
                disk_trash_data = self.trash_manager._load_metadata(metadata_path)
                
                for item in disk_trash_data:
                    if "trash_dir" not in item:
                         item["trash_dir"] = trash_dir
                    all_trash_data.append(item)

        trash_data = all_trash_data
        self.trash_table.setRowCount(0)
        row_count = 0

        for item in trash_data:
            self.trash_table.insertRow(row_count)
            
            # 0. Sütun: CheckBox
            check_item = QTableWidgetItem()
            check_item.setFlags(Qt.ItemIsUserCheckable | Qt.ItemIsEnabled)
            check_item.setCheckState(Qt.CheckState.Unchecked)
            self.trash_table.setItem(row_count, 0, check_item)
            
            # 1. Çöp Dosya Adı
            trash_filename = item.get("trash_filename", "")
            trash_dir = item.get("trash_dir", "") 
            trash_name_item = QTableWidgetItem(trash_filename)

            # İKON EKLEME
            trash_file_path = os.path.join(trash_dir, trash_filename)
            file_icon = self._get_file_icon(trash_file_path)
            trash_name_item.setIcon(file_icon)

            self.trash_table.setItem(row_count, 1, trash_name_item)
            
            # 2. Orijinal Yolu
            original_path = item.get("original_path", "")
            original_path_item = QTableWidgetItem(original_path)
            
            original_path_item.setData(Qt.UserRole, trash_filename)
            original_path_item.setData(Qt.UserRole + 1, trash_dir) 

            self.trash_table.setItem(row_count, 2, original_path_item)
            
            # 3. Silinme Tarihi
            date_item = QTableWidgetItem(item.get("deletion_date", ""))
            self.trash_table.setItem(row_count, 3, date_item)
            
            row_count += 1
            
        self.trash_table.setRowCount(row_count)
        is_any_file = row_count > 0
        self.restore_button.setEnabled(is_any_file)
        self.purge_button.setEnabled(is_any_file)
        
    @Slot()
    def _select_all_trash_files(self):
        """Çöp tablosundaki tüm dosyaları işaretler."""
        for row in range(self.trash_table.rowCount()):
            item = self.trash_table.item(row, 0)
            if item:
                item.setCheckState(Qt.CheckState.Checked)

    @Slot()
    def _unselect_all_trash_files(self):
        """Çöp tablosundaki tüm dosyaların işaretini kaldırır."""
        for row in range(self.trash_table.rowCount()):
            item = self.trash_table.item(row, 0)
            if item:
                item.setCheckState(Qt.CheckState.Unchecked)


    @Slot()
    def _get_selected_trash_items(self):
        """Çöp tablosunda seçilen dosyaların listesini döndürür."""
        selected_items = []
        for row in range(self.trash_table.rowCount()):
            check_item = self.trash_table.item(row, 0)
            if check_item and check_item.checkState() == Qt.CheckState.Checked:
                trash_filename = self.trash_table.item(row, 1).text() 
                original_path_item = self.trash_table.item(row, 2)
                
                if original_path_item:
                    original_path = original_path_item.text()
                    trash_dir = original_path_item.data(Qt.UserRole + 1)
                    
                    selected_items.append({
                        "trash_filename": trash_filename, 
                        "original_path": original_path,
                        "trash_dir": trash_dir 
                    })
        return selected_items

    @Slot()
    def _restore_selected_files(self):
        """Seçili dosyaları orijinal konumuna geri yükler."""
        selected_files = self._get_selected_trash_items()
        if not selected_files:
            QMessageBox.warning(self, get_text("restore_confirm_title"), get_text("restore_error_select"))
            return

        reply = QMessageBox.question(
            self,
            get_text("restore_confirm_title"),
            get_text("restore_confirm_text").format(len(selected_files)),
            QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No,
            QMessageBox.StandardButton.No
        )

        if reply == QMessageBox.StandardButton.No: return

        restored_count = 0
        error_count = 0

        self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_restoring_files")}')
        for file_data in selected_files:
            if self.trash_manager.restore_file(file_data["trash_filename"], file_data["original_path"], file_data["trash_dir"]):
                restored_count += 1
            else:
                error_count += 1

        self.update_trash_tab()
        
        if error_count == 0:
            final_message = get_text("restore_success").format(restored_count)
            QMessageBox.information(self, get_text("restore_confirm_title"), final_message)
        else:
            final_message = get_text("restore_error").format(restored_count, error_count)
            QMessageBox.warning(self, get_text("restore_confirm_title"), final_message)

        self.status_label.setText(f'{get_text("status_prefix")}: {final_message}')


    @Slot()
    def _purge_selected_files(self):
        """Seçili dosyaları diskten kalıcı olarak siler."""
        selected_files = self._get_selected_trash_items()
        if not selected_files:
            QMessageBox.warning(self, get_text("purge_confirm_title"), get_text("purge_error_select"))
            return

        reply = QMessageBox.question(
            self,
            get_text("purge_confirm_title"),
            get_text("purge_confirm_text").format(len(selected_files)),
            QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No,
            QMessageBox.StandardButton.No
        )

        if reply == QMessageBox.StandardButton.No: return

        purged_count = 0
        error_count = 0

        self.status_label.setText(f'{get_text("status_prefix")}: {get_text("status_purging_files")}')
        for file_data in selected_files:
            if self.trash_manager.purge_file(file_data["trash_filename"], file_data["original_path"], file_data["trash_dir"]):
                purged_count += 1
            else:
                error_count += 1

        self.update_trash_tab()

        if error_count == 0:
            final_message = get_text("purge_success").format(purged_count)
            QMessageBox.information(self, get_text("purge_confirm_title"), final_message)
        else:
            final_message = get_text("purge_error").format(purged_count, error_count)
            QMessageBox.warning(self, get_text("purge_confirm_title"), final_message)

        self.status_label.setText(f'{get_text("status_prefix")}: {final_message}')

#----Riders on the Storm----        
#Riders on the storm
#Riders on the storm
#Into this house, we're born
#Into this world, we're thrown
#Like a dog without a bone
#An actor out on loan
#Riders on the storm
#There's a killer on the road
#His brain is squirmin' like a toad
#Take a long holiday
#Let your children play
#If you give this man a ride
#Sweet family will die
#Killer on the road, yeah
#Girl, you gotta love your man
#Girl, you gotta love your man
#Take him by the hand
#Make him understand
#The world on you depends
#Our life will never end
#Gotta love your man, yeah
#Riders on the storm
#Riders on the storm
#Into this house, we're born
#Into this world, we're thrown
#Like a dog without a bone
#An actor out on loan
#Riders on the storm
#Riders on the storm
#Riders on the storm
#Riders on the storm
#Riders on the storm
#-----The Doors-----

# 4. UYGULAMA BAŞLANGICI

if __name__ == "__main__":
    app = QApplication(sys.argv)
    
    # <<< KRİTİK: KARANLIK TEMA UYGULAMASI >>>
    app.setStyleSheet(DARK_THEME_QSS)
    # <<< UYGULAMA SONU >>>

    # Dil dosyalarını yükle (Şimdi INI'den okuyacak - YZ sağolsun)
    load_language_files()
    
    window = PhotoFinderApp()
    window.show()
    sys.exit(app.exec_()) # PyQt5'te exec() yerine exec_() kullanılır.
