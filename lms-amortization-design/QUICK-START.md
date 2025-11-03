# Quick Start Guide

## For New Team Members

### 1. Start Here
Read these documents in order:
1. [README.md](README.md) - Project overview and navigation
2. [Problem Statement](01-problem-statement/problem-statement.md) - Understand the problem
3. [Scenario Catalog](02-scenarios/scenario-catalog.md) - See what we're building
4. [Milestone Overview](03-milestones/milestone-overview.md) - Understand the plan

### 2. Current Work
- Check [Current Status](09-progress-tracking/overall-status.md) for latest progress
- We're currently working on Milestone 1, Scenario S1.1

### 3. Key Concepts to Understand
- **Bi-temporal parameters**: Two time dimensions (valid-time and transaction-time)
- **Product versioning**: Each version is an immutable Java class
- **Migration**: Automated process with validation
- **Cascade recalculation**: Back-dated transactions affect all subsequent events

See [Glossary](10-reference/glossary.md) for complete definitions.

### 4. Contributing
- All documents are in Markdown
- Follow the document conventions in main README
- Record decisions in the ADR log
- Update status documents when completing work

## For Developers

### Setup
1. Clone this repository
2. Review technical constraints in [Assumptions](01-problem-statement/assumptions-and-constraints.md)
3. Study the [Detailed Flows](04-detailed-flows/) for scenarios you'll implement
4. Follow [Implementation Guides](07-implementation-guides/) (when available)

### Key Design Patterns
- Parameter resolution hierarchy: Loan → Product → Global
- Bi-temporal queries: Valid-time for lookups, transaction-time for audit
- Version selection: Effective date determines which version to use
- Immutability: Never edit posted transactions, always create adjustments

## For Business/Product

### Understanding Requirements
1. [Problem Statement](01-problem-statement/problem-statement.md) explains the business needs
2. [Scenario Catalog](02-scenarios/scenario-catalog.md) shows all use cases
3. [Milestone Overview](03-milestones/milestone-overview.md) shows delivery plan

### Providing Feedback
- Review scenario descriptions for accuracy
- Validate milestone priorities
- Confirm business rules in detailed flows
- Approve milestone completions

## Questions?

Check the [Glossary](10-reference/glossary.md) first, then review the relevant documentation section.
