<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: LGPL-3.0-or-later
-->

# Documents for EDSOS (Explicit Distributed Single Operating System)

---

v0.2.1

---

- [Documents for EDSOS (Explicit Distributed Single Operating System)](#documents-for-edsos-explicit-distributed-single-operating-system)
  - [1 Foundations](#1-foundations)
    - [1.1 Memory-Model](#11-memory-model)
    - [1.2 Execution-Model](#12-execution-model)
  - [2 Architecture](#2-architecture)
    - [2.1 Memory-Subsystem](#21-memory-subsystem)
    - [2.2 Distributed-Memory-Protocol](#22-distributed-memory-protocol)
    - [2.3 Execution-Subsystem](#23-execution-subsystem)
    - [2.4 Hardware-Cooperative-Execution](#24-hardware-cooperative-execution)
    - [2.5 Syscall-Model](#25-syscall-model)
    - [2.6 Failure-and-Recovery](#26-failure-and-recovery)
  - [3 System-Service](#3-system-service)
    - [3.1 I/O-System](#31-io-system)
    - [3.2 Authorization-and-Authentication-System](#32-authorization-and-authentication-system)
    - [3.3 Registration-System](#33-registration-system)
  - [4 User-Programming-Model](#4-user-programming-model)
    - [4.1 Library-Developing-Reference](#41-library-developing-reference)
    - [4.2 Native-Programming-Reference](#42-native-programming-reference)
    - [4.3 POSIX-Compatibility](#43-posix-compatibility)
    - [4.4 RDMA-RC-Verbs-Compatibility](#44-rdma-rc-verbs-compatibility)
    - [4.5 Windows-App-Compatibility](#45-windows-app-compatibility)
    - [4.6 Linux-App-Compatibility](#46-linux-app-compatibility)
    - [4.7 MacOS-App-Compatibility](#47-macos-app-compatibility)
    - [4.8 API-List](#48-api-list)
    - [4.9 ABI-List](#49-abi-list)
  - [5 Engineering-Guides](#5-engineering-guides)
    - [5.1 Build-and-Test-Infrastructure](#51-build-and-test-infrastructure)
    - [5.2 Debugging-and-Profiling](#52-debugging-and-profiling)
    - [5.3 Deployment-Scenarios](#53-deployment-scenarios)


---

## 1 Foundations

本编主要介绍作为约定和接口的EDSOS系统模型框架。

### 1.1 Memory-Model
### 1.2 Execution-Model

## 2 Architecture

本编主要介绍实现了前述框架的EDSOS工程细节设计。

### 2.1 Memory-Subsystem
### 2.2 Distributed-Memory-Protocol
### 2.3 Execution-Subsystem
### 2.4 Hardware-Cooperative-Execution
### 2.5 Syscall-Model
### 2.6 Failure-and-Recovery

## 3 System-Service

本编主要介绍EDSOS的系统服务组件。

### 3.1 I/O-System
### 3.2 Authorization-and-Authentication-System
### 3.3 Registration-System

## 4 User-Programming-Model

本编主要介绍如何开展EDSOS编程，以及如何将既有程序迁移到EDSOS上运行。

### 4.1 Library-Developing-Reference
### 4.2 Native-Programming-Reference
### 4.3 POSIX-Compatibility
### 4.4 RDMA-RC-Verbs-Compatibility
### 4.5 Windows-App-Compatibility
### 4.6 Linux-App-Compatibility
### 4.7 MacOS-App-Compatibility
### 4.8 API-List
### 4.9 ABI-List

## 5 Engineering-Guides

本编主要介绍如何在各类硬件上部署EDSOS。

### 5.1 Build-and-Test-Infrastructure
### 5.2 Debugging-and-Profiling
### 5.3 Deployment-Scenarios