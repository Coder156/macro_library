# macro_library
DO NOT use the autoclicker it is broken :(

Here is the code for it
import tkinter as tk
from tkinter import ttk, messagebox, filedialog, colorchooser
import pyautogui
import keyboard
import time
import json
import os
import threading
import webbrowser
import ctypes

# For setting the wallpaper on Windows
SPI_SETDESKWALLPAPER = 20

# Define the path to Google Chrome (replace this with your actual Chrome path if necessary)
chrome_path = "C:/Program Files/Google/Chrome/Application/chrome.exe %s"


class Macro:
    """Handles storing and executing macro actions. Actions can include mouse clicks, keyboard presses, delays, etc."""

    def __init__(self, name="Temporary Macro"):
        self.name = name
        self.actions = []
        self.is_paused = False
        self.is_stopped = False

    def add_action(self, action_type, params):
        """Add a new action to the macro."""
        self.actions.append((action_type, params))

    def move_action(self, old_index, new_index):
        """Move an action from one position to another."""
        self.actions.insert(new_index, self.actions.pop(old_index))

    def delete_action(self, index):
        """Delete a specific action."""
        if 0 <= index < len(self.actions):
            del self.actions[index]

    def execute(self):
        """Execute the macro by going through all the actions."""
        self.is_stopped = False  # Reset stop flag
        for action, params in self.actions:
            if self.is_stopped:
                break
            while self.is_paused:
                time.sleep(0.1)
            if action == "Click":
                pyautogui.click(params["x"], params["y"])
            elif action == "Right Click":
                pyautogui.rightClick(params["x"], params["y"])
            elif action == "Double Click":
                pyautogui.doubleClick(params["x"], params["y"])
            elif action == "Scroll":
                pyautogui.scroll(params["amount"])
            elif action == "Press Key":
                keyboard.press_and_release(params["key"])
            elif action == "Type":
                pyautogui.moveTo(params["x"], params["y"])  # Move to the position for typing
                pyautogui.write(params["text"])
            elif action == "Delay":
                time.sleep(params["duration"] / 1000.0)  # Convert ms to seconds
            elif action == "Open Application":
                os.startfile(params["path"])
            elif action == "Open URL":
                webbrowser.get(f"{chrome_path}").open_new_tab(params["url"])  # Opens URL in Chrome
            elif action == "Set Background":
                self.set_wallpaper(params["file_path"])

    def set_wallpaper(self, file_path):
        """Set the desktop wallpaper (Windows only)."""
        try:
            ctypes.windll.user32.SystemParametersInfoW(SPI_SETDESKWALLPAPER, 0, file_path, 0)
        except Exception as e:
            messagebox.showerror("Error", f"Failed to set wallpaper: {e}")

    def to_dict(self):
        """Convert macro actions to a serializable format."""
        return {"name": self.name, "actions": self.actions}

    @staticmethod
    def from_dict(data):
        """Create a Macro object from a dictionary."""
        macro = Macro(data["name"])
        macro.actions = data["actions"]
        return macro

    def pause(self):
        """Pause the macro execution."""
        self.is_paused = not self.is_paused

    def stop(self):
        """Stop the macro execution."""
        self.is_stopped = True


class MacroGUI:
    """GUI for building and managing macros. Users can create, run, save, and load macros with various actions."""

    def __init__(self, root):
        self.root = root
        self.root.title("Macro Library")  # Add the title "Macro Library"
        self.current_macro = Macro()  # Always have a default in-memory macro
        self.macros = {}

        # Tab Layout
        self.notebook = ttk.Notebook(root)
        self.notebook.grid(row=0, column=0, sticky="nsew")

        # Main Macro Tab
        self.macro_frame = tk.Frame(self.notebook, bg="#1c1c1c")
        self.notebook.add(self.macro_frame, text="Macro")

        # Autoclicker Tab
        self.autoclicker_frame = tk.Frame(self.notebook, bg="#1c1c1c")
        self.notebook.add(self.autoclicker_frame, text="AutoClicker")

        # Color Wheel Tab
        self.color_wheel_frame = tk.Frame(self.notebook, bg="#1c1c1c")
        self.notebook.add(self.color_wheel_frame, text="Color Wheel")

        self.setup_macro_tab()
        self.setup_autoclicker_tab()
        self.setup_color_wheel_tab()

    def setup_macro_tab(self):
        """Setup the main macro tab with buttons and input fields for creating macros."""
        # Top Section for Creating Macros
        self.name_label = tk.Label(self.macro_frame, text="Macro Name:", bg="#1c1c1c", fg="white")
        self.name_label.grid(row=0, column=0, padx=10, pady=10)

        self.macro_name_entry = tk.Entry(self.macro_frame)
        self.macro_name_entry.grid(row=0, column=1, padx=10)

        self.add_macro_button = tk.Button(self.macro_frame, text="Add Macro", command=self.add_macro, bg="#2b2b2b",
                                          fg="white")
        self.add_macro_button.grid(row=0, column=2, padx=10)

        self.load_macro_button = tk.Button(self.macro_frame, text="Load Macro", command=self.load_macro, bg="#2b2b2b",
                                           fg="white")
        self.load_macro_button.grid(row=0, column=3, padx=10)

        self.save_macro_button = tk.Button(self.macro_frame, text="Save Macro", command=self.save_macro, bg="#2b2b2b",
                                           fg="white")
        self.save_macro_button.grid(row=0, column=4, padx=10)

        # Action Selection and Inputs
        self.action_label = tk.Label(self.macro_frame, text="Action:", bg="#1c1c1c", fg="white")
        self.action_label.grid(row=1, column=0, padx=10)

        self.action_type = ttk.Combobox(self.macro_frame,
                                        values=["Click", "Right Click", "Double Click", "Press Key", "Type", "Scroll",
                                                "Delay", "Open Application", "Open URL", "Set Background"])
        self.action_type.grid(row=1, column=1, padx=10)
        self.action_type.bind("<<ComboboxSelected>>", self.update_action_inputs)

        # Inputs that change dynamically based on action
        self.position_label = tk.Label(self.macro_frame, text="Position (x,y):", bg="#1c1c1c", fg="white")
        self.position_entry = tk.Entry(self.macro_frame)
        self.capture_position_button = tk.Button(self.macro_frame, text="Capture Position",
                                                 command=self.start_capture_position,
                                                 bg="#2b2b2b", fg="white")

        self.text_label = tk.Label(self.macro_frame, text="Text to Type:", bg="#1c1c1c", fg="white")
        self.text_entry = tk.Entry(self.macro_frame)

        self.key_label = tk.Label(self.macro_frame, text="Key to Press:", bg="#1c1c1c", fg="white")
        self.key_entry = tk.Entry(self.macro_frame)

        self.app_label = tk.Label(self.macro_frame, text="Select Application:", bg="#1c1c1c", fg="white")
        self.app_button = tk.Button(self.macro_frame, text="Browse...", command=self.select_application, bg="#2b2b2b",
                                    fg="white")
        self.app_path = ""

        self.url_label = tk.Label(self.macro_frame, text="URL to Open:", bg="#1c1c1c", fg="white")
        self.url_entry = tk.Entry(self.macro_frame)

        # Action Buttons
        self.add_action_button = tk.Button(self.macro_frame, text="Add Action", command=self.add_action, bg="#2b2b2b",
                                           fg="white")
        self.add_action_button.grid(row=3, column=0, padx=10)

        self.run_macro_button = tk.Button(self.macro_frame, text="Run Macro", command=self.run_macro, bg="#2b2b2b",
                                          fg="white")
        self.run_macro_button.grid(row=3, column=1, padx=10)

        self.clear_actions_button = tk.Button(self.macro_frame, text="Clear Actions", command=self.clear_actions,
                                              bg="#2b2b2b", fg="white")
        self.clear_actions_button.grid(row=3, column=2, padx=10)

        self.delete_action_button = tk.Button(self.macro_frame, text="Delete Selected Action",
                                              command=self.delete_action, bg="#2b2b2b", fg="white")
        self.delete_action_button.grid(row=3, column=3, padx=10)

        # Listbox for showing actions with Scrollbar
        self.action_listbox = tk.Listbox(self.macro_frame, bg="black", fg="white")
        self.action_listbox.grid(row=4, column=0, columnspan=5, padx=10, pady=10, sticky="nsew")

        self.scrollbar = tk.Scrollbar(self.macro_frame, orient="vertical", command=self.action_listbox.yview)
        self.action_listbox.config(yscrollcommand=self.scrollbar.set)
        self.scrollbar.grid(row=4, column=5, sticky="ns")

        # Hotkey Section
        self.hotkey_label = tk.Label(self.macro_frame, text="Hotkeys (default):", bg="#1c1c1c", fg="white")
        self.hotkey_label.grid(row=5, column=0, padx=10)

        self.hotkey_instructions = tk.Label(self.macro_frame, text="(Use key names like 'ctrl+s', 'shift+a')",
                                            bg="#1c1c1c", fg="gray")
        self.hotkey_instructions.grid(row=6, column=0, padx=10, columnspan=2)

        # Default hotkeys and entries
        self.run_hotkey_entry = tk.Entry(self.macro_frame)
        self.run_hotkey_entry.insert(0, "ctrl+r")  # Default run hotkey
        self.run_hotkey_entry.grid(row=7, column=1, padx=10)
        self.run_hotkey_button = tk.Button(self.macro_frame, text="Bind Run Hotkey", command=self.bind_run_hotkey,
                                           bg="#2b2b2b", fg="white")
        self.run_hotkey_button.grid(row=7, column=2, padx=10)

        self.save_hotkey_entry = tk.Entry(self.macro_frame)
        self.save_hotkey_entry.insert(0, "ctrl+s")  # Default save hotkey
        self.save_hotkey_entry.grid(row=8, column=1, padx=10)
        self.save_hotkey_button = tk.Button(self.macro_frame, text="Bind Save Hotkey", command=self.bind_save_hotkey,
                                            bg="#2b2b2b", fg="white")
        self.save_hotkey_button.grid(row=8, column=2, padx=10)

        self.pause_hotkey_entry = tk.Entry(self.macro_frame)
        self.pause_hotkey_entry.insert(0, "ctrl+p")  # Default pause hotkey
        self.pause_hotkey_entry.grid(row=9, column=1, padx=10)
        self.pause_hotkey_button = tk.Button(self.macro_frame, text="Bind Pause Hotkey", command=self.bind_pause_hotkey,
                                             bg="#2b2b2b", fg="white")
        self.pause_hotkey_button.grid(row=9, column=2, padx=10)

        self.stop_hotkey_entry = tk.Entry(self.macro_frame)
        self.stop_hotkey_entry.insert(0, "ctrl+q")  # Default stop hotkey
        self.stop_hotkey_entry.grid(row=10, column=1, padx=10)
        self.stop_hotkey_button = tk.Button(self.macro_frame, text="Bind Stop Hotkey", command=self.bind_stop_hotkey,
                                            bg="#2b2b2b", fg="white")
        self.stop_hotkey_button.grid(row=10, column=2, padx=10)

        # Status Label for Mouse Capture
        self.status_label = tk.Label(self.macro_frame, text="Status: Ready", bg="#1c1c1c", fg="white")
        self.status_label.grid(row=11, column=0, columnspan=5, sticky="w")

        # Bind default hotkeys initially
        self.bind_run_hotkey()
        self.bind_save_hotkey()
        self.bind_pause_hotkey()
        self.bind_stop_hotkey()

    def update_action_inputs(self, event):
        """Update input fields based on selected action."""
        action_type = self.action_type.get()

        # Remove unnecessary inputs
        self.position_label.grid_forget()
        self.position_entry.grid_forget()
        self.capture_position_button.grid_forget()
        self.text_label.grid_forget()
        self.text_entry.grid_forget()
        self.key_label.grid_forget()
        self.key_entry.grid_forget()
        self.app_label.grid_forget()
        self.app_button.grid_forget()
        self.url_label.grid_forget()
        self.url_entry.grid_forget()

        # Add relevant inputs based on action type
        if action_type == "Click" or action_type == "Right Click" or action_type == "Double Click":
            self.position_label.grid(row=2, column=0, padx=10)
            self.position_entry.grid(row=2, column=1, padx=10)
            self.capture_position_button.grid(row=2, column=2, padx=10)
        elif action_type == "Type":
            self.text_label.grid(row=2, column=0, padx=10)
            self.text_entry.grid(row=2, column=1, padx=10)
            self.position_label.grid(row=3, column=0, padx=10)
            self.position_entry.grid(row=3, column=1, padx=10)
            self.capture_position_button.grid(row=3, column=2, padx=10)
        elif action_type == "Press Key":
            self.key_label.grid(row=2, column=0, padx=10)
            self.key_entry.grid(row=2, column=1, padx=10)
        elif action_type == "Open Application":
            self.app_label.grid(row=2, column=0, padx=10)
            self.app_button.grid(row=2, column=1, padx=10)
        elif action_type == "Open URL":
            self.url_label.grid(row=2, column=0, padx=10)
            self.url_entry.grid(row=2, column=1, padx=10)
        elif action_type == "Set Background":
            self.app_label.grid(row=2, column=0, padx=10)
            self.app_button.grid(row=2, column=1, padx=10)

    def select_application(self):
        """Open file dialog to select an application to run."""
        self.app_path = filedialog.askopenfilename(filetypes=[("Executable Files", "*.exe"), ("All Files", "*.*")])

    def add_macro(self):
        """Create a new macro and store it."""
        name = self.macro_name_entry.get()
        if name:
            self.current_macro = Macro(name)
            self.macros[name] = self.current_macro
            messagebox.showinfo("Success", f"Macro '{name}' created!")
        else:
            self.current_macro = Macro("Temporary Macro")  # Create a default temporary macro
            messagebox.showinfo("Info", "A temporary macro is created for you!")

    def add_action(self):
        """Add an action to the current macro."""
        if not self.current_macro:
            self.add_macro()  # Automatically create a temporary macro if none exists

        action_type = self.action_type.get()
        if action_type == "Delay":
            try:
                delay = int(self.delay_entry.get())
                self.current_macro.add_action(action_type, {"duration": delay})
                self.action_listbox.insert(tk.END, f"Delay: {delay} ms")
            except ValueError:
                messagebox.showerror("Error", "Invalid delay value!")
        elif action_type == "Type":
            text = self.text_entry.get()
            pos = self.position_entry.get().split(',')
            if len(pos) == 2:
                try:
                    x, y = int(pos[0]), int(pos[1])
                    self.current_macro.add_action(action_type, {"text": text, "x": x, "y": y})
                    self.action_listbox.insert(tk.END, f"Type: '{text}' at ({x},{y})")
                except ValueError:
                    messagebox.showerror("Error", "Invalid position value!")
            else:
                messagebox.showerror("Error", "Position must be in format x,y!")
        elif action_type == "Press Key":
            key = self.key_entry.get()
            self.current_macro.add_action(action_type, {"key": key})
            self.action_listbox.insert(tk.END, f"Press Key: '{key}'")
        elif action_type == "Open Application":
            if self.app_path:
                self.current_macro.add_action(action_type, {"path": self.app_path})
                self.action_listbox.insert(tk.END, f"Open Application: {self.app_path}")
            else:
                messagebox.showerror("Error", "No application selected!")
        elif action_type == "Open URL":
            url = self.url_entry.get()
            if url:
                self.current_macro.add_action(action_type, {"url": url})
                self.action_listbox.insert(tk.END, f"Open URL: {url}")
            else:
                messagebox.showerror("Error", "No URL provided!")
        elif action_type == "Set Background":
            if self.app_path:
                self.current_macro.add_action(action_type, {"file_path": self.app_path})
                self.action_listbox.insert(tk.END, f"Set Background: {self.app_path}")
            else:
                messagebox.showerror("Error", "No image file selected!")
        else:
            pos = self.position_entry.get().split(',')
            if len(pos) == 2:
                try:
                    x, y = int(pos[0]), int(pos[1])
                    self.current_macro.add_action(action_type, {"x": x, "y": y})
                    self.action_listbox.insert(tk.END, f"{action_type} at ({x},{y})")
                except ValueError:
                    messagebox.showerror("Error", "Invalid position value!")
            else:
                messagebox.showerror("Error", "Position must be in format x,y!")

    def delete_action(self):
        """Delete the selected action."""
        selected = self.action_listbox.curselection()
        if selected:
            index = selected[0]
            self.action_listbox.delete(index)
            self.current_macro.delete_action(index)

    def start_capture_position(self):
        """Capture mouse position by waiting for user to press the backtick key (`)."""
        self.status_label.config(text="Move the mouse to the desired position and press the backtick (`) key.")

        # Wait for the user to press the backtick key after moving the mouse to the desired position
        keyboard.wait('`')

        # Get the current mouse position
        x, y = pyautogui.position()

        # Set the position in the entry box
        self.position_entry.delete(0, tk.END)
        self.position_entry.insert(0, f"{x},{y}")

        self.status_label.config(text="Position captured. Ready.")

    def clear_actions(self):
        """Clear all actions in the current macro."""
        if self.current_macro:
            self.current_macro.actions.clear()
            self.action_listbox.delete(0, tk.END)

    def run_macro(self):
        """Run the current in-memory macro."""
        if not self.current_macro.actions:
            messagebox.showerror("Error", "No actions in the macro!")
            return

        self.status_label.config(text="Status: Running...")
        self.current_macro.execute()
        self.status_label.config(text="Status: Ready")

    def save_macro(self):
        """Save the current macro to a file."""
        if not self.current_macro:
            messagebox.showerror("Error", "No macro selected!")
            return

        file_path = filedialog.asksaveasfilename(defaultextension=".json", filetypes=[("JSON files", "*.json")])
        if file_path:
            with open(file_path, 'w') as f:
                json.dump(self.current_macro.to_dict(), f)
            messagebox.showinfo("Success", f"Macro saved to {file_path}")

    def load_macro(self):
        """Load a macro from a file."""
        file_path = filedialog.askopenfilename(filetypes=[("JSON files", "*.json")])
        if file_path:
            with open(file_path, 'r') as f:
                macro_data = json.load(f)
                self.current_macro = Macro.from_dict(macro_data)
                self.macros[self.current_macro.name] = self.current_macro
                self.macro_name_entry.delete(0, tk.END)
                self.macro_name_entry.insert(0, self.current_macro.name)

                # Populate the action listbox
                self.action_listbox.delete(0, tk.END)
                for action, params in self.current_macro.actions:
                    if action == "Delay":
                        self.action_listbox.insert(tk.END, f"Delay: {params['duration']} ms")
                    elif action == "Type":
                        self.action_listbox.insert(tk.END, f"Type: '{params['text']}' at ({params['x']},{params['y']})")
                    elif action == "Press Key":
                        self.action_listbox.insert(tk.END, f"Press Key: '{params['key']}'")
                    elif action == "Open Application":
                        self.action_listbox.insert(tk.END, f"Open Application: {params['path']}")
                    elif action == "Open URL":
                        self.action_listbox.insert(tk.END, f"Open URL: {params['url']}")
                    elif action == "Set Background":
                        self.action_listbox.insert(tk.END, f"Set Background: {params['file_path']}")
                    else:
                        self.action_listbox.insert(tk.END, f"{action} at ({params['x']},{params['y']})")

    def bind_run_hotkey(self):
        """Bind a hotkey to run the current in-memory macro."""
        hotkey = self.run_hotkey_entry.get()
        if hotkey:
            keyboard.add_hotkey(hotkey, self.run_macro)
            messagebox.showinfo("Success", f"Run Hotkey '{hotkey}' bound to run the current macro.")
        else:
            messagebox.showerror("Error", "No hotkey provided!")

    def bind_save_hotkey(self):
        """Bind a hotkey to save the current macro."""
        hotkey = self.save_hotkey_entry.get()
        if hotkey:
            keyboard.add_hotkey(hotkey, self.save_macro)
            messagebox.showinfo("Success", f"Save Hotkey '{hotkey}' bound to save the current macro.")
        else:
            messagebox.showerror("Error", "No hotkey provided!")

    def bind_pause_hotkey(self):
        """Bind a hotkey to pause/resume the current macro."""
        hotkey = self.pause_hotkey_entry.get()
        if hotkey:
            keyboard.add_hotkey(hotkey, self.current_macro.pause)
            messagebox.showinfo("Success", f"Pause Hotkey '{hotkey}' bound to pause/resume the current macro.")
        else:
            messagebox.showerror("Error", "No hotkey provided!")

    def bind_stop_hotkey(self):
        """Bind a hotkey to stop the current macro."""
        hotkey = self.stop_hotkey_entry.get()
        if hotkey:
            keyboard.add_hotkey(hotkey, self.current_macro.stop)
            messagebox.showinfo("Success", f"Stop Hotkey '{hotkey}' bound to stop the current macro.")
        else:
            messagebox.showerror("Error", "No hotkey provided!")

    # --- AUTCLICKER RELATED ---
    def setup_autoclicker_tab(self):
        """Setup the AutoClicker tab."""
        self.auto_click_button = tk.Button(self.autoclicker_frame, text="Start AutoClicker",
                                           command=self.start_autoclicker, bg="#2b2b2b", fg="white")
        self.auto_click_button.grid(row=0, column=0, padx=10, pady=10)

        self.stop_click_button = tk.Button(self.autoclicker_frame, text="Stop AutoClicker",
                                           command=self.stop_autoclicker, bg="#2b2b2b", fg="white")
        self.stop_click_button.grid(row=0, column=1, padx=10, pady=10)

        self.auto_click_speed_label = tk.Label(self.autoclicker_frame, text="Click Interval (ms):", bg="#1c1c1c",
                                               fg="white")
        self.auto_click_speed_label.grid(row=1, column=0, padx=10)

        self.auto_click_speed_entry = tk.Entry(self.autoclicker_frame)
        self.auto_click_speed_entry.insert(0, "100")  # Default interval
        self.auto_click_speed_entry.grid(row=1, column=1, padx=10)

        self.click_type_label = tk.Label(self.autoclicker_frame, text="Click Type:", bg="#1c1c1c", fg="white")
        self.click_type_label.grid(row=2, column=0, padx=10)

        self.click_type_combo = ttk.Combobox(self.autoclicker_frame,
                                             values=["Left Click", "Right Click", "Double Click"])
        self.click_type_combo.grid(row=2, column=1, padx=10)
        self.click_type_combo.set("Left Click")

        self.click_times_label = tk.Label(self.autoclicker_frame, text="Number of Clicks (0 = infinite):", bg="#1c1c1c",
                                          fg="white")
        self.click_times_label.grid(row=3, column=0, padx=10)

        self.click_times_entry = tk.Entry(self.autoclicker_frame)
        self.click_times_entry.insert(0, "0")  # Default to infinite clicks
        self.click_times_entry.grid(row=3, column=1, padx=10)

        self.autoclick_hotkey_label = tk.Label(self.autoclicker_frame, text="Autoclick Hotkey (e.g., 'ctrl+shift+c'):",
                                               bg="#1c1c1c", fg="white")
        self.autoclick_hotkey_label.grid(row=4, column=0, padx=10)

        self.autoclick_hotkey_entry = tk.Entry(self.autoclicker_frame)
        self.autoclick_hotkey_entry.insert(0, "ctrl+shift+c")
        self.autoclick_hotkey_entry.grid(row=4, column=1, padx=10)

        self.bind_autoclick_hotkey_button = tk.Button(self.autoclicker_frame, text="Bind Hotkey",
                                                      command=self.bind_autoclick_hotkey, bg="#2b2b2b", fg="white")
        self.bind_autoclick_hotkey_button.grid(row=4, column=2, padx=10)

        self.stop_autoclicker_flag = False
        self.autoclick_hotkey = None

    def start_autoclicker(self):
        """Start the autoclicker with user-defined settings."""
        interval = int(self.auto_click_speed_entry.get()) / 1000.0
        click_type = self.click_type_combo.get()
        click_times = int(self.click_times_entry.get())

        def autoclick():
            clicks = 0
            while not self.stop_autoclicker_flag:
                if click_times > 0 and clicks >= click_times:
                    break

                if click_type == "Left Click":
                    pyautogui.click()
                elif click_type == "Right Click":
                    pyautogui.rightClick()
                elif click_type == "Double Click":
                    pyautogui.doubleClick()
                time.sleep(interval)
                clicks += 1

        self.stop_autoclicker_flag = False
        threading.Thread(target=autoclick).start()

    def stop_autoclicker(self):
        """Stop the autoclicker."""
        self.stop_autoclicker_flag = True

    def bind_autoclick_hotkey(self):
        """Bind the autoclicker start/stop hotkey."""
        hotkey = self.autoclick_hotkey_entry.get()
        if self.autoclick_hotkey:
            keyboard.remove_hotkey(self.autoclick_hotkey)
        self.autoclick_hotkey = keyboard.add_hotkey(hotkey, self.toggle_autoclick)

    def toggle_autoclick(self):
        """Toggle the start/stop state of the autoclicker using the hotkey."""
        if self.stop_autoclicker_flag:
            self.start_autoclicker()
        else:
            self.stop_autoclicker()

    # --- COLOR WHEEL TAB ---
    def setup_color_wheel_tab(self):
        """Setup the color wheel tab for color picking."""
        self.color_wheel_label = tk.Label(self.color_wheel_frame,
                                          text="Hover over the color to get RGB and Hex values.", bg="#1c1c1c",
                                          fg="white")
        self.color_wheel_label.grid(row=0, column=0, padx=10, pady=10)

        self.pick_color_button = tk.Button(self.color_wheel_frame, text="Pick Color", command=self.pick_color,
                                           bg="#2b2b2b", fg="white")
        self.pick_color_button.grid(row=1, column=0, padx=10, pady=10)

        self.rgb_label = tk.Label(self.color_wheel_frame, text="RGB:", bg="#1c1c1c", fg="white")
        self.rgb_label.grid(row=2, column=0, padx=10)

        self.rgb_value_label = tk.Label(self.color_wheel_frame, text="", bg="#1c1c1c", fg="white")
        self.rgb_value_label.grid(row=2, column=1, padx=10)

        self.hex_label = tk.Label(self.color_wheel_frame, text="Hex:", bg="#1c1c1c", fg="white")
        self.hex_label.grid(row=3, column=0, padx=10)

        self.hex_value_label = tk.Label(self.color_wheel_frame, text="", bg="#1c1c1c", fg="white")
        self.hex_value_label.grid(row=3, column=1, padx=10)

    def pick_color(self):
        """Pick a color from the screen and display its RGB and Hex values."""
        color = colorchooser.askcolor()[0]  # Get RGB values
        if color:
            r, g, b = color
            hex_color = f"#{int(r):02x}{int(g):02x}{int(b):02x}".upper()

            # Update labels
            self.rgb_value_label.config(text=f"{int(r)}, {int(g)}, {int(b)}")
            self.hex_value_label.config(text=hex_color)


# Main execution
if __name__ == "__main__":
    root = tk.Tk()
    gui = MacroGUI(root)
    root.mainloop()
