
# FoodOrderingWebsite
Here's the updated `README.md` with the addition that User1's cart is isolated from User2's cart, reflecting the fact that carts are tied to individual user IDs via JWT authentication. This ensures data separation between users.

---
# Chatoree Frontend

A basic static frontend for interacting with the Zomato Backend API.
```
## File Structure
zomato-frontend/
├── index.html         # Main page with Add Item form and Cart display
├── styles.css         # Custom CSS
├── script.js          # JavaScript logic for API interaction
├── README.md          # This file
```

## Setup
1. Serve the frontend:
   ```bash
   cd chatoree-frontend
   npx serve -s .
---
# Zomato Backend API

This is a backend API for a Zomato-like food ordering system built with Node.js, Express, and MongoDB (via Mongoose). It includes cart management functionality allowing users to add, remove, and clear items from their cart, with specific logic to handle restaurant restrictions, data validation, and user-specific cart isolation.

## Features
- **Cart Management**: Add items to a cart, remove items, reduce quantities, and clear the cart.
- **Restaurant Restriction**: Ensures all items in a cart belong to the same restaurant.
- **Authentication**: Uses JWT for user authentication, ensuring each user's cart is isolated (e.g., User1's cart is not reflected in User2's cart).
- **Validation**: Strict validation of MongoDB `ObjectId`s and required fields.

## Installation
1. Clone the repository:
   ```bash
   git clone https://github.com/rv1304/FoodOrderingWebsite
   cd https://github.com/rv1304/FoodOrderingWebsite
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Set up environment variables in a `.env` file:
   ```plaintext
   MONGODB_URI=<your-mongodb-uri>
   JWT_SECRET=<your-jwt-secret>
   ```
4. Start the server:
   ```bash
   npm start
   ```
   Or use `nodemon` for development:
   ```bash
   npm run dev
   ```

## API Endpoints
- **GET /api/cart**: Retrieve the user’s cart.
- **POST /api/cart/add**: Add an item to the cart.
- **POST /api/cart/remove**: Remove an item from the cart by `dishId`.
- **POST /api/cart/clear**: Clear the entire cart.
- **POST /api/cart/reduce**: Reduce the quantity of an item (custom endpoint).

*Note*: All endpoints require an `Authorization` header with a Bearer token (JWT) unique to each user.

---

## Cart Schema
The cart is stored in MongoDB with the following schema:
```javascript
const cartSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, required: true },
  restaurantId: { type: mongoose.Schema.Types.ObjectId, required: true },
  items: [
    {
      dishId: { type: mongoose.Schema.Types.ObjectId, required: true },
      productId: { type: mongoose.Schema.Types.ObjectId, required: true }, // Can be made optional
      name: { type: String, required: true },
      price: { type: Number, required: true },
      quantity: { type: Number, required: true },
    },
  ],
  total: { type: Number, default: 0 },
});
```

---

## Corner Cases and Handling

Here are the key corner cases addressed in the cart management logic, including user isolation:

### 1. **User-Specific Cart Isolation**
- **Problem**: User1’s cart should not be visible or modifiable by User2, ensuring privacy and data integrity.
- **Solution**: Each cart is tied to a unique `userId` extracted from the JWT token. Queries (e.g., `Cart.findOne({ userId })`) filter by the authenticated user’s ID, preventing cross-user cart reflection.
- **Example**: 
  - User1 (token with `userId: "67bc579369ac049493656ee2"`) adds an item:
    ```bash
    curl -X POST http://localhost:5010/api/cart/add \
      -H "Authorization: Bearer <user1-token>" \
      -d '{"restaurantId": "507f1f77bcf86cd799439011", "dishId": "507f191e810c19729de860ea", "productId": "507f191e810c19729de860eb", "name": "Burger", "price": 10.5, "quantity": 1}'
    ```
  - User2 (token with a different `userId`) calls `GET /api/cart`:
    ```bash
    curl -X GET http://localhost:5010/api/cart \
      -H "Authorization: Bearer <user2-token>"
    ```
    Response (User2 sees their own cart, not User1’s):
    ```json
    { "message": "Cart is empty" }
    ```

### 2. **Invalid `ObjectId` Values**
- **Problem**: Request data (e.g., `restaurantId`, `dishId`, `productId`) might contain invalid strings (e.g., `"123"`) instead of valid 24-character hexadecimal `ObjectId`s.
- **Solution**: Validate all `ObjectId` fields using `mongoose.Types.ObjectId.isValid()`. If invalid, return a 400 error (e.g., `"Invalid restaurantId"`).
- **Example**:
  ```bash
  curl -X POST http://localhost:5010/api/cart/add \
    -H "Authorization: Bearer <token>" \
    -d '{"restaurantId": "123", "dishId": "456", "productId": "789", "name": "Burger", "price": 10.5, "quantity": 1}'
  ```
  Response:
  ```json
  { "message": "Invalid restaurantId" }
  ```

