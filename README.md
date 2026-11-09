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

This is an Express server backed by MongoDB (Mongoose). Users register and log in to get a JWT, which the client sends in the `auth-token` header. Cabs are stored with their coordinates, town, pincode and a booking status. Booking a cab marks it `Not Available` and links it to the user. Ending a trip moves the cab to the drop-off coordinates and makes it `Available` again. An admin, identified by the `isAdmin` flag in the token, can list users and delete non-admin users.

The deployed frontend calls this API at `https://smart-cab-allocation-backend.onrender.com`.

## Features

- **Registration and login:** passwords hashed with `bcryptjs`, input validated with `express-validator`, and JWTs that expire after 1 hour
- **Admin role:** a user who registers with the configured admin email gets `isAdmin: true`
- **Cab management:** add a cab (name, phone, email, latitude/longitude, town, pincode) and list available cabs
- **Booking lifecycle:** book an available cab, then end the trip to update the cab's location and free it up
- **Admin user management:** list users (passwords excluded) and delete non-admin users
- **Google Maps proxy:** nearby places, place details and reverse geocoding (returns the postal code)
- **Socket.IO relay:** a `cabLocationUpdate` event is re-broadcast to all connected clients

## Tech stack

| Area | Tools |
| --- | --- |
| Runtime | Node.js, Express 4 |
| Database | MongoDB with Mongoose 8 |
| Auth | `jsonwebtoken`, `bcryptjs`, `express-validator` |
| Realtime | Socket.IO 4 |
| External APIs | Google Maps Places / Geocoding (via `axios`) |
| Config | `dotenv`, `cors` |

## Project structure

```
smart-cab-allocation-backend/
├── server.js              # Express + Socket.IO entry point (port 5001 by default)
├── config/db.js           # MongoDB connection
├── controllers/
│   ├── authController.js  # register, login, getUser
│   ├── adminController.js # list / delete users
│   └── cabController.js   # (not mounted by any route)
├── middleware/
│   ├── fetchUser.js       # verifies the JWT from the auth-token header
│   └── isAdmin.js         # verifies the JWT and requires isAdmin
├── models/                # User, Cab
└── routes/                # authRoutes, cabRoutes, adminRoutes, placesRoute
```

## Getting started

### Prerequisites

- Node.js and npm
- A MongoDB database (local or MongoDB Atlas)
- A Google Maps API key with the Places and Geocoding APIs enabled

### Install and run

```bash
git clone https://github.com/VamsiKrishnaThota03/smart-cab-allocation-backend.git
cd smart-cab-allocation-backend
npm install
node server.js        # or: npx nodemon server.js
```

The server listens on `PORT` (default `5001`).

### Environment variables

Create a `.env` file in the project root:

| Variable | Description |
| --- | --- |
| `PORT` | Port for the HTTP server (optional, default `5001`) |
| `api` | Google Maps API key used by the `/api/places`, `/api/place-details` and `/api/geocode` routes |
| `JWT_SECRET` | Secret for signing JWTs (see the note below) |

```env
PORT=5001
api=<your-google-maps-api-key>
JWT_SECRET=<a-long-random-secret>
```

> **Note:** in the current code, the MongoDB connection string (`config/db.js`) and the JWT secret (`controllers/authController.js` and `middleware/*.js`) are hard-coded rather than read from the environment. To use your own database and secret, change those files to read from `process.env`, for example `process.env.MONGO_URI` and `process.env.JWT_SECRET`.

## API

Protected routes need the JWT in an `auth-token` request header.

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/api/auth/register` | - | Register a user (`name`, `email`, `password`) and return a JWT |
| POST | `/api/auth/login` | - | Log in and return `{ token, isAdmin }` |
| POST | `/api/auth/getuser` | User | Get the logged-in user's profile |
| POST | `/api/cabs/addCab` | User | Add a cab (`name`, `phone`, `email`, `latitude`, `longitude`, `pincode`, `town`) |
| GET | `/api/cabs/availableCabs` | - | List cabs whose status is `Available` |
| POST | `/api/cabs/bookCab/:cabId` | User | Book an available cab for the current user |
| POST | `/api/cabs/endTrip/:cabId` | - | End a trip: move the cab to `destinationLat` / `destinationLng` and make it available |
| GET | `/api/admin/users` | Admin | List all users (passwords excluded) |
| DELETE | `/api/admin/users/:id` | Admin | Delete a non-admin user |
| GET | `/api/places?lat=&lng=` | - | Proxy to the Google Places Nearby Search API |
| GET | `/api/place-details?placeId=` | - | Proxy to the Google Place Details API |
| GET | `/api/geocode?lat=&lng=` | - | Reverse geocode coordinates and return `{ postalCode }` |

### Socket.IO

| Event | Direction | Purpose |
| --- | --- | --- |
| `cabLocationUpdate` | client to server to all clients | Re-broadcasts a cab location payload to every connected client |
