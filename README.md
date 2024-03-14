# API-for-a-vending-machine-
#API for a vending machine, allowing users with a “seller” role to add, update or remove products, while users with a “buyer” role can deposit coins into the machine and make purchases. Your vending machine should only accept 5, 10, 20, 50 and 100 cent coins
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash

# Initialize Flask app
app = Flask(__name__)

# Database configuration
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///vending_machine.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# User model definition
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password_hash = db.Column(db.String(128))
    deposit = db.Column(db.Integer, default=0)
    role = db.Column(db.String(10), nullable=False)  # Can be either 'buyer' or 'seller'
    # Method to hash the password
    def set_password(self, password):
        self.password_hash = generate_password_hash(password)
        # Method to verify the password
    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

# Product model definition
class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    amountAvailable = db.Column(db.Integer, nullable=False)
    cost = db.Column(db.Integer, nullable=False)  # Cost in cents
    productName = db.Column(db.String(80), nullable=False)
    sellerId = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)

# Route to create a new user
@app.route('/users', methods=['POST'])
def create_user():
    data = request.get_json()
    user = User(username=data['username'], role=data['role'])
    user.set_password(data['password'])
    db.session.add(user)
    db.session.commit()
    return jsonify({'message': 'User created successfully'}), 201

# Route to get user details
@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = User.query.get_or_404(user_id)
    return jsonify({
        'username': user.username, 
        'role': user.role, 
        'deposit': user.deposit
    })

# Route to update user details
@app.route('/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    user = User.query.get_or_404(user_id)
    data = request.get_json()
    if 'password' in data:
        user.set_password(data['password'])
    user.deposit = data.get('deposit', user.deposit)
    user.role = data.get('role', user.role)
    db.session.commit()
    return jsonify({'message': 'User updated successfully'})

# Route to delete a user
@app.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    user = User.query.get_or_404(user_id)
    db.session.delete(user)
    db.session.commit()
    return jsonify({'message': 'User deleted successfully'})

# Route to add a new product
@app.route('/products', methods=['POST'])
def add_product():
    data = request.get_json()
    user = User.query.filter_by(id=data['sellerId']).first()
    if not user or user.role != 'seller':
        return jsonify({'message': 'Unauthorized'}), 403
    new_product = Product(amountAvailable=data['amountAvailable'], cost=data['cost'],
                          productName=data['productName'], sellerId=user.id)
    db.session.add(new_product)
    db.session.commit()
    return jsonify({'message': 'Product added successfully'}), 201

# Route to update product details
@app.route('/products/<int:product_id>', methods=['PUT'])
def update_product(product_id):
    product = Product.query.get_or_404(product_id)
    data = request.get_json()
    user = User.query.get(product.sellerId)
    if 'sellerId' in data and data['sellerId'] != user.id:
        return jsonify({'message': 'Unauthorized'}), 403
    product.amountAvailable = data.get('amountAvailable', product.amountAvailable)
    product.cost = data.get('cost', product.cost)
    product.productName = data.get('productName', product.productName)
    db.session.commit()
    return jsonify({'message': 'Product updated successfully'})

# Route to delete a product
@app.route('/products/<int:product_id>', methods=['DELETE'])
def delete_product(product_id):
    data = request.get_json()
    user = User.query.filter_by(id=data['sellerId']).first()
    product = Product.query.get_or_404(product_id)
    if product.sellerId != user.id or user.role != 'seller':
        return jsonify({'message': 'Unauthorized'}), 403
    db.session.delete(product)
    db.session.commit()
    return jsonify({'message': 'Product deleted successfully'})

# Route to get a list of all products
@app.route('/products', methods=['GET'])
def get_products():
    all_products = Product.query.all()
    output = []
    for product in all_products:
        product_data = {
            'id': product.id,
            'amountAvailable': product.amountAvailable,
            'cost': product.cost,
            'productName': product.productName,
            'sellerId': product.sellerId
        }
        output.append(product_data)
    return jsonify({'products': output})

# Accepted coin values
ACCEPTABLE_COINS = [5, 10, 20, 50, 100]

# Route for buyers to deposit coins
@app.route('/deposit/<int:user_id>', methods=['POST'])
def deposit(user_id):
    user = User.query.get_or_404(user_id)
    if user.role != 'buyer':
        return jsonify({'message': 'Only buyers can deposit coins'}), 403
    data = request.get_json()
    coin = data['coin']
    if coin not in ACCEPTABLE_COINS:
        return jsonify({'message': 'Invalid coin'}), 400
    user.deposit += coin
    db.session.commit()
    return jsonify({'message': 'Deposit successful', 'current_deposit': user.deposit})

# Route for buyers to make purchases
@app.route('/purchase/<int:user_id>', methods=['POST'])
def purchase(user_id):
    user = User.query.get_or_404(user_id)
    if user.role != 'buyer':
        return jsonify({'message': 'Only buyers can make purchases'}), 403
    data = request.get_json()
    product = Product.query.get_or_404(data['product_id'])
    if user.deposit < product.cost:
        return jsonify({'message': 'Insufficient balance'}), 400
    if product.amountAvailable <= 0:
        return jsonify({'message': 'Product not available'}), 400
    user.deposit -= product.cost
    product.amountAvailable -= 1
    db.session.commit()
    return jsonify({
        'message': 'Purchase successful',
        'product': product.productName,
        'cost': product.cost,
        'remaining_balance': user.deposit
    })

# Main entry point for running the Flask app
if __name__ == '__main__':
    with app.app_context():
        # Ensure all database tables are created
        db.create_all()
    # Start the Flask application
    app.run(debug=True)
