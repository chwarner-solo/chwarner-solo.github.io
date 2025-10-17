# RFC-NNN: [Feature/System Name]

**Author:** [Your Name]  
**Status:** Draft  
**Created:** [YYYY-MM-DD]  
**Last Updated:** [YYYY-MM-DD]

---

**Note on Authorship:**  
[Optional: Document any collaborative aspects of the RFC creation, AI assistance, etc.]

---

## 1. Context

### Problem

[What problem are we solving? Why does this matter? What's the current pain point?]

### Goals

[What are we trying to achieve? Be specific and measurable.]

- Goal 1
- Goal 2
- Goal 3

### Non-Goals (V1)

[What are we explicitly NOT solving in this iteration? This prevents scope creep.]

- Non-goal 1
- Non-goal 2
- Non-goal 3

---

## 2. Requirements

### Must Have

[Core requirements that define the minimum viable implementation.]

- Requirement 1
- Requirement 2
- Requirement 3

### Nice to Have (Future)

[Features that would be great but aren't blockers for V1.]

- Future requirement 1
- Future requirement 2

### Explicitly Out of Scope

[Things we will NOT do, even if requested. Explain why.]

- Out of scope 1
- Out of scope 2

---

## 3. High-Level Design

### Architecture Flow

[Describe how data/requests flow through the system. Use text diagrams or bullet points.]

```
Component A → Component B → Component C
  ↓
Component D
```

### Design Patterns

[If using specific patterns (Ports & Adapters, CQRS, etc.), explain why.]

### Data Model

[Define key entities, relationships, and data structures. Use schemas, Cypher patterns, or ERDs.]

```
[Show data structure here]
```

### Key Components

[List the major components/modules and their responsibilities.]

**Component 1:**
- Responsibility A
- Responsibility B

**Component 2:**
- Responsibility A
- Responsibility B

---

## 4. Detailed Architecture (if applicable)

[For complex systems using Ports & Adapters or similar patterns, detail the layers.]

### Interfaces/Behaviors

[Define contracts/interfaces that implementations must satisfy.]

```elixir
[Code example of behavior/interface]
```

### Adapters/Implementations

[Describe concrete implementations and how they're configured.]

### Domain Logic

[Explain core business logic separate from infrastructure concerns.]

---

## 5. Task Breakdown (2-hour increments)

### Task 1: [Task Name]

**Objective:** [What does this task accomplish?]

**Implementation:**
- Step 1
- Step 2
- Step 3

**Success Criteria:**
- Criterion 1
- Criterion 2

---

### Task 2: [Task Name]

**Objective:** [What does this task accomplish?]

**Implementation:**
- Step 1
- Step 2
- Step 3

**Success Criteria:**
- Criterion 1
- Criterion 2

---

[Continue with additional tasks. Aim for 5-15 tasks for most features.]

---

## 6. Data Flow Examples

### [Key Operation 1]

[Show step-by-step how data flows for a critical operation.]

```
Step 1: User action
  ↓
Step 2: System response
  ↓
Step 3: Database update
  ↓
Step 4: Final result
```

### [Key Operation 2]

[Another important flow, especially for error cases or edge scenarios.]

---

## 7. Success Criteria

### End-to-End Test

[Define the complete flow that validates the entire feature works.]

1. Step 1
2. Step 2
3. Step 3
4. Expected outcome

### Unit/Integration Tests

[Key test scenarios that must pass.]

- Test scenario 1
- Test scenario 2
- Test scenario 3

---

## 8. Security Considerations

[If applicable, document security implications and mitigations.]

- Security concern 1 → Mitigation
- Security concern 2 → Mitigation

---

## 9. Performance Considerations

[If applicable, document performance expectations and constraints.]

- Expected load: [e.g., 100 requests/second]
- Latency requirements: [e.g., <200ms p95]
- Scalability concerns: [e.g., horizontal scaling strategy]

---

## 10. Open Questions

[Document unresolved decisions or areas needing further research.]

1. **Question 1:** [Description of uncertainty]
    - Option A: [Pro/Con]
    - Option B: [Pro/Con]

2. **Question 2:** [Description of uncertainty]

---

## 11. Alternatives Considered

[Document other approaches considered and why they were rejected.]

### Alternative 1: [Approach Name]

**Pros:**
- Pro 1
- Pro 2

**Cons:**
- Con 1
- Con 2

**Decision:** [Why we chose not to pursue this]

### Alternative 2: [Approach Name]

[Same structure as above]

---

## 12. Future Enhancements (Post-V1)

[Features and improvements planned for future iterations.]

- Enhancement 1
- Enhancement 2
- Enhancement 3

---

## 13. Migration/Rollout Strategy

[If applicable, how will this be deployed? Gradual rollout? Feature flags? Data migration?]

- Phase 1: [Description]
- Phase 2: [Description]
- Phase 3: [Description]

---

## 14. Monitoring & Observability

[How will we know this is working in production?]

- Metric 1: [What to measure]
- Metric 2: [What to measure]
- Alert conditions: [When to page someone]

---

## 15. Dependencies

[List external dependencies, APIs, services, or other RFCs this relies on.]

- Dependency 1: [Description]
- Dependency 2: [Description]

---

## 16. References

[Link to relevant documentation, similar implementations, research papers, etc.]

- [Link 1 title](URL)
- [Link 2 title](URL)

---

## 17. Changelog

[Track major changes to the RFC during the design phase.]

- **YYYY-MM-DD:** Initial draft
- **YYYY-MM-DD:** Updated based on feedback from [person/source]
- **YYYY-MM-DD:** Status changed to Accepted

---

**End of RFC-NNN**