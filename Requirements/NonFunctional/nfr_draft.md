# Architecture Characteristics Worksheet Template

**System/Project:**  MobilityCorp

**Architect/Team:**  Katalysis

**Domain/Quantum:**  Entire domain

**Date Modified:**   10/19/2025

**Next Review Date:** 10/21/2025

---

## Top 3 Driving Characteristics

| # | Driving Characteristic | Implicit Characteristic |
|---|------------------------|-------------------------|
| 1* | Availability | Feasibility (cost/time) |
| 2* | Responsiveness | Security |
| 3* | Agility | Maintainability |
| 4 | Interoperability | Observability |
| 5 | Configurability |  |
| 6 | Scalability |  |
| 7 | Extensability |  |

---

## Others Considered

_List any additional characteristics identified that weren’t deemed as important as the top 7._

---

## Instructions

- Identify **no more than 7 driving characteristics**.  
- Pick the **top 3 characteristics** (in any order).  
- **Implicit characteristics** can become driving characteristics if they are critical concerns.  
- Add additional characteristics identified that weren’t deemed as important as the list of 7 to the **“Others Considered”** section.  
- **Definitions** are on the following page.

---

## Common Architectural Characteristics

| Characteristic | Definition |
|----------------|-------------|
| **Performance** | The amount of time it takes for the system to process a business request. |
| **Responsiveness** | The amount of time it takes to get a response to the user. |
| **Availability** | The amount of uptime of a system; usually measured in 9’s (e.g., 99.9%). |
| **Fault Tolerance** | When fatal errors occur, other parts of the system continue to function. |
| **Scalability** | As the number of users or requests increase in the system, responsiveness, performance, and error rates remain constant. |
| **Elasticity** | The system can expand and respond quickly to unexpected or anticipated extreme loads. |
| **Data Integrity** | The data across the system is correct and there is no data loss. |
| **Data Consistency** | The data across the system is in sync and consistent across databases and tables. |
| **Adaptability** | The ease in which a system can adapt to changes in environment and functionality. |
| **Concurrency** | The ability of the system to process simultaneous requests in order of receipt. |
| **Interoperability** | The ability of the system to interact with other systems to complete a business request. |
| **Extensibility** | The ease in which a system can be extended with additional features and functionality. |
| **Deployability** | The ceremony, frequency, and risk involved in releasing software. |
| **Testability** | The ease and completeness of testing. |
| **Abstraction** | The level at which parts of the system are isolated from other parts. |
| **Workflow** | The ability of the system to manage complex workflows across services. |
| **Configurability** | The ability to support multiple or custom configurations and updates. |
| **Recoverability** | The ability to resume from a crash or failure. |

---

## Implicit Architectural Characteristics

| Characteristic | Definition |
|----------------|-------------|
| **Feasibility** | Considering timeframes, budgets, and developer skills when making architectural choices. |
| **Security** | The ability of the system to restrict access to sensitive information or functionality. |
| **Maintainability** | The level of effort required to locate and apply changes to the system. |
| **Observability** | The ability of a system to expose metrics such as health, uptime, and performance. |

---

