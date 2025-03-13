# open-source
Prototyping an open-source book lending system for libraries
This system consists of the following features:
~Users: Users can register, view available books, and borrow or return them.
~Books: The library has a collection of books. Each book has a title, author, and status (available/borrowed).
~Transactions: Borrowing and returning books is tracked in a history log.
Code Implementation:
1.Setup the Environment: Install the necessary packages:
pip install flask sqlite3
2.Create the Flask App:
app.py: The main application file.

from flask import Flask, request, jsonify, render_template
import sqlite3

app = Flask(__name__)

# Database setup (SQLite)
DATABASE = 'library.db'

def init_db():
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    # Create tables for users, books, and transactions
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS books (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            author TEXT NOT NULL,
            status TEXT NOT NULL DEFAULT 'available'
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL,
            password TEXT NOT NULL
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            book_id INTEGER,
            action TEXT,
            date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(user_id) REFERENCES users(id),
            FOREIGN KEY(book_id) REFERENCES books(id)
        )
    ''')
    conn.commit()
    conn.close()

# Route to view available books
@app.route('/books', methods=['GET'])
def get_books():
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM books WHERE status = 'available'")
    books = cursor.fetchall()
    conn.close()
    return jsonify(books)

# Route to borrow a book
@app.route('/borrow/<int:book_id>', methods=['POST'])
def borrow_book(book_id):
    user_id = request.json.get('user_id')
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM books WHERE id = ?", (book_id,))
    book = cursor.fetchone()

    if not book:
        return jsonify({"error": "Book not found"}), 404
    if book[3] == 'borrowed':
        return jsonify({"error": "Book is already borrowed"}), 400

    cursor.execute("UPDATE books SET status = 'borrowed' WHERE id = ?", (book_id,))
    cursor.execute("INSERT INTO transactions (user_id, book_id, action) VALUES (?, ?, 'borrow')", (user_id, book_id))
    conn.commit()
    conn.close()
    return jsonify({"message": f"Book '{book[1]}' borrowed successfully"}), 200

# Route to return a book
@app.route('/return/<int:book_id>', methods=['POST'])
def return_book(book_id):
    user_id = request.json.get('user_id')
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM books WHERE id = ?", (book_id,))
    book = cursor.fetchone()

    if not book:
        return jsonify({"error": "Book not found"}), 404
    if book[3] == 'available':
        return jsonify({"error": "This book was not borrowed"}), 400

    cursor.execute("UPDATE books SET status = 'available' WHERE id = ?", (book_id,))
    cursor.execute("INSERT INTO transactions (user_id, book_id, action) VALUES (?, ?, 'return')", (user_id, book_id))
    conn.commit()
    conn.close()
    return jsonify({"message": f"Book '{book[1]}' returned successfully"}), 200

# Route for user registration (just basic functionality for demo)
@app.route('/register', methods=['POST'])
def register_user():
    username = request.json.get('username')
    password = request.json.get('password')
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute("INSERT INTO users (username, password) VALUES (?, ?)", (username, password))
    conn.commit()
    conn.close()
    return jsonify({"message": "User registered successfully"}), 201

# Route to view transaction history
@app.route('/transactions', methods=['GET'])
def get_transactions():
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM transactions")
    transactions = cursor.fetchall()
    conn.close()
    return jsonify(transactions)

if __name__ == '__main__':
    init_db()
    app.run(debug=True)




