# Blogs  
**A Blog app built using the MERN stack** – a full‑stack JavaScript application that lets users sign‑in, create, edit, and delete their own blog posts.

---

## Table of Contents
1. [Features](#features)  
2. [Prerequisites](#prerequisites)  
3. [Installation](#installation)  
4. [Running the Application](#running-the-application)  
5. [Environment Variables](#environment-variables)  
6. [API Documentation](#api-documentation)  
7. [Frontend Usage](#frontend-usage)  
8. [Examples](#examples)  
9. [Testing](#testing)  
10. [Contributing](#contributing)  
11. [License](#license)  

---

## Features
- **User authentication** (JWT‑based) – login & registration.  
- **CRUD** for blog posts (create, read, update, delete).  
- **Protected routes** – only the author can edit/delete their posts.  
- **Responsive UI** built with React + Tailwind CSS.  
- **RESTful API** built with Express.js & MongoDB (Mongoose).  
- **Docker support** for easy local development and CI/CD.

---

## Prerequisites
| Tool | Minimum Version |
|------|-----------------|
| Node.js | 18.x |
| npm or yarn | 9.x |
| Docker (optional) | 24.x |
| MongoDB | 5.x (or use the provided Docker container) |
| Git | 2.30+ |

---

## Installation

### 1️⃣ Clone the repository
```bash
git clone https://github.com/your-username/Blogs.git
cd Blogs
```

### 2️⃣ Install server dependencies
```bash
cd server
npm ci   # or `yarn install --frozen-lockfile`
```

### 3️⃣ Install client dependencies
```bash
cd ../client
npm ci   # or `yarn install --frozen-lockfile`
```

### 4️⃣ (Optional) Run with Docker
If you prefer containerised development, a `docker-compose.yml` is provided.

```bash
docker compose up --build
# The API will be available at http://localhost:5000
# The React client will be available at http://localhost:3000
```

> **Tip:** Docker automatically creates a MongoDB container (`mongo`) and seeds it with a test user (`test@example.com` / `password123`).

---

## Running the Application

### Development (hot‑reload)

#### Server
```bash
cd server
npm run dev   # uses nodemon, watches for changes
```

#### Client
```bash
cd client
npm start     # runs `react-scripts start`
```

The client proxies API requests to `http://localhost:5000/api`.

### Production (static build)

```bash
# Build the client
cd client
npm run build

# Serve the build with the Express server
cd ../server
npm start   # runs `node index.js`
```

The server will serve the React static files from `client/build`.

---

## Environment Variables

Create a `.env` file in the **server** directory (copy from `.env.example`).

| Variable | Description | Example |
|----------|-------------|---------|
| `PORT` | Port for Express server | `5000` |
| `MONGO_URI` | MongoDB connection string | `mongodb://mongo:27017/blogs` (Docker) |
| `JWT_SECRET` | Secret key for signing JWTs | `superSecretKey123` |
| `CLIENT_URL` | URL of the React client (used for CORS) | `http://localhost:3000` |
| `NODE_ENV` | `development` or `production` | `development` |

> **Security note:** Never commit the real `.env` file. Add it to `.gitignore` (already done).

---

## API Documentation

All endpoints are prefixed with `/api`.  
Responses are JSON. Errors follow the shape:

```json
{
  "message": "Error description",
  "status": 400
}
```

### Authentication

| Method | Endpoint | Description | Body | Returns |
|--------|----------|-------------|------|---------|
| **POST** | `/api/auth/register` | Register a new user | `{ "name": "John", "email": "john@example.com", "password": "secret" }` | `{ "token": "...", "user": { "_id", "name", "email" } }` |
| **POST** | `/api/auth/login` | Login existing user | `{ "email": "john@example.com", "password": "secret" }` | Same as register |
| **GET** | `/api/auth/me` | Get current user (protected) | – | `{ "_id", "name", "email" }` |

### Blog Posts

> **All blog routes require a valid JWT in the `Authorization: Bearer <token>` header.**

| Method | Endpoint | Description | Body | Returns |
|--------|----------|-------------|------|---------|
| **GET** | `/api/posts` | List all public posts (paginated) | `?page=1&limit=10` | `{ posts: [...], total, page, limit }` |
| **GET** | `/api/posts/:id` | Get a single post by ID | – | `{ post }` |
| **POST** | `/api/posts` | Create a new post (author = logged‑in user) | `{ "title": "My Post", "content": "Markdown…", "tags": ["react","node"] }` | `{ post }` |
| **PUT** | `/api/posts/:id` | Update a post (only author) | Same shape as POST | `{ post }` |
| **DELETE** | `/api/posts/:id` | Delete a post (only author) | – | `{ message: "Post deleted" }` |

### Example Request (cURL)

```bash
# Register
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","password":"password123"}'

# Create a post (replace <TOKEN>)
curl -X POST http://localhost:5000/api/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"title":"Hello MERN","content":"# My first post\nHello world!","tags":["mern","blog"]}'
```

---

## Frontend Usage

### Routing Overview (React Router v6)

| Path | Component | Protected? |
|------|-----------|------------|
| `/login` | `LoginPage` | No |
| `/register` | `RegisterPage` | No |
| `/` | `HomePage` (list of posts) | No |
| `/posts/:id` | `PostDetail` | No |
| `/dashboard` | `Dashboard` (my posts) | Yes |
| `/posts/new` | `PostForm` (create) | Yes |
| `/posts/:id/edit` | `PostForm` (edit) | Yes |

### State Management
- **Context API** (`AuthContext`) stores the JWT and user data.  
- **React Query** (`@tanstack/react-query`) handles data fetching, caching, and optimistic updates.

### Styling
- Tailwind CSS (utility‑first) – edit `tailwind.config.js` for custom colors or fonts.

### Example Component (Create Post)

```tsx
import { useState } from 'react';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import axios from 'axios';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function PostForm() {
  const { token } = useAuth();
  const navigate = useNavigate();
  const queryClient = useQueryClient();

  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [tags, setTags] = useState<string[]>([]);

  const createPost = useMutation(
    (newPost) =>
      axios.post('/api/posts', newPost, {
        headers: { Authorization: `Bearer ${token}` },
      }),
    {
      onSuccess: () => {
        queryClient.invalidateQueries(['posts']);
        navigate('/dashboard');
      },
    }
  );

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    createPost.mutate({ title, content, tags });
  };

  return (
    <form onSubmit={handleSubmit} className="max-w-xl mx-auto p-4">
      <h2 className="text-2xl mb-4">Create New Blog Post</h2>

      <label className="block mb-2">
        Title
        <input
          className="w-full border rounded p-2"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          required
        />
      </label>

      <label className="block mb-2">
        Content (Markdown)
        <textarea
          className="w-full border rounded p-2 h-48"
          value={content}
          onChange