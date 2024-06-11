For a scenario where different users have their own accounts and can only access and modify their own data, handling user authentication and authorization is essential. Using API keys for individual users is not ideal for this purpose because it lacks flexibility and security features provided by modern authentication methods.

Instead, you can implement user authentication using JWT (JSON Web Tokens) or session-based authentication. Here’s a high-level overview of how you can handle this:

1. **User Registration and Login**:
   - Users can register with their credentials (username, email, password, etc.).
   - Upon successful login, a JWT token or session is created and sent back to the client.

2. **Protected Routes**:
   - API routes are protected using middleware that checks for a valid JWT token or session.
   - Only authenticated users can access their data.

3. **Authorization**:
   - Ensure that users can only access and modify their own data.

### Step-by-Step Implementation

#### 1. User Registration and Login

**Install necessary packages:**

```sh
npm install bcrypt jsonwebtoken
```

**User model (example with a SQL database):**

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**API route for user registration (`pages/api/register.js`):**

```javascript
import { Pool } from 'pg';
import bcrypt from 'bcrypt';

const pool = new Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  port: process.env.DB_PORT,
});

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    res.setHeader('Allow', ['POST']);
    return res.status(405).end(`Method ${req.method} Not Allowed`);
  }

  const { username, email, password } = req.body;

  try {
    const hashedPassword = await bcrypt.hash(password, 10);
    const client = await pool.connect();
    const result = await client.query(
      'INSERT INTO users (username, email, password) VALUES ($1, $2, $3) RETURNING id',
      [username, email, hashedPassword]
    );
    client.release();

    res.status(201).json({ id: result.rows[0].id });
  } catch (err) {
    console.error('Registration error:', err);
    res.status(500).json({ error: 'Internal Server Error' });
  }
}
```

**API route for user login (`pages/api/login.js`):**

```javascript
import { Pool } from 'pg';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

const pool = new Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  port: process.env.DB_PORT,
});

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    res.setHeader('Allow', ['POST']);
    return res.status(405).end(`Method ${req.method} Not Allowed`);
  }

  const { email, password } = req.body;

  try {
    const client = await pool.connect();
    const result = await client.query('SELECT * FROM users WHERE email = $1', [email]);
    client.release();

    if (result.rows.length === 0) {
      return res.status(401).json({ error: 'Invalid email or password' });
    }

    const user = result.rows[0];
    const isValidPassword = await bcrypt.compare(password, user.password);

    if (!isValidPassword) {
      return res.status(401).json({ error: 'Invalid email or password' });
    }

    const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET, { expiresIn: '1h' });

    res.status(200).json({ token });
  } catch (err) {
    console.error('Login error:', err);
    res.status(500).json({ error: 'Internal Server Error' });
  }
}
```

#### 2. Protected Routes

**Middleware for JWT authentication (`middleware/auth.js`):**

```javascript
import jwt from 'jsonwebtoken';

export function authenticateToken(handler) {
  return async (req, res) => {
    const token = req.headers['authorization']?.split(' ')[1];

    if (!token) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req.user = decoded;
      return handler(req, res);
    } catch (err) {
      console.error('JWT error:', err);
      return res.status(403).json({ error: 'Forbidden' });
    }
  };
}
```

**Protected API route (`pages/api/user-todo.js`):**

```javascript
import { Pool } from 'pg';
import { authenticateToken } from '../../middleware/auth';

const pool = new Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  port: process.env.DB_PORT,
});

async function handler(req, res) {
  if (req.method === 'GET') {
    try {
      const client = await pool.connect();
      const result = await client.query('SELECT * FROM todos WHERE user_id = $1', [req.user.userId]);
      client.release();

      res.status(200).json({ todos: result.rows });
    } catch (err) {
      console.error('Database query error:', err);
      res.status(500).json({ error: 'Internal Server Error' });
    }
  } else {
    res.setHeader('Allow', ['GET']);
    res.status(405).end(`Method ${req.method} Not Allowed`);
  }
}

export default authenticateToken(handler);
```

**Client-side example for fetching user-specific todos:**

```javascript
async function FetchUserTodos() {
  try {
    const token = 'your-jwt-token'; // Store this securely, e.g., in a cookie or local storage
    const response = await fetch('/api/user-todo', {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    });

    if (!response.ok) {
      throw new Error('Network response was not ok');
    }

    const data = await response.json();
    console.log(data); // { todos: [...] }
  } catch (error) {
    console.error('Fetch error:', error);
  }
}

FetchUserTodos();
```

### Summary:
- **User Authentication**: Implement user registration and login with JWT.
- **Middleware**: Protect routes using JWT middleware.
- **Authorization**: Ensure users can only access their own data.

This approach ensures secure user authentication and authorization, allowing users to interact with their data securely. 🚀🔒