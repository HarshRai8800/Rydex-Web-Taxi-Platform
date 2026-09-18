# Rydex — Real-Time Taxi Platform

Rydex is a full-stack, real-time taxi and ride-hailing platform built with **Next.js, TypeScript, MongoDB, Socket.IO, and Razorpay**.

The platform supports three primary roles — **Users, Partners, and Admins** — and provides the complete ride lifecycle from partner onboarding and verification to ride booking, payment, live driver tracking, and ride completion.

The application uses a **Next.js full-stack architecture** for the main application and API layer, together with a dedicated **Socket.IO server** for real-time communication and location updates.

---

## 🚀 Features

### 👤 User Features

* User registration and email verification
* Email/password authentication
* Google authentication
* Browse/search available vehicles and partners
* Select pickup and drop locations
* Fare calculation
* Ride booking
* Active ride tracking
* Live driver location updates
* Ride cancellation
* Online payment through Razorpay
* Cash payment support
* Ride history
* Driver/user communication through real-time chat

### 🚗 Partner Features

Partners can register as drivers and complete a multi-step onboarding process.

* Partner registration
* Email verification
* Multi-step partner onboarding
* Mobile number management
* Vehicle registration
* Vehicle document submission
* Aadhaar, RC and driving-license document uploads
* Bank account / UPI details
* Partner approval workflow
* Video KYC
* View pending ride requests
* Accept/reject ride requests
* Manage active rides
* Update driver availability
* Live location sharing
* Ride history
* Earnings dashboard

### 🛡️ Admin Features

Admins manage the platform's users, partners, vehicles, verification and financial information.

* Admin authentication
* Dashboard
* Partner review and approval
* Vehicle review and approval
* Partner document verification
* Video KYC verification
* Approve/reject partners
* Approve/reject vehicles
* Rejection reason management
* Earnings and commission monitoring
* Platform-level booking information

---

# 🏗️ Architecture

Rydex follows a **hybrid full-stack architecture**.

```text
                    ┌──────────────────────┐
                    │       Browser        │
                    │                      │
                    │  User / Partner /    │
                    │       Admin          │
                    └──────────┬───────────┘
                               │
                               │ HTTP
                               ▼
                 ┌──────────────────────────┐
                 │       Next.js App        │
                 │                          │
                 │  React UI + App Router  │
                 │  API Route Handlers      │
                 │  NextAuth Authentication │
                 └───────────┬──────────────┘
                             │
                ┌────────────┼─────────────┐
                │            │             │
                ▼            ▼             ▼
          ┌──────────┐ ┌───────────┐ ┌────────────┐
          │ MongoDB  │ │ Razorpay  │ │ Cloudinary │
          │ Mongoose │ │ Payments  │ │ File Store │
          └──────────┘ └───────────┘ └────────────┘
                            

                    Real-Time Layer
                           │
                           ▼
                 ┌─────────────────────┐
                 │ Dedicated Socket.IO │
                 │      Server         │
                 │                     │
                 │ • Online status     │
                 │ • Location updates  │
                 │ • Ride rooms        │
                 │ • Chat              │
                 │ • Notifications     │
                 └──────────┬──────────┘
                            │
                            ▼
                    Connected clients
```

### Why a separate Socket.IO server?

The main Next.js application handles normal HTTP requests, authentication, database operations and business logic.

Real-time operations are handled by a dedicated Socket.IO server. This keeps persistent socket connections separate from the request/response API layer.

The socket server manages:

* User socket identity
* Online/offline status
* Driver location updates
* Ride-specific rooms
* Driver location broadcasting
* Ride chat
* Server-triggered notifications

---

# 🔄 Ride Flow

A typical ride follows this lifecycle:

```text
User selects pickup/drop
          │
          ▼
Search available partner/vehicle
          │
          ▼
Create Booking
          │
          ▼
Booking = requested
          │
          ▼
Partner receives real-time notification
          │
          ▼
Partner accepts request
          │
          ▼
Payment
   ┌──────┴──────┐
   │             │
Online          Cash
   │             │
Razorpay        Cash
   │             │
   └──────┬──────┘
          ▼
Booking = confirmed
          │
          ▼
Pickup verification
          │
          ▼
Ride started
          │
          ▼
Live driver tracking
          │
          ▼
Drop verification
          │
          ▼
Ride completed
```

