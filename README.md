import psutil
import customtkinter as ctk
from tkinter import messagebox, ttk, filedialog
import threading
import time
import csv
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from matplotlib.animation import FuncAnimation
import traceback

# Настройка внешнего вида
ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("blue")

class ProcessManagerApp(ctk.CTk):
    def __init__(self):
        super().__init__()

        self.title("✨ Менеджер процессов PRO")
        self.geometry("1100x720")
        self.minsize(1000, 680)

        # Плавное появление
        self.fade_in()

        # Текущий выбранный PID для графика
        self.selected_pid = None
        self.anim = None
        self.figure = None
        self.ax_cpu = None
        self.ax_mem = None
        self.cpu_data = []
        self.mem_data = []
        self.time_data = []
        self.is_updating = False  # Флаг для защиты от перегрузки

        # Создание вкладок
        self.tabview = ctk.CTkTabview(self, corner_radius=10)
        self.tabview.pack(fill="both", expand=True, padx=20, pady=20)

        self.tab_processes = self.tabview.add("📋 Процессы")
        self.tab_graph = self.tabview.add("📈 График")
        self.tab_system = self.tabview.add("🖥️ Система")

        # === ВКЛАДКА: ПРОЦЕССЫ ===
        self.setup_processes_tab()

        # === ВКЛАДКА: ГРАФИК ===
        self.setup_graph_tab()

        # === ВКЛАДКА: СИСТЕМА ===
        self.setup_system_tab()

        # Загрузка процессов
        self.refresh_process_list()
        self.auto_refresh()  # каждые 5 сек

        # Обновление системной информации каждые 2 сек
        self.update_system_info()

        # Подпись разработчика
        dev_label = ctk.CTkLabel(
            self,
            text="👨‍💻 Разработчик: @ShadowDecrypt",
            font=("Arial", 12, "underline"),
            text_color="gray",
            cursor="hand2"
        )
        dev_label.pack(side="bottom", pady=5)
        dev_label.bind("<Button-1>", lambda e: self.open_developer_link())

    def open_developer_link(self):
        import webbrowser
        webbrowser.open("https://t.me/ShadowDecrypt")

    def fade_in(self):
        """Плавное появление окна"""
        self.attributes("-alpha", 0)
        for i in range(0, 101, 5):
            if not self.winfo_exists():
                break
            self.attributes("-alpha", i / 100)
            self.update()
            time.sleep(0.005)

    def setup_processes_tab(self):
        header = ctk.CTkLabel(self.tab_processes, text="🔍 Управление процессами", font=("Arial", 22, "bold"))
        header.pack(pady=15)

        # Поиск
        search_frame = ctk.CTkFrame(self.tab_processes, fg_color="transparent")
        search_frame.pack(fill="x", padx=20, pady=5)

        search_label = ctk.CTkLabel(search_frame, text="Поиск:", font=("Arial", 14))
        search_label.pack(side="left", padx=(0, 10))

        self.search_entry = ctk.CTkEntry(search_frame, placeholder_text="Введите имя процесса...", width=300)
        self.search_entry.pack(side="left", padx=5)
        self.search_entry.bind("<KeyRelease>", self.debounce_search)

        # Кнопки
        button_frame = ctk.CTkFrame(self.tab_processes, fg_color="transparent")
        button_frame.pack(pady=10)

        self.refresh_button = ctk.CTkButton(button_frame, text="🔄 Обновить", command=self.refresh_process_list, width=120)
        self.refresh_button.pack(side="left", padx=5)

        self.export_button = ctk.CTkButton(button_frame, text="💾 Экспорт CSV", command=self.export_to_csv, width=120, fg_color="green")
        self.export_button.pack(side="left", padx=5)

        self.kill_button = ctk.CTkButton(button_frame, text="❌ Завершить", command=self.kill_selected_process, width=120, fg_color="red")
        self.kill_button.pack(side="left", padx=5)

        # Таблица
        self.process_frame = ctk.CTkFrame(self.tab_processes)
        self.process_frame.pack(fill="both", expand=True, padx=20, pady=10)

        columns = ("pid", "name", "status", "memory")
        self.tree = ttk.Treeview(self.process_frame, columns=columns, show="headings", selectmode="browse")
        self.tree.heading("pid", text="PID")
        self.tree.heading("name", text="Имя процесса")
        self.tree.heading("status", text="Статус")
        self.tree.heading("memory", text="Память (МБ)")

        self.tree.column("pid", width=80, anchor="center")
        self.tree.column("name", width=350)
        self.tree.column("status", width=100, anchor="center")
        self.tree.column("memory", width=120, anchor="center")

        vsb = ttk.Scrollbar(self.process_frame, orient="vertical", command=self.tree.yview)
        hsb = ttk.Scrollbar(self.process_frame, orient="horizontal", command=self.tree.xview)
        self.tree.configure(yscrollcommand=vsb.set, xscrollcommand=hsb.set)

        self.tree.grid(row=0, column=0, sticky='nsew')
        vsb.grid(row=0, column=1, sticky='ns')
        hsb.grid(row=1, column=0, sticky='ew')

        self.process_frame.grid_rowconfigure(0, weight=1)
        self.process_frame.grid_columnconfigure(0, weight=1)

        # Информация о процессе
        self.info_label = ctk.CTkLabel(self.tab_processes, text="📋 Выберите процесс для просмотра деталей", font=("Arial", 14), justify="left", wraplength=800)
        self.info_label.pack(pady=15, padx=20)

        self.tree.bind("<<TreeviewSelect>>", self.on_process_select)

    def setup_graph_tab(self):
        graph_label = ctk.CTkLabel(self.tab_graph, text="📊 График использования CPU и памяти (для выбранного процесса)", font=("Arial", 16))
        graph_label.pack(pady=15)

        # Создаем график
        self.figure, (self.ax_cpu, self.ax_mem) = plt.subplots(2, 1, figsize=(10, 6), facecolor='#2b2b2b')
        self.figure.patch.set_facecolor('#2b2b2b')

        for ax in [self.ax_cpu, self.ax_mem]:
            ax.set_facecolor('#333333')
            ax.tick_params(colors='white', which='both')
            ax.xaxis.label.set_color('white')
            ax.yaxis.label.set_color('white')
            ax.title.set_color('white')
            ax.spines['bottom'].set_color('white')
            ax.spines['top'].set_color('white')
            ax.spines['left'].set_color('white')
            ax.spines['right'].set_color('white')

        self.ax_cpu.set_title("CPU (%)")
        self.ax_mem.set_title("Память (МБ)")
        self.ax_cpu.set_ylim(0, 100)
        self.ax_mem.set_ylim(0, 500)

        self.canvas = FigureCanvasTkAgg(self.figure, self.tab_graph)
        self.canvas.get_tk_widget().pack(fill="both", expand=True, padx=20, pady=10)

        hint_label = ctk.CTkLabel(self.tab_graph, text="👉 Выберите процесс во вкладке 'Процессы', чтобы начать мониторинг", font=("Arial", 12, "italic"))
        hint_label.pack(pady=5)

    def setup_system_tab(self):
        header = ctk.CTkLabel(self.tab_system, text="🖥️ Системная информация", font=("Arial", 20, "bold"))
        header.pack(pady=20)

        self.system_info_label = ctk.CTkLabel(
            self.tab_system,
            text="Загрузка...",
            font=("Courier", 14),
            justify="left",
            wraplength=900
        )
        self.system_info_label.pack(pady=20, padx=30)

        self.cpu_bar = ctk.CTkProgressBar(self.tab_system, width=400)
        self.cpu_bar.pack(pady=5)
        self.cpu_bar.set(0)

        self.ram_bar = ctk.CTkProgressBar(self.tab_system, width=400, progress_color="orange")
        self.ram_bar.pack(pady=5)
        self.ram_bar.set(0)

    def get_process_info(self, pid):
        try:
            p = psutil.Process(pid)
            info = {
                "pid": p.pid,
                "name": p.name(),
                "status": p.status(),
                "memory_mb": round(p.memory_info().rss / (1024 ** 2), 2),
                "cpu_percent": p.cpu_percent(interval=0.1),
                "create_time": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(p.create_time())),
                "exe": p.exe() if p.exe() else "Неизвестно",
                "username": p.username(),
            }
            return info
        except (psutil.NoSuchProcess, psutil.AccessDenied, psutil.ZombieProcess):
            return None

    def refresh_process_list(self):
        for item in self.tree.get_children():
            self.tree.delete(item)
        threading.Thread(target=self._load_processes, daemon=True).start()

    def _load_processes(self):
        processes = []
        for proc in psutil.process_iter(['pid', 'name', 'status']):
            try:
                p = psutil.Process(proc.info['pid'])
                mem = p.memory_info().rss / (1024 ** 2)
                processes.append((
                    proc.info['pid'],
                    proc.info['name'][:50] + "..." if len(proc.info['name']) > 50 else proc.info['name'],
                    proc.info['status'],
                    f"{mem:.1f}"
                ))
            except:
                continue

        processes.sort(key=lambda x: float(x[3]) if x[3].replace('.', '').isdigit() else 0, reverse=True)
        self.after(0, self._insert_processes, processes)

    def _insert_processes(self, processes):
        for p in processes:
            self.tree.insert("", "end", values=p)

    def debounce_search(self, event=None):
        if hasattr(self, '_after_id'):
            self.after_cancel(self._after_id)
        self._after_id = self.after(300, self.filter_processes)

    def filter_processes(self):
        query = self.search_entry.get().lower()
        for item in self.tree.get_children():
            self.tree.delete(item)
        threading.Thread(target=self._load_processes_filtered, args=(query,), daemon=True).start()

    def _load_processes_filtered(self, query):
        processes = []
        for proc in psutil.process_iter(['pid', 'name', 'status']):
            try:
                name = proc.info['name']
                if query not in name.lower():
                    continue
                p = psutil.Process(proc.info['pid'])
                mem = p.memory_info().rss / (1024 ** 2)
                processes.append((
                    proc.info['pid'],
                    name[:50] + "..." if len(name) > 50 else name,
                    proc.info['status'],
                    f"{mem:.1f}"
                ))
            except:
                continue

        processes.sort(key=lambda x: float(x[3]) if x[3].replace('.', '').isdigit() else 0, reverse=True)
        self.after(0, self._insert_processes, processes)

    def on_process_select(self, event=None):
        selected_item = self.tree.selection()
        if not selected_item:
            return

        item = self.tree.item(selected_item[0])
        pid = int(item['values'][0])

        threading.Thread(target=self._fetch_and_display_process_info, args=(pid,), daemon=True).start()
        self.start_graph_monitoring(pid)

    def _fetch_and_display_process_info(self, pid):
        info = self.get_process_info(pid)
        if info is None:
            self.after(0, lambda: messagebox.showerror("Ошибка", f"Не удалось получить информацию о процессе {pid}"))
            return

        info_text = (
            f"📌 PID: {info['pid']}\n"
            f"📛 Имя: {info['name']}\n"
            f"📊 Статус: {info['status']}\n"
            f"🧠 Память: {info['memory_mb']} МБ\n"
            f"⏱️ CPU: {info['cpu_percent']}%\n"
            f"🕒 Запущен: {info['create_time']}\n"
            f"👤 Пользователь: {info['username']}\n"
            f"📂 Путь: {info['exe']}"
        )

        self.after(0, lambda: self.info_label.configure(text=info_text))

    def start_graph_monitoring(self, pid):
        self.selected_pid = pid
        self.cpu_data.clear()
        self.mem_data.clear()
        self.time_data.clear()

        if self.anim:
            self.anim.event_source.stop()

        self.anim = FuncAnimation(
            self.figure,
            self.update_graph,
            interval=2000,  # обновляем каждые 2 сек — меньше нагрузка
            cache_frame_data=False
        )
        self.canvas.draw()

    def update_graph(self, frame):
        if not self.selected_pid:
            return

        try:
            p = psutil.Process(self.selected_pid)
            cpu = p.cpu_percent(interval=0.1)
            mem = p.memory_info().rss / (1024 ** 2)

            self.cpu_data.append(cpu)
            self.mem_data.append(mem)
            self.time_data.append(time.time())

            if len(self.cpu_data) > 60:
                self.cpu_data.pop(0)
                self.mem_data.pop(0)
                self.time_data.pop(0)

            self.ax_cpu.clear()
            self.ax_mem.clear()

            self.ax_cpu.plot(self.time_data, self.cpu_data, color='cyan', linewidth=2, marker='o', markersize=2)
            self.ax_mem.plot(self.time_data, self.mem_data, color='orange', linewidth=2, marker='o', markersize=2)

            self.ax_cpu.set_title("CPU (%)", color='white')
            self.ax_mem.set_title("Память (МБ)", color='white')
            max_cpu = max(self.cpu_data) if self.cpu_data else 100
            max_mem = max(self.mem_data) if self.mem_data else 500
            self.ax_cpu.set_ylim(0, max_cpu + 10)
            self.ax_mem.set_ylim(0, max_mem + 50)

            for ax in [self.ax_cpu, self.ax_mem]:
                ax.set_facecolor('#333333')
                ax.tick_params(colors='white')
                ax.grid(True, color='gray', linestyle='--', linewidth=0.5)

            self.figure.tight_layout()
            self.canvas.draw()

        except Exception as e:
            print("Ошибка графика:", e)
            pass

    def export_to_csv(self):
        filename = filedialog.asksaveasfilename(
            defaultextension=".csv",
            filetypes=[("CSV файлы", "*.csv"), ("TXT файлы", "*.txt")],
            title="Сохранить список процессов"
        )
        if not filename:
            return

        try:
            with open(filename, mode='w', newline='', encoding='utf-8') as file:
                writer = csv.writer(file)
                writer.writerow(["PID", "Имя процесса", "Статус", "Память (МБ)"])
                for item in self.tree.get_children():
                    writer.writerow(self.tree.item(item)['values'])
            messagebox.showinfo("Экспорт", f"Список процессов сохранён в:\n{filename}")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось сохранить файл: {e}")

    def kill_selected_process(self):
        selected_item = self.tree.selection()
        if not selected_item:
            messagebox.showwarning("Внимание", "Сначала выберите процесс!")
            return

        item = self.tree.item(selected_item[0])
        pid = int(item['values'][0])
        name = item['values'][1]

        if messagebox.askyesno("Подтверждение", f"Завершить процесс:\n{name} (PID: {pid})?"):
            try:
                p = psutil.Process(pid)
                p.terminate()
                p.wait(timeout=3)
                messagebox.showinfo("Успех", f"Процесс {name} завершён.")
                self.refresh_process_list()
                self.selected_pid = None
                if self.anim:
                    self.anim.event_source.stop()
            except Exception as e:
                messagebox.showerror("Ошибка", f"Не удалось завершить процесс: {e}")

    def update_system_info(self):
        if self.is_updating:
            return  # Защита от параллельных вызовов

        self.is_updating = True
        try:
            cpu_percent = psutil.cpu_percent(interval=0.5)
            ram = psutil.virtual_memory()

            # Безопасное получение информации о дисках
            disk_info = []
            for partition in psutil.disk_partitions():
                try:
                    if 'cdrom' in partition.opts or partition.fstype == '':
                        continue
                    usage = psutil.disk_usage(partition.mountpoint)
                    disk_info.append(f"{partition.mountpoint} ({usage.percent}%)")
                except (PermissionError, OSError) as e:
                    # Игнорируем недоступные диски
                    continue

            uptime_seconds = time.time() - psutil.boot_time()
            hours, remainder = divmod(uptime_seconds, 3600)
            minutes, _ = divmod(remainder, 60)

            info = (
                f"⏱️  Загрузка CPU: {cpu_percent}%\n"
                f"🧠  Использование RAM: {ram.percent}% ({ram.used // (1024**2)} МБ из {ram.total // (1024**2)} МБ)\n"
                f"⏳  Аптайм системы: {int(hours)} ч {int(minutes)} мин\n"
                f"🧩  Всего процессов: {len(psutil.pids())}\n"
                f"💽  Диски: {', '.join(disk_info) if disk_info else 'Нет доступных дисков'}"
            )

            self.after(0, lambda: self.system_info_label.configure(text=info))
            self.cpu_bar.set(cpu_percent / 100)
            self.ram_bar.set(ram.percent / 100)

        except Exception as e:
            print("Ошибка обновления системной информации:", e)
        finally:
            self.is_updating = False

        self.after(2000, self.update_system_info)

    def auto_refresh(self):
        self.refresh_process_list()
        self.after(5000, self.auto_refresh)

if __name__ == "__main__":
    app = ProcessManagerApp()
    app.mainloop()
