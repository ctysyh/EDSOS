<!--
SPDX-FileCopyrightText: © 2025-2026 Bib Guake
SPDX-License-Identifier: LGPL-3.0-or-later
-->

# Documents for EDSOS Kernel
*Explicit Distributed Single Operating System Kernel*

---

v0.2.1

---

- [Documents for EDSOS Kernel](#documents-for-edsos-kernel)
  - [1 Core-Models](#1-core-models)
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

## 1 Core-Models

[本编](./1-Core-Models/)主要介绍作为约定的 EDSOS Kernel 系统模型框架。值得注意的是，本编描述EDSOS Kernel内部的核心模型，基于但不同于Arbor Strux理论；Arbor Strux理论的完整内容请参考[Arbor Strux文档目录](../../Arbor-Strux/)。

### 1.1 Memory-Model

[本篇](./1-Core-Models/1.1-Memory-Model.md)介绍内存模型的基本结构，并详细说明关键数据结构。

### 1.2 Execution-Model

[本篇](./1-Core-Models/1.2-Execution-Model.md)介绍执行模型的基本结构，并介绍执行子系统的基本组织形式。

---

## 2 Architecture

[本编](./2-Architecture/)主要介绍实现了前述框架的 EDSOS Kernel 工程细节设计。

### 2.1 Memory-Subsystem

[本篇](./2-Architecture/2.1-Memory-Subsystem.md)介绍内存子系统的基本组织形式，并详细介绍以回调形式供执行子系统调度器调用的内存管理逻辑。

### 2.2 Distributed-Memory-Protocol

[本篇](./2-Architecture/2.2-Distributed-Memory-Protocol.md)介绍基于 EDSOS Kernel 分布式内存模型的几个数据一致性协议。

### 2.3 Execution-Subsystem

[本篇](./2-Architecture/2.3-Execution-Subsystem.md)介绍执行子系统的各个组件详情，说明 EDSOS Kernel 从系统启动开始的全生命周期运行方式。

### 2.4 Hardware-Cooperative-Execution

[本篇](./2-Architecture/2.4-Hardware-Cooperative-Execution.md)介绍内存子系统和执行子系统的硬件交互面细节，包括对 Native-ISA 的利用和在其他 ISA 上的模拟实现。

### 2.5 Syscall-Model

[本篇](./2-Architecture/2.5-Syscall-Model.md)介绍 EDSOS Kernel 的系统调用模型基本结构与用法。

### 2.6 Failure-and-Recovery

[本篇](./2-Architecture/2.6-Failure-and-Recovery.md)介绍 EDSOS Kernel 在各类边界条件和错误情景下的失败、回退与修复逻辑。

---

## 3 System-Service

[本编](./3-System-Service/)主要介绍 EDSOS Kernel 的系统服务组件。

### 3.1 I/O-System
### 3.2 Authorization-and-Authentication-System
### 3.3 Registration-System

## 4 User-Programming-Model

[本编](./4-User-Programming-Model/)主要介绍如何开展面向 EDSOS Kernel 的应用程序编程，以及如何将既有程序迁移到 EDSOS Kernel 上运行。

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

[本编](./5-Engineering-Guides/)主要介绍如何在各类硬件上部署 EDSOS Kernel。

### 5.1 Build-and-Test-Infrastructure
### 5.2 Debugging-and-Profiling
### 5.3 Deployment-Scenarios

---

*End of Document.*