# 🏨 Hotel Booking System - Architecture & Flow Documentation

This document provides a comprehensive overview of the full-stack architecture, including the request lifecycle, API endpoints, database schema, and core class structures.

---

## 1. High-Level Website Navigation Flow
This is a simple, non-technical flowchart showing how a user navigates through the website's pages.

```mermaid
flowchart TD
    %% Main Entry Point
    Landing["🏠 Home Page<br/>(Browse & Search Hotels)"]
    
    %% Authentication Zone
    subgraph Auth [Authentication]
        Login["🔐 Login Page"]
        Register["📝 Register Page"]
        Login <--> Register
    end

    %% Standard User Interactions
    subgraph UserFlow [User Flow]
        Details["🏨 Hotel Details<br/>(Select Rooms)"]
        Dashboard["👤 My Dashboard<br/>(Manage Bookings)"]
    end

    %% Administrator Interactions
    subgraph AdminFlow [Admin Flow]
        AdminDash["🛡️ Admin Panel"]
        ManageBookings["📋 Approve/Reject"]
        AddHotel["🏗️ Create Property"]
    end

    %% Routing Connections
    Landing -->|Click Login| Login
    Login -->|Success| Landing
    
    Landing -->|Select Hotel| Details
    Details -->|Book Room| Dashboard
    
    Landing -->|Admin Login| AdminDash
    AdminDash --> ManageBookings
    AdminDash --> AddHotel
```

---

## 2. Technical API Request Lifecycle

### 🟢 Standard User Flow (Login & Booking)
This diagram illustrates how a normal user authenticates and books a hotel room.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant ReactUI as React UI (*Page.jsx)
    participant Axios as Axios (api.js)
    participant WebSec as Security Layer (AuthTokenFilter.java)
    participant Controllers as REST Controllers (*Controller.java)
    participant Services as Business Logic (*Service.java)
    participant Repository as Repositories (*Repository.java)
    participant Database as PostgreSQL Database

    %% Authentication Flow
    User->>ReactUI: Enters credentials on LoginPage.jsx
    ReactUI->>Axios: Call login(email, password)
    Axios->>WebSec: POST /api/auth/signin
    WebSec->>Controllers: Route to AuthController.java
    Controllers->>Services: Validate user via UserDetailsServiceImpl.java
    Services->>Repository: Query UserRepository.java
    Repository->>Database: SELECT query
    Database-->>Repository: Return User data
    Repository-->>Services: Return User Entity
    Services-->>Controllers: Generate JWT in JwtUtils.java
    Controllers-->>ReactUI: Return JWT Token to Frontend
    
    %% Booking Flow
    User->>ReactUI: Clicks "Book Now" on HotelDetailsPage.jsx
    ReactUI->>Axios: api.post('/bookings', data)
    Note over Axios: Axios Interceptor attaches JWT<br/>Authorization: Bearer <token>
    Axios->>WebSec: POST /api/bookings
    Note over WebSec: AuthTokenFilter.java validates JWT
    WebSec->>Controllers: Route to BookingController.java
    Controllers->>Services: BookingService.java createBooking()
    Services->>Repository: BookingRepository.java save()
    Repository->>Database: INSERT query
    Database-->>Repository: Success
    Repository-->>Services: Booking Entity saved
    Note over Services: EmailService.java sends Brevo HTML Email
    Services-->>Controllers: Return HTTP 200 OK
    Controllers-->>ReactUI: Show Success Alert in DashboardPage.jsx
```

### 🔴 Admin Flow (Hotel & Room Creation)
This diagram illustrates how an Administrator uses the secured dashboard to create new properties.

```mermaid
sequenceDiagram
    autonumber
    actor Admin
    participant ReactUI as React UI (*Page.jsx)
    participant Axios as Axios (api.js)
    participant WebSec as Security Layer (AuthTokenFilter.java)
    participant Controllers as REST Controllers (*Controller.java)
    participant Services as Business Logic (*Service.java)
    participant Repository as Repositories (*Repository.java)
    participant Database as PostgreSQL Database

    %% Admin Flow (Create Hotel)
    Admin->>ReactUI: Fills "Add Hotel" form on AdminDashboardPage.jsx
    ReactUI->>Axios: api.post('/hotels', hotelData)
    Note over Axios: Axios Interceptor attaches JWT<br/>Authorization: Bearer <token>
    Axios->>WebSec: POST /api/hotels
    Note over WebSec: AuthTokenFilter.java validates JWT
    Note over WebSec: WebSecurityConfig.java strictly requires hasAuthority("ROLE_ADMIN")
    WebSec->>Controllers: Route to HotelController.java
    Controllers->>Services: HotelService.java createHotel()
    Services->>Repository: HotelRepository.java & RoomRepository.java save()
    Repository->>Database: INSERT queries (Hotel & Rooms)
    Database-->>Repository: Success
    Repository-->>Services: Hotel & Rooms Entities saved
    Services-->>Controllers: Return HTTP 200 OK
    Controllers-->>ReactUI: Show Success Alert in AdminDashboardPage.jsx
    
    %% Admin Flow (Approve/Reject Booking)
    Admin->>ReactUI: Clicks "Approve/Reject" on AdminDashboardPage.jsx
    ReactUI->>Axios: api.put('/bookings/{id}/status', status)
    Note over Axios: Axios Interceptor attaches JWT<br/>Authorization: Bearer <token>
    Axios->>WebSec: PUT /api/bookings/{id}/status
    Note over WebSec: AuthTokenFilter.java validates JWT
    Note over WebSec: WebSecurityConfig.java strictly requires hasAuthority("ROLE_ADMIN")
    WebSec->>Controllers: Route to BookingController.java
    Controllers->>Services: BookingService.java updateStatus()
    Services->>Repository: BookingRepository.java save()
    Repository->>Database: UPDATE query (change status)
    Database-->>Repository: Success
    Repository-->>Services: Updated Booking Entity saved
    Note over Services: EmailService.java sends Brevo HTML Status Email
    Services-->>Controllers: Return HTTP 200 OK
    Controllers-->>ReactUI: Refresh Bookings List in AdminDashboardPage.jsx
