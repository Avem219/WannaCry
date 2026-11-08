pip install cryptography requests customtkinter pyinstaller

import os
import base64
import requests
import customtkinter as ctk
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

# Configure the GUI look and feel
ctk.set_appearance_mode("System")
ctk.set_default_color_theme("blue")

# Change this to your live deployed cloud server address
SERVER_URL = "https://your-payment-server.com" 

class PortableDecryptor(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.title("Secure Data Portal")
        self.geometry("550x650")
        self.resizable(False, False)
        self.setup_ui()

    def setup_ui(self):
        # Header
        self.title_label = ctk.CTkLabel(self, text="🔓 Data Activation & Decryption", font=ctk.CTkFont(size=20, weight="bold"))
        self.title_label.pack(pady=20)

        # Tab 1: Mobile Money Payment Request
        self.pay_frame = ctk.CTkFrame(self)
        self.pay_frame.pack(fill="x", padx=20, pady=10)
        
        self.pay_title = ctk.CTkLabel(self.pay_frame, text="Step 1: Mobile Money Payment (Airtel / MTN)", font=ctk.CTkFont(weight="bold"))
        self.pay_title.pack(pady=5)
        
        self.phone_entry = ctk.CTkEntry(self.pay_frame, placeholder_text="Enter Phone Number (e.g., 25677xxxxxx)")
        self.phone_entry.pack(fill="x", padx=20, pady=5)
        
        self.tx_entry = ctk.CTkEntry(self.pay_frame, placeholder_text="Transaction ID / File ID reference")
        self.tx_entry.pack(fill="x", padx=20, pady=5)
        
        self.pay_btn = ctk.CTkButton(self.pay_frame, text="Request Payment Prompt", command=self.trigger_payment, fg_color="#2e7d32", hover_color="#1b5e20")
        self.pay_btn.pack(pady=10)

        # Tab 2: Decryption Execution
        self.crypto_frame = ctk.CTkFrame(self)
        self.crypto_frame.pack(fill="x", padx=20, pady=10)
        
        self.crypto_title = ctk.CTkLabel(self.crypto_frame, text="Step 2: Decrypt Purchased File", font=ctk.CTkFont(weight="bold"))
        self.crypto_title.pack(pady=5)

        self.payload_input = ctk.CTkTextbox(self.crypto_frame, height=100)
        self.payload_input.pack(fill="x", padx=20, pady=5)
        self.payload_input.insert("0.0", "Paste encrypted package data here...")

        self.decrypt_btn = ctk.CTkButton(self.crypto_frame, text="Verify Payment & Unlock Data", command=self.execute_decryption)
        self.decrypt_btn.pack(pady=10)

        # Output Results Window
        self.output_label = ctk.CTkLabel(self, text="Output Console:")
        self.output_label.pack(anchor="w", padx=25)
        self.output_box = ctk.CTkTextbox(self, height=120, fg_color="#1e1e1e", text_color="#00ff00")
        self.output_box.pack(fill="x", padx=20, pady=5)

    def log(self, text):
        self.output_box.delete("0.0", "end")
        self.output_box.insert("0.0", text)

    def trigger_payment(self):
        phone = self.phone_entry.get().strip()
        tx_id = self.tx_entry.get().strip()
        
        if not phone or not tx_id:
            self.log("Error: Phone number and Reference ID are required.")
            return
            
        self.log(f"Sending Mobile Money push to {phone}...\nPlease check your phone for the PIN prompt.")
        
        # Make network request to your cloud back-end to launch USSD / API Push
        try:
            res = requests.post(f"{SERVER_URL}/api/initiate-payment", json={"phone": phone, "reference": tx_id}, timeout=10)
            if res.status_code == 200:
                self.log("Payment initiated! Complete the prompt on your handset then click Step 2.")
            else:
                self.log(f"Server rejected prompt request: {res.text}")
        except Exception as e:
            self.log(f"Connection failure to Payment Server: {str(e)}")

    def execute_decryption(self):
        encrypted_blob = self.payload_input.get("0.0", "end").strip()
        tx_id = self.tx_entry.get().strip()
        
        self.log("Verifying transaction status with central database...")
        
        try:
            # Contact server to fetch token if payment is cleared
            res = requests.post(f"{SERVER_URL}/api/verify-and-get-token", json={"reference": tx_id}, timeout=10)
            if res.status_code != 200:
                self.log("Access Denied: Payment not detected or transaction pending.")
                return
                
            token_data = res.json()
            decryptor_token = token_data.get("key") # Base64 Key strings
            
            # Perform local, standalone decryption
            raw_key = base64.b64decode(decryptor_token.encode('utf-8'))
            combined_payload = base64.b64decode(encrypted_blob.encode('utf-8'))
            
            nonce = combined_payload[:12]
            ciphertext = combined_payload[12:]
            
            aesgcm = AESGCM(raw_key)
            decrypted_bytes = aesgcm.decrypt(nonce, ciphertext, None)
            
            self.log(f"✨ SUCCESS! DATA UNLOCKED:\n\n{decrypted_bytes.decode('utf-8')}")
            
        except Exception as e:
            self.log(f"Decryption failed: Ensure data package matches token registry. Detail: {str(e)}")

if __name__ == "__main__":
    app = PortableDecryptor()
    app.mainloop()


pyinstaller --noconsole --onefile client_app.py
