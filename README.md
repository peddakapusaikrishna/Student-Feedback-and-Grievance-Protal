# Student Feedback & Grievance Portal (Improved)
# Technology: Python + MySQL (mysql-connector-python) + bcrypt

import mysql.connector
import getpass
import bcrypt
import os
import platform

# --- Configuration ---
# It's better to load these from a config file or environment variables
DB_CONFIG = {
    'host': "localhost",
    'user': "root",
    'password': "root", # Replace with your MySQL password
    'database': "feedback_portal"
}

# --- Utility Functions ---
def clear_screen():
    """Clears the console screen."""
    command = 'cls' if platform.system() == 'Windows' else 'clear'
    os.system(command)

def press_enter_to_continue():
    """Pauses execution until the user presses Enter."""
    input("\nPress Enter to continue...")

def hash_password(password):
    """Hashes a password for storing."""
    # Note: password must be encoded to bytes for hashing
    return bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())

def check_password(hashed_password, user_password):
    """Verifies a password against a hash."""
    # Note: Both arguments must be encoded to bytes
    return bcrypt.checkpw(user_password.encode('utf-8'), hashed_password.encode('utf-8'))

# --- Database Management ---
def connect_db():
    """Establishes a connection to the database."""
    try:
        return mysql.connector.connect(**DB_CONFIG)
    except mysql.connector.Error as err:
        if err.errno == 1049: # Error code for 'Unknown database'
            print(f"Database '{DB_CONFIG['database']}' does not exist.")
            # Optional: Add logic here to create the database
        else:
            print(f"Database connection error: {err}")
        return None

def init_db():
    """Initializes the database schema and creates tables if they don't exist."""
    # Connect without specifying a database to create it first
    try:
        temp_conn = mysql.connector.connect(
            host=DB_CONFIG['host'],
            user=DB_CONFIG['user'],
            password=DB_CONFIG['password']
        )
        cursor = temp_conn.cursor()
        cursor.execute(f"CREATE DATABASE IF NOT EXISTS {DB_CONFIG['database']}")
        temp_conn.close()
    except mysql.connector.Error as err:
        print(f"Failed to create database: {err}")
        exit() # Exit if we can't create/connect to the DB

    conn = connect_db()
    if not conn:
        exit() # Exit if connection failed after creation attempt
        
    with conn.cursor() as cursor:
        cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            username VARCHAR(50) UNIQUE NOT NULL,
            password VARCHAR(255) NOT NULL,
            role ENUM('admin', 'student') NOT NULL
        )""")
        cursor.execute("""
        CREATE TABLE IF NOT EXISTS feedback (
            id INT AUTO_INCREMENT PRIMARY KEY,
            user_id INT,
            department VARCHAR(50) NOT NULL,
            message TEXT NOT NULL,
            status ENUM('Pending', 'In Progress', 'Resolved') DEFAULT 'Pending',
            response TEXT,
            is_anonymous BOOLEAN DEFAULT FALSE,
            FOREIGN KEY (user_id) REFERENCES users(id)
        )""")

        # --- Create a default admin if one doesn't exist ---
        cursor.execute("SELECT id FROM users WHERE username = 'admin'")
        if cursor.fetchone() is None:
            hashed_admin_pass = hash_password('admin123')
            cursor.execute(
                "INSERT INTO users (username, password, role) VALUES (%s, %s, %s)",
                ('admin', hashed_admin_pass, 'admin')
            )
        conn.commit()
    conn.close()

# --- User Authentication Functions ---
def register():
    """Handles new user registration."""
    clear_screen()
    print("--- Student Registration ---")
    username = input("Enter a username: ")
    password = getpass.getpass("Enter a password: ")
    
    if not username or not password:
        print("Username and password cannot be empty.")
        return

    hashed_pw = hash_password(password)
    
    conn = connect_db()
    if not conn: return

    try:
        with conn.cursor() as cursor:
            cursor.execute(
                "INSERT INTO users (username, password, role) VALUES (%s, %s, 'student')",
                (username, hashed_pw)
            )
            conn.commit()
            print("\nRegistration successful! You can now log in.")
    except mysql.connector.IntegrityError:
        print("\nUsername already exists. Please choose another one.")
    except mysql.connector.Error as err:
        print(f"Database error during registration: {err}")
    finally:
        conn.close()

def login():
    """Handles user login and returns user info if successful."""
    clear_screen()
    print("--- Login ---")
    username = input("Username: ")
    password = getpass.getpass("Password: ")
    
    conn = connect_db()
    if not conn: return None, None

    user_data = None
    try:
        with conn.cursor() as cursor:
            cursor.execute("SELECT id, password, role FROM users WHERE username = %s", (username,))
            user_data = cursor.fetchone()
    except mysql.connector.Error as err:
        print(f"Database error during login: {err}")
    finally:
        conn.close()

    if user_data:
        user_id, hashed_password, role = user_data
        # Note: The stored hash might be bytes, so decode for check_password
        if check_password(hashed_password, password):
            print("\nLogin successful!")
            return {'id': user_id, 'username': username, 'role': role}
    
    print("\nInvalid username or password.")
    return None

# --- Student Specific Functions ---
def submit_feedback(user):
    """Allows a logged-in student to submit feedback."""
    clear_screen()
    print("--- Submit Feedback ---")
    department = input("Enter Department (e.g., Academics, Hostel, Sports): ")
    message = input("Enter Feedback/Grievance: ")
    anonymous_choice = input("Submit anonymously? (y/n): ").lower()
    
    is_anonymous = True if anonymous_choice == 'y' else False

    conn = connect_db()
    if not conn: return

    try:
        with conn.cursor() as cursor:
            query = """
            INSERT INTO feedback (user_id, department, message, is_anonymous) 
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, (user['id'], department, message, is_anonymous))
            conn.commit()
            print("\nFeedback submitted successfully.")
    except mysql.connector.Error as err:
        print(f"Failed to submit feedback: {err}")
    finally:
        conn.close()