### 3. **Missing Required Fields (e.g., `productId`)**
- **Problem**: The schema requires `productId`, but the request might omit it.
- **Solution**: Check for `productId` in `addToCart`. If missing or invalid, return a 400 error. Optionally, make `productId` non-required in the schema.
- **Example (Required)**:
  ```bash
  curl -X POST http://localhost:5010/api/cart/add \
    -H "Authorization: Bearer <token>" \
    -d '{"restaurantId": "507f1f77bcf86cd799439011", "dishId": "507f191e810c19729de860ea", "name": "Burger", "price": 10.5, "quantity": 1}'
  ```
  Response:
  ```json
  { "message": "productId is required and must be a valid ObjectId" }
  ```

### 4. **Adding Items from Different Restaurants**
- **Problem**: Users might try to add items from a different restaurant to an existing cart.
- **Solution**: Check if the new `restaurantId` matches the cart’s `restaurantId`. If not, return a 400 error suggesting to clear the cart.
- **Example**:
  ```bash
  curl -X POST http://localhost:5010/api/cart/add \
    -H "Authorization: Bearer <token>" \
    -d '{"restaurantId": "507f1f77bcf86cd799439012", "dishId": "507f191e810c19729de860ec", "productId": "507f191e810c19729de860ed", "name": "Pizza", "price": 15, "quantity": 2}'
  ```
  Response (if cart has items from a different restaurant):
  ```json
  { "message": "You can only add items from one restaurant at a time. Clear your cart to add from another restaurant." }
  ```

### 5. **Empty Cart or Non-Existent Cart**
- **Problem**: Operations might be attempted on an empty or non-existent cart.
- **Solution**: Check if the cart exists in `removeFromCart`, `reduceFromCart`, and `getCart`. Return appropriate messages (e.g., `"Cart is empty"`).
- **Example**:
  ```bash
  curl -X POST http://localhost:5010/api/cart/remove \
    -H "Authorization: Bearer <token>" \
    -d '{"dishId": "507f191e810c19729de860ea"}'
  ```
  Response (if cart is empty):
  ```json
  { "message": "Cart is empty" }
  ```

### 6. **Reducing Quantity Below Zero**
- **Problem**: Reducing an item’s quantity might result in a negative or zero value.
- **Solution**: In `reduceFromCart`, remove the item if quantity drops to 0 or below. Delete the cart if it becomes empty.
- **Example**:
  ```bash
  curl -X POST http://localhost:5010/api/cart/reduce \
    -H "Authorization: Bearer <token>" \
    -d '{"dishId": "507f191e810c19729de860ea"}'
  ```
  If quantity becomes 0:
  ```json
  { "message": "Cart cleared" }
  ```

### 7. **Duplicate Items in Cart**
- **Problem**: Adding the same `dishId` multiple times could create duplicates.
- **Solution**: In `addToCart`, increment the `quantity` of an existing item instead of adding a duplicate.
- **Example**:
  ```bash
  curl -X POST http://localhost:5010/api/cart/add \
    -H "Authorization: Bearer <token>" \
    -d '{"restaurantId": "507f1f77bcf86cd799439011", "dishId": "507f191e810c19729de860ea", "productId": "507f191e810c19729de860eb", "name": "Burger", "price": 10.5, "quantity": 1}'
  ```
  Updated cart:
  ```json
  {
    "items": [{ "dishId": "507f191e810c19729de860ea", "quantity": 2, ... }],
    "total": 21.0
  }
  ```

### 8. **Authentication Failures**
- **Problem**: Invalid or expired JWT tokens could block access.
- **Solution**: Middleware verifies the token, returning a 401 error if invalid.
- **Example**:
  ```bash
  curl -X POST http://localhost:5010/api/cart/add \
    -H "Authorization: Bearer <invalid-token>" \
    -d '{}'
  ```
  Response:
  ```json
  { "message": "Unauthorized" }
  ```

---

## Optional `productId` Configuration
To make `productId` optional:
1. Update the schema:
   ```javascript
   productId: { type: mongoose.Schema.Types.ObjectId, required: false }
   ```
2. Modify `addToCart` validation:
   ```javascript
   if (productId && !mongoose.Types.ObjectId.isValid(productId)) {
     return res.status(400).json({ message: "Invalid productId" });
   }
   const productObjectId = productId ? new mongoose.Types.ObjectId(productId) : null;
   ```
3. Test with:
   ```bash
   curl -X POST http://localhost:5010/api/cart/add \
     -H "Authorization: Bearer <token>" \
     -d '{"restaurantId": "507f1f77bcf86cd799439011", "dishId": "507f191e810c19729de860ea", "name": "Burger", "price": 10.5, "quantity": 1}'
   ```

---

## Usage Examples
### Add Item
```bash
curl -X POST http://localhost:5010/api/cart/add \
  -H "Authorization: Bearer <token>" \
  -d '{"restaurantId": "507f1f77bcf86cd799439011", "dishId": "507f191e810c19729de860ea", "productId": "507f191e810c19729de860eb", "name": "Burger", "price": 10.5, "quantity": 1}'
```

### Reduce Quantity
```bash
curl -X POST http://localhost:5010/api/cart/reduce \
  -H "Authorization: Bearer <token>" \
  -d '{"dishId": "507f191e810c19729de860ea"}'
```

### Clear Cart
```bash
curl -X POST http://localhost:5010/api/cart/clear \
  -H "Authorization: Bearer <token>" \
  -d '{}'
```

---

## Contributing
Submit issues or pull requests for features like multiple restaurants per cart or order history.

## License
MIT License

---