The booking model maintains explicit ride states including:

* `requested`
* `awaiting_payment`
* `confirmed`
* `started`
* `completed`
* `cancelled`
* `rejected`
* `expired`

Payment states include:

* `pending`
* `paid`
* `cash`
* `failed`

---

# 🔐 Authentication & Authorization

Rydex uses **NextAuth** with JWT-based sessions.

Supported authentication methods:

* Credentials
* Google OAuth

Credentials are stored using password hashing with `bcryptjs`.

The application supports three roles:

```text
user
partner
admin
```

Role-aware route protection is implemented through the application's authentication/proxy layer.

Examples:

```text
/admin/*       → Admin only

/partner/*     → Partner only

/api/*         → Authenticated users

/partner/onboarding
               → Available during partner onboarding
```

JWT session data contains user identity and role information, allowing the application to perform role-based authorization.

---

# 📍 Real-Time Location Tracking

The platform uses **Socket.IO** for real-time communication.

Each connected client identifies itself with its user ID.

The socket server maintains:

```text
userId
socketId
isOnline
location
```

Driver locations are stored using MongoDB GeoJSON coordinates:

```text
{
  type: "Point",
  coordinates: [longitude, latitude]
}
```

A `2dsphere` index is created on user locations to support geospatial queries.

During an active ride:

```text
Driver
  │
  │ location update
  ▼
Socket.IO Server
  │
  │ ride room
  ▼
Passenger
```

Ride-specific rooms use the booking ID, allowing driver location updates and chat messages to be delivered only to participants of that ride.

---

# 💬 Real-Time Ride Chat

Each booking can have a dedicated Socket.IO room:

```text
ride-{bookingId}
```

Clients join the room when an active ride begins.

Messages are broadcast through the room:

```text
Passenger ──► Socket.IO ──► Driver
Driver    ──► Socket.IO ──► Passenger
```

Chat messages are also represented by a MongoDB model containing:

* Booking ID
* Sender
* Message text
* Timestamp

---

# 💳 Payment Integration

Rydex integrates **Razorpay** for online payments.

The payment flow is:

```text
Booking
   │
   ▼
Create Razorpay Order
   │
   ▼
User completes payment
   │
   ▼
Razorpay returns payment details
   │
   ▼
Server verifies signature
   │
   ▼
Booking marked as paid
   │
   ▼
Booking confirmed
```

Payment verification uses an HMAC SHA-256 signature generated from:

```text
order_id + "|" + payment_id
```

The platform also calculates:

```text
Admin Commission
Partner Amount
```

after successful payment verification.

---

# 🎥 Video KYC

Partner verification includes a video KYC workflow.

The platform uses **ZEGOCLOUD UIKit Prebuilt** for one-to-one video calls.

The workflow is:

```text
Admin selects partner
        │
        ▼
KYC room generated
        │
        ▼
Partner/Admin joins video call
        │
        ▼
Verification
   ┌────┴─────┐
   │          │
Approve     Reject
   │          │
   ▼          ▼
KYC        Rejection
approved    reason
```

KYC states include:

```text
not_required
pending
in_progress
approved
rejected
```

---

# 📄 Partner Verification

Partners submit information required for platform approval.

### Partner documents

* Aadhaar
* Vehicle RC
* Driving License

Documents are uploaded using **Cloudinary** and their secure URLs are stored in MongoDB.

### Vehicle information

Vehicles contain:

* Vehicle type
* Vehicle model
* Registration number
* Vehicle image
* Base fare
* Price per kilometer
* Waiting charge
* Approval status

Supported vehicle types include:

```text
bike
car
auto
loading
truck
```

---

# 💰 Fare & Commission

Each booking stores:

```text
fare
adminCommission
partnerAmount
```

