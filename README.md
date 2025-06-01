import tkinter as tk
from tkinter import ttk
from itertools import cycle
import tkinter.messagebox as messagebox
import json
import os
import os.path
import win32print  # Yazıcı işlemleri için (Windows)
import win32ui
import win32con
from PIL import Image, ImageDraw, ImageFont, ImageWin
from reportlab.lib.units import mm
from reportlab.pdfgen import canvas
from datetime import datetime

def print_image_directly(file_path, printer_name=None):
    if printer_name is None:
        printer_name = win32print.GetDefaultPrinter()
    hDC = win32ui.CreateDC()
    hDC.CreatePrinterDC(printer_name)
    hDC.StartDoc("Barkod Baskısı")
    hDC.StartPage()
    bmp = Image.open(file_path)
    dib = ImageWin.Dib(bmp)
    printer_size = (hDC.GetDeviceCaps(win32con.PHYSICALWIDTH),
                    hDC.GetDeviceCaps(win32con.PHYSICALHEIGHT))
    x = int((printer_size[0] - bmp.size[0]) / 2)
    y = int((printer_size[1] - bmp.size[1]) / 2)
    dib.draw(hDC.GetHandleOutput(), (x, y, x + bmp.size[0], y + bmp.size[1]))
    hDC.EndPage()
    hDC.EndDoc()
    hDC.DeleteDC()

