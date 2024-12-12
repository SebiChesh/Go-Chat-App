# 📜 Project Overview

This is an **instant messaging chat app** made \***\*in part to demonstrate my abilities in full-stack development, and focusing on learning web security concepts as well as practising program architecture and scalability. It was built with a **Go** backend, a **React** frontend, and a **MySQL** database. The app is containerized using **Docker Compose\*\* for repeatable and easy deployment.

I've taken time to implement advanced patterns like **dependency injection** for modularity, scalability, and ease of testing. I also investigated and implemented security measures such as **session management**, **CSRF tokens**, and custom **CORS middleware** for better understanding and hands-on learning. The project is complete with some example mock services and unit tests for demonstraigtion of how to do unit tests in Go.

# 🚩Features

### Backend (Go)

- WebSockets for Real-Time chat and active user information.
- **Security Best Practices**:
  - Session Management and CSRF Protection implemented from scratch for deeper understanding.
  - Custom CORS Middleware to handle cross-origin requests.
- **Architecture**:
  - Dependency Injection for services (database and authentication), allowing easy testing with mock implementations.
  - Use of Go channels to broadcast messages to notify active users in real time.
- **Database Integration:**
  - MySQL for user authentication and message persistence.
  - Mock database implementations for unit testing.
- **Testing:**
  - Demonstration unit tests for authentication and database interactions.

### Frontend (React)

- Built with React Javascript/Typescript. Was good practice although not the focus of this project.
- Frontend communicates with the backend WebSocket API and REST endpoints.

### Database (MySQL)

- Used MySQL for practising implementing SQL despite it being overkill for this project.

### Docker

- Containerized using **Docker Compose** for easy consistent deployments.

### Tools

- **Postman**: API testing during development.
- **Git**: Version control for codebase management.

# Screenshots

<p align="center">
  <img src='docs/Screenshot-login.png'  style="width:75%;height:75%;">
</p>
<p align="center">
  <img src='docs/Screenshot-main.png'  style="width:75%;height:75%;">
</p>
<p align="center">
  <img src='docs/Screenshot-logged-out.png' style="width:75%;height:75%;">
  <p align="center"> When a user is logged out they cant connect to the websocket </p> 
</p>

# 💡 Motivation

This project was built as a learning exercise while teaching myself **Go** and exploring full-stack development and best practices for web development.

There are lots of Go frameworks such a (gin?) that handle some of the stuff implemented such as dependency injection or middleware however that aim was to practice and learn and I will be better equipped to use fraimewoks in future now I better understand the underlying mechanics.

# 📂 Project Structure (Less Important Bits Omitted)

```
chat-app/
├── backend/
│   ├── main.go          # Entry point for the Go server
│   ├── auth/            # Authentication logic
│   │   ├── auth.go      # Fnctions like Register, LoginUser and utilities for password hashing and token generation
│   │   └── auth_test.go # Unit tests for authentication functions
|   |
│   ├── broadcast/       # Handles broadcasting and notification of chat messages and active user updates
│   │   └── broadcast.go
│   ├── db/                 # Database logic and mock implementations
│   │   ├── db.go           # Functions for live MySQL database interactions (e.g., SaveMessage, GetChatHistory)
│   │   ├── db_mock.go      # Mock database implementation for testing
│   │   └── db_mock_test.go # Tests for mock database functions
|   |
│   ├── handlers/        # Request handlers for handling connections and chat history requests
│   │   └── handlers.go
│   ├── middleware/      # Custom CORS middleware to handle cross-origin requests
│   │   └── middleware.go
│   ├── models/          # Defines the data models used in the app
│   │   └── models.go
│   ├── routes/          # API route setup
│   │   └── routes.go
│   ├── services/        # Service initializations
│   │   └── services.go
│   └── utils/           # Utility functions like GetBroadcastChannel and RegisterClient
│       └── utils.go
├── frontend/
│   ├── src/
│   │   ├── App.tsx      # Main React entry point
│   │   └── Chat.tsx     # Chat component
│   │   └── TopBar.tsx   # Topbar component
│   │   ├── ....         # Other frontend bits
├── db/
│   ├── init.sql/        # Database initialisation config
├── docker-compose.yml   # Containerization configuration
└── .env                 # Example environment variables

```

# 🧑‍💻 Development Highlights

### Websockets:

I first started this project to get more hands on experiance with websockets. Initialy just for the instant messaging comunication, I later expanded this to also communicate active user updates as well.

At the moment Gorilla/websockets is defacto standard library for websockets in Go.

### **Concurrency in Go**:

This program uses concurrency by making use of Go’s Goroutines, channels, and mutex to handle tasks that can run independently and in parallel. Goroutines are lightweight threads managed by Go's runtime, allowing us to execute multiple tasks at the same time. Channels provide a way for Goroutines to communicate safely, ensuring data consistency and avoiding race conditions. Mutexes (mutual exclusions) ensure safe access to shared resources

