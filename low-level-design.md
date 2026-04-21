# Low-Level Design: Food Delivery System (Zomato/Swiggy)

## Database Schema

### 1. Users Table
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role ENUM('customer', 'restaurant_owner', 'delivery_partner', 'admin') DEFAULT 'customer',
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_phone (phone)
);

CREATE TABLE restaurants (
    restaurant_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    cuisine_type VARCHAR(100),
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    pincode VARCHAR(10) NOT NULL,
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(150),
    opening_time TIME,
    closing_time TIME,
    is_active BOOLEAN DEFAULT TRUE,
    rating DECIMAL(3, 2) DEFAULT 0.00,
    total_ratings INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (owner_id) REFERENCES users(user_id) ON DELETE CASCADE,
    INDEX idx_location (latitude, longitude),
    INDEX idx_city_pincode (city, pincode),
    INDEX idx_is_active (is_active)
);

CREATE TABLE menu_items (
    item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id UUID NOT NULL,
    name VARCHAR(150) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    price DECIMAL(10, 2) NOT NULL,
    discount_percentage DECIMAL(5, 2) DEFAULT 0.00,
    is_available BOOLEAN DEFAULT TRUE,
    preparation_time_minutes INT DEFAULT 15,
    image_url TEXT,
    spice_level ENUM('mild', 'medium', 'hot') DEFAULT 'medium',
    is_veg BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id) ON DELETE CASCADE,
    INDEX idx_restaurant (restaurant_id),
    INDEX idx_availability (is_available)
);

CREATE TABLE orders (
    order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,
    restaurant_id UUID NOT NULL,
    delivery_partner_id UUID,
    order_status ENUM('pending', 'confirmed', 'preparing', 'ready_for_pickup', 'picked_up', 'on_the_way', 'delivered', 'cancelled', 'refunded') DEFAULT 'pending',
    payment_status ENUM('pending', 'paid', 'failed', 'refunded') DEFAULT 'pending',
    payment_method ENUM('credit_card', 'debit_card', 'upi', 'cash', 'wallet'),
    subtotal DECIMAL(10, 2) NOT NULL,
    tax_amount DECIMAL(10, 2) DEFAULT 0.00,
    delivery_fee DECIMAL(10, 2) DEFAULT 0.00,
    discount_amount DECIMAL(10, 2) DEFAULT 0.00,
    total_amount DECIMAL(10, 2) NOT NULL,
    customer_address TEXT NOT NULL,
    customer_latitude DECIMAL(10, 8),
    customer_longitude DECIMAL(11, 8),
    special_instructions TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    delivered_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES users(user_id),
    FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id),
    FOREIGN KEY (delivery_partner_id) REFERENCES users(user_id),
    INDEX idx_customer (customer_id),
    INDEX idx_restaurant (restaurant_id),
    INDEX idx_status (order_status),
    INDEX idx_created_at (created_at)
);

CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    item_id UUID NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL,
    total_price DECIMAL(10, 2) NOT NULL,
    special_requests TEXT,
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (item_id) REFERENCES menu_items(item_id),
    INDEX idx_order (order_id)
);

CREATE TABLE delivery_tracking (
    tracking_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL UNIQUE,
    delivery_partner_id UUID NOT NULL,
    current_latitude DECIMAL(10, 8) NOT NULL,
    current_longitude DECIMAL(11, 8) NOT NULL,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    estimated_delivery_time TIMESTAMP,
    distance_to_restaurant_km DECIMAL(8, 2),
    distance_to_customer_km DECIMAL(8, 2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (delivery_partner_id) REFERENCES users(user_id),
    INDEX idx_partner (delivery_partner_id),
    INDEX idx_last_updated (last_updated)
);

CREATE TABLE reviews (
    review_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    user_id UUID NOT NULL,
    restaurant_id UUID NOT NULL,
    rating INT NOT NULL CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id),
    INDEX idx_restaurant (restaurant_id),
    INDEX idx_user (user_id)
);

CREATE TABLE payment_transactions (
    transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    payment_gateway VARCHAR(50),
    gateway_transaction_id VARCHAR(255),
    status ENUM('initiated', 'success', 'failed', 'refunded') DEFAULT 'initiated',
    failure_reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    INDEX idx_order (order_id),
    INDEX idx_gateway (gateway_transaction_id)
);
```
####2.Class Entitites
```python
from enum import Enum
from datetime import datetime
from uuid import UUID
from typing import List, Optional

# Enums
class UserRole(Enum):
    CUSTOMER = "customer"
    RESTAURANT_OWNER = "restaurant_owner"
    DELIVERY_PARTNER = "delivery_partner"
    ADMIN = "admin"

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    PREPARING = "preparing"
    READY_FOR_PICKUP = "ready_for_pickup"
    PICKED_UP = "picked_up"
    ON_THE_WAY = "on_the_way"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class PaymentStatus(Enum):
    PENDING = "pending"
    PAID = "paid"
    FAILED = "failed"
    REFUNDED = "refunded"

# User Class
class User:
    def __init__(self, user_id: UUID, full_name: str, email: str, phone: str, role: UserRole):
        self.user_id = user_id
        self.full_name = full_name
        self.email = email
        self.phone = phone
        self.role = role
        self.is_verified = False
        self.created_at = datetime.now()
        self.last_login = None
    
    def verify_user(self):
        self.is_verified = True
    
    def update_last_login(self):
        self.last_login = datetime.now()

