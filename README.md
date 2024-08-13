# API-Doc-Scissor-Project


### API Documentation for Scissor URL-Shortening Project

#### 1. **User Registration**
- **Endpoint:** `/api/auth/register`
- **Method:** POST
- **Description:** Registers a new user.
- **Request Body:**
  ```json
  {
    "username": "string",
    "email": "string",
    "password": "string"
  }
  ```
- **Response:**
  - **201 Created:**
    ```json
    {
      "token": "string"
    }
    ```
  - **400 Bad Request:** User already exists.
  - **500 Internal Server Error:** Server error.

#### 2. **User Login**
- **Endpoint:** `/api/auth/login`
- **Method:** POST
- **Description:** Logs in a user.
- **Request Body:**
  ```json
  {
    "email": "string",
    "password": "string"
  }
  ```
- **Response:**
  - **200 OK:**
    ```json
    {
      "token": "string"
    }
    ```
  - **400 Bad Request:** Invalid credentials.
  - **500 Internal Server Error:** Server error.

#### 3. **URL Shortening**
- **Endpoint:** `/api/urls/shorten`
- **Method:** POST
- **Description:** Shortens a given URL.
- **Request Body:**
  ```json
  {
    "originalUrl": "string",
    "customCode": "string",
    "expiresAt": "ISO8601 string"
  }
  ```
- **Response:**
  - **201 Created:**
    ```json
    {
      "shortUrl": "string",
      "originalUrl": "string"
    }
    ```
  - **400 Bad Request:** Invalid URL.
  - **500 Internal Server Error:** Server error.

#### 4. **URL Redirection**
- **Endpoint:** `/api/urls/:shortCode`
- **Method:** GET
- **Description:** Redirects to the original URL for a given short code.
- **Response:**
  - **302 Found:** Redirects to the original URL.
  - **404 Not Found:** URL not found.
  - **410 Gone:** URL has expired.
  - **500 Internal Server Error:** Server error.

#### 5. **URL Deletion**
- **Endpoint:** `/api/urls/delete/:shortCode`
- **Method:** DELETE
- **Description:** Deletes a URL with the given short code.
- **Response:**
  - **200 OK:**
    ```json
    {
      "message": "URL deleted successfully",
      "deletedUrl": {
        "originalUrl": "string",
        "shortUrl": "string"
      }
    }
    ```
  - **404 Not Found:** URL not found.
  - **500 Internal Server Error:** Server error.

#### 6. **User's URLs**
- **Endpoint:** `/api/urls/`
- **Method:** GET
- **Description:** Retrieves all URLs created by the authenticated user.
- **Response:**
  - **200 OK:**
    ```json
    [
      {
        "originalUrl": "string",
        "shortUrl": "string",
        "expiresAt": "ISO8601 string"
      }
    ]
    ```
  - **500 Internal Server Error:** Server error.

#### 7. **Track Clicks**
- **Endpoint:** `/api/analytics/clicks/:shortCode`
- **Method:** GET
- **Description:** Retrieves the number of clicks for a given short code.
- **Response:**
  - **200 OK:**
    ```json
    {
      "clicks": number
    }
    ```
  - **500 Internal Server Error:** Server error.

### Middleware
- **Auth Middleware**
  - **Location:** `src/middlewares/authMiddleware.ts`
  - **Purpose:** Verifies JWT token for protected routes.

### Models
- **User Model**
  - **Location:** `src/models/User.ts`
  - **Schema:**
    ```typescript
    import { Schema, model, Document } from 'mongoose';

    interface IUser extends Document {
      username: string;
      email: string;
      password: string;
    }

    const userSchema = new Schema<IUser>({
      username: { type: String, required: true },
      email: { type: String, required: true, unique: true },
      password: { type: String, required: true },
    });

    export default model<IUser>('User', userSchema);
    ```