For example, the `broadcast.StartBroadcastListener()` Goroutine listens on a shared channel to receive messages and broadcasts them to all connected clients A mutex ensures that the shared `clients` map is accessed safely:

```go
// Example Channel for broadcasting messages
var broadcast = make(chan models.Message)

// Example code from broadcast.go
// Goroutine to listen and handle messages
func StartBroadcastListener() {
	broadcast := utils.GetBroadcastChannel() //
	clients, mutex := utils.GetClients()

	for msg := range broadcast {
		messageBytes, _ := json.Marshal(msg)
		mutex.Lock() // Lock the mutex to prevent concurrent writes to the clients map

		for client := range clients {
			select {
			case client.Send <- messageBytes: // Send message to each client
			default:
				utils.DeregisterClient(client) // Remove client if unresponsive
			}
		}
		mutex.Unlock() // Unlock the mutex after processing
	}
}

// Example sending a message to the channel
func BroadcastMessage(msg models.Message) {
    broadcast <- msg // Send the message to the broadcast channel
}

// Example starting the go routine
go broadcast.StartBroadcastListener()
```

Here, `StartBroadcastListener` runs as a Goroutine and continuously listens for messages on the `broadcast` channel. When a message is received, it is sent to all connected WebSocket clients via their respective `Send` channels. This approach allows the program to handle multiple clients and messages simultaneously without blocking other tasks.

### **Session Authentication and CRFT Tokens**:

As part of this I really enjoyed learning more about session and csrf tokens, and implementing them myself from scratch. While JWT and OAuth are more modern standards, session tokens are still used a lot and learning about the security vulnerabilities introduced by those and how csrf tokens are secure against that was very interesting.

**Explination:**

The core idea is that a session token is a way of identifying a user for a given period. This token is given to the user as a cookie when they log in and can be used to identify themselves when they make a request (such as connecting to the chat web socket or accessing their profile). benefits?

However this can introduce a vulnerability called CSRF (cross sight request forgery). Because cookies are automatically sent with requests, a malicious website could redirect an unexpecting user to make a request without the users knowing.

CSRF tokens protect against this by verifying the origin of the request. By sending a user a crsf token when they login, also as a cookie, cross-origin site polliciy only allowed authorised pages to access the crsf token and attach it as a customer header.

**Downsides:**

- highly distributed systems can put a strain on reading session tokens from db.
- Improper token handling (e.g., storing session tokens insecurely) can lead to vulnerabilities.

### Dependency Injection:

- Explanation
  - What is it
  - Why use it? benefits?
- Explination of Code
  - explain why using struc
  - explain how the interface works
  - explain how Go does object orientations and inheretance
- benefits
- Expansions
  - Repositry pattern for better abstraction and easier testing and new database integration

### Middleware Patern and CORS:

Because my backend was on a different port to my frontend, i had to add Cross-Origin Resource Sharing headers to my requests. To do this I implemented a Middleware patter t

### Unit Tests:

Unit tests have been written for the auth service and the mock database. I chose to not unit test withfull code coverage because I understand unit testing, I just wanted practice in Go and the tests are more for demonstraightion rather than an actual test suit.

Some notes on unit testing in go:

- Best practice is to place test files (`_test.go`) in the same package and directory as the code they are testing. This makes it easy to find the tests and supposedly encourages writing tests alongside the code. Use separate directories for integration tests.
- Test functions should be named `TestXxx` where `Xxx` describes the test subject.
- Use `t.Run` to group related test cases in subtests.

### **DevOps Skills**:

Utilized Docker Compose to create an isolated development environment.

# 🏗️ Further Expansion

- Chat paging and offset
- Repository Pattern: I was investigating other patterns such as using a repository pattern. by doing so I could increased testability of my database code and allow easy integration with out databases. However given the size of the program, and a general less is more mindset in Go, I chose to stop at dependency injection.
- WebSocket Scalability
- Rate Limiting and other security measures
- Implement CI/CD pipelines.

## 📊 Results and Insights

- Gained proficiency in **Go** for backend development.
- Practice integrate **React.js** .
- Better knowledge of **security best practices** in web development.
- Practiced **DevOps concepts** with containerized deployment using Docker.

## 🚀 How to Run

### Prerequisites

- **Docker** installed.

### Steps

1. Clone the repository:

   ```bash
   git clone https://github.com/your-username/chat-app.git
   cd chat-app
   ```

2. Start the application using Docker Compose:

   ```bash
   Copy code
   docker-compose up --build
   ```

3. Access the app:
   - Frontend: http://localhost:3000
   - Backend API: http://localhost:8080

## 🤝 Contact

Feel free to reach out if you have questions about the project or would like to collaborate.

- **Email**: peterboughton11@gmail.com
- **LinkedIn**: [LinkedIn](https://www.linkedin.com/in/peter-semrau-boughton/)
- **GitHub**: [GitHub](https://github.com/Peter-SB)
