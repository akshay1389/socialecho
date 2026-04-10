# SocialEcho

A social networking platform with automated content moderation and context-based authentication.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Future Enhancements](#future-enhancements)
- [License](#license)

---

## Features

- **User Authentication and Authorization** -- JWT-based authentication with access and refresh tokens, managed through Passport.js.
- **Context-Based Authentication** -- Detects suspicious logins by evaluating user location (via geoip-lite), IP address, and device fingerprint. Contextual data is encrypted with AES (Crypto-JS) and stored in MongoDB. Users receive email alerts on unrecognized login attempts and can confirm or block devices.
- **Automated Content Moderation** -- Posts are screened in real time using multiple NLP services before publication:
  - Google Perspective API for toxicity, spam, profanity, and harassment detection.
  - TextRazor API for topic and content categorization.
  - Hugging Face BART-Large-MNLI (via the Interface API or the included Flask classifier server) for zero-shot content classification.
- **Community System** -- Users can join topic-based communities, each governed by its own moderation rules and category filters.
- **Post Management** -- Create, edit, delete, like, comment on, save, and share posts within communities. File and image uploads are supported via Multer.
- **User Profiles** -- Profile creation and editing, avatar uploads, and public profile pages.
- **Follow System** -- Follow and unfollow other users to curate a personalized feed.
- **Reporting and Manual Review** -- Users can report posts for guideline violations. Reported posts enter a review queue for moderators.
- **Role-Based Access Control** -- Three roles with distinct permissions:
  - **Admin** -- System-wide management including moderator assignment, community configuration, and API service toggling.
  - **Moderator** -- Community-level management and manual review of reported content.
  - **General User** -- Standard social networking capabilities.
- **Admin Dashboard** -- Manage moderators, communities, content moderation API configuration, and monitor platform activity.
- **Moderator Dashboard** -- Review reported posts, manage community-level settings.
- **Device Management** -- Users can view and revoke access for registered devices.
- **Email Notifications** -- Automated email delivery via Nodemailer for suspicious login alerts and email verification.
- **Rate Limiting** -- Request throttling via express-rate-limit to protect against abuse.
- **Search** -- Search for users and content across the platform.

---

## Tech Stack

### Frontend

| Technology | Purpose |
|---|---|
| React 18 | UI library |
| Redux / Redux Toolkit | Global state management |
| React Router v6 | Client-side routing |
| Tailwind CSS 3 | Utility-first styling |
| Axios | HTTP client |
| Headless UI / Heroicons | Accessible UI primitives and icons |
| jwt-decode | Client-side token inspection |

### Backend

| Technology | Purpose |
|---|---|
| Node.js / Express 4 | REST API server |
| MongoDB / Mongoose 6 | Database and ODM |
| Passport.js (JWT strategy) | Authentication middleware |
| JSON Web Tokens | Stateless auth tokens |
| Crypto-JS (AES) | Encryption of context data |
| bcrypt | Password hashing |
| Nodemailer | Transactional email |
| Multer | File upload handling |
| express-rate-limit | API rate limiting |
| express-validator | Request validation |
| geoip-lite | IP geolocation |
| Morgan | HTTP request logging |

### NLP / ML (Classifier Server)

| Technology | Purpose |
|---|---|
| Python 3 / Flask | Classifier API server |
| Hugging Face Transformers | NLP model loading and inference |
| facebook/bart-large-mnli | Zero-shot text classification model |
| PyTorch | Deep learning framework |
| Docker | Containerized deployment of the classifier |

### External APIs

| Service | Purpose |
|---|---|
| Google Perspective API | Toxicity and spam analysis |
| TextRazor API | Content categorization |
| Hugging Face Inference API | Cloud-hosted zero-shot classification (alternative to local Flask server) |

---

## Architecture

SocialEcho follows a three-tier architecture composed of independently deployable services:

```
+-----------------+         +------------------+         +---------------------+
|                 |  HTTP   |                  |  HTTP   |                     |
|   React Client  +-------->+  Express Server  +-------->+  Flask Classifier   |
|   (Port 3000)   |         |  (Port 4000)     |         |  (Port 5000)        |
|                 |         |                  |         |                     |
+-----------------+         +--------+---------+         +---------------------+
                                     |
                                     | Mongoose
                                     v
                              +------+-------+
                              |              |
                              |   MongoDB    |
                              |              |
                              +--------------+
```

**Request flow for content moderation:**

1. A user submits a post from the React client.
2. The Express server receives the request and passes the post content through the content analysis pipeline (`services/analyzeContent.js`).
3. The pipeline invokes configured NLP services (Perspective API for toxicity checks, and either TextRazor, Hugging Face Inference API, or the local Flask classifier for category classification).
4. The category filter service (`services/categoryFilterService.js`) validates the classified categories against the target community's allowed categories.
5. If the post passes all checks, it is saved to MongoDB. Otherwise, it is rejected or queued as a pending post for moderator review.

**Context-based authentication flow:**

1. On login, the server collects the user's IP address, geolocation, and device/user-agent information.
2. This context is encrypted using AES and compared against previously stored trusted contexts.
3. If the context is unrecognized, a verification email is sent to the user, and the login attempt is flagged as suspicious until confirmed.

---

## Project Structure

```
socialecho/
|-- client/                          # React frontend application
|   |-- public/                      # Static assets
|   |-- src/
|   |   |-- assets/                  # Images and static resources
|   |   |-- components/              # Reusable UI components
|   |   |   |-- admin/               # Admin dashboard components
|   |   |   |-- community/           # Community-related components
|   |   |   |-- form/                # Form components
|   |   |   |-- home/                # Home feed components
|   |   |   |-- loader/              # Loading indicators
|   |   |   |-- modals/              # Modal dialogs
|   |   |   |-- moderator/           # Moderator panel components
|   |   |   |-- post/                # Post display and interaction
|   |   |   |-- profile/             # User profile components
|   |   |   |-- shared/              # Common/shared components
|   |   |-- hooks/                   # Custom React hooks
|   |   |-- middlewares/             # Client-side middleware
|   |   |-- pages/                   # Route-level page components
|   |   |-- redux/                   # State management
|   |   |   |-- actions/             # Redux action creators
|   |   |   |-- api/                 # API service layer
|   |   |   |-- constants/           # Action type constants
|   |   |   |-- reducers/            # Redux reducers
|   |   |   |-- store.js             # Redux store configuration
|   |   |-- utils/                   # Utility functions
|   |   |-- routes.js                # Route definitions
|   |   |-- App.jsx                  # Root application component
|   |   |-- AppContainer.jsx         # App wrapper with providers
|   |   |-- PrivateRoute.jsx         # Auth-protected route wrapper
|   |   |-- index.jsx                # Application entry point
|   |   |-- index.css                # Global styles
|   |-- package.json
|   |-- tailwind.config.js
|   |-- postcss.config.js
|   |-- .env.example
|
|-- server/                          # Node.js/Express backend
|   |-- config/
|   |   |-- database.js              # MongoDB connection manager
|   |   |-- passport.js              # Passport JWT strategy
|   |-- controllers/
|   |   |-- admin.controller.js      # Admin operations
|   |   |-- auth.controller.js       # Authentication logic
|   |   |-- community.controller.js  # Community CRUD
|   |   |-- post.controller.js       # Post CRUD and interactions
|   |   |-- profile.controller.js    # Profile management
|   |   |-- search.controller.js     # Search functionality
|   |   |-- user.controller.js       # User operations
|   |-- data/
|   |   |-- communities.json         # Seed data for communities
|   |   |-- community-classifier.csv # Category mapping data
|   |   |-- moderationRules.json     # Default moderation rules
|   |-- middlewares/
|   |   |-- auth/                    # Authentication middleware
|   |   |-- limiter/                 # Rate limiting middleware
|   |   |-- logger/                  # Request logging middleware
|   |   |-- post/                    # Post validation middleware
|   |   |-- users/                   # User validation middleware
|   |-- models/
|   |   |-- admin.model.js           # Admin schema
|   |   |-- comment.model.js         # Comment schema
|   |   |-- community.model.js       # Community schema
|   |   |-- config.model.js          # API configuration schema
|   |   |-- context.model.js         # Auth context schema
|   |   |-- email.model.js           # Email record schema
|   |   |-- log.model.js             # Activity log schema
|   |   |-- pendingPost.model.js     # Pending post schema
|   |   |-- post.model.js            # Post schema
|   |   |-- preference.model.js      # User preference schema
|   |   |-- relationship.model.js    # Follow relationship schema
|   |   |-- report.model.js          # Post report schema
|   |   |-- rule.model.js            # Moderation rule schema
|   |   |-- suspiciousLogin.model.js # Suspicious login schema
|   |   |-- token.admin.model.js     # Admin token schema
|   |   |-- token.model.js           # User token schema
|   |   |-- user.model.js            # User schema
|   |-- routes/
|   |   |-- admin.route.js           # /admin endpoints
|   |   |-- community.route.js       # /communities endpoints
|   |   |-- context-auth.route.js    # /auth endpoints
|   |   |-- post.route.js            # /posts endpoints
|   |   |-- user.route.js            # /users endpoints
|   |-- scripts/
|   |   |-- add-community.js         # Seed communities
|   |   |-- add-moderator.js         # Assign moderators
|   |   |-- add-rules.js             # Seed moderation rules
|   |   |-- create-admin.js          # Create admin account
|   |   |-- remove-moderator.js      # Remove moderators
|   |-- services/
|   |   |-- analyzeContent.js        # Content moderation pipeline
|   |   |-- apiServices.js           # NLP API integrations
|   |   |-- categoryFilterService.js # Category validation
|   |   |-- manageClassifier.js      # Classifier service manager
|   |   |-- processPost.js           # Post processing logic
|   |-- utils/
|   |   |-- confirmationToken.js     # Token generation
|   |   |-- contextData.js           # Context extraction
|   |   |-- emailTemplates.js        # Email HTML templates
|   |   |-- encryption.js            # AES encrypt/decrypt
|   |   |-- timeConverter.js         # Time formatting
|   |-- app.js                       # Express application entry point
|   |-- admin_tool.sh                # CLI admin setup tool
|   |-- package.json
|   |-- .env.example
|
|-- classifier_server/               # Python Flask classifier service
|   |-- classifier_api.py            # Flask API with BART-Large-MNLI
|   |-- requirements.txt             # Python dependencies
|   |-- Dockerfile                   # Container build file
|   |-- Readme.md                    # Classifier-specific docs
|
|-- resources/                       # Documentation assets
|   |-- Schema-Diagram.png
|   |-- UI-community.png
|
|-- LICENSE
|-- README.md
|-- .gitignore
```

---

## Getting Started

### Prerequisites

- **Node.js** (v16 or higher recommended)
- **npm** (v8 or higher)
- **MongoDB** (local instance or MongoDB Atlas account)
- **Python 3.8+** and **pip** (only if running the classifier server locally)
- **Docker** (optional, for running the classifier server in a container)

### 1. Clone the Repository

```bash
git clone https://github.com/akshay1389/socialecho.git
cd socialecho
```

### 2. Install Dependencies

**Client:**

```bash
cd client
npm install
```

**Server:**

```bash
cd server
npm install
```

**Classifier Server (optional):**

```bash
cd classifier_server
pip install -r requirements.txt
```

> Note: The classifier server downloads the BART-Large-MNLI model files (~1.6 GB) on first run.

### 3. Configure Environment Variables

Create `.env` files in both the `client/` and `server/` directories based on the provided `.env.example` files. See the [Environment Variables](#environment-variables) section below for details.

```bash
cp client/.env.example client/.env
cp server/.env.example server/.env
```

### 4. Set Up the Database

Run the admin setup tool from the `server/` directory to create the admin account, seed communities, and configure moderation rules:

```bash
cd server
chmod +x admin_tool.sh
./admin_tool.sh
```

The interactive script provides the following options:

1. Create an admin account
2. Add communities and rules to the database
3. Add rules to communities
4. Add moderators to communities
5. Remove moderators from communities

### 5. Start the Services

**Start the backend server (Port 4000):**

```bash
cd server
npm start
```

**Start the frontend client (Port 3000):**

```bash
cd client
npm start
```

**Start the classifier server (Port 5000) -- choose one method:**

Option A -- Run directly with Python:

```bash
cd classifier_server
python classifier_api.py
```

Option B -- Run with Docker:

```bash
docker pull neaz/classifier_api
docker run -p 5000:5000 neaz/classifier_api
```

The application will be accessible at `http://localhost:3000`.

---

## Environment Variables

### Server (`server/.env`)

| Variable | Description | Required |
|---|---|---|
| `CLIENT_URL` | Frontend URL (default: `http://localhost:3000`) | Yes |
| `MONGODB_URI` | MongoDB connection string | Yes |
| `PORT` | Server port (default: `4000`) | Yes |
| `SECRET` | JWT access token secret | Yes |
| `REFRESH_SECRET` | JWT refresh token secret | Yes |
| `CRYPTO_KEY` | AES encryption key for context data | Yes |
| `EMAIL` | Email address for sending notifications | For context auth |
| `PASSWORD` | Email account password or app password | For context auth |
| `EMAIL_SERVICE` | Email provider (e.g., `Zoho`, `Gmail`) | For context auth |
| `PERSPECTIVE_API_KEY` | Google Perspective API key | For moderation |
| `PERSPECTIVE_API_DISCOVERY_URL` | Perspective API discovery endpoint | For moderation |
| `TEXTRAZOR_API_KEY` | TextRazor API key | For moderation (alt.) |
| `TEXTRAZOR_API_URL` | TextRazor API endpoint | For moderation (alt.) |
| `INTERFACE_API_KEY` | Hugging Face Inference API key | For moderation (alt.) |
| `INTERFACE_API_URL` | Hugging Face model endpoint | For moderation (alt.) |
| `CLASSIFIER_API_URL` | Local Flask classifier URL (default: `http://127.0.0.1:5000/classify`) | For moderation (alt.) |

> Note: Content moderation requires `PERSPECTIVE_API_KEY` for toxicity analysis. For content categorization, configure at least one of: `TEXTRAZOR_API_KEY`, `INTERFACE_API_KEY`, or `CLASSIFIER_API_URL`. These services can be enabled, disabled, or switched from the admin dashboard at runtime.

### Client (`client/.env`)

| Variable | Description | Required |
|---|---|---|
| `REACT_APP_API_URL` | Backend API URL (default: `http://127.0.0.1:4000`) | Yes |

---

## Future Enhancements

- **Real-Time Notifications** -- WebSocket-based live notifications for likes, comments, follows, and moderation actions.
- **Direct Messaging** -- Private messaging between users with end-to-end encryption.
- **Media Processing Pipeline** -- Image and video moderation using computer vision models to detect inappropriate visual content.
- **Advanced Search** -- Full-text search with filters for communities, users, posts, and date ranges using Elasticsearch or MongoDB Atlas Search.
- **OAuth Integration** -- Social login support via Google, GitHub, and other OAuth 2.0 providers.
- **Internationalization (i18n)** -- Multi-language support for the user interface.
- **Analytics Dashboard** -- Platform-wide analytics for admins, including user growth, post engagement metrics, and moderation statistics.
- **Caching Layer** -- Redis-based caching for frequently accessed data such as feeds and community listings.
- **CI/CD Pipeline** -- Automated testing, linting, and deployment workflows using GitHub Actions.
- **Mobile Application** -- React Native client for iOS and Android platforms.
- **Content Recommendation Engine** -- Personalized feed ranking based on user interests and engagement history.
- **Two-Factor Authentication (2FA)** -- TOTP-based second factor for enhanced account security.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

Copyright (c) 2023 Neaz Mahmud.