- **URL Model**
  - **Location:** `src/models/Url.ts`
  - **Schema:**
    ```typescript
    import { Schema, model, Document } from 'mongoose';

    interface IUrl extends Document {
      originalUrl: string;
      shortCode: string;
      createdBy: Schema.Types.ObjectId;
      expiresAt: Date;
    }

    const urlSchema = new Schema<IUrl>({
      originalUrl: { type: String, required: true },
      shortCode: { type: String, required: true, unique: true },
      createdBy: { type: Schema.Types.ObjectId, ref: 'User', required: true },
      expiresAt: { type: Date },
    });

    export default model<IUrl>('Url', urlSchema);
    ```

### Services
- **Auth Service**
  - **Location:** `src/services/authService.ts`
  - **Functions:**
    ```typescript
    import bcrypt from 'bcrypt';
    import jwt from 'jsonwebtoken';
    import User from '../models/User';

    export const registerUser = async (username: string, email: string, password: string) => {
      const hashedPassword = await bcrypt.hash(password, 10);
      const user = new User({ username, email, password: hashedPassword });
      await user.save();
      return user;
    };

    export const loginUser = async (email: string, password: string) => {
      const user = await User.findOne({ email });
      if (!user || !(await bcrypt.compare(password, user.password))) {
        throw new Error('Invalid email or password');
      }
      const token = jwt.sign({ id: user._id, roles: user.roles }, process.env.JWT_SECRET!, { expiresIn: '1h' });
      return { user, token };
    };
    ```

- **URL Shortening Service**
  - **Location:** `src/services/urlShorteningService.ts`
  - **Functions:**
    ```typescript
    import { nanoid } from 'nanoid';
    import Url from '../models/Url';

    export const shortenUrl = async (originalUrl: string, customCode?: string, expiresAt?: Date) => {
      const shortCode = customCode || nanoid(8);
      const url = new Url({ originalUrl, shortCode, expiresAt });
      await url.save();
      return url;
    };
    ```

### Routes
- **Auth Routes**
  - **Location:** `src/routes/authRoutes.ts`
  - **Code:**
    ```typescript
    import express from 'express';
    import { register, login } from '../controllers/authController';

    const router = express.Router();

    router.post('/register', register);
    router.post('/login', login);

    export default router;
    ```

- **URL Routes**
  - **Location:** `src/routes/urlRoutes.ts`
  - **Code:**
    ```typescript
    import express from 'express';
    import Url from '../models/Url';
    import { shortenUrl } from '../services/urlShorteningService';
    import { authMiddleware } from '../middlewares/authMiddleware';

    const router = express.Router();

    router.post('/shorten', async (req, res) => {
      const { originalUrl, customCode, expiresAt } = req.body;

      try {
        if (!validUrl.isUri(originalUrl)) {
          return res.status(400).json({ error: 'Invalid URL' });
        }

        const shortUrl = await shortenUrl(originalUrl, customCode, expiresAt);
        res.status(201).json(shortUrl);
      } catch (error) {
        res.status(400).json({ error: error.message });
      }
    });

    router.get('/:shortCode', async (req, res) => {
      const { shortCode } = req.params;

      try {
        await trackClick(shortCode);

        const url = await Url.findOne({ shortCode });
        if (url) {
          if (url.expiresAt && new Date() > url.expiresAt) {
            return res.status(410).send('URL has expired');
          }
          res.redirect(url.originalUrl);
        } else {
          res.status(404).send('URL not found');
        }
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    router.delete('/delete/:shortCode', authMiddleware, async (req, res) => {
      const { shortCode } = req.params;

      try {
        const url = await Url.findOneAndDelete({ shortCode });
        if (!url) {
          return res.status(404).send('URL not found');
        }
        res.send('URL deleted');
      } catch (error) {
        res.status(500).send(error.message);
      }
    });

    export default router;
    ```

### Frontend
- **URL Analytics Component**
  - **Location:** `src/components/UrlAnalytics.tsx`
  - **Code:**
    ```typescript
    import React, { useEffect, useState } from 'react';
    import axios from 'axios';

    interface UrlAnalyticsProps {
      shortCode: string;
    }

    const UrlAnalytics: React.FC<UrlAnalyticsProps> = ({ shortCode }) => {
      const [click

