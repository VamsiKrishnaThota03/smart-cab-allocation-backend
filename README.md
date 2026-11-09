# Smart Cab Allocation - Backend

The REST API for the Smart Cab Allocation app. It handles user authentication, cab inventory and bookings, admin user management, and proxies the Google Maps Places and Geocoding APIs.

![Node.js](https://img.shields.io/badge/Node.js-Express%204-339933?logo=nodedotjs&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose%208-47A248?logo=mongodb&logoColor=white)
![JWT](https://img.shields.io/badge/auth-JWT-000000?logo=jsonwebtokens)
![Socket.IO](https://img.shields.io/badge/Socket.IO-4-010101?logo=socketdotio)

> **Related repositories**
> - Frontend (React): [smart-cab-allocation-frontend](https://github.com/VamsiKrishnaThota03/smart-cab-allocation-frontend)
> - Combined repo (frontend + `backend/` folder): [smart-cab-allocation](https://github.com/VamsiKrishnaThota03/smart-cab-allocation)

## Overview

This is an Express server backed by MongoDB (Mongoose). Users register and log in to get a JWT, which the client sends in the `auth-token` header. Cabs are stored with their coordinates, town, pincode and a booking status. Booking a cab marks it `Not Available` and links it to the user. Ending a trip moves the cab to the drop-off coordinates and makes it `Available` again. Admins (users whose email is in `ADMIN_EMAILS` when they register) add cabs, list users and delete non-admin users.

The deployed frontend calls this API at `https://smart-cab-allocation-backend.onrender.com`.

## Features

- **Registration and login:** passwords hashed with `bcryptjs`, input validated with `express-validator`, JWTs that expire after 1 hour
- **Admin role:** granted only to the emails in `ADMIN_EMAILS`; admin routes check the signed `isAdmin` claim
- **Cab management:** admins add cabs; anyone can list available cabs
- **Booking lifecycle:** atomic booking (two riders cannot book the same cab), one active booking per rider, `GET /api/cabs/myCab` for the current booking, and ending a trip only by the rider who booked it (or an admin)
- **Admin user management:** list users (passwords excluded) and delete non-admin users
- **Google Maps proxy:** nearby places, place details and reverse geocoding (`{ postalCode, town }`), for logged-in users only, with validated coordinates
- **Socket.IO relay:** a `cabLocationUpdate` event is re-broadcast to all connected clients
- JSON error responses, a health route, and a configurable CORS allow list

## Tech stack

| Area | Tools |
| --- | --- |
| Runtime | Node.js 18+, Express 4 |
| Database | MongoDB with Mongoose 8 |
| Auth | `jsonwebtoken`, `bcryptjs`, `express-validator` |
| Realtime | Socket.IO 4 |
| External APIs | Google Maps Places / Geocoding (via `axios`) |
| Config | `dotenv`, `cors` |
| Tests | `node:test` |

## Project structure

```
smart-cab-allocation-backend/
├── server.js              # checks config, connects MongoDB, starts HTTP + Socket.IO
├── app.js                 # Express app: CORS, routes, 404 and error handlers
├── config/
│   ├── env.js             # requireEnv / listEnv helpers
│   └── db.js              # MongoDB connection (MONGO_URI)
├── controllers/
│   ├── authController.js  # register, login, getUser
│   └── adminController.js # list / delete users
├── middleware/
│   ├── fetchUser.js       # verifies the JWT from the auth-token header
│   ├── isAdmin.js         # requires the isAdmin claim (after fetchUser)
│   └── validate.js        # express-validator error handler
├── models/                # User, Cab
├── routes/                # authRoutes, cabRoutes, adminRoutes, placesRoute
└── test/api.test.js       # smoke tests (no database needed)
```

## Getting started

### Prerequisites

- Node.js 18 or newer and npm
- MongoDB (local or MongoDB Atlas)
- Optional: a Google Maps API key with the Places and Geocoding APIs enabled

### Install and run

```bash
git clone https://github.com/VamsiKrishnaThota03/smart-cab-allocation-backend.git
cd smart-cab-allocation-backend
npm install
cp .env.example .env     # then fill in the values
npm run dev              # nodemon; or: npm start
npm test                 # smoke tests
```

The server listens on `PORT` (default `5001`).

### Environment variables

| Variable | Required | Description |
| --- | --- | --- |
| `MONGO_URI` | Yes | MongoDB connection string, for example `mongodb://127.0.0.1:27017/smart-cab` |
| `JWT_SECRET` | Yes | Long random string used to sign JWTs |
| `GOOGLE_MAPS_API_KEY` | For geocoding | Server-side key with the Geocoding and Places APIs enabled. Without it the `/api/geocode`, `/api/places` and `/api/place-details` routes return 503 and the "nearest cabs" list falls back to the 5 km radius only |
| `ADMIN_EMAILS` | No | Comma-separated emails that get admin rights when they register |
| `CORS_ORIGIN` | No | Comma-separated allowed browser origins, for example `https://smart-cab-allocation-frontend.vercel.app`. When empty, every origin is allowed (fine for local development) |
| `PORT` | No | API port, default `5001` |

The server exits with a clear message if `MONGO_URI` or `JWT_SECRET` is missing.

## API

Protected routes need the JWT in an `auth-token` header. Errors are returned as `{ "message": "..." }`.

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| GET | `/` | - | Health check |
| POST | `/api/auth/register` | - | Register (`name`, `email`, `password`) and return `{ token, isAdmin }` |
| POST | `/api/auth/login` | - | Log in and return `{ token, isAdmin }` |
| GET / POST | `/api/auth/getuser` | User | Current user's profile |
| POST | `/api/cabs/addCab` | Admin | Add a cab (`name`, `phone`, `email`, `latitude`, `longitude`, optional `town`, `pincode`) |
| GET | `/api/cabs/availableCabs` | - | List available cabs (`name`, `latitude`, `longitude`) |
| GET | `/api/cabs/myCab` | User | The cab the current user has booked, or `{ cab: null }` |
| POST | `/api/cabs/bookCab/:cabId` | User | Book an available cab (one active booking per user) |
| POST | `/api/cabs/endTrip/:cabId` | Booking user or admin | End the trip; with `destinationLat` / `destinationLng` the cab moves to the drop-off point |
| GET | `/api/admin/users` | Admin | List users (passwords excluded) |
| DELETE | `/api/admin/users/:id` | Admin | Delete a non-admin user |
| GET | `/api/places?lat=&lng=` | User | Google Places Nearby Search proxy |
| GET | `/api/place-details?placeId=` | User | Google Place Details proxy |
| GET | `/api/geocode?lat=&lng=` | User | Reverse geocode and return `{ postalCode, town }` |

The server also runs a Socket.IO relay that re-broadcasts `cabLocationUpdate` events to every client. The current frontend does not use it; the trip movement is animated in the browser.

## Deployment notes (Render)

- Set `MONGO_URI`, `JWT_SECRET`, `GOOGLE_MAPS_API_KEY`, `ADMIN_EMAILS` and `CORS_ORIGIN` (the Vercel frontend URL). The start command is `npm start`.
- Earlier commits contained a MongoDB password, a JWT secret and a Google Maps key (`.env` and `config/db.js`). They are gone from the current code but still in the git history, so rotate all three.
- Changing `JWT_SECRET` logs everyone out once. Existing admin accounts keep their `isAdmin` flag in the database.

## Known limitations

- No rate limiting on login or registration.
- `GET /api/cabs/availableCabs` is public by design (the map shows cabs before booking).
- The Socket.IO relay is unauthenticated and not used by the current frontend.
- `npm audit` reports a moderate issue in `nodemon`'s `minimatch` (development only).
