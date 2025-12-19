# Food Delivery Aggregator System

**Distributed Systems Mini Project – Addis Ababa Science and Technology University (AASTU)**

---

## Overview

The **Food Delivery Aggregator System** is a **microservices-based platform** connecting **customers, restaurants, and delivery agents**. It supports food ordering, payments, and efficient delivery tracking using **Nestjs**, and an **event-driven architecture** with **RabbitMQ**.

The project demonstrates key **distributed system concepts**: service independence, asynchronous communication, eventual consistency, and scalability.

---

## Architecture

**Core Microservices:**

| Service             | Purpose                                                 |
| ------------------- | ------------------------------------------------------- |
| **Auth Service**    | User registration, login, and role-based access control |
| **Order Service**   | Manages orders, tracks status, and publishes events     |
| **Payment Service** | Processes payments and publishes confirmation events    |
| **Notification Service** | Consumes domain events and delivers system notifications |


### Architecture Sketch


<img width="783" height="690" alt="image" src="https://github.com/user-attachments/assets/0576f3d2-f96c-4cbc-897d-94ed833716da" />


**Supporting Components:**

* **API Gateway:** Routes requests to services
* **Message Broker (Rabbit-MQ):** Handles asynchronous communication
* **Databases:** Each service has its own PostgreSQL database
* **Redis:** Caching frequently accessed data

---

## Technologies

Nestjs, PostgreSQL, Redis, RabbitMQ, Docker, JWT, Nginx/Nestjs Gateway

---

## Team

* **Miraf Debebe**
* **Mistire Daniel (Team Lead)**
* **Nasifay Chala**
* **Natan Addis**
* **Nathnael Keleme**
* **Rediet Birhanu**

