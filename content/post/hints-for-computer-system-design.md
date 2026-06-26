---
title: Hints for Computer System Design
date: "2026-06-26T07:36:06Z"
author: bubnovd
authorTwitter: bubnovdnet
image: "/img/nebula/logo.jpg"
description: Nebula - overlay сеть от разработчиков Slack в качестве замены традиционных VPN сетей
tags:
- Computer Science
- System Design
- Lampson
keywords:
- Computer Science
- System Design
- Lampson
showFullContent: false
readingTime: true
hideComments: false
---


# Hints for Computer System Design спустя 40 лет

## Введение

-   Кто такой Батлер Лэмпсон.
-   Почему статья 1983 года до сих пор цитируется.
-   Чем она необычна (не учебник, а набор инженерных эвристик).
-   Что удивительно --- многие современные идеи уже были описаны там.

------------------------------------------------------------------------

# Раздел 1. Простота важнее универсальности

## 2.1 Keep it simple

-   Главная мысль
-   Пример Лэмпсона: Alto
-   Современный пример: Kubernetes Controller

## 2.2 Make it fast rather than general

-   Главная мысль
-   Пример Лэмпсона: Tenex
-   Современный пример: специализированные API Kubernetes вместо
    универсальных CRD

## 2.3 Separate normal and worst cases

-   Главная мысль
-   Пример Лэмпсона: Bravo
-   Современный пример: ClickHouse MergeTree / LSM Compaction

## 2.4 Making implementations work

-   Главная мысль
-   Пример Лэмпсона: Mesa
-   Современный пример: Kubernetes Informers

## 2.5 Handling all the cases

-   Главная мысль
-   Пример Лэмпсона: Reference Counting + Trace GC
-   Современный пример: PostgreSQL VACUUM

------------------------------------------------------------------------

# Раздел 2. Производительность

## Split resources

-   Пример Лэмпсона: разделение памяти
-   Современный пример: Node Pools / Read Replicas

## Static analysis

-   Пример Лэмпсона: компилятор
-   Современный пример: Helm Template / SQL Planner

## Dynamic translation

-   Пример Лэмпсона: динамическая трансляция внутреннего представления
-   Современный пример: JIT / eBPF

## Cache answers

-   Пример Лэмпсона: виртуальная память
-   Современный пример: Informer Cache / DNS

## Use hints

-   Пример Лэмпсона: Smalltalk Inline Cache
-   Современный пример: pg_statistic / Data Skipping Index

## When in doubt, use brute force

-   Пример Лэмпсона: перестроение структуры
-   Современный пример: Terraform Refresh / Kubernetes Reconciliation

## Compute in background

-   Пример Лэмпсона: Garbage Collection
-   Современный пример: ClickHouse Merge / PostgreSQL VACUUM

## Batch processing

-   Пример Лэмпсона: I/O
-   Современный пример: Kafka Batch / PostgreSQL COPY

## Safety first

-   Пример Лэмпсона: Thrashing
-   Современный пример: Requests & Limits / PgBouncer

## Shed load to control demand

-   Пример Лэмпсона: виртуальная память
-   Современный пример: HTTP 429 / Circuit Breaker

------------------------------------------------------------------------

# Раздел 3. Отказоустойчивость

## End-to-end

-   Пример Лэмпсона: копирование файла
-   Современный пример: TCP vs бизнес-подтверждение / банковский перевод

## Log updates

-   Пример Лэмпсона: Bravo
-   Современный пример: WAL / Kafka

## Make actions atomic or restartable

-   Пример Лэмпсона: Commit
-   Современный пример: Kubernetes PUT / Idempotency-Key

## Use hints (в контексте восстановления)

-   Пример Лэмпсона: восстановление файловой системы
-   Современный пример: Rebuild Index / Informer Cache

------------------------------------------------------------------------

# Заключение

Основные идеи статьи:

-   Делай систему проще.
-   Не выполняй одну и ту же работу дважды.
-   Переноси тяжёлую работу из критического пути пользователя.
-   Храни один источник истины, а всё остальное считай производными
    структурами.
-   Проектируй систему, исходя из неизбежности сбоев.

## Главная идея статьи

Показать, что отдельные советы Лэмпсона образуют единую инженерную
философию:

-   **Truth → Hints → Cache → Log**
-   **Normal/Worst Case → Background → Batch**
-   **End-to-End → Atomic → Restartable**