class KeseciSiraSistemi:
    def __init__(self, master):
        self.master = master
        master.title("Hamam Keseci Sıra Sistemi")
        master.geometry("800x700")

        self.data_dir = "HamamProgram"
        if not os.path.exists(self.data_dir):
            os.makedirs(self.data_dir)

        # Log dosyası yolu ve log içeriği yüklemesi
        self.log_file = os.path.join(self.data_dir, "log.txt")
        self.load_log()

        self.next_id = 1
        self.load_data()

        self.son_keseci = None
        self.barkod_sayisi = 0
        self.keseci_buttons = {}
        self.edit_window = None
        self.edit_entries = {}

        self.update_cycle()

        self.notebook = ttk.Notebook(master)
        self.notebook.pack(fill="both", expand=True)

        self.tab_main = tk.Frame(self.notebook)
        self.tab_print = tk.Frame(self.notebook)
        self.notebook.add(self.tab_main, text="Ana Ekran")
        self.notebook.add(self.tab_print, text="Keseciler Yazdırma")

        # ANA EKRAN
        self.top_frame = tk.Frame(self.tab_main)
        self.top_frame.pack(side=tk.TOP, fill=tk.X)
        self.edit_button = tk.Button(self.top_frame, text="Düzenle", font=("Arial", 14), command=self.open_edit_window)
        self.edit_button.pack(side=tk.LEFT, padx=10, pady=10)
        self.print_button = tk.Button(self.top_frame, text="Yazdır", font=("Arial", 14), command=self.open_print_dialog)
        self.print_button.pack(side=tk.LEFT, padx=10, pady=10)

        self.label = tk.Label(self.tab_main, text="Numara Bekleniyor...", font=("Arial", 24))
        self.label.pack(pady=10)

        self.button_frame = tk.Frame(self.tab_main)
        self.button_frame.pack()
        self.update_keseci_buttons()

        self.yeni_musteri_btn = tk.Button(self.tab_main, text="Yeni Müşteri", font=("Arial", 20), command=self.siradaki_keseci)
        self.yeni_musteri_btn.pack(pady=10)

        self.log_text = tk.Text(self.tab_main, height=10)
        self.log_text.pack(pady=10)
        self.log_text.insert(tk.END, self.log_contents)

        self.save_log_btn = tk.Button(self.tab_main, text="Log Kaydet", font=("Arial", 16), command=self.save_log)
        self.save_log_btn.pack(pady=5)

        self.bottom_frame = tk.Frame(self.tab_main)
        self.bottom_frame.pack(side=tk.BOTTOM, fill=tk.X, padx=10, pady=5)
        self.saya_label = tk.Label(self.bottom_frame, text=f"Barkod Sayısı: {self.barkod_sayisi}", font=("Arial", 16))
        self.saya_label.pack(side=tk.LEFT, padx=5)
        self.reset_button = tk.Button(self.bottom_frame, text="Sayaç Sıfırla", font=("Arial", 16), command=self.reset_sayac)
        self.reset_button.pack(side=tk.LEFT, padx=5)

        master.bind("<F1>", lambda event: self.siradaki_keseci())
        master.bind("<F2>", lambda event: self.reset_sayac())

        self.create_print_tab()

    def load_log(self):
        if os.path.exists(self.log_file):
            with open(self.log_file, "r", encoding="utf-8") as file:
                self.log_contents = file.read()
        else:
            self.log_contents = ""

    def add_log(self, message):
        timestamp = datetime.now().strftime("[%Y-%m-%d %H:%M:%S] ")
        full_message = timestamp + message + "\n"
        self.log_text.insert(tk.END, full_message)
        self.log_text.see(tk.END)
        with open(self.log_file, "a", encoding="utf-8") as file:
            file.write(full_message)

    def load_data(self):
        file_path = os.path.join(self.data_dir, "keseciler.json")
        if os.path.exists(file_path):
            try:
                with open(file_path, "r", encoding="utf-8") as file:
                    data = json.load(file)
                if isinstance(data, list) and data:
                    self.keseciler = data
                    self.next_id = max(item.get("id", 0) for item in self.keseciler) + 1
                else:
                    self.create_default_keseciler()
            except Exception as e:
                messagebox.showerror("Hata", f"Veriler yüklenirken hata oluştu: {e}")
                self.create_default_keseciler()
        else:
            self.create_default_keseciler()

    def create_default_keseciler(self):
        self.keseciler = []
        for i in range(1, 14):
            self.keseciler.append({
                "id": self.next_id,
                "code": str(i),
                "name": f"Keseci {i}",
                "active": True
            })
            self.next_id += 1

    def save_data(self):
        file_path = os.path.join(self.data_dir, "keseciler.json")
        try:
            with open(file_path, "w", encoding="utf-8") as file:
                json.dump(self.keseciler, file, ensure_ascii=False, indent=4)
        except Exception as e:
            messagebox.showerror("Hata", f"Veriler kaydedilemedi: {e}")

    def update_cycle(self):
        active = [k for k in self.keseciler if k.get("active", True)]
        if self.son_keseci is not None and self.son_keseci in active:
            idx = active.index(self.son_keseci)
            rotated = active[idx+1:] + active[:idx+1]
            self.keseci_cycle = cycle(rotated)
        else:
            self.keseci_cycle = cycle(active)

    def update_keseci_buttons(self):
        for widget in self.button_frame.winfo_children():
            widget.destroy()
        self.keseci_buttons = {}
        for idx, keseci in enumerate(self.keseciler):
            row = idx // 4
            col = idx % 4
            code = keseci.get("code", "N/A")
            name = keseci.get("name", "N/A")
            active = keseci.get("active", True)
            text = f"{code} - {name} ({'Açık' if active else 'Kapalı'})"
            color = "green" if active else "red"
            btn = tk.Button(self.button_frame, text=text, font=("Arial", 14), bg=color,
                            width=20, command=lambda kid=keseci["id"]: self.toggle_keseci(kid))
            btn.grid(row=row, column=col, padx=5, pady=5)
            self.keseci_buttons[keseci["id"]] = btn

    def toggle_keseci(self, keseci_id):
        for keseci in self.keseciler:
            if keseci.get("id") == keseci_id:
                if keseci.get("active", True):
                    if not messagebox.askyesno("Onay", f"{keseci.get('code', 'N/A')} - {keseci.get('name', 'N/A')} kesecisini kapatmak istediğinize emin misiniz?"):
                        return
                    keseci["active"] = False
                    self.add_log(f"Keseci {keseci.get('code', 'N/A')} - {keseci.get('name', 'N/A')} kapatıldı.")
                else:
                    keseci["active"] = True
                    self.add_log(f"Keseci {keseci.get('code', 'N/A')} - {keseci.get('name', 'N/A')} açıldı.")
                break
        self.update_keseci_buttons()
        self.update_cycle()
        self.save_data()
        self.create_print_tab()

    def siradaki_keseci(self):
        active_keseciler = [k for k in self.keseciler if k.get("active", True)]
        if active_keseciler:
            while True:
                keseci = next(self.keseci_cycle)
                if keseci.get("active", True):
                    self.son_keseci = keseci
                    self.label.config(text=f"{keseci.get('code', 'N/A')} - {keseci.get('name', 'N/A')}")
                    self.barkod_sayisi += 1
                    self.saya_label.config(text=f"Barkod Sayısı: {self.barkod_sayisi}")
                    self.add_log(f"Müşteri {self.barkod_sayisi}: {keseci.get('code', 'N/A')} - {keseci.get('name', 'N/A')}")
                    break
        else:
            self.label.config(text="Aktif keseci yok!")
            self.add_log("Müşteri çağrıldı ancak aktif keseci yok!")
            return

        # PDF etiket oluştur ve hemen yazdır (Acrobat Reader açılmadan)
        pdf_file = self.generate_pdf_label(self.son_keseci)
        if pdf_file:
            try:
                # Bu satır Acrobat Reader açmadan sessizce yazdırır.
                os.startfile(pdf_file, "print")
            except Exception as e:
                self.add_log(f"PDF yazdırma hatası: {e}")

    def reset_sayac(self):
        self.barkod_sayisi = 0
        self.saya_label.config(text=f"Barkod Sayısı: {self.barkod_sayisi}")
        self.add_log("Sayaç sıfırlandı.")

    def save_log(self):
        file_path = os.path.join(self.data_dir, "log.txt")
        try:
            with open(file_path, "w", encoding="utf-8") as file:
                file.write(self.log_text.get("1.0", tk.END))
            self.add_log("Log dosyası kaydedildi.")
        except Exception as e:
            messagebox.showerror("Hata", f"Log dosyası kaydedilemedi: {e}")

    # --- PDF Tabanlı Yazdırma Özelliği ---
    def generate_pdf_label(self, keseci):
        pdf_filename = os.path.join(self.data_dir, f"label_{keseci.get('code', 'n/a')}.pdf")
        page_width = 50 * mm
        page_height = 48 * mm
        c = canvas.Canvas(pdf_filename, pagesize=(page_width, page_height))
        center_x = page_width / 2
        center_y = page_height / 2
        # Font boyutunu 18 punto, dikeyde -8 point ayarlıyoruz.
        c.setFont("Helvetica-Bold", 18)
        text = f"{keseci.get('code', 'N/A')} - {keseci.get('name', 'N/A')}"
        c.drawCentredString(center_x, center_y - 8, text)
        c.showPage()
        c.save()
        return pdf_filename

    def print_pdf_label(self, keseci):
        pdf_file = self.generate_pdf_label(keseci)
        if pdf_file:
            try:
                os.startfile(pdf_file, "print")
            except Exception as e:
                self.add_log(f"PDF yazdırma hatası: {e}")

    def open_print_dialog(self):
        file_path = self.generate_print_image()
        if file_path is None:
            return
        printers = win32print.EnumPrinters(win32print.PRINTER_ENUM_LOCAL | win32print.PRINTER_ENUM_CONNECTIONS)
        dialog = tk.Toplevel(self.master)
        dialog.title("Yazıcı Seçimi")
        dialog.geometry("300x200")
        lb = tk.Listbox(dialog, font=("Arial", 12))
        for printer in printers:
            lb.insert(tk.END, printer[2])
        lb.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        def on_select():
            selection = lb.curselection()
            if not selection:
                messagebox.showwarning("Uyarı", "Lütfen bir yazıcı seçiniz.")
                return
            selected_printer = lb.get(selection[0])
            dialog.destroy()
            self.perform_print_image(file_path, selected_printer)
        select_button = tk.Button(dialog, text="Yazdır", font=("Arial", 12), command=on_select)
        select_button.pack(pady=5)

    def perform_print_image(self, file_path, printer_name, silent_print=False):
        try:
            cmd = f'mspaint /pt "{file_path}" "{printer_name}"'
            os.system(cmd)
            if not silent_print:
                messagebox.showinfo("Yazdır", "Yazdırma işlemi başlatıldı.")
        except Exception as e:
            if not silent_print:
                messagebox.showerror("Yazdır Hatası", str(e))
            else:
                print("Yazdır Hatası:", e)

    def print_specific_keseci(self, keseci):
        file_path = self.generate_print_image_for_keseci(keseci)
        if file_path:
            try:
                default_printer = win32print.GetDefaultPrinter()
                self.perform_print_image(file_path, default_printer, silent_print=True)
            except Exception as e:
                self.add_log(f"Yazdırma hatası (keseci yazdırma): {e}")

    def generate_print_image_for_keseci(self, keseci):
        """
        Kesecinin bilgilerini içeren imajı oluşturur.
        Hedef boyut: 50 mm x 48 mm'nin yarısı: 900x225 piksel.
        DPI: 200x200.
        """
        try:
            from PIL import Image, ImageDraw, ImageFont
        except ImportError:
            messagebox.showerror("Hata", "Pillow kütüphanesi yüklü değil! (pip install pillow)")
            return None
        width, height = 900, 225
        image = Image.new("RGB", (width, height), "white")
        draw = ImageDraw.Draw(image)
        try:
            font = ImageFont.truetype("arial.ttf", 120)
        except Exception:
            font = ImageFont.load_default()
        text = f"{keseci.get('code', 'N/A')} - {keseci.get('name', 'N/A')}"
        try:
            bbox = draw.multiline_textbbox((0, 0), text, font=font)
            text_w = bbox[2] - bbox[0]
            text_h = bbox[3] - bbox[1]
        except AttributeError:
            lines = text.split('\n')
            text_w = 0
            text_h = 0
            for line in lines:
                w, h = draw.textsize(line, font=font)
                text_w = max(text_w, w)
                text_h += h
        x = (width - text_w) / 2
        y = (height - text_h) / 2
        draw.multiline_text((x, y), text, fill="black", font=font, align="center")
        print_dir = os.path.join(self.data_dir, "PrintImages")
        if not os.path.exists(print_dir):
            os.makedirs(print_dir)
        filename = os.path.join(print_dir, f"print_{keseci.get('code', 'n/a')}.jpg")
        image.save(filename, "JPEG", dpi=(200,200))
        return filename

    def generate_barcode_label_image(self, keseci):
        """
        Kesecinin bilgilerine göre, fiziksel çıktı boyutu 35 mm x 30 mm (300 DPI, yaklaşık 413x354 piksel)
        olan etiket imajı oluşturur.
        Font boyutu 80, kenar boşluğu 15 piksel.
        """
        try:
            from PIL import Image, ImageDraw, ImageFont
        except ImportError:
            messagebox.showerror("Hata", "Pillow kütüphanesi yüklü değil! (pip install pillow)")
            return None
        dpi = 300
        mm_width = 35
        mm_height = 30
        width_px = int(mm_width / 25.4 * dpi)
        height_px = int(mm_height / 25.4 * dpi)
        image = Image.new("RGB", (width_px, height_px), "white")
        draw = ImageDraw.Draw(image)
        try:
            font = ImageFont.truetype("arial.ttf", 80)
        except Exception:
            font = ImageFont.load_default()
        text = f"{keseci.get('code', 'N/A')} - {keseci.get('name', 'N/A')}"
        margin = 15
        effective_width = width_px - 2 * margin
        effective_height = height_px - 2 * margin
        try:
            bbox = draw.multiline_textbbox((0, 0), text, font=font)
            text_w = bbox[2] - bbox[0]
            text_h = bbox[3] - bbox[1]
        except AttributeError:
            text_w, text_h = draw.textsize(text, font=font)
        x = margin + (effective_width - text_w) / 2
        y = margin + (effective_height - text_h) / 2
        draw.multiline_text((x, y), text, fill="black", font=font, align="center")
        print_dir = os.path.join(self.data_dir, "PrintImages")
        if not os.path.exists(print_dir):
            os.makedirs(print_dir)
        filename = os.path.join(print_dir, f"barcode_label_{keseci.get('code', 'n/a')}.jpg")
        image.save(filename, "JPEG", dpi=(dpi, dpi))
        return filename

    def print_barcode_label(self, keseci):
        if keseci is None:
            messagebox.showinfo("Bilgi", "Önce bir keseci seçmelisiniz!")
            return
        file_path = self.generate_barcode_label_image(keseci)
        if file_path:
            try:
                default_printer = win32print.GetDefaultPrinter()
                self.perform_print_image(file_path, default_printer, silent_print=True)
            except Exception as e:
                self.add_log(f"Barkod etiket yazdırma hatası: {e}")

    def create_print_tab(self):
        for widget in self.tab_print.winfo_children():
            widget.destroy()
        header = tk.Frame(self.tab_print)
        header.pack(fill=tk.X, pady=5)
        tk.Label(header, text="Kod", font=("Arial", 12), width=10).pack(side=tk.LEFT, padx=5)
        tk.Label(header, text="İsim", font=("Arial", 12), width=20).pack(side=tk.LEFT, padx=5)
        tk.Label(header, text="İşlem", font=("Arial", 12), width=10).pack(side=tk.LEFT, padx=5)
        for keseci in self.keseciler:
            row = tk.Frame(self.tab_print)
            row.pack(fill=tk.X, pady=2, padx=5)
            tk.Label(row, text=keseci.get("code", "N/A"), font=("Arial", 12), width=10).pack(side=tk.LEFT, padx=5)
            tk.Label(row, text=keseci.get("name", "N/A"), font=("Arial", 12), width=20).pack(side=tk.LEFT, padx=5)
            btn = tk.Button(row, text="Yazdır", font=("Arial", 12),
                            command=lambda k=keseci: self.print_specific_keseci(k))
            btn.pack(side=tk.LEFT, padx=5)

    def open_edit_window(self):
        if self.edit_window is not None:
            self.edit_window.lift()
            return
        self.edit_window = tk.Toplevel(self.master)
        self.edit_window.title("Keseci Düzenle")
        self.edit_window.geometry("400x400")
        self.edit_window.protocol("WM_DELETE_WINDOW", self.close_edit_window)
        container = tk.Frame(self.edit_window)
        container.pack(fill="both", expand=True, padx=10, pady=10)
        self.edit_canvas = tk.Canvas(container, height=300)
        self.edit_canvas.pack(side="left", fill="both", expand=True)
        scrollbar = tk.Scrollbar(container, orient="vertical", command=self.edit_canvas.yview)
        scrollbar.pack(side="right", fill="y")
        self.edit_canvas.configure(yscrollcommand=scrollbar.set)
        self.edit_frame = tk.Frame(self.edit_canvas)
        self.edit_canvas.create_window((0, 0), window=self.edit_frame, anchor="nw")
        self.edit_frame.bind("<Configure>", lambda e: self.edit_canvas.configure(scrollregion=self.edit_canvas.bbox("all")))
        header = tk.Frame(self.edit_frame)
        header.pack(fill=tk.X)
        tk.Label(header, text="Kod", width=10).pack(side=tk.LEFT, padx=5)
        tk.Label(header, text="İsim", width=20).pack(side=tk.LEFT, padx=5)
        tk.Label(header, text="").pack(side=tk.LEFT, padx=5)
        self.edit_entries = {}
        for keseci in self.keseciler:
            self.add_edit_row(keseci)
        btn_frame = tk.Frame(self.edit_window)
        btn_frame.pack(pady=10)
        add_btn = tk.Button(btn_frame, text="Yeni Keseci Ekle", command=self.add_new_edit_row)
        add_btn.pack(side=tk.LEFT, padx=5)
        save_btn = tk.Button(btn_frame, text="Kaydet", command=self.save_edit_changes)
        save_btn.pack(side=tk.LEFT, padx=5)
        scroll_down_btn = tk.Button(btn_frame, text="Aşağı Kaydır", command=self.scroll_edit_to_bottom)
        scroll_down_btn.pack(side=tk.LEFT, padx=5)
        scroll_up_btn = tk.Button(btn_frame, text="Yukarı Kaydır", command=self.scroll_edit_to_top)
        scroll_up_btn.pack(side=tk.LEFT, padx=5)
        close_btn = tk.Button(btn_frame, text="Kapat", command=self.close_edit_window)
        close_btn.pack(side=tk.LEFT, padx=5)

    def scroll_edit_to_bottom(self):
        self.edit_canvas.yview_moveto(1)

    def scroll_edit_to_top(self):
        self.edit_canvas.yview_moveto(0)

    def add_edit_row(self, keseci):
        row_frame = tk.Frame(self.edit_frame)
        row_frame.pack(fill=tk.X, pady=2)
        code_var = tk.StringVar(value=keseci.get("code", ""))
        name_var = tk.StringVar(value=keseci.get("name", ""))
        code_entry = tk.Entry(row_frame, textvariable=code_var, width=10)
        code_entry.pack(side=tk.LEFT, padx=5)
        name_entry = tk.Entry(row_frame, textvariable=name_var, width=20)
        name_entry.pack(side=tk.LEFT, padx=5)
        del_btn = tk.Button(row_frame, text="Sil", command=lambda kid=keseci["id"], rf=row_frame: self.delete_edit_row(kid, rf))
        del_btn.pack(side=tk.LEFT, padx=5)
        self.edit_entries[keseci["id"]] = {"code_entry": code_entry, "name_entry": name_entry, "frame": row_frame}

    def add_new_edit_row(self):
        new_keseci = {"id": self.next_id, "code": "", "name": "", "active": True}
        self.next_id += 1
        self.keseciler.append(new_keseci)
        self.add_edit_row(new_keseci)
        self.edit_canvas.configure(scrollregion=self.edit_canvas.bbox("all"))

    def delete_edit_row(self, keseci_id, row_frame):
        self.keseciler = [k for k in self.keseciler if k.get("id") != keseci_id]
        if keseci_id in self.edit_entries:
            del self.edit_entries[keseci_id]
        row_frame.destroy()
        self.edit_canvas.configure(scrollregion=self.edit_canvas.bbox("all"))
        self.save_data()
        self.create_print_tab()

    def save_edit_changes(self):
        for keseci in self.keseciler:
            eid = keseci.get("id")
            if eid in self.edit_entries:
                new_code = self.edit_entries[eid]["code_entry"].get().strip()
                new_name = self.edit_entries[eid]["name_entry"].get().strip()
                keseci["code"] = new_code if new_code else keseci.get("code", "")
                keseci["name"] = new_name if new_name else keseci.get("name", "")
        self.update_keseci_buttons()
        self.update_cycle()
        self.add_log("Keseci düzenlemeleri kaydedildi.")
        self.save_data()
        self.close_edit_window()
        self.create_print_tab()

    def close_edit_window(self):
        if self.edit_window:
            self.edit_window.destroy()
            self.edit_window = None

if __name__ == '__main__':
    root = tk.Tk()
    app = KeseciSiraSistemi(root)
    root.mainloop()