def view_my_feedback(user):
    """Displays all feedback submitted by the current user."""
    clear_screen()
    print("--- Your Submitted Feedback ---")
    conn = connect_db()
    if not conn: return

    try:
        with conn.cursor(dictionary=True) as cursor: # dictionary=True is very useful!
            cursor.execute("SELECT * FROM feedback WHERE user_id = %s ORDER BY id DESC", (user['id'],))
            records = cursor.fetchall()
            if not records:
                print("You have not submitted any feedback yet.")
            else:
                for row in records:
                    anon_status = "(Anonymous)" if row['is_anonymous'] else ""
                    print("-" * 20)
                    print(f"ID: {row['id']} {anon_status}")
                    print(f"Department: {row['department']}")
                    print(f"Status: {row['status']}")
                    print(f"Your Message: {row['message']}")
                    print(f"Admin Response: {row['response'] or 'No response yet.'}")
    except mysql.connector.Error as err:
        print(f"Failed to retrieve feedback: {err}")
    finally:
        conn.close()


# --- Admin Specific Functions ---
def view_all_feedback(conn):
    """Fetches and displays all feedback for the admin."""
    clear_screen()
    print("--- All User Feedback ---")
    try:
        with conn.cursor(dictionary=True) as cursor:
            # Join with users table to show who submitted it (if not anonymous)
            query = """
            SELECT f.id, f.department, f.message, f.status, f.response, f.is_anonymous, u.username
            FROM feedback f
            JOIN users u ON f.user_id = u.id
            ORDER BY f.id DESC
            """
            cursor.execute(query)
            rows = cursor.fetchall()
            if not rows:
                print("No feedback has been submitted yet.")
                return
            for row in rows:
                submitter = "Anonymous" if row['is_anonymous'] else row['username']
                print("-" * 30)
                print(f"ID: {row['id']} | Submitter: {submitter}")
                print(f"Department: {row['department']} | Status: {row['status']}")
                print(f"Message: {row['message']}")
                print(f"Response: {row['response'] if row['response'] else 'None'}")
    except mysql.connector.Error as err:
        print(f"Database error while fetching feedback: {err}")

def update_feedback_status(conn):
    """Allows an admin to update the status and add a response to feedback."""
    view_all_feedback(conn) # Show feedback first for context
    try:
        fid = int(input("\nEnter the Feedback ID to update: "))
    except ValueError:
        print("Invalid ID. Please enter a number.")
        return

    print("\nChoose new status: 1. In Progress  2. Resolved")
    status_choice = input("Enter choice (1 or 2): ")
    status_map = {'1': 'In Progress', '2': 'Resolved'}

    if status_choice not in status_map:
        print("Invalid status choice.")
        return

    new_status = status_map[status_choice]
    response = input("Enter response message (optional, press Enter to skip): ")

    try:
        with conn.cursor() as cursor:
            cursor.execute(
                "UPDATE feedback SET status = %s, response = %s WHERE id = %s",
                (new_status, response, fid)
            )
            # cursor.rowcount gives the number of rows affected.
            if cursor.rowcount > 0:
                print("Feedback updated successfully!")
            else:
                print("No feedback found with that ID.")
            conn.commit()
    except mysql.connector.Error as err:
        print(f"Database error during update: {err}")


# --- Menus ---
def student_menu(user):
    """Displays the menu for student users."""
    while True:
        clear_screen()
        print(f"--- Welcome, {user['username']} (Student) ---")
        print("1. Submit Feedback")
        print("2. View My Feedback")
        print("3. Logout")
        choice = input("Enter your choice: ")
        
        if choice == '1':
            submit_feedback(user)
            press_enter_to_continue()
        elif choice == '2':
            view_my_feedback(user)
            press_enter_to_continue()
        elif choice == '3':
            print("Logging out...")
            break
        else:
            print("Invalid option. Please try again.")
            press_enter_to_continue()

def admin_menu(user):
    """Displays the menu for admin users."""
    conn = connect_db()
    if not conn:
        print("Could not connect to the database. Admin panel is unavailable.")
        return
        
    while True:
        clear_screen()
        print(f"--- Welcome, {user['username']} (Admin) ---")
        print("1. View All Feedback")
        print("2. Update Feedback Status")
        print("3. Logout")
        choice = input("Enter your choice: ")

        if choice == '1':
            view_all_feedback(conn)
            press_enter_to_continue()
        elif choice == '2':
            update_feedback_status(conn)
            press_enter_to_continue()
        elif choice == '3':
            print("Logging out...")
            break
        else:
            print("Invalid option. Please try again.")
            press_enter_to_continue()
    
    conn.close()

# ------------------ MAIN APPLICATION FLOW ------------------
def main():
    """The main function to run the application."""
    init_db() # Ensure DB and tables exist
    while True:
        clear_screen()
        print("=== Student Feedback & Grievance Portal ===")
        print("1. Login")
        print("2. Register (Students only)")
        print("3. Exit")
        choice = input("Enter choice: ")

        if choice == '1':
            user = login()
            if user:
                press_enter_to_continue()
                if user['role'] == 'student':
                    student_menu(user)
                elif user['role'] == 'admin':
                    admin_menu(user)
        elif choice == '2':
            register()
            press_enter_to_continue()
        elif choice == '3':
            print("Thank you for using the portal. Goodbye!")
            break
        else:
            print("Invalid choice, please try again.")
            press_enter_to_continue()


if __name__ == "__main__":
    main()

