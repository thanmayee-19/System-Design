# 3-Tier Containerized URL Shortener System

## Project Documentation

**Project Title:** 3-Tier Containerized URL Shortener System  
**Technology Stack:** Python, Flask, PostgreSQL, Redis, Docker, Docker Compose  
**Architecture:** 3-Tier Architecture  
**Project Type:** System Design and Containerization  

---

# Table of Contents

1. [Project Overview]
2. [Objectives]
3. [System Architecture]
4. [Docker Networking]
5. [Build the Containers]
6. [Start the Application]
7. [Validate Containers]
8. [Health Check]
9. [Test URL Shortening]
10. [Test URL Redirection]
11. [Validate PostgreSQL]
12. [Validate Redis]
13. [Validate Cache Hit]
14. [Validate Cache Miss]
15. [Validate Database Persistence]
16.  [Conclusion]

---

# 1. Project Overview

The **3-Tier Containerized URL Shortener System** is a web-based application that converts long URLs into shorter URLs that are easier to share.

The system is implemented using:

- Python
- Flask
- PostgreSQL
- Redis
- Docker
- Docker Compose

The application follows a **3-tier architecture** where the Flask application acts as the application layer, PostgreSQL acts as the persistent data layer, and Redis acts as the caching layer.

The system provides two primary operations:

1. **URL Shortening**
   - The user submits a long URL.
   - The URL is stored in PostgreSQL.
   - PostgreSQL generates an auto-incrementing ID.
   - The ID is converted into a Base62 short code.
   - The mapping is stored in Redis.
   - The short URL is returned to the user.

2. **URL Redirection**
   - The user accesses the short URL.
   - Flask first checks Redis.
   - If the URL exists in Redis, the application immediately redirects the user.
   - If the URL is not present in Redis, PostgreSQL is queried.
   - The result is stored in Redis.
   - The user is redirected using HTTP status code `302`.

---

# 2. Objectives

The main objectives of this project are:

- To design a 3-tier web application.
- To implement URL shortening using Flask.
- To store URL information persistently using PostgreSQL.
- To use PostgreSQL auto-incrementing IDs for unique URL identifiers.
- To implement Base62 encoding for short URL generation.
- To implement Redis caching.
- To demonstrate the cache-aside strategy.
- To containerize the application using Docker.
- To manage multiple containers using Docker Compose.
- To implement health checks for PostgreSQL and Redis.
- To use persistent Docker volumes for database storage.
- To demonstrate communication between multiple Docker services.

---

# 3. System Architecture

The system follows a 3-tier architecture.

<img width="752" height="647" alt="image" src="https://github.com/user-attachments/assets/b833bc25-1929-43eb-8ffd-e718e2b4c04c" />

---

## 4. Docker Networking

Docker Compose creates a private network for all application containers.

The Flask Web container communicates with:

- PostgreSQL using the service name `db`
- Redis using the service name `redis`

The containers communicate internally using Docker service names instead of localhost.

---

## 5. Build the Containers

The Docker images for the Flask application, PostgreSQL, and Redis are prepared using Docker Compose.

The Flask application image is created using the Python 3.12 base image and the required Python dependencies.

---

## 6. Start the Application

The complete application is started using Docker Compose in detached mode.

After starting, three containers are available:

- Web application
- PostgreSQL database
- Redis cache

The Flask application is accessible through port 5000 on the local machine.

<img width="1385" height="952" alt="image" src="https://github.com/user-attachments/assets/7045b659-4447-4fd0-945c-04445c04b8b7" />

---

## 7. Validate Containers

The running containers are checked using Docker Compose.

The expected status is:

- Web container – Running
- PostgreSQL container – Running and Healthy
- Redis container – Running and Healthy

This confirms that all application components have started successfully.

<img width="1388" height="953" alt="image" src="https://github.com/user-attachments/assets/8c7bd2dc-8008-4d6a-9b11-8442f86c7928" />

---

## 8. Health Check

The Flask application's health endpoint is accessed to verify that the web application is running correctly.

The PostgreSQL health check verifies that the database is ready to accept connections.

The Redis health check verifies that Redis is responding correctly.

Expected result:

**Application status: Healthy**

<img width="1386" height="701" alt="image" src="https://github.com/user-attachments/assets/4d02c1aa-7a34-4919-87c7-84016e6c2d20" />


---

## 9. Test URL Shortening

A long URL is submitted to the Flask application's URL shortening endpoint.

The application:

1. Receives the original URL.
2. Stores the URL in PostgreSQL.
3. Generates a unique numeric ID.
4. Converts the ID into a Base62 short ID.
5. Stores the URL in Redis.
6. Returns the generated short URL.

The response contains the original URL, short ID, and short URL.

<img width="1391" height="778" alt="image" src="https://github.com/user-attachments/assets/37924caa-2e39-4123-a932-e24674c7d341" />

<img width="1383" height="792" alt="image" src="https://github.com/user-attachments/assets/672ce49e-db8b-443b-b3a2-497315c77df9" />

---

## 10. Test URL Redirection

The generated short URL is opened in a browser.

The Flask application retrieves the original URL and redirects the user to it using an HTTP 302 redirect.

The browser should open the original destination URL successfully.

<img width="1406" height="842" alt="image" src="https://github.com/user-attachments/assets/59798433-4d43-4a46-8a3d-32ee68281c9f" />

<img width="1350" height="940" alt="image" src="https://github.com/user-attachments/assets/4d26767f-0ae5-44aa-9434-21df1bc6f39e" />

---

## 11. Validate PostgreSQL

PostgreSQL is checked to confirm that the shortened URL has been stored successfully.

The URLs table contains:

- URL ID
- Original long URL
- Creation timestamp

This confirms that PostgreSQL is being used for persistent storage.

---

## 12. Validate Redis

Redis is checked to confirm that shortened URLs are stored in the cache.

The Redis cache contains the short ID as the key and the original URL as its value.

This allows frequently accessed URLs to be retrieved without querying PostgreSQL every time.

---

## 13. Validate Cache Hit

A cache hit occurs when the requested short ID is already available in Redis.

The request flow is:

**User → Flask → Redis → Original URL → 302 Redirect**

Since the URL is available in Redis, PostgreSQL does not need to be accessed.

This provides faster response time.

<img width="1377" height="818" alt="image" src="https://github.com/user-attachments/assets/02064b97-1f63-42b1-b731-52780f698a24" />

---

## 14. Validate Cache Miss

A cache miss occurs when the requested short ID is not present in Redis.

The request flow is:

**User → Flask → Redis Miss → PostgreSQL → Redis → 302 Redirect**

The application retrieves the URL from PostgreSQL and then stores it in Redis.

Future requests can therefore be served directly from the cache.

<img width="1362" height="928" alt="image" src="https://github.com/user-attachments/assets/30392684-4e94-4432-83d7-48f4f5877cbe" />

---

## 15. Validate Database Persistence

The PostgreSQL database uses a persistent Docker volume.

The containers can be stopped and restarted without losing the stored URL records.

After restarting the application, previously created URLs should still be available.

This confirms that database persistence is working correctly.

---

## 16. Conclusion

The URL Shortener System successfully demonstrates a containerized three-tier architecture using Flask, PostgreSQL, and Redis.

Flask handles the application logic and URL redirection.

PostgreSQL provides persistent storage for shortened URLs.

Redis provides fast caching and supports cache hit and cache miss handling.

Docker Compose manages the complete application using separate containers, health checks, networking, and persistent storage.

The system successfully demonstrates URL shortening, Base62 ID generation, database persistence, Redis caching, cache-aside behavior, and HTTP 302 redirection.
