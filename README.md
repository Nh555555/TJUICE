# TJUICE
from flask import Flask, request, jsonify, render_template, redirect, url_for, session
from PIL import Image
import os
import cv2
import pytesseract
import numpy as np
from flask_mail import Mail, Message
from werkzeug.security import generate_password_hash, check_password_hash
import sqlite3

app = Flask(__name__)
app.secret_key = 'supersecretkey'

UPLOAD_FOLDER = 'uploads'
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

# Configuration de l'email
app.config['MAIL_SERVER'] = 'smtp.gmail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USERNAME'] = 'votre_email@gmail.com'
app.config['MAIL_PASSWORD'] = 'votre_mot_de_passe'
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USE_SSL'] = False
mail = Mail(app)

# Assurez-vous que le dossier d'uploads existe
if not os.path.exists(UPLOAD_FOLDER):
    os.makedirs(UPLOAD_FOLDER)

# Initialiser la base de données
def init_db():
    with sqlite3.connect('sales.db') as conn:
        cursor = conn.cursor()
        cursor.execute('''CREATE TABLE IF NOT EXISTS sales (
                          id INTEGER PRIMARY KEY AUTOINCREMENT,
                          product TEXT,
                          total REAL,
                          commission REAL,
                          date TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')
        conn.commit()

# Utilisateurs d'administration simulés
users = {
    'admin': generate_password_hash('password123')
}

def extract_product_info(image_path):
    """Utilise OCR pour extraire les informations du ticket"""
    try:
        image = cv2.imread(image_path)
        if image is None:
            raise ValueError("Impossible de charger l'image. Vérifiez le chemin et le format.")
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        text = pytesseract.image_to_string(gray, lang='fra')
        lines = text.split('\n')
        product_detected = False
        total = 0

        # Testez avec différents tickets pour valider la fiabilité de la détection
        print("Analyse des lignes extraites du ticket :")
        for line in lines:
            print(line)
            if "T-Juice" in line:
                product_detected = True
            if "Total" in line or "total" in line:
                try:
                    total = float(line.split()[-1].replace(',', '.'))
                except Exception as e:
                    print(f"Erreur d'extraction du total: {e}")

        return product_detected, total
    except Exception as e:
        print(f"Erreur lors de l'extraction du texte: {e}")
        return False, 0

@app.route('/')
def home():
    return render_template('index.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        if username in users and check_password_hash(users[username], password):
            session['user'] = username
            return redirect(url_for('admin_dashboard'))
        return "Échec de la connexion"
    return render_template('login.html')

@app.route('/admin')
def admin_dashboard():
    if 'user' not in session:
        return redirect(url_for('login'))
    with sqlite3.connect('sales.db') as conn:
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM sales')
        sales = cursor.fetchall()
    return render_template('dashboard.html', sales=sales)

@app.route('/upload', methods=['POST'])
def upload_ticket():
    if 'file' not in request.files:
        return jsonify({"error": "Aucun fichier fourni"}), 400

    file = request.files['file']
    if file.filename == '':
        return jsonify({"error": "Nom de fichier vide"}), 400

    file_path = os.path.join(app.config['UPLOAD_FOLDER'], file.filename)
    try:
        file.save(file_path)
    except Exception as e:
        return jsonify({"error": f"Impossible d'enregistrer le fichier: {e}"}), 500

    product_detected, total = extract_product_info(file_path)

    if product_detected:
        commission = total * 0.05  # 5% de commission par exemple
        try:
            msg = Message('Nouvelle vente de T-Juice!', sender='votre_email@gmail.com', recipients=['vendeur_email@example.com'])
            msg.body = f"Un produit T-Juice a été vendu. Total: {total}€, Commission: {commission}€"
            mail.send(msg)
        except Exception as e:
            print(f"Erreur lors de l'envoi de l'email: {e}")

        # Enregistrer la vente dans la base de données
        with sqlite3.connect('sales.db') as conn:
            cursor = conn.cursor()
            cursor.execute('INSERT INTO sales (product, total, commission) VALUES (?, ?, ?)',
                           ('T-Juice', total, commission))
            conn.commit()

        return jsonify({"message": "Produit T-Juice détecté!", "total": total, "commission": commission})
    else:
        return jsonify({"message": "Aucun produit T-Juice détecté."})

@app.route('/logout')
def logout():
    session.pop('user', None)
    return redirect(url_for('home'))

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=8080, debug=False, threaded=True)
