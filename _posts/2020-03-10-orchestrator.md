---
layout: blog
published: false
category: posts
tags: null
title: The role a task orchestrator in CI
---

## wtf is task orchestrator? and why would I care?

- CI jobs come in an almost infinite number of forms, executing project specific logic to complete their task(s)
- Common to almost all of those actions is the idea of a 'task', aka the thing(s) the CI job is supposed to complete
- Logically a 'task' looks pretty similar across different toolchains, languages, platform etc. A task has inputs, a definition of its logic/steps etc and outputs.
- This is true of a `Makefile` defining steps to construct a `golang` binary, or a `pom.xml` declaring how to build a JAR file or a `webpack.js` setting configuration for your production JS build. They are all different but almost all generalisable into our concept of a 'task'.
- Task orchestrator in this context is a tool that aims 
- At scale, the reality of modern systems is that there are going to be many moving parts that make up the whole. It is inevitable that these parts won't all be built the same way, backend vs web vs mobile vs serverless etc. As the parts and the disparity between them grows, CI optimisation becomes harder and more expensive, the same problem has to be solved in N different language/toolchain specific ways.
- The task orchestrator in this context is a single tool that works across all/most toolchains, and is a way to bridge the disparity and get economies of scale in your CI optimisation investments.

### OK.. but what does that mean?

- One of the guiding principles described [here]() is, the best way to make CI faster is to do less work. Doing the same work faster is important too, but doing the least amount of work possible is the primary goal.
- In terms of CI, where the same work/actions are repeated frequently, the best way to do less work is to only execute a given unique task (based on inputs and definition) once and share the outputs betwen other future executions of that task.
- Lots of toolchains/languages have something very similar to this, incremental builds/compilation, and its generally key to a fast local workflow and tight feedback loop. It works by having a known state of outputs (what it built previously) and just building the delta of what inputs have changed. This incremental build process however is rarely (or never) used in CI for most languages. 
- The task orchestrators primary purpose is to play this role in a toolchain and language agnostic way, you want to define your important tasks in the task orchestrator and have it execute the task as required based on changes to inputs and task definition. It will store previously generated outputs, and hash inputs and task definitions to easily re-use outputs of tasks it has previously executed. With this in place you can invest in one way to avoid repeated work for most CI jobs and avoid unnecessary work. A huge win!

## What does that look like though?

- In its most efficient form it looks like a new tool, and a set of conventions that tasks will conform to to make them supportable by the new task orchestrator tool.
- Instead of a CI Job calling `npm build` directly, it calls `orchestrator npm build` or similar and it will decide whether to actually call `npm` or not.
- One way to do this looks like Bazel (or Buck which I haven't tried). Bazel is a fairly pure incarnation of this idea, use its DSL and conventions (plus Rules and friends) to define your required tasks and then call Bazel to execute your task instead. It knows how to introspect the task definition and decide whether the task has been run before or not and avoid work if possible.
- My preferred approach is with Gradle however, which I believe is more adoptable and has better developer ergonomics.

### The problem with Bazel

- It is a pure task orchestrator, Bazel configuration and Rules bootstrap what tasks exist and what they do but it purposefully has little insight into what it is actually executing and whether there is a better way to do it.
- This sounds like the right balance to strike but is challenged by the need for fast local workflow iterations, developers want incremental builds locally and the only way to achieve that is to understand more about the thing you are building and how it works.
- If the task orchestrator is just an additional layer on top of a toolchain, like if it calls `bazel ..` which then calls `npm ..`, there is freedom underneath the orchestrator to solve for local problems, just use `webpack --dev` or whatever and ignore the orchestrator which will just be used in CI. However if there isn't clean separation and they bleed together, without huge investment and community support this often means a compromised developer experience. This is the case with Bazel and at least how it works with Java.
- In idiomatic Bazel+Java, the Bazel config defines the setup of the Java components, there is no `pom.xml` or `build.gradle` to do the actual work, there is no clean separation between orchestrator and toolchain they are one and the same. This in combination with the blackbox nature of Bazel execution and therefore lack of incremental builds forces the developer to re-structure the codebase to suit the tool, not the human.
- This structure is essentially trying to reduce the size of the component being built. Instead of building an entire functional component as a JAR which might be hundreds/thousands of classes, best practice is to break components to the package level and build them independently. This through brute-force and developer tears roughly achieves the goal of 'doing less work' while still being a blackbox. The smaller the components, the fewer inputs, the fewer components any given change is going to affect, the fewer things that will need to be built.
- In practice reducing component boundaries to sub-functional component level like a package is ergonomically painful, avoiding circular dependencies requires careful planning and refactoring, and maintaining the dependency graph of which depends on what becomes more work. Ideally component boundaries should reflect what makes sense to the humans, going this far to suit the needs of the tool doesn't seem like the right choice.
- It also has a much smaller community, poor support in lots of areas esp IntelliJ, in lots of way your are on your own. This same problem is seen in a few tools (Closure compiler comes to mind as another) that spin out of or are inspired by giant tech companies internal tools, especially Google for some reason. Where these internal tools are way ahead of their time and have evolved privately in parallel with public community tools and end up on very different paths, making different tradeoffs. Often this means that the public community tool has stronger adoption and support, has better general ergonomics and with enough investment eventually closes the gap on features and wins.
- A good analysis of the Java ergonomics problems is here: [https://blog.gradle.org/gradle-vs-bazel-jvm](https://blog.gradle.org/gradle-vs-bazel-jvm)

### My preferred approach, Gradle.

- Gradle can do everything Bazel can do, but it can also do more.
- Gradle can be a general task orchestrator and have limited insight into _what_ it is actually building, but for some first-class supported languages it has deep understanding of what a task does and how to execute it efficiently. This is an artifact of it initially starting as a replacement for Maven to build Java projects, the scope of it then expanded but Java still has the best support.
- This blurs the line between orchestrator and toolchain, same as Bazel, but in this case Gradle is the idiomatic way to configure JVM projects and has the biggest community and widest support, so it ends up being a benefit not a drawback.
- Community support and wide developer acceptance is important, it means ergonomics are a priority and have been addressed enough for it to gain adoption. It also obviously means you won't be the only one with a problem, others will have forged the path ahead for you etc too.
- Gradle does have less guardrails and thus more footguns, but that is effectively solvable with plugins to add missing guardrails/conventions.
- Details of a good Gradle setup topic for another post.

## OK ignoring the tool, what does adoption look like?

- Define important tasks in given tool, adopt required conventions (this could be tricky!)
- Change CI to execute tasks via chosen tool instead of directly calling toolchain
- Now you won't be doing the same work again and again, saving time and cycles.

## What are the gotchas?

- it is an investment both initially and overtime, and another moving part but huge point of optimisation at scale
- finding the right balance of orchestrator requirements/convention vs developer ergonomics can be tricky
- avoiding duplicate task definitions and keeping task definition up to date with an evolving code base can be effort, there are a few lessons learnt here.
- there will be language/toolchain specific work to on-board tasks onto the new approach, to understand how to define tasks, what their inputs are etc and to maintain it over time.
