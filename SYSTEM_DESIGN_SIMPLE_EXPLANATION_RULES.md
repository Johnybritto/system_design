# Simple Explanation Rules

## Purpose

This file defines how every system design lesson should be explained.

The learner should be treated as an average student who wants simple, practical, interview-ready understanding without unnecessary jargon or information overload.

The goal is not to sound advanced first. The goal is to understand first, then slowly build toward senior-level interview answers.

---

## Main Rule

Every explanation must be simple first, deep later.

Use this order:

```text
1. Plain English meaning
2. Simple real-life analogy
3. Small diagram
4. Basic example
5. AWS version
6. Azure version
7. Kubernetes/platform version
8. Failure scenario
9. Interview-ready answer
10. Revision note
```

Do not start with too many service names, acronyms, or advanced trade-offs.

---

## Explanation Style

Use:

- Simple words
- Short sentences
- Small diagrams
- One concept at a time
- Practical examples
- Interview-focused language
- Repetition of key memory lines

Avoid:

- Large theory dumps
- Too many cloud services in one answer
- Too many acronyms without explanation
- Jumping from beginner concept to advanced architecture too quickly
- Covering AWS, Azure, Kubernetes, HA, DR, security, and observability all at once unless the micro-lesson specifically requires it

---

## Every Concept Should Have a Memory Line

Each lesson should include one simple memory line.

Example:

```text
Load Balancer distributes traffic.
API Gateway controls API access.
```

Example:

```text
HPA adds pods.
Cluster Autoscaler adds nodes.
```

Example:

```text
Lambda is for short event-driven work.
EC2 is for long-running compute and control.
```

---

## Use Layered Explanation

Each topic should be explained in layers.

### Layer 1 — Beginner Meaning

Explain what it means in plain English.

### Layer 2 — Why It Exists

Explain the problem it solves.

### Layer 3 — Simple Architecture

Use a small text diagram.

### Layer 4 — Cloud Mapping

Map to AWS and Azure only after the basic idea is clear.

### Layer 5 — Kubernetes / Platform Mapping

Explain how it works in Kubernetes, EKS, AKS, OpenShift, GitOps, or platform engineering.

### Layer 6 — Interview Answer

Convert the concept into a short interview answer.

---

## Micro-Lesson Size Control

Each micro-lesson should answer only one main question.

Good:

```text
What is the difference between ALB and NLB?
```

Too broad:

```text
Explain load balancers, API gateways, ingress, Gateway API, Azure equivalents, HA, security, observability, and cost.
```

If a topic feels large, split it.

---

## Required Output Per Micro-Lesson

Every micro-lesson must produce:

1. One simple explanation
2. One diagram
3. One comparison table if useful
4. One small exercise
5. One interview-ready answer
6. One revision note update

---

## Example Format

Use this format wherever possible:

```markdown
# Micro-Lesson X — Topic Name

## 1. Simple Meaning

Explain in plain English.

## 2. Why It Is Needed

Explain the problem it solves.

## 3. Simple Diagram

```text
User
 |
 v
Component
 |
 v
Backend
```

## 4. AWS Version

Explain only the relevant AWS service.

## 5. Azure Version

Explain only the relevant Azure service.

## 6. Kubernetes Version

Explain the Kubernetes/platform equivalent.

## 7. Failure Case

Explain one thing that can fail.

## 8. Small Exercise

Ask one short question.

## 9. Interview-Ready Answer

Give a short polished answer.

## 10. Revision Note

Write the final summary into revision notes.
```

---

## Difficulty Progression

Each topic should progress like this:

```text
Beginner understanding
   -> Practical cloud example
      -> Kubernetes/platform example
         -> Production failure case
            -> Interview-ready answer
```

Do not jump directly to advanced architecture.

---

## Learner-Friendly Rules

- Assume the learner may forget previous details.
- Briefly recap before continuing.
- Use the same example across related lessons when possible.
- Prefer one strong example over five weak examples.
- Ask one exercise only.
- Keep revision notes short but useful.
- Mark weak areas clearly.

---

## Interview Depth Rule

Simple does not mean shallow.

The lesson should start simple, but the final answer should still include senior-level thinking:

- Trade-off
- Failure handling
- Security
- Observability
- Cost or operational impact

But these should be added gradually after the basic idea is clear.

---

## Final Rule

If the explanation feels heavy, split it again.

The project goal is not to finish fast.

The project goal is to retain concepts and explain them clearly in interviews.