The current implementation calculates the platform commission as **10% of the booking fare** during successful Razorpay payment verification.

```text
Booking Fare
     │
     ├── 10% → Platform Commission
     │
     └── 90% → Partner Amount
```

---

# 🗃️ Database Design

MongoDB is used as the primary database through Mongoose.

Main models include:

```text
User
 ├── authentication
 ├── role
 ├── partner status
 ├── KYC status
 ├── online status
 └── location

Booking
 ├── user
 ├── driver
 ├── vehicle
 ├── pickup/drop locations
 ├── fare
 ├── booking status
 ├── payment status
 ├── OTPs
 └── commission

Vehicle
 ├── owner
 ├── type
 ├── model
 ├── registration number
 ├── pricing
 └── approval status

PartnerDocs
 ├── owner
 ├── Aadhaar
 ├── RC
 ├── License
 └── verification status

PartnerBank
 ├── account holder
 ├── account number
 ├── IFSC
 ├── UPI
 └── verification status

ChatMessage
 ├── booking
 ├── sender
 └── message
```

---

# 🛠️ Tech Stack

## Frontend / Full Stack

* Next.js 16
* React 19
* TypeScript
* Tailwind CSS
* Redux Toolkit
* React Leaflet
* Axios
* Motion
* Lucide React

## Backend

* Next.js Route Handlers
* Node.js
* Express
* Socket.IO

## Database

* MongoDB
* Mongoose

## Authentication

* NextAuth
* Google OAuth
* Credentials authentication
* bcryptjs
* JWT sessions

## Third-Party Services

* Razorpay — payments
* Cloudinary — document/file storage
* ZEGOCLOUD — video KYC
* Nodemailer — email communication

---

# 📁 Project Structure

```text
Rydex-Web-Taxi-Platform/
│
├── rydex/
│   ├── public/
│   │
│   ├── src/
│   │   ├── app/
│   │   │   ├── admin/
│   │   │   ├── partner/
│   │   │   ├── user/
│   │   │   ├── video-kyc/
│   │   │   │
│   │   │   └── api/
│   │   │       ├── admin/
│   │   │       ├── auth/
│   │   │       ├── booking/
│   │   │       ├── chat/
│   │   │       ├── partner/
│   │   │       ├── payment/
│   │   │       ├── user/
│   │   │       └── vehicles/
│   │   │
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── lib/
│   │   ├── models/
│   │   ├── redux/
│   │   ├── auth.ts
│   │   └── proxy.ts
│   │
│   ├── package.json
│   └── next.config.ts
│
└── socketServer/
    ├── models/
    │   └── user.model.js
    ├── index.js
    └── package.json
```

---

# ⚙️ Installation & Setup

## Prerequisites

Make sure you have:

* Node.js 20+
* npm
* MongoDB database
* Razorpay account
* Cloudinary account
* Google OAuth credentials
* ZEGOCLOUD credentials

---

## 1. Clone the repository

```bash
git clone https://github.com/HarshRai8800/Rydex-Web-Taxi-Platform.git

cd Rydex-Web-Taxi-Platform
```

---

## 2. Setup the Next.js application

```bash
cd rydex

npm install
```

Create:

```text
rydex/.env.local
```

Add the required environment variables:

```env
MONGODB_URL=your_mongodb_connection_string

AUTH_SECRET=your_nextauth_secret

AUTH_GOOGLE_ID=your_google_client_id
AUTH_GOOGLE_SECRET=your_google_client_secret

NEXT_PUBLIC_SOCKET_SERVER_URL=http://localhost:8000

RAZORPAY_KEY_ID=your_razorpay_key_id
RAZORPAY_KEY_SECRET=your_razorpay_key_secret

CLOUDINARY_CLOUD_NAME=your_cloudinary_cloud_name
CLOUDINARY_API_KEY=your_cloudinary_api_key
CLOUDINARY_API_SECRET=your_cloudinary_api_secret

NEXT_PUBLIC_ZEGO_APP_ID=your_zego_app_id
NEXT_PUBLIC_ZEGO_SERVER_SECRET=your_zego_server_secret
```

