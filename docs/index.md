# Architecture Lab

## Cloud Architecture • Platform Engineering • DevSecOps • AI

Welcome to my architecture engineering laboratory.

This repository contains architecture designs, infrastructure patterns,
AI applications, automation experiments, Python applications, and
platform engineering projects.

---

## Featured Architecture

### On-Premises Three-Tier LAMP Platform

A highly available enterprise web application architecture built on
on-premises Linux infrastructure.

The platform follows a classic three-tier architecture:

- **Web Tier** — Linux + Apache
- **Application Tier** — Linux + PHP
- **Database Tier** — Linux + MySQL/MariaDB

---

## Architecture at a Glance

```mermaid
flowchart TB

    USER["Users"]

    LB["Load Balancer"]

    subgraph WEB["WEB TIER"]
        WEB01["WEB01<br/>Linux + Apache"]
        WEB02["WEB02<br/>Linux + Apache"]
    end

    subgraph APP["APPLICATION TIER"]
        APP01["APP01<br/>Linux + PHP"]
        APP02["APP02<br/>Linux + PHP"]
    end

    subgraph DB["DATABASE TIER"]
        DB01["DB01<br/>Linux + MySQL/MariaDB"]
        DB02["DB02<br/>Linux + MySQL/MariaDB"]
    end

    USER --> LB

    LB --> WEB01
    LB --> WEB02

    WEB01 --> APP01
    WEB02 --> APP02

    APP01 --> DB01
    APP02 --> DB02
```