---
title: Introduction
---

---

## Contents

- [Overview](#overview)
- [Background](#background)
- [Design](#design)
- [Approach](#approach)
- [Code Presentation](#code-presentation)
- [Technology Choices](#technology-choices)

---

## Overview

This first section sets the scene for the project:

* What are we trying to achieve?

* And why?

* What are the goals and design philosophies of the project?

* Why is it Java-based?

* How do we bind to the native Vulkan libraries?

If the reader has no interest in the _why_ and wants to get straight to the _how_ then this introductory section can be skipped.

Inevitably there is usually some _thrashing_ at the start of a new project, especially one employing new and relatively unknown technologies.  In particular there will be a major change of direction in terms of binding to the native Vulkan library, which is the subject of the following chapters in this section.  Again the reader can skip to the [start](/JOVE/blog/part-2-triangle/instance) of the blog proper with the bindings assumed to be in place.

---

## Background

For several years as a personal project we had been developing a Java-based OpenGL library and suite of demo applications.  With the advent of Vulkan we decided to archive the previous project and start afresh with a Vulkan implementation named __JOVE__.

Our rationale for this project is:

* A general interest in 3D graphics and mathematics.

* The challenge of adapting a complex native library to an object orientated (OO) design.

* A personal project that can picked up or put aside as mood dictates.

* A test-bed for learning new development approaches, technologies and Java features.

It was suggested by a peer that the start of a new project was a good opportunity to try our hand at a blog documenting the approach, design and technical decisions that were made, issues and problems faced, etc.

We have attempted to structure the blog so that each chapter builds on the preceding sections to incrementally deliver the functionality of the JOVE library, roughly following the [Vulkan tutorial](https://vulkan-tutorial.com/).

Each chapter generally consists of:

* An introduction that covers the goals(s) for that chapter and the functionality to be developed.

* A walk-through of the design and development of new or refactored software components to deliver that functionality, including any challenges or problems that arose.

* One or more _integration_ steps where we use the new functionality to extend the demo applications.

* A retrospective of any identified improvements and enhancements that lead to refactoring of existing code.

---

## Design

The general idea for the JOVE project is to provide a toolkit of Vulkan and general 3D functionality, consisting of a set of reusable components that collaborate to implement higher level features and demo applications.  This library of components should satisfy the following requirements:

* Avoid cut-and-paste code in demo applications.

* Encapsulate the inherent complexity of a Vulkan application.

* Cohesive components that can be unit-tested in relative isolation.

Having read through the [Vulkan tutorial](https://vulkan-tutorial.com/) we make the following observations that direct the detailed design:

1. The Vulkan API is (in our opinion) extremely well designed, especially considering the limitations of defining abstract data types in C.

2. Generally Vulkan is a _declarative_ API whereas OpenGL largely followed an _imperative_ approach.

3. Vulkan components are often highly configurable, functionality is declared via a C structure (referred to as a _descriptor_) which is passed to an API method to instantiate the native object.

4. However this configuration often largely consists of 'empty' or default information for even the simplest use case.  For example, the most basic rendering demo requires a pipeline configuration consisting of around a dozen descriptors, with much of that configuration being default settings.

5. Vulkan is designed to support multi-threaded applications from the ground up.

Given the above we can make the following design decisions:

* JOVE components will correspond closely to the underlying API and will follow the same naming conventions.

* We will make extensive use of the _builder_ pattern and/or static factory methods to create and configure domain objects (however constructors are often package-private for testability).

* Builders and constructors will attempt to offer sensible defaults to reduce boiler-plate code and simplify common use cases.

* All classes are _immutable_ by default unless there is a compelling reason to provide mutator methods.  In any case the majority of the Vulkan components are immutable by design, e.g. pipelines, semaphores, etc.  This also has the side benefit of simplifying the design and mitigating risk for multi-threaded applications.

* Unless explicitly stated otherwise __all__ mutable components can be considered __non__ thread safe.

* Data transfer operations are implemented using NIO buffers since this is the most convenient 'primitive' supported by the JNA library.

---

## Approach

Although this is a personal project we will still strive to apply best practice to design and development:

* Maintainability - The prime directive: code should be designed and implemented such that it can be easily extended, refactored and fixed.

* Clarity - The purpose of any code component should be clear and coherent, i.e. aim for simplicity where possible.

* Testability - All code is developed with testing in mind from the outset.

To attempt to achieve these goals our approach is:

* Aggressive refactoring of existing components to avoid code duplication and complexity, or where better understanding of Vulkan invalidates previous design decisions or assumptions.

* Comprehensive argument and state validation to identify incorrect or illogical usage of the software components.

* High test coverage with unit-tests developed _test-first_ or _test-in-parallel_ with the source code.

* Detailed documentation throughout, with a focus on examples and illustration of use-cases.

* Third-party libraries and technologies should be well-documented and widely supported.

On the other hand this _is_ a personal project so we allow ourselves some freedoms that might not be possible under the constraints of a real-world project - we can reinvent as many wheels as we choose if there is sufficient reason (or challenge) in doing so.

---

## Code Presentation

Source code is presented as fragments interspersed with commentary, rather than (for example) links to the source files followed by a discussion.

We also follow these additional coding guidelines:

* Method arguments can be assumed to be non-null by default unless explicitly specified (and documented) as optional.  Additionally null pointer exceptions are not declared in the method documentation.

* Similarly method return values can be assumed to be non-null with potentially empty results modelled by the `Optional` wrapper type.

* The `var` keyword is used to avoid duplication where the type is complex or long-winded (but is also present in the statement).

* Local variables are `final` by default.

* Latest Java features are used where appropriate or convenient, e.g. lambdas rather than anonymous classes.

* If a coding guideline is broken then this should be explicitly documented in the code.

The following are silently omitted unless their inclusion better illustrates the code:

* In-code comments and JavaDoc

* Local variable `final` modifiers

* Validation

* Trivial constructors, getters and setters

* Trivial equals, hash-code and `toString` implementations

* Exception error messages

* Warnings suppression

* Method `@Override` annotations

* Unit-tests

* Package structure

Note that the presented code represents the state of the JOVE library at that stage of development, even if that code is eventually refactored, replaced, removed, etc.  i.e. the blog is not refactored.

---

## Technology Choices

Finally in this introductory chapter we cover the various supporting technologies, frameworks and libraries used in the JOVE project.

On the face of it Java may seem an odd choice for a project which requires interacting with a very complex native library (not exactly a strength of Java).  However it the language with which we have most experience (both personal and professional) and as previously mentioned one of the interesting challenges is how to develop an OO project dependant on a C/C++ native, which is the focus of the next chapter.

We develop using the latest stable JDK release supported by the IDE (Java 16 at the time of writing).  Although not particularly relevant to the blog our IDE of choice is Eclipse (mainly out of habit).

The JOVE library and associated demo applications are implemented as Maven projects backed by a [Git repository](https://github.com/stridecolossus/JOVE).

Besides Vulkan itself the JOVE project also uses the following supporting libraries:

| __dependency__ | __purpose__ |  __scope__ |
| Apache Commons & Collections  | General helpful utilities and supporting classes | all |
| Library                       | Argument validation (another personal project) | all |
| JUnit                         | Unit-testing framework | testing |
| Mockito                       | Mocking and stubbing | testing |
| JNA                           | Interaction with native libraries | JOVE |
| GLFW                          | Management of native windows and input-devices | JOVE |
| Spring Boot                   | Dependency injection | demos only |

Where _scope_ indicates whether the library is used to support JOVE, unit-testing or demo applications.

It can be assumed that all libraries use the latest stable release versions.

Finally the blog is authored using _Markdown_ and hosted on [GitHub Pages](https://pages.github.com/) using [Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll) for rendering.  Where appropriate UML diagrams are used to illustrate how the various components collaborate, these diagrams are generated using [Mermaid](https://github.com/mermaid-js/mermaid).  However support for this extension is patchy at best and a browser extension may be required to properly render the diagrams.