Use your actual environment variable names from the source code and **never commit real credentials**.

---

## 3. Setup the Socket.IO server

Open another terminal:

```bash
cd socketServer

npm install
```

Create:

```text
socketServer/.env
```

Example:

```env
PORT=8000
MONGODB_URL=your_mongodb_connection_string
NEXT_BASE_URL=http://localhost:3000
```

---

# ▶️ Running the Application

You need to run both services.

### Terminal 1 — Next.js

```bash
cd rydex
npm run dev
```

The application will run at:

```text
http://localhost:3000
```

### Terminal 2 — Socket.IO Server

```bash
cd socketServer
npm run dev
```

The real-time server will run on:

```text
http://localhost:8000
```

---

# 🧪 Production Build

For the Next.js application:

```bash
cd rydex

npm install
npm run build
npm start
```

For the Socket.IO server:

```bash
cd socketServer

npm install
node index.js
```

---

# 🔌 Important API Areas

The Next.js application exposes API route groups for:

```text
/api/auth
/api/booking
/api/payment
/api/partner
/api/vehicles
/api/user
/api/admin
/api/chat
```

### Booking

```text
POST /api/booking/create
GET  /api/booking/active
```

Booking routes also support operations such as confirmation and cancellation.

### Payments

```text
POST /api/payment/create
POST /api/payment/verify
```

### Partner

Partner APIs cover:

```text
Onboarding
Bookings
Earnings
Video KYC
Active rides
```

### Admin

Admin APIs cover:

```text
Dashboard
Partner reviews
Vehicle reviews
Video KYC
Earnings
```

---

# 🔒 Security Considerations

The application implements several security mechanisms:

* Password hashing with bcrypt
* JWT-based authentication
* Role-based route protection
* Razorpay payment signature verification
* Authenticated API access
* Environment-based secret configuration
* MongoDB-backed user authorization
* Partner and vehicle approval workflows

For production deployment, additional protections such as rate limiting, stronger validation, CSRF considerations, secret rotation, audit logging and centralized error monitoring should be added.

---

# ⚠️ Current Limitations

This project is primarily a full-stack engineering project and has several areas that would require additional work for production-scale deployment:

* Socket.IO infrastructure would need horizontal scaling support for multiple server instances.
* Real-time location updates could be optimized using dedicated geospatial infrastructure.
* Rate limiting and request throttling should be added to public/authentication endpoints.
* More extensive input validation and API-level schema validation would strengthen the backend.
* Payment webhook handling can be expanded for production-grade payment reconciliation.
* Automated tests and integration tests can be expanded.
* Monitoring, logging and centralized error tracking should be added for production deployment.
* Sensitive environment variables must be managed through deployment secrets rather than committed files.

---

# 🚀 Future Improvements

Potential improvements include:

* Redis adapter for horizontally scaled Socket.IO servers
* Redis-based driver availability/location caching
* Advanced driver matching based on distance
* Push notifications
* Trip route optimization
* Dynamic/surge pricing
* Driver ratings and reviews
* Automated payment reconciliation
* CI/CD pipeline
* Automated unit and integration testing
* Centralized logging and monitoring
* Docker-based deployment
* AWS deployment with load balancing and autoscaling

---

# 📚 Engineering Concepts Demonstrated

This project demonstrates practical implementation of:

* Full-stack Next.js architecture
* REST API design
* Real-time WebSocket communication
* Socket.IO rooms
* Geospatial MongoDB data
* JWT authentication
* OAuth authentication
* Role-based authorization
* Payment processing
* Payment signature verification
* File uploads
* Cloud storage
* Video communication
* Multi-step onboarding
* Ride state machines
* Database relationships using MongoDB references
* Separation of HTTP and real-time services

---

# 👨‍💻 Author

**Harsh Rai**

GitHub: [HarshRai8800](https://github.com/HarshRai8800)

---

## ⭐ If you find this project useful

Consider giving the repository a star and exploring the implementation.
