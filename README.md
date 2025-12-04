# admin code 
```
from app import app, db, ActivationCode

with app.app_context():
    db.create_all()
    # Create the Master Key
    key = ActivationCode(code="ADMIN-KEY-1234", days_valid=3650)
    db.session.add(key)
    db.session.commit()
    print("SUCCESS: Admin Key Created!")

exit()
```

#master key2
```
from app import app, db, ActivationCode

with app.app_context():
    db.create_all()
    # Create a Master Admin Key
    master_key = ActivationCode(code="ADMIN-START-KEY", days_valid=9999)
    db.session.add(master_key)
    db.session.commit()
    print("MASTER KEY CREATED: ADMIN-START-KEY")
```

# Auth-Page-

Creds to the server 

root:Nano_tech99!!


```bash
# database setup
sudo apt update
sudo apt install mysql-server -y

sudo systemctl start mysql
sudo systemctl enable mysql

sudo mysql_secure_installation

My_sql_data_tech99!
```

Creating the db values. 
```
CREATE DATABASE platform_db;
USE platform_db;

# auth table.

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    secret_key VARCHAR(64),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP NULL
);

Creating flask user to manage the platform db.
CREATE USER 'platform_user'@'localhost' IDENTIFIED BY 'StrongPassword123!!';
GRANT ALL PRIVILEGES ON platform_db.* TO 'platform_user'@'localhost';
FLUSH PRIVILEGES;

ALTER TABLE users
ADD COLUMN activation_code VARCHAR(50) NOT NULL;

For preporulated activatoin code.
CREATE TABLE activation_codes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    used BOOLEAN DEFAULT FALSE
);

# Random Activation codes.

INSERT INTO activation_codes (code) VALUES
('XTR-B9F2-4M1P-J8A3-7W5L-E2S4'),
('PJD-1A7K-C3G5-T9E2-H6R8-Q4B0'),
('KZM-5E8R-N2J7-Y6T1-C0V9-F3D4'),
('LHG-3B9D-6P4F-V1Z0-M7W2-S5X8'),
('WFA-R4T0-Q2H6-M8K1-G5J9-D3C7'),
('YQB-7C5V-A3X9-2S1B-4E6R-8T0I'),
('MNC-F0S9-H4L7-J5Q3-K2R6-P1T8'),
('EKS-2W1Z-6Y0V-8U3X-4T7R-5Q9P'),
('TUV-D5G8-H1K4-L6M9-N3P7-R0S2'),
('GZL-9R0Q-4A3C-B7D2-F5E8-H1J6'),
('IHP-1M4K-J7L0-N3P6-Q9R2-S5T8'),
('OAX-6T8W-V2Y5-Z0B3-C4D7-F9G1'),
('VRE-3C5D-G7H9-J1K2-L4M6-P8Q0'),
('BQU-8N7M-6L5K-4J3H-2G1F-0D9C'),
('CST-Z0Y9-X8W7-V6U5-T4S3-R2Q1'),
('NKG-P3L7-J2H6-F1D5-C0B4-A9Z8'),
('QDW-5E8F-G1H4-J7K0-L3M6-P9R2'),
('JFL-4I7O-E2A9-U5D1-S8T0-V3W6'),
('HDM-0B1C-2D3E-4F5G-6H7I-8J9K'),
('ZPY-W6V5-T4S3-R2Q1-P0O9-N8M7');

```
# Flask code to handel auth. 
```python
from flask import Flask, render_template, request, redirect, url_for, session, flash
import mysql.connector
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)
app.secret_key = "SuperSecretKey123!"

# MySQL connection
db = mysql.connector.connect(
    host="localhost",
    user="platform_user",
    password="StrongPassword123!!",
    database="platform_db"
)

# REGISTER
@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        username = request.form.get("username")
        email = request.form.get("email")
        password = request.form.get("password")
        confirm = request.form.get("confirm")
        activation_code = request.form.get("activation_code")

        # Validate fields
        if not all([username, email, password, confirm, activation_code]):
            flash("All fields are required!", "error")
            return redirect(url_for("register"))
        if password != confirm:
            flash("Passwords do not match!", "error")
            return redirect(url_for("register"))

        cursor = db.cursor(dictionary=True)

        # Check activation code
        cursor.execute("SELECT * FROM activation_codes WHERE code=%s AND used=0", (activation_code,))
        code_row = cursor.fetchone()
        if not code_row:
            flash("Invalid or already used activation code!", "error")
            cursor.close()
            return redirect(url_for("register"))

        # Check if email already exists
        cursor.execute("SELECT * FROM users WHERE email=%s", (email,))
        if cursor.fetchone():
            flash("Email already registered!", "error")
            cursor.close()
            return redirect(url_for("register"))

        # Hash password
        password_hash = generate_password_hash(password)

        # Insert new user
        cursor.execute(
            "INSERT INTO users (username, email, password_hash, activation_code) VALUES (%s, %s, %s, %s)",
            (username, email, password_hash, activation_code)
        )

        # Mark activation code as used
        cursor.execute("UPDATE activation_codes SET used=1 WHERE code=%s", (activation_code,))

        db.commit()
        cursor.close()

        flash("Registration successful! You can now log in.", "success")
        return redirect(url_for("login"))

    return render_template("register.html")

# LOGIN
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        email = request.form.get("email")
        password = request.form.get("password")

        cursor = db.cursor(dictionary=True)
        cursor.execute("SELECT * FROM users WHERE email=%s", (email,))
        user = cursor.fetchone()
        cursor.close()

        if user and check_password_hash(user["password_hash"], password):
            session["user_id"] = user["id"]
            session["username"] = user["username"]
            return redirect(url_for("dashboard"))
        else:
            flash("Invalid email or password!", "error")
            return redirect(url_for("login"))

    return render_template("login.html")

# DASHBOARD
@app.route("/dashboard")
def dashboard():
    if "user_id" not in session:
        return redirect(url_for("login"))
    return render_template("dashboard.html", username=session["username"])

# LOGOUT
@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("login"))

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```
Create the enviroment for it. 

```bash
apt install python3.10-venv
cd /var/www/the_platform
python3 -m venv venv
source venv/bin/activate
```
Requiremetns.txt. 
```
pip freeze > requirements.txt
cat requirements.txt
blinker==1.9.0
click==8.3.1
Flask==3.1.2
itsdangerous==2.2.0
Jinja2==3.1.6
MarkupSafe==3.0.3
mysql-connector-python==9.5.0
Werkzeug==3.1.3
```
For it to work the platform structure should be. 
```
/var/www/the_platform/
├── app.py
├── templates/
│   ├── dashboard.html
│   ├── login.html
│   └── register.html
└── static/
    └── js/
```bash
cd /var/www/the_platform
mkdir templates
mv static/html/*.html templates/
```
DB managment
```msyql
USE platform_db;
DESCRIBE users;
``` 