# Restaurant Class
class Restaurant:
    def __init__(self, restaurant_id: UUID, owner: User, name: str, latitude: float, longitude: float):
        self.restaurant_id = restaurant_id
        self.owner = owner
        self.name = name
        self.latitude = latitude
        self.longitude = longitude
        self.menu_items: List[MenuItem] = []
        self.is_active = True
        self.rating = 0.0
        self.total_ratings = 0
        self.created_at = datetime.now()
    
    def add_menu_item(self, item: 'MenuItem'):
        self.menu_items.append(item)
    
    def update_rating(self, new_rating: int):
        total_score = self.rating * self.total_ratings + new_rating
        self.total_ratings += 1
        self.rating = total_score / self.total_ratings
    
    def close_restaurant(self):
        self.is_active = False

# MenuItem Class
class MenuItem:
    def __init__(self, item_id: UUID, name: str, price: float, category: str, is_veg: bool = True):
        self.item_id = item_id
        self.name = name
        self.price = price
        self.category = category
        self.is_veg = is_veg
        self.is_available = True
        self.discount_percentage = 0.0
        self.preparation_time_minutes = 15
    
    def apply_discount(self, percentage: float):
        self.discount_percentage = min(percentage, 50.0)  # Max 50% discount
    
    def update_availability(self, available: bool):
        self.is_available = available

# Order Class
class Order:
    def __init__(self, order_id: UUID, customer: User, restaurant: Restaurant):
        self.order_id = order_id
        self.customer = customer
        self.restaurant = restaurant
        self.items: List[OrderItem] = []
        self.delivery_partner: Optional[User] = None
        self.order_status = OrderStatus.PENDING
        self.payment_status = PaymentStatus.PENDING
        self.subtotal = 0.0
        self.tax_amount = 0.0
        self.delivery_fee = 0.0
        self.discount_amount = 0.0
        self.total_amount = 0.0
        self.customer_address = ""
        self.created_at = datetime.now()
        self.updated_at = datetime.now()
    
    def add_item(self, menu_item: MenuItem, quantity: int):
        order_item = OrderItem(menu_item, quantity)
        self.items.append(order_item)
        self._recalculate_total()
    
    def _recalculate_total(self):
        self.subtotal = sum(item.total_price for item in self.items)
        self.total_amount = self.subtotal + self.tax_amount + self.delivery_fee - self.discount_amount
    
    def update_status(self, new_status: OrderStatus):
        self.order_status = new_status
        self.updated_at = datetime.now()
    
    def assign_delivery_partner(self, partner: User):
        self.delivery_partner = partner
        self.delivery_partner_id = partner.user_id
    
    def confirm_payment(self):
        self.payment_status = PaymentStatus.PAID

# OrderItem Class
class OrderItem:
    def __init__(self, menu_item: MenuItem, quantity: int):
        self.menu_item = menu_item
        self.quantity = quantity
        self.unit_price = menu_item.price
        self.total_price = menu_item.price * quantity
        self.special_requests = ""

# DeliveryTracking Class
class DeliveryTracking:
    def __init__(self, order: Order, delivery_partner: User):
        self.order = order
        self.delivery_partner = delivery_partner
        self.current_latitude = 0.0
        self.current_longitude = 0.0
        self.last_updated = datetime.now()
        self.estimated_delivery_time = None
    
    def update_location(self, latitude: float, longitude: float):
        self.current_latitude = latitude
        self.current_longitude = longitude
        self.last_updated = datetime.now()
    
    def update_eta(self, eta_minutes: int):
        from datetime import timedelta
        self.estimated_delivery_time = datetime.now() + timedelta(minutes=eta_minutes)

# Review Class
class Review:
    def __init__(self, review_id: UUID, user: User, restaurant: Restaurant, rating: int, comment: str = ""):
        self.review_id = review_id
        self.user = user
        self.restaurant = restaurant
        self.rating = rating
        self.comment = comment
        self.created_at = datetime.now()

# Service Layer Classes
class LocationService:
    @staticmethod
    def calculate_distance(lat1: float, lon1: float, lat2: float, lon2: float) -> float:
        """Calculate distance in kilometers using Haversine formula"""
        from math import radians, sin, cos, sqrt, atan2
        R = 6371  # Earth's radius in km
        lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
        dlat = lat2 - lat1
        dlon = lon2 - lon1
        a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
        c = 2 * atan2(sqrt(a), sqrt(1-a))
        return R * c
    
    def find_nearby_restaurants(self, user_lat: float, user_lon: float, radius_km: float) -> List[Restaurant]:
        pass  # Implementation would query database with geospatial index

class OrderService:
    def create_order(self, customer: User, restaurant: Restaurant, items: List[tuple]) -> Order:
        order = Order(UUID.uuid4(), customer, restaurant)
        for menu_item, quantity in items:
            order.add_item(menu_item, quantity)
        return order
    
    def cancel_order(self, order: Order) -> bool:
        if order.order_status in [OrderStatus.PENDING, OrderStatus.CONFIRMED]:
            order.update_status(OrderStatus.CANCELLED)
            return True
        return False
    
    def track_order(self, order_id: UUID) -> DeliveryTracking:
        pass  # Fetch from delivery_tracking table

class PaymentService:
    def process_payment(self, order: Order, payment_method: str) -> bool:
        # Integrate with payment gateway (Razorpay, Stripe, etc.)
        pass
    
    def refund_payment(self, order: Order) -> bool:
        if order.payment_status == PaymentStatus.PAID:
            # Process refund
            order.payment_status = PaymentStatus.REFUNDED
            return True
        return False
```
