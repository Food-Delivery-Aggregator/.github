# Food Delivery Aggregator System

**Distributed Systems Mini Project – Addis Ababa Science and Technology University (AASTU)**

---

## Overview

The **Food Delivery Aggregator System** is a **microservices-based platform** connecting **customers, restaurants, and delivery agents**. It supports real-time food ordering, secure payments, and efficient delivery tracking using **Node.js**, **Express**, and an **event-driven architecture** with **RabbitMQ/Kafka**.

The project demonstrates key **distributed system concepts**: service independence, asynchronous communication, eventual consistency, and scalability.

---

## Architecture

**Core Microservices:**

| Service             | Purpose                                                 |
| ------------------- | ------------------------------------------------------- |
| **Auth Service**    | User registration, login, and role-based access control |
| **Order Service**   | Manages orders, tracks status, and publishes events     |
| **Payment Service** | Processes payments and publishes confirmation events    |

**Architecture Sketch** *[link](https://drive.google.com/file/d/1TsfNNpvfdPcuw_0-iZNFmy29JnGv_7G2/view?usp=sharing)*


**Supporting Components:**

* **API Gateway:** Routes requests to services
* **Message Broker:** Handles asynchronous communication
* **Databases:** Each service has its own PostgreSQL database
* **Redis:** Caching frequently accessed data

---

## Technologies

Node.js, Express, PostgreSQL, Redis, RabbitMQ/Kafka, Docker, JWT, Nginx/Express Gateway, Git & GitHub.

---

## Team

Mistire Daniel (Team Lead), Miraf Debebe, Nasifay Chala, Natan Addis, Nathnael Keleme, Rediet Birhanu.