```

---

## 3. Database Schema (Entity Relationship Diagram)

```mermaid
erDiagram
    USERS ||--o{ BOOKINGS : "1:N (makes)"
    HOTELS ||--o{ ROOMS : "1:N (contains)"
    ROOMS ||--o{ BOOKINGS : "1:N (has)"

    USERS {
        Long id PK
        String name
        String email UK
        String password
        String provider "local or google"
        Role role "ROLE_USER or ROLE_ADMIN"
    }

    HOTELS {
        Long id PK
        String name
        String location
        String description
        String amenities
    }

    ROOMS {
        Long id PK
        String roomType "e.g. Deluxe"
        BigDecimal price
        Boolean isAvailable
        Long hotel_id FK
    }

    BOOKINGS {
        Long id PK
        LocalDate checkInDate
        LocalDate checkOutDate
        BookingStatus status "PENDING, CONFIRMED, CANCELLED"
        Long user_id FK
        Long room_id FK
    }
```

---

## 4. Backend Class & Interface Definitions

### 🏛️ Core Models (Entities)
- `User.java`: Represents the user profile and credentials.
- `Hotel.java`: Represents a physical hotel property.
- `Room.java`: Represents a specific room category inside a hotel.
- `Booking.java`: The transactional record linking a User, a Room, and Dates.
- `BookingStatus.java`: Enum (`PENDING`, `CONFIRMED`, `CANCELLED`).

### 📦 DTOs (Data Transfer Objects)
- `LoginRequest.java` / `SignupRequest.java`: Payloads from frontend Auth forms.
- `JwtResponse.java`: The token payload sent back to the frontend.
- `BookingRequest.java`: Contains `roomId`, `checkInDate`, `checkOutDate`.
- `HotelRequest.java`: Contains `name`, `location`, `description`, `amenities`, and `List<RoomRequest>`.
- `RoomRequest.java`: Contains `roomType`, `price`.

### 🗄️ Repositories (Interfaces extending JpaRepository)
- `UserRepository.java`: Queries users by email.
- `HotelRepository.java`: Queries hotels by location.
- `RoomRepository.java`: Finds available rooms for specific date ranges using custom `@Query`.
- `BookingRepository.java`: Queries bookings by `UserId` (for dashboard) or gets all (for admin).

### ⚙️ Services (Business Logic)
- `HotelService.java`: Logic for searching hotels and dynamically creating new ones.
- `BookingService.java`: Logic for checking room availability, creating a booking, and verifying ownership before cancellation.
- `EmailService.java`: Communicates with the Brevo HTTP API to send HTML notifications.
- `UserDetailsServiceImpl.java`: Implements Spring Security's `UserDetailsService` to load users from the DB during login.

### 🔌 Controllers (REST Endpoints)
- `AuthController.java`: Exposes `/api/auth/signin` and `/api/auth/signup`.
- `HotelController.java`: Exposes `/api/hotels` (Public GET, Admin POST).
- `BookingController.java`: Exposes `/api/bookings` (User standard flow) and `/api/bookings/admin` (Admin approval flow).

### 🛡️ Security
- `WebSecurityConfig.java`: The master configuration. Enables CORS, disables CSRF (since we use JWT), and secures endpoints based on user roles (`ROLE_USER`, `ROLE_ADMIN`).
- `AuthTokenFilter.java`: Intercepts HTTP requests, validates the `Authorization` header, and sets the Security Context.
- `JwtUtils.java`: Helper class to generate and parse JSON Web Tokens.
