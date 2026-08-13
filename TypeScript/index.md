---
layout: default
title: TypeScript Mastery — From Fundamentals to Senior/FAANG-Level
description: A complete, structured TypeScript curriculum — compiler internals, the type system, generics, advanced types, architecture, and a full interview reference.
---

# TypeScript Mastery

A complete, structured path from TypeScript fundamentals to senior/staff-level depth — how the compiler actually works, why the type system behaves the way it does, and how to reason about it well enough to solve unfamiliar problems and excel in technical interviews.

Every chapter follows the same format: intuition → technical explanation → compiler internals → inference behavior → real-world use cases → good/bad practices → common mistakes → edge cases → performance notes → interview questions at every level (junior → mid → senior → FAANG/tricky) → practical exercises.

**How to use this site:** read in order — each chapter builds on the last, especially Chapters 3, 6, and 8, which are the conceptual hinge points for everything after them. Chapter 16 ties the whole thing into one mental model; the Appendix is a standalone answer key you can drill against independently once you've been through the material.

---

## Part I — Foundations

- [Chapter 1 — Foundations](01-foundations.html)
  Why TypeScript exists, its relationship to JavaScript (type erasure), the compiler pipeline, transpilation vs. type-checking, `tsconfig.json`, and every major compiler option.

- [Chapter 2 — Types: Primitives to Narrowing](02-core-types.html)
  Primitives, literal types, object types, arrays, tuples, enums, `as const`, and the widening/narrowing mechanics that drive control-flow-based type refinement.

## Part II — The Type System

- [Chapter 3 — The Type System](03-type-system.html)
  Structural vs. nominal vs. duck typing, assignability as the master relation, and the full variance picture — covariance, contravariance, and TS's deliberately unsound bivariance.

## Part III — Functions & Objects

- [Chapter 4 — Functions](04-functions.html)
  Function types, optional/default parameters, overload resolution, rest parameters, and `this` typing.

- [Chapter 5 — Objects: Interfaces, Type Aliases, Unions & Intersections](05-objects.html)
  `interface` vs. `type`, declaration merging, `extends` vs. `&`, and discriminated unions.

## Part IV — Generics

- [Chapter 6 — Generics](06-generics.html)
  Generic functions, interfaces, and classes; constraints, defaults, and variadic tuple types.

## Part V — Advanced Type-Level Programming

- [Chapter 7 — Advanced Types I](07-advanced-types-1.html)
  `keyof`, `typeof`, indexed access, and mapped types — the introspection/transformation toolkit.

- [Chapter 8 — Advanced Types II](08-advanced-types-2.html)
  Conditional types, `infer`, distributive conditional types, and recursive types.

- [Chapter 9 — Advanced Types III](09-advanced-types-3.html)
  Template literal types and advanced branded/nominal type patterns.

- [Chapter 10 — Utility Types: The Complete Reference](10-utility-types.html)
  Every built-in utility type implemented from first principles, including the `Omit`-on-unions bug.

## Part VI — Modules & Compiler Features

- [Chapter 11 — Modules & Declarations](11-modules.html)
  ES modules, namespaces, `.d.ts` files, ambient declarations, and module augmentation.

- [Chapter 12 — Compiler-Level Features](12-compiler-features.html)
  Decorators (legacy vs. TC39 Stage 3), metadata/reflection, mixins, and the full declaration-merging deep dive.

## Part VII — Inference & Architecture

- [Chapter 13 — Type Inference & Control Flow Internals](13-inference-internals.html)
  How inference actually works, the control-flow-graph mechanism behind narrowing, and exhaustiveness checking.

- [Chapter 14 — Architecture & Best Practices](14-architecture.html)
  The `satisfies` operator, type-safe error handling (`Result<T,E>`), API design principles, and scaling TypeScript in large codebases.

## Part VIII — Frameworks & Capstone

- [Chapter 15 — Framework Integration](15-frameworks.html)
  React, Node/Express, NestJS, Next.js, and designing generic libraries.

- [Chapter 16 — Capstone: The Complete Mental Model](16-capstone.html)
  Every concept in the course connected into one diagram, plus a full mixed mock interview (junior → FAANG).

## Appendix

- [Master Exercise Reference](17-exercises-reference.html)
  Every practical exercise from Chapters 1–15, fully worked, with the underlying mechanism explained for each answer — the standalone drill/study companion.

---

*Curriculum status: complete — 16 chapters plus the exercise appendix.*
